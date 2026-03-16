# Custom ItemReader·ItemWriter 작성 — ItemStream으로 재시작 가능한 컴포넌트 구현

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `ItemStream`의 `open·update·close` 세 메서드는 각각 언제 호출되는가?
- `ExecutionContext`에 재시작 포인트를 저장하고 복원하는 정확한 구현 방법은?
- Kafka Consumer 기반 ItemReader에서 오프셋 관리를 Spring Batch와 연동하는 방법은?
- `AbstractItemCountingItemStreamItemReader`를 상속하는 것과 직접 `ItemStreamReader`를 구현하는 것의 차이는?
- Custom ItemWriter에서 트랜잭션 롤백 시 안전성을 보장하는 방법은?

---

## 🔍 왜 Custom Reader/Writer가 필요한가

### 문제: 기본 제공 Reader/Writer로 처리할 수 없는 소스가 있다

```
기본 제공 Reader/Writer로 처리 가능한 것:
  DB (JPA, JDBC), CSV/XML 파일, JSON 파일

Custom Reader/Writer가 필요한 경우:
  - Kafka Topic에서 메시지 소비
  - REST API 페이지네이션 응답
  - MongoDB, Redis, Elasticsearch
  - AWS S3 스트리밍 읽기
  - FTP 서버의 여러 파일 순차 읽기
  - 복잡한 멀티소스 조인 읽기

Custom 구현 시 핵심 과제:
  1. read()가 null을 반환하는 종료 조건 정의
  2. 재시작을 위한 ExecutionContext 저장/복원
  3. open/close의 리소스 관리 (커넥션, 파일 핸들 등)
  4. Thread-safety (Multi-threaded Step 고려)
```

---

## 😱 흔한 실수

### Before: ItemStream을 구현하지 않아 재시작이 불가능하다

```java
// ❌ ItemStream 미구현 — 재시작 시 처음부터 시작
@Component
public class ApiItemReader implements ItemReader<Product> {

    private int currentPage = 0;
    private List<Product> buffer = new ArrayList<>();

    @Override
    public Product read() throws Exception {
        if (buffer.isEmpty()) {
            // 다음 페이지 로드
            List<Product> page = apiClient.getProducts(currentPage++, PAGE_SIZE);
            if (page.isEmpty()) return null;
            buffer.addAll(page);
        }
        return buffer.remove(0);
    }
    // currentPage는 인스턴스 변수에만 존재
    // 재시작 시 currentPage=0으로 리셋 → 처음부터 다시 처리
}

// ✅ ItemStream 구현 → currentPage를 ExecutionContext에 저장
@Component
public class ApiItemReader implements ItemStreamReader<Product> {
    // ItemStreamReader = ItemReader + ItemStream

    private int currentPage = 0;
    private List<Product> buffer = new ArrayList<>();
    private static final String CURRENT_PAGE_KEY = "apiReader.currentPage";

    @Override
    public void open(ExecutionContext executionContext) {
        // 재시작 시 이전 페이지 위치 복원
        if (executionContext.containsKey(CURRENT_PAGE_KEY)) {
            currentPage = executionContext.getInt(CURRENT_PAGE_KEY);
        }
    }

    @Override
    public void update(ExecutionContext executionContext) {
        // Chunk 커밋마다 현재 페이지 저장
        executionContext.putInt(CURRENT_PAGE_KEY, currentPage);
    }

    @Override
    public void close() { /* 리소스 정리 */ }
}
```

### Before: update()에서 너무 많은 데이터를 ExecutionContext에 저장한다

```java
// ❌ 처리된 모든 ID를 저장
@Override
public void update(ExecutionContext executionContext) {
    // 10만 건 처리 후 processedIds = [1, 2, 3, ..., 100000]
    executionContext.put("processedIds", processedIds);  // List<Long> 10만 개
    // → 직렬화 크기: 수 MB → SHORT_CONTEXT(2500자) 초과
    // → SERIALIZED_CONTEXT 사용 + 매 Chunk 커밋마다 수 MB UPDATE
}

// ✅ 최소한의 재시작 포인트만 저장
@Override
public void update(ExecutionContext executionContext) {
    // "어디까지 읽었는가"만 저장 (재시작에 필요한 최소 정보)
    executionContext.putLong("lastProcessedId", lastProcessedId);
    // 처리 여부 확인은 DB 상태로 판단
}
```

---

## ✨ ItemStream 인터페이스 완전 이해

```java
// ItemStream 인터페이스
public interface ItemStream {

    /**
     * Step 시작 시 한 번 호출
     * - 리소스 열기 (DB 커넥션, 파일 핸들, Kafka Consumer 등)
     * - 재시작 시 ExecutionContext에서 이전 상태 복원
     */
    default void open(ExecutionContext executionContext) throws ItemStreamException {}

    /**
     * Chunk 커밋마다 호출 (트랜잭션 커밋 직전)
     * - 현재 처리 위치를 ExecutionContext에 저장
     * - 이 정보가 DB에 저장되어 재시작 포인트가 됨
     */
    default void update(ExecutionContext executionContext) throws ItemStreamException {}

    /**
     * Step 종료 시 한 번 호출 (성공/실패 모두)
     * - 리소스 닫기 (커넥션 반환, 파일 닫기 등)
     */
    default void close() throws ItemStreamException {}
}

// 호출 순서 타임라인:
// Step 시작 → open(EC)
// Chunk 1: read×1000 → process×1000 → write×1000 → update(EC) → COMMIT
// Chunk 2: read×1000 → process×1000 → write×1000 → update(EC) → COMMIT
// ...
// Chunk N: read → null → process → write → update(EC) → COMMIT
// Step 종료 → close()
```

---

## 🔬 내부 동작 원리

### 1. ItemStream이 Step에 등록되는 방법

```java
// AbstractStep.execute() — ItemStream 등록
// CompositeItemStream이 모든 스트림을 관리

public class CompositeItemStream implements ItemStream {
    private List<ItemStream> streams = new ArrayList<>();

    public void register(ItemStream stream) {
        streams.add(stream);
    }

    @Override
    public void open(ExecutionContext executionContext) {
        for (ItemStream stream : streams) {
            stream.open(executionContext);  // 순서대로 open
        }
    }

    @Override
    public void update(ExecutionContext executionContext) {
        for (ItemStream stream : streams) {
            stream.update(executionContext);  // 순서대로 update
        }
    }

    @Override
    public void close() {
        // 역순으로 close (open의 역순)
        for (int i = streams.size() - 1; i >= 0; i--) {
            streams.get(i).close();
        }
    }
}

// TaskletStep이 ChunkOrientedTasklet을 생성할 때
// Reader, Processor, Writer 중 ItemStream 구현체를 CompositeItemStream에 등록
```

### 2. ExecutionContext 키 네이밍 — 충돌 방지

```java
// Spring Batch 내부 컨벤션: "BeanName.field.name"
// AbstractItemStreamItemReader.getExecutionContextKey()

protected String getExecutionContextKey(String key) {
    // Bean 이름을 prefix로 사용해 키 충돌 방지
    return this.name + "." + key;
}

// 사용 예:
// reader.setName("orderCsvReader");
// getExecutionContextKey("read.count") → "orderCsvReader.read.count"

// 여러 Reader를 사용할 때 키 충돌 방지:
// reader1.setName("pendingOrderReader");
// reader2.setName("processedOrderReader");
// → EC: {"pendingOrderReader.read.count": 5000,
//        "processedOrderReader.read.count": 3000}
```

---

## 💻 실전 구현

### 1. REST API 페이지네이션 Custom Reader

```java
@Component
@StepScope
public class RestApiPagingItemReader<T> extends AbstractItemStreamItemReader<T> {

    private final RestTemplate restTemplate;
    private final String apiUrl;
    private final Class<T> targetType;

    private int currentPage = 0;
    private int totalPages = -1;  // -1: 아직 모름
    private List<T> buffer = new ArrayList<>();
    private boolean exhausted = false;

    private static final String CURRENT_PAGE_KEY = "currentPage";

    // === ItemStream 구현 ===

    @Override
    public void open(ExecutionContext executionContext) {
        // 재시작: 이전 페이지 위치 복원
        if (executionContext.containsKey(getExecutionContextKey(CURRENT_PAGE_KEY))) {
            currentPage = executionContext.getInt(
                getExecutionContextKey(CURRENT_PAGE_KEY));
            log.info("재시작 감지: page={} 부터 재개", currentPage);
            // 버퍼 다시 채우기 (현재 페이지 재로드)
            loadPage();
        }
    }

    @Override
    public void update(ExecutionContext executionContext) {
        executionContext.putInt(getExecutionContextKey(CURRENT_PAGE_KEY), currentPage);
    }

    @Override
    public void close() {
        buffer.clear();
    }

    // === ItemReader 구현 ===

    @Override
    public T read() throws Exception {
        if (exhausted) return null;

        if (buffer.isEmpty()) {
            if (!loadPage()) {
                exhausted = true;
                return null;  // 데이터 소진
            }
        }

        return buffer.isEmpty() ? null : buffer.remove(0);
    }

    private boolean loadPage() {
        try {
            String url = apiUrl + "?page=" + currentPage + "&size=100";
            PageResponse<T> response = restTemplate.getForObject(url,
                getParameterizedTypeRef());

            if (response == null || response.getContent().isEmpty()) {
                return false;
            }

            buffer.addAll(response.getContent());
            totalPages = response.getTotalPages();
            currentPage++;

            return true;
        } catch (RestClientException e) {
            throw new ItemStreamException("API 페이지 로드 실패: page=" + currentPage, e);
        }
    }
}
```

### 2. Kafka Consumer Custom Reader — 배치 오프셋 관리

```java
@Component
@StepScope
public class KafkaItemReader<K, V> extends AbstractItemStreamItemReader<V> {

    private final ConsumerFactory<K, V> consumerFactory;
    private final String topic;
    private final Duration pollTimeout;

    private Consumer<K, V> consumer;
    private final Queue<ConsumerRecord<K, V>> buffer = new LinkedList<>();

    // ExecutionContext 키 (파티션별 오프셋 저장)
    private static final String OFFSET_KEY_PREFIX = "kafka.offset.";

    // 최대 poll 횟수 (무한 루프 방지)
    private int maxPollCount;
    private int pollCount = 0;

    @Override
    public void open(ExecutionContext executionContext) {
        // Kafka Consumer 생성 (자동 오프셋 커밋 비활성화)
        Map<String, Object> props = new HashMap<>(consumerFactory.getConfigurationProperties());
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        // → Spring Batch가 Chunk 커밋 시점에 오프셋을 수동 커밋
        consumer = consumerFactory.createConsumer(null, null, null, props);
        consumer.subscribe(List.of(topic));

        // 재시작: 이전 오프셋으로 seek
        if (executionContext.containsKey(getExecutionContextKey(OFFSET_KEY_PREFIX + "0"))) {
            consumer.poll(Duration.ofMillis(100));  // 파티션 할당 대기
            consumer.assignment().forEach(partition -> {
                String key = getExecutionContextKey(OFFSET_KEY_PREFIX + partition.partition());
                if (executionContext.containsKey(key)) {
                    long offset = executionContext.getLong(key);
                    consumer.seek(partition, offset);
                    log.info("재시작: partition={}, offset={}", partition, offset);
                }
            });
        }
    }

    @Override
    public void update(ExecutionContext executionContext) {
        // Chunk 커밋 직전: 현재 오프셋 저장
        if (consumer != null) {
            consumer.assignment().forEach(partition -> {
                long offset = consumer.position(partition);
                executionContext.putLong(
                    getExecutionContextKey(OFFSET_KEY_PREFIX + partition.partition()),
                    offset);
            });
        }
        // ← 트랜잭션과 함께 커밋 → DB의 오프셋과 메시지 처리가 원자적
    }

    @Override
    public void close() {
        if (consumer != null) {
            consumer.close();
        }
    }

    @Override
    public V read() throws Exception {
        if (buffer.isEmpty()) {
            if (pollCount >= maxPollCount) {
                return null;  // 최대 poll 횟수 도달 → Step 종료
            }
            ConsumerRecords<K, V> records = consumer.poll(pollTimeout);
            pollCount++;
            if (records.isEmpty()) {
                return null;  // 타임아웃 내 메시지 없음 → Step 종료
            }
            records.forEach(buffer::add);
        }
        return buffer.poll().value();
    }
}

// 설정 예시
@Bean
@StepScope
public KafkaItemReader<String, OrderEvent> kafkaOrderReader(
        ConsumerFactory<String, OrderEvent> consumerFactory) {

    KafkaItemReader<String, OrderEvent> reader = new KafkaItemReader<>();
    reader.setConsumerFactory(consumerFactory);
    reader.setTopic("order-events");
    reader.setPollTimeout(Duration.ofSeconds(5));
    reader.setMaxPollCount(100);   // 최대 100번 poll (타임아웃 방지)
    reader.setName("kafkaOrderReader");
    return reader;
}
```

### 3. Custom ItemWriter — 외부 시스템 쓰기 + 트랜잭션 안전성

```java
@Component
public class S3BatchItemWriter<T> implements ItemStreamWriter<T> {

    private final AmazonS3 s3Client;
    private final String bucket;
    private final String keyPrefix;
    private final ObjectMapper objectMapper;

    // 트랜잭션 롤백 시 S3에 이미 쓴 데이터를 추적 (보상 트랜잭션용)
    private final List<String> writtenKeys = new ArrayList<>();

    @Override
    public void open(ExecutionContext executionContext) {
        // S3 클라이언트 초기화
        writtenKeys.clear();
    }

    @Override
    public void write(Chunk<? extends T> chunk) throws Exception {
        List<? extends T> items = chunk.getItems();
        String chunkKey = keyPrefix + "/" + System.currentTimeMillis() + ".json";

        try {
            String json = objectMapper.writeValueAsString(items);
            s3Client.putObject(bucket, chunkKey, json);
            writtenKeys.add(chunkKey);
            // ← S3는 트랜잭션 미지원 → 롤백 시 수동 삭제 필요
        } catch (Exception e) {
            // 쓰기 실패 → 트랜잭션 롤백 → 이미 쓴 파일 삭제 (보상)
            cleanupWrittenFiles();
            throw new ItemStreamException("S3 쓰기 실패: " + chunkKey, e);
        }
    }

    @Override
    public void update(ExecutionContext executionContext) {
        // Chunk 커밋 성공 → writtenKeys를 EC에 기록
        executionContext.put("writtenKeys", new ArrayList<>(writtenKeys));
        writtenKeys.clear();  // 다음 Chunk를 위해 초기화
    }

    @Override
    public void close() { /* 정리 */ }

    private void cleanupWrittenFiles() {
        writtenKeys.forEach(key -> {
            try {
                s3Client.deleteObject(bucket, key);
                log.info("롤백 보상: S3 파일 삭제 {}", key);
            } catch (Exception e) {
                log.error("롤백 보상 실패: {}", key, e);
            }
        });
        writtenKeys.clear();
    }
}
```

---

## 📊 Custom 구현 방식 비교

```
Custom Reader 구현 방식:

┌───────────────────────────────────┬──────────────────────────┬───────────────────────┐
│ 방식                               │ 적합한 경우                 │ 주의사항                │
├───────────────────────────────────┼──────────────────────────┼───────────────────────┤
│ AbstractItemCountingItemStream-   │ 단순 카운트 기반 재시작        │ jumpToItem() 구현 필요  │
│ ItemReader 상속                    │                          │                       │
│ ItemStreamReader 직접 구현          │ 복잡한 재시작 로직            │ open/update/close 모두│
│                                   │ (Kafka 오프셋 등)          │ 직접 구현               │
│ ItemReader만 구현                  │ 재시작 불필요한 경우           │ saveState=false 설정  │
│ (ItemStream 없음)                  │                          │                       │
└───────────────────────────────────┴──────────────────────────┴───────────────────────┘
```

---

## ⚖️ 트레이드오프

```
Custom ItemReader/Writer 구현:

  장점
    Spring Batch의 재시작·Skip·Retry 인프라 활용 가능
    ItemStream으로 재시작 포인트 관리
    Chunk 기반 트랜잭션 자동 적용

  단점
    ItemStream 구현 복잡도 (특히 재시작 로직)
    트랜잭션 미지원 외부 시스템(S3, Kafka 등) 보상 트랜잭션 필요
    ExecutionContext 크기 관리 (너무 많은 상태 저장 시 오버헤드)

Kafka + Spring Batch 특수 고려사항:

  Exactly-once 보장의 어려움
  → DB COMMIT과 Kafka 오프셋 커밋의 원자적 처리 불가
  → At-least-once(재처리 가능 설계) 또는
    Transactional Kafka Consumer (Kafka transactions) 활용

  Spring Kafka: KafkaItemReader (공식 지원, Spring Batch 4.3+)
  → org.springframework.batch.item.kafka.KafkaItemReader
```

---

## 📌 핵심 정리

```
ItemStream 3메서드 호출 시점

  open(EC)    → Step 시작 시 1회 (리소스 열기 + 재시작 상태 복원)
  update(EC)  → Chunk 커밋마다 (현재 처리 위치 저장 = 재시작 포인트)
  close()     → Step 종료 시 1회 (리소스 정리, 성공/실패 모두)

재시작 가능한 Reader 구현 핵심

  1. update()에서 "어디까지 읽었는가"를 EC에 저장
     → 최소한의 정보만 저장 (ID, 오프셋, 페이지 번호 등)
  2. open()에서 EC에 저장된 값으로 위치 복원
     → EC 키 없으면 신규 실행, 있으면 재시작
  3. 키 네이밍: name + "." + key (충돌 방지)

Custom Writer 트랜잭션 안전성

  트랜잭션 지원 시스템 (DB, Kafka transactions)
  → 트랜잭션 롤백 시 자동 복원

  트랜잭션 미지원 시스템 (S3, REST API 등)
  → write()에서 쓴 것을 추적
  → 예외 발생 시 수동 삭제 (보상 트랜잭션)
  → 멱등성 설계 (같은 데이터 재쓰기 시 안전)
```

---

## 🤔 생각해볼 문제

**Q1.** `update(ExecutionContext)`는 Chunk 커밋 직전에 호출되고, `executionContext`의 내용은 DB에 저장됩니다. 그런데 `update()`가 호출된 후 트랜잭션 커밋이 실패하면 어떻게 되는가? EC에 저장된 포인트와 실제 처리된 데이터가 불일치하는 상황이 발생하는가?

**Q2.** Kafka Consumer 기반 Reader에서 Chunk 커밋 시 `update()`가 Kafka 오프셋을 EC에 저장합니다. 재시작 시 `open()`에서 해당 오프셋으로 `seek()`합니다. 이 과정에서 "DB는 커밋됐는데 EC가 저장 안 된 경우"와 "EC는 저장됐는데 DB가 롤백된 경우" 중 어떤 상황이 더 문제가 되는가? 각각 어떻게 처리해야 하는가?

**Q3.** Custom Reader가 `@StepScope` Bean으로 등록될 때, 동일 Step에서 두 번 `read()`가 순서대로 실행되면 인스턴스 변수(`currentPage`, `buffer`)는 올바르게 유지되는가? `@StepScope`가 없는 싱글톤 Reader에서 같은 Job을 동시에 실행하면?

> 💡 **해설**
>
> **Q1.** `update()`와 트랜잭션 커밋은 **같은 트랜잭션** 안에서 수행됩니다. `TaskletStep`의 트랜잭션 템플릿 내에서: ①`write()`, ②`stepExecution.apply(contribution)`, ③`jobRepository.updateExecutionContext(se)` — EC의 DB 저장, ④`jobRepository.update(se)`, ⑤ 트랜잭션 커밋 순서로 실행됩니다. 커밋이 실패하면 ①~④가 모두 롤백되므로 EC도 이전 상태로 돌아갑니다. 따라서 EC와 실제 처리 데이터는 항상 동일한 트랜잭션 경계를 가집니다 — 불일치가 발생하지 않습니다.
>
> **Q2.** "DB 커밋됐는데 EC 저장 안 된 경우"는 발생하지 않습니다(Q1 해설 참조, 같은 트랜잭션). 실제 위험한 경우는 "EC(오프셋)는 저장됐는데 DB가 롤백된 경우"도 발생하지 않습니다. 문제가 되는 경우는 **서버 강제 종료** 시입니다: 예를 들어 EC에 offset=1000이 저장돼 DB에 반영됐지만, Kafka 오프셋은 아직 커밋되지 않은 상태에서 서버가 죽으면, 재시작 시 offset=1000으로 seek하지만 Kafka 브로커는 같은 메시지를 다시 보낼 수 있습니다. 이를 해결하려면 멱등성(같은 메시지 재처리 시 결과가 동일)을 보장하는 설계가 필요합니다.
>
> **Q3.** `@StepScope` Bean은 Step 실행 시점에 인스턴스가 생성되고, Step이 끝날 때 소멸됩니다. 따라서 동일 Step 내에서 `read()`를 여러 번 호출해도 같은 인스턴스를 사용하므로 `currentPage`, `buffer`가 올바르게 유지됩니다. 반면 `@StepScope`가 없는 싱글톤 Reader를 동일 Job을 동시에 실행하면, 두 JobExecution이 같은 Reader 인스턴스를 공유합니다. `currentPage++`처럼 상태 변경이 있으면 두 쓰레드가 같은 값을 참조하거나 건너뛰는 Race Condition이 발생합니다. 해결책: `@StepScope` 사용(각 Step 실행마다 새 인스턴스) 또는 `ThreadLocal`로 상태 분리합니다.

---

<div align="center">

**[⬅️ 이전: Cursor vs Paging 선택 기준](./06-cursor-vs-paging.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 3 — Sequential Flow ➡️](../job-flow-control/01-sequential-flow.md)**

</div>
