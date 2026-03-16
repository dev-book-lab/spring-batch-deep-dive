# Chunk Size 튜닝 — 트랜잭션 빈도·메모리·JDBC 배치 효율의 균형점

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Chunk Size를 10배 늘리면 성능은 어떻게 변화하는가? 항상 빠른가?
- OOM 없이 처리할 수 있는 Chunk Size의 한계는 어떻게 계산하는가?
- Chunk Size와 DB 커넥션 풀, JDBC 배치 크기의 관계는?
- `JpaPagingItemReader.pageSize`와 `chunk size`는 같아야 하는가?
- 실제 운영 환경에서 최적 Chunk Size를 찾는 체계적인 방법은?

---

## 🔍 Chunk Size가 영향을 미치는 요소

### 문제: Chunk Size는 하나의 값이지만 여러 트레이드오프가 얽혀있다

```
Chunk Size = 1000으로 설정했을 때:

  트랜잭션 레벨:
    커밋 횟수 = 총 건수 / 1000
    → 100만 건: 1000번 커밋
    → 각 커밋마다: BATCH_STEP_EXECUTION UPDATE + EC UPDATE

  메모리 레벨:
    Chunk<I> inputs  = 1000개의 읽기 객체
    Chunk<O> outputs = 1000개의 처리 결과 객체
    → 힙에 동시에 존재: 2000개 객체

  JDBC 레벨:
    batchUpdate(sql, 1000개 파라미터)
    → 1000개를 한 번의 네트워크 왕복으로 전송
    → DB: 1000개를 하나의 실행 계획으로 처리

  재시작 레벨:
    실패 시 최대 재처리 건수 = 1000건
    → 마지막 커밋 후 읽은 건수만큼

Chunk Size를 바꾸면 이 모든 것이 동시에 변한다
→ 최적값 = 성능·메모리·재시작 안전성의 균형
```

---

## 😱 흔한 실수

### Before: Chunk Size를 최대한 크게 잡으면 항상 빠르다

```
❌ 잘못된 가정: Chunk Size ∝ 성능 (비례 관계)

실제 성능 곡선 (100만 건, MySQL, JdbcBatchItemWriter):

  Chunk 10   → 35분 (커밋 오버헤드 100,000번)
  Chunk 100  → 6분
  Chunk 1000 → 45초  ← 급격한 개선 구간
  Chunk 5000 → 40초  ← 개선 미미해짐
  Chunk 10000→ 42초  ← 오히려 느려질 수 있음 (GC 압력)
  Chunk 50000→ OOM   ← 힙 512m 환경에서 OOM 발생

→ 1000~5000 구간에서 성능이 수렴
→ 그 이상은 메모리 위험만 커짐
```

### Before: pageSize와 chunk size를 맞추지 않는다

```java
// ❌ pageSize와 chunk size 불일치
@Bean
public JpaPagingItemReader<Order> reader() {
    return new JpaPagingItemReaderBuilder<Order>()
        .pageSize(100)    // 100건씩 페이지 조회
        .build();
}

@Bean
public Step step() {
    return stepBuilderFactory.get("step")
        .<Order, SettledOrder>chunk(1000)   // 1000건 Chunk
        .reader(reader())
        .build();
}
// 문제:
// chunk(1000)은 1000번 read()를 호출해 Chunk를 채움
// pageSize(100)은 100번마다 새 페이지 쿼리 실행
// → Chunk 1개 = 10번의 페이지 쿼리 = 10번의 EntityManager 재생성
// 작동은 하지만 불필요한 EntityManager 생성 오버헤드 발생

// ✅ pageSize = chunk size로 일치 권장
.pageSize(1000)  // Chunk당 정확히 1번의 페이지 쿼리
```

---

## ✨ OOM 없는 Chunk Size 계산 공식

```
안전한 Chunk Size 계산:

  사용 가능한 힙 = JVM -Xmx 설정값 (예: 512MB)
  배치용 힙 비율 = 전체 힙의 30~50% (나머지는 JVM 내부 사용)
  아이템 1개 크기 (직렬화 포함, 실측 권장): 예) 5KB

  안전 Chunk Size = 배치용 힙 / (아이템 크기 × 2)
                  = (512MB × 0.4) / (5KB × 2)
                  = 204MB / 10KB
                  ≈ 20,000건

  실측에서는 이론값의 50%로 설정 권장:
  → 10,000건 (안전 마진 확보)

  실제 측정 방법:
  JVM 플래그: -verbose:gc -XX:+PrintGCDetails
  또는: -Xmx512m -XX:+HeapDumpOnOutOfMemoryError
```

---

## 🔬 내부 동작 원리

### 1. Chunk Size와 트랜잭션 커밋 오버헤드

```java
// 커밋마다 발생하는 작업 목록:
// (TaskletStep.doExecute() 트랜잭션 템플릿 내)

① itemWriter.write(chunk)                    // 실제 쓰기 (DB INSERT)
② stepExecution.apply(contribution)          // 카운터 업데이트
③ jobRepository.updateExecutionContext(se)   // BATCH_STEP_EXECUTION_CONTEXT UPDATE
④ jobRepository.update(stepExecution)        // BATCH_STEP_EXECUTION UPDATE
⑤ transactionManager.commit()                // 실제 커밋

// Chunk Size 10 → 100만 건:
// 위 5단계 × 100,000번 = 500,000번의 DB 작업
// Chunk Size 1000:
// 위 5단계 × 1,000번 = 5,000번의 DB 작업
// → 100배 감소
```

### 2. 실측 성능 데이터 — MySQL 환경

```
실험 환경:
  데이터: 100만 건 orders 테이블 (order_id, amount, status, customer_id, order_date)
  DB: MySQL 8.0 (로컬)
  JVM: -Xmx512m
  Reader: JdbcCursorItemReader
  Writer: JdbcBatchItemWriter (rewriteBatchedStatements=true)
  Processor: 수수료 계산 (순수 연산, DB 조회 없음)

Chunk Size별 측정 결과:
┌────────────┬──────────┬─────────────┬──────────────┬──────────────────┐
│ Chunk Size │ 처리 시간  │ TPS         │ GC 횟수       │ 최대 힙 사용량       │
├────────────┼──────────┼─────────────┼──────────────┼──────────────────┤
│ 10         │ 2,100초   │ ~476 건/초   │ 1,200회      │ 45MB             │
│ 100        │ 360초     │ ~2,778 건/초 │ 350회        │ 50MB             │
│ 1,000      │ 45초      │ ~22,222 건/초│ 40회         │ 80MB             │
│ 5,000      │ 40초      │ ~25,000 건/초│ 12회         │ 250MB            │
│ 10,000     │ 42초      │ ~23,810 건/초│ 8회          │ 480MB            │
│ 50,000     │ OOM      │ -           │ -            │ >512MB           │
└────────────┴──────────┴─────────────┴──────────────┴──────────────────┘

분석:
  10→100:   6배 개선 (커밋 오버헤드 10배 감소)
  100→1000: 8배 개선 (JDBC 배치 효율 + 커밋 감소)
  1000→5000: 11% 개선 (수확 체감)
  5000→10000: 오히려 느림 (GC 압력 증가)
  50000: OOM (힙 512m 초과)
```

### 3. Chunk Size와 JDBC 배치 효율의 관계

```
MySQL rewriteBatchedStatements=true:

Chunk 10:
  INSERT INTO t VALUES (1,2),(3,4),...,(19,20)  ← 10건 합산 쿼리
  → 합산 SQL 크기: ~300 bytes
  → 100만 건: 100,000번의 배치 실행

Chunk 1000:
  INSERT INTO t VALUES (1,2),(3,4),...,(1999,2000)  ← 1000건 합산 쿼리
  → 합산 SQL 크기: ~30,000 bytes (30KB)
  → 100만 건: 1,000번의 배치 실행

Chunk 10000:
  INSERT INTO t VALUES ... ← 10,000건 합산 쿼리
  → 합산 SQL 크기: ~300,000 bytes (300KB)
  → MySQL max_allowed_packet 초과 위험! (기본값 4MB)
  → DB 패킷 크기 제한 확인 필요

# MySQL 패킷 크기 설정
spring.datasource.url: jdbc:mysql://...?rewriteBatchedStatements=true&maxAllowedPacket=67108864
# 또는 my.cnf: max_allowed_packet = 64M
```

### 4. Processor가 무거울 때 Chunk Size 조정

```java
// Processor가 외부 API를 호출하는 경우
@Component
public class ExternalApiProcessor implements ItemProcessor<Order, EnrichedOrder> {

    @Override
    public EnrichedOrder process(Order order) throws Exception {
        // 외부 API 호출 — 건당 100ms 소요
        CustomerProfile profile = externalApiClient.getProfile(order.getCustomerId());
        return new EnrichedOrder(order, profile);
    }
}

// Chunk Size 1000 + 건당 100ms = Chunk당 100초 → 타임아웃 위험
// → Processor가 무거우면 Chunk Size를 줄여야 함
// → 또는 AsyncItemProcessor로 병렬 처리 (Ch6-01 참조)

// 적정 Chunk Size = 허용 트랜잭션 시간 / 건당 처리 시간
// 10초 타임아웃 / 100ms = 최대 100건

@Bean
public Step heavyProcessorStep() {
    return stepBuilderFactory.get("heavyProcessorStep")
        .<Order, EnrichedOrder>chunk(100)    // 외부 API: 건당 100ms → Chunk당 10초
        .reader(reader())
        .processor(externalApiProcessor())
        .writer(writer())
        .build();
}
```

---

## 💻 실전 구현

### 동적 Chunk Size — JobParameters로 런타임 조정

```java
@Bean("processStep")
@JobScope
public Step processStep(
        StepBuilderFactory stepBuilderFactory,
        @Value("#{jobParameters['chunkSize'] ?: 1000}") Integer chunkSize,
        ItemReader<Order> reader,
        ItemProcessor<Order, SettledOrder> processor,
        ItemWriter<SettledOrder> writer) {

    log.info("Chunk Size: {}", chunkSize);
    return stepBuilderFactory.get("processStep")
        .<Order, SettledOrder>chunk(chunkSize)
        .reader(reader)
        .processor(processor)
        .writer(writer)
        .build();
}

// 실행 시 Chunk Size 지정:
// jobLauncher.run(job, new JobParametersBuilder()
//     .addLong("chunkSize", 5000L)
//     .toJobParameters());
```

### Chunk Size 자동 튜닝 — 처리 시간 기반 적응형

```java
// Chunk 처리 시간을 모니터링해 동적으로 Chunk Size 조정
@Component
public class AdaptiveChunkSizeListener implements ChunkListener {

    private static final long TARGET_CHUNK_MS = 1000;  // 목표: Chunk당 1초
    private static final int MIN_CHUNK_SIZE = 100;
    private static final int MAX_CHUNK_SIZE = 10000;

    private long chunkStartTime;
    private int currentChunkSize = 1000;

    @Override
    public void beforeChunk(ChunkContext context) {
        chunkStartTime = System.currentTimeMillis();
    }

    @Override
    public void afterChunk(ChunkContext context) {
        long elapsed = System.currentTimeMillis() - chunkStartTime;
        StepExecution se = context.getStepContext().getStepExecution();
        int actualChunkSize = (int) (se.getWriteCount() + se.getFilterCount())
            % currentChunkSize;

        // 비율 조정: 실제 소요 시간 → 목표 시간 비율로 Chunk Size 조정
        double ratio = (double) TARGET_CHUNK_MS / elapsed;
        int newSize = (int) (currentChunkSize * ratio);
        currentChunkSize = Math.max(MIN_CHUNK_SIZE, Math.min(MAX_CHUNK_SIZE, newSize));

        log.info("Chunk 처리: {}ms, 다음 Chunk Size 조정: {}", elapsed, currentChunkSize);
    }
}
// 주의: 실제 chunk() 설정은 Step 빌드 시 고정됨
// → 이 패턴은 모니터링 목적으로 사용하고, 실제 조정은 다음 실행 시 적용
```

---

## 📊 Chunk Size 선택 가이드

```
상황별 권장 Chunk Size:

┌─────────────────────────────┬───────────────┬──────────────────────────┐
│ 상황                         │ 권장 Chunk Size│ 이유                      │
├─────────────────────────────┼───────────────┼──────────────────────────┤
│ 순수 DB 읽기/쓰기 (빠른 처리)     │ 1,000~5,000   │ JDBC 배치 효율 최대화       │
│ 외부 API 호출 포함 (건당 100ms)  │ 50~100        │ 타임아웃 방지               │
│ 외부 API 호출 포함 (건당 10ms)   │ 500~1,000     │ 적정 균형                  │
│ 파일 기반 처리                  │ 500~2,000     │ 파일 I/O 특성              │
│ 메모리 제한 환경 (-Xmx256m)     │ 100~500       │ OOM 방지                  │
│ Skip/Retry가 많은 경우         │ 100~500       │ 재처리 부담 최소화           │
│ Partitioning (워커 다수)      │ 500~1,000     │ 병렬 트랜잭션 충돌 방지        │
└─────────────────────────────┴───────────────┴──────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
Chunk Size의 트레이드오프:

  크게 설정 (5000+)
    장점  JDBC 배치 효율 최대 (네트워크 왕복 감소)
          커밋 횟수 최소 (JobRepository DB 부하 감소)
          GC 빈도 감소
    단점  메모리 사용량 증가 (OOM 위험)
          실패 시 재처리 건수 증가
          DB 배치 패킷 크기 제한 초과 위험
          트랜잭션 타임아웃 위험 (Processor가 무거울 경우)

  작게 설정 (~100)
    장점  메모리 안전
          실패 시 재처리 최소화
          Processor 타임아웃 안전
    단점  커밋 오버헤드 증가
          JDBC 배치 효율 낮아짐

  황금 구간 (1000~5000)
    일반적인 대용량 배치에서 성능·안전성 균형
```

---

## 📌 핵심 정리

```
Chunk Size가 영향 미치는 4가지

  ① 트랜잭션 커밋 횟수  Chunk Size ↑ → 커밋 횟수 ↓ (JobRepository 부하 감소)
  ② 메모리 사용량        Chunk Size ↑ → 힙 사용량 ↑ (OOM 위험 증가)
  ③ JDBC 배치 효율       Chunk Size ↑ → 배치 효율 ↑ (수확 체감 — 1000 이후 미미)
  ④ 재시작 재처리 건수   Chunk Size ↑ → 재처리 건수 ↑ (실패 비용 증가)

성능 수렴 구간: 1000~5000 (이후 수확 체감)
OOM 계산: 배치용 힙 / (아이템 크기 × 2)

pageSize = chunk size 권장
  → Chunk당 정확히 1번 페이지 쿼리 실행
  → EntityManager 재생성 최소화

외부 API 포함 시 Chunk Size 계산
  허용 트랜잭션 시간 / 건당 처리 시간 = 최대 Chunk Size
```

---

## 🤔 생각해볼 문제

**Q1.** Chunk Size를 1000으로 설정했는데, `ItemProcessor`에서 50%의 아이템을 필터링(`null` 반환)합니다. 실제로 Writer에 전달되는 Chunk의 크기는 얼마인가? 이 경우 JDBC 배치 효율은 어떻게 달라지는가?

**Q2.** `JpaPagingItemReader.pageSize(1000)`과 `chunk(1000)`으로 설정했을 때, 정확히 2500건이 있다면 EntityManager는 몇 번 생성되는가?

**Q3.** Chunk Size를 10000으로 설정했더니 5000으로 설정했을 때보다 오히려 2초 느려졌습니다. 왜 이런 현상이 발생하는가? 어떻게 확인할 수 있는가?

> 💡 **해설**
>
> **Q1.** Writer에 전달되는 Chunk 크기는 약 500건 (1000건의 50%)입니다. JDBC 배치 효율 측면에서는 `batchUpdate(sql, 500개)` vs `batchUpdate(sql, 1000개)`의 차이가 생깁니다. 배치 효율은 저하되지만, 처리 시간에 Processor 필터링 부분이 없으므로 전체 TPS는 향상될 수 있습니다. 중요한 것은 `readCount=1000, writeCount=500, filterCount=500`으로 기록된다는 점입니다. Processor 필터링이 많다면 실제 Write되는 건수를 기준으로 Chunk Size를 조정하는 것이 더 의미 있습니다.
>
> **Q2.** `JpaPagingItemReader`는 페이지가 소진될 때마다(pageSize건 읽을 때마다) 새 EntityManager를 생성합니다. 2500건 / 1000건 per page = 3페이지가 필요합니다. 단, 마지막(4번째) 페이지 쿼리는 0건을 반환해 루프가 종료되므로, 실제 EntityManager는 4번(또는 3번) 생성됩니다. Chunk는 2500건 / 1000건 per chunk = 3번 처리됩니다. pageSize = chunkSize이면 Chunk당 정확히 1번의 EntityManager 생성이 보장됩니다.
>
> **Q3.** Chunk Size 10000이 5000보다 느린 이유는 GC 압력 증가입니다. 힙에 동시에 20000개 객체(input 10000 + output 10000)가 올라가면, Young Generation이 자주 가득 차서 Minor GC 빈도는 줄지만 Major GC가 더 오래 걸립니다(Stop-the-world). 확인 방법: `-XX:+PrintGCDetails -XX:+PrintGCDateStamps` 로그에서 GC pause 시간을 확인합니다. 또는 JVM Flight Recorder나 VisualVM으로 힙 사용량 그래프를 관찰합니다. 해결책: Chunk Size를 5000으로 유지하거나, `-Xmx`를 늘려서 GC 압력을 낮춥니다.

---

<div align="center">

**[⬅️ 이전: ItemWriter 구현](./04-item-writer-implementations.md)** | **[홈으로 🏠](../README.md)** | **[다음: Cursor vs Paging 선택 기준 ➡️](./06-cursor-vs-paging.md)**

</div>
