# ItemWriter 구현 — JPA, JDBC, File Writer 내부와 배치 최적화

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `JpaItemWriter`가 `entityManager.merge()`를 사용하는 이유와 `persist()`와의 차이는?
- `JdbcBatchItemWriter`가 내부에서 `JdbcTemplate.batchUpdate()`를 사용하는 과정은?
- `FlatFileItemWriter`의 `append` 모드가 재시작 안전성을 보장하는 방법은?
- Chunk 단위로 `write(List<T>)`를 받는 설계의 의도는?
- `CompositeItemWriter`로 여러 목적지에 동시에 쓰는 방법은?

---

## 🔍 왜 write(List<T>)를 받는가 — 단건 write가 아닌 이유

### 문제: 1건씩 INSERT하면 100만 건 처리가 너무 느리다

```
단건 INSERT vs 배치 INSERT 성능 차이:

  1건씩 INSERT (Chunk Size 1000, 100만 건):
    실행 횟수: 1,000,000번 INSERT
    각 INSERT: 네트워크 왕복 + DB 파싱 + 실행 계획
    총 시간: ~1,000,000 × 1ms = 약 16분

  배치 INSERT (Chunk Size 1000, 100만 건):
    실행 횟수: 1,000번 (Chunk당 1번)
    JDBC batchUpdate: 1000건을 한 번의 네트워크로 전송
    DB: 한 번의 실행 계획으로 1000건 처리
    총 시간: ~1,000 × 5ms = 약 5초

  → 200배 이상 차이

ItemWriter가 List<T>를 받는 이유:
  → Chunk 전체를 한 번에 받아 배치 처리 가능
  → JDBC batchUpdate(), JPA EntityManager flush 최적화
  → 단건 write API였다면 배치 처리 불가
```

---

## 😱 흔한 실수

### Before: JpaItemWriter를 사용할 때 ID 생성 전략을 고려하지 않는다

```java
// ❌ IDENTITY 전략 + JpaItemWriter → 배치 INSERT 불가
@Entity
public class SettledOrder {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // ← 문제
    private Long id;
}

// IDENTITY 전략:
// INSERT 후 DB가 ID를 생성 → JPA가 ID를 알려면 INSERT 즉시 실행
// → Hibernate가 배치 INSERT를 비활성화!
// → 1000건을 1000번의 단건 INSERT로 실행

// 실제 Hibernate 로그:
// HHH90009021: Disabling contextual LOB creation as JDBC driver 
// "rewriteBatchedStatements" is disabled

// ✅ 배치 INSERT를 위한 올바른 ID 전략
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE)  // SEQUENCE 사용
@SequenceGenerator(name = "settled_order_seq", allocationSize = 1000)
private Long id;

// 또는 UUID/애플리케이션 생성 ID
@Id
private String id = UUID.randomUUID().toString();

// MySQL 사용 시: application.yml
spring.jpa.properties.hibernate.jdbc.batch_size: 1000
spring.jpa.properties.hibernate.order_inserts: true
spring.jpa.properties.hibernate.order_updates: true
```

### Before: FlatFileItemWriter 재시작 시 파일이 처음부터 덮어써진다

```java
// ❌ append 모드 설정 안 함
@Bean
@StepScope
public FlatFileItemWriter<SettledOrder> writer() {
    return new FlatFileItemWriterBuilder<SettledOrder>()
        .name("settlementWriter")
        .resource(new FileSystemResource("/data/output.csv"))
        // append(true) 빠짐!
        .lineAggregator(new DelimitedLineAggregator<>())
        .build();
}
// 재시작 시: output.csv를 처음부터 덮어씀
// → 이전에 성공한 Chunk의 결과가 사라짐

// ✅ 올바른 설정
.append(true)        // 재시작 시 기존 내용 뒤에 추가
.shouldDeleteIfExists(false)  // 파일 있어도 삭제 안 함
.shouldDeleteIfEmpty(true)    // 빈 파일은 삭제 (선택적)
```

---

## ✨ 올바른 이해와 사용

### JpaItemWriter vs JdbcBatchItemWriter 선택 기준

```
Writer 선택:

  JpaItemWriter
    → JPA 엔티티 저장, 연관관계 관리가 필요할 때
    → Cascade 저장이 필요할 때
    → 단점: IDENTITY 전략 시 배치 비활성화

  JdbcBatchItemWriter
    → 빠른 성능이 최우선일 때
    → 단순 INSERT/UPDATE (연관관계 불필요)
    → IDENTITY 전략과도 배치 동작 (DB가 배치 처리)
    → 권장: 대용량 배치에서 가장 빠름
```

---

## 🔬 내부 동작 원리

### 1. JpaItemWriter — merge() 사용 이유

```java
// JpaItemWriter.java
public class JpaItemWriter<T> extends AbstractItemStreamItemWriter<T> {

    private EntityManagerFactory entityManagerFactory;
    private boolean usePersist = false;  // 기본값: merge

    @Override
    public void write(Chunk<? extends T> chunk) throws Exception {
        EntityManager entityManager = EntityManagerFactoryUtils
            .getTransactionalEntityManager(entityManagerFactory);

        if (entityManager == null) {
            throw new DataAccessResourceFailureException("...");
        }

        for (T item : chunk) {
            if (!usePersist) {
                // ① merge() — ID 있으면 UPDATE, 없으면 INSERT
                //   배치 재시작 시 같은 ID로 다시 들어와도 안전 (중복 INSERT 없음)
                entityManager.merge(item);
            } else {
                // ② persist() — 반드시 신규 INSERT
                //   ID가 이미 존재하면 EntityExistsException 발생
                entityManager.persist(item);
            }
        }

        // ③ Chunk 전체를 한 번에 flush
        // → 이 시점에 실제 SQL이 실행됨 (배치 INSERT)
        entityManager.flush();

        // ④ 1차 캐시 초기화 (다음 Chunk를 위해 메모리 해제)
        entityManager.clear();
    }
}

// merge() vs persist() 선택:
// .usePersist(true)  → persist (신규 데이터 확실할 때, 약간 더 빠름)
// .usePersist(false) → merge (기본값, 재처리 안전, 약간 느림)
```

### 2. JdbcBatchItemWriter — batchUpdate() 내부

```java
// JdbcBatchItemWriter.java
public class JdbcBatchItemWriter<T> implements ItemWriter<T>, ... {

    private NamedParameterJdbcOperations namedParameterJdbcTemplate;
    private String sql;
    private ItemSqlParameterSourceProvider<T> itemSqlParameterSourceProvider;

    @Override
    public void write(Chunk<? extends T> chunk) throws Exception {
        if (logger.isDebugEnabled()) {
            logger.debug("Executing batch with " + chunk.size() + " items.");
        }

        // ① 각 아이템 → SqlParameterSource 변환
        List<T> items = chunk.getItems();
        SqlParameterSource[] batchArgs = new SqlParameterSource[items.size()];
        int i = 0;
        for (T item : items) {
            batchArgs[i++] = itemSqlParameterSourceProvider.createSqlParameterSource(item);
        }

        // ② JDBC batchUpdate 실행 (1000건을 한 번의 네트워크 왕복으로)
        int[] updateCounts = namedParameterJdbcTemplate.batchUpdate(sql, batchArgs);

        // ③ assertUpdates: 업데이트 건수 0이면 예외 (삭제된 레코드 업데이트 등)
        if (assertUpdates) {
            for (int count : updateCounts) {
                if (count == 0) {
                    throw new EmptyResultDataAccessException(
                        "Item was expected to be affected: " + sql, 1);
                }
            }
        }
    }
}
```

### 3. MySQL JDBC 배치 활성화 필수 설정

```yaml
# application.yml — MySQL에서 배치 INSERT 활성화
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/batchdb?rewriteBatchedStatements=true
    #                                      ↑ 이 설정 없으면 배치 비활성화
    # rewriteBatchedStatements=true:
    # INSERT INTO t (a, b) VALUES (1, 2); × 1000건
    # → INSERT INTO t (a, b) VALUES (1, 2), (3, 4), ..., (1999, 2000);
    # → 하나의 SQL로 합쳐서 실행 (네트워크 1번)
```

### 4. FlatFileItemWriter — append 모드와 재시작 안전성

```java
// FlatFileItemWriter.java
public class FlatFileItemWriter<T> extends AbstractFileItemWriter<T> {

    private boolean append = false;
    private boolean shouldDeleteIfExists = true;

    @Override
    public void open(ExecutionContext executionContext) {
        // ① 재시작인지 확인 (ExecutionContext에 파일 위치 있는지)
        if (executionContext.containsKey(getExecutionContextKey(RESTART_DATA_KEY))) {
            // 재시작: 이전에 쓴 위치(byte offset) 복원
            long filePosition = executionContext.getLong(
                getExecutionContextKey(RESTART_DATA_KEY));

            // ② 파일을 append 모드로 열고, 이전 위치까지 truncate
            // → 이전 Chunk의 마지막 완전한 줄 이후로 위치 조정
            initializeBufferedWriter(resource, true, filePosition);
        } else {
            // 신규 실행: shouldDeleteIfExists에 따라 파일 처리
            initializeBufferedWriter(resource, append, 0);
        }
    }

    // Chunk 커밋마다 현재 파일 위치 저장
    @Override
    public void update(ExecutionContext executionContext) {
        if (state != null && isSaveState()) {
            // 현재 파일 포인터 위치를 바이트 단위로 저장
            executionContext.putLong(getExecutionContextKey(RESTART_DATA_KEY),
                state.position());
            // → BATCH_STEP_EXECUTION_CONTEXT에 저장
            // → 재시작 시 이 위치 이후에 다시 쓰기 시작
        }
    }
}
```

### 5. FlatFileItemWriter 재시작 흐름

```
정상 처리:
  Chunk 1: 1~1000건 → output.csv (0~5000 byte)
  [COMMIT: ExecutionContext에 filePosition=5000 저장]
  Chunk 2: 1001~2000건 → output.csv (5000~10000 byte)
  [COMMIT: ExecutionContext에 filePosition=10000 저장]
  Chunk 3: 2001~2500건 → output.csv ← 실패 (DB 오류)
  [ROLLBACK: filePosition은 10000으로 유지됨]

재시작:
  open(executionContext) → filePosition=10000 확인
  → output.csv를 10000 byte까지만 유지 (뒤쪽 부분 제거)
  → 2001번 아이템부터 다시 쓰기 시작
  → 중복 없이 정확히 이어서 작성
```

### 6. CompositeItemWriter — 여러 목적지에 동시 쓰기

```java
// CompositeItemWriter.java
public class CompositeItemWriter<T> implements ItemStreamWriter<T> {

    private List<ItemWriter<? super T>> delegates;

    @Override
    public void write(Chunk<? extends T> chunk) throws Exception {
        for (ItemWriter<? super T> writer : delegates) {
            writer.write(chunk);  // 각 Writer에 같은 Chunk 전달
        }
    }

    // 각 Writer의 open/update/close도 모두 위임
    @Override
    public void open(ExecutionContext executionContext) {
        for (ItemStream stream : streams) {
            stream.open(executionContext);
        }
    }
}

// 사용 예: DB + 파일에 동시 쓰기
@Bean
public CompositeItemWriter<SettledOrder> compositeWriter() {
    CompositeItemWriter<SettledOrder> writer = new CompositeItemWriter<>();
    writer.setDelegates(List.of(
        jdbcWriter(),    // DB INSERT
        csvWriter(),     // CSV 파일 기록
        auditWriter()    // 감사 로그 테이블 INSERT
    ));
    return writer;
}
```

---

## 💻 실전 구현

### JdbcBatchItemWriter — 성능 최적화 완전 설정

```java
@Bean
public JdbcBatchItemWriter<SettledOrder> settlementWriter(DataSource dataSource) {
    return new JdbcBatchItemWriterBuilder<SettledOrder>()
        .dataSource(dataSource)
        .sql("""
            INSERT INTO settled_order (
                order_id, original_amount, fee, net_amount, settled_at, customer_id
            ) VALUES (
                :orderId, :originalAmount, :fee, :netAmount, :settledAt, :customerId
            ) ON DUPLICATE KEY UPDATE  -- 재처리 시 중복 무시
                fee = VALUES(fee),
                net_amount = VALUES(net_amount),
                settled_at = VALUES(settled_at)
            """)
        .beanMapped()              // BeanPropertySqlParameterSource 사용
        .assertUpdates(false)      // ON DUPLICATE KEY UPDATE 시 count=2 허용
        .build();
}
```

### ClassifierCompositeItemWriter — 조건별 다른 Writer

```java
// 금액에 따라 다른 테이블에 저장
@Bean
public ClassifierCompositeItemWriter<SettledOrder> classifierWriter(
        ItemWriter<SettledOrder> normalWriter,
        ItemWriter<SettledOrder> highValueWriter) {

    BackToBackPatternClassifier classifier = new BackToBackPatternClassifier();
    classifier.setRouterDelegate((SettledOrder order) -> {
        // 100만원 초과는 high_value_settlement 테이블
        return order.getOriginalAmount().compareTo(new BigDecimal("1000000")) > 0
            ? "high" : "normal";
    });
    classifier.setMatcherMap(Map.of(
        "normal", normalWriter,
        "high", highValueWriter
    ));

    ClassifierCompositeItemWriter<SettledOrder> writer = new ClassifierCompositeItemWriter<>();
    writer.setClassifier(classifier);
    return writer;
}
```

---

## 📊 Writer 성능 비교 (100만 건 기준)

```
MySQL 환경, Chunk Size 1000, 힙 -Xmx512m:

┌────────────────────────────┬──────────────┬────────────────────────────────────┐
│ Writer 설정                 │ 처리 시간      │ 특이사항                              │
├────────────────────────────┼──────────────┼────────────────────────────────────┤
│ JpaItemWriter + IDENTITY   │ ~18분         │ Hibernate 배치 비활성화               │
│ JpaItemWriter + SEQUENCE   │ ~3분          │ flush() 배치 동작                    │
│ JdbcBatchItemWriter        │ ~45초         │ rewriteBatchedStatements=true 필수  │
│ JdbcBatchItemWriter (없음)  │ ~15분         │ rewriteBatchedStatements=false     │
└────────────────────────────┴──────────────┴────────────────────────────────────┘

결론: 성능이 중요하면 JdbcBatchItemWriter + rewriteBatchedStatements=true
```

---

## ⚖️ 트레이드오프

```
JpaItemWriter:

  장점  JPA 편의성 (연관관계, Cascade, 2차 캐시)
        merge()로 재처리 안전
  단점  IDENTITY 전략 시 배치 불가 → SEQUENCE/UUID로 변경 필요
        EntityManager flush + clear 오버헤드
        실수로 N+1 발생할 수 있음 (Lazy Loading 주의)

JdbcBatchItemWriter:

  장점  가장 빠름 (JDBC 배치 + rewriteBatchedStatements)
        IDENTITY 전략 무관하게 배치 동작
        SQL 직접 제어 (ON DUPLICATE KEY, UPSERT 등)
  단점  JPA 연관관계 없음
        SQL 직접 작성 필요

FlatFileItemWriter:

  append + 재시작 포인트 저장으로 재처리 안전
  단점: 파일 I/O, 인코딩 설정 주의 (EUC-KR 환경)
```

---

## 📌 핵심 정리

```
Writer가 List<T>를 받는 이유
  → Chunk 전체를 배치 처리 (JDBC batchUpdate, JPA flush)
  → 단건 API였다면 배치 최적화 불가

JpaItemWriter 핵심
  merge() 기본값 → ID 있으면 UPDATE, 없으면 INSERT (재처리 안전)
  persist()  → 신규 삽입만 (약간 빠름)
  flush() + clear() → Chunk 단위 SQL 실행 + 1차 캐시 초기화

JdbcBatchItemWriter 핵심
  batchUpdate(sql, SqlParameterSource[]) 한 번 호출 = 1000건
  MySQL: rewriteBatchedStatements=true 필수
  assertUpdates=false: ON DUPLICATE KEY UPDATE 등 0건 허용 시

FlatFileItemWriter 재시작 안전성
  append=true: 재시작 시 기존 내용 뒤에 추가
  update(): Chunk 커밋마다 byte 위치 저장
  open(): 재시작 시 이전 byte 위치까지 truncate 후 이어쓰기
```

---

## 🤔 생각해볼 문제

**Q1.** `JpaItemWriter`가 `flush()` 후 `clear()`를 호출합니다. `clear()`를 호출하지 않으면 어떤 문제가 발생하는가? 그리고 `clear()` 이후 같은 트랜잭션 내에서 이미 flush된 엔티티를 다시 조회하면 어떻게 되는가?

**Q2.** `CompositeItemWriter`에서 두 번째 Writer에서 예외가 발생했을 때, 첫 번째 Writer가 이미 쓴 데이터는 어떻게 되는가? 트랜잭션 컨텍스트와의 관계는?

**Q3.** `FlatFileItemWriter`가 Chunk 커밋마다 파일 위치를 바이트 단위로 저장합니다. 멀티 쓰레드 환경(Parallel Flow)에서 두 Step이 같은 파일에 동시에 쓰면 어떤 문제가 발생하는가?

> 💡 **해설**
>
> **Q1.** `clear()` 없이 100만 건을 처리하면 1차 캐시에 모든 엔티티가 누적됩니다. 메모리 사용량이 지속 증가해 결국 OOM이 발생합니다. 또한 Hibernate는 dirty checking을 위해 스냅샷을 유지하므로 메모리 부담이 2배로 늘어납니다. `clear()` 이후 같은 트랜잭션에서 이미 flush된 엔티티를 `find()`로 조회하면, 1차 캐시가 비워졌으므로 DB에서 다시 SELECT합니다. 단, 트랜잭션은 아직 유효하므로 방금 INSERT한 데이터는 조회됩니다.
>
> **Q2.** 두 번째 Writer에서 예외가 발생하면 예외가 `CompositeItemWriter.write()` 밖으로 전파됩니다. `TaskletStep`의 트랜잭션 템플릿이 이를 catch해 전체 트랜잭션을 롤백합니다. 따라서 첫 번째 Writer가 쓴 데이터도 롤백됩니다. 두 Writer가 같은 트랜잭션에 참여하고 있기 때문입니다. 단, 파일 Writer와 DB Writer를 조합할 때는 파일은 트랜잭션 롤백이 안 되므로, DB는 롤백되고 파일은 그대로 남는 불일치가 발생합니다. 이를 위해 `FlatFileItemWriter`는 재시작 시 파일 위치를 복원하는 메커니즘을 제공합니다.
>
> **Q3.** 두 Step이 같은 파일에 동시에 쓰면 파일 내용이 뒤섞입니다(`interleaving`). `FlatFileItemWriter`는 `RandomAccessFile`을 사용해 특정 바이트 위치에 쓰는데, 두 쓰레드가 동시에 `seek() + write()`를 수행하면 순서가 보장되지 않습니다. 해결책: (1) 각 Step이 별도 파일에 씁니다. (2) Parallel Flow가 아닌 Sequential Flow를 사용합니다. (3) 파일 쓰기를 하나의 집계 Step으로 분리하고, 병렬 Step은 DB에 중간 결과를 쓴 후, 마지막 Step에서 DB에서 읽어 하나의 파일로 씁니다.

---

<div align="center">

**[⬅️ 이전: ItemProcessor 체인과 조건부 처리](./03-item-processor-chain.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chunk Size 튜닝 ➡️](./05-chunk-size-tuning.md)**

</div>
