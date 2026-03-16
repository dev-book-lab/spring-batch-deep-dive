# JobInstance vs JobExecution vs StepExecution — 세 개념의 관계와 재시작 메커니즘

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `JobInstance`, `JobExecution`, `StepExecution`은 각각 무엇을 나타내며 어떤 관계를 가지는가?
- 매일 실행되는 배치 Job에서 `JobInstance`는 매번 새로 생성되는가, 재사용되는가?
- 실패한 `JobExecution`을 재시작하면 새 `JobExecution`이 생성되는가, 기존 것을 사용하는가?
- `COMPLETED` 상태인 `JobInstance`를 강제로 재실행하는 방법은?
- `StepExecution`은 재시작 시 어떻게 복원되는가?

---

## 🔍 왜 세 개념을 분리하는가

### 문제: 실행 이력과 실행 결과를 구분해야 한다

```
시나리오: 매일 실행되는 정산 배치

  2024-01-01 실행 → 실패 (DB 연결 오류)
  2024-01-01 재시도 → 성공
  2024-01-02 실행 → 성공

이 이력에서 필요한 구분:

  "2024-01-01 배치 작업" (동일 데이터 처리)
    → JobInstance: 1개 (날짜가 같으므로 같은 인스턴스)

  "2024-01-01 첫 번째 실행" (실패)
  "2024-01-01 두 번째 실행" (성공)
    → JobExecution: 2개 (실행 횟수만큼 생성)

  "2024-01-01 두 번째 실행의 processStep"
    → StepExecution: 1개 (JobExecution당 Step당 1개)

개념이 합쳐지면:
  "2024-01-01 배치를 재시도했는가?" 조회 불가
  "첫 번째 실행에서 몇 건 처리했다가 실패했는가?" 조회 불가
```

---

## 😱 흔한 실수

### Before: JobInstance와 JobExecution을 같은 개념으로 혼동한다

```java
// ❌ 잘못된 이해 — "JobExecution이 FAILED면 같은 파라미터로 재실행이 안 된다"
// 실제로는:
// FAILED JobExecution → 재시작 가능 (새 JobExecution이 같은 JobInstance에 생성됨)
// COMPLETED JobExecution → 재시작 불가 (JobInstance가 완료 상태이므로)

// 재실행 시도:
JobParameters params = new JobParametersBuilder()
    .addString("targetDate", "2024-01-01")
    .toJobParameters();

// FAILED 상태에서 재실행 → ✅ 성공
// JobExecution #2가 JobInstance #1에 연결되어 생성됨
jobLauncher.run(job, params);

// COMPLETED 상태에서 재실행 → ❌ JobInstanceAlreadyCompleteException
// JobInstance #1은 이미 완료됨
```

### Before: 재시작 시 이미 완료된 Step이 다시 실행된다고 생각한다

```java
// ❌ 잘못된 이해
// "Job을 재시작하면 모든 Step이 처음부터 다시 실행된다"

// 실제:
// Job 재시작 시 COMPLETED 상태인 Step은 건너뜀
// processStep이 FAILED → 재시작 시 downloadStep(COMPLETED) 건너뛰고
//                         processStep부터 재시작

// 확인 로그:
// "Step already complete or not restartable, so no action to execute: downloadStep"
```

---

## ✨ 올바른 이해와 사용

### 세 개념의 관계

```
JobInstance (1개)
  ├── 정의: 논리적 Job 실행 단위 (JobName + JobParameters의 조합)
  ├── 생성: 새로운 JobParameters 조합 첫 실행 시
  └── 특징: 한 번 COMPLETED되면 같은 파라미터로 재생성 불가

  JobExecution (N개 per JobInstance)
    ├── 정의: 하나의 JobInstance에 대한 물리적 실행 시도
    ├── 생성: run() 호출마다 (성공·실패 모두)
    └── 상태: STARTING → STARTED → COMPLETED/FAILED/STOPPED

    StepExecution (N개 per JobExecution)
      ├── 정의: 특정 JobExecution 내에서 Step 한 번의 실행
      ├── 생성: Step이 실행될 때마다 (재시작 시 새로 생성)
      └── 포함: readCount, writeCount, commitCount, ExecutionContext
```

### 재시작 시 흐름

```
JobInstance #1 (targetDate=2024-01-01)
  │
  ├── JobExecution #1 (첫 번째 시도 → FAILED)
  │     ├── StepExecution #1: downloadStep → COMPLETED
  │     └── StepExecution #2: processStep → FAILED (50만 건 처리 중 오류)
  │
  └── JobExecution #2 (재시작 → COMPLETED)
        ├── [downloadStep 건너뜀 — COMPLETED이므로]
        └── StepExecution #3: processStep → COMPLETED (50만 1건부터 재개)
              └── ExecutionContext에서 read.count=500000 복원
```

---

## 🔬 내부 동작 원리

### 1. JobInstance 생성 로직

```java
// SimpleJobRepository.java
public JobExecution createJobExecution(String jobName, JobParameters jobParameters) {

    // ① JobName + JobParameters → MD5 해시(JOB_KEY) 계산
    JobInstance jobInstance = jobInstanceDao.getJobInstance(jobName, jobParameters);

    if (jobInstance == null) {
        // ② 신규 JobInstance 생성 (BATCH_JOB_INSTANCE INSERT)
        jobInstance = jobInstanceDao.createJobInstance(jobName, jobParameters);
        // BATCH_JOB_INSTANCE: JOB_NAME="orderSettlementJob", JOB_KEY="d41d8cd9..."

    } else {
        // ③ 기존 JobInstance 재사용 — 마지막 실행 상태 검사
        List<JobExecution> executions = jobExecutionDao.findJobExecutions(jobInstance);

        for (JobExecution execution : executions) {
            // 실행 중이면 예외
            if (execution.isRunning() || execution.isStopping()) {
                throw new JobExecutionAlreadyRunningException(...);
            }
        }

        // ④ 마지막 실행이 COMPLETED이면 재시작 불가
        JobExecution lastExecution = executions.get(0); // 최신순 정렬
        if (lastExecution.getStatus() == BatchStatus.COMPLETED) {
            throw new JobInstanceAlreadyCompleteException(...);
        }
        // FAILED, STOPPED → 재시작 허용 (새 JobExecution 생성)
    }

    // ⑤ 새 JobExecution 생성 (BATCH_JOB_EXECUTION INSERT, status=STARTING)
    JobExecution jobExecution = jobInstance.createJobExecution();
    jobExecutionDao.saveJobExecution(jobExecution);
    return jobExecution;
}
```

### 2. 재시작 시 Step 건너뛰기 로직

```java
// SimpleStepHandler.java
public StepExecution handleStep(Step step, JobExecution execution) throws JobInterruptedException {

    // ① 이전 StepExecution 조회 (가장 최근 실행)
    StepExecution lastStepExecution = jobRepository.getLastStepExecution(
        execution.getJobInstance(), step.getName());

    // ② 이미 COMPLETED인 Step은 건너뜀
    if (stepExecutionPartOfExistingJobExecution(execution, lastStepExecution)) {
        // 현재 JobExecution에 이미 이 Step의 StepExecution이 있으면 건너뜀
        log.info("Step already complete or not restartable: " + step.getName());
        return lastStepExecution;
    }

    // ③ Step 재시작 가능 여부 확인
    if (lastStepExecution != null) {
        if (!step.isAllowStartIfComplete()
                && lastStepExecution.getStatus() == BatchStatus.COMPLETED) {
            // allowStartIfComplete=false(기본값) + COMPLETED → 건너뜀
            log.info("Step already complete: " + step.getName());
            return lastStepExecution;
        }
        if (lastStepExecution.getStatus() == BatchStatus.FAILED
                || lastStepExecution.getStatus() == BatchStatus.STOPPED) {
            // FAILED/STOPPED → 재시작 (새 StepExecution 생성)
            log.info("Restarting step after FAILED/STOPPED: " + step.getName());
        }
    }

    // ④ 새 StepExecution 생성 및 실행
    StepExecution currentStepExecution = execution.createStepExecution(step.getName());
    jobRepository.add(currentStepExecution);
    step.execute(currentStepExecution);
    return currentStepExecution;
}
```

### 3. StepExecution에서 ExecutionContext 복원

```java
// AbstractStep.execute() — 재시작 시 EC 복원
public final void execute(StepExecution stepExecution) throws JobInterruptedException {
    // ① 이전 StepExecution의 ExecutionContext 로드
    ExecutionContext executionContext = stepExecution.getExecutionContext();
    // → BATCH_STEP_EXECUTION_CONTEXT에서 JSON 역직렬화

    // ② ItemStream 등록된 컴포넌트들에게 open(EC) 호출
    stream.open(executionContext);
    // → JpaPagingItemReader: read.count=500000 복원
    //   → 내부적으로 page 계산 후 500001번째 레코드부터 조회

    // ③ Tasklet 실행 ...
}

// JpaPagingItemReader.open() 복원 예시
@Override
public void open(ExecutionContext executionContext) throws ItemStreamException {
    super.open(executionContext);
    if (executionContext.containsKey(getExecutionContextKey("read.count"))) {
        // 이전 read.count 복원
        int itemCount = executionContext.getInt(getExecutionContextKey("read.count"));
        // 해당 위치까지 skip (페이지 계산)
        for (int i = 0; i < itemCount; i++) {
            read();  // 버림 (위치만 이동)
        }
        // ❗ 이 접근법은 느림 — JdbcCursorItemReader의 FETCH 방식이 더 효율적
    }
}
```

### 4. COMPLETED JobInstance 강제 재실행하는 방법

```java
// 방법 1: RunIdIncrementer로 매번 새 JobInstance 생성
@Bean
public Job orderJob() {
    return jobBuilderFactory.get("orderJob")
        .incrementer(new RunIdIncrementer())  // run.id 파라미터 자동 증가
        // → 매 실행마다 run.id=1, run.id=2, ... → 항상 새 JobInstance
        .start(processStep())
        .build();
}
// 실행:
jobLauncher.run(job, new RunIdIncrementer().getNext(new JobParameters()));

// 방법 2: 날짜/시간 파라미터로 고유성 보장
JobParameters params = new JobParametersBuilder()
    .addString("targetDate", "2024-01-01")
    .addLong("runAt", System.currentTimeMillis())  // 매번 다른 값
    .toJobParameters();

// 방법 3: JobOperator로 마지막 실행에서 파라미터 가져오기
@Autowired
private JobOperator jobOperator;

Long executionId = jobOperator.startNextInstance("orderJob");
// → JobParametersIncrementer.getNext()로 파라미터 자동 증가 후 실행
```

---

## 💻 실전 구현

### 재시작 시나리오 시뮬레이션

```java
@SpringBootTest
class RestartScenarioTest {

    @Autowired
    private JobLauncher jobLauncher;

    @Autowired
    private Job orderSettlementJob;

    @Autowired
    private JobExplorer jobExplorer;

    @Test
    void 실패_후_재시작_시_COMPLETED_Step은_건너뜀() throws Exception {
        // 1. 첫 실행 (processStep에서 강제 실패 주입)
        JobParameters params = new JobParametersBuilder()
            .addString("targetDate", "2024-01-01")
            .toJobParameters();

        JobExecution firstExecution = jobLauncher.run(orderSettlementJob, params);
        assertThat(firstExecution.getStatus()).isEqualTo(BatchStatus.FAILED);

        // 2. 실패한 Step 확인
        firstExecution.getStepExecutions().forEach(se ->
            System.out.printf("Step: %s, Status: %s, Read: %d%n",
                se.getStepName(), se.getStatus(), se.getReadCount())
        );

        // 3. 재시작 (같은 파라미터)
        JobExecution secondExecution = jobLauncher.run(orderSettlementJob, params);
        assertThat(secondExecution.getStatus()).isEqualTo(BatchStatus.COMPLETED);

        // 4. downloadStep이 건너뛰어졌는지 확인
        secondExecution.getStepExecutions().forEach(se ->
            System.out.printf("Restarted Step: %s, Status: %s%n",
                se.getStepName(), se.getStatus())
        );
        // downloadStep이 목록에 없거나 start_time이 null이어야 함
    }
}
```

---

## 📊 세 개념 비교 요약

```
┌──────────────────┬────────────────────────────┬─────────────────────────────────────┐
│ 개념              │ 생성 시점                    │ 특징                                 │
├──────────────────┼────────────────────────────┼─────────────────────────────────────┤
│ JobInstance      │ 신규 JobParameters 조합      │ 같은 파라미터 = 같은 Instance            │
│                  │ 첫 실행 시                   │ COMPLETED 후 재생성 불가                │
├──────────────────┼────────────────────────────┼─────────────────────────────────────┤
│ JobExecution     │ JobLauncher.run() 호출마다   │ 실패 후 재시작 = 새 JobExecution        │
│                  │                            │ 같은 JobInstance에 연결                │
├──────────────────┼────────────────────────────┼─────────────────────────────────────┤
│ StepExecution    │ Step 실행 시작 시             │ 재시작 = 새 StepExecution 생성         │
│                  │                            │ ExecutionContext로 이전 상태 복원       │
└──────────────────┴────────────────────────────┴─────────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
JobInstance 재사용 설계:

  장점
    "2024-01-01 정산"은 단 하나의 JobInstance → 이력 추적 명확
    중복 처리 방지 (같은 데이터를 두 번 처리할 수 없음)

  단점
    COMPLETED 후 재처리 필요 시 파라미터 변경 필요 (불편)
    RunIdIncrementer 사용 시 JobInstance가 무한 증가

재시작 시 COMPLETED Step 건너뛰기:

  장점
    이미 성공한 단계를 다시 처리하지 않아 효율적
    멱등성 없는 작업(이메일 발송 등)의 중복 실행 방지

  단점
    Step의 처리 결과가 이후 Step에 영향을 미치는 경우
    재처리가 필요한 Step을 건너뛸 수 있음
    → allowStartIfComplete(true)로 명시적 재실행 설정 필요
```

---

## 📌 핵심 정리

```
JobInstance = 논리적 배치 작업 단위
  JobName + JobParameters 조합이 고유 식별자
  COMPLETED 후 같은 파라미터로 재실행 불가

JobExecution = 물리적 실행 시도
  실패해도 새 JobExecution으로 재시작 가능
  여러 번 실패 = 여러 개의 JobExecution (모두 같은 JobInstance에 연결)

StepExecution = Step의 단일 실행 기록
  재시작 시 COMPLETED Step은 건너뜀 (allowStartIfComplete 기본값 false)
  FAILED Step은 새 StepExecution으로 재실행
  ExecutionContext에서 이전 처리 위치 복원

COMPLETED Instance 재실행 방법
  RunIdIncrementer → 항상 새 JobInstance (재처리 목적)
  identifying=false 파라미터 추가 → 고유 파라미터 없이 실행
  JobOperator.startNextInstance() → Incrementer 자동 적용
```

---

## 🤔 생각해볼 문제

**Q1.** `JobExecution #2`(재시작)에서 `downloadStep`이 건너뛰어질 때, `JobExecution #2`의 `StepExecution` 목록에는 `downloadStep`이 포함되는가? `JobExecution #2`의 `READ_COUNT`에는 이전 실행의 읽기 건수가 포함되는가?

**Q2.** 10개의 Step으로 구성된 Job에서 7번째 Step이 FAILED되어 재시작합니다. 8번째 Step에 `allowStartIfComplete(true)`가 설정되어 있다면, 재시작 시 8번째 Step은 어떻게 처리되는가?

**Q3.** `RunIdIncrementer`를 사용하면 매번 새 `JobInstance`가 생성됩니다. 하루에 여러 번 실행하는 경우 `BATCH_JOB_INSTANCE` 레코드가 빠르게 증가합니다. 이 문제를 해결하면서도 COMPLETED 재실행을 허용하는 다른 전략이 있는가?

> 💡 **해설**
>
> **Q1.** `JobExecution #2`의 `StepExecution` 목록에는 실제로 실행된(또는 재시작된) Step만 포함됩니다. 건너뛰어진 `downloadStep`은 새 `StepExecution`이 생성되지 않으므로 포함되지 않습니다. `JobExecution #2`의 집계 카운트(`READ_COUNT` 등)도 해당 실행에서 처리한 건수만 포함합니다. 전체 처리 건수를 알려면 같은 `JobInstance`의 모든 `JobExecution`을 합산해야 합니다. `JobExplorer.getJobExecutions(jobInstance)`로 조회할 수 있습니다.
>
> **Q2.** 재시작 시 Step은 `SimpleStepHandler`가 각 Step의 마지막 `StepExecution` 상태를 확인합니다. 1~6번 Step은 `COMPLETED`이므로 건너뜁니다. 7번 Step은 `FAILED`이므로 재실행합니다. 7번 Step이 성공하면 8번 Step을 실행합니다. `allowStartIfComplete(true)`는 "이 Step이 이전에 COMPLETED였더라도 다시 실행하라"는 의미입니다. 8번 Step은 이번 재시작에서는 아직 실행된 적이 없으므로(`COMPLETED` 상태가 없으므로) 이 설정과 무관하게 정상 실행됩니다.
>
> **Q3.** 대안 전략들: (1) `JobParametersIncrementer`를 직접 구현해 날짜 기반 파라미터를 증가시킵니다 — 같은 날짜 내에서는 카운터를 올리고, 날짜가 바뀌면 카운터를 리셋합니다. (2) `identifying=false`인 타임스탬프 파라미터를 추가합니다 — `JOB_KEY` 계산에서 제외되므로 `JobInstance`는 하나지만, `identifying=true` 파라미터가 `COMPLETED`면 여전히 재실행 불가 문제가 남습니다. (3) 근본적으로 "하루 한 번만 실행해야 하는가?"를 재검토합니다 — 정말 재처리가 필요하다면 별도 재처리 Job(`reprocess-orderSettlementJob`)을 만들어 운영하는 것이 이력 추적 측면에서 더 명확합니다.

---

<div align="center">

**[⬅️ 이전: JobRepository와 메타데이터 관리](./03-job-repository-metadata.md)** | **[홈으로 🏠](../README.md)** | **[다음: JobParameters와 실행 컨텍스트 ➡️](./05-job-parameters-context.md)**

</div>
