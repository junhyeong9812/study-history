# 21 — Strategy

> **분류**: 행위 패턴 (Behavioral)
> **한 줄**: 알고리즘을 캡슐화하여 *교체 가능* 하게. 정책 주입의 본체.

---

## 1. 의도 (Intent)

> "Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from clients that use it."

알고리즘을 *값* 처럼 다루기 — 주입 / 교체 / 합성.

## 2. 문제 상황 (Motivation)

배송비 계산 — 구독 등급에 따라 다름.

```java
// ❌ 분기로
class ShippingCalculator {
    public Money calculate(String tier, Money orderAmount) {
        if (tier.equals("premium")) {
            return Money.zero();
        } else if (tier.equals("basic")) {
            if (orderAmount.greaterThan(Money.of(30000))) return Money.zero();
            return Money.of(1500);
        } else {
            if (orderAmount.greaterThan(Money.of(50000))) return Money.zero();
            return Money.of(3000);
        }
    }
}
```

새 등급 (VIP) 추가 시 메서드 수정. 단위 테스트가 *모든 등급* 을 한 메서드에서 검증.

해결 : 각 정책을 *클래스* 로.

```java
ShippingFeePolicy policy = chooseFor(tier);
Money fee = policy.calculate(orderAmount);
```

## 3. 구조 (Structure)

```
Context                    Strategy (interface)
  ┌─────────────────┐       ┌──────────────┐
  │ - strategy      │ ────► │ algorithm()  │
  │ + execute() {   │       └──────┬───────┘
  │     strategy.algo()              │
  │ }               │         ┌─────┴──────┐
  └─────────────────┘         │            │
                          ConcreteStrategyA, B
```

## 4. 참여자 (Participants)

- **Strategy**: 알고리즘 인터페이스.
- **ConcreteStrategy**: 알고리즘 구현.
- **Context**: Strategy 보유 + 사용. 어떤 ConcreteStrategy 인지 모름.

## 5. Java 예제

### 5.1 정통 Strategy

```java
import java.math.BigDecimal;

// Strategy
interface ShippingFeePolicy {
    BigDecimal calculate(BigDecimal orderAmount);
}

// Concrete Strategies
class StandardShipping implements ShippingFeePolicy {
    @Override public BigDecimal calculate(BigDecimal amount) {
        return amount.compareTo(BigDecimal.valueOf(50000)) >= 0
            ? BigDecimal.ZERO
            : BigDecimal.valueOf(3000);
    }
}

class BasicShipping implements ShippingFeePolicy {
    @Override public BigDecimal calculate(BigDecimal amount) {
        return amount.compareTo(BigDecimal.valueOf(30000)) >= 0
            ? BigDecimal.ZERO
            : BigDecimal.valueOf(1500);
    }
}

class PremiumShipping implements ShippingFeePolicy {
    @Override public BigDecimal calculate(BigDecimal amount) {
        return BigDecimal.ZERO;
    }
}

// Context
class OrderProcessor {
    private final ShippingFeePolicy policy;

    public OrderProcessor(ShippingFeePolicy policy) {
        this.policy = policy;
    }

    public BigDecimal calculateTotal(BigDecimal orderAmount) {
        return orderAmount.add(policy.calculate(orderAmount));
    }
}

// 사용
public class Main {
    public static void main(String[] args) {
        // 등급에 따라 정책 주입
        ShippingFeePolicy policy = pickFor("premium");
        OrderProcessor processor = new OrderProcessor(policy);

        BigDecimal total = processor.calculateTotal(BigDecimal.valueOf(20000));
        System.out.println(total);
    }

    static ShippingFeePolicy pickFor(String tier) {
        return switch (tier) {
            case "premium" -> new PremiumShipping();
            case "basic" -> new BasicShipping();
            default -> new StandardShipping();
        };
    }
}
```

### 5.2 함수형 (Java 8+)

```java
import java.util.function.Function;
import java.math.BigDecimal;

class OrderProcessor {
    private final Function<BigDecimal, BigDecimal> shippingFee;

    public OrderProcessor(Function<BigDecimal, BigDecimal> shippingFee) {
        this.shippingFee = shippingFee;
    }

    public BigDecimal calculateTotal(BigDecimal amount) {
        return amount.add(shippingFee.apply(amount));
    }
}

// 사용 — 람다 한 줄로
OrderProcessor premium = new OrderProcessor(amount -> BigDecimal.ZERO);
OrderProcessor basic = new OrderProcessor(amount ->
    amount.compareTo(BigDecimal.valueOf(30000)) >= 0
        ? BigDecimal.ZERO
        : BigDecimal.valueOf(1500));
```

람다 = 1-메서드 인터페이스의 익명 구현. 단순한 알고리즘은 람다가 깔끔.

### 5.3 DI 와 결합 (Spring)

```java
@Component
public class PremiumShippingPolicy implements ShippingFeePolicy { ... }

@Component
public class BasicShippingPolicy implements ShippingFeePolicy { ... }

@Service
public class OrderService {
    private final Map<String, ShippingFeePolicy> policies;

    @Autowired
    public OrderService(Map<String, ShippingFeePolicy> policies) {
        // Spring 이 모든 ShippingFeePolicy 를 자동 주입 — 빈 이름이 키
        this.policies = policies;
    }

    public BigDecimal feeFor(String tier, BigDecimal amount) {
        return policies.getOrDefault(tier + "ShippingPolicy", policies.get("standardShippingPolicy"))
            .calculate(amount);
    }
}
```

## 6. 변형 (Variants)

### 6.1 Default Strategy
```java
class OrderProcessor {
    private ShippingFeePolicy policy = new StandardShipping();    // default
    public void setPolicy(ShippingFeePolicy p) { this.policy = p; }
}
```

### 6.2 Strategy 합성
```java
ShippingFeePolicy combined = (amount) -> {
    BigDecimal base = standard.calculate(amount);
    return base.subtract(coupon.discount(amount));
};
```

### 6.3 Strategy Registry
```java
Map<String, ShippingFeePolicy> registry = ...;
ShippingFeePolicy p = registry.get(tier);
```

## 7. 함정 / 흔한 오해

### 7.1 State (20) 와의 차이
- **State**: 객체의 *상태에 따라* 자기 행동 변경. State 가 *전이* 결정.
- **Strategy**: *외부* 가 알고리즘 결정.
- 코드는 동일해 보일 수 있음. 의도가 다름.

### 7.2 람다와 클래스
한 줄 알고리즘은 람다, 복잡하면 클래스 — 상황에 맞춰.

### 7.3 Strategy 의 상태
Strategy 가 stateless 면 인스턴스 공유 가능 (Singleton / Flyweight). stateful 이면 매번 새로.

### 7.4 Strategy vs if/else
Strategy 가 *2~3 가지뿐* 이면 단순 if/else 가 더 간결할 수도. *변동 가능성 + 복잡도* 보고 선택.

## 8. 관련 패턴

- **State (20)**: 형태 비슷. 의도 다름.
- **Bridge (07)**: 추상 + 구현 분리. Strategy 는 알고리즘 교체.
- **Template Method (22)**: Template 의 빈 칸을 Strategy 로 채우는 변형.
- **Decorator (09)**: Strategy 를 wrapping 으로.
- **Factory Method (03) / Abstract Factory (01)**: Strategy 인스턴스 선택.

## 9. 실무 사례

- `Comparator<T>` — 정렬 전략 (`Collections.sort(list, comparator)`)
- `Runnable` — 실행 전략
- `Function<T, R>`, `Predicate<T>` — 함수형 strategy
- `java.security.MessageDigest` — 해시 알고리즘 (SHA-256, MD5, ...)
- `javax.crypto.Cipher` — 암호화 알고리즘
- Spring `BCryptPasswordEncoder` ←→ `Argon2PasswordEncoder`
- TLS / SSL 의 cipher suite 협상
- 결제 PG 선택 (Stripe / Toss / Inicis)
- 추천 엔진의 알고리즘 선택 (collaborative / content-based)
- 압축 알고리즘 (gzip / zstd / brotli)

```java
// Java 의 Comparator 가 Strategy 의 산업 표준
List<Order> orders = ...;
orders.sort(Comparator.comparing(Order::getCreatedAt));        // 한 strategy
orders.sort(Comparator.comparing(Order::getTotal).reversed());  // 다른 strategy
```

> 본 ShopTracker 의 `DiscountPolicy`, `ShippingFeePolicy` 가 정확히 Strategy 패턴. DI 가 어떤 ConcreteStrategy 를 주입할지 결정.
