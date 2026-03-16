# Chunk 처리 원리 — Read → Process → Write 반복 루프의 내부

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `ChunkOrientedTasklet.execute()`의 Read 루프는 정확히 어떤 순서로 동작하는가?
- Chunk Size가 트랜잭션 경계와 동일한 이유는 무엇인가?
- `ItemReader.read()`가 `null`을 반환할 때 루프가 종료되는 메커니즘은?
- `Chunk<I>`와 `Chunk<O>`는 서로 다른 타입인데 어떻게 연결되는가?
- Chunk 처리 도중 예외가 발생했을 때 트랜잭션 경계는 어떻게 되는가?

---

## 🔍 왜 Chunk 단위로 처리하는가

### 문제: 100만 건을 한 트랜잭션으로 처리하면 안 되는 이유

```
단순 접근 — 전체를 하나의 트랜잭션으로:

  BEGIN TRANSACTION
    read 100만 건 (메모리에 모두 로드)
    process 100만 건
    write 100만 건
  COMMIT

  문제 1 — 메모리:
    100만 건 × 1KB = 1GB가 힙에 올라옴
    JVM -Xmx512m 설정이면 OOM 즉시 발생

  문제 2 — 트랜잭션:
    999,999건 성공 후 마지막 1건에서 오류 발생
    → 전체 100만 건 롤백
    → 처음부터 다시 시작해야 함

  문제 3 — 잠금:
    100만 건에 대한 DB 행 잠금이 COMMIT 전까지 유지
    → 다른 트랜잭션 대기 시간 폭증

Chunk 처리 해결책:
  1000건씩 읽어 → 처리 → 쓰고 → COMMIT (메모리 해제)
  → 재시작 시 마지막 COMMIT 지점부터 재개
  → 메모리: 항상 1000건만 힙에 존재
  → 잠금: COMMIT마다 해제
```

---

## 😱 흔한 실수

### Before: Chunk 처리 중 write()에서 예외가 나면 read()부터 다시 시작한다고 생각한다

```java
// ❌ 잘못된 이해
// "chunk(1000) 설정에서 500건 write 중 예외 발생
//  → 501번째 read부터 재시작"

// 실제:
// write()에서 예외 → 현재 Chunk(1000건) 전체 롤백
// → 재시작 시: 이 Chunk의 첫 번째 read부터 다시 시작
//              (ExectuionContext의 read.count 기준)

// Skip 설정이 없으면 Step 전체가 FAILED
// Skip 설정이 있으면 ChunkProcessor가 1건씩 재처리 (스캐터-개더)
// → Ch4-01 Skip 전략에서 상세 설명
```

### Before: ItemProcessor를 생략하면 read()한 객체가 그대로 write()된다

```java
// ✅ processor가 없으면 실제로 read 타입 = write 타입이어야 함
@Bean
public Step noProcessorStep() {
    return stepBuilderFactory.get("noProcessorStep")
        .<Order, Order>chunk(1000)  // <I, O>가 같은 타입
        .reader(orderReader())
        // .processor(...)  생략 가능
        .writer(orderWriter())
        .build();
}
// 내부적으로 PassThroughItemProcessor가 사용됨 (항등 함수)
// I와 O 타입이 다른데 processor를 생략하면 컴파일 오류
```

---

## ✨ 올바른 이해와 사용

### ChunkOrientedTasklet 전체 실행 흐름

```java
// ChunkOrientedTasklet.java — 핵심 execute() 로직 (단순화)
@Override
public RepeatStatus execute(StepContribution contribution,
                             ChunkContext chunkContext) throws Exception {
    // ① 이전 Chunk 처리 중이었던 경우 (재시작) 복원
    Chunk<I> inputs = (Chunk<I>) chunkContext.getAttribute(INPUTS_KEY);

    if (inputs == null) {
        // ② 새 Chunk 읽기 시작 (ChunkProvider)
        inputs = chunkProvider.provide(contribution);
        if (chunkContext.isComplete() && inputs.isEmpty()) {
            return RepeatStatus.FINISHED;  // 더 읽을 데이터 없음 → Step 종료
        }
    }

    // ③ Process + Write (ChunkProcessor)
    chunkProcessor.process(contribution, inputs);

    // ④ 처리 완료된 Chunk 정보 제거
    chunkContext.removeAttribute(INPUTS_KEY);
    chunkContext.setComplete();

    // ⑤ 아직 데이터가 있을 수 있으므로 CONTINUABLE 반환
    return RepeatStatus.CONTINUABLE;
}
```

### ChunkProvider.provide() — Read 루프

```java
// SimpleChunkProvider.java
@Override
public Chunk<I> provide(StepContribution contribution) throws Exception {
    Chunk<I> inputs = new Chunk<>();

    repeatOperations.iterate(new RepeatCallback() {
        @Override
        public RepeatStatus doInIteration(RepeatContext context) throws Exception {
            I item = null;
            try {
                // ItemReader.read() 호출
                item = itemReader.read();
            } catch (SkipableReadException e) {
                // Skip 처리 (Ch4 참조)
            }

            if (item == null) {
                // ← null = 데이터 소진 → 루프 종료 신호
                inputs.setEnd();
                return RepeatStatus.FINISHED;
            }

            contribution.incrementReadCount();
            inputs.add(item);  // Chunk에 추가
            return RepeatStatus.CONTINUABLE;
        }
    });
    // RepeatOperations가 chunkSize번 반복하면 자동으로 루프 종료
    // (CompletionPolicy: SimpleCompletionPolicy — count == chunkSize)

    return inputs;  // Chunk<I> 반환
}
```

### ChunkProcessor.process() — Process + Write

```java
// SimpleChunkProcessor.java
@Override
public void process(StepContribution contribution, Chunk<I> inputs) throws Exception {

    // ① Process 단계: Chunk<I> → Chunk<O>
    Chunk<O> outputs = transform(contribution, inputs);

    // ② Write 단계: Chunk<O> 일괄 쓰기
    write(contribution, inputs, outputs);
}

protected Chunk<O> transform(StepContribution contribution, Chunk<I> inputs)
        throws Exception {

    Chunk<O> outputs = new Chunk<>();

    for (Chunk<I>.ChunkIterator iterator = inputs.iterator(); iterator.hasNext(); ) {
        final I item = iterator.next();
        O output;
        try {
            // ItemProcessor.process() 호출
            output = doProcess(item);
        } catch (Exception e) {
            // Skip 처리 (Ch4 참조)
            iterator.remove(e);  // inputs에서 제거
            continue;
        }

        if (output != null) {
            outputs.add(output);  // null이 아닌 것만 outputs에 추가
        } else {
            // Processor가 null 반환 → 필터링 (Write에서 제외)
            contribution.incrementFilterCount(1);
            iterator.remove();  // inputs에서도 제거
        }
    }

    return outputs;
}

protected void write(StepContribution contribution,
                      Chunk<I> inputs,
                      Chunk<O> outputs) throws Exception {
    try {
        // ItemWriter.write() 호출 — 한 번에 Chunk 전체 전달
        doWrite(outputs.getItems());
    } catch (Exception e) {
        // Skip 처리 (Ch4 참조)
        throw e;
    }
    contribution.incrementWriteCount(outputs.size());
}
```

---

## 🔬 내부 동작 원리

### 1. 트랜잭션 경계와 Chunk의 관계

```java
// TaskletStep.java — 트랜잭션 템플릿으로 Tasklet 래핑
protected void doExecute(StepExecution stepExecution) throws Exception {
    TransactionTemplate transactionTemplate =
        new TransactionTemplate(transactionManager, transactionAttribute);

    RepeatStatus status;
    do {
        // 매 Chunk = 하나의 트랜잭션
        status = transactionTemplate.execute(txStatus -> {
            StepContribution contribution = stepExecution.createStepContribution();
            ChunkContext chunkContext = new ChunkContext(new StepContext(stepExecution));

            // ① ChunkOrientedTasklet.execute() 호출
            //    → ChunkProvider.provide() (read 1000건)
            //    → ChunkProcessor.process() (process + write)
            RepeatStatus result = tasklet.execute(contribution, chunkContext);

            // ② StepExecution 카운터 업데이트
            stepExecution.apply(contribution);

            // ③ ExecutionContext 저장 (재시작 포인트 갱신)
            getJobRepository().updateExecutionContext(stepExecution);
            getJobRepository().update(stepExecution);

            return result;  // CONTINUABLE or FINISHED
        });
        // ④ 트랜잭션 COMMIT (Chunk 경계)

    } while (RepeatStatus.CONTINUABLE.equals(status));
    // FINISHED 반환 시 루프 종료 → Step 완료
}
```

### 2. Chunk 처리 타임라인 (1000건 × 3 Chunk)

```
Step 시작
  │
  ▼  [Chunk 1 — 트랜잭션 시작]
  ├─ Read   001~1000번 아이템
  ├─ Process 001~1000번 아이템
  ├─ Write  001~1000번 아이템 (BATCH INSERT)
  ├─ UPDATE BATCH_STEP_EXECUTION (readCount=1000, writeCount=1000)
  ├─ UPDATE BATCH_STEP_EXECUTION_CONTEXT (read.count=1000)
  └─ COMMIT ──────────────────────────────────── 재시작 안전 지점 ①
  │
  ▼  [Chunk 2 — 트랜잭션 시작]
  ├─ Read   1001~2000번 아이템
  ├─ Process 1001~2000번 아이템
  ├─ Write  1001~2000번 아이템
  ├─ UPDATE BATCH_STEP_EXECUTION (readCount=2000, writeCount=2000)
  ├─ UPDATE BATCH_STEP_EXECUTION_CONTEXT (read.count=2000)
  └─ COMMIT ──────────────────────────────────── 재시작 안전 지점 ②
  │
  ▼  [Chunk 3 — 트랜잭션 시작]
  ├─ Read   2001~2500번 아이템 (마지막 Chunk, 500건)
  ├─ Read   → null 반환 (데이터 소진)
  ├─ Process 2001~2500번 아이템
  ├─ Write  2001~2500번 아이템
  ├─ UPDATE BATCH_STEP_EXECUTION (readCount=2500, writeCount=2500)
  ├─ UPDATE BATCH_STEP_EXECUTION_CONTEXT (read.count=2500)
  └─ COMMIT
  │
  ▼
  ChunkOrientedTasklet.execute() → FINISHED 반환
  Step COMPLETED
```

### 3. null 반환으로 루프 종료되는 내부 흐름

```java
// RepeatOperations (SimpleRepeatTemplate)가 루프를 제어
// ChunkSize = CompletionPolicy의 상한

// SimpleCompletionPolicy — count 기반 종료
public class SimpleCompletionPolicy implements CompletionPolicy {
    private int chunkSize;

    @Override
    public boolean isComplete(RepeatContext context, RepeatStatus result) {
        // ① ItemReader가 null 반환 → FINISHED → 즉시 종료
        if (result == RepeatStatus.FINISHED) return true;

        // ② count == chunkSize → Chunk 가득 참 → 종료
        return ((SimpleRepeatContext) context).getStartedCount() >= chunkSize;
    }
}

// 결과:
// read() → item   → CONTINUABLE → 아직 count < chunkSize → 계속
// read() → item   → CONTINUABLE → count == chunkSize     → 루프 종료 (Chunk 완성)
// read() → null   → FINISHED    → 즉시 루프 종료         (데이터 소진)
```

### 4. Chunk<I>와 Chunk<O> — 타입 변환 경로

```
ItemReader<Order>          ItemProcessor<Order, SettledOrder>    ItemWriter<SettledOrder>
        │                              │                                  │
        │ read() → Order               │                                  │
        │─────────────────────────────▶                                   │
        │                     process(Order) → SettledOrder               │
        │                     ──────────────────────────────────────────▶ │
        │                     null 반환 시 → 필터링 (Writer 미전달)            │
        │                                                                 │
 Chunk<Order>  inputs                                  Chunk<SettledOrder> outputs
  [Order, Order, ...]     transform()              [SettledOrder, SettledOrder, ...]
                          ──────────▶
                                                    write(List<SettledOrder>) 한 번에 전달
```

---

## 💻 실전 구현

### 완전한 Chunk Job 구현 예시

```java
@Configuration
public class OrderSettlementJobConfig {

    @Bean
    public Job orderSettlementJob() {
        return jobBuilderFactory.get("orderSettlementJob")
            .incrementer(new RunIdIncrementer())
            .start(settlementStep())
            .build();
    }

    @Bean
    public Step settlementStep() {
        return stepBuilderFactory.get("settlementStep")
            .<Order, SettledOrder>chunk(1000)         // Chunk Size = 트랜잭션 단위
            .reader(orderReader())
            .processor(orderProcessor())
            .writer(settlementWriter())
            .listener(new ChunkProgressListener())    // 처리 진행률 로깅
            .build();
    }

    @Bean
    @StepScope
    public JpaPagingItemReader<Order> orderReader(
            @Value("#{jobParameters['targetDate']}") String targetDate) {
        return new JpaPagingItemReaderBuilder<Order>()
            .name("orderReader")
            .entityManagerFactory(emf)
            .queryString("SELECT o FROM Order o WHERE o.date = :date AND o.status = 'PENDING'")
            .parameterValues(Map.of("date", LocalDate.parse(targetDate)))
            .pageSize(1000)
            .build();
    }

    @Bean
    public ItemProcessor<Order, SettledOrder> orderProcessor() {
        return order -> {
            if (order.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
                return null;  // null 반환 → 이 아이템은 Write 제외 (필터링)
            }
            BigDecimal fee = order.getAmount().multiply(new BigDecimal("0.015"));
            return new SettledOrder(order, fee);
        };
    }

    @Bean
    public JdbcBatchItemWriter<SettledOrder> settlementWriter(DataSource dataSource) {
        return new JdbcBatchItemWriterBuilder<SettledOrder>()
            .dataSource(dataSource)
            .sql("INSERT INTO settled_order (order_id, amount, fee, settled_at) " +
                 "VALUES (:orderId, :amount, :fee, NOW())")
            .beanMapped()
            .build();
    }
}

// Chunk 처리 진행률 로깅 Listener
public class ChunkProgressListener implements ChunkListener {
    private long startTime;
    private long totalWritten = 0;

    @Override
    public void beforeChunk(ChunkContext context) {
        startTime = System.currentTimeMillis();
    }

    @Override
    public void afterChunk(ChunkContext context) {
        StepExecution se = context.getStepContext().getStepExecution();
        totalWritten = se.getWriteCount();
        long elapsed = System.currentTimeMillis() - startTime;
        log.info("[Chunk 완료] 누적 처리: {}건, 이번 Chunk: {}ms",
            totalWritten, elapsed);
    }

    @Override
    public void afterChunkError(ChunkContext context) {
        log.error("[Chunk 실패] 누적 처리: {}건에서 오류", totalWritten);
    }
}
```

---

## 📊 Chunk Size별 동작 비교 (100만 건 기준)

```
Chunk Size가 트랜잭션과 성능에 미치는 영향:

┌────────────┬──────────┬────────────┬─────────────┬──────────────────────────┐
│ Chunk Size │ 트랜잭션   │ 메모리 사용   │ JDBC 배치    │ 재시작 시 재처리 최대 건수     │
│            │ 커밋 횟수  │ (힙 최대)    │ 효율         │                          │
├────────────┼──────────┼────────────┼─────────────┼──────────────────────────┤
│ 10         │ 100,000  │ ~10KB      │ 낮음         │ 10건                      │
│ 100        │ 10,000   │ ~100KB     │ 중간         │ 100건                     │
│ 1,000      │ 1,000    │ ~1MB       │ 높음         │ 1,000건                   │
│ 5,000      │ 200      │ ~5MB       │ 매우 높음     │ 5,000건                   │
│ 100,000    │ 10       │ ~100MB     │ 최고         │ 100,000건                 │
└────────────┴──────────┴────────────┴─────────────┴──────────────────────────┘

권장: 1,000 ~ 5,000 (성능과 재시작 안전성의 균형)
상세 성능 측정 → Ch2-05 참조
```

---

## ⚖️ 트레이드오프

```
Chunk 크기:

  크게 설정 시
    장점  DB 배치 효율 증가 (JDBC batch INSERT 효과)
          트랜잭션 커밋 횟수 감소 (JobRepository DB 업데이트 빈도 감소)
    단점  메모리 사용량 증가
          실패 시 재처리해야 하는 건수 증가

  작게 설정 시
    장점  메모리 사용량 최소화
          실패 시 재처리 건수 최소화
    단점  DB 배치 효율 낮아짐
          트랜잭션 커밋 오버헤드 증가

Chunk 기반 처리 vs Tasklet 전체 처리:

  Chunk
    장점  메모리 제어, 재시작 가능, Skip/Retry 적용 가능
    단점  설정 복잡 (Reader/Processor/Writer 분리)

  단일 Tasklet
    장점  구현 단순
    단점  OOM 위험, 재시작 불가, 단일 트랜잭션 (실패 시 전체 롤백)
```

---

## 📌 핵심 정리

```
ChunkOrientedTasklet 실행 흐름

  1. ChunkProvider.provide()    → ItemReader.read()를 ChunkSize번 호출
                                  null 반환 시 즉시 종료 (Chunk.isEnd() = true)
  2. ChunkProcessor.transform() → ItemProcessor.process() 건별 호출
                                  null 반환 시 해당 아이템 필터링 (Writer 제외)
  3. ChunkProcessor.write()     → ItemWriter.write(List) 한 번에 전달
  4. 트랜잭션 COMMIT            → ExecutionContext 갱신 (재시작 포인트)
  5. CONTINUABLE 반환 → 루프 반복
     FINISHED 반환  → Step 종료

트랜잭션 경계 = Chunk 경계
  → Chunk 내 어떤 아이템에서든 예외 발생 시 해당 Chunk 전체 롤백
  → 다음 Chunk는 이전 COMMIT까지는 안전하게 처리된 상태

null 반환의 의미
  ItemReader.read() → null  = 데이터 소진, 루프 종료 신호
  ItemProcessor.process() → null = 이 아이템을 Writer에 전달하지 말 것 (필터링)
```

---

## 🤔 생각해볼 문제

**Q1.** `ChunkOrientedTasklet.execute()`는 한 번 호출될 때 하나의 Chunk만 처리하고 `CONTINUABLE`을 반환합니다. 그런데 `TaskletStep`은 `do-while`로 반복 호출합니다. 이 구조에서 `ChunkContext`가 왜 필요한가? `Chunk<I>`를 인스턴스 변수에 저장하면 안 되는 이유는?

**Q2.** `ItemProcessor`가 `null`을 반환했을 때 `contribution.incrementFilterCount(1)`이 호출됩니다. 이 `filterCount`는 `BATCH_STEP_EXECUTION`의 어떤 컬럼에 저장되는가? 그리고 `filterCount`는 `readCount`와 `writeCount`의 차이(`readCount - writeCount`)와 항상 같은가?

**Q3.** `chunk(1000)`으로 설정했는데 DB에 데이터가 정확히 2500건 있습니다. Chunk 처리는 몇 번 일어나고, 마지막 Chunk에서 Write되는 건수는 몇 건인가? `ItemReader.read()`는 총 몇 번 호출되는가?

> 💡 **해설**
>
> **Q1.** `ChunkContext`는 `TaskletStep`의 트랜잭션 템플릿 내에서 생성되고 트랜잭션과 함께 존재합니다. 만약 `Chunk<I>`를 `ChunkOrientedTasklet`의 인스턴스 변수에 저장하면, `Tasklet` Bean은 싱글톤이므로 멀티 쓰레드 환경(Multi-threaded Step)에서 여러 쓰레드가 같은 `Chunk<I>` 인스턴스를 공유해 Race Condition이 발생합니다. `ChunkContext`는 각 `StepExecution` 별로 독립적이므로 쓰레드 안전합니다. 또한 Chunk 처리 도중 예외로 트랜잭션이 롤백될 때, `ChunkContext`에서 `INPUTS_KEY` 속성을 제거해 다음 트랜잭션에서 새로 Read하도록 합니다.
>
> **Q2.** `filterCount`는 `BATCH_STEP_EXECUTION.FILTER_COUNT` 컬럼에 저장됩니다. `filterCount`와 `readCount - writeCount`는 Skip이 없는 경우에는 같습니다. 하지만 Skip이 발생하면 `readSkipCount`나 `writeSkipCount`가 추가되므로 `readCount - writeCount = filterCount + skipCount(합계)`가 됩니다. 즉, `filterCount`는 Processor가 `null`을 반환한 건수만 계산하고, Skip된 건수는 별도 카운터로 관리됩니다.
>
> **Q3.** Chunk 처리는 3번 일어납니다. Chunk 1: 1~1000번 (1000건), Chunk 2: 1001~2000번 (1000건), Chunk 3: 2001~2500번 (500건). `ItemReader.read()`는 총 **2501번** 호출됩니다 — 2500번은 실제 아이템을 반환하고, 마지막 1번(2501번째)은 `null`을 반환해 Chunk 3의 Read 루프를 종료시킵니다. 정확히는 Chunk 3에서 Read 루프가 `chunkSize(1000)`에 도달하기 전에 `null`을 만나므로 500건만 읽고 루프가 종료됩니다.

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: ItemReader 종류와 구현 ➡️](./02-item-reader-types.md)**

</div>
