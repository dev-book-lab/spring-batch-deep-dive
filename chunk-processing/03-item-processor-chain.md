# ItemProcessor 체인과 조건부 처리 — null 필터링과 CompositeItemProcessor

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `CompositeItemProcessor`는 여러 Processor를 어떻게 체인으로 연결하는가?
- `ItemProcessor.process()`가 `null`을 반환했을 때 Chunk 처리 흐름에서 정확히 무슨 일이 일어나는가?
- 조건부 필터링과 조건부 Skip의 차이는 무엇이고 언제 각각을 선택해야 하는가?
- Processor 체인 중간에서 타입이 바뀔 때(`ItemProcessor<A, B>`, `ItemProcessor<B, C>`) 어떻게 연결되는가?
- Processor를 여러 개로 분리하면 어떤 장점이 있는가?

---

## 🔍 왜 ItemProcessor를 별도로 분리하는가

### 문제: Read와 Write 사이의 변환 로직이 점점 복잡해진다

```
변환 로직의 복잡성 증가 시나리오:

  초기:
    Order → SettledOrder 변환 (수수료 계산)

  요구사항 추가 1:
    VIP 고객은 수수료 0% 적용

  요구사항 추가 2:
    환율 변환 (USD → KRW)

  요구사항 추가 3:
    블랙리스트 고객 필터링

  요구사항 추가 4:
    이상 거래 감지 (1억 이상 주문 별도 검토)

하나의 Processor에 모두 구현하면:
  public SettledOrder process(Order order) {
      if (isBlacklisted(order.getCustomerId())) return null;  // 필터
      if (order.getCurrency().equals("USD")) convertCurrency(order);
      BigDecimal fee = isVip(order.getCustomerId()) ? ZERO : calculateFee(order);
      if (order.getAmount().compareTo(ANOMALY_THRESHOLD) > 0) flagAnomaly(order);
      return new SettledOrder(order, fee);
  }
  // → 단일 책임 원칙 위반
  // → 테스트 어려움 (각 규칙을 독립적으로 테스트할 수 없음)
  // → 요구사항 변경 시 전체 Processor 수정

CompositeItemProcessor로 분리:
  BlacklistFilterProcessor    → null 반환 시 이후 Processor 스킵
  CurrencyConversionProcessor → USD → KRW 변환
  FeeCalculationProcessor     → VIP 여부에 따른 수수료 계산
  AnomalyDetectionProcessor   → 이상 거래 플래그
```

---

## 😱 흔한 실수

### Before: null 반환이 Skip과 같다고 생각한다

```java
// ❌ 잘못된 이해
public SettledOrder process(Order order) {
    try {
        return calculateSettlement(order);
    } catch (Exception e) {
        return null;  // "예외 발생 시 이 건을 Skip하려고 null 반환"
    }
}
// 실제:
// null 반환 → FILTER (정상 처리, writeCount에 포함 안 됨, filterCount++)
// 예외 throw → Skip 정책에 따라 처리 (skipCount++, 로그, ItemSkipListener 호출)
// → null 반환은 "의도적으로 Writer에서 제외"
// → 예외 처리와 혼용하면 오류 상황이 조용히 무시됨

// ✅ 의도를 명확히 분리
public SettledOrder process(Order order) {
    if (!isEligible(order)) {
        return null;  // 의도적 필터링 (정상)
    }
    return calculateSettlement(order);  // 실패 시 예외 throw → Skip/Retry 정책 적용
}
```

### Before: CompositeItemProcessor 타입 연결을 잘못 이해한다

```java
// ❌ 타입 불일치 — 컴파일은 되지만 런타임 ClassCastException
@Bean
public CompositeItemProcessor processor() {
    CompositeItemProcessor processor = new CompositeItemProcessor();  // raw type
    processor.setDelegates(List.of(
        new OrderToRawProcessor(),      // Order → RawOrder
        new FeeCalculationProcessor()   // Order → SettledOrder (❌ 입력이 RawOrder여야 함)
    ));
    return processor;
}

// ✅ 제네릭 타입 맞추기
// OrderToRawProcessor: ItemProcessor<Order, RawOrder>
// RawToSettledProcessor: ItemProcessor<RawOrder, SettledOrder>
// 두 번째 Processor의 입력 = 첫 번째 Processor의 출력
```

---

## ✨ 올바른 이해와 사용

### CompositeItemProcessor 체인 구성

```java
// 각 Processor를 독립적으로 정의 (단일 책임)
@Component
public class BlacklistFilterProcessor implements ItemProcessor<Order, Order> {
    @Autowired
    private BlacklistRepository blacklistRepo;

    @Override
    public Order process(Order order) throws Exception {
        if (blacklistRepo.existsByCustomerId(order.getCustomerId())) {
            log.info("블랙리스트 고객 필터링: customerId={}", order.getCustomerId());
            return null;  // Write에서 제외
        }
        return order;  // 통과
    }
}

@Component
public class FeeCalculationProcessor implements ItemProcessor<Order, SettledOrder> {
    @Override
    public SettledOrder process(Order order) throws Exception {
        BigDecimal feeRate = order.isVip() ? BigDecimal.ZERO : new BigDecimal("0.015");
        BigDecimal fee = order.getAmount().multiply(feeRate);
        return new SettledOrder(order.getId(), order.getAmount(), fee);
    }
}

// CompositeItemProcessor로 체인 구성
@Bean
public CompositeItemProcessor<Order, SettledOrder> compositeProcessor() {
    CompositeItemProcessor<Order, SettledOrder> processor = new CompositeItemProcessor<>();
    processor.setDelegates(List.of(
        blacklistFilterProcessor,    // Order → Order (null이면 이후 Processor 스킵)
        feeCalculationProcessor      // Order → SettledOrder
    ));
    return processor;
}
```

---

## 🔬 내부 동작 원리

### 1. CompositeItemProcessor.process() 체인 실행

```java
// CompositeItemProcessor.java
public class CompositeItemProcessor<I, O> implements ItemProcessor<I, O>,
        ItemStream, InitializingBean {

    private List<? extends ItemProcessor<?, ?>> delegates;

    @Override
    @SuppressWarnings("unchecked")
    public O process(I item) throws Exception {
        Object result = item;

        for (ItemProcessor<?, ?> delegate : delegates) {
            if (result == null) {
                return null;  // ← 중간에 null이 나오면 이후 Processor 모두 스킵
            }
            // 각 Processor 호출 (raw type 캐스팅)
            result = ((ItemProcessor<Object, Object>) delegate).process(result);
        }

        return (O) result;
    }
}

// 체인 실행 예시:
// BlacklistFilterProcessor.process(order) → null  (블랙리스트 고객)
// FeeCalculationProcessor.process(null)   → 호출 안 됨 (null 전파)
// CompositeProcessor 반환값: null
// → SimpleChunkProcessor가 filterCount++ 처리
```

### 2. null 반환 후 SimpleChunkProcessor 처리

```java
// SimpleChunkProcessor.transform() — null 처리 부분 상세
protected Chunk<O> transform(StepContribution contribution, Chunk<I> inputs)
        throws Exception {

    Chunk<O> outputs = new Chunk<>();

    for (Chunk<I>.ChunkIterator iterator = inputs.iterator(); iterator.hasNext(); ) {
        final I item = iterator.next();
        O output;

        try {
            output = doProcess(item);  // ItemProcessor.process() 호출
        } catch (Exception e) {
            // 예외 → Skip 정책 확인 (FaultTolerantChunkProcessor에서 처리)
            iterator.remove(e);
            continue;
        }

        if (output != null) {
            outputs.add(output);
            // writeCount에 포함 예정
        } else {
            // null 반환 → 필터링
            contribution.incrementFilterCount(1);  // filterCount++
            iterator.remove();  // inputs에서도 제거 (Skip 로그에 남지 않음)
            // ← filterCount: BATCH_STEP_EXECUTION.FILTER_COUNT 컬럼
        }
    }

    return outputs;
}
```

### 3. 조건부 필터링 vs 조건부 Skip — 처리 흐름 비교

```
아이템 처리 결과 분기:

  ┌───────────────────────────────────────────────────────┐
  │ process(item) 실행                                     │
  └───────────────────────────────────────────────────────┘
               │
       ┌───────┴────────┐
       ▼                ▼
   결과 반환          null 반환           예외 throw
   (정상 처리)        (필터링)           (오류)
       │                │                  │
       ▼                ▼                  ▼
  outputs에 추가    filterCount++     FaultTolerant?
  writeCount++       (조용히 제외)            │
                                    ┌──────┴──────┐
                                    ▼             ▼
                               Skip 설정        Retry 설정
                                    │             │
                                    ▼             ▼
                               skipCount++    재시도 N회
                               ItemSkipListener  성공 시 정상
                               호출              실패 시 skipCount++
```

### 4. Processor 단계에서의 타입 변환 추적

```java
// 타입이 중간에 변경되는 체인 예시:
// Step 선언: <Order, SettledOrder>chunk()
//
// Processor 1: ItemProcessor<Order, EnrichedOrder>
//   Order에 고객 정보를 enrichment
//
// Processor 2: ItemProcessor<EnrichedOrder, SettledOrder>
//   EnrichedOrder를 정산 결과로 변환
//
// CompositeItemProcessor<Order, SettledOrder>
//   내부적으로 Object 타입으로 캐스팅해 전달

@Bean
public CompositeItemProcessor<Order, SettledOrder> processor(
        ItemProcessor<Order, EnrichedOrder> enricher,
        ItemProcessor<EnrichedOrder, SettledOrder> settler) {

    CompositeItemProcessor<Order, SettledOrder> processor = new CompositeItemProcessor<>();
    processor.setDelegates(List.of(enricher, settler));
    // 타입 안전성: 컴파일러가 중간 타입 체크 불가 (raw type 캐스팅)
    // → 중간 타입이 맞지 않으면 런타임 ClassCastException
    return processor;
}
```

---

## 💻 실전 구현

### 실전 Processor 체인: 주문 정산 파이프라인

```java
// 1단계: 블랙리스트 필터 (Order → Order or null)
@Component
@StepScope
public class BlacklistFilterProcessor implements ItemProcessor<Order, Order> {

    @Autowired
    private BlacklistRepository blacklistRepo;

    private final Set<Long> cachedBlacklist = new HashSet<>();

    @PostConstruct
    public void loadBlacklist() {
        // Step 시작 시 블랙리스트 캐싱 (건마다 DB 조회 방지)
        cachedBlacklist.addAll(blacklistRepo.findAllCustomerIds());
    }

    @Override
    public Order process(Order order) {
        if (cachedBlacklist.contains(order.getCustomerId())) {
            return null;  // 블랙리스트 → Writer 제외
        }
        return order;
    }
}

// 2단계: 환율 변환 (Order → Order)
@Component
public class CurrencyConversionProcessor implements ItemProcessor<Order, Order> {

    @Autowired
    private ExchangeRateService rateService;

    @Override
    public Order process(Order order) throws Exception {
        if ("USD".equals(order.getCurrency())) {
            BigDecimal krw = order.getAmount()
                .multiply(rateService.getCurrentRate("USD", "KRW"));
            order.setAmount(krw);
            order.setCurrency("KRW");
        }
        return order;
    }
}

// 3단계: 수수료 계산 + 타입 변환 (Order → SettledOrder)
@Component
public class SettlementProcessor implements ItemProcessor<Order, SettledOrder> {

    @Override
    public SettledOrder process(Order order) throws Exception {
        BigDecimal fee = calculateFee(order);
        return SettledOrder.builder()
            .orderId(order.getId())
            .originalAmount(order.getAmount())
            .fee(fee)
            .netAmount(order.getAmount().subtract(fee))
            .settledAt(LocalDateTime.now())
            .build();
    }

    private BigDecimal calculateFee(Order order) {
        if (order.isVip()) return BigDecimal.ZERO;
        BigDecimal rate = order.getAmount().compareTo(new BigDecimal("1000000")) > 0
            ? new BigDecimal("0.010")  // 100만원 초과: 1%
            : new BigDecimal("0.015"); // 100만원 이하: 1.5%
        return order.getAmount().multiply(rate).setScale(0, RoundingMode.HALF_UP);
    }
}

// 조합
@Bean
public CompositeItemProcessor<Order, SettledOrder> settlementProcessor() {
    CompositeItemProcessor<Order, SettledOrder> processor = new CompositeItemProcessor<>();
    processor.setDelegates(List.of(
        blacklistFilterProcessor,      // ① 필터 (null 반환 가능)
        currencyConversionProcessor,   // ② 변환
        settlementProcessor            // ③ 타입 변환 + 계산
    ));
    return processor;
}
```

### ItemProcessorAdapter — 기존 서비스 메서드 재사용

```java
// 기존 서비스 메서드를 Processor로 래핑
@Bean
public ItemProcessorAdapter<Order, SettledOrder> serviceAdapter(
        OrderSettlementService settlementService) {
    ItemProcessorAdapter<Order, SettledOrder> adapter = new ItemProcessorAdapter<>();
    adapter.setTargetObject(settlementService);
    adapter.setTargetMethod("calculateSettlement");  // Object → Object 매핑
    return adapter;
}
// settlementService.calculateSettlement(Order) → SettledOrder

// ScriptItemProcessor — Groovy/JS 스크립트로 변환 로직 외부화
@Bean
public ScriptItemProcessor<Order, SettledOrder> scriptProcessor() {
    ScriptItemProcessor<Order, SettledOrder> processor = new ScriptItemProcessor<>();
    processor.setScript(new ClassPathResource("settlement.groovy"));
    return processor;
}
```

---

## 📊 Processor 설계 패턴 비교

```
Processor 설계 방식:

┌───────────────────────┬─────────────────────────────────┬──────────────────────────┐
│ 패턴                   │ 장점                             │ 단점                      │
├───────────────────────┼─────────────────────────────────┼──────────────────────────┤
│ 단일 Processor         │ 단순, 컨텍스트 공유 쉬움              │ 책임 혼재, 테스트 어려움      │
│ CompositeProcessor    │ 단일 책임, 독립 테스트, 재사용         │ 타입 체인 복잡              │
│ null 필터링             │ 코드 단순 (예외 없음)               │ 오류와 필터 구분 불명확        │
│ 예외 throw + Skip      │ 오류 추적 명확 (skipCount, 로그)    │ Skip 정책 설정 필요          │
└───────────────────────┴─────────────────────────────────┴──────────────────────────┘

null 반환 vs 예외 사용 기준:

  null 반환 (필터링)
    → 비즈니스 규칙에 의한 정상적인 제외
    → 예: 이미 처리된 데이터, 조건 미충족 항목
    → filterCount에 기록 (오류 없음)

  예외 throw (오류)
    → 처리할 수 없는 예외적 상황
    → 예: 데이터 형식 오류, 필수 데이터 없음
    → skipCount에 기록, ItemSkipListener로 알림 발송 가능
```

---

## ⚖️ 트레이드오프

```
CompositeItemProcessor:

  장점
    각 Processor 독립 단위 테스트 가능
    Processor 교체/추가가 Step 설정만으로 가능
    null 전파로 이후 Processor 자동 스킵 (불필요한 처리 없음)

  단점
    raw type 캐스팅으로 컴파일 타임 중간 타입 체크 불가
    Processor 수가 많으면 체인 추적이 어려움
    각 Processor가 독립적 Spring Bean → 주입 관계 복잡해질 수 있음

null 필터링:

  장점  간단하고 직관적 (예외 없음)
  단점  어떤 데이터가 왜 필터됐는지 기본 로그만으로는 알 수 없음
       → ItemReadListener/ItemWriteListener보다 추적이 어려움
       → 별도 로그를 Processor 내부에서 명시적으로 남겨야 함
```

---

## 📌 핵심 정리

```
CompositeItemProcessor 체인 규칙

  각 Processor는 이전 Processor의 출력을 입력으로 받음
  중간에 null 반환 → 이후 Processor 모두 건너뜀 → 전체 null 반환
  최종 null → SimpleChunkProcessor가 filterCount++ 처리 (Writer 제외)

null 반환 vs 예외의 차이

  null 반환
    → filterCount++ (BATCH_STEP_EXECUTION.FILTER_COUNT)
    → ItemSkipListener 호출 없음, 조용한 제외
    → 재처리 없음

  예외 throw
    → Skip 정책 적용 → skipCount++ 또는 Step FAILED
    → ItemSkipListener.onSkipInProcess() 호출 → 알림/로그
    → FaultTolerant 설정 필요

Processor 선택 기준

  비즈니스 규칙에 의한 정상 제외  → null 반환 (필터링)
  처리 불가능한 오류 상황         → 예외 throw (Skip/Retry)
  여러 변환 단계 필요             → CompositeItemProcessor
  기존 서비스 메서드 재사용       → ItemProcessorAdapter
```

---

## 🤔 생각해볼 문제

**Q1.** `CompositeItemProcessor`의 첫 번째 Processor가 `null`을 반환했을 때 두 번째 Processor는 `null`을 입력으로 받지 않고 스킵됩니다. 만약 두 번째 Processor가 `null` 입력을 처리해야 하는 경우(예: null도 기록해야 하는 감사 로그)라면 어떻게 설계해야 하는가?

**Q2.** `ItemProcessor`는 Thread-safe한가? `@StepScope`가 아닌 싱글톤 Processor가 Multi-threaded Step에서 인스턴스 변수를 공유하면 어떤 문제가 발생하는가?

**Q3.** `BlacklistFilterProcessor`에서 블랙리스트를 `@PostConstruct`에서 메모리에 캐싱했습니다. 배치 실행 중 블랙리스트가 업데이트된다면 이 캐시는 언제 갱신되는가? 이를 개선하는 방법은?

> 💡 **해설**
>
> **Q1.** `CompositeItemProcessor`의 `null` 전파 동작을 우회하려면, 첫 번째 Processor를 `CompositeItemProcessor` 바깥으로 빼고 `ItemReadListener` 또는 `ItemProcessListener`에서 필터링 감사 로그를 별도로 처리합니다. 또는 `null` 대신 특수 마커 객체(`FilteredOrder` 등)를 반환해서 이후 Processor들이 이를 인식하고 처리하는 패턴을 사용합니다. 단, 이는 `CompositeItemProcessor`의 `null` 규약을 무너뜨리므로 Processor 간 결합도가 높아집니다. 가장 깔끔한 방법은 `ItemWriteListener.beforeWrite()`에서 필터된 아이템을 감지하는 것이지만, 이미 `inputs`에서 제거됐으므로 `ItemProcessListener.afterProcess(item, null)`을 활용하는 것이 현실적입니다.
>
> **Q2.** `ItemProcessor`는 기본적으로 Thread-safe 여부가 규정되어 있지 않습니다. 싱글톤 Processor가 인스턴스 변수에 상태를 저장하면 Multi-threaded Step에서 여러 쓰레드가 같은 인스턴스를 공유하므로 Race Condition이 발생합니다. 예를 들어 환율 변환 Processor가 `private BigDecimal currentRate`를 인스턴스 변수로 유지하면, 한 쓰레드가 값을 읽는 중 다른 쓰레드가 변경할 수 있습니다. 해결책: Processor를 상태 없이(stateless) 설계하거나, `@StepScope`로 쓰레드별 인스턴스를 생성하거나, `ThreadLocal`을 사용합니다.
>
> **Q3.** `@PostConstruct`에서 캐싱한 블랙리스트는 Step 시작 시 한 번만 로드되므로, 배치 실행 중에는 갱신되지 않습니다. 개선 방법: (1) 캐시에 TTL을 적용해 일정 시간마다 재조회합니다(`Caffeine.newBuilder().expireAfterWrite(5, TimeUnit.MINUTES)`). (2) 블랙리스트를 캐시 서버(Redis)에서 관리하고 배치에서는 Redis를 조회합니다. (3) 블랙리스트 변경이 즉시 반영되어야 한다면 캐싱을 포기하고 건별 DB 조회로 전환하되, 성능을 위해 `SELECT * WHERE id IN (batch_ids)` 방식의 청크 단위 조회로 최적화합니다.

---

<div align="center">

**[⬅️ 이전: ItemReader 종류와 구현](./02-item-reader-types.md)** | **[홈으로 🏠](../README.md)** | **[다음: ItemWriter 구현 ➡️](./04-item-writer-implementations.md)**

</div>
