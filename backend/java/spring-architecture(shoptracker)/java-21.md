# Java 21 (LTS)

> 이 프로젝트의 toolchain은 Java 21 (`build.gradle:11`).

Java 21은 2023년 9월 LTS. 이전 LTS인 Java 17 이후 큰 언어 변화가 누적되어 모던 Java의 새 출발점으로 통한다.

## 이 프로젝트에서 활용된 핵심 기능

### 1. Record (Java 14 정식, 16에서 안정)
```java
// shared/SubscriptionContext.java
public record SubscriptionContext(String customerName, String tier, boolean isActive) {}
```
- **불변** 데이터 캐리어. 자동으로 생성자/accessor/`equals`/`hashCode`/`toString` 생성.
- 이 프로젝트 전반의 이벤트(`OrderCreatedEvent`, `PaymentApprovedEvent` 등), 값 객체(`Money`, `OrderId`, `SubscriptionId`), DTO(`CreateOrderRequest`, `OrderSummary`) 가 모두 record.

### 2. Sealed (Java 17)
하위 타입을 명시적으로 제한. 패턴 매칭과 결합해 완전성 검사.

### 3. Switch Expression + Pattern Matching (Java 21 정식)
```java
// orders/domain/model/OrderStatus.java
return switch (this) {
    case CREATED -> target == PAYMENT_PENDING || target == CANCELLED;
    case PAYMENT_PENDING -> target == PAID || target == CANCELLED;
    case PAID -> target == SHIPPING;
    case SHIPPING -> target == DELIVERED;
    case DELIVERED, CANCELLED -> false;
};
```
- 표현식이라 값을 반환.
- enum 모든 케이스를 다루지 않으면 컴파일 에러 → **상태 전이 누락 방지**.

```java
// payments/internal/PaymentsPolicyConfig.java
return switch (subCtx.tier()) {
    case "premium" -> new SubscriptionDiscountPolicy(...);
    case "basic"   -> new SubscriptionDiscountPolicy(...);
    default        -> new NoDiscountPolicy();
};
```

### 4. Virtual Threads (Java 21 정식, JEP 444)
경량 스레드. OS 스레드를 점유하지 않고 JVM이 스케줄링.

```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true
```
- 한 줄로 Tomcat의 요청 처리 스레드가 가상 스레드로 전환.
- `FakePaymentGateway.process()`의 `Thread.sleep(300)`이 OS 스레드를 점유하지 않음.
- 자세한 내용은 `tomcat-virtual-threads.md` 참조.

### 5. Text Block (Java 15)
```java
.content("""
    {"customerName": "홍길동", "tier": "premium"}
    """)
```
- 통합 테스트의 JSON payload에서 활용.

### 6. var 지역변수 추론 (Java 10)
```java
var policy = new StandardShippingFeePolicy();
```

## 그 외 의미 있는 변화 (이 프로젝트 외)

| 기능 | JEP | 의의 |
|------|-----|------|
| ZGC Generational | JEP 439 | 큰 힙에서도 ms 단위 일시정지 |
| Sequenced Collections | JEP 431 | List/Set의 첫/끝 일관 API |
| Pattern Matching for switch | JEP 441 | 타입 + 가드 분기 |
| Record Patterns | JEP 440 | record 분해 매칭 |

## FastAPI/Python 대응

| Python | Java 21 |
|--------|---------|
| `@dataclass(frozen=True)` | `record` |
| `match` 문 | `switch` 표현식 + 패턴 매칭 |
| `asyncio` | Virtual Threads (다른 모델 — 동기 코드를 그대로) |
| `typing.Protocol` | `interface` |
| 삼중 따옴표 문자열 | text block |
