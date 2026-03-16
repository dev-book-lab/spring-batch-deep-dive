# JobLauncher와 Job 실행 메커니즘 — run()에서 execute()까지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `JobLauncher.run(job, parameters)`를 호출했을 때 내부에서 정확히 어떤 순서로 무슨 일이 일어나는가?
- `SimpleJobLauncher`는 `JobExecution`을 어떻게 생성하고 `Job.execute()`를 어떻게 위임하는가?
- 동기 실행과 비동기 실행의 차이는 무엇이고, 어떤 `TaskExecutor`를 주입하면 달라지는가?
- 이미 실행 중인 같은 `Job`을 재실행하면 어떤 예외가 발생하는가?
- Spring Boot에서 `CommandLineRunner`로 Job이 자동 실행되는 경로는?

---

## 🔍 왜 JobLauncher가 별도로 존재하는가

### 문제: Job 실행 진입점이 하나여야 한다

```
Job을 실행하는 방법은 여러 가지다:

  1. Spring Boot 애플리케이션 시작 시 자동 실행
  2. REST API 요청으로 실행 (/jobs/{jobName}/run)
  3. Scheduler (Spring Scheduler, Quartz)
  4. 메시지 큐 이벤트 (Kafka, RabbitMQ)
  5. 명령줄 직접 실행 (CommandLineJobRunner)

각 진입점마다 다음을 공통으로 처리해야 한다:
  - 중복 실행 방지 (같은 JobInstance가 이미 실행 중인지 확인)
  - JobExecution 생성 및 JobRepository에 저장
  - 비동기 실행 지원
  - 실행 전 JobParameters 유효성 확인

JobLauncher가 없다면:
  → 각 진입점이 위 로직을 중복 구현
  → 중복 실행 체크를 빠뜨리면 같은 JobInstance가 두 번 실행됨
  → JobRepository 연동이 각 진입점에 분산됨
```

---

## 😱 흔한 실수

### Before: JobLauncher를 우회해서 Job을 직접 호출한다

```java
// ❌ Job.execute()를 직접 호출 — 절대 하지 말 것
@RestController
public class BadBatchController {

    @Autowired
    private Job myJob;

    @Autowired
    private JobRepository jobRepository;

    @PostMapping("/run-job")
    public void runJob() throws Exception {
        // JobExecution을 수동으로 만들어서 직접 실행 시도
        JobExecution execution = jobRepository.createJobExecution(
            "myJob", new JobParameters());
        myJob.execute(execution);  // ❌ 중복 실행 체크 없음!
    }
}
// 문제: 동시에 두 요청이 오면 같은 JobInstance가 두 번 실행됨
// JobRepository 상태 관리가 엉킴
```

### Before: 비동기 실행 시 HTTP 응답을 기다린다

```java
// ❌ 동기 JobLauncher로 오래 걸리는 Job을 HTTP 요청에서 실행
@PostMapping("/run")
public ResponseEntity<String> runJob() throws Exception {
    // SyncTaskExecutor(기본값) → HTTP 요청 스레드에서 Job 실행
    // Job이 1시간 걸리면 HTTP 응답이 1시간 후에 옴 → 타임아웃!
    jobLauncher.run(myJob, new JobParameters());
    return ResponseEntity.ok("완료");
}
```

---

## ✨ 올바른 이해와 사용

### After: SimpleJobLauncher.run() 전체 흐름

```java
// SimpleJobLauncher.java — 핵심 로직
@Override
public JobExecution run(final Job job, final JobParameters jobParameters)
        throws JobExecutionAlreadyRunningException,
               JobRestartException,
               JobInstanceAlreadyCompleteException,
               JobParametersInvalidException {

    Assert.notNull(job, "The Job must not be null.");
    Assert.notNull(jobParameters, "The JobParameters must not be null.");

    // ① JobParameters 유효성 검증
    job.getJobParametersValidator().validate(jobParameters);

    // ② JobRepository에서 마지막 JobExecution 조회
    JobExecution lastExecution = jobRepository.getLastJobExecution(
        job.getName(), jobParameters);

    // ③ 실행 가능 여부 확인 (중복 실행 방지)
    if (lastExecution != null) {
        if (!job.isRestartable()) {
            throw new JobRestartException("...");
        }
        // 이미 실행 중인 상태면 예외
        for (String status : BatchStatus.getRunningStatuses()) {
            if (lastExecution.getStatus().matches(status)) {
                throw new JobExecutionAlreadyRunningException("...");
            }
        }
        // COMPLETED 상태면 재시작 불가 (JobInstance 재사용 불가)
        if (lastExecution.getStatus() == BatchStatus.COMPLETED) {
            throw new JobInstanceAlreadyCompleteException("...");
        }
    }

    // ④ 새로운 JobExecution 생성 + JobRepository에 저장 (STARTING 상태)
    final JobExecution jobExecution = jobRepository.createJobExecution(
        job.getName(), jobParameters);

    // ⑤ TaskExecutor로 실행 (동기 or 비동기)
    try {
        taskExecutor.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    // 실제 Job 실행
                    job.execute(jobExecution);
                } catch (Throwable t) {
                    log.error("Job: [" + job + "] failed unexpectedly.", t);
                }
            }
        });
    } catch (TaskRejectedException e) {
        jobExecution.upgradeStatus(BatchStatus.FAILED);
        jobRepository.update(jobExecution);
    }

    // ⑥ JobExecution 반환 (동기: 완료 후 반환 / 비동기: STARTING 상태로 즉시 반환)
    return jobExecution;
}
```

### 동기 vs 비동기 실행 설정

```java
// 동기 실행 (기본값) — SyncTaskExecutor
@Bean
public JobLauncher syncJobLauncher(JobRepository jobRepository) {
    SimpleJobLauncher launcher = new SimpleJobLauncher();
    launcher.setJobRepository(jobRepository);
    launcher.setTaskExecutor(new SyncTaskExecutor());  // 기본값
    // → run() 호출 스레드에서 Job 실행, Job 완료 후 run() 반환
    return launcher;
}

// 비동기 실행 — SimpleAsyncTaskExecutor
@Bean
public JobLauncher asyncJobLauncher(JobRepository jobRepository) {
    SimpleJobLauncher launcher = new SimpleJobLauncher();
    launcher.setJobRepository(jobRepository);
    launcher.setTaskExecutor(new SimpleAsyncTaskExecutor());
    // → 별도 스레드에서 Job 실행, run()은 STARTING 상태의 JobExecution 즉시 반환
    return launcher;
}
```

---

## 🔬 내부 동작 원리

### 1. JobExecution 생성 — JobRepository와의 협력

```java
// JobRepositoryFactoryBean이 생성한 SimpleJobRepository.createJobExecution()

public JobExecution createJobExecution(String jobName, JobParameters jobParameters)
        throws JobExecutionAlreadyRunningException,
               JobRestartException,
               JobInstanceAlreadyCompleteException {

    Assert.notNull(jobName, "Job name must not be null.");
    Assert.notNull(jobParameters, "JobParameters must not be null.");

    // ① BATCH_JOB_INSTANCE에서 기존 JobInstance 조회
    JobInstance jobInstance = jobInstanceDao.getJobInstance(jobName, jobParameters);

    // ② JobInstance가 없으면 새로 생성
    if (jobInstance == null) {
        jobInstance = jobInstanceDao.createJobInstance(jobName, jobParameters);
    } else {
        // ③ 기존 JobInstance의 마지막 실행 상태 확인
        List<JobExecution> executions = jobExecutionDao.findJobExecutions(jobInstance);
        for (JobExecution execution : executions) {
            if (execution.isRunning() || execution.isStopping()) {
                throw new JobExecutionAlreadyRunningException(
                    "A job execution for this job is already running: " + jobInstance);
            }
            // COMPLETED 상태면 재실행 불가
            BatchStatus status = execution.getStatus();
            if (status == BatchStatus.UNKNOWN) {
                throw new JobRestartException("Cannot restart job from UNKNOWN status");
            }
            if (status.isUnsuccessful()) {
                // FAILED, STOPPED → 재시작 가능
                continue;
            }
            throw new JobInstanceAlreadyCompleteException(
                "A job instance already exists and is complete for parameters=" + jobParameters);
        }
    }

    // ④ 새 JobExecution 생성 및 DB 저장 (status=STARTING)
    JobExecution jobExecution = jobInstance.createJobExecution();
    jobExecution.setJobParameters(jobParameters);
    jobExecutionDao.saveJobExecution(jobExecution);

    return jobExecution;
}
```

### 2. Job.execute() → AbstractJob.execute()

```java
// AbstractJob.java
public final void execute(JobExecution execution) {
    Assert.notNull(execution, "jobExecution must not be null");

    // ① 실행 시작 상태로 변경 (STARTING → STARTED)
    execution.setStartTime(new Date());
    execution.setStatus(BatchStatus.STARTED);
    getJobRepository().update(execution);  // DB 업데이트

    // ② JobExecutionListener.beforeJob() 호출
    listener.beforeJob(execution);

    try {
        // ③ 실제 실행 위임 (SimpleJob 또는 FlowJob)
        doExecute(execution);

        // ④ Step 실행 결과에 따라 ExitStatus 최종 결정
        ExitStatus exitStatus = execution.getExitStatus();
        if (!exitStatus.getExitCode().equals(ExitStatus.STOPPED.getExitCode())) {
            execution.upgradeStatus(BatchStatus.COMPLETED);
            execution.setExitStatus(exitStatus.and(ExitStatus.COMPLETED));
        }
    } catch (JobInterruptedException e) {
        // ⑤ 인터럽트 처리
        execution.setStatus(BatchStatus.STOPPED);
        execution.setExitStatus(ExitStatus.STOPPED);
    } catch (Throwable t) {
        // ⑥ 예외 처리
        execution.upgradeStatus(BatchStatus.FAILED);
        execution.setExitStatus(execution.getExitStatus().and(ExitStatus.FAILED));
    } finally {
        // ⑦ 종료 시간 + 상태 저장
        execution.setEndTime(new Date());
        getJobRepository().update(execution);

        // ⑧ JobExecutionListener.afterJob() 호출
        listener.afterJob(execution);
    }
}
```

### 3. Spring Boot 자동 실행 경로

```java
// Spring Boot — JobLauncherApplicationRunner
// spring.batch.job.enabled=true (기본값) 일 때 자동 등록

public class JobLauncherApplicationRunner
        implements ApplicationRunner, Ordered, ApplicationEventPublisherAware {

    @Override
    public void run(ApplicationArguments args) throws Exception {
        // 등록된 모든 Job Bean을 찾아서 실행
        // (spring.batch.job.name으로 특정 Job만 실행 가능)
        for (Job job : this.jobs) {
            JobParameters jobParameters = getJobParameters(job, args);
            execute(job, jobParameters);
        }
    }

    private void execute(Job job, JobParameters jobParameters) throws Exception {
        JobExecution execution = this.jobLauncher.run(job, jobParameters);
        // ApplicationEvent 발행 → 모니터링, 알림 연동 가능
        this.publisher.publishEvent(new JobExecutionEvent(execution));
    }
}
```

### 4. 전체 실행 흐름 ASCII 다이어그램

```
[진입점] Spring Boot 시작 / REST API / Scheduler
    │
    ▼
JobLauncher.run(job, jobParameters)
    │
    ├─ ① job.getJobParametersValidator().validate()
    │       ↓ 유효하지 않으면 JobParametersInvalidException
    │
    ├─ ② jobRepository.getLastJobExecution()
    │       ↓ 이미 RUNNING → JobExecutionAlreadyRunningException
    │       ↓ COMPLETED   → JobInstanceAlreadyCompleteException
    │       ↓ FAILED/STOPPED → 재시작 허용
    │
    ├─ ③ jobRepository.createJobExecution()
    │       ↓ BATCH_JOB_INSTANCE (없으면 INSERT)
    │       ↓ BATCH_JOB_EXECUTION INSERT (status=STARTING)
    │       ↓ BATCH_JOB_EXECUTION_PARAMS INSERT
    │
    └─ ④ taskExecutor.execute(job::execute)
            │
            ▼ [동기: 현재 스레드] / [비동기: 새 스레드]
            │
            AbstractJob.execute(jobExecution)
                │
                ├─ status: STARTING → STARTED (DB update)
                ├─ listener.beforeJob()
                ├─ SimpleJob.doExecute() → Step 실행 (01 문서 참고)
                ├─ status → COMPLETED / FAILED / STOPPED (DB update)
                └─ listener.afterJob()

[반환] JobExecution
  동기:  Job 완료 상태의 JobExecution (COMPLETED/FAILED)
  비동기: STARTING 상태의 JobExecution (즉시 반환, 완료 전)
```

---

## 💻 실전 구현

### REST API로 Job 비동기 실행 + 상태 조회

```java
@RestController
@RequestMapping("/api/batch")
public class BatchController {

    @Autowired
    @Qualifier("asyncJobLauncher")  // 비동기 Launcher
    private JobLauncher asyncJobLauncher;

    @Autowired
    private Job orderSettlementJob;

    @Autowired
    private JobExplorer jobExplorer;

    // Job 실행 요청
    @PostMapping("/jobs/order-settlement")
    public ResponseEntity<BatchJobResponse> runJob(
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate targetDate) {
        try {
            JobParameters params = new JobParametersBuilder()
                .addString("targetDate", targetDate.toString())
                .addLong("timestamp", System.currentTimeMillis())  // 중복 실행 방지
                .toJobParameters();

            // 비동기 실행 → 즉시 JobExecution 반환 (status=STARTING)
            JobExecution execution = asyncJobLauncher.run(orderSettlementJob, params);

            return ResponseEntity.accepted()
                .body(new BatchJobResponse(
                    execution.getId(),
                    execution.getStatus().name()
                ));
        } catch (JobExecutionAlreadyRunningException e) {
            return ResponseEntity.status(409)
                .body(new BatchJobResponse(null, "ALREADY_RUNNING"));
        } catch (Exception e) {
            return ResponseEntity.internalServerError()
                .body(new BatchJobResponse(null, "LAUNCH_FAILED"));
        }
    }

    // 실행 상태 조회
    @GetMapping("/jobs/executions/{executionId}")
    public BatchJobResponse getStatus(@PathVariable Long executionId) {
        JobExecution execution = jobExplorer.getJobExecution(executionId);
        if (execution == null) {
            throw new ResponseStatusException(HttpStatus.NOT_FOUND);
        }
        return new BatchJobResponse(executionId, execution.getStatus().name());
    }
}
```

### 특정 상황에서만 Job 재실행 허용 설정

```java
@Bean
public Job retryableJob() {
    return jobBuilderFactory.get("retryableJob")
        .preventRestart()              // 재시작 완전 금지
        // .restartable(true)          // 기본값 — FAILED/STOPPED 시 재시작 허용
        .start(processStep())
        .build();
}
```

---

## 📊 동기 vs 비동기 실행 비교

```
┌────────────────────┬──────────────────────────┬──────────────────────────────┐
│ 항목                │ 동기 (SyncTaskExecutor)   │비동기 (SimpleAsyncTaskExecutor)│
├────────────────────┼──────────────────────────┼──────────────────────────────┤
│ run() 반환 시점      │ Job 완료 후                │ 즉시 (STARTING 상태)           │
│ 반환 JobExecution   │ 최종 상태 포함               │ STARTING 상태                 │
│ HTTP 타임아웃 위험    │ 있음 (Job이 오래 걸리면)      │ 없음                          │
│ 예외 전파            │ run() 호출자에게 전파        │ 내부 스레드에서 처리              │
│ 사용 시나리오         │ CLI, 테스트, 단순 스케줄      │ REST API, UI 기반 실행         │
├────────────────────┼──────────────────────────┼──────────────────────────────┤
│ Spring Boot 기본    │ ✅ 기본값                  │ 별도 Bean 등록 필요             │
└────────────────────┴──────────────────────────┴──────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
JobLauncher 중복 실행 방지 메커니즘:

  장점
    JobParameters + JobName 조합으로 자동 중복 체크
    실수로 같은 Job을 두 번 실행하는 것을 방지
    클러스터 환경에서도 DB 기반으로 단일 체크 포인트

  단점
    매 실행마다 DB 조회 발생 (lastExecution 조회)
    COMPLETED 상태 JobInstance는 재실행 불가
    → JobParametersIncrementer로 새 JobInstance 생성해야 함

비동기 실행:

  장점
    HTTP 타임아웃 없음, UI 응답성 유지
    여러 Job 동시 실행 가능

  단점
    실행 상태를 별도로 폴링해야 함 (JobExplorer.getJobExecution())
    예외가 run() 호출자에게 전파되지 않아 모니터링 별도 필요
```

---

## 📌 핵심 정리

```
SimpleJobLauncher.run() 실행 순서

  1. JobParametersValidator.validate()       → 파라미터 유효성 검증
  2. jobRepository.getLastJobExecution()     → 중복/재시작 가능 여부 확인
  3. jobRepository.createJobExecution()      → DB에 STARTING 상태 저장
  4. taskExecutor.execute(job::execute)      → 동기 or 비동기 실행
  5. job.execute()                           → AbstractJob.execute()
     → beforeJob() → doExecute() → afterJob()
  6. JobExecution 반환

동기 vs 비동기 핵심 차이
  SyncTaskExecutor  → run() 반환 = Job 완료 (HTTP 타임아웃 위험)
  AsyncTaskExecutor → run() 반환 = STARTING 즉시 반환 (상태 별도 조회 필요)

예외 종류
  JobExecutionAlreadyRunningException  → 이미 RUNNING 중
  JobInstanceAlreadyCompleteException  → COMPLETED 상태 재실행 시도
  JobRestartException                  → preventRestart() 또는 UNKNOWN 상태
  JobParametersInvalidException        → 파라미터 유효성 실패
```

---

## 🤔 생각해볼 문제

**Q1.** `SimpleJobLauncher`가 비동기 `TaskExecutor`를 사용할 때, `run()`이 반환한 `JobExecution`의 상태는 `STARTING`입니다. 호출자가 최종 결과를 알려면 어떻게 해야 하는가?

**Q2.** 두 개의 서버 인스턴스가 동시에 같은 `Job`을 같은 `JobParameters`로 실행하려 할 때, `JobLauncher`의 중복 실행 방지 메커니즘이 정상 동작하는가? DB 격리 수준과 어떤 관련이 있는가?

**Q3.** `spring.batch.job.enabled=false`로 설정하면 `JobLauncherApplicationRunner`가 등록되지 않습니다. 이 설정을 언제 사용해야 하며, Job을 실행하는 다른 방법은 무엇인가?

> 💡 **해설**
>
> **Q1.** `JobExplorer.getJobExecution(executionId)`를 주기적으로 폴링하거나, `JobExecutionListener.afterJob()`에서 이벤트를 발행해 WebSocket/SSE로 클라이언트에 푸시하는 방법을 사용합니다. Spring Batch Integration의 `JobExecutionEvent`를 구독하는 것도 방법입니다. 반환된 `JobExecution.getId()`를 클라이언트에게 즉시 전달하고, 클라이언트가 이 ID로 상태를 주기적으로 조회하는 패턴이 일반적입니다.
>
> **Q2.** `jobRepository.createJobExecution()`이 `BATCH_JOB_INSTANCE`에 대해 DB SELECT 후 상태를 확인합니다. 두 인스턴스가 동시에 SELECT를 수행하면 둘 다 기존 실행이 없다고 판단할 수 있습니다(Race Condition). Spring Batch는 이를 방지하기 위해 낙관적 잠금(version 컬럼)을 사용합니다. `BATCH_JOB_EXECUTION`의 `version` 컬럼을 UPDATE할 때 충돌이 발생하면 `OptimisticLockingFailureException`이 발생해 한 인스턴스는 실패합니다. DB의 `REPEATABLE_READ` 이상 격리 수준이 필요합니다.
>
> **Q3.** `spring.batch.job.enabled=false`는 멀티 Job 애플리케이션에서 특정 Job만 선택 실행하거나, REST API/스케줄러로만 실행을 제어할 때 사용합니다. 이 경우 `JobLauncher`를 직접 주입받아 `@Scheduled`, REST 컨트롤러, 또는 `ApplicationRunner`에서 명시적으로 실행합니다. 또는 `spring.batch.job.name=specificJobName`으로 특정 Job만 자동 실행하도록 제한할 수도 있습니다.

---

<div align="center">

**[⬅️ 이전: Job·Step·Tasklet 관계와 구조](./01-job-step-tasklet-structure.md)** | **[홈으로 🏠](../README.md)** | **[다음: JobRepository와 메타데이터 관리 ➡️](./03-job-repository-metadata.md)**

</div>
