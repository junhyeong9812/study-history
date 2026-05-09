# 🏗️ Spring Boot 모듈러 모놀리스 설계 문서

## 프로젝트명: **ShopTracker (Spring Edition)**

> 주문 → 결제 → 배송 흐름을 모듈러 모놀리스로 구현하며,
> Spring Boot 4 + Hexagonal Architecture의 핵심 패턴을 체감하는 학습 프로젝트

---

## 1. 프로젝트 목표

### 학습 목표 (이 프로젝트를 통해 체감할 것들)

| # | 패턴 | 체감 포인트 |
|---|------|------------|
| 1 | **Hexagonal Architecture** | 도메인에 Spring/JPA import가 없는 걸 직접 확인 |
| 2 | **의존성 역전 (DIP)** | Repository를 인터페이스로 정의하고, 구현체를 교체해보기 |
| 3 | **DI + 정책 주입** | 같은 결제 흐름인데 구독 등급에 따라 할인/배송비 정책이 바뀌는 것 |
| 4 | **Spring ApplicationEvent** | 모듈 간 직접 import 없이 이벤트로만 소통 |
| 5 | **CQRS (간소화)** | 주문 생성(Command)과 주문 목록 조회(Query)의 모델 분리 |
| 6 | **Saga / 추적 도메인** | 주문→결제→배송 전체 흐름을 Tracking이 기록 |
| 7 | **횡단 관심사 설계** | 구독권이 결제 할인 + 배송비 할인에 모두 영향, 그러나 느슨하게 |
| 8 | **Spring Modulith** | 모듈 경계 검증, 이벤트 퍼블리싱, 모듈 테스트 |
| 9 | **테스트** | 도메인 단위 테스트 (DB 없이), @SpringBootTest 통합 테스트 |
| 10 | **프로덕션 배포** | Docker Compose + GraalVM 네이티브 or JVM 배포 |

### 비학습 목표 (의도적으로 빼는 것들)

- 실제 PG사 연동 → **Fake Payment Gateway**로 대체
- 프론트엔드 → API만 구현 (Swagger UI로 테스트)
- 인증/인가 → 간소화 (X-Customer-Name 헤더, Spring Security까지는 안 감)
- 실제 배송 추적 → 상태 전이 시뮬레이션
- 구독 결제 자동 갱신 → 수동 생성/만료로 단순화

---

## 2. 기술 스택

| 영역 | 기술 | 선택 이유 |
|------|------|-----------|
| **Framework** | Spring Boot 4.0.x | 학습 대상, 최신 모듈화 아키텍처 |
| **Java** | JDK 21 (LTS) | Virtual Threads, Records, Pattern Matching |
| **Spring Framework** | 7.0.x | JSpecify null safety, API versioning |
| **모듈화** | Spring Modulith 2.0 | 모듈 경계 검증, 이벤트 지속성 |
| **DB** | PostgreSQL 16 | 실무 표준 |
| **ORM** | Spring Data JPA + Hibernate 7 | Spring 생태계 표준 |
| **Migration** | Flyway | Spring Boot 기본 통합 |
| **DI** | Spring IoC (내장) | 프레임워크 기본 기능 |
| **Validation** | Jakarta Validation (Bean Validation 3.0) | 표준 검증 |
| **Testing** | JUnit 5 + Testcontainers + MockMvc | Spring 생태계 표준 |
| **Logging** | SLF4J + Logback (Structured JSON) | Spring Boot 기본 |
| **Build** | Gradle 9 (Kotlin DSL) | Spring Boot 4 지원 |
| **Container** | Docker Compose | PostgreSQL + App 구성 |
| **Observability** | spring-boot-starter-opentelemetry | Spring Boot 4 내장 |

### FastAPI ↔ Spring 기술 매핑

| FastAPI 스택 | Spring 스택 | 비고 |
|-------------|------------|------|
| FastAPI | Spring Boot 4 + Spring MVC | 동기 MVC (Virtual Threads 활용) |
| Pydantic v2 | Java Records + Bean Validation | DTO 검증 |
| SQLAlchemy 2.0 (async) | Spring Data JPA + Hibernate 7 | |
| Alembic | Flyway | 마이그레이션 |
| Dishka (외부 DI) | Spring IoC (내장 DI) | Spring은 DI가 프레임워크 핵심 |
| pydantic-settings | application.yml + @ConfigurationProperties | |
| structlog (JSON) | Logback + spring-boot-starter-opentelemetry | |
| InMemoryEventBus | Spring ApplicationEventPublisher / Spring Modulith Events | |
| Protocol (typing) | Java Interface | Port 정의 |
| pytest + httpx | JUnit 5 + MockMvc + Testcontainers | |
| Gunicorn + Uvicorn | 내장 Tomcat (Virtual Threads) | |

---

## 3. 도메인 설계

### 3.1 Bounded Contexts (모듈)

```
┌──────────────────────────────────────────────────────────────┐
│                      ShopTracker App                         │
│              (Spring Boot 4 + Spring Modulith 2)             │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   orders    │  │  payments   │  │  shipping   │         │
│  │   (주문)    │  │   (결제)    │  │   (배송)    │         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │
│         │                │                │                  │
│         │          ┌─────┴─────┐          │                  │
│         │          │subscription│          │                  │
│         │          │  (구독권)  │          │                  │
│         │          └─────┬─────┘          │                  │
│         │                │                │                  │
│         └────────────────┼────────────────┘                  │
│                          │                                   │
│                 ┌────────▼────────┐                          │
│                 │    tracking     │                          │
│                 │   (추적/Saga)   │                          │
│                 └─────────────────┘                          │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Spring Modulith Events                   │   │
│  │     (ApplicationEventPublisher + @ModuleListener)     │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

> **핵심 규칙**: 모듈 간 직접 import 금지.
> Spring Modulith가 모듈 경계를 **테스트 시점에 검증**해준다.
> Payments는 Subscription 모듈의 내부 클래스를 import하지 않는다.
> 대신 Spring DI가 구독 상태를 기반으로 적절한 **정책 Bean**을 주입한다.

### 3.2 구독권이 다른 모듈에 영향을 주는 방식

```
┌──────────────┐
│ subscription │  구독 등급 정보를 가진 유일한 소스
│    Module    │
└──────┬───────┘
       │
       │  SubscriptionContext (읽기 전용 Record)
       │  → Spring DI가 Request Scope Bean으로 제공
       │
       ├──────────────────────────────────────────┐
       │                                          │
       ▼                                          ▼
┌──────────────┐                        ┌──────────────┐
│  payments    │                        │  shipping    │
│              │                        │              │
│ DiscountPolicy ← Spring이 구독 등급에  │ ShippingFeePolicy ← Spring이 구독 등급에
│   따라 주입      맞는 빈 주입         │   따라 주입        맞는 빈 주입
└──────────────┘                        └──────────────┘
```

> **Spring 특화 설계 포인트**:
> FastAPI의 Dishka와 달리, Spring에서는 `@RequestScope` Bean + `FactoryBean` 패턴 또는
> `@Configuration`의 `@Bean` 메서드에서 조건부 생성으로 구현한다.
> 또 다른 접근으로, Strategy Pattern + `SubscriptionContext`를 파라미터로 받는 방식도 가능하다.

### 3.3 각 모듈의 도메인 엔티티

#### Subscriptions (구독)

```java
// domain/model/Subscription.java — 순수 Java, Spring 의존성 없음
public class Subscription {
    private final SubscriptionId id;
    private final String customerName;
    private final SubscriptionTier tier;
    private SubscriptionStatus status;
    private final Instant startedAt;
    private final Instant expiresAt;

    public boolean isActive() {
        return this.status == SubscriptionStatus.ACTIVE
            && this.expiresAt.isAfter(Instant.now());
    }
}

public enum SubscriptionTier {
    NONE("none"),
    BASIC("basic"),       // 결제 5% 할인, 배송비 50% 할인
    PREMIUM("premium");   // 결제 10% 할인, 배송비 무료

    private final String value;
    SubscriptionTier(String value) { this.value = value; }
    public String getValue() { return value; }
}

public enum SubscriptionStatus {
    ACTIVE, EXPIRED, CANCELLED
}
```

**구독 등급별 혜택 정리:**

| 혜택 | NONE (미구독) | BASIC | PREMIUM |
|------|:---:|:---:|:---:|
| 결제 할인 | 0% | 5% | 10% |
| 배송비 | 정상 (3,000원) | 50% 할인 (1,500원) | 무료 |
| 무료배송 기준 | 50,000원 이상 | 30,000원 이상 | 항상 무료 |

#### SubscriptionContext (모듈 간 공유 Record)

```java
// shared/SubscriptionContext.java — 모듈 간 공유되는 불변 DTO
public record SubscriptionContext(
    String customerName,
    String tier,          // "none", "basic", "premium"
    boolean isActive
) {
    public static SubscriptionContext none(String customerName) {
        return new SubscriptionContext(customerName, "none", false);
    }
}
```

> **설계 포인트**: `SubscriptionContext`는 `shared` 패키지에 위치.
> 다른 모듈이 구독 "상태"를 참고할 수 있게 하는 **얇은 계약(thin contract)**.
> Java의 `record`는 불변성과 equals/hashCode를 자동 보장하여 FastAPI의 `frozen=True` dataclass에 대응.

#### Orders (주문)

```java
// domain/model/Order.java
public class Order {
    private final OrderId id;
    private final String customerName;
    private final List<OrderItem> items;
    private OrderStatus status;
    private final Money totalAmount;
    private Money shippingFee;
    private Money discountAmount;
    private Money finalAmount;
    private final Instant createdAt;

    public static Order create(String customerName, List<OrderItem> items) {
        if (items.isEmpty()) {
            throw new OrderHasNoItemsException();
        }
        Money total = items.stream()
            .map(OrderItem::subtotal)
            .reduce(Money.ZERO, Money::add);
        if (total.isNegativeOrZero()) {
            throw new InvalidOrderAmountException();
        }
        return new Order(
            OrderId.generate(), customerName, items,
            OrderStatus.CREATED, total, Money.ZERO, Money.ZERO, total,
            Instant.now()
        );
    }

    public void transitionTo(OrderStatus newStatus) {
        if (!this.status.canTransitionTo(newStatus)) {
            throw new InvalidStatusTransitionException(this.status, newStatus);
        }
        this.status = newStatus;
    }
}

// domain/model/Money.java — Value Object
public record Money(BigDecimal amount, String currency) {
    public static final Money ZERO = new Money(BigDecimal.ZERO, "KRW");

    public Money(BigDecimal amount) {
        this(amount, "KRW");
    }

    public Money add(Money other) {
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public Money subtract(Money other) {
        return new Money(this.amount.subtract(other.amount), this.currency);
    }

    public Money applyRate(BigDecimal rate) {
        return new Money(this.amount.multiply(rate).setScale(0, RoundingMode.FLOOR), this.currency);
    }

    public boolean isGreaterThanOrEqual(Money other) {
        return this.amount.compareTo(other.amount) >= 0;
    }

    public boolean isNegativeOrZero() {
        return this.amount.compareTo(BigDecimal.ZERO) <= 0;
    }
}

// domain/model/OrderStatus.java
public enum OrderStatus {
    CREATED, PAYMENT_PENDING, PAID, SHIPPING, DELIVERED, CANCELLED;

    public boolean canTransitionTo(OrderStatus target) {
        return switch (this) {
            case CREATED -> target == PAYMENT_PENDING || target == CANCELLED;
            case PAYMENT_PENDING -> target == PAID || target == CANCELLED;
            case PAID -> target == SHIPPING;
            case SHIPPING -> target == DELIVERED;
            case DELIVERED, CANCELLED -> false;
        };
    }
}
```

#### Payments (결제)

```java
// domain/model/Payment.java
public class Payment {
    private final PaymentId id;
    private final OrderId orderId;
    private final Money originalAmount;
    private final Money discountAmount;
    private final Money finalAmount;
    private final PaymentMethod method;
    private PaymentStatus status;
    private final String appliedDiscountType;
    private Instant processedAt;
}

public enum PaymentMethod {
    CREDIT_CARD, BANK_TRANSFER, VIRTUAL_ACCOUNT
}

public enum PaymentStatus {
    PENDING, APPROVED, REJECTED, REFUNDED
}
```

#### Shipping (배송)

```java
// domain/model/Shipment.java
public class Shipment {
    private final ShipmentId id;
    private final OrderId orderId;
    private ShipmentStatus status;
    private final Address address;
    private final Money shippingFee;
    private final Money originalFee;
    private final String feeDiscountType;
    private String trackingNumber;
    private LocalDate estimatedDelivery;
}

public record Address(String street, String city, String zipCode) {}

public enum ShipmentStatus {
    PREPARING, IN_TRANSIT, DELIVERED
}
```

#### Tracking (추적/Saga)

```java
// domain/model/OrderTracking.java
public class OrderTracking {
    private final TrackingId id;
    private final OrderId orderId;
    private final String customerName;
    private final String subscriptionTier;   // 추적 시점의 스냅샷
    private final List<TrackingEvent> events;
    private TrackingPhase currentPhase;
    private final Instant startedAt;
    private Instant completedAt;

    public void addEvent(TrackingEvent event) {
        this.events.add(event);
    }
}

public record TrackingEvent(
    String eventType,
    Instant timestamp,
    String module,
    Map<String, Object> detail
) {}

public enum TrackingPhase {
    ORDER_PLACED, PAYMENT_PROCESSING, PAYMENT_COMPLETED,
    SHIPPING, DELIVERED, FAILED
}
```

### 3.4 이벤트 흐름

FastAPI 버전과 동일한 흐름이지만, **Spring ApplicationEventPublisher**와 **Spring Modulith의 @ApplicationModuleListener**를 사용합니다.

```
[고객 주문 생성]
       │
       ▼
  ┌──────────┐   1. OrderCreatedEvent          ┌──────────────┐
  │  orders  │ ─── (ApplicationEventPublisher) ─▶│ subscription │
  │          │                                  │   (via DI)   │
  └──────────┘                                  └──────┬───────┘
       │                                               │
       │   OrderCreatedEvent                           │ SubscriptionContext
       │   (@ApplicationModuleListener)                ▼
       │                                        ┌─────────────┐
       ├───────────────────────────────────────▶│  payments   │
       │                                        │ 정책 주입:   │
       │                                        │ - 할인율 결정 │
       │                                        │ - Fake 결제  │
       │                                        └──────┬──────┘
       │                                               │
       │                                    ┌──────────┴──────────┐
       │                                    ▼                     ▼
       │                     PaymentApprovedEvent     PaymentRejectedEvent
       │                                    │                     │
       │                     ┌──────────────┤          ┌──────────┤
       │                     ▼              ▼          ▼          ▼
       │              ┌──────────┐   ┌──────────┐ ┌────────┐ ┌──────────┐
       │              │ shipping │   │ tracking │ │ orders │ │ tracking │
       │              │ 배송비   │   │ 기록     │ │→CANCEL │ │ FAILED   │
       │              │ 정책주입 │   └──────────┘ └────────┘ └──────────┘
       │              └────┬─────┘
       │                   ▼
       │        ShipmentCreatedEvent
       │              ┌────┴────┐
       │              ▼         ▼
       │         ┌────────┐ ┌──────────┐
       │         │ orders │ │ tracking │
       │         │→SHIPPING│ │ 기록     │
       │         └────────┘ └──────────┘
       │
       │   OrderCreatedEvent
       └───────────────────────────────────▶ ┌──────────┐
                                             │ tracking │
                                             │ 주문생성 │
                                             └──────────┘
```

### 3.5 이벤트 정의

```java
// shared/events/ — 모든 모듈이 공유하는 이벤트 계약 (Java Record 활용)

public record OrderCreatedEvent(
    UUID orderId,
    String customerName,
    BigDecimal totalAmount,
    int itemsCount,
    Instant timestamp
) {}

public record PaymentApprovedEvent(
    UUID paymentId,
    UUID orderId,
    BigDecimal originalAmount,
    BigDecimal discountAmount,
    BigDecimal finalAmount,
    String appliedDiscountType,   // "none", "basic_subscription", "premium_subscription"
    String method,
    Instant timestamp
) {}

public record PaymentRejectedEvent(
    UUID paymentId,
    UUID orderId,
    String reason,
    Instant timestamp
) {}

public record ShipmentCreatedEvent(
    UUID shipmentId,
    UUID orderId,
    BigDecimal shippingFee,
    String feeDiscountType,       // "none", "basic_half", "premium_free"
    String trackingNumber,
    Instant timestamp
) {}

public record ShipmentStatusChangedEvent(
    UUID shipmentId,
    UUID orderId,
    String newStatus,
    Instant timestamp
) {}

public record SubscriptionActivatedEvent(
    UUID subscriptionId,
    String customerName,
    String tier,
    Instant expiresAt,
    Instant timestamp
) {}

public record SubscriptionExpiredEvent(
    UUID subscriptionId,
    String customerName,
    String previousTier,
    Instant timestamp
) {}
```

---

## 4. 아키텍처 설계

### 4.1 각 모듈의 내부 구조 (Hexagonal)

```
orders/                            # 하나의 Bounded Context (Spring Modulith 모듈)
├── domain/                        # 🎯 핵심: 순수 Java, Spring 의존성 없음
│   ├── model/
│   │   ├── Order.java             # Aggregate Root
│   │   ├── OrderItem.java         # Entity
│   │   ├── Money.java             # Value Object (record)
│   │   ├── OrderId.java           # Value Object (record)
│   │   └── OrderStatus.java       # Enum + 상태 전이 규칙
│   ├── exception/
│   │   ├── OrderNotFoundException.java
│   │   └── InvalidStatusTransitionException.java
│   └── port/
│       └── out/
│           └── OrderRepository.java          # Output Port (Interface)
│
├── application/                   # 🔄 Use Cases: 도메인 조율
│   ├── port/
│   │   └── in/
│   │       ├── CreateOrderUseCase.java       # Input Port
│   │       ├── CancelOrderUseCase.java
│   │       └── GetOrderUseCase.java
│   ├── command/
│   │   ├── CreateOrderCommand.java           # record
│   │   └── CancelOrderCommand.java
│   ├── query/
│   │   ├── GetOrderQuery.java
│   │   └── ListOrdersQuery.java
│   ├── service/
│   │   ├── OrderCommandService.java          # Use Case 구현
│   │   └── OrderQueryService.java
│   └── eventhandler/
│       └── OrderEventHandler.java            # 외부 이벤트 수신
│
├── adapter/                       # 🔧 외부 세계 연결
│   ├── out/
│   │   └── persistence/
│   │       ├── OrderJpaEntity.java           # JPA Entity (@Entity)
│   │       ├── SpringDataOrderRepository.java # Spring Data JPA
│   │       ├── OrderPersistenceAdapter.java  # Port 구현체 (@Repository)
│   │       └── OrderMapper.java              # Domain ↔ JPA 변환
│   └── in/
│       └── web/
│           ├── OrderController.java          # @RestController
│           ├── CreateOrderRequest.java       # Request DTO (record)
│           └── OrderResponse.java            # Response DTO (record)
│
└── internal/                      # Spring Modulith 내부 패키지
    └── OrderModuleConfig.java     # 모듈 내부 설정
```

> **의존성 방향 검증**: `domain/` 패키지에서
> `grep -r "import org.springframework\|import jakarta.persistence\|import org.hibernate"` → 결과 0건이어야 함
>
> **Spring Modulith 검증**: `ApplicationModules.of(ShopTrackerApp.class).verify()` 로 모듈 경계 자동 검증

### 4.2 DI 설계 (Spring IoC) — 구독 기반 정책 주입

```java
// shared/config/SubscriptionContextConfig.java
@Configuration
public class SubscriptionContextConfig {

    /**
     * 요청 스코프 빈: 매 HTTP 요청마다 고객의 구독 상태를 조회하여 SubscriptionContext를 생성.
     * 다른 모듈은 이 SubscriptionContext만 주입받으면 된다.
     */
    @Bean
    @RequestScope
    public SubscriptionContext subscriptionContext(
            HttpServletRequest request,
            SubscriptionQueryPort subscriptionQueryPort) {

        String customerName = request.getHeader("X-Customer-Name");
        if (customerName == null || customerName.isBlank()) {
            return SubscriptionContext.none("guest");
        }
        return subscriptionQueryPort.findActiveByCustomer(customerName)
            .filter(Subscription::isActive)
            .map(sub -> new SubscriptionContext(
                customerName, sub.getTier().getValue(), true))
            .orElse(SubscriptionContext.none(customerName));
    }
}
```

```java
// payments/internal/PaymentsPolicyConfig.java
@Configuration
class PaymentsPolicyConfig {

    /**
     * ★ 핵심 학습 포인트:
     * Payments 모듈은 Subscription을 import하지 않는다.
     * Spring DI가 SubscriptionContext(Request Scope)를 보고 적절한 정책을 조립해준다.
     */
    @Bean
    @RequestScope
    DiscountPolicy discountPolicy(SubscriptionContext subCtx) {
        return switch (subCtx.tier()) {
            case "premium" -> new SubscriptionDiscountPolicy(
                new BigDecimal("0.10"), "premium_subscription");
            case "basic" -> new SubscriptionDiscountPolicy(
                new BigDecimal("0.05"), "basic_subscription");
            default -> new NoDiscountPolicy();
        };
    }

    @Bean
    PaymentValidationPolicy paymentValidationPolicy() {
        return new StandardPaymentValidationPolicy();
    }

    @Bean
    PaymentGateway paymentGateway() {
        return new FakePaymentGateway();
    }
}
```

```java
// shipping/internal/ShippingPolicyConfig.java
@Configuration
class ShippingPolicyConfig {

    /**
     * ★ 핵심 학습 포인트:
     * Shipping 모듈도 Subscription을 import하지 않는다.
     * Spring DI가 SubscriptionContext를 보고 적절한 배송비 정책을 조립해준다.
     */
    @Bean
    @RequestScope
    ShippingFeePolicy shippingFeePolicy(SubscriptionContext subCtx) {
        return switch (subCtx.tier()) {
            case "premium" -> new PremiumShippingFeePolicy();   // 항상 무료
            case "basic" -> new BasicShippingFeePolicy();       // 50% 할인, 3만원 이상 무료
            default -> new StandardShippingFeePolicy();         // 3,000원, 5만원 이상 무료
        };
    }
}
```

### 4.3 정책 객체 설계 — 결제 할인

```java
// payments/domain/port/out/DiscountPolicy.java
public interface DiscountPolicy {
    DiscountResult calculateDiscount(Money amount);
}

// payments/domain/model/DiscountResult.java
public record DiscountResult(
    Money discountAmount,
    String discountType    // "none", "basic_subscription", "premium_subscription"
) {}

// payments/domain/model/policy/SubscriptionDiscountPolicy.java
public class SubscriptionDiscountPolicy implements DiscountPolicy {
    private final BigDecimal rate;
    private final String discountType;

    public SubscriptionDiscountPolicy(BigDecimal rate, String discountType) {
        this.rate = rate;
        this.discountType = discountType;
    }

    @Override
    public DiscountResult calculateDiscount(Money amount) {
        return new DiscountResult(
            amount.applyRate(rate),
            discountType
        );
    }
}

// payments/domain/model/policy/NoDiscountPolicy.java
public class NoDiscountPolicy implements DiscountPolicy {
    @Override
    public DiscountResult calculateDiscount(Money amount) {
        return new DiscountResult(Money.ZERO, "none");
    }
}
```

### 4.4 정책 객체 설계 — 배송비

```java
// shipping/domain/port/out/ShippingFeePolicy.java
public interface ShippingFeePolicy {
    ShippingFeeResult calculateFee(Money orderAmount);
}

// shipping/domain/model/ShippingFeeResult.java
public record ShippingFeeResult(
    Money fee,
    Money originalFee,
    String discountType,    // "none", "basic_half", "premium_free"
    String reason
) {}

// shipping/domain/model/policy/StandardShippingFeePolicy.java
public class StandardShippingFeePolicy implements ShippingFeePolicy {
    private static final Money BASE_FEE = new Money(new BigDecimal("3000"));
    private static final Money FREE_THRESHOLD = new Money(new BigDecimal("50000"));

    @Override
    public ShippingFeeResult calculateFee(Money orderAmount) {
        if (orderAmount.isGreaterThanOrEqual(FREE_THRESHOLD)) {
            return new ShippingFeeResult(Money.ZERO, BASE_FEE, "none", "50,000원 이상 무료배송");
        }
        return new ShippingFeeResult(BASE_FEE, BASE_FEE, "none", "기본 배송비");
    }
}

// shipping/domain/model/policy/BasicShippingFeePolicy.java
public class BasicShippingFeePolicy implements ShippingFeePolicy {
    private static final Money BASE_FEE = new Money(new BigDecimal("3000"));
    private static final Money HALF_FEE = new Money(new BigDecimal("1500"));
    private static final Money FREE_THRESHOLD = new Money(new BigDecimal("30000"));

    @Override
    public ShippingFeeResult calculateFee(Money orderAmount) {
        if (orderAmount.isGreaterThanOrEqual(FREE_THRESHOLD)) {
            return new ShippingFeeResult(Money.ZERO, BASE_FEE, "basic_half",
                "Basic 구독 30,000원 이상 무료배송");
        }
        return new ShippingFeeResult(HALF_FEE, BASE_FEE, "basic_half",
            "Basic 구독 배송비 50% 할인");
    }
}

// shipping/domain/model/policy/PremiumShippingFeePolicy.java
public class PremiumShippingFeePolicy implements ShippingFeePolicy {
    private static final Money BASE_FEE = new Money(new BigDecimal("3000"));

    @Override
    public ShippingFeeResult calculateFee(Money orderAmount) {
        return new ShippingFeeResult(Money.ZERO, BASE_FEE, "premium_free",
            "Premium 구독 무료배송");
    }
}
```

### 4.5 Fake Payment Gateway

```java
// payments/adapter/out/FakePaymentGateway.java
@Component
public class FakePaymentGateway implements PaymentGateway {
    private final Random random = new Random();

    @Override
    public GatewayResult process(Payment payment) {
        try {
            Thread.sleep(300); // 네트워크 지연 시뮬레이션 (Virtual Thread에서 효율적)
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        if (random.nextDouble() < 0.9) {
            return new GatewayResult(true, UUID.randomUUID().toString(), "Payment approved");
        }
        return new GatewayResult(false, null, "Insufficient funds");
    }
}
```

### 4.6 이벤트 버스 (Spring Modulith)

FastAPI의 InMemoryEventBus와 달리, Spring은 프레임워크 수준에서 이벤트 퍼블리싱을 지원합니다.

```java
// orders/application/service/OrderCommandService.java
@Service
@Transactional
public class OrderCommandService implements CreateOrderUseCase {
    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;  // Spring 내장

    public OrderCommandService(OrderRepository orderRepository,
                                ApplicationEventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.eventPublisher = eventPublisher;
    }

    @Override
    public UUID createOrder(CreateOrderCommand command) {
        Order order = Order.create(command.customerName(), command.items());
        orderRepository.save(order);

        // Spring ApplicationEvent 발행
        eventPublisher.publishEvent(new OrderCreatedEvent(
            order.getId().value(),
            command.customerName(),
            order.getTotalAmount().amount(),
            command.items().size(),
            Instant.now()
        ));

        return order.getId().value();
    }
}

// payments/application/eventhandler/PaymentEventHandler.java
@Component
public class PaymentEventHandler {
    private final ProcessPaymentUseCase processPaymentUseCase;

    /**
     * Spring Modulith의 @ApplicationModuleListener:
     * - 비동기 이벤트 수신
     * - 이벤트 지속성 (실패 시 자동 재시도)
     * - 트랜잭션 분리
     */
    @ApplicationModuleListener
    public void on(OrderCreatedEvent event) {
        processPaymentUseCase.processPayment(
            new ProcessPaymentCommand(event.orderId(), event.totalAmount())
        );
    }
}

// tracking/application/eventhandler/TrackingEventHandler.java
@Component
public class TrackingEventHandler {
    private final TrackingRepository trackingRepository;

    @ApplicationModuleListener
    public void onOrderCreated(OrderCreatedEvent event) {
        // 추적 기록 생성
    }

    @ApplicationModuleListener
    public void onPaymentApproved(PaymentApprovedEvent event) {
        // 결제 성공 기록
    }

    @ApplicationModuleListener
    public void onPaymentRejected(PaymentRejectedEvent event) {
        // 결제 실패 기록
    }

    @ApplicationModuleListener
    public void onShipmentCreated(ShipmentCreatedEvent event) {
        // 배송 시작 기록
    }
}
```

### 4.7 CQRS 적용 (간소화)

```java
// Command 측: OrderCommandService (위에서 정의)
// Query 측: OrderQueryService
@Service
@Transactional(readOnly = true)
public class OrderQueryService implements GetOrderUseCase {
    private final OrderRepository orderRepository;  // EventPublisher 의존 없음!

    public OrderQueryService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Override
    public OrderSummary getOrder(UUID orderId) {
        return orderRepository.findById(new OrderId(orderId))
            .map(OrderMapper::toSummary)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }

    public Page<OrderSummary> listOrders(ListOrdersQuery query) {
        return orderRepository.findAll(query.toPageable())
            .map(OrderMapper::toSummary);
    }
}
```

---

## 5. 프로젝트 디렉토리 구조

```
shoptracker/
├── docker-compose.yml
├── Dockerfile
├── build.gradle.kts
├── settings.gradle.kts
│
├── src/
│   ├── main/
│   │   ├── java/com/shoptracker/
│   │   │   ├── ShopTrackerApplication.java
│   │   │   │
│   │   │   ├── shared/                      # 🔗 공유 계약 (이벤트, DTO)
│   │   │   │   ├── SubscriptionContext.java
│   │   │   │   ├── config/
│   │   │   │   │   ├── SubscriptionContextConfig.java
│   │   │   │   │   └── JpaConfig.java
│   │   │   │   ├── events/
│   │   │   │   │   ├── OrderCreatedEvent.java
│   │   │   │   │   ├── PaymentApprovedEvent.java
│   │   │   │   │   ├── PaymentRejectedEvent.java
│   │   │   │   │   ├── ShipmentCreatedEvent.java
│   │   │   │   │   ├── ShipmentStatusChangedEvent.java
│   │   │   │   │   ├── SubscriptionActivatedEvent.java
│   │   │   │   │   └── SubscriptionExpiredEvent.java
│   │   │   │   └── exception/
│   │   │   │       └── GlobalExceptionHandler.java
│   │   │   │
│   │   │   ├── orders/                       # 📦 주문 모듈
│   │   │   │   ├── domain/
│   │   │   │   │   ├── model/
│   │   │   │   │   │   ├── Order.java
│   │   │   │   │   │   ├── OrderItem.java
│   │   │   │   │   │   ├── OrderId.java
│   │   │   │   │   │   ├── OrderStatus.java
│   │   │   │   │   │   └── Money.java
│   │   │   │   │   ├── exception/
│   │   │   │   │   │   ├── OrderNotFoundException.java
│   │   │   │   │   │   └── InvalidStatusTransitionException.java
│   │   │   │   │   └── port/
│   │   │   │   │       └── out/
│   │   │   │   │           └── OrderRepository.java
│   │   │   │   ├── application/
│   │   │   │   │   ├── port/in/
│   │   │   │   │   │   ├── CreateOrderUseCase.java
│   │   │   │   │   │   └── GetOrderUseCase.java
│   │   │   │   │   ├── command/
│   │   │   │   │   │   └── CreateOrderCommand.java
│   │   │   │   │   ├── query/
│   │   │   │   │   │   └── ListOrdersQuery.java
│   │   │   │   │   ├── service/
│   │   │   │   │   │   ├── OrderCommandService.java
│   │   │   │   │   │   └── OrderQueryService.java
│   │   │   │   │   └── eventhandler/
│   │   │   │   │       └── OrderEventHandler.java
│   │   │   │   ├── adapter/
│   │   │   │   │   ├── out/persistence/
│   │   │   │   │   │   ├── OrderJpaEntity.java
│   │   │   │   │   │   ├── SpringDataOrderRepository.java
│   │   │   │   │   │   ├── OrderPersistenceAdapter.java
│   │   │   │   │   │   └── OrderMapper.java
│   │   │   │   │   └── in/web/
│   │   │   │   │       ├── OrderController.java
│   │   │   │   │       ├── CreateOrderRequest.java
│   │   │   │   │       └── OrderResponse.java
│   │   │   │   └── internal/
│   │   │   │       └── OrderModuleConfig.java
│   │   │   │
│   │   │   ├── payments/                     # 💳 결제 모듈
│   │   │   │   ├── domain/
│   │   │   │   │   ├── model/
│   │   │   │   │   │   ├── Payment.java
│   │   │   │   │   │   ├── PaymentId.java
│   │   │   │   │   │   ├── PaymentMethod.java
│   │   │   │   │   │   ├── PaymentStatus.java
│   │   │   │   │   │   ├── DiscountResult.java
│   │   │   │   │   │   ├── GatewayResult.java
│   │   │   │   │   │   └── policy/
│   │   │   │   │   │       ├── SubscriptionDiscountPolicy.java
│   │   │   │   │   │       └── NoDiscountPolicy.java
│   │   │   │   │   ├── exception/
│   │   │   │   │   │   └── PaymentProcessingException.java
│   │   │   │   │   └── port/out/
│   │   │   │   │       ├── DiscountPolicy.java
│   │   │   │   │       ├── PaymentGateway.java
│   │   │   │   │       ├── PaymentValidationPolicy.java
│   │   │   │   │       └── PaymentRepository.java
│   │   │   │   ├── application/
│   │   │   │   │   ├── port/in/
│   │   │   │   │   │   └── ProcessPaymentUseCase.java
│   │   │   │   │   ├── command/
│   │   │   │   │   │   └── ProcessPaymentCommand.java
│   │   │   │   │   ├── service/
│   │   │   │   │   │   └── PaymentCommandService.java
│   │   │   │   │   └── eventhandler/
│   │   │   │   │       └── PaymentEventHandler.java
│   │   │   │   ├── adapter/
│   │   │   │   │   ├── out/
│   │   │   │   │   │   ├── persistence/
│   │   │   │   │   │   │   ├── PaymentJpaEntity.java
│   │   │   │   │   │   │   ├── SpringDataPaymentRepository.java
│   │   │   │   │   │   │   ├── PaymentPersistenceAdapter.java
│   │   │   │   │   │   │   └── PaymentMapper.java
│   │   │   │   │   │   └── FakePaymentGateway.java
│   │   │   │   │   └── in/web/
│   │   │   │   │       ├── PaymentController.java
│   │   │   │   │       └── PaymentResponse.java
│   │   │   │   └── internal/
│   │   │   │       └── PaymentsPolicyConfig.java
│   │   │   │
│   │   │   ├── shipping/                     # 🚚 배송 모듈
│   │   │   │   ├── domain/
│   │   │   │   │   ├── model/
│   │   │   │   │   │   ├── Shipment.java
│   │   │   │   │   │   ├── ShipmentId.java
│   │   │   │   │   │   ├── Address.java
│   │   │   │   │   │   ├── ShipmentStatus.java
│   │   │   │   │   │   ├── ShippingFeeResult.java
│   │   │   │   │   │   └── policy/
│   │   │   │   │   │       ├── StandardShippingFeePolicy.java
│   │   │   │   │   │       ├── BasicShippingFeePolicy.java
│   │   │   │   │   │       └── PremiumShippingFeePolicy.java
│   │   │   │   │   ├── exception/
│   │   │   │   │   │   └── ShipmentNotFoundException.java
│   │   │   │   │   └── port/out/
│   │   │   │   │       ├── ShippingFeePolicy.java
│   │   │   │   │       └── ShipmentRepository.java
│   │   │   │   ├── application/
│   │   │   │   │   ├── port/in/
│   │   │   │   │   │   └── CreateShipmentUseCase.java
│   │   │   │   │   ├── service/
│   │   │   │   │   │   └── ShipmentCommandService.java
│   │   │   │   │   └── eventhandler/
│   │   │   │   │       └── ShippingEventHandler.java
│   │   │   │   ├── adapter/
│   │   │   │   │   ├── out/persistence/
│   │   │   │   │   │   ├── ShipmentJpaEntity.java
│   │   │   │   │   │   ├── SpringDataShipmentRepository.java
│   │   │   │   │   │   ├── ShipmentPersistenceAdapter.java
│   │   │   │   │   │   └── ShipmentMapper.java
│   │   │   │   │   └── in/web/
│   │   │   │   │       ├── ShippingController.java
│   │   │   │   │       └── ShipmentResponse.java
│   │   │   │   └── internal/
│   │   │   │       └── ShippingPolicyConfig.java
│   │   │   │
│   │   │   ├── subscription/                 # 🎫 구독 모듈
│   │   │   │   ├── domain/
│   │   │   │   │   ├── model/
│   │   │   │   │   │   ├── Subscription.java
│   │   │   │   │   │   ├── SubscriptionId.java
│   │   │   │   │   │   ├── SubscriptionTier.java
│   │   │   │   │   │   └── SubscriptionStatus.java
│   │   │   │   │   ├── exception/
│   │   │   │   │   │   └── DuplicateSubscriptionException.java
│   │   │   │   │   └── port/out/
│   │   │   │   │       └── SubscriptionRepository.java
│   │   │   │   ├── application/
│   │   │   │   │   ├── port/in/
│   │   │   │   │   │   ├── CreateSubscriptionUseCase.java
│   │   │   │   │   │   ├── CancelSubscriptionUseCase.java
│   │   │   │   │   │   └── SubscriptionQueryPort.java
│   │   │   │   │   ├── command/
│   │   │   │   │   │   └── CreateSubscriptionCommand.java
│   │   │   │   │   └── service/
│   │   │   │   │       ├── SubscriptionCommandService.java
│   │   │   │   │       └── SubscriptionQueryService.java
│   │   │   │   ├── adapter/
│   │   │   │   │   ├── out/persistence/
│   │   │   │   │   │   ├── SubscriptionJpaEntity.java
│   │   │   │   │   │   ├── SpringDataSubscriptionRepository.java
│   │   │   │   │   │   ├── SubscriptionPersistenceAdapter.java
│   │   │   │   │   │   └── SubscriptionMapper.java
│   │   │   │   │   └── in/web/
│   │   │   │   │       ├── SubscriptionController.java
│   │   │   │   │       ├── CreateSubscriptionRequest.java
│   │   │   │   │       └── SubscriptionResponse.java
│   │   │   │   └── internal/
│   │   │   │       └── SubscriptionModuleConfig.java
│   │   │   │
│   │   │   └── tracking/                     # 📊 추적 모듈 (Saga)
│   │   │       ├── domain/
│   │   │       │   ├── model/
│   │   │       │   │   ├── OrderTracking.java
│   │   │       │   │   ├── TrackingId.java
│   │   │       │   │   ├── TrackingEvent.java
│   │   │       │   │   └── TrackingPhase.java
│   │   │       │   ├── exception/
│   │   │       │   └── port/out/
│   │   │       │       └── TrackingRepository.java
│   │   │       ├── application/
│   │   │       │   ├── port/in/
│   │   │       │   │   └── GetTrackingUseCase.java
│   │   │       │   ├── service/
│   │   │       │   │   └── TrackingQueryService.java
│   │   │       │   └── eventhandler/
│   │   │       │       └── TrackingEventHandler.java
│   │   │       ├── adapter/
│   │   │       │   ├── out/persistence/
│   │   │       │   │   ├── TrackingJpaEntity.java
│   │   │       │   │   ├── SpringDataTrackingRepository.java
│   │   │       │   │   ├── TrackingPersistenceAdapter.java
│   │   │       │   │   └── TrackingMapper.java
│   │   │       │   └── in/web/
│   │   │       │       ├── TrackingController.java
│   │   │       │       └── TrackingResponse.java
│   │   │       └── internal/
│   │   │           └── TrackingModuleConfig.java
│   │   │
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       ├── application-prod.yml
│   │       └── db/migration/
│   │           ├── V1__create_subscriptions.sql
│   │           ├── V2__create_orders.sql
│   │           ├── V3__create_payments.sql
│   │           ├── V4__create_shipments.sql
│   │           └── V5__create_tracking.sql
│   │
│   └── test/
│       └── java/com/shoptracker/
│           ├── ModuleStructureTest.java        # Spring Modulith 모듈 경계 검증
│           ├── unit/                            # 🧪 DB 없음, 순수 도메인 테스트
│           │   ├── OrderTest.java
│           │   ├── MoneyTest.java
│           │   ├── OrderStatusTransitionTest.java
│           │   ├── DiscountPolicyTest.java
│           │   ├── ShippingFeePolicyTest.java
│           │   └── SubscriptionTest.java
│           ├── integration/                     # 🔗 @SpringBootTest + Testcontainers
│           │   ├── OrderFlowIntegrationTest.java
│           │   ├── PaymentWithSubscriptionTest.java
│           │   ├── ShippingWithSubscriptionTest.java
│           │   └── FullSagaIntegrationTest.java
│           └── module/                          # 📦 Spring Modulith 모듈 테스트
│               ├── OrdersModuleTest.java
│               ├── PaymentsModuleTest.java
│               └── ShippingModuleTest.java
```

---

## 6. API 엔드포인트 설계

### Subscriptions (`/api/v1/subscriptions`)

| Method | Path | 설명 | 유형 |
|--------|------|------|------|
| `POST` | `/api/v1/subscriptions` | 구독 생성 (tier, customerName) | Command |
| `GET` | `/api/v1/subscriptions/{id}` | 구독 상세 | Query |
| `GET` | `/api/v1/subscriptions/customer/{customerName}` | 고객의 활성 구독 조회 | Query |
| `POST` | `/api/v1/subscriptions/{id}/cancel` | 구독 취소 | Command |

### Orders (`/api/v1/orders`)

| Method | Path | 설명 | 유형 |
|--------|------|------|------|
| `POST` | `/api/v1/orders` | 주문 생성 | Command |
| `GET` | `/api/v1/orders` | 주문 목록 (필터, 페이지네이션) | Query |
| `GET` | `/api/v1/orders/{id}` | 주문 상세 (할인/배송비 내역 포함) | Query |
| `POST` | `/api/v1/orders/{id}/cancel` | 주문 취소 | Command |

### Payments (`/api/v1/payments`)

| Method | Path | 설명 | 유형 |
|--------|------|------|------|
| `GET` | `/api/v1/payments/{id}` | 결제 상세 (할인 내역 포함) | Query |
| `GET` | `/api/v1/payments/order/{orderId}` | 주문별 결제 조회 | Query |

### Shipping (`/api/v1/shipping`)

| Method | Path | 설명 | 유형 |
|--------|------|------|------|
| `GET` | `/api/v1/shipping/{id}` | 배송 상세 (배송비 내역 포함) | Query |
| `GET` | `/api/v1/shipping/order/{orderId}` | 주문별 배송 조회 | Query |
| `POST` | `/api/v1/shipping/{id}/update-status` | 배송 상태 변경 (시뮬레이션) | Command |

### Tracking (`/api/v1/tracking`)

| Method | Path | 설명 | 유형 |
|--------|------|------|------|
| `GET` | `/api/v1/tracking/order/{orderId}` | 주문 전체 여정 | Query |
| `GET` | `/api/v1/tracking/order/{orderId}/timeline` | 타임라인 형태 | Query |

---

## 7. 테스트 전략

### 7.1 Spring Modulith 모듈 경계 검증

```java
// ModuleStructureTest.java
class ModuleStructureTest {
    @Test
    void verifyModuleStructure() {
        ApplicationModules modules = ApplicationModules.of(ShopTrackerApplication.class);
        modules.verify();  // 모듈 경계 위반 시 즉시 실패!
    }

    @Test
    void createModuleDocumentation() {
        ApplicationModules modules = ApplicationModules.of(ShopTrackerApplication.class);
        new Documenter(modules)
            .writeModulesAsPlantUml()
            .writeIndividualModulesAsPlantUml();
    }
}
```

### 7.2 단위 테스트 — DB 없음, 순수 도메인

```java
// unit/DiscountPolicyTest.java
class DiscountPolicyTest {

    @Test
    void premiumSubscription_10PercentDiscount() {
        DiscountPolicy policy = new SubscriptionDiscountPolicy(
            new BigDecimal("0.10"), "premium_subscription");

        DiscountResult result = policy.calculateDiscount(new Money(new BigDecimal("100000")));

        assertThat(result.discountAmount()).isEqualTo(new Money(new BigDecimal("10000")));
        assertThat(result.discountType()).isEqualTo("premium_subscription");
    }

    @Test
    void basicSubscription_5PercentDiscount() {
        DiscountPolicy policy = new SubscriptionDiscountPolicy(
            new BigDecimal("0.05"), "basic_subscription");

        DiscountResult result = policy.calculateDiscount(new Money(new BigDecimal("100000")));

        assertThat(result.discountAmount()).isEqualTo(new Money(new BigDecimal("5000")));
    }

    @Test
    void noSubscription_noDiscount() {
        DiscountPolicy policy = new NoDiscountPolicy();

        DiscountResult result = policy.calculateDiscount(new Money(new BigDecimal("100000")));

        assertThat(result.discountAmount()).isEqualTo(Money.ZERO);
        assertThat(result.discountType()).isEqualTo("none");
    }
}
```

### 7.3 통합 테스트 — Testcontainers + @SpringBootTest

```java
// integration/PaymentWithSubscriptionTest.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class PaymentWithSubscriptionTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @Autowired TestRestTemplate restTemplate;

    @Test
    void premiumSubscriber_gets10PercentDiscount() {
        // 1. Premium 구독 생성
        restTemplate.postForEntity("/api/v1/subscriptions",
            new CreateSubscriptionRequest("홍길동", "premium"), Void.class);

        // 2. 주문 생성
        HttpHeaders headers = new HttpHeaders();
        headers.set("X-Customer-Name", "홍길동");
        var orderResponse = restTemplate.exchange(
            "/api/v1/orders", HttpMethod.POST,
            new HttpEntity<>(new CreateOrderRequest("홍길동",
                List.of(new OrderItemRequest("노트북", 1, 1000000))), headers),
            OrderResponse.class
        );
        UUID orderId = orderResponse.getBody().id();

        // 이벤트 처리 대기
        await().atMost(Duration.ofSeconds(5)).untilAsserted(() -> {
            var payment = restTemplate.getForObject(
                "/api/v1/payments/order/{orderId}", PaymentResponse.class, orderId);
            assertThat(payment).isNotNull();
            assertThat(payment.appliedDiscountType()).isEqualTo("premium_subscription");
            assertThat(payment.discountAmount()).isEqualTo(100000);  // 10% of 1,000,000
        });
    }
}
```

### 7.4 Spring Modulith 모듈 통합 테스트

```java
// module/PaymentsModuleTest.java
@ApplicationModuleTest
class PaymentsModuleTest {

    @Autowired
    private Scenario scenario;

    @Test
    void orderCreated_triggersPayment_publishesResult() {
        scenario.publish(new OrderCreatedEvent(
                UUID.randomUUID(), "테스트고객",
                new BigDecimal("50000"), 1, Instant.now()))
            .andWaitForEventOfType(PaymentApprovedEvent.class)
            .matching(e -> e.orderId() != null)
            .toArriveAndVerify(event -> {
                assertThat(event.finalAmount()).isNotNull();
            });
    }
}
```

---

## 8. 구현 단계 (Phase)

### Phase 1: 뼈대 + Subscription + Orders 모듈

> 목표: Hexagonal 구조, Spring DI, 기본 CRUD가 돌아가는 것 확인

- [ ] 프로젝트 셋업 (build.gradle.kts, Docker Compose, PostgreSQL)
- [ ] Spring Modulith + Spring Data JPA + Flyway 의존성
- [ ] `shared/` 패키지 (config, events, SubscriptionContext)
- [ ] Subscription 모듈 전체 (도메인 → 어댑터 → 컨트롤러)
- [ ] Orders 도메인 레이어 (Entity, Value Object, Port)
- [ ] Orders 어댑터 레이어 (JPA Entity, Repository, Mapper)
- [ ] Orders 컨트롤러 (REST API)
- [ ] Flyway 마이그레이션 (V1, V2)
- [ ] 단위 테스트: Order, Money, 상태 전이, Subscription
- [ ] Spring Modulith 모듈 경계 검증 테스트

**체감 체크포인트:**
- `domain/` 패키지에서 `grep -r "import org.springframework"` → 0건
- 단위 테스트가 DB 없이 0.1초 안에 끝남
- `ApplicationModules.of(...).verify()` 통과

### Phase 2: Spring Events + Payments + 구독 할인 정책

> 목표: 모듈 간 이벤트 통신 + DI 정책 주입 체감

- [ ] Spring ApplicationEventPublisher로 이벤트 발행
- [ ] Orders → OrderCreatedEvent 발행
- [ ] Payments 도메인 (Entity, DiscountPolicy 인터페이스)
- [ ] 할인 정책 구현체 (SubscriptionDiscountPolicy, NoDiscountPolicy)
- [ ] FakePaymentGateway
- [ ] PaymentsPolicyConfig: SubscriptionContext → DiscountPolicy Bean 조립
- [ ] @ApplicationModuleListener로 이벤트 수신
- [ ] 단위 테스트: 할인 정책 (3가지 등급)
- [ ] 통합 테스트: Premium 구독자 주문 → 10% 할인 확인
- [ ] Spring Modulith Scenario 테스트

**체감 체크포인트:**
- `orders/` 에서 `grep -r "import.*payments"` → 0건
- `payments/` 에서 `grep -r "import.*subscription"` → 0건
- DI 설정에서 tier만 바꾸면 할인율이 달라지는 것 확인

### Phase 3: Shipping + 구독 배송비 정책

> 목표: 두 번째 정책 주입 + 구독이 여러 도메인에 영향을 주되 느슨한 것 체감

- [ ] Shipping 모듈 (도메인 + 어댑터 + 컨트롤러)
- [ ] 배송비 정책 구현체 (Standard, Basic, Premium)
- [ ] ShippingPolicyConfig: SubscriptionContext → ShippingFeePolicy Bean 조립
- [ ] PaymentApprovedEvent → 배송 자동 생성
- [ ] 단위 테스트: 배송비 정책 (3가지 등급 × 금액 조건)
- [ ] 통합 테스트: 구독 등급별 배송비 확인

**체감 체크포인트:**
- `shipping/` 에서 `grep -r "import.*subscription"` → 0건
- Premium 1,000원 주문 → 배송비 0원
- Basic 40,000원 주문 → 배송비 0원 (3만원 이상), 20,000원 → 1,500원

### Phase 4: Tracking 모듈 + 전체 Saga

> 목표: 모든 이벤트가 연결되고, 실패 보상 흐름까지 동작

- [ ] Tracking 모듈 (모든 이벤트 구독, 기록)
- [ ] Tracking 조회 API (타임라인)
- [ ] 보상 로직: PaymentRejectedEvent → Order CANCELLED
- [ ] SubscriptionActivated/Expired 이벤트도 Tracking에 기록
- [ ] 전체 Saga 통합 테스트 (성공 + 실패)
- [ ] Spring Modulith 모듈 문서 자동 생성

**체감 체크포인트:**
- 주문 1개 생성 → Tracking에 모든 이벤트 타임라인 확인
- 결제 실패 → Order CANCELLED 자동 전환
- Modulith 자동 생성 문서에서 모듈 간 이벤트 흐름 시각화

### Phase 5: CQRS 고도화 + Observability

> 목표: Command/Query 분리 심화 + OpenTelemetry 관찰성

- [ ] Command/Query 서비스 정리
- [ ] 주문 목록 Query에 ReadModel 적용
- [ ] 페이지네이션, 필터링 (Spring Data Pageable)
- [ ] spring-boot-starter-opentelemetry 설정
- [ ] @Observed 어노테이션으로 핵심 메서드 계측
- [ ] GlobalExceptionHandler 표준화
- [ ] Swagger/OpenAPI 문서 설정

### Phase 6: 프로덕션 배포

> 목표: Docker로 프로덕션 환경 구성

- [ ] Dockerfile (multi-stage build)
- [ ] docker-compose.yml (PostgreSQL + App)
- [ ] Virtual Threads 활성화 (`spring.threads.virtual.enabled=true`)
- [ ] 환경별 설정 (application-dev.yml / application-prod.yml)
- [ ] Health check 엔드포인트 (Actuator)
- [ ] (선택) GraalVM 네이티브 이미지 빌드

---

## 9. 핵심 설계 결정 요약

| 결정 | 선택 | 이유 |
|------|------|------|
| 모놀리스 vs MSA | **모듈러 모놀리스 (Spring Modulith)** | 모듈 경계 검증, 이벤트 지속성 내장 |
| DB 공유 vs 분리 | **공유 (모듈별 테이블)** | 모놀리스이므로 1개 DB |
| 이벤트 | **Spring ApplicationEvent + @ApplicationModuleListener** | Spring Modulith 기본 |
| DI | **Spring IoC (내장)** | 프레임워크 핵심 기능 |
| 구독 → 다른 모듈 영향 | **SubscriptionContext (Request Scope) + @Bean 조건부 정책 주입** | 모듈 간 직접 의존 없음 |
| CQRS 수준 | **서비스 분리 (CommandService / QueryService)** | 별도 DB까지는 오버 |
| 결제 | **Fake Gateway** | 정책 패턴에 집중 |
| 인증 | **X-Customer-Name 헤더** | 아키텍처 학습에 집중 |
| 동기 vs 비동기 | **동기 MVC + Virtual Threads** | Spring Boot 4 기본 |
| 테스트 | **JUnit 5 + Testcontainers + Spring Modulith Scenario** | 표준 |

---

## 10. FastAPI vs Spring 설계 차이점 요약

| 관점 | FastAPI (ShopTracker) | Spring (ShopTracker) |
|------|----------------------|---------------------|
| **이벤트 버스** | 직접 구현 (InMemoryEventBus) | Spring 내장 (ApplicationEventPublisher) |
| **이벤트 수신** | 직접 subscribe 등록 | @ApplicationModuleListener (자동) |
| **이벤트 지속성** | 직접 구현 필요 | Spring Modulith 내장 (DB 자동 저장) |
| **모듈 경계 검증** | `grep` 수동 확인 | ApplicationModules.verify() 자동 |
| **DI 컨테이너** | Dishka (외부 라이브러리) | Spring IoC (프레임워크 핵심) |
| **Request Scope** | Dishka Scope.REQUEST | @RequestScope Bean |
| **정책 주입** | Dishka Provider match 문 | @Configuration @Bean 메서드 |
| **Port 정의** | Protocol (typing) | Java Interface |
| **불변 DTO** | dataclass(frozen=True) | Java Record |
| **DB 접근** | SQLAlchemy 2.0 async | Spring Data JPA |
| **마이그레이션** | Alembic | Flyway |
| **로깅** | structlog JSON | Logback + OpenTelemetry |
| **테스트** | pytest + httpx | JUnit 5 + TestRestTemplate + Testcontainers |
| **모듈 테스트** | 없음 (수동) | @ApplicationModuleTest + Scenario |
| **프로덕션 서버** | Gunicorn + Uvicorn | 내장 Tomcat + Virtual Threads |

---

## 11. 참고: 나중에 확장 가능한 방향

- Spring ApplicationEvent → **Spring Cloud Stream (Kafka/RabbitMQ)** (도메인 코드 변경 0)
- `FakePaymentGateway` → **실제 PG 연동** (인터페이스 교체)
- `SubscriptionContext` → 외부 구독 관리 서비스 연동
- 쿠폰 할인 정책 추가 (DiscountPolicy 구현체 추가 + @Bean 등록만)
- Spring Security + JWT 인증
- 모듈 → 별도 서비스 분리 (Spring Modulith → 실제 MSA 전환)
- GraalVM 네이티브 이미지 (시작 시간 ~50ms)

---