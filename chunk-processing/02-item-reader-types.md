# ItemReader 종류와 구현 — JpaPaging vs JdbcCursor vs FlatFile

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `JpaPagingItemReader`와 `JdbcCursorItemReader`는 내부적으로 어떻게 다른가?
- Paging Reader에서 페이지마다 쿼리가 재실행될 때 발생하는 데이터 누락 문제는?
- `JdbcCursorItemReader`가 하나의 DB 커넥션을 Step 내내 유지하는 이유는?
- `FlatFileItemReader`는 라인을 어떻게 파싱해 객체로 변환하는가?
- Multi-threaded Step에서 어느 Reader가 안전하고 어느 것이 위험한가?

---

## 🔍 왜 ItemReader 종류가 다양한가

### 문제: 데이터 소스의 특성이 다르면 읽기 전략도 달라야 한다

```
데이터 소스별 읽기 전략의 차이:

  DB — 페이징 방식 (Paging Reader)
    SELECT * FROM orders LIMIT 1000 OFFSET 0
    SELECT * FROM orders LIMIT 1000 OFFSET 1000
    → 커넥션을 잠깐씩 사용, 쿼리를 반복 실행
    → Thread-safe (각 페이지마다 독립적 쿼리)
    → 단점: OFFSET이 커질수록 느려짐

  DB — 커서 방식 (Cursor Reader)
    SELECT * FROM orders  ← 한 번만 실행
    → DB 커서로 한 줄씩 fetch
    → 커넥션을 Step 전체 동안 유지
    → Thread-unsafe (커서 위치는 공유 불가)
    → 장점: OFFSET 성능 저하 없음

  파일 — 순차 읽기 (FlatFile Reader)
    라인 단위로 읽어 객체로 변환
    → 파일 포인터를 유지
    → 재시작 시 마지막 읽은 라인부터 재개

  메시지 큐 — 이벤트 수신 (Custom Reader)
    Kafka Consumer, RabbitMQ Consumer 등
    → 폴링 기반, 타임아웃 처리 필요
```

---

## 😱 흔한 실수

### Before: JpaPagingItemReader 사용 중 데이터가 누락된다

```java
// ❌ 처리 중 상태를 변경하는 Reader — 데이터 누락 발생!
@Bean
@StepScope
public JpaPagingItemReader<Order> orderReader(EntityManagerFactory emf) {
    return new JpaPagingItemReaderBuilder<Order>()
        .name("orderReader")
        .entityManagerFactory(emf)
        // status = 'PENDING'인 주문을 읽어서 처리
        .queryString("SELECT o FROM Order o WHERE o.status = 'PENDING' ORDER BY o.id")
        .pageSize(1000)
        .build();
}

// Writer에서:
// order.setStatus("PROCESSED");  // 이 아이템이 다음 페이지 쿼리에서 사라짐!

// 문제:
// 페이지 1: PENDING 1000건 (id 1~1000) → PROCESSED로 변경
// 페이지 2: PENDING 1000건 쿼리 → id 1~1000이 이미 PROCESSED로 바뀜
//           → 실제로는 id 1001~2000 대신 id 2001~3000이 결과에 나옴
//           → id 1001~2000은 영구적으로 처리되지 않음!

// ✅ 올바른 방법: 처리 여부와 무관한 정렬 기준 사용
.queryString("SELECT o FROM Order o WHERE o.status = 'PENDING' ORDER BY o.id ASC")
// + Writer는 별도 UPDATE 쿼리로 상태 변경 (동일 트랜잭션 내)
// 또는 JdbcCursorItemReader 사용 (커서는 변경 영향 없음)
```

### Before: Multi-threaded Step에서 JdbcCursorItemReader를 사용한다

```java
// ❌ Multi-threaded Step에서 Cursor Reader — Thread-unsafe!
@Bean
public Step multiThreadedStep() {
    return stepBuilderFactory.get("multiThreadedStep")
        .<Order, SettledOrder>chunk(1000)
        .reader(jdbcCursorReader())          // ❌ 커서는 공유 불가
        .processor(processor())
        .writer(writer())
        .taskExecutor(new SimpleAsyncTaskExecutor())  // 멀티 쓰레드
        .build();
}
// 여러 쓰레드가 같은 커서를 동시에 forward → 데이터 중복/누락

// ✅ Multi-threaded Step에서는 Paging Reader 사용
.reader(synchronizedPagingReader())  // SynchronizedItemStreamReader로 래핑
```

---

## ✨ 올바른 이해와 사용

### 주요 ItemReader 비교

```
Reader 종류와 특성:

┌──────────────────────────┬──────────────┬────────────┬───────────┬───────────────┐
│ Reader                   │ DB 연결 유지   │ Thread-safe│ 재시작      │ OFFSET 성능    │
├──────────────────────────┼──────────────┼────────────┼───────────┼───────────────┤
│ JpaPagingItemReader      │ 쿼리당 잠깐     │ ✅ (래핑 시)│ ✅        │ ❌ 느려짐       │
│ JdbcPagingItemReader     │ 쿼리당 잠깐     │ ✅ (래핑 시)│ ✅        │ 중간           │
│ JdbcCursorItemReader     │ Step 내내     │ ❌         │ ✅        │ ✅ 빠름        │
│ FlatFileItemReader       │ N/A (파일)    │ ❌         │ ✅        │ N/A           │
│ StaxEventItemReader      │ N/A (XML)    │ ❌         │ ✅        │ N/A           │
└──────────────────────────┴──────────────┴────────────┴───────────┴───────────────┘
```

---

## 🔬 내부 동작 원리

### 1. JpaPagingItemReader — 페이지마다 쿼리 재실행

```java
// JpaPagingItemReader.java
public class JpaPagingItemReader<T> extends AbstractPagingItemReader<T> {

    private EntityManagerFactory entityManagerFactory;
    private EntityManager entityManager;
    private String queryString;
    private Map<String, Object> parameterValues;

    @Override
    protected void doReadPage() {
        // ① 이전 EntityManager 닫기 (페이지마다 새로 생성 — 영속성 컨텍스트 초기화)
        if (entityManager != null) {
            entityManager.close();
        }
        entityManager = entityManagerFactory.createEntityManager();

        // ② 페이지 쿼리 실행
        TypedQuery<T> query = entityManager.createQuery(queryString, type);

        if (parameterValues != null) {
            parameterValues.forEach(query::setParameter);
        }

        // ③ OFFSET 적용 (페이지 × 페이지 크기)
        query.setFirstResult(getPage() * getPageSize());
        query.setMaxResults(getPageSize());

        results = new CopyOnWriteArrayList<>(query.getResultList());
        // ← 이 페이지의 모든 결과를 리스트에 저장
    }

    // AbstractPagingItemReader.doRead() — 페이지 내 아이템을 순서대로 반환
    @Override
    protected T doRead() throws Exception {
        if (results == null || current >= pageSize) {
            // 현재 페이지 소진 → 다음 페이지 로드
            doReadPage();
        }
        return results.isEmpty() ? null : results.get(current++);
    }
}
```

### 2. JdbcCursorItemReader — 커서로 한 줄씩 fetch

```java
// JdbcCursorItemReader.java
public class JdbcCursorItemReader<T> extends AbstractCursorItemReader<T> {

    @Override
    protected void openCursor(Connection con) {
        // ① 한 번만 쿼리 실행 → DB 커서 생성
        preparedStatement = con.prepareStatement(sql,
            ResultSet.TYPE_FORWARD_ONLY,     // 앞으로만 이동 (커서 최적화)
            ResultSet.CONCUR_READ_ONLY);

        // ② MySQL 스트리밍 핵심 설정
        // fetchSize = Integer.MIN_VALUE → MySQL에게 스트리밍 모드 신호
        preparedStatement.setFetchSize(fetchSize);

        rs = preparedStatement.executeQuery();
        // ← 쿼리 실행 완료, 결과는 DB에 커서로 유지
    }

    @Override
    protected T readCursor(ResultSet rs, int currentRow) throws SQLException {
        // ③ 한 행씩 fetch (ResultSet.next())
        return rowMapper.mapRow(rs, currentRow);
        // 매 read() 호출 = rs.next() + rowMapper 변환
    }

    // Step 종료 시 커넥션 반환
    @Override
    public void close() {
        // ResultSet, Statement, Connection 순서로 닫음
        JdbcUtils.closeResultSet(rs);
        JdbcUtils.closeStatement(preparedStatement);
        DataSourceUtils.releaseConnection(con, dataSource);
    }
}
```

### 3. MySQL JdbcCursorItemReader 스트리밍 설정

```java
// MySQL에서 스트리밍(커서) 읽기를 위한 필수 설정
@Bean
@StepScope
public JdbcCursorItemReader<Order> cursorReader(DataSource dataSource) {
    return new JdbcCursorItemReaderBuilder<Order>()
        .name("orderCursorReader")
        .dataSource(dataSource)
        .sql("SELECT id, amount, status, order_date FROM orders WHERE status = 'PENDING' ORDER BY id")
        .rowMapper(new OrderRowMapper())

        // ← MySQL 스트리밍 핵심 3가지
        .fetchSize(Integer.MIN_VALUE)          // ① MySQL 스트리밍 모드 활성화
        .driverSupportsAbsolute(false)         // ② 커서 절대 위치 이동 비활성화
        .verifyCursorPosition(false)           // ③ 커서 위치 검증 비활성화

        // ← 커넥션 설정 (스트리밍 중 커넥션 유지)
        // DataSource의 connectionTimeout을 넉넉하게 설정 필요
        // hikari.connection-timeout: 3600000 (1시간)
        .build();
}
```

### 4. FlatFileItemReader — 라인 파싱 전략

```java
// FlatFileItemReader.java — 라인 읽기 + LineMapper 위임
public class FlatFileItemReader<T> extends AbstractItemCountingItemStreamItemReader<T>
        implements ResourceAwareItemReaderItemStream<T> {

    private BufferedReaderFactory bufferedReaderFactory;
    private LineMapper<T> lineMapper;
    private RecordSeparatorPolicy recordSeparatorPolicy;

    @Override
    protected T doRead() throws Exception {
        if (noInput) return null;

        // ① 라인 읽기
        String line = readLine();  // BufferedReader.readLine()
        if (line == null) return null;  // EOF → 데이터 소진

        // ② 여러 줄에 걸친 레코드 처리 (RecordSeparatorPolicy)
        while (recordSeparatorPolicy.isEndOfRecord(line)) {
            line += readLine();
        }

        // ③ 라인 → 객체 변환 (LineMapper)
        return lineMapper.mapLine(line, lineNumber);
    }

    // LineMapper 구현 종류:
    // DefaultLineMapper = LineTokenizer + FieldSetMapper 조합
    // BeanWrapperFieldSetMapper: 필드명 기반 자동 매핑
    // PassThroughLineMapper: 라인 문자열 그대로 반환
}

// CSV 파싱 설정 예시
@Bean
@StepScope
public FlatFileItemReader<Order> csvReader(
        @Value("#{jobParameters['inputFile']}") String inputFile) {

    return new FlatFileItemReaderBuilder<Order>()
        .name("csvReader")
        .resource(new FileSystemResource(inputFile))
        .linesToSkip(1)                    // 헤더 줄 건너뜀
        .delimited()                       // DelimitedLineTokenizer
            .delimiter(",")
            .names("orderId", "amount", "status", "orderDate")
        .targetType(Order.class)           // BeanWrapperFieldSetMapper
        .build();
}
```

### 5. SynchronizedItemStreamReader — Thread-safe 래핑

```java
// Multi-threaded Step에서 PagingReader를 안전하게 사용
@Bean
@StepScope
public SynchronizedItemStreamReader<Order> synchronizedReader() {
    // JpaPagingItemReader를 synchronized로 래핑
    // read() 메서드에 동기화 적용
    SynchronizedItemStreamReader<Order> reader = new SynchronizedItemStreamReader<>();
    reader.setDelegate(orderPagingReader());  // 내부 Reader
    return reader;
}

// SynchronizedItemStreamReader 내부 구조
public class SynchronizedItemStreamReader<T>
        implements ItemStreamReader<T>, InitializingBean {

    private ItemStreamReader<T> delegate;

    @Override
    public synchronized T read() throws Exception {
        // synchronized: 한 번에 하나의 쓰레드만 read() 진입
        return delegate.read();
    }

    // open/update/close는 delegate에 위임
}
```

### 6. AbstractItemCountingItemStreamItemReader — 재시작 포인트

```java
// 대부분의 Reader가 상속하는 추상 클래스
// ExecutionContext에 읽기 카운트를 저장해 재시작 지원

public abstract class AbstractItemCountingItemStreamItemReader<T>
        extends AbstractItemStreamItemReader<T> {

    private int currentItemCount = 0;  // 현재까지 읽은 건수
    private int maxItemCount = Integer.MAX_VALUE;

    // ExecutionContext 저장 (Chunk 커밋마다 호출)
    @Override
    public void update(ExecutionContext executionContext) {
        super.update(executionContext);
        if (isSaveState()) {
            executionContext.putInt(getExecutionContextKey(READ_COUNT), currentItemCount);
            // → BATCH_STEP_EXECUTION_CONTEXT에 저장
            // 예: {"orderReader.read.count": 5000}
        }
    }

    // 재시작 시 이전 위치 복원
    @Override
    public void open(ExecutionContext executionContext) {
        super.open(executionContext);
        if (isSaveState()
                && executionContext.containsKey(getExecutionContextKey(READ_COUNT))) {
            int itemCount = executionContext.getInt(getExecutionContextKey(READ_COUNT));
            // 이전에 읽은 위치까지 skip
            jumpToItem(itemCount);  // 각 Reader마다 구현 방식 다름
            currentItemCount = itemCount;
        }
    }
}
```

---

## 💻 실전 구현

### 대용량 처리를 위한 JdbcCursorItemReader 최적화

```java
@Bean
@StepScope
public JdbcCursorItemReader<Order> optimizedCursorReader(DataSource dataSource) {
    return new JdbcCursorItemReaderBuilder<Order>()
        .name("optimizedOrderReader")
        .dataSource(dataSource)
        .sql("""
            SELECT o.id, o.amount, o.status, o.customer_id,
                   c.name as customer_name
            FROM orders o
            JOIN customers c ON o.customer_id = c.id
            WHERE o.order_date = ?
              AND o.status = 'PENDING'
            ORDER BY o.id ASC
            """)
        .preparedStatementSetter(ps -> ps.setDate(1,
            Date.valueOf(LocalDate.now().minusDays(1))))
        .rowMapper((rs, rowNum) -> Order.builder()
            .id(rs.getLong("id"))
            .amount(rs.getBigDecimal("amount"))
            .customerName(rs.getString("customer_name"))
            .build())
        .fetchSize(Integer.MIN_VALUE)     // MySQL 스트리밍
        .saveState(true)                  // 재시작 가능
        .build();
}
```

### JpaPagingItemReader — OFFSET 성능 문제 우회 (No-Offset 패턴)

```java
// ❌ 기본 OFFSET 방식 — 데이터가 많을수록 느려짐
// SELECT * FROM orders LIMIT 1000 OFFSET 999000  ← 100만 번째 페이지

// ✅ No-Offset 방식 — 마지막 처리 ID 기반 페이징
@Bean
@StepScope
public JpaPagingItemReader<Order> noOffsetReader(EntityManagerFactory emf) {
    Map<String, Object> params = new HashMap<>();
    params.put("lastId", 0L);  // 첫 페이지는 0부터

    return new JpaPagingItemReaderBuilder<Order>()
        .name("noOffsetOrderReader")
        .entityManagerFactory(emf)
        // id > :lastId 조건으로 OFFSET 없이 페이징
        .queryString("""
            SELECT o FROM Order o
            WHERE o.id > :lastId
              AND o.status = 'PENDING'
            ORDER BY o.id ASC
            """)
        .parameterValues(params)
        .pageSize(1000)
        .build();
    // 단, 페이지가 넘어갈 때 :lastId를 업데이트해야 함
    // → ItemReadListener 또는 커스텀 Reader로 구현
}
```

---

## 📊 Reader 선택 가이드

```
상황별 Reader 선택:

  단일 쓰레드 + 대용량 + 처리 중 데이터 변경 없음
    → JdbcCursorItemReader (OFFSET 없이 빠름, 커넥션 유지 감수)

  단일 쓰레드 + 처리 중 데이터 상태 변경
    → JpaPagingItemReader (No-Offset 패턴 적용)

  Multi-threaded Step
    → JpaPagingItemReader + SynchronizedItemStreamReader 래핑
    → 또는 Partitioning (각 Worker가 독립 범위 처리)

  CSV/Excel 파일 처리
    → FlatFileItemReader (CSV), PoiItemReader (Excel)

  XML 처리
    → StaxEventItemReader

  Kafka 메시지 처리
    → Custom ItemReader (Ch2-07 참조)
```

---

## ⚖️ 트레이드오프

```
JpaPagingItemReader:

  장점  Thread-safe (SynchronizedItemStreamReader 래핑 시)
        JPA 영속성 컨텍스트 지원 (Lazy Loading 가능)
        커넥션 점유 없음 (페이지 처리 후 반환)
  단점  페이지마다 EntityManager 재생성 (비용)
        OFFSET 성능 저하 (수백만 페이지 이후)
        처리 중 데이터 변경 시 누락 위험

JdbcCursorItemReader:

  장점  한 번의 쿼리 실행 (OFFSET 없음)
        메모리 효율 (fetchSize만큼만 버퍼)
        처리 중 데이터 변경에 영향 없음 (커서 시점 스냅샷)
  단점  Step 내내 DB 커넥션 점유
        Thread-unsafe (Multi-threaded Step 사용 불가)
        Long-running 쿼리 시 DB 타임아웃 위험
```

---

## 📌 핵심 정리

```
Paging vs Cursor 핵심 차이

  Paging (JpaPaging, JdbcPaging)
    페이지마다 새 쿼리 + 새 커넥션
    Thread-safe (SynchronizedItemStreamReader 래핑 시)
    단점: OFFSET → 깊은 페이지일수록 느림

  Cursor (JdbcCursor)
    한 번 쿼리 → DB 커서로 한 줄씩 fetch
    Thread-unsafe (커서 위치 공유 불가)
    MySQL: fetchSize=Integer.MIN_VALUE 필수 (스트리밍 활성화)
    단점: DB 커넥션 점유 (Step 전체 동안)

재시작 포인트 저장
  AbstractItemCountingItemStreamItemReader.update()
  → ExecutionContext에 "readerId.read.count" 저장
  → Chunk 커밋마다 갱신
  → 재시작 시 open()에서 복원 → jumpToItem()으로 이전 위치 이동

Multi-threaded Step Reader 선택
  PagingReader + SynchronizedItemStreamReader 래핑 ← 안전
  CursorReader ← 위험 (커서 공유 불가)
```

---

## 🤔 생각해볼 문제

**Q1.** `JpaPagingItemReader`는 페이지마다 새 `EntityManager`를 생성합니다. 이는 JPA 1차 캐시가 페이지마다 초기화된다는 의미입니다. 이것이 왜 의도된 설계인가? 캐시가 누적되면 어떤 문제가 생기는가?

**Q2.** `JdbcCursorItemReader`가 Step 전체 동안 DB 커넥션을 점유합니다. HikariCP 커넥션 풀 크기가 10이고, 동시에 10개의 Step이 실행되면 어떤 문제가 발생하는가?

**Q3.** `FlatFileItemReader`가 재시작될 때 `jumpToItem(5000)`을 호출합니다. 이는 내부적으로 어떻게 구현되는가? 5000번째 줄로 파일 포인터를 직접 이동시키는가?

> 💡 **해설**
>
> **Q1.** JPA 1차 캐시(영속성 컨텍스트)에 처리된 엔티티가 계속 쌓이면 메모리 사용량이 선형으로 증가합니다. 100만 건 처리 시 100만 개의 엔티티 객체가 캐시에 남아 OOM이 발생합니다. 페이지마다 `EntityManager`를 재생성하는 것은 이를 방지하기 위한 의도된 설계입니다. 1차 캐시를 매 페이지마다 초기화해 메모리를 일정하게 유지합니다. 대신 Lazy Loading은 페이지 내에서만 동작하고, 페이지가 바뀌면 새 `EntityManager`로 다시 로드해야 합니다.
>
> **Q2.** HikariCP 풀 크기 10 + 동시 10개 Step이 각각 `JdbcCursorItemReader`를 사용하면, 커넥션이 모두 소진됩니다. 추가 커넥션 요청(Writer의 INSERT 등)은 `connectionTimeout`까지 대기 후 `SQLTimeoutException`이 발생합니다. Cursor Reader는 Step 동안 커넥션 1개를 독점하므로, N개의 동시 Step = N개의 커넥션이 상시 필요합니다. 해결책: 커넥션 풀 크기를 `(동시 실행 Step 수 × 2) + 여유분`으로 설정하거나, Paging Reader로 교체합니다.
>
> **Q3.** `FlatFileItemReader.jumpToItem(5000)`은 파일 포인터를 직접 5000번째 줄로 이동시키지 않습니다. 대신 5000번 `doRead()`를 반복 호출해 5000번째 줄까지 순차적으로 읽고 버립니다(`restartLineCount`까지 읽기). 파일 형식이 고정 길이가 아닌 이상 줄 오프셋을 미리 알 수 없기 때문입니다. 이는 재시작 시 느릴 수 있습니다. 성능 개선을 원한다면 Custom Reader에서 `ExecutionContext`에 바이트 오프셋을 저장하고 `FileInputStream`의 `skip()`으로 직접 이동하는 방법을 사용할 수 있습니다.

---

<div align="center">

**[⬅️ 이전: Chunk 처리 원리](./01-chunk-oriented-processing.md)** | **[홈으로 🏠](../README.md)** | **[다음: ItemProcessor 체인과 조건부 처리 ➡️](./03-item-processor-chain.md)**

</div>
