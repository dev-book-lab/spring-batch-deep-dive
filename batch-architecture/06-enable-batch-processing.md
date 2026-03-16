# @EnableBatchProcessing과 Batch 자동 구성 — Spring Boot 3.x의 변화와 내부 초기화

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@EnableBatchProcessing`이 등록하는 Bean은 무엇이며, 각각 어떤 역할을 하는가?
- Spring Boot 3.x에서 `@EnableBatchProcessing` 없이도 배치가 동작하는 이유는?
- `DataSource`가 있을 때와 없을 때 `JobRepository` 구현체가 달라지는 이유는?
- `spring.batch.job.enabled=false`를 설정해야 하는 경우는 언제인가?
- 커스텀 `JobRepository` 설정이 필요한 경우 어떻게 오버라이드하는가?

---

## 🔍 왜 @EnableBatchProcessing이 필요했는가

### 문제: Batch 인프라 Bean을 매번 수동으로 등록하기 번거롭다

```
Spring Batch가 동작하려면 필요한 인프라 Bean:

  JobRepository         → 메타데이터 DB 저장
  JobLauncher           → Job 실행 진입점
  JobExplorer           → 실행 이력 조회 (읽기 전용)
  JobRegistry           → Job Bean 등록 및 조회
  JobOperator           → Job 운영 (중지, 재시작, 목록 조회)
  PlatformTransactionManager → 트랜잭션 관리 (Chunk 커밋용)

@EnableBatchProcessing 이전 방식 (Spring Batch 3.x):
  @Bean
  public JobRepository jobRepository() { ... }
  @Bean
  public SimpleJobLauncher jobLauncher() { ... }
  // ... 매 프로젝트마다 반복

@EnableBatchProcessing 이후:
  @EnableBatchProcessing  // 한 줄로 모든 인프라 Bean 자동 등록
  public class BatchConfig { }
```

---

## 😱 흔한 실수

### Before: Spring Boot 3.x에서 @EnableBatchProcessing을 추가해 자동 구성을 깨뜨린다

```java
// ❌ Spring Boot 3.x에서의 잘못된 설정
@Configuration
@EnableBatchProcessing  // Spring Boot 3.x에서 이것을 추가하면 문제 발생!
public class BatchConfig {
    // Spring Boot 3.x에서 @EnableBatchProcessing을 붙이면
    // BatchAutoConfiguration이 비활성화됨
    // → 자동 구성(auto-configuration)이 꺼짐
    // → BatchProperties, JobLauncherApplicationRunner 등이 등록 안 됨
    // → spring.batch.job.name 설정이 무시됨
}

// Spring Boot 3.x에서 올바른 방법:
@Configuration
// @EnableBatchProcessing 생략 — BatchAutoConfiguration이 자동으로 처리
public class BatchConfig {
    // Job, Step Bean만 정의
}
```

### Before: DataSource 없이 배치를 운영 환경에서 실행한다

```java
// ❌ 운영 환경에서 DataSource 없이 실행 (인메모리 JobRepository)
// application.yml에 datasource 설정 없음
// → Spring Batch가 Map 기반 JobRepository 사용
// → 서버 재시작 시 모든 실행 이력 유실
// → 재시작 불가, 중복 실행 방지 불가

// ✅ 운영 환경에서는 반드시 DataSource 설정
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/batchdb
    username: batch
    password: secret
  batch:
    jdbc:
      initialize-schema: always  # 최초 실행 시 스키마 자동 생성
```

---

## ✨ 올바른 이해와 사용

### Spring Boot 버전별 @EnableBatchProcessing 동작

```
Spring Boot 2.x + Spring Batch 4.x:

  @EnableBatchProcessing 필요
  → BatchConfigurer를 통해 인프라 Bean 커스터마이징
  → DefaultBatchConfigurer 상속으로 오버라이드

Spring Boot 3.x + Spring Batch 5.x (현재):

  @EnableBatchProcessing 불필요 (오히려 방해)
  → BatchAutoConfiguration이 자동으로 모든 인프라 Bean 등록
  → BatchProperties로 application.yml에서 설정 가능
  → DefaultBatchConfiguration을 상속해서 커스터마이징
```

---

## 🔬 내부 동작 원리

### 1. @EnableBatchProcessing이 하는 일 (Spring Batch 5.x 기준)

```java
// @EnableBatchProcessing → BatchRegistrar 등록
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(BatchRegistrar.class)
public @interface EnableBatchProcessing {
    // observabilityConvention: 메트릭 관찰 설정
    // modular: true이면 ApplicationContext를 분리해 각 Job을 별도 컨텍스트에서 실행
    boolean modular() default false;
}

// BatchRegistrar → DefaultBatchConfiguration의 Bean들을 등록
// 등록되는 Bean:
//   jobRepository (JobRepository)
//   jobLauncher (JobLauncher)  
//   jobExplorer (JobExplorer)
//   jobRegistry (JobRegistry)
//   jobRegistryBeanPostProcessor (BeanPostProcessor — Job Bean 자동 등록)
//   jobOperator (JobOperator)
//   transactionManager (PlatformTransactionManager) — DataSource에 연결
```

### 2. BatchAutoConfiguration — Spring Boot 자동 구성

```java
// spring-boot-autoconfigure: BatchAutoConfiguration.java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ JobLauncher.class, DataSource.class, DatabasePopulator.class })
@ConditionalOnBean(JobLauncher.class)
// @EnableBatchProcessing이 있으면 이 자동 구성은 비활성화됨!
@ConditionalOnMissingBean(value = DefaultBatchConfiguration.class,
    annotation = EnableBatchProcessing.class)
@Import({ BatchConfigurerConfiguration.class, BatchRepositoryConfiguration.class })
@AutoConfigureAfter({ HibernateJpaAutoConfiguration.class,
    TransactionAutoConfiguration.class })
public class BatchAutoConfiguration {

    // JobLauncherApplicationRunner 등록 (spring.batch.job.enabled=true 시)
    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(prefix = "spring.batch.job", name = "enabled",
        havingValue = "true", matchIfMissing = true)
    public JobLauncherApplicationRunner jobLauncherApplicationRunner(
            JobLauncher jobLauncher, JobExplorer jobExplorer,
            JobRepository jobRepository, BatchProperties properties) {
        JobLauncherApplicationRunner runner =
            new JobLauncherApplicationRunner(jobLauncher, jobExplorer, jobRepository);

        // spring.batch.job.name으로 특정 Job만 실행
        String jobName = properties.getJob().getName();
        if (StringUtils.hasText(jobName)) {
            runner.setJobName(jobName);
        }
        return runner;
    }
}
```

### 3. DataSource 유무에 따른 JobRepository 선택

```java
// BatchRepositoryConfiguration.java

// ① DataSource + JDBC → JdbcJobRepository (운영 환경)
@Configuration(proxyBeanMethods = false)
@ConditionalOnSingleCandidate(DataSource.class)
@ConditionalOnMissingBean(JobRepository.class)
static class JdbcBatchRepositoryConfiguration {

    @Bean
    public JobRepository jobRepository(DataSource dataSource,
                                        DatabaseInitializerJobRepositoryFactoryBean factory,
                                        PlatformTransactionManager transactionManager)
            throws Exception {
        JobRepositoryFactoryBean factoryBean = new JobRepositoryFactoryBean();
        factoryBean.setDataSource(dataSource);
        factoryBean.setTransactionManager(transactionManager);
        factoryBean.setDatabaseType(DatabaseType.fromMetaData(dataSource).name());
        // BATCH_JOB_INSTANCE 등 6개 테이블에 접근
        factoryBean.afterPropertiesSet();
        return factoryBean.getObject();
    }
}

// ② DataSource 없음 → InMemoryJobRepository (개발/테스트)
@Configuration(proxyBeanMethods = false)
@ConditionalOnMissingBean({ DataSource.class, JobRepository.class })
static class InMemoryBatchRepositoryConfiguration {

    @Bean
    public JobRepository jobRepository(PlatformTransactionManager transactionManager) {
        // Map 기반 — 재시작 불가, 서버 재시작 시 이력 유실
        return new InMemoryJobRepository();
        // Spring Batch 5.x: MapJobRepositoryFactoryBean deprecated
        // → InMemoryJobRepository 사용
    }
}
```

### 4. 초기화 스키마 자동 생성

```java
// BatchDataSourceScriptDatabaseInitializer
// spring.batch.jdbc.initialize-schema 설정에 따라 동작

// ALWAYS  → 항상 스크립트 실행 (기존 테이블 DROP 후 재생성)
// EMBEDDED → H2, HSQLDB 등 내장 DB에서만 실행 (기본값)
// NEVER   → 스크립트 실행 안 함 (운영 환경 권장 — flyway/liquibase 사용)

// 스크립트 위치: spring-batch-core.jar 내부
// /org/springframework/batch/core/schema-mysql.sql
// /org/springframework/batch/core/schema-postgresql.sql
// /org/springframework/batch/core/schema-oracle.sql
// ...
```

### 5. 전체 초기화 흐름 ASCII 다이어그램

```
Spring Boot 애플리케이션 시작
    │
    ▼
AutoConfiguration 처리 (spring.factories / AutoConfiguration.imports)
    │
    ├─ DataSourceAutoConfiguration
    │    → DataSource Bean 생성 (MySQL 연결 풀)
    │
    ├─ TransactionAutoConfiguration
    │    → PlatformTransactionManager Bean 생성
    │
    ├─ BatchAutoConfiguration (@EnableBatchProcessing 없을 때만)
    │    ├─ BatchRepositoryConfiguration
    │    │    → JobRepository (JdbcJobRepository) Bean 생성
    │    │    → JobExplorer Bean 생성
    │    ├─ JobLauncher Bean 생성 (SimpleJobLauncher)
    │    ├─ JobRegistry Bean 생성
    │    └─ JobLauncherApplicationRunner Bean 생성
    │         → ApplicationRunner → 앱 시작 시 Job 자동 실행
    │
    ├─ BatchDataSourceScriptDatabaseInitializer
    │    → initialize-schema 설정에 따라 DDL 실행
    │    → BATCH_JOB_INSTANCE 등 6개 테이블 생성
    │
    └─ 사용자 정의 @Configuration (Job, Step, Reader, Writer Bean)
         → JobRegistryBeanPostProcessor가 Job Bean 자동 감지 → JobRegistry 등록
```

---

## 💻 실전 구현

### Spring Boot 3.x 커스텀 BatchConfiguration

```java
// DefaultBatchConfiguration 상속으로 커스터마이징
@Configuration
public class CustomBatchConfig extends DefaultBatchConfiguration {

    @Autowired
    private DataSource dataSource;

    @Autowired
    private PlatformTransactionManager transactionManager;

    // JobRepository 커스터마이징 (테이블 prefix 변경 등)
    @Override
    protected DataSource getDataSource() {
        return dataSource;
    }

    @Override
    protected PlatformTransactionManager getTransactionManager() {
        return transactionManager;
    }

    @Override
    protected String getTablePrefix() {
        return "MY_BATCH_";  // 기본값: "BATCH_" → 테이블 prefix 변경
    }

    @Override
    protected int getMaxVarCharLength() {
        return 1000;  // EXIT_MESSAGE 최대 길이 조정
    }

    // 비동기 JobLauncher 오버라이드
    @Bean
    @Override
    public JobLauncher jobLauncher() throws BatchConfigurationException {
        TaskExecutorJobLauncher launcher = new TaskExecutorJobLauncher();
        launcher.setJobRepository(jobRepository());
        launcher.setTaskExecutor(new SimpleAsyncTaskExecutor("batch-"));
        try {
            launcher.afterPropertiesSet();
        } catch (Exception e) {
            throw new BatchConfigurationException(e);
        }
        return launcher;
    }
}
```

### 멀티 DataSource 환경 설정 (배치 전용 DB 분리)

```java
@Configuration
public class BatchDataSourceConfig {

    // 배치 메타데이터 전용 DataSource (별도 DB)
    @Bean
    @ConfigurationProperties("spring.batch.datasource")
    public DataSource batchDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    public PlatformTransactionManager batchTransactionManager(
            @Qualifier("batchDataSource") DataSource batchDataSource) {
        return new DataSourceTransactionManager(batchDataSource);
    }

    @Bean
    public JobRepository jobRepository(
            @Qualifier("batchDataSource") DataSource batchDataSource,
            @Qualifier("batchTransactionManager") PlatformTransactionManager txManager)
            throws Exception {
        JobRepositoryFactoryBean factory = new JobRepositoryFactoryBean();
        factory.setDataSource(batchDataSource);
        factory.setTransactionManager(txManager);
        factory.setIsolationLevelForCreate("ISOLATION_SERIALIZABLE");
        factory.afterPropertiesSet();
        return factory.getObject();
    }
}

# application.yml
spring:
  batch:
    datasource:
      url: jdbc:mysql://batch-db:3306/batchmeta  # 배치 메타데이터 전용 DB
      username: batch_user
    jdbc:
      initialize-schema: never  # flyway로 관리
  datasource:
    url: jdbc:mysql://app-db:3306/appdata  # 비즈니스 데이터 DB
```

---

## 📊 주요 BatchProperties 설정

```yaml
spring:
  batch:
    job:
      enabled: true           # false: JobLauncherApplicationRunner 비활성화
      name: orderSettlementJob # 특정 Job만 자동 실행 (없으면 모든 Job 실행)
    jdbc:
      initialize-schema: embedded  # always | embedded | never
      table-prefix: BATCH_         # 테이블 prefix (기본값)
      isolation-level-for-create: SERIALIZABLE  # JobInstance 생성 시 격리 수준
```

---

## ⚖️ 트레이드오프

```
@EnableBatchProcessing vs BatchAutoConfiguration:

  @EnableBatchProcessing (수동 설정)
    장점  Spring Boot 없이 사용 가능 (순수 Spring Framework)
          세밀한 Bean 커스터마이징 가능
    단점  Spring Boot 3.x에서 자동 구성 비활성화
          BatchProperties, application.yml 설정 무시됨

  BatchAutoConfiguration (Spring Boot 자동 구성)
    장점  설정 최소화, Spring Boot 생태계와 통합
          spring.batch.* 프로퍼티로 간편 설정
    단점  @EnableBatchProcessing과 공존 불가
          자동 구성이 어떻게 동작하는지 알아야 커스터마이징 가능

DataSource 유무:

  DB 기반 JobRepository (운영)
    장점  재시작, 중복 방지, 이력 관리
    단점  DB 테이블 필요, 매 Chunk마다 DB UPDATE

  InMemoryJobRepository (테스트)
    장점  설정 불필요, 빠른 테스트
    단점  재시작 불가, 이력 없음, 서버 재시작 시 유실
```

---

## 📌 핵심 정리

```
Spring Boot 버전별 설정 방식

  Spring Boot 2.x  → @EnableBatchProcessing 필요
                     BatchConfigurer 구현으로 커스터마이징

  Spring Boot 3.x  → @EnableBatchProcessing 불필요 (오히려 자동 구성 깨짐)
                     BatchAutoConfiguration이 자동으로 모든 인프라 Bean 등록
                     DefaultBatchConfiguration 상속으로 커스터마이징

BatchAutoConfiguration이 등록하는 주요 Bean
  JobRepository       → JDBC (DataSource 있을 때) / InMemory (없을 때)
  JobLauncher         → SimpleJobLauncher (SyncTaskExecutor 기본)
  JobExplorer         → 실행 이력 조회 (읽기 전용)
  JobRegistry         → Job Bean 목록 관리
  JobLauncherApplicationRunner → 앱 시작 시 Job 자동 실행

핵심 spring.batch 설정
  spring.batch.job.enabled=false      → 자동 실행 비활성화 (REST API 제어용)
  spring.batch.job.name=specificJob   → 특정 Job만 자동 실행
  spring.batch.jdbc.initialize-schema → 스키마 자동 생성 여부
```

---

## 🤔 생각해볼 문제

**Q1.** `spring.batch.jdbc.initialize-schema=always` 설정은 운영 환경에서 위험합니다. 왜 그런가? 그리고 운영 환경에서 배치 스키마를 관리하는 권장 방법은?

**Q2.** 같은 Spring Boot 애플리케이션에 두 개의 독립적인 배치 Job이 있고, 각 Job이 서로 다른 DataSource를 사용해야 합니다. `JobRepository`는 하나인데 어떻게 설정해야 하는가?

**Q3.** `spring.batch.job.enabled=false`로 설정한 후 특정 조건(예: 매일 특정 시간)에만 Job을 실행하는 패턴을 구현하려면 어떻게 해야 하는가?

> 💡 **해설**
>
> **Q1.** `initialize-schema=always`는 앱 시작마다 `DROP TABLE IF EXISTS`를 실행하고 테이블을 재생성합니다. 운영 환경에서는 기존 배치 이력이 모두 삭제됩니다. 권장 방법: (1) **Flyway** — `spring.batch.jdbc.initialize-schema=never`로 설정하고 `V1__create_batch_schema.sql`에 Spring Batch 스키마를 추가합니다. (2) **Liquibase** — 마찬가지로 배치 스키마를 changeset으로 관리합니다. (3) DBA가 직접 초기 스키마 생성 후 `never`로 유지합니다. 두 번 이상 실행해도 안전한 `CREATE TABLE IF NOT EXISTS` 형식으로 작성하는 것이 중요합니다.
>
> **Q2.** `JobRepository`는 배치 메타데이터 전용으로 하나의 DataSource를 사용합니다. 각 Job의 Step에서 사용하는 `ItemReader`/`ItemWriter`가 다른 DataSource를 사용하게 하되, Chunk 트랜잭션 관리를 위해 `ChainedTransactionManager` 또는 XA 트랜잭션(`JtaTransactionManager`)을 사용합니다. 단, XA는 복잡하므로 가능하면 각 Step이 하나의 DataSource만 사용하도록 설계하고, `JobRepository`용 DB와 비즈니스 DB를 분리하는 것이 권장됩니다.
>
> **Q3.** `spring.batch.job.enabled=false`로 자동 실행을 끈 후, `@Scheduled` 어노테이션으로 스케줄링합니다. `@EnableScheduling`을 추가하고, `@Scheduled(cron = "0 0 2 * * *")` (매일 새벽 2시)으로 메서드를 설정합니다. 해당 메서드에서 `JobLauncher.run(job, params)`를 호출합니다. 더 복잡한 스케줄링이 필요하면 Quartz Scheduler와 통합하거나, Spring Integration의 `PollingConsumer`를 사용합니다. 운영 환경에서는 여러 인스턴스가 동시에 실행을 시도할 수 있으므로 `ShedLock` 같은 분산 잠금 라이브러리와 함께 사용하는 것이 좋습니다.

---

<div align="center">

**[⬅️ 이전: JobParameters와 실행 컨텍스트](./05-job-parameters-context.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 2 — Chunk 처리 원리 ➡️](../chunk-processing/01-chunk-oriented-processing.md)**

</div>
