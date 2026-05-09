# Spring IoC / DI

> "Inversion of Control"과 "Dependency Injection". Spring의 가장 근본 메커니즘.

## 개념

**IoC**: 객체가 자기 의존성을 직접 만드는 게 아니라 **외부(컨테이너)** 가 만들어 넣어준다. 제어 흐름이 뒤집힌다.

**DI**: IoC를 구현하는 구체적 방법. 생성자/세터/필드 중 하나로 의존성을 주입.

```java
// Bad: 직접 생성 — 결합도 ↑, 테스트 ↓
public class OrderService {
    private final OrderRepository repo = new JpaOrderRepository();
}

// Good: 주입 — 인터페이스에만 의존
public class OrderService {
    private final OrderRepository repo;
    public OrderService(OrderRepository repo) { this.repo = repo; }
}
```

## Spring 컨테이너의 동작

1. **부팅**: `@SpringBootApplication`의 컴포넌트 스캔으로 `@Component`/`@Service`/`@Repository`/`@Controller`/`@Configuration` 클래스를 모두 찾음.
2. **빈 정의 등록**: 각 클래스를 빈 정의(BeanDefinition)로 등록. `@Bean` 메서드도 마찬가지.
3. **의존성 그래프 분석**: 생성자 파라미터 타입을 보고 어떤 빈이 필요한지 결정.
4. **인스턴스화**: 위상 정렬해서 의존성이 없는 것부터 생성. 순환 의존성이 있으면 실패.
5. **주입**: 생성자에 빈을 넘기며 인스턴스 생성.

## 빈 스코프

| 스코프 | 의미 |
|--------|------|
| `singleton`(기본) | 컨테이너당 하나 |
| `prototype` | 매 요청마다 새로 |
| `request` (`@RequestScope`) | HTTP 요청당 하나 |
| `session` | HTTP 세션당 하나 |

이 프로젝트의 핵심:
```java
// shared/config/SubscriptionContextConfig.java
@Bean
@RequestScope
public SubscriptionContext subscriptionContext(...) { ... }
```
요청 스코프 빈 → 매 요청마다 X-Customer-Name 헤더로 구독 조회 → `DiscountPolicy`/`ShippingFeePolicy` 가 이 컨텍스트를 보고 동적으로 조립됨.

## DI 주입 방식

### 1. 생성자 주입 (이 프로젝트의 기본)
```java
@Service
public class OrderCommandService implements CreateOrderUseCase {
    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;

    public OrderCommandService(OrderRepository orderRepository,
                               ApplicationEventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.eventPublisher = eventPublisher;
    }
}
```
- 불변 필드 가능(`final`).
- 단위 테스트에서 그냥 `new`로 만들 수 있음.
- **권장**.

### 2. 필드 주입 (`@Autowired private ...`)
편하지만 `final` 불가, 테스트 어려움. **권장 안 함**.

### 3. 세터 주입
선택적 의존성에만 가끔.

## `@Configuration` + `@Bean` 패턴

스프링이 컴포넌트 스캔만으로 못 만드는 빈(외부 라이브러리 객체, 조건부 객체)을 명시적으로 등록.

```java
// payments/internal/PaymentsPolicyConfig.java
@Configuration
class PaymentsPolicyConfig {

    @Bean
    @RequestScope
    DiscountPolicy discountPolicy(SubscriptionContext subCtx) {
        return switch (subCtx.tier()) {
            case "premium" -> new SubscriptionDiscountPolicy(new BigDecimal("0.10"), "premium_subscription");
            case "basic"   -> new SubscriptionDiscountPolicy(new BigDecimal("0.05"), "basic_subscription");
            default        -> new NoDiscountPolicy();
        };
    }
}
```
이 프로젝트의 핵심 패턴: **DI가 정책을 동적으로 조립**. Payments/Shipping 모듈은 Subscription을 import하지 않고도 구독 등급별 정책을 받음.

## 자격 한정자 (Qualifier)

같은 타입의 빈이 여러 개일 때 어느 걸 주입할지 지정.
```java
@Bean DiscountPolicy noDiscount() { ... }
@Bean DiscountPolicy premiumDiscount() { ... }

@Service
public class X {
    public X(@Qualifier("premiumDiscount") DiscountPolicy policy) { ... }
}
```
이 프로젝트는 `@RequestScope`로 한 시점에 하나만 활성이라 Qualifier 불필요.

## FastAPI/Dishka 대응

| Dishka (FastAPI) | Spring |
|------------------|--------|
| `Provider` 클래스 | `@Configuration` |
| `@provide` 데코레이터 | `@Bean` 메서드 |
| `Scope.REQUEST` | `@RequestScope` |
| `Scope.APP` | `singleton` (기본) |
| `Protocol` 타입 | `interface` 타입 |
| `from_context()` | `@RequestScope` Bean을 주입 |
| `match` 분기로 구현체 선택 | `switch` 표현식으로 동일 |
