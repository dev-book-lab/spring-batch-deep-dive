# Cursor vs Paging 선택 기준 — DB 커넥션과 OFFSET 성능의 트레이드오프

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Cursor 방식과 Paging 방식은 내부에서 DB와 어떻게 다르게 통신하는가?
- Paging에서 OFFSET이 커질수록 성능이 저하되는 정확한 이유는?
- MySQL에서 Cursor Reader를 스트리밍 모드로 동작시키는 필수 조건은?
- Multi-threaded Step에서 Paging이 더 안전한 이유는?
- No-Offset 패턴은 어떻게 OFFSET 성능 저하를 회피하는가?

---

## 🔍 두 방식의 근본적인 차이

### 문제: "빠른 읽기"와 "안전한 읽기" 중 무엇을 선택할 것인가

```
100만 건을 읽는 두 가지 방식:

  Paging 방식:
    SELECT * FROM orders LIMIT 1000 OFFSET 0      → 1~1000번 행
    SELECT * FROM orders LIMIT 1000 OFFSET 1000   → 1001~2000번 행
    SELECT * FROM orders LIMIT 1000 OFFSET 999000 → 마지막 페이지
    
    특징:
    - 쿼리를 1000번 실행
    - 각 쿼리마다 커넥션 대여 → 반환 (풀 사용)
    - Thread-safe (쿼리 자체가 독립적)
    - 문제: OFFSET 999000 → DB가 100만 행을 스캔 후 마지막 1000개만 반환

  Cursor 방식:
    SELECT * FROM orders   → 한 번 실행, DB에 커서 생성
    rs.next() × 1,000,000 → 한 줄씩 가져오기

    특징:
    - 쿼리를 1번만 실행
    - 커넥션을 Step 전체 동안 유지
    - Thread-unsafe (커서 위치는 공유 불가)
    - OFFSET 없음 → 어느 위치든 일정한 속도
```

---

## 😱 흔한 실수

### Before: MySQL에서 Cursor Reader를 설정했는데 결국 전체 ResultSet이 메모리에 올라온다

```java
// ❌ MySQL에서 fetchSize 설정 없이 Cursor Reader 사용
@Bean
public JdbcCursorItemReader<Order> cursorReader(DataSource dataSource) {
    return new JdbcCursorItemReaderBuilder<Order>()
        .dataSource(dataSource)
        .sql("SELECT * FROM orders")
        .rowMapper(new BeanPropertyRowMapper<>(Order.class))
        // fetchSize 설정 없음!
        .build();
}
// MySQL JDBC 드라이버의 기본 동작:
// fetchSize = 0 (기본값) → 전체 ResultSet을 한 번에 메모리로 로드
// → 100만 건이 한꺼번에 힙에 올라옴 → OOM
// → 커서처럼 보이지만 실제로는 전체 로드!

// ✅ MySQL 스트리밍 모드 활성화
.fetchSize(Integer.MIN_VALUE)   // MySQL 전용 스트리밍 신호
// → 실제로 한 줄씩 네트워크로 가져옴 (진정한 스트리밍)
```

### Before: Paging Reader에서 처리 중 데이터를 업데이트해 데이터가 누락된다

```java
// ❌ 처리 대상 조건이 변경되는 Paging Reader
.queryString("SELECT o FROM Order o WHERE o.status = 'PENDING' ORDER BY o.id")
// + Writer에서 status를 'PROCESSED'로 변경

// 페이지 1: id 1~100 (PENDING) → PROCESSED로 변경
// 페이지 2: PENDING인 것의 OFFSET 100
//         → id 1~100이 이미 PROCESSED로 제외됨
//         → id 201~300이 2페이지 결과로 나옴
//         → id 101~200은 영구 누락!

// ✅ 처리 여부와 무관한 안정적인 정렬 기준 사용
.queryString("SELECT o FROM Order o ORDER BY o.id ASC")
// 또는 처리 전 target_date 기준 별도 컬럼 사용
```

---

## ✨ 올바른 이해와 사용

### 두 방식의 비교표

```
┌────────────────────────┬──────────────────────────┬──────────────────────────┐
│ 항목                    │ Cursor                   │ Paging                   │
├────────────────────────┼──────────────────────────┼──────────────────────────┤
│ DB 쿼리 실행 횟수         │ 1회                       │ (총 건수 / pageSize)회     │
│ DB 커넥션 유지            │ Step 전체 동안             │ 쿼리당 잠깐                 │
│ Thread-safety          │ ❌ (커서 공유 불가)         │ ✅ (독립 쿼리)             │
│ OFFSET 성능 저하         │ 없음                      │ 있음 (페이지 깊이 비례)       │
│ 처리 중 데이터 변경        │ 안전 (커서 시점 스냅샷)       │ 위험 (누락 발생)             │
│ 메모리 사용              │ fetchSize만큼 버퍼         │ pageSize만큼               │
│ 재시작                  │ ✅                       │ ✅                       │
│ MySQL 스트리밍 설정       │ 필수 (MIN_VALUE)          │ 불필요                     │
└────────────────────────┴──────────────────────────┴──────────────────────────┘
```

---

## 🔬 내부 동작 원리

### 1. Paging OFFSET 성능 저하 — 왜 느린가

```sql
-- OFFSET 성능 측정 (MySQL, orders 테이블 100만 건)

-- 첫 번째 페이지: 빠름
SELECT * FROM orders ORDER BY id LIMIT 1000 OFFSET 0;
-- 실행 계획: index range scan → 1000건 반환
-- 실행 시간: ~2ms

-- 중간 페이지: 느림
SELECT * FROM orders ORDER BY id LIMIT 1000 OFFSET 500000;
-- 실행 계획: index scan 500,000행 → 건너뜀 → 다음 1,000건 반환
-- 실행 시간: ~800ms

-- 마지막 페이지: 매우 느림
SELECT * FROM orders ORDER BY id LIMIT 1000 OFFSET 999000;
-- 실행 계획: index scan 999,000행 → 건너뜀 → 마지막 1,000건 반환
-- 실행 시간: ~1,500ms

-- 총 100만 건 읽기 시간:
-- Paging: 평균 200ms × 1,000페이지 = ~200,000ms = 200초
-- Cursor: ~40초 (OFFSET 없음)
```

### 2. MySQL Cursor 스트리밍 — fetchSize=Integer.MIN_VALUE

```java
// MySQL JDBC 드라이버 내부 동작
// com.mysql.cj.jdbc.StatementImpl.setFetchSize()

public void setFetchSize(int rows) throws SQLException {
    if (rows == Integer.MIN_VALUE) {
        // 특수 신호: 스트리밍 모드 활성화
        // → 결과를 한 번에 로드하지 않고
        //   rs.next() 호출 시 한 행씩 네트워크로 가져옴
        this.resultSetType = ResultSet.TYPE_FORWARD_ONLY;
        this.streamResults = true;
    } else if (rows > 0) {
        // N행씩 버퍼링 (Oracle, PostgreSQL에서 효과적)
        // MySQL에서는 N>0이면 전체 로드 후 클라이언트에서 N개씩 반환
        this.fetchSize = rows;
    }
    // rows == 0 → 기본값: 전체 결과를 한 번에 로드
}

// 결론:
// MySQL: fetchSize=Integer.MIN_VALUE → 진짜 스트리밍
//        fetchSize=1000             → 여전히 전체 로드!
// PostgreSQL: fetchSize=1000        → 진짜 1000행씩 가져옴 (MySQL과 다름)
```

### 3. No-Offset 패턴 — Paging의 OFFSET 없애기

```java
// 문제: OFFSET이 커질수록 느림
// 해결: 마지막 처리한 ID를 기억해 WHERE id > lastId 로 처리

// 방법 1: JpaPagingItemReader + ItemReadListener
@Component
public class NoOffsetOrderReader implements ItemStreamReader<Order> {

    private Long lastId = 0L;
    private final JpaRepository<Order, Long> orderRepo;
    private Iterator<Order> currentPage;
    private static final int PAGE_SIZE = 1000;

    @Override
    public Order read() throws Exception {
        if (currentPage == null || !currentPage.hasNext()) {
            // 다음 페이지 로드 (lastId 기준)
            List<Order> page = orderRepo.findByIdGreaterThanAndStatus(
                lastId, "PENDING",
                PageRequest.of(0, PAGE_SIZE, Sort.by("id").ascending())
            );
            if (page.isEmpty()) return null;  // 데이터 소진
            currentPage = page.iterator();
            lastId = page.get(page.size() - 1).getId();
        }
        return currentPage.next();
    }

    @Override
    public void update(ExecutionContext executionContext) {
        executionContext.putLong("lastId", lastId);  // 재시작 포인트
    }

    @Override
    public void open(ExecutionContext executionContext) {
        if (executionContext.containsKey("lastId")) {
            lastId = executionContext.getLong("lastId");  // 재시작 복원
        }
    }
}
```

### 4. Multi-threaded Step에서의 안전한 Paging 사용

```java
// PagingItemReader를 Multi-threaded Step에서 안전하게 사용
@Bean
@StepScope
public SynchronizedItemStreamReader<Order> synchronizedPagingReader(
        EntityManagerFactory emf) {

    // JpaPagingItemReader는 내부 상태(page, results)를 가짐
    // → Multi-threaded 환경에서 동기화 필요
    JpaPagingItemReader<Order> delegate = new JpaPagingItemReaderBuilder<Order>()
        .name("orderReader")
        .entityManagerFactory(emf)
        .queryString("SELECT o FROM Order o ORDER BY o.id")
        .pageSize(1000)
        .build();

    SynchronizedItemStreamReader<Order> reader = new SynchronizedItemStreamReader<>();
    reader.setDelegate(delegate);
    // synchronized read() → 한 번에 한 쓰레드만 페이지 로드
    return reader;
}

@Bean
public Step multiThreadedStep() {
    return stepBuilderFactory.get("multiThreadedStep")
        .<Order, SettledOrder>chunk(1000)
        .reader(synchronizedPagingReader(emf))
        .processor(processor())
        .writer(writer())
        .taskExecutor(new SimpleAsyncTaskExecutor())
        .throttleLimit(4)      // 동시 쓰레드 수 제한 (DB 커넥션 풀 고려)
        .build();
}
```

### 5. Cursor 커넥션 유지로 인한 커넥션 풀 계산

```
커넥션 풀 최소 크기 계산:

  동시 실행 Step 수: N개
  각 Step이 Cursor Reader를 사용할 경우:
    → N개의 커넥션을 Step 전체 동안 점유

  각 Chunk Write 시 필요한 커넥션: 트랜잭션 내 1개
  → Step당: 1 (Cursor) + 1 (Write) = 2개 커넥션 필요

  단, JdbcCursorItemReader와 Writer가 같은 DataSource + 같은 트랜잭션이면
  커넥션을 공유할 수 있음 (트랜잭션 동기화)

  안전한 풀 크기:
    pool_size ≥ (동시 Step 수) × 2 + 여유분

  예: 4개 Step 동시 실행 + Cursor Reader
    pool_size ≥ 4 × 2 + 2 = 10개
```

---

## 💻 실전 구현

### 상황별 Reader 선택 + 완전한 설정

```java
// 상황 1: 단일 쓰레드, 대용량, 처리 중 데이터 변경 없음
//          → JdbcCursorItemReader (추천)
@Bean
@StepScope
public JdbcCursorItemReader<Order> bestPerformanceReader(DataSource dataSource) {
    return new JdbcCursorItemReaderBuilder<Order>()
        .name("singleThreadCursorReader")
        .dataSource(dataSource)
        .sql("SELECT id, amount, customer_id FROM orders WHERE status='PENDING' ORDER BY id")
        .rowMapper(new BeanPropertyRowMapper<>(Order.class))
        .fetchSize(Integer.MIN_VALUE)      // MySQL 스트리밍 필수
        .saveState(true)                   // 재시작 가능
        .build();
}

// 상황 2: Multi-threaded Step
//          → JpaPagingItemReader + SynchronizedItemStreamReader
@Bean
@StepScope
public SynchronizedItemStreamReader<Order> threadSafeReader(EntityManagerFactory emf) {
    JpaPagingItemReader<Order> pagingReader = new JpaPagingItemReaderBuilder<Order>()
        .name("multiThreadPagingReader")
        .entityManagerFactory(emf)
        .queryString("SELECT o FROM Order o WHERE o.status='PENDING' ORDER BY o.id")
        .pageSize(1000)
        .saveState(false)   // Multi-thread 환경에서 EC 저장 비활성화 (Partitioning 권장)
        .build();

    SynchronizedItemStreamReader<Order> syncReader = new SynchronizedItemStreamReader<>();
    syncReader.setDelegate(pagingReader);
    return syncReader;
}

// 상황 3: No-Offset Paging (대용량 + 처리 중 데이터 변경)
@Bean
@StepScope
public JdbcPagingItemReader<Order> noOffsetReader(DataSource dataSource) {
    Map<String, Order> sortKeys = new LinkedHashMap<>();
    sortKeys.put("id", Order.class);  // 정렬 키

    MySqlPagingQueryProvider provider = new MySqlPagingQueryProvider();
    provider.setSelectClause("SELECT id, amount, customer_id, status");
    provider.setFromClause("FROM orders");
    provider.setWhereClause("WHERE status = 'PENDING'");
    provider.setSortKeys(Map.of("id", Order.class));
    // → JdbcPagingItemReader가 내부적으로 id > lastId 쿼리 생성

    return new JdbcPagingItemReaderBuilder<Order>()
        .name("noOffsetReader")
        .dataSource(dataSource)
        .queryProvider(provider)
        .rowMapper(new BeanPropertyRowMapper<>(Order.class))
        .pageSize(1000)
        .build();
}
```

---

## 📊 100만 건 읽기 성능 비교

```
MySQL 8.0, orders 테이블 100만 건, pageSize/fetchSize=1000:

┌──────────────────────────────────┬──────────────┬───────────────────────────────┐
│ Reader                           │ 읽기 시간      │ 특이사항                         │
├──────────────────────────────────┼──────────────┼───────────────────────────────┤
│ JdbcCursorItemReader (스트리밍)    │ ~30초         │ fetchSize=MIN_VALUE 필수       │
│ JdbcCursorItemReader (전체 로드)   │ OOM/느림       │ fetchSize 미설정 시             │
│ JpaPagingItemReader              │ ~200초        │ OFFSET 성능 저하                │
│ JdbcPagingItemReader (No-Offset) │ ~60초         │ id > lastId 방식               │
│ JpaPagingItemReader (No-Offset)  │ ~70초         │ Custom 구현 필요                │
└──────────────────────────────────┴──────────────┴───────────────────────────────┘

결론: 단일 쓰레드 대용량 → JdbcCursorItemReader
      OFFSET 회피 필요   → JdbcPagingItemReader (No-Offset)
```

---

## ⚖️ 트레이드오프

```
Cursor:

  장점  빠름 (쿼리 1번, OFFSET 없음)
        처리 중 데이터 변경에 안전 (커서 시점 스냅샷)
        단순한 재시작 (read.count 기반)
  단점  DB 커넥션 Step 전체 점유 → 풀 고갈 위험
        Thread-unsafe → Multi-threaded Step 사용 불가
        Long-running 시 커넥션 타임아웃 위험
        MySQL fetchSize=MIN_VALUE 설정 필수 (잊으면 OOM)

Paging:

  장점  커넥션 짧게 사용 (풀 효율적)
        Thread-safe (SynchronizedItemStreamReader 래핑 시)
        안정적 (각 페이지가 독립 트랜잭션)
  단점  OFFSET 성능 저하 (깊은 페이지)
        처리 중 데이터 변경 시 누락 위험
        페이지마다 쿼리 재실행 오버헤드
        No-Offset 패턴으로 OFFSET 해결 가능 (복잡도 증가)
```

---

## 📌 핵심 정리

```
Cursor vs Paging 선택 기준

  Cursor를 선택하는 경우
    단일 쓰레드 Step + 대용량 + 처리 중 상태 변경 없음
    커넥션 풀이 충분한 환경
    MySQL fetchSize=Integer.MIN_VALUE 설정 가능

  Paging을 선택하는 경우
    Multi-threaded Step (SynchronizedItemStreamReader 래핑)
    처리 중 데이터 상태 변경이 있는 경우
    커넥션 풀이 제한적인 환경
    Partitioning (각 Worker가 독립적 페이지 범위)

MySQL 스트리밍 필수 설정
  fetchSize = Integer.MIN_VALUE
  → 없으면 전체 ResultSet을 메모리에 로드 → OOM

OFFSET 성능 저하 해결
  JdbcPagingItemReader: MySqlPagingQueryProvider가 내부적으로
                        id > lastId 방식 사용 (No-Offset)
  직접 구현: WHERE id > :lastId ORDER BY id LIMIT pageSize
```

---

## 🤔 생각해볼 문제

**Q1.** `JdbcCursorItemReader`가 MySQL에서 Step 전체 동안 커넥션을 유지할 때, 이 커넥션에서 실행된 쿼리의 트랜잭션 격리 수준은 어떻게 되는가? Chunk 트랜잭션과 커서 커넥션의 트랜잭션은 같은 것인가?

**Q2.** `JdbcPagingItemReader`의 `MySqlPagingQueryProvider`가 내부적으로 No-Offset 쿼리를 생성합니다. 이 Provider가 생성하는 실제 SQL 쿼리는 어떤 형태인가? 첫 번째 페이지와 두 번째 페이지의 쿼리는 어떻게 다른가?

**Q3.** `SynchronizedItemStreamReader`로 `JpaPagingItemReader`를 래핑하면 `read()` 메서드가 `synchronized`됩니다. 4개의 쓰레드가 동시에 `read()`를 호출하면 실제로는 순차적으로 실행됩니다. 이러면 Multi-threaded Step의 병렬 처리 이점이 무엇인가?

> 💡 **해설**
>
> **Q1.** `JdbcCursorItemReader`의 커넥션과 Chunk 트랜잭션의 커넥션은 **다릅니다**. Spring의 `DataSourceTransactionManager`는 트랜잭션 시작 시 별도의 커넥션을 풀에서 가져오고, 커밋/롤백 후 반환합니다. 반면 Cursor Reader는 `open()` 시점에 별도 커넥션을 가져와 `close()` 시까지 유지합니다. 두 커넥션은 독립적입니다. 단, 같은 `DataSource`를 사용할 때 `DataSourceUtils.getConnection(dataSource)`을 사용하면 현재 트랜잭션의 커넥션을 재사용하려 합니다 — `AbstractCursorItemReader`는 `ConnectionHolder`를 통해 이 동작을 제어합니다.
>
> **Q2.** `MySqlPagingQueryProvider`가 생성하는 쿼리: 첫 번째 페이지 — `SELECT id, amount FROM orders WHERE (status='PENDING') ORDER BY id ASC LIMIT 1000`. 두 번째 페이지 — `SELECT id, amount FROM orders WHERE (status='PENDING') AND ((id > ?)) ORDER BY id ASC LIMIT 1000` (여기서 `?`는 직전 페이지의 마지막 `id` 값). 즉, OFFSET 없이 이전 페이지의 마지막 정렬 키 값보다 큰 조건을 WHERE에 추가합니다. 이 방식으로 OFFSET 스캔 없이 인덱스 range scan으로 처리됩니다.
>
> **Q3.** `read()`가 동기화됐어도 Multi-threaded Step의 이점은 **Process와 Write 단계**에 있습니다. 각 쓰레드는 독립적인 Chunk를 처리합니다. 쓰레드 1이 Chunk 1의 Process/Write를 하는 동안, 쓰레드 2는 `read()`로 Chunk 2를 가져오고, 쓰레드 3은 Chunk 3의 Process를 합니다. 즉, `read()`는 병목이지만 Process와 Write는 진정으로 병렬 실행됩니다. `Processor`가 외부 API 호출처럼 느린 경우에 Multi-threaded Step의 효과가 극대화됩니다. 순수 DB 읽기/쓰기의 경우 Partitioning이 더 효과적입니다.

---

<div align="center">

**[⬅️ 이전: Chunk Size 튜닝](./05-chunk-size-tuning.md)** | **[홈으로 🏠](../README.md)** | **[다음: Custom ItemReader·ItemWriter 작성 ➡️](./07-custom-reader-writer.md)**

</div>
