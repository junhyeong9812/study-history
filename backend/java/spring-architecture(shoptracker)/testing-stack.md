# 테스트 스택: JUnit 5 + Testcontainers + Modulith Scenario

> 이 프로젝트는 3계층 테스트 전략을 사용 — 단위 / 통합 / 모듈 시나리오.

## 1. JUnit 5 (Jupiter)

자바 표준 테스트 프레임워크. JUnit 4 대비 모듈화/람다 친화/확장 모델 정비.

```java
@Test
void createOrder_calculatesTotal() {
    Order order = Order.create("홍길동", List.of(
        new OrderItem("노트북", 1, new Money(1000000))));
    assertThat(order.getTotalAmount()).isEqualTo(new Money(1050000));
}
```

핵심 어노테이션:
- `@Test` 단일 테스트
- `@ParameterizedTest` 여러 입력
- `@BeforeEach`, `@AfterEach`, `@BeforeAll`, `@AfterAll`
- `@DisplayName` 한글 가능
- `@Nested` 중첩 클래스로 그룹화
- `@ExtendWith` 확장 (`@MockitoExtension`, `@SpringExtension` 등)

AssertJ와 함께: `assertThat(...).isEqualTo(...)`, `.hasMessageContaining(...)`, `.isInstanceOf(...)`.

## 2. 단위 테스트 — DB 없이

이 프로젝트의 도메인은 Spring/JPA 의존성이 0이라 단위 테스트가 0.x초 안에 끝난다.

```
src/test/java/com/shoptracker/unit/
├── MoneyTest.java
├── OrderTest.java
├── OrderStatusTransitionTest.java
├── SubscriptionTest.java
├── DiscountPolicyTest.java
└── ShippingFeePolicyTest.java
```

```bash
./gradlew test --tests "com.shoptracker.unit.*"
```

도메인이 깨끗하면 단위 테스트가 단위 테스트답게 빠르고, "이걸 매 푸시마다 돌리는" 흐름이 자연스럽다.

## 3. Testcontainers — 진짜 DB 띄우기

Mock DB는 마이그레이션 차이/드라이버 차이를 못 잡는다. Testcontainers는 테스트 시작 시 **Docker 컨테이너로 실제 PostgreSQL** 을 띄운다.

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
@Testcontainers
class PaymentWithSubscriptionTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @Autowired MockMvc mockMvc;

    @Test
    void premiumSubscriber_gets10PercentDiscount() throws Exception {
        ... // 실제 PostgreSQL에서 실행됨
    }
}
```

핵심:
- `@Testcontainers` 확장이 컨테이너 라이프사이클 관리.
- `@Container static`: 클래스당 한 번만 띄우고 모든 테스트가 공유.
- `@ServiceConnection` (Boot 3.1+): 컨테이너 정보를 자동으로 `spring.datasource.*` 에 주입. 직접 `@DynamicPropertySource` 안 써도 됨.

장점: Flyway 마이그레이션이 실제로 실행되고, JPA가 실제 PostgreSQL 방언으로 SQL 생성.

## 4. MockMvc — 컨트롤러 테스트

서블릿 컨테이너 없이도 컨트롤러 호출을 시뮬레이션.

```java
String response = mockMvc.perform(post("/api/v1/orders")
        .contentType(MediaType.APPLICATION_JSON)
        .header("X-Customer-Name", "홍길동")
        .content("""
            {"customerName":"홍길동","items":[{"productName":"노트북","quantity":1,"unitPrice":1000000}]}
            """))
    .andExpect(status().isCreated())
    .andReturn().getResponse().getContentAsString();
```

- 실제 HTTP 스택 안 거치고 디스패처 서블릿만 호출 → 빠름.
- `jsonPath("$.appliedDiscountType").value("premium_subscription")` 으로 응답 검증.

Spring Boot 4 에서는 `spring-boot-webmvc-test` 모듈에 분리됐다 (이 프로젝트의 build.gradle 참고).

## 5. Spring Modulith Scenario

이벤트 기반 흐름을 시나리오로 검증.

```java
@ApplicationModuleTest
class PaymentsModuleTest {
    @Test
    void orderCreated_triggersPayment(Scenario scenario) {
        scenario.publish(new OrderCreatedEvent(...))
            .andWaitForEventOfType(PaymentApprovedEvent.class)
            .matching(e -> e.orderId() != null)
            .toArriveAndVerify(event -> {
                assert event.finalAmount() != null;
            });
    }
}
```

- `@ApplicationModuleTest`: 해당 모듈만 부팅 (Spring 컨텍스트 슬라이스).
- `Scenario.publish().andWaitForEventOfType()`: 이벤트가 비동기로 처리되어도 결과를 기다려 검증.

## 6. Awaitility — 비동기 단언

Spring Modulith의 `@ApplicationModuleListener`는 비동기로 동작. 결과를 폴링하려면:

```java
import static org.awaitility.Awaitility.*;
import static java.time.Duration.*;

await().atMost(ofSeconds(5)).untilAsserted(() -> {
    mockMvc.perform(get("/api/v1/payments/order/" + orderId))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.appliedDiscountType").value("premium_subscription"));
});
```

`Thread.sleep(5000)` 같은 어림짐작 대신 조건이 만족되면 즉시 통과.

## 7. Mockito (참고 — 이 프로젝트는 거의 안 씀)

```java
@MockitoBean PaymentGateway gateway;
when(gateway.process(any())).thenReturn(new GatewayResult(false, null, "force fail"));
```

Spring Boot 3.4+ 의 `@MockitoBean`이 `@MockBean` 을 대체. 이 프로젝트는 Fake 구현체(`FakePaymentGateway`) 를 쓰는 편이라 Mockito 사용이 적음. 도메인이 깨끗하면 mock 자체가 덜 필요.

## 실행

```bash
./gradlew test                          # 전체
./gradlew test --tests "*.unit.*"       # 단위만
./gradlew test --tests "*ModuleTest"    # 모듈 테스트
./gradlew test --info                   # 자세한 로그
```

리포트: `build/reports/tests/test/index.html`

## FastAPI/pytest 대응

| pytest + httpx | JUnit 5 + MockMvc |
|----------------|--------------------|
| `pytest` | `@Test` |
| `pytest fixture` | `@BeforeEach` |
| `httpx.AsyncClient` | `MockMvc` |
| `pytest-asyncio` | (Spring은 동기 테스트가 기본) |
| `Faker` | (별도 라이브러리, 이 프로젝트는 미사용) |
| `dockerized DB` 직접 | `Testcontainers` |
| `freeze_time` | `Clock` 빈 주입 패턴 |
