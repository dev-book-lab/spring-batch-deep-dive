# JobParameters와 실행 컨텍스트 — 파라미터 주입과 Step 간 데이터 전달

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `JobParameters`와 `ExecutionContext`는 어떻게 다른가? 둘 다 데이터를 담는데 왜 구분하는가?
- `@Value("#{jobParameters['date']}")`가 런타임에 값을 주입받는 정확한 메커니즘은?
- `@StepScope`·`@JobScope` 없이 `JobParameters`를 주입받으면 왜 `null`이 되는가?
- Job-scope `ExecutionContext`와 Step-scope `ExecutionContext`의 공유 범위 차이는?
- `JobParametersIncrementer`는 어떻게 매 실행마다 고유한 파라미터를 생성하는가?

---

## 🔍 왜 두 가지 데이터 전달 수단이 필요한가

### 문제: 실행 입력값과 실행 중 상태는 성격이 다르다

```
배치 실행 시 데이터의 두 가지 유형:

  유형 1 — 실행 입력값 (JobParameters)
    "2024-01-01 정산 배치를 실행하라"
    → targetDate=2024-01-01
    → 실행 전에 결정됨, 실행 중 변경 안 됨
    → JobInstance 식별에 사용 (같은 파라미터 = 같은 JobInstance)
    → DB에 BATCH_JOB_EXECUTION_PARAMS로 영구 저장

  유형 2 — 실행 중 상태 (ExecutionContext)
    "Step 1이 처리한 파일 경로를 Step 2에 전달"
    "지금까지 읽은 레코드 수 (재시작용)"
    → 실행 중에 생성·변경됨
    → JobInstance 식별과 무관
    → DB에 BATCH_[JOB/STEP]_EXECUTION_CONTEXT로 저장 (재시작용)

두 개념이 합쳐지면:
  "파라미터로 전달된 값"과 "처리 중에 생성된 값"을 구분할 수 없음
  → 재시작 시 어떤 값이 입력값이고 어떤 값이 중간 상태인지 불명확
```

---

## 😱 흔한 실수

### Before: @StepScope 없이 JobParameters를 주입받는다

```java
// ❌ @StepScope 없는 Bean에서 JobParameters SpEL 주입 시도
@Bean
// @StepScope 빠진 상태
public ItemReader<Order> orderReader(
        @Value("#{jobParameters['targetDate']}") String targetDate) {
    // targetDate = null !
    // Spring Context 초기화 시점에는 JobParameters가 없음
    // Bean 생성 시 SpEL 평가 → JobParameters 없음 → null
    return new JpaPagingItemReader<>()
        .queryString("SELECT o FROM Order o WHERE o.date = '" + targetDate + "'");
}

// ✅ 올바른 방법
@Bean
@StepScope  // Job 실행 시 Step 시작 시점에 Bean 생성 → JobParameters 사용 가능
public ItemReader<Order> orderReader(
        @Value("#{jobParameters['targetDate']}") String targetDate) {
    // targetDate = "2024-01-01" ← 정상 주입
    return new JpaPagingItemReaderBuilder<Order>()
        .queryString("SELECT o FROM Order o WHERE o.orderDate = :targetDate")
        .parameterValues(Map.of("targetDate", LocalDate.parse(targetDate)))
        .build();
}
```

### Before: Step 간 데이터 전달에 인스턴스 변수를 사용한다

```java
// ❌ 상태를 인스턴스 변수에 저장 (재시작 시 유실됨)
@Component
public class BatchState {
    public String downloadedFilePath;  // Step 1이 저장, Step 2가 읽음
}

// → 서버 재시작 시 인스턴스 변수는 사라짐
// → 클러스터 환경(여러 인스턴스)에서 Step 1 서버 A, Step 2 서버 B → 공유 불가

// ✅ ExecutionContext 사용
// Step 1의 Listener에서:
execution.getJobExecution().getExecutionContext()
    .put("downloadedFilePath", "/data/orders_20240101.csv");

// Step 2의 Reader에서:
String path = chunkContext.getStepContext()
    .getJobExecutionContext().get("downloadedFilePath").toString();
```

---

## ✨ 올바른 이해와 사용

### JobParameters 타입과 identifying 속성

```java
// JobParameters 생성
JobParameters params = new JobParametersBuilder()
    // identifying=true (기본값) — JOB_KEY 계산에 포함 (JobInstance 식별)
    .addString("targetDate", "2024-01-01")           // identifying=true
    .addLong("batchSize", 1000L)                      // identifying=true
    .addDouble("exchangeRate", 1350.5)                // identifying=true
    .addLocalDate("processDate", LocalDate.now())     // Spring Batch 5.x

    // identifying=false — DB에 저장되지만 JOB_KEY 계산 제외
    .addLong("timestamp", System.currentTimeMillis(), false)  // 고유성 보장용
    .addString("requestId", UUID.randomUUID().toString(), false)

    .toJobParameters();

// JobParameters 접근
String targetDate = params.getString("targetDate");
Long batchSize = params.getLong("batchSize");
```

### @StepScope와 @JobScope의 Late Binding 메커니즘

```java
// @StepScope가 동작하는 원리
// 1. Bean 정의 시점: CGLIB Proxy가 생성됨 (실제 Bean 아님)
// 2. Step 실행 시점: StepContext가 활성화되면 실제 Bean 생성
// 3. SpEL 평가 시점: 실제 Bean 생성 시 = Step 실행 시 = JobParameters 존재

@Bean
@StepScope
public FlatFileItemReader<Order> csvReader(
        @Value("#{jobParameters['inputFile']}") String inputFile,
        @Value("#{stepExecutionContext['partitionKey']}") String partitionKey) {
    // inputFile: jobParameters에서 Late Binding
    // partitionKey: stepExecutionContext에서 Late Binding (Partitioning 시)
    return new FlatFileItemReaderBuilder<Order>()
        .resource(new FileSystemResource(inputFile))
        .build();
}

@Bean
@JobScope  // Job 실행 시점에 Bean 생성 (Step보다 넓은 범위)
public Step processStep(
        @Value("#{jobParameters['chunkSize']}") Integer chunkSize) {
    return stepBuilderFactory.get("processStep")
        .<Order, SettledOrder>chunk(chunkSize)  // 파라미터로 동적 Chunk Size
        .reader(csvReader(null, null))  // null 전달 — Proxy가 처리
        .build();
}
```

### ExecutionContext — Job-scope vs Step-scope

```java
// Job-scope ExecutionContext: 모든 Step이 공유
// Step 1에서 저장
@Component
public class Step1Listener implements StepExecutionListener {
    @Override
    public ExitStatus afterStep(StepExecution stepExecution) {
        // Job EC에 저장 — Step 2에서 읽을 수 있음
        stepExecution.getJobExecution().getExecutionContext()
            .put("processedFileCount", downloadedFiles.size());
        return ExitStatus.COMPLETED;
    }
}

// Step 2에서 조회
@Bean
@StepScope
public Tasklet step2Tasklet(
        @Value("#{jobExecutionContext['processedFileCount']}") Integer fileCount) {
    return (contribution, chunkContext) -> {
        log.info("처리할 파일 수: {}", fileCount);
        return RepeatStatus.FINISHED;
    };
}

// Step-scope ExecutionContext: 해당 Step 내부에서만 사용
// 주로 ItemStream이 재시작 포인트 저장용으로 사용
@Override
public void update(ExecutionContext executionContext) {
    // Step EC에 저장 — 이 Step의 재시작 포인트
    executionContext.put(getExecutionContextKey("read.count"), currentReadCount);
    // 다른 Step에서는 이 값을 볼 수 없음
}
```

---

## 🔬 내부 동작 원리

### 1. @StepScope Proxy 생성 과정

```java
// StepScope.java — Proxy 생성
public class StepScope implements BeanFactoryPostProcessor, Scope {

    @Override
    public Object get(String name, ObjectFactory<?> objectFactory) {
        // ① 현재 StepExecution에서 Bean 캐시 확인
        StepContext context = StepSynchronizationManager.getContext();
        if (context == null) {
            throw new IllegalStateException(
                "No StepContext! @StepScope Bean을 Step 외부에서 호출할 수 없습니다.");
        }

        // ② StepContext에 Bean이 없으면 새로 생성
        Object scopedObject = context.getAttribute(name);
        if (scopedObject == null) {
            scopedObject = objectFactory.getObject();  // 실제 Bean 생성 (SpEL 평가)
            context.setAttribute(name, scopedObject);
        }
        return scopedObject;
    }
}

// SpEL 평가 경로:
// @Value("#{jobParameters['date']}") 
// → BeanExpressionContext가 jobParameters 객체를 SpEL 컨텍스트에 등록
// → StandardEvaluationContext.rootObject = BeanExpressionContext
// → jobParameters['date'] = jobParameters.getParameter("date").getValue()
```

### 2. JobParametersIncrementer 구현

```java
// RunIdIncrementer (Spring Batch 기본 제공)
public class RunIdIncrementer implements JobParametersIncrementer {

    private static final String RUN_ID_KEY = "run.id";

    @Override
    public JobParameters getNext(@Nullable JobParameters parameters) {
        JobParameters params = (parameters == null) ? new JobParameters() : parameters;

        // 이전 run.id + 1
        long runId = params.getLong(RUN_ID_KEY, 0L) + 1;

        return new JobParametersBuilder(params)
            .addLong(RUN_ID_KEY, runId)
            .toJobParameters();
    }
}

// 날짜 기반 커스텀 Incrementer
@Component
public class DailyJobParametersIncrementer implements JobParametersIncrementer {

    @Override
    public JobParameters getNext(@Nullable JobParameters parameters) {
        String today = LocalDate.now().toString();  // "2024-01-15"

        if (parameters != null && today.equals(parameters.getString("targetDate"))) {
            // 같은 날 재시도: 시도 횟수 증가
            long attempt = parameters.getLong("attempt", 0L) + 1;
            return new JobParametersBuilder(parameters)
                .addLong("attempt", attempt)
                .toJobParameters();
        }

        // 날짜 변경 시: 새 날짜로 파라미터 초기화
        return new JobParametersBuilder()
            .addString("targetDate", today)
            .addLong("attempt", 1L)
            .toJobParameters();
    }
}
```

### 3. ExecutionContext 직렬화·역직렬화

```java
// Jackson2ExecutionContextStringSerializer (Spring Batch 5.x 기본)
// ExecutionContext → JSON 문자열

// 저장 예시 (BATCH_STEP_EXECUTION_CONTEXT.SHORT_CONTEXT):
// {"@class":"java.util.HashMap",
//  "FlatFileItemReader.read.count":500000,
//  "FlatFileItemReader.read.count.max":-1}

// 커스텀 객체 저장 시 주의:
// ExecutionContext에 커스텀 객체를 저장하면
// Jackson이 직렬화할 수 있어야 함 (@JsonTypeInfo 필요할 수도 있음)

// ✅ 권장: 기본 타입(String, Long, int)만 저장
executionContext.put("lastProcessedOrderId", 500000L);
executionContext.put("partitionKey", "REGION_A");

// ❌ 비권장: 복잡한 객체 저장 (직렬화 위험)
executionContext.put("order", order);  // Order 객체 직렬화 문제 가능
```

---

## 💻 실전 구현

### 동적 Chunk Size + 날짜 파라미터 활용

```java
@Configuration
public class DynamicJobConfig {

    @Bean
    public Job dynamicJob(JobBuilderFactory jobBuilderFactory,
                           @Qualifier("dynamicStep") Step dynamicStep) {
        return jobBuilderFactory.get("dynamicJob")
            .incrementer(new RunIdIncrementer())
            .validator(new DefaultJobParametersValidator(
                new String[]{"targetDate", "chunkSize"},  // 필수 파라미터
                new String[]{"description"}               // 선택 파라미터
            ))
            .start(dynamicStep)
            .build();
    }

    @Bean("dynamicStep")
    @JobScope  // chunkSize 파라미터를 Late Binding
    public Step dynamicStep(StepBuilderFactory stepBuilderFactory,
                             @Value("#{jobParameters['chunkSize']}") Integer chunkSize,
                             ItemReader<Order> reader,
                             ItemWriter<Order> writer) {
        return stepBuilderFactory.get("dynamicStep")
            .<Order, Order>chunk(chunkSize)
            .reader(reader)
            .writer(writer)
            .build();
    }

    @Bean
    @StepScope
    public JpaPagingItemReader<Order> reader(
            EntityManagerFactory emf,
            @Value("#{jobParameters['targetDate']}") String targetDate) {
        return new JpaPagingItemReaderBuilder<Order>()
            .name("orderReader")
            .entityManagerFactory(emf)
            .queryString("SELECT o FROM Order o WHERE o.orderDate = :date AND o.status = 'PENDING'")
            .parameterValues(Map.of("date", LocalDate.parse(targetDate)))
            .pageSize(1000)
            .build();
    }
}
```

### Step 간 데이터 전달 패턴

```java
// Step 1 완료 후 결과를 Job EC에 저장
@Component
public class Step1CompletionListener implements StepExecutionListener {

    @Override
    public ExitStatus afterStep(StepExecution stepExecution) {
        long processedCount = stepExecution.getWriteCount();
        String outputFile = "/data/output_" + LocalDate.now() + ".csv";

        // Job EC에 저장 → 다음 Step에서 접근 가능
        stepExecution.getJobExecution().getExecutionContext()
            .put("step1.processedCount", processedCount)
            .put("step1.outputFile", outputFile);

        return ExitStatus.COMPLETED;
    }
}

// Step 2 Tasklet에서 Job EC 조회
@Bean
@StepScope
public Tasklet reportTasklet(
        @Value("#{jobExecutionContext['step1.processedCount']}") Long processedCount,
        @Value("#{jobExecutionContext['step1.outputFile']}") String outputFile) {

    return (contribution, chunkContext) -> {
        log.info("보고서 생성: {} 건 처리, 파일: {}", processedCount, outputFile);
        emailService.sendReport(outputFile, processedCount);
        return RepeatStatus.FINISHED;
    };
}
```

---

## 📊 JobParameters vs ExecutionContext 비교

```
┌───────────────────┬─────────────────────────────────┬──────────────────────────────────────┐
│ 항목               │ JobParameters                   │ ExecutionContext                     │
├───────────────────┼─────────────────────────────────┼──────────────────────────────────────┤
│ 목적               │ 실행 입력값                        │ 실행 중 생성된 상태                      │
│ 변경 가능 여부       │ 불변 (실행 중 변경 불가)             │ 가변 (실행 중 지속 변경)                  │
│ JobInstance 영향   │ JobInstance 식별에 사용            │ JobInstance 식별과 무관                │
│ 저장 위치           │ BATCH_JOB_EXECUTION_PARAMS      │ BATCH_[JOB/STEP]_EXECUTION_CONTEXT   │
│ 접근 방법           │@Value("#{jobParameters['k']}")  │ @Value("#{jobExecutionContext['k']}")│
│                   │stepExecution.getJobParameters() │ stepExecution.getExecutionContext()  │
│ 재시작 역할          │ JobInstance 재식별               │ 처리 재개 포인트 (read.count 등)          │
└───────────────────┴─────────────────────────────────┴──────────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
@StepScope Late Binding:

  장점
    JobParameters를 Bean 생성 시점에 주입 가능
    Partitioning에서 Worker마다 다른 파라미터 주입 가능 (stepExecutionContext)
    테스트 시 파라미터를 쉽게 변경 가능

  단점
    Spring Context 초기화 시 실제 Bean이 생성되지 않음
    → 초기화 오류가 늦게 발견될 수 있음
    CGLIB Proxy 오버헤드 (미미하지만 존재)
    @StepScope Bean을 Step 외부에서 주입받으면 Proxy만 반환됨

ExecutionContext 크기 관리:

  장점
    DB에 영구 저장 → 재시작 가능
    모든 직렬화 가능한 객체 저장 가능

  단점
    매 Chunk 커밋마다 UPDATE → 저장하는 데이터가 클수록 I/O 부하
    SHORT_CONTEXT 길이 제한 (2500자)
    복잡한 객체 저장 시 직렬화/역직렬화 오류 위험
```

---

## 📌 핵심 정리

```
JobParameters vs ExecutionContext

  JobParameters  → 실행 입력값 (불변, JobInstance 식별, 실행 전 결정)
  ExecutionContext → 실행 중 상태 (가변, 재시작 포인트, 실행 중 생성)

@StepScope Late Binding 동작 원리
  1. Bean 정의 시: CGLIB Proxy 생성 (실제 Bean 아님)
  2. Step 실행 시: StepContext 활성화
  3. 실제 호출 시: Proxy → StepContext에서 실제 Bean 조회 또는 생성
  4. SpEL 평가: jobParameters['targetDate'] → 실제 값 주입

@StepScope 없으면 null이 되는 이유
  Spring Context 초기화 시 JobParameters가 아직 없음
  → SpEL 평가 결과 null
  → @StepScope로 평가 시점을 Step 실행 시점으로 지연해야 함

ExecutionContext 범위
  Job-scope EC  → 모든 Step이 공유 (Step 간 데이터 전달용)
  Step-scope EC → 해당 Step 내부만 (재시작 포인트 저장용)
```

---

## 🤔 생각해볼 문제

**Q1.** `@StepScope` Bean을 `@SpringBootTest`에서 테스트할 때 `No StepContext` 예외가 발생합니다. `@StepScope` Bean을 단위 테스트하는 방법은?

**Q2.** `JobParameters`가 불변임에도 불구하고 `identifying=false` 파라미터를 사용하는 이유는 무엇인가? 어떤 시나리오에서 이 기능이 유용한가?

**Q3.** 두 Step이 동시에 실행되는 Parallel Split 구성에서, 두 Step이 동시에 Job-scope `ExecutionContext`에 데이터를 저장하면 어떤 문제가 발생할 수 있는가?

> 💡 **해설**
>
> **Q1.** `@StepScope` Bean을 단위 테스트하는 방법은 두 가지입니다. (1) `StepScopeTestExecutionListener`를 등록하고 테스트 메서드에 `@SuppressWarnings`와 함께 `StepExecution`을 반환하는 메서드를 정의합니다. Spring Batch Test가 `StepContext`를 자동으로 설정해줍니다. (2) `@SpringBatchTest` 어노테이션을 사용하면 `JobLauncherTestUtils`, `JobRepositoryTestUtils`와 함께 Step scope 지원이 자동으로 설정됩니다. `TestJobExecutionContext`를 통해 원하는 파라미터를 주입할 수 있습니다.
>
> **Q2.** `identifying=false` 파라미터는 `JobInstance` 식별에서 제외하면서도 DB에 기록됩니다. 유용한 시나리오: (1) 같은 날짜(`targetDate=2024-01-01`)의 배치를 하루에 여러 번 실행해야 할 때 — `identifying=false` 타임스탬프로 실행 기록을 구분합니다. (2) 디버그 목적의 메타데이터(실행 서버명, 요청자 ID)를 파라미터에 포함하고 싶을 때 — 이 값이 `JobInstance`를 다르게 만들면 안 됩니다. (3) `RunIdIncrementer`를 쓰지 않고도 동일 `JobInstance`에 여러 번 접근하는 전략을 구현할 때입니다.
>
> **Q3.** 두 Step이 동시에 Job EC를 수정하면 Race Condition이 발생합니다. `ExecutionContext`는 `ConcurrentHashMap` 기반이므로 개별 `put()` 연산은 Thread-safe하지만, 값을 읽고 수정하는 복합 연산은 안전하지 않습니다. 더 큰 문제는 DB 저장입니다 — 두 Step이 동시에 `BATCH_JOB_EXECUTION_CONTEXT`를 UPDATE하면 마지막에 저장된 값이 이전 값을 덮어씁니다. 해결책: Parallel Step에서는 각 Step의 Step-scope EC를 사용하거나, Job EC는 읽기 전용으로만 사용합니다. 쓰기가 필요하다면 동기화 메커니즘(DB 잠금, 별도 집계 Step)을 사용합니다.

---

<div align="center">

**[⬅️ 이전: JobInstance vs JobExecution vs StepExecution](./04-job-instance-execution.md)** | **[홈으로 🏠](../README.md)** | **[다음: @EnableBatchProcessing과 Batch 자동 구성 ➡️](./06-enable-batch-processing.md)**

</div>
