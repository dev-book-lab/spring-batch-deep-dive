# Job·Step·Tasklet 관계와 구조 — Spring Batch 실행 단위 계층

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Job`, `Step`, `Tasklet`은 각각 무엇을 책임지며, 세 계층으로 분리된 이유는?
- `Tasklet`과 `ChunkOrientedTasklet`의 차이는 무엇이고 언제 각각을 선택해야 하는가?
- `SimpleJob.execute()`는 `Step`들을 어떻게 순서대로 실행하는가?
- `StepContribution`과 `ChunkContext`는 실행 중 어떤 정보를 담고 있는가?
- 단순 파일 이동 작업과 100만 건 처리 작업을 같은 `Tasklet`으로 구현하면 안 되는 이유는?

---

## 🔍 왜 이 계층이 존재하는가

### 문제: 배치 작업은 구조가 다양하고 복잡도 차이가 크다

```
배치 작업의 다양성:

  단순 작업 (단일 동작)
    - 파일 이동: /tmp/input.csv → /archive/20240101.csv
    - DB 테이블 Truncate
    - FTP 파일 다운로드

  복잡한 작업 (대용량 처리)
    - 100만 건 주문 데이터 → 정산 계산 → DB 저장
    - 고객 테이블 전체 → 등급 재계산 → 업데이트
    - 로그 파일 파싱 → 집계 → 리포트 생성

  복합 작업 (여러 단계 조합)
    - FTP 다운로드 → CSV 파싱 → DB 적재 → 결과 이메일 발송

이걸 하나의 클래스로 구현하면:
  if (type == SIMPLE) { moveFile(); }
  else if (type == CHUNK) { while(read()) { process(); write(); } }
  // → 작업 종류마다 코드 폭발
  // → 재시작/복구 로직을 모든 타입에 중복 구현해야 함
  // → 실패 지점 추적이 불가능

해결: 3계층 분리
  Job    = 전체 배치 작업의 정의와 메타데이터
  Step   = 독립적으로 실행·실패·재시작 가능한 단위
  Tasklet = Step이 실제로 수행하는 작업 (단순 or Chunk 기반)
```

---

## 😱 흔한 실수

### Before: 모든 배치 작업을 단일 Tasklet으로 구현한다

```java
// ❌ 잘못된 패턴 — 100만 건 처리를 단순 Tasklet으로
@Bean
public Step processAllOrdersStep() {
    return stepBuilderFactory.get("processAllOrdersStep")
        .tasklet((contribution, chunkContext) -> {
            // 전체 주문 조회 (100만 건이 메모리에 올라옴!)
            List<Order> allOrders = orderRepository.findAll();
            for (Order order : allOrders) {
                Order processed = orderProcessor.process(order);
                orderRepository.save(processed);
                // 단 하나의 트랜잭션 — 중간 실패 시 전체 롤백
                // OOM 발생 가능
            }
            return RepeatStatus.FINISHED;
        })
        .build();
}
```

```
문제점:
  1. 메모리: 100만 건이 한 번에 힙에 올라옴 → OOM
  2. 트랜잭션: 단일 트랜잭션 → 1건 실패 시 100만 건 전체 롤백
  3. 재시작: 실패 지점을 알 수 없어 처음부터 재처리 필요
  4. 성능: JDBC 배치 없이 건당 INSERT → 극도로 느림
```

### Before: 여러 Step을 하나의 Step으로 합친다

```java
// ❌ 단일 Step에 모든 로직을 구겨넣기
@Bean
public Step megaStep() {
    return stepBuilderFactory.get("megaStep")
        .tasklet((contribution, chunkContext) -> {
            downloadFile();     // FTP 다운로드
            parseAndSave();     // 파싱 + DB 저장 (100만 건)
            sendEmail();        // 결과 이메일
            return RepeatStatus.FINISHED;
        })
        .build();
}
// → 파싱 단계 실패 시 다운로드부터 다시 시작해야 함
// → 각 단계의 실행 상태를 독립적으로 추적 불가
```

---

## ✨ 올바른 이해와 사용

### After: 작업 특성에 따라 Tasklet vs Chunk 선택

```java
// ✅ 단순 작업 → 일반 Tasklet
@Bean
public Step downloadFileStep() {
    return stepBuilderFactory.get("downloadFileStep")
        .tasklet(new FtpDownloadTasklet(ftpClient))
        // 파일 1개 다운로드 = 단일 동작, Chunk 불필요
        .build();
}

// ✅ 대용량 처리 → ChunkOrientedTasklet (chunk())
@Bean
public Step processOrdersStep() {
    return stepBuilderFactory.get("processOrdersStep")
        .<Order, SettledOrder>chunk(1000)          // 1000건씩 트랜잭션 커밋
        .reader(orderItemReader())
        .processor(orderItemProcessor())
        .writer(settledOrderItemWriter())
        .build();
}

// ✅ 여러 Step을 독립적으로 분리
@Bean
public Job orderSettlementJob() {
    return jobBuilderFactory.get("orderSettlementJob")
        .start(downloadFileStep())      // Step 1: 실패해도 다음 실행 시 이 Step부터 재시작
        .next(processOrdersStep())      // Step 2: Chunk 기반 100만 건 처리
        .next(sendReportStep())         // Step 3: 앞 단계 성공 시에만 실행
        .build();
}
```

---

## 🔬 내부 동작 원리

### 1. Job 계층 구조 — 클래스 다이어그램

```
Job (interface)
  └── AbstractJob
        ├── SimpleJob          ← Step을 순서대로 실행
        └── FlowJob            ← 조건부 분기 실행

Step (interface)
  └── AbstractStep
        ├── TaskletStep        ← Tasklet을 실행하는 Step (모든 Step의 실제 구현)
        └── PartitionStep      ← Partitioning용

Tasklet (interface)
  ├── ChunkOrientedTasklet    ← chunk() 빌더가 생성하는 Tasklet
  └── [사용자 구현 Tasklet]    ← tasklet() 빌더에 전달하는 람다/구현체
```

### 2. SimpleJob.execute() — Step 순차 실행

```java
// SimpleJob.java
public class SimpleJob extends AbstractJob {

    private List<Step> steps = new ArrayList<>();  // 등록된 Step 목록

    @Override
    protected void doExecute(JobExecution execution) throws JobInterruptedException {
        StepExecution stepExecution = null;

        for (Step step : steps) {
            // ① 이전 Step 결과가 STOPPED이면 중단
            if (stepExecution != null && stepExecution.getStatus() == BatchStatus.STOPPED) {
                execution.setStatus(BatchStatus.STOPPED);
                execution.setExitStatus(ExitStatus.STOPPED);
                return;
            }

            // ② Step 실행 위임 (StepHandler가 JobRepository와 연동)
            stepExecution = handleStep(step, execution);

            // ③ Step이 FAILED이면 Job도 FAILED로 종료
            if (stepExecution.getStatus() == BatchStatus.FAILED) {
                execution.setStatus(BatchStatus.FAILED);
                execution.setExitStatus(stepExecution.getExitStatus());
                return;
            }
        }

        // ④ 모든 Step 성공 → Job COMPLETED
        upgradeStatus(execution, BatchStatus.COMPLETED);
        execution.setExitStatus(ExitStatus.COMPLETED);
    }
}
```

### 3. TaskletStep.execute() — Tasklet 반복 호출

```java
// TaskletStep.java
public class TaskletStep extends AbstractStep {

    private Tasklet tasklet;   // ChunkOrientedTasklet 또는 사용자 구현체

    @Override
    protected void doExecute(StepExecution stepExecution) throws Exception {
        // 트랜잭션 템플릿으로 각 Tasklet 호출 래핑
        TransactionTemplate transactionTemplate = new TransactionTemplate(transactionManager);

        RepeatStatus status;
        do {
            // 트랜잭션 시작
            status = transactionTemplate.execute(txStatus -> {
                StepContribution contribution = stepExecution.createStepContribution();
                ChunkContext chunkContext = new ChunkContext(
                    new StepContext(stepExecution));

                // ① Tasklet 실행 (Chunk 처리 or 단순 작업)
                RepeatStatus result = tasklet.execute(contribution, chunkContext);

                // ② StepExecution에 처리 결과 반영
                contribution.incrementCommitCount();
                stepExecution.apply(contribution);

                // ③ 상태 저장 (JobRepository)
                getJobRepository().update(stepExecution);

                return result;
            });

        } while (RepeatStatus.CONTINUABLE.equals(status));
        // RepeatStatus.FINISHED 반환 시 루프 종료
    }
}
```

### 4. Tasklet.execute() 반환값의 의미

```java
// Tasklet (interface)
public interface Tasklet {
    /**
     * @return CONTINUABLE: 아직 처리할 것이 남아 있음 → 다시 호출
     *         FINISHED:    처리 완료 → Step 종료
     */
    @Nullable
    RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext)
        throws Exception;
}

// ChunkOrientedTasklet은 언제 FINISHED를 반환하는가?
// → ItemReader.read()가 null을 반환할 때 (더 읽을 데이터 없음)
// → 사용자 Tasklet은 직접 제어
```

### 5. StepContribution과 ChunkContext

```java
// StepContribution — 현재 Tasklet 실행에서의 처리 카운트
public class StepContribution {
    private volatile long readCount = 0;       // 읽은 건수
    private volatile long writeCount = 0;      // 쓴 건수
    private volatile long filterCount = 0;     // 필터링된 건수 (Processor null 반환)
    private volatile long readSkipCount = 0;   // 읽기 Skip 건수
    private volatile long writeSkipCount = 0;  // 쓰기 Skip 건수
    private ExitStatus exitStatus = ExitStatus.EXECUTING;

    // Tasklet 실행 완료 후 StepExecution.apply()로 누적됨
}

// ChunkContext — Step과 Job 전체의 실행 컨텍스트 접근
public class ChunkContext {
    private final StepContext stepContext;
    private boolean complete = false;

    public Map<String, Object> getAttribute(String name) {
        return stepContext.getAttribute(name);  // ExecutionContext 접근
    }

    // 예: 재시작 시 이전 진행 상황 확인
    public ExecutionContext getStepExecutionContext() {
        return stepContext.getStepExecution().getExecutionContext();
    }
}
```

### 6. 전체 실행 흐름 ASCII 다이어그램

```
JobLauncher.run(job, params)
    │
    ▼
SimpleJob.execute(jobExecution)
    │
    ├─ Step 1: downloadFileStep (TaskletStep)
    │      │
    │      └─ FtpDownloadTasklet.execute()
    │             → RepeatStatus.FINISHED
    │             → StepExecution: COMPLETED
    │
    ├─ Step 2: processOrdersStep (TaskletStep)
    │      │
    │      └─ ChunkOrientedTasklet.execute()  ←── 반복 (CONTINUABLE)
    │             │
    │             ├─ Read  1000건 (ItemReader)
    │             ├─ Process 1000건 (ItemProcessor)
    │             ├─ Write  1000건 (ItemWriter)
    │             └─ Commit ──── 다시 CONTINUABLE
    │             ...
    │             └─ Read  → null → RepeatStatus.FINISHED
    │             → StepExecution: COMPLETED
    │
    └─ Step 3: sendReportStep (TaskletStep)
           │
           └─ EmailTasklet.execute()
                  → RepeatStatus.FINISHED
                  → StepExecution: COMPLETED

JobExecution: COMPLETED
```

---

## 💻 실전 구현

### 재시작 가능한 파일 이동 Tasklet

```java
@Component
@StepScope
public class FileMoveTasklet implements Tasklet {

    @Value("#{jobParameters['inputFile']}")
    private String inputFilePath;

    @Value("#{jobParameters['archiveDir']}")
    private String archiveDir;

    @Override
    public RepeatStatus execute(StepContribution contribution,
                                ChunkContext chunkContext) throws Exception {
        File input = new File(inputFilePath);

        if (!input.exists()) {
            // 이미 이동됐을 수 있음 (재시작 시나리오)
            log.info("파일 없음, 이미 처리된 것으로 간주: {}", inputFilePath);
            return RepeatStatus.FINISHED;
        }

        File archive = new File(archiveDir, input.getName());
        boolean moved = input.renameTo(archive);

        if (!moved) {
            throw new ItemStreamException("파일 이동 실패: " + inputFilePath);
        }

        log.info("파일 이동 완료: {} → {}", inputFilePath, archive.getAbsolutePath());
        contribution.incrementWriteCount(1);  // 처리 카운트 반영
        return RepeatStatus.FINISHED;
    }
}

@Configuration
public class FileBatchConfig {

    @Bean
    public Step fileMoveStep(StepBuilderFactory stepBuilderFactory,
                              FileMoveTasklet fileMoveTasklet) {
        return stepBuilderFactory.get("fileMoveStep")
            .tasklet(fileMoveTasklet)
            .allowStartIfComplete(true)  // COMPLETED여도 재실행 허용
            .build();
    }
}
```

### Chunk 기반 주문 처리 Job 전체 구조

```java
@Configuration
@EnableBatchProcessing
public class OrderSettlementJobConfig {

    @Bean
    public Job orderSettlementJob(JobBuilderFactory jobBuilderFactory,
                                   Step downloadStep,
                                   Step processStep,
                                   Step reportStep) {
        return jobBuilderFactory.get("orderSettlementJob")
            .incrementer(new RunIdIncrementer())  // 매 실행마다 run.id 증가
            .start(downloadStep)
            .next(processStep)
            .next(reportStep)
            .build();
    }

    @Bean
    public Step processStep(StepBuilderFactory stepBuilderFactory,
                             ItemReader<Order> reader,
                             ItemProcessor<Order, SettledOrder> processor,
                             ItemWriter<SettledOrder> writer) {
        return stepBuilderFactory.get("processStep")
            .<Order, SettledOrder>chunk(1000)    // 1000건 = 1 트랜잭션
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .build();
    }
}
```

---

## 📊 Tasklet vs Chunk 선택 기준

```
작업 유형별 적합한 방식:

┌─────────────────────────────┬──────────────┬──────────────────────────────┐
│ 작업 유형                     │ 방식          │ 이유                          │
├─────────────────────────────┼──────────────┼──────────────────────────────┤
│ 파일 이동/삭제                 │ Tasklet      │ 단일 동작, 반복 불필요             │
│ FTP 다운로드                  │ Tasklet      │ 파일 단위 처리                   │
│ 테이블 Truncate              │ Tasklet      │ 단일 쿼리                       │
│ 외부 API 단건 호출             │ Tasklet      │ 건수 고정                       │
├─────────────────────────────┼──────────────┼──────────────────────────────┤
│ 대용량 DB 읽기/쓰기             │ Chunk        │ 메모리 제어, 트랜잭션 분리         │
│ CSV/Excel 파일 파싱 → 저장      │ Chunk        │ 건단위 처리, 실패 복구           │
│ 전체 테이블 데이터 변환           │ Chunk        │ OOM 방지                     │
│ 외부 API 대량 호출              │ Chunk        │ 배치 크기 조절 가능             │
└─────────────────────────────┴──────────────┴──────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
3계층 분리(Job/Step/Tasklet)의 트레이드오프:

  장점
    각 Step이 독립적으로 실패·재시작 가능
    Step 단위로 실행 상태 추적 (JobRepository)
    단순 작업과 복잡한 처리를 같은 Job 안에서 혼용 가능
    Step 재사용 — 여러 Job에서 동일 Step 공유

  단점
    설정 코드가 많음 (JobBuilder, StepBuilder, ItemReader/Processor/Writer 각각 Bean 등록)
    작은 배치 작업에도 동일한 보일러플레이트 필요
    Spring Context 의존 → 단독 실행 테스트가 복잡

ChunkOrientedTasklet의 트레이드오프:

  장점
    Chunk 크기 = 트랜잭션 경계 → 메모리와 커밋 빈도를 독립 조절
    Read/Process/Write 각 단계 교체 가능 (전략 패턴)
    Skip/Retry 적용 지점이 Item 단위로 세분화

  단점
    단순 작업에는 과도한 추상화
    ItemReader 상태(커서 위치 등)를 직접 관리해야 재시작 가능
```

---

## 📌 핵심 정리

```
3계층 실행 단위

  Job     → 전체 배치 작업 정의, Step 목록 관리, 메타데이터(JobExecution) 생성
  Step    → 독립 실행 단위, StepExecution 생성, 트랜잭션 관리 위임
  Tasklet → 실제 작업 수행, RepeatStatus로 반복 여부 신호

Tasklet 종류 선택 기준

  단순 Tasklet  → 파일 처리, 단일 쿼리, 단순 API 호출 (반복 없는 단발성 작업)
  ChunkOrientedTasklet → 대용량 데이터 처리 (Read→Process→Write 반복, 트랜잭션 분리 필요)

SimpleJob Step 실행 규칙

  앞 Step STOPPED  → Job STOPPED, 남은 Step 실행 안 함
  앞 Step FAILED   → Job FAILED, 남은 Step 실행 안 함
  앞 Step COMPLETED → 다음 Step 실행

RepeatStatus

  CONTINUABLE → Tasklet을 다시 호출 (더 처리할 것 있음)
  FINISHED    → Step 종료 (완료)
```

---

## 🤔 생각해볼 문제

**Q1.** `TaskletStep`에서 `Tasklet.execute()`가 예외를 던졌을 때 `StepExecution`의 상태는 어떻게 되는가? 그리고 이 예외가 트랜잭션에 미치는 영향은?

**Q2.** `allowStartIfComplete(true)`를 설정하지 않은 Step이 `COMPLETED` 상태일 때, 해당 Job을 재시작하면 이 Step은 어떻게 되는가?

**Q3.** 하나의 `Step`을 두 개의 다른 `Job`이 공유할 수 있는가? 이 경우 `StepExecution`은 어떻게 관리되는가?

> 💡 **해설**
>
> **Q1.** `Tasklet.execute()`에서 예외가 발생하면 `transactionTemplate.execute()` 내에서 예외가 전파되므로 트랜잭션이 롤백됩니다. `TaskletStep`은 예외를 catch해서 `StepExecution.status = FAILED`, `exitStatus = FAILED`로 설정하고 `JobRepository.update(stepExecution)`을 호출해 실패 상태를 저장합니다. 이후 `SimpleJob`이 Step FAILED를 감지해 Job도 FAILED로 종료합니다.
>
> **Q2.** `allowStartIfComplete(false)`가 기본값입니다. 재시작 시 `COMPLETED` 상태인 Step은 `SimpleStepHandler`가 건너뜁니다(`shouldStart()` 반환 false). 즉, 이미 성공한 Step은 재실행하지 않고 다음 Step으로 넘어갑니다. 멱등성이 보장되는 Step(예: 파일 이동)이라면 `allowStartIfComplete(true)`로 재실행을 허용하는 것이 안전합니다.
>
> **Q3.** `Step` Bean은 공유할 수 있지만, `StepExecution`은 각 `JobExecution`마다 독립적으로 생성됩니다. `JobRepository`에서 `StepExecution`은 `(jobExecutionId, stepName)`으로 식별되므로 Job A의 실행과 Job B의 실행은 별도의 `StepExecution`을 가집니다. 단, 같은 `Step` Bean을 두 Job이 동시에 실행할 경우 `ItemReader`에 상태(커서 위치 등)가 있다면 Thread-safety 문제가 발생할 수 있습니다.

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: JobLauncher와 Job 실행 메커니즘 ➡️](./02-job-launcher-execution.md)**

</div>
