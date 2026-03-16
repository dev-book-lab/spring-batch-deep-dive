# JobRepository와 메타데이터 관리 — 6개 테이블이 배치 상태를 기록하는 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Spring Batch가 사용하는 6개 메타데이터 테이블은 각각 무엇을 저장하고, 서로 어떻게 연관되는가?
- `JobRepository`가 `JobExecution`·`StepExecution` 상태를 저장하는 정확한 시점은 언제인가?
- 배치 실패 후 DB에서 어떤 컬럼을 보면 실패 원인과 재시작 가능 여부를 알 수 있는가?
- `ExecutionContext`는 어떤 형식으로 직렬화되어 DB에 저장되는가?
- 운영 환경에서 메타데이터 테이블이 너무 커지면 어떻게 관리해야 하는가?

---

## 🔍 왜 메타데이터가 필요한가

### 문제: 배치는 언제든 중단될 수 있다

```
배치 실패 시나리오:

  시나리오 A — 중간 실패
    주문 정산 Job: 100만 건 처리 중 50만 건째에서 DB 연결 끊김
    → 재시작 시 처음부터? 아니면 50만 1건부터?
    → "어디까지 처리했는가"를 영구 저장소에 기록해야만 재시작 가능

  시나리오 B — 중복 실행 방지
    스케줄러가 같은 Job을 두 번 실행 시도
    → "지금 실행 중인가?"를 DB에서 확인해야 함
    → 인메모리 상태는 서버 재시작 시 사라짐

  시나리오 C — 이력 관리
    "지난달 정산 배치가 몇 시에 완료됐나?"
    "어느 Step에서 몇 건이 Skip됐나?"
    → 실행 이력을 쿼리할 수 있어야 함

해결: JobRepository를 통한 DB 기반 상태 관리
  → 모든 실행 상태를 DB에 저장
  → 서버 재시작, 클러스터 환경에서도 일관된 상태 유지
```

---

## 😱 흔한 실수

### Before: 메타데이터 테이블의 의미를 모르고 운영한다

```sql
-- ❌ "배치가 왜 실행 안 되지?"라며 원인을 못 찾는 경우
SELECT * FROM BATCH_JOB_EXECUTION ORDER BY CREATE_TIME DESC LIMIT 5;

-- 결과를 보면:
-- STATUS = 'UNKNOWN' → 서버가 강제 종료됐을 때 발생하는 상태
-- EXIT_CODE = 'UNKNOWN' → 정상 종료 처리가 되지 않음
-- → UNKNOWN 상태는 JobRepository가 자동으로 복구하지 않음
-- → 수동으로 FAILED로 변경해야 재시작 가능

-- ✅ 올바른 처리:
UPDATE BATCH_JOB_EXECUTION
SET STATUS = 'FAILED', EXIT_CODE = 'FAILED', EXIT_MESSAGE = 'Manually set to FAILED for restart'
WHERE JOB_EXECUTION_ID = ? AND STATUS = 'UNKNOWN';
```

### Before: ExecutionContext가 너무 커지도록 방치한다

```java
// ❌ ExecutionContext에 대용량 데이터를 저장
@Override
public void update(ExecutionContext executionContext) {
    // 처리한 모든 ID 목록을 EC에 저장 (1만 건이면 수백 KB)
    executionContext.put("processedIds", processedIds);  // List<Long> 1만 개
}
// → BATCH_STEP_EXECUTION_CONTEXT의 SHORT_CONTEXT 컬럼 길이 초과
// → 직렬화 크기가 커져 매 Chunk 커밋마다 DB 부하 증가
```

---

## ✨ 올바른 이해와 사용

### 6개 메타데이터 테이블 구조

```sql
-- ① BATCH_JOB_INSTANCE — Job 고유 인스턴스 (JobName + JobKey 조합)
CREATE TABLE BATCH_JOB_INSTANCE (
    JOB_INSTANCE_ID  BIGINT       NOT NULL PRIMARY KEY,
    VERSION          BIGINT,                          -- 낙관적 잠금
    JOB_NAME         VARCHAR(100) NOT NULL,           -- "orderSettlementJob"
    JOB_KEY          VARCHAR(32)  NOT NULL,           -- JobParameters의 MD5 해시
    UNIQUE (JOB_NAME, JOB_KEY)                        -- 같은 파라미터 조합 중복 불가
);

-- ② BATCH_JOB_EXECUTION — Job 실행 이력 (1 Instance : N Execution)
CREATE TABLE BATCH_JOB_EXECUTION (
    JOB_EXECUTION_ID    BIGINT        NOT NULL PRIMARY KEY,
    VERSION             BIGINT,
    JOB_INSTANCE_ID     BIGINT        NOT NULL REFERENCES BATCH_JOB_INSTANCE,
    CREATE_TIME         DATETIME(6)   NOT NULL,
    START_TIME          DATETIME(6),
    END_TIME            DATETIME(6),
    STATUS              VARCHAR(10),   -- STARTING, STARTED, COMPLETED, FAILED, STOPPED, UNKNOWN
    EXIT_CODE           VARCHAR(2500), -- COMPLETED, FAILED, STOPPED
    EXIT_MESSAGE        VARCHAR(2500), -- 실패 시 스택 트레이스 일부
    LAST_UPDATED        DATETIME(6)
);

-- ③ BATCH_JOB_EXECUTION_PARAMS — JobParameters 저장
CREATE TABLE BATCH_JOB_EXECUTION_PARAMS (
    JOB_EXECUTION_ID    BIGINT       NOT NULL REFERENCES BATCH_JOB_EXECUTION,
    PARAMETER_NAME      VARCHAR(100) NOT NULL,
    PARAMETER_TYPE      VARCHAR(100) NOT NULL,  -- STRING, LONG, DOUBLE, DATE
    PARAMETER_VALUE     VARCHAR(2500),
    IDENTIFYING         CHAR(1)      NOT NULL   -- 'Y': JobInstance 식별에 사용
);

-- ④ BATCH_JOB_EXECUTION_CONTEXT — Job 수준 ExecutionContext
CREATE TABLE BATCH_JOB_EXECUTION_CONTEXT (
    JOB_EXECUTION_ID    BIGINT        NOT NULL PRIMARY KEY REFERENCES BATCH_JOB_EXECUTION,
    SHORT_CONTEXT       VARCHAR(2500) NOT NULL,  -- JSON 직렬화 (2500자 이내)
    SERIALIZED_CONTEXT  TEXT                     -- 2500자 초과 시 사용
);

-- ⑤ BATCH_STEP_EXECUTION — Step 실행 이력 (1 JobExecution : N StepExecution)
CREATE TABLE BATCH_STEP_EXECUTION (
    STEP_EXECUTION_ID    BIGINT        NOT NULL PRIMARY KEY,
    VERSION              BIGINT        NOT NULL,
    STEP_NAME            VARCHAR(100)  NOT NULL,
    JOB_EXECUTION_ID     BIGINT        NOT NULL REFERENCES BATCH_JOB_EXECUTION,
    CREATE_TIME          DATETIME(6)   NOT NULL,
    START_TIME           DATETIME(6),
    END_TIME             DATETIME(6),
    STATUS               VARCHAR(10),
    COMMIT_COUNT         BIGINT,        -- Chunk 커밋 횟수
    READ_COUNT           BIGINT,        -- 읽은 아이템 수
    FILTER_COUNT         BIGINT,        -- 필터링된 아이템 수 (Processor null 반환)
    WRITE_COUNT          BIGINT,        -- 쓴 아이템 수
    READ_SKIP_COUNT      BIGINT,        -- 읽기 Skip 수
    WRITE_SKIP_COUNT     BIGINT,        -- 쓰기 Skip 수
    PROCESS_SKIP_COUNT   BIGINT,        -- 처리 Skip 수
    ROLLBACK_COUNT       BIGINT,        -- 롤백 횟수
    EXIT_CODE            VARCHAR(2500),
    EXIT_MESSAGE         VARCHAR(2500),
    LAST_UPDATED         DATETIME(6)
);

-- ⑥ BATCH_STEP_EXECUTION_CONTEXT — Step 수준 ExecutionContext (재시작 키)
CREATE TABLE BATCH_STEP_EXECUTION_CONTEXT (
    STEP_EXECUTION_ID   BIGINT        NOT NULL PRIMARY KEY REFERENCES BATCH_STEP_EXECUTION,
    SHORT_CONTEXT       VARCHAR(2500) NOT NULL,
    SERIALIZED_CONTEXT  TEXT
);
```

---

## 🔬 내부 동작 원리

### 1. JobRepository가 상태를 저장하는 시점 — 타임라인

```
JobLauncher.run() 호출

  T1: jobRepository.createJobExecution()
      → BATCH_JOB_INSTANCE INSERT (신규 시)
      → BATCH_JOB_EXECUTION INSERT (status=STARTING)
      → BATCH_JOB_EXECUTION_PARAMS INSERT

  T2: AbstractJob.execute() 시작
      → BATCH_JOB_EXECUTION UPDATE (status=STARTED, startTime=now)

  T3: AbstractStep.execute() 시작
      → BATCH_STEP_EXECUTION INSERT (status=STARTING)
      → BATCH_STEP_EXECUTION UPDATE (status=STARTED, startTime=now)

  T4: 각 Chunk 처리 후 커밋 시 (ChunkOrientedTasklet)
      → BATCH_STEP_EXECUTION UPDATE (readCount+=N, writeCount+=N, commitCount++)
      → BATCH_STEP_EXECUTION_CONTEXT UPDATE (ExecutionContext 저장 — 재시작 포인트!)

  T5: Step 완료
      → BATCH_STEP_EXECUTION UPDATE (status=COMPLETED, endTime=now)

  T6: 모든 Step 완료
      → BATCH_JOB_EXECUTION UPDATE (status=COMPLETED, endTime=now)
      → BATCH_JOB_EXECUTION_CONTEXT UPDATE (최종 상태)
```

### 2. SimpleJobRepository — 핵심 메서드 소스 추적

```java
// SimpleJobRepository.java

// Step 실행 시작 기록
@Override
public void add(StepExecution stepExecution) {
    validateStepExecution(stepExecution);
    stepExecution.setId(stepExecutionIncrementer.nextLongValue());
    stepExecutionDao.saveStepExecution(stepExecution);      // INSERT
    ecDao.saveExecutionContext(stepExecution);               // EC INSERT (빈 상태)
}

// Chunk 커밋마다 StepExecution + ExecutionContext 갱신
@Override
public void update(StepExecution stepExecution) {
    validateStepExecution(stepExecution);
    stepExecution.setLastUpdated(new Date());
    stepExecutionDao.updateStepExecution(stepExecution);    // UPDATE count 컬럼들
    checkForInterruption(stepExecution);
}

// ExecutionContext 저장 (재시작 포인트!)
@Override
public void updateExecutionContext(StepExecution stepExecution) {
    ExecutionContext executionContext = stepExecution.getExecutionContext();
    if (executionContext != null) {
        ecDao.updateExecutionContext(stepExecution);  // SHORT_CONTEXT UPDATE
    }
}
```

### 3. ExecutionContext 직렬화 방식

```java
// ExecutionContextSerializer (Jackson 기반)
// ExecutionContext → JSON 문자열

// 예시: JdbcCursorItemReader의 ExecutionContext
{
  "FlatFileItemReader.read.count": 50000,
  "FlatFileItemReader.read.count.max": -1
}

// JpaPagingItemReader의 ExecutionContext
{
  "JpaPagingItemReader.read.count": 50000
}

// 재시작 시 ItemStream.open(executionContext)에서 이 값을 읽어
// 50000번째 아이템부터 재개

// ❗ BATCH_STEP_EXECUTION_CONTEXT.SHORT_CONTEXT 길이 제한
// Spring Batch 5.x: VARCHAR(2500) — 초과 시 SERIALIZED_CONTEXT 사용
// 직렬화된 크기가 2500자를 넘으면 자동으로 SERIALIZED_CONTEXT에 저장
```

### 4. 테이블 관계 ERD

```
BATCH_JOB_INSTANCE (1)
    ├── JOB_INSTANCE_ID (PK)
    ├── JOB_NAME
    └── JOB_KEY (JobParameters MD5)
         │
         │ 1:N
         ▼
BATCH_JOB_EXECUTION (N)
    ├── JOB_EXECUTION_ID (PK)
    ├── JOB_INSTANCE_ID (FK)
    ├── STATUS
    └── EXIT_CODE
         │                    │
         │ 1:1                │ 1:N
         ▼                    ▼
BATCH_JOB_EXECUTION_CONTEXT  BATCH_JOB_EXECUTION_PARAMS
    └── SHORT_CONTEXT             └── PARAMETER_NAME/VALUE
         │
         │ (JOB_EXECUTION_ID 공유)
         │
BATCH_STEP_EXECUTION (N per JobExecution)
    ├── STEP_EXECUTION_ID (PK)
    ├── JOB_EXECUTION_ID (FK)
    ├── STEP_NAME
    ├── READ_COUNT / WRITE_COUNT
    └── COMMIT_COUNT
         │
         │ 1:1
         ▼
BATCH_STEP_EXECUTION_CONTEXT
    └── SHORT_CONTEXT (재시작 포인트)
```

---

## 💻 실전 구현

### 배치 실행 이력 조회 쿼리 모음

```sql
-- 1. 최근 10개 Job 실행 이력 (상태 포함)
SELECT
    ji.JOB_NAME,
    je.JOB_EXECUTION_ID,
    je.STATUS,
    je.EXIT_CODE,
    je.CREATE_TIME,
    je.START_TIME,
    je.END_TIME,
    TIMESTAMPDIFF(SECOND, je.START_TIME, je.END_TIME) AS duration_sec
FROM BATCH_JOB_EXECUTION je
JOIN BATCH_JOB_INSTANCE ji ON je.JOB_INSTANCE_ID = ji.JOB_INSTANCE_ID
ORDER BY je.CREATE_TIME DESC
LIMIT 10;

-- 2. 특정 Job의 Step별 처리 건수 확인
SELECT
    se.STEP_NAME,
    se.STATUS,
    se.READ_COUNT,
    se.WRITE_COUNT,
    se.FILTER_COUNT,
    se.READ_SKIP_COUNT + se.WRITE_SKIP_COUNT + se.PROCESS_SKIP_COUNT AS total_skip,
    se.ROLLBACK_COUNT,
    se.COMMIT_COUNT
FROM BATCH_STEP_EXECUTION se
WHERE se.JOB_EXECUTION_ID = ?
ORDER BY se.START_TIME;

-- 3. 재시작 가능한 실패 Job 목록
SELECT
    ji.JOB_NAME,
    ji.JOB_INSTANCE_ID,
    je.JOB_EXECUTION_ID,
    je.STATUS,
    je.EXIT_MESSAGE
FROM BATCH_JOB_EXECUTION je
JOIN BATCH_JOB_INSTANCE ji ON je.JOB_INSTANCE_ID = ji.JOB_INSTANCE_ID
WHERE je.STATUS IN ('FAILED', 'STOPPED')
  AND je.JOB_EXECUTION_ID = (
      SELECT MAX(je2.JOB_EXECUTION_ID)
      FROM BATCH_JOB_EXECUTION je2
      WHERE je2.JOB_INSTANCE_ID = je.JOB_INSTANCE_ID
  );

-- 4. Step의 ExecutionContext 확인 (재시작 포인트)
SELECT
    sec.STEP_EXECUTION_ID,
    se.STEP_NAME,
    sec.SHORT_CONTEXT
FROM BATCH_STEP_EXECUTION_CONTEXT sec
JOIN BATCH_STEP_EXECUTION se ON sec.STEP_EXECUTION_ID = se.STEP_EXECUTION_ID
WHERE se.JOB_EXECUTION_ID = ?;
```

### 오래된 메타데이터 정리 (운영 필수)

```java
// Spring Batch 5.x — JobOperator로 정리
@Component
public class BatchMetadataCleanup {

    @Autowired
    private JobOperator jobOperator;

    @Autowired
    private JobExplorer jobExplorer;

    @Scheduled(cron = "0 0 2 * * *")  // 매일 새벽 2시
    public void cleanupOldExecutions() {
        // 30일 이전 완료된 실행 삭제
        LocalDate cutoffDate = LocalDate.now().minusDays(30);
        Set<Long> jobExecutionIds = jobExplorer.findRunningJobExecutions("orderSettlementJob")
            .stream()
            .map(JobExecution::getId)
            .collect(Collectors.toSet());

        // JobExecutionDao.deleteJobExecution()를 직접 사용하거나
        // spring-batch-admin 라이브러리의 JobService.abandon() 활용
    }
}

-- 또는 직접 SQL로 정리 (cascade 순서 중요!)
DELETE FROM BATCH_STEP_EXECUTION_CONTEXT
WHERE STEP_EXECUTION_ID IN (
    SELECT se.STEP_EXECUTION_ID FROM BATCH_STEP_EXECUTION se
    JOIN BATCH_JOB_EXECUTION je ON se.JOB_EXECUTION_ID = je.JOB_EXECUTION_ID
    WHERE je.CREATE_TIME < DATE_SUB(NOW(), INTERVAL 30 DAY)
      AND je.STATUS IN ('COMPLETED', 'FAILED')
);
-- (같은 방식으로 STEP_EXECUTION → JOB_EXECUTION_CONTEXT → JOB_EXECUTION_PARAMS → JOB_EXECUTION → JOB_INSTANCE 순으로 삭제)
```

---

## 📊 STATUS 값 의미

```
BatchStatus 값과 상황:

┌────────────┬────────────────────────────────────────────────────────┐
│ STATUS     │ 의미 및 발생 상황                                          │
├────────────┼────────────────────────────────────────────────────────┤
│ STARTING   │ JobLauncher가 JobExecution을 생성한 직후                   │
│ STARTED    │ Job.execute() 진입 시점                                  │
│ COMPLETED  │ 모든 Step 성공 완료                                       │
│ FAILED     │ Step에서 예외 발생 또는 ExitStatus=FAILED                  │
│ STOPPED    │ JobOperator.stop() 또는 Step이 STOPPED 반환               │
│ STOPPING   │ stop() 요청 접수, 다음 Chunk 처리 후 실제 중단 예정            │
│ ABANDONED  │ 재시작하지 않기로 결정한 FAILED 실행                          │
│ UNKNOWN    │ JVM 강제 종료 등 비정상 종료 — 수동 FAILED 변경 필요            │
└────────────┴────────────────────────────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
DB 기반 JobRepository:

  장점
    영구 저장 — 서버 재시작 후에도 상태 유지
    클러스터 환경에서 여러 인스턴스가 공유
    SQL로 실행 이력 조회 가능
    낙관적 잠금으로 중복 실행 방지

  단점
    매 Chunk 커밋마다 DB UPDATE 발생 (추가 I/O)
    메타데이터 테이블이 커짐 → 주기적 정리 필요
    테스트 시 H2 설정 필요 (또는 MapJobRepository 사용)

MapJobRepository (인메모리, 테스트용):
  장점  DB 불필요, 테스트 빠름
  단점  재시작 불가, 클러스터 환경 사용 불가
        Spring Batch 5.x에서 deprecated
```

---

## 📌 핵심 정리

```
6개 테이블 역할 요약

  BATCH_JOB_INSTANCE          → Job + JobParameters 조합의 고유 식별자
  BATCH_JOB_EXECUTION         → 실행 이력 (시작/종료 시간, 상태)
  BATCH_JOB_EXECUTION_PARAMS  → 실행 시 사용된 JobParameters
  BATCH_JOB_EXECUTION_CONTEXT → Job 수준 ExecutionContext (Job 간 데이터 공유)
  BATCH_STEP_EXECUTION        → Step별 처리 카운트, 상태
  BATCH_STEP_EXECUTION_CONTEXT→ Step 수준 ExecutionContext (재시작 포인트 ← 핵심)

상태 저장 시점 핵심
  Chunk 커밋마다: BATCH_STEP_EXECUTION 카운트 UPDATE
                  BATCH_STEP_EXECUTION_CONTEXT UPDATE (재시작 포인트 갱신)
  재시작 시:      BATCH_STEP_EXECUTION_CONTEXT에서 읽은 위치 복원
                  → ItemStream.open(executionContext) 호출

UNKNOWN 상태 처리
  JVM 강제 종료 시 발생
  → 수동으로 FAILED로 UPDATE 후 재시작
```

---

## 🤔 생각해볼 문제

**Q1.** 같은 `orderSettlementJob`을 `targetDate=2024-01-01`로 실행하면 항상 같은 `JOB_KEY`가 생성됩니다. 이 `JOB_KEY`는 어떻게 계산되며, `JobParameters`의 어떤 파라미터들이 `JOB_KEY` 계산에 포함되는가?

**Q2.** 배치 서버가 JVM `kill -9`로 강제 종료됐을 때 `BATCH_JOB_EXECUTION`의 STATUS가 `UNKNOWN`이 되는 이유는? Spring Batch가 이를 자동으로 `FAILED`로 복구하지 않는 이유는?

**Q3.** `BATCH_STEP_EXECUTION_CONTEXT`의 `SHORT_CONTEXT`가 Chunk 커밋마다 UPDATE된다면, 100만 건을 1000건 씩 처리할 때 총 1000번 UPDATE가 발생합니다. 이 오버헤드를 줄일 수 있는 방법은?

> 💡 **해설**
>
> **Q1.** `JOB_KEY`는 `JobParametersConverter`가 `JobParameters`를 문자열로 변환한 후 MD5 해시를 생성합니다. 중요한 점은 `identifying=true`로 설정된 파라미터만 `JOB_KEY`에 포함된다는 것입니다. `JobParametersBuilder.addLong("timestamp", ..., false)`처럼 `identifying=false`로 설정하면 해당 파라미터는 DB에 저장되지만 `JOB_KEY` 계산에서 제외됩니다. 이를 이용해 "같은 날짜 파라미터로 실행하되, 타임스탬프로 고유성 부여" 없이도 재실행이 가능하게 설계할 수 있습니다.
>
> **Q2.** Spring Batch는 `ShutdownHook`을 통해 정상 종료 시 상태를 저장합니다. `kill -9`는 ShutdownHook을 실행하지 않으므로 `BATCH_JOB_EXECUTION`은 마지막으로 UPDATE된 상태(`STARTED` 또는 `STARTING`)로 남고, Spring Batch는 이를 `UNKNOWN`으로 간주합니다. 자동 복구를 하지 않는 이유는 "중간에 일부만 커밋된 데이터가 있을 수 있으므로" 사람이 데이터 정합성을 확인한 후 재시작 여부를 결정하도록 설계됐기 때문입니다.
>
> **Q3.** `ExecutionContext` UPDATE 오버헤드를 줄이는 방법: (1) Chunk Size를 키워 커밋 횟수를 줄입니다 (1000건 → 5000건 = UPDATE 횟수 1/5). (2) `ExecutionContext`에 저장하는 데이터 크기를 최소화합니다 — 꼭 필요한 재시작 포인트(읽기 위치)만 저장합니다. (3) `@StepScope`로 ItemReader를 빈으로 등록할 때 `saveState(false)` 설정 시 EC를 저장하지 않습니다(단, 재시작 불가). Spring Batch의 EC UPDATE는 트랜잭션 내에서 Chunk Write와 같이 묶여 실행되므로 별도 Round-trip을 추가하지는 않습니다.

---

<div align="center">

**[⬅️ 이전: JobLauncher와 Job 실행 메커니즘](./02-job-launcher-execution.md)** | **[홈으로 🏠](../README.md)** | **[다음: JobInstance vs JobExecution vs StepExecution ➡️](./04-job-instance-execution.md)**

</div>
