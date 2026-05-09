# Testcontainers — 진짜 인프라로 통합 테스트하기

> 이 프로젝트의 모든 통합 테스트가 사용하는 도구. `TransactionalEventPublishingTest`를 따라가며 동작 원리와 사용법을 정리한 학습 노트.

---

## 0. 분석 대상 코드

```java
// common/src/test/java/com/hybrid/common/event/TransactionalEventPublishingTest.java
package com.hybrid.common.event;

import java.math.BigDecimal;
import java.util.concurrent.atomic.AtomicLong;

import com.hybrid.common.event.EventStore;
import com.hybrid.common.event.TransactionalEventPublisher;
import com.hybrid.common.event.contract.OrderCreated;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.support.TransactionTemplate;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@SpringBootTest
@Testcontainers
class TransactionalEventPublishingTest {

    @Container
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:17");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", pg::getJdbcUrl);
        r.add("spring.datasource.username", pg::getUsername);
        r.add("spring.datasource.password", pg::getPassword);
    }

    @Autowired TransactionalEventPublisher publisher;
    @Autowired PlatformTransactionManager txm;
    @Autowired EventStore store;

    @Test
    void 트랜잭션_커밋_후에_이벤트가_디스패치된다() {
        TransactionTemplate tx = new TransactionTemplate(txm);
        AtomicLong offsetDuringTx = new AtomicLong(-1);

        tx.executeWithoutResult(status -> {
            publisher.publish(new OrderCreated(1L, BigDecimal.TEN));
            offsetDuringTx.set(store.latestOffset());        // 아직 append 전
        });

        assertThat(offsetDuringTx.get()).isZero();
        assertThat(store.latestOffset()).isEqualTo(1L);       // 커밋 후
    }

    @Test
    void 트랜잭션_롤백_시_이벤트는_폐기된다() {
        TransactionTemplate tx = new TransactionTemplate(txm);

        assertThatThrownBy(() -> tx.executeWithoutResult(status -> {
            publisher.publish(new OrderCreated(1L, BigDecimal.TEN));
            throw new RuntimeException("rollback");
        })).isInstanceOf(RuntimeException.class);

        assertThat(store.latestOffset()).isZero();
    }
}
```

이 짧은 클래스 안에 **JUnit 5 + Spring Boot Test + Testcontainers + PostgreSQL + 트랜잭션 동기화** 가 모두 모여 있다. 하나씩 풀어보자.

---

## 1. Testcontainers란

> "Throwaway, lightweight instances of real databases, message brokers, web browsers, or just about anything that can run in a Docker container." — testcontainers.org

자바 테스트 코드 안에서 **Docker 컨테이너를 자동 기동/정리**해주는 라이브러리. 통합 테스트가 필요할 때 진짜 인프라(PostgreSQL, Kafka, Redis, Selenium 등)를 컨테이너로 띄워서 검증한다.

### 1.1 왜 필요한가

기존 자바 통합 테스트의 선택지:

| 접근 | 장점 | 한계 |
|------|------|------|
| **Mock** (Mockito 등) | 빠름, 외부 의존 없음 | 진짜 DB 동작과 다름. 마이그레이션·SQL·트랜잭션 검증 불가 |
| **임베디드 DB** (H2, HSQLDB) | 빠름, 의존 없음 | 운영 DB와 SQL 방언·기능 차이 (예: H2의 JSONB는 PostgreSQL과 다름) |
| **수동 Docker / 로컬 설치** | 진짜 DB | 환경 설정 부담, CI에서 재현성 떨어짐, 테스트 격리 안 됨 |
| **Testcontainers** | 진짜 DB + 자동 기동/정리 + 격리 | Docker 의존, 첫 실행 시 이미지 풀 시간 |

핵심 가치는 **"운영과 동일한 인프라로 테스트한다"**. 이 프로젝트가 PostgreSQL 17, Kafka 3.9를 운영 대상으로 선언했으니, 테스트도 같은 버전으로 돌린다. SQL 방언 차이, 트랜잭션 동작 차이, JSONB 같은 PostgreSQL 전용 기능까지 정확히 검증된다.

### 1.2 의존성

```groovy
// build.gradle (root subprojects 블록)
testImplementation 'org.testcontainers:junit-jupiter:1.20.4'
testImplementation 'org.testcontainers:postgresql:1.20.4'
testImplementation 'org.testcontainers:kafka:1.20.4'
```

- `junit-jupiter`: JUnit 5 통합 (`@Testcontainers`, `@Container` 어노테이션 제공)
- `postgresql`, `kafka`: 각 컨테이너 모듈 (PostgreSQLContainer, KafkaContainer)

각 인프라마다 별도 모듈이 있고, 일반 컨테이너는 `GenericContainer`로 직접 다룰 수 있다.

### 1.3 동작 원리 한 줄

```
@Container 필드 발견 → Docker 데몬에 컨테이너 기동 요청
              → 무작위 포트 매핑·환경변수 주입
              → JDBC URL/bootstrap-servers 등을 동적으로 노출
              → 테스트 실행
              → 테스트 종료 시 컨테이너 정리 (Ryuk 데몬이 보장)
```

---

## 2. 어노테이션 풀이

### 2.1 `@Testcontainers`

```java
@Testcontainers
class TransactionalEventPublishingTest { ... }
```

JUnit 5 확장(extension) 등록.

- `@Container` 필드를 자동 감지.
- 테스트 라이프사이클에 맞춰 컨테이너의 `start()` / `stop()` 호출.
- 없으면 `@Container`만 써도 컨테이너가 안 뜨므로 NullPointerException 만남.

### 2.2 `@Container`

```java
@Container
static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:17");
```

이 필드를 라이프사이클 관리 대상으로 표시.

**`static` 여부에 따라 의미가 갈린다:**

| 선언 | 의미 |
|------|------|
| `static` (위 코드) | 클래스 단위(`@BeforeAll`/`@AfterAll`). **모든 테스트가 같은 컨테이너 공유** |
| 인스턴스 필드 | 메서드 단위(`@BeforeEach`/`@AfterEach`). **테스트마다 컨테이너 새로 기동** |

이 프로젝트는 **`static`** 을 쓴다. 이유는 명백하다:

- PostgreSQL 컨테이너 부팅이 ~3-5초.
- 테스트 메서드마다 새로 띄우면 클래스 안에 메서드 5개면 25초 추가.
- 같은 클래스의 테스트들은 어차피 격리해야 하니 **테스트 사이의 데이터 정리는 별도로 처리**(`@Transactional` 롤백, truncate 등).

### 2.3 `@DynamicPropertySource`

```java
@DynamicPropertySource
static void props(DynamicPropertyRegistry r) {
    r.add("spring.datasource.url", pg::getJdbcUrl);
    r.add("spring.datasource.username", pg::getUsername);
    r.add("spring.datasource.password", pg::getPassword);
}
```

이게 **Testcontainers와 Spring Boot를 잇는 가장 중요한 다리**다.

문제: PostgreSQL 컨테이너가 띄워지면 그 컨테이너의 **포트는 무작위**다 (`32789`, `49183` 등). Spring의 `application.yml`에 `jdbc:postgresql://localhost:5432/...`라고 박아놓을 수 없다.

해결:

```
1. JUnit이 static 필드(@Container)를 초기화
   → PostgreSQLContainer 인스턴스 생성 (start는 아직 안 됨)

2. @Testcontainers 확장이 컨테이너를 start()
   → 실제 포트 매핑됨 (예: 32789)

3. Spring Boot 컨텍스트가 시작되기 직전에 @DynamicPropertySource 실행
   → r.add(키, Supplier<String>) — Supplier는 lazy
   → Spring은 프로퍼티가 필요할 때마다 pg::getJdbcUrl을 호출
   → 매번 최신 컨테이너 URL을 반환

4. Spring DataSource가 그 URL로 연결
```

**왜 `Supplier` 형태(`pg::getJdbcUrl`)로 등록하는가?** 컨테이너가 아직 start되기 전이면 `getJdbcUrl()`이 예외를 던진다. 람다(메서드 레퍼런스)로 늦게 평가하기 때문에, Spring이 실제로 값이 필요한 시점엔 컨테이너가 이미 떠있다.

### 2.4 비교 — 옛날 방식 (`@TestPropertySource`, `@DirtiesContext`)

Testcontainers 초창기 / Spring 5.2 이전엔:

```java
@TestPropertySource(properties = "spring.datasource.url=" + ???)   // 컴파일 타임 상수만 가능 ❌
```

이래서 `static {}` 블록에서 `System.setProperty` 같은 트릭을 썼었다. Spring 5.2+에서 `@DynamicPropertySource`가 추가되면서 깔끔해졌다.

---

## 3. `PostgreSQLContainer` 자세히 보기

```java
new PostgreSQLContainer<>("postgres:17")
```

- 인자는 Docker 이미지 태그. `"postgres:17"`은 Docker Hub의 공식 PostgreSQL 17 이미지.
- 제네릭 와일드카드 `<?>`는 자기 참조 제네릭의 표현. 보통 그냥 `<?>`로 둔다.

### 3.1 자동으로 해주는 것

```java
PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:17")
    .withDatabaseName("hybrid")
    .withUsername("hybrid")
    .withPassword("hybrid");
```

설정 안 해도 **기본값**이 들어있다.

| 설정 | 기본값 |
|------|------|
| 데이터베이스 이름 | `test` |
| 사용자 | `test` |
| 비밀번호 | `test` |
| 포트 | 5432 (호스트 쪽은 무작위 할당) |

생성 후 `start()` 시:

1. Docker 이미지 풀(처음 한 번).
2. `docker run`으로 컨테이너 띄움.
3. 환경변수로 DB/USER/PASSWORD 주입.
4. **헬스 체크** — PostgreSQL이 연결 받을 준비 될 때까지 대기.
5. `getJdbcUrl()`이 유효한 URL 반환.

### 3.2 핵심 메서드

```java
pg.getJdbcUrl()       // jdbc:postgresql://localhost:32789/test
pg.getUsername()      // test
pg.getPassword()      // test
pg.getMappedPort(5432)  // 32789 (호스트 쪽 매핑된 포트)
pg.getHost()          // localhost
pg.execInContainer("psql", "-c", "SELECT 1");   // 컨테이너 안에서 명령 실행
```

### 3.3 모듈별 컨테이너

| 클래스 | 인프라 |
|-------|------|
| `PostgreSQLContainer` | PostgreSQL |
| `MySQLContainer` | MySQL |
| `KafkaContainer` | Kafka (이 프로젝트 Phase 2) |
| `MongoDBContainer` | MongoDB |
| `RedisContainer` (서드파티) | Redis |
| `LocalStackContainer` | AWS 시뮬레이션 |
| `GenericContainer` | 임의 이미지 |

```java
// GenericContainer 예시
GenericContainer<?> redis = new GenericContainer<>("redis:7")
    .withExposedPorts(6379);
```

---

## 4. 테스트 코드 한 줄씩 따라가기

이제 도구들을 다 알았으니, `TransactionalEventPublishingTest`의 **전체 흐름**을 시간순으로 풀어본다.

### 4.1 클래스 로딩 ~ 컨테이너 기동

```java
@SpringBootTest                                  // Spring Boot 전체 컨텍스트 로드
@Testcontainers                                  // Testcontainers 확장 활성화
class TransactionalEventPublishingTest {

    @Container
    static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:17");
```

```
[T+0.0s]  JVM이 클래스 로드 → static 필드 pg 초기화 (인스턴스만 생성, start 전)
[T+0.0s]  JUnit 5가 @Testcontainers 확장 활성화
[T+0.1s]  확장이 @BeforeAll 시점에 pg.start() 호출
[T+0.1s]  Docker 데몬에 "postgres:17 컨테이너 시작" 요청
[T+~3s]   컨테이너 부팅 완료, 포트 32789로 매핑됨
[T+~3s]   Testcontainers가 헬스체크 통과 확인
```

이 동안 별도 컨테이너 **Ryuk**도 함께 돈다. Ryuk은 Testcontainers의 정리 보안관 — JVM이 비정상 종료되어도 Docker 컨테이너가 좀비처럼 남지 않게 보장한다.

### 4.2 Spring 컨텍스트 시작

```java
    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", pg::getJdbcUrl);
        r.add("spring.datasource.username", pg::getUsername);
        r.add("spring.datasource.password", pg::getPassword);
    }
```

```
[T+~3s]  Spring Boot 컨텍스트 초기화 시작
[T+~3s]  @DynamicPropertySource 메서드 실행 → 동적 프로퍼티 등록
         (Supplier 형태라 실제 호출은 나중)
[T+~3s]  Spring이 spring.datasource.url 값을 요청
         → pg::getJdbcUrl 호출 → "jdbc:postgresql://localhost:32789/test"
[T+~4s]  HikariCP 커넥션 풀이 그 URL로 연결
[T+~4s]  Flyway 마이그레이션 실행 (있다면)
[T+~5s]  TransactionalEventPublisher, EventStore, JpaTransactionManager 등 빈 구성 완료
```

### 4.3 의존성 주입

```java
    @Autowired TransactionalEventPublisher publisher;
    @Autowired PlatformTransactionManager txm;
    @Autowired EventStore store;
```

Spring이 위 세 빈을 테스트 인스턴스 필드에 주입한다.

### 4.4 첫 테스트 — 커밋 후 디스패치

```java
@Test
void 트랜잭션_커밋_후에_이벤트가_디스패치된다() {
    TransactionTemplate tx = new TransactionTemplate(txm);
    AtomicLong offsetDuringTx = new AtomicLong(-1);

    tx.executeWithoutResult(status -> {
        publisher.publish(new OrderCreated(1L, BigDecimal.TEN));   // ①
        offsetDuringTx.set(store.latestOffset());                   // ②
    });

    assertThat(offsetDuringTx.get()).isZero();                      // ③
    assertThat(store.latestOffset()).isEqualTo(1L);                 // ④
}
```

흐름:

```
[1] tx.executeWithoutResult — 트랜잭션 시작
    │
    ├─ ① publisher.publish(OrderCreated)
    │     └─ TransactionalEventPublisher가 TransactionSynchronization 등록
    │        (즉시 실행 안 됨 — afterCommit에 예약)
    │
    ├─ ② store.latestOffset() = 0
    │     └─ 아직 EventStore에 append 안 됐음 (트랜잭션 커밋 전)
    │
    └─ 람다 끝 → 트랜잭션 커밋
       └─ afterCommit 훅 발화 → bus.publish(event) 실제 호출
          └─ store.append(event) → offset 1 부여

[2] 트랜잭션 종료 후 검증
    ③ offsetDuringTx == 0  ← 트랜잭션 안에서는 0이었다는 증거
    ④ store.latestOffset() == 1  ← 커밋 후엔 1이라는 증거
```

이 두 단언이 함께 통과해야 **"커밋 후 디스패치"** 라는 계약이 증명된다.

### 4.5 두 번째 테스트 — 롤백 시 폐기

```java
@Test
void 트랜잭션_롤백_시_이벤트는_폐기된다() {
    TransactionTemplate tx = new TransactionTemplate(txm);

    assertThatThrownBy(() -> tx.executeWithoutResult(status -> {
        publisher.publish(new OrderCreated(1L, BigDecimal.TEN));   // ①
        throw new RuntimeException("rollback");                     // ②
    })).isInstanceOf(RuntimeException.class);

    assertThat(store.latestOffset()).isZero();                      // ③
}
```

흐름:

```
[1] tx.executeWithoutResult — 트랜잭션 시작
    │
    ├─ ① publisher.publish — afterCommit 훅 등록 (아직 실행 안 됨)
    │
    └─ ② RuntimeException 발생
       └─ TransactionTemplate이 예외 감지 → rollback() 호출
          └─ afterCommit 훅이 호출되지 않음 (커밋이 안 됐으니까)
          └─ afterCompletion(STATUS_ROLLED_BACK)만 발화

[2] 예외가 위로 전파됨 → assertThatThrownBy가 잡음

[3] 검증
    ③ store.latestOffset() == 0
       ← 이벤트가 EventStore에 append되지 않았다는 증거
       (즉, 도메인 변경은 없었던 일이 됐고 이벤트도 발행되지 않았다)
```

> **이게 phase-1-tdd.md Step 4의 핵심 계약** — "트랜잭션 롤백 시 이벤트가 발행되지 않는다". Outbox 패턴의 인메모리 버전이다.

### 4.6 정리

```
[모든 @Test 종료]
[T+종료]  @Testcontainers 확장이 pg.stop() 호출
          → 컨테이너 종료 + 정리
          → Ryuk가 정리됐는지 확인
```

---

## 5. 자주 마주치는 패턴

### 5.1 공통 베이스 클래스로 추출

여러 테스트가 같은 컨테이너 설정을 쓰면 베이스 클래스로 뽑는다.

```java
// common/src/test/java/com/hybrid/common/support/KafkaIntegrationTestBase.java
@SpringBootTest
@Testcontainers
public abstract class KafkaIntegrationTestBase {

    @Container
    static final PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:17");

    @Container
    static final KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.7.0"));

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", pg::getJdbcUrl);
        r.add("spring.datasource.username", pg::getUsername);
        r.add("spring.datasource.password", pg::getPassword);
        r.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }
}
```

이 프로젝트의 Phase 2/3 모든 통합 테스트가 이걸 상속한다.

### 5.2 컨테이너 재사용 (`.withReuse(true)`)

```java
@Container
static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:17")
    .withReuse(true);
```

각 테스트 클래스마다 컨테이너를 새로 띄우는 대신 **재사용**한다. 로컬 개발 시 매우 빠르지만:

- `~/.testcontainers.properties` 에 `testcontainers.reuse.enable=true` 가 필요.
- CI에서는 보통 끄고 매번 새로 (테스트 격리·결정성).
- 데이터 정리는 `@Transactional` 롤백이나 `truncate`로 직접 챙겨야 함.

### 5.3 Singleton 컨테이너 패턴

```java
public class ContainerSingleton {
    public static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:17");

    static { POSTGRES.start(); }   // 한 번만 시작
}
```

테스트 클래스가 100개라도 컨테이너는 하나. JVM 종료 시 Ryuk이 정리. 가장 빠른 패턴이지만 격리 설계가 더 어렵다.

### 5.4 Wait 전략 — 컨테이너가 "준비됐다"를 정의

기본 컨테이너는 자동 헬스체크가 있지만, 커스텀이 필요하면:

```java
new GenericContainer<>("my-app:latest")
    .waitingFor(Wait.forHttp("/healthz").forStatusCode(200))
    .withStartupTimeout(Duration.ofSeconds(30));
```

`Wait.forLogMessage(...)`, `Wait.forListeningPort()`, `Wait.forSuccessfulCommand(...)` 등 다양한 전략 제공.

---

## 6. 트레이드오프

### 6.1 비용

| 비용 | 설명 |
|------|------|
| **첫 실행 이미지 풀 시간** | postgres:17 ≈ 100MB, kafka ≈ 600MB. 처음 한 번만. |
| **컨테이너 부팅 시간** | PG ≈ 3-5초, Kafka ≈ 10-15초. 매 테스트 클래스마다. |
| **Docker 의존** | Docker 데몬이 떠있어야 함. CI는 docker-in-docker 또는 dind. |
| **CPU/메모리** | 컨테이너 + Ryuk + Spring Boot 컨텍스트. 많은 테스트를 병렬로 돌릴 때 부담. |

### 6.2 가치

| 가치 | 설명 |
|------|------|
| **운영 동등성** | H2 같은 가짜와 다르게 진짜 PostgreSQL의 SQL/트랜잭션/JSONB 등 그대로 검증 |
| **테스트 격리** | 컨테이너 단위로 깨끗한 환경. 테스트 간섭 거의 없음 |
| **재현성** | "내 머신에선 되는데"가 사라짐. 같은 이미지 = 같은 환경 |
| **Mock 회피** | 진짜 동작을 검증하므로 Mock 동기화 부담 없음 |
| **CI 친화** | docker만 있으면 어디서나 동일하게 동작 |

### 6.3 언제 안 쓰는가

- 순수 도메인 단위 테스트 — `OrderTest`처럼 외부 인프라가 없는 경우. POJO만 있으면 굳이 컨테이너 필요 없음.
- 슬라이스 테스트 — `@WebMvcTest`처럼 특정 레이어만 검증할 때.
- 마이크로 벤치마크 — JMH 등으로 코드 자체의 성능을 측정할 때.

---

## 7. 이 프로젝트에서의 사용처

| 테스트 | 사용 컨테이너 | 검증 대상 |
|--------|------------|---------|
| `TransactionalEventPublishingTest` | PostgreSQL | 트랜잭션 동기화 (커밋/롤백 정책) |
| `OrderServiceTest` | PostgreSQL | 주문 생성 + 이벤트 발행 |
| `PaymentEventHandlerTest` | PostgreSQL | OrderCreated → Payment 처리 |
| `OutboxRelayTest` (Phase 2) | PostgreSQL + Kafka | Outbox 폴링 + Kafka 발행 |
| `InboxConsumerTest` (Phase 2) | PostgreSQL | 멱등성 처리 |
| `KafkaFailureRecoveryTest` (Phase 2) | PostgreSQL + Kafka | Kafka 정지/재기동 시나리오 |
| `Phase2E2ETest` | PostgreSQL + Kafka | Order → Notification 전체 흐름 |
| `NotificationStandaloneTest` (Phase 3) | PostgreSQL + Kafka | Notification만 단독 부팅 검증 |
| `ChaosScenarioTest` (Phase 3) | PostgreSQL + Kafka | 부하·장애 시나리오 |

Phase 1은 PostgreSQL 단독, Phase 2부터는 Kafka 추가. 진화에 따라 베이스 클래스가 다르다.

---

## 8. CI에서 Testcontainers 돌리기

대부분의 CI 환경(GitHub Actions, GitLab CI 등)은 Docker가 기본 설치돼 있어 별도 설정이 거의 없다.

```yaml
# .github/workflows/ci.yml 예시
- uses: actions/checkout@v4
- uses: actions/setup-java@v4
  with:
    java-version: '25'
    distribution: 'temurin'
- run: ./gradlew test
```

GitHub Actions는 호스트에 Docker 데몬이 있어 그대로 돌아간다.

GitLab/Jenkins에서 docker-in-docker(dind)를 쓰면 약간의 설정 필요:

```yaml
services:
  - docker:dind
variables:
  TESTCONTAINERS_HOST_OVERRIDE: "docker"
  DOCKER_HOST: "tcp://docker:2375"
```

---

## 9. 한 줄 요약

> **Testcontainers는 "Docker 컨테이너로 진짜 인프라를 띄워 통합 테스트하는 자바 라이브러리"이고, `@Container`로 라이프사이클을 관리하고 `@DynamicPropertySource`로 무작위 포트를 Spring 프로퍼티에 동적으로 연결한다. 이 프로젝트의 통합 테스트는 전부 이 도구를 통해 운영과 동일한 PostgreSQL/Kafka에서 실행된다.**

`TransactionalEventPublishingTest`는 이 도구의 모든 핵심 요소(`@Testcontainers`, `@Container static`, `@DynamicPropertySource`, `PostgreSQLContainer`)를 한 클래스에 응축한 좋은 입문 예제다. 여기서 패턴을 익히면 Phase 2/3의 모든 통합 테스트가 같은 구조다.

---

## 10. 함께 보면 좋은 자료

- 공식 사이트: https://testcontainers.com/
- `docs/phase-1-tdd.md` Step 4 — 이 테스트의 TDD 구현 사이클
- `docs/phase-2-tdd.md` 0.3 — `KafkaIntegrationTestBase` 공통 베이스
- Spring Boot 공식: https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#features.testing.testcontainers
- Sergei Egorov, *Testcontainers: A Library for Integration Testing with Real Dependencies*
