# 22 — Python match-case (Structural Pattern Matching)

> ShopTracker 의 정책 주입 핵심에는 *한 줄짜리 분기* 가 있다 : `match sub_ctx.tier: case "premium": ... case "basic": ... case _: ...`. 이 문법은 Python 3.10 (2021) 부터 들어왔고 단순 `if/elif` 체인을 대체할 뿐 아니라 *구조적 패턴 매칭* (Structural Pattern Matching) 이라는 더 강력한 기능을 가진다. ShopTracker 의 사용 예에서 시작해 match-case 의 본질을 정리.

---

## §0 왜 이 문법이 필요했나

### 0.1 if/elif 체인의 한계

Python 3.10 이전 :

```python
def discount_policy_for(tier: str) -> DiscountPolicy:
    if tier == "premium":
        return SubscriptionDiscountPolicy(Decimal("0.10"), "premium_subscription")
    elif tier == "basic":
        return SubscriptionDiscountPolicy(Decimal("0.05"), "basic_subscription")
    else:
        return NoDiscountPolicy()
```

기능적으로 동작한다. 그러나 :

1. **반복적 비교** — `tier == "premium"`, `tier == "basic"` 같은 *왼쪽 변수 반복*.
2. **switch 문 없음** — Python 은 오랫동안 다른 언어의 switch 가 *유의미하지 않다* 고 거부했다 (Guido 의 입장 : if/elif 면 충분).
3. **구조 분해 안 됨** — 입력이 dict / 튜플 / 객체일 때 *형태 자체* 를 매칭하고 싶은 경우 불가능.

### 0.2 다른 언어의 패턴 매칭 영향

Rust, Scala, Haskell, OCaml 같은 함수형 언어가 *구조적 패턴 매칭* 을 가진다 :

```rust
// Rust
match tier {
    Tier::Premium => SubscriptionDiscount(0.10),
    Tier::Basic => SubscriptionDiscount(0.05),
    _ => NoDiscount,
}
```

Python 3.10 의 PEP 634 가 이 발상을 가져왔다. 단순 switch 가 아니라 **타입 + 구조 + 값** 을 한 번에 매칭하는 도구.

### 0.3 ShopTracker 가 쓰는 곳

본 작업에서 도입된 두 곳 :

```python
# di_container.py — 할인 정책 분기
@provide(scope=Scope.REQUEST)
def discount_policy(self, sub_ctx: SubscriptionContext) -> DiscountPolicy:
    match sub_ctx.tier:
        case "premium":
            return SubscriptionDiscountPolicy(Decimal("0.10"), "premium_subscription")
        case "basic":
            return SubscriptionDiscountPolicy(Decimal("0.05"), "basic_subscription")
        case _:
            return NoDiscountPolicy()


# di_container.py — 배송비 정책 분기
@provide(scope=Scope.REQUEST)
def shipping_fee_policy(self, sub_ctx: SubscriptionContext) -> ShippingFeePolicy:
    match sub_ctx.tier:
        case "premium":
            return PremiumShippingFeePolicy()
        case "basic":
            return BasicShippingFeePolicy()
        case _:
            return StandardShippingFeePolicy()
```

이 두 곳이 정책 주입의 *유일한 매핑 진입점*. match-case 의 가독성이 직접적인 가치.

---

## §1 본질 메커니즘

### 1.1 기본 형태

```python
match subject:
    case pattern_1:
        # body
    case pattern_2 if guard:
        # body
    case _:
        # default
```

- `subject` : 매칭 대상 (변수 / 표현식)
- `pattern` : 매칭할 형태
- `if guard` : 추가 조건
- `_` : 와일드카드 — 항상 매칭

### 1.2 패턴 종류 7 가지

| 패턴 | 예 | 의미 |
|---|---|---|
| **Literal** | `case 0:`, `case "premium":` | 값 비교 (`==`) |
| **Capture** | `case x:` | 변수 바인딩 — *항상* 매칭됨 |
| **Wildcard** | `case _:` | 매칭하되 바인딩 안 함 |
| **Sequence** | `case [a, b, c]:`, `case (1, *rest):` | 리스트/튜플 분해 |
| **Mapping** | `case {"name": str(name)}:` | dict 분해 |
| **Class** | `case Point(x=0, y=y):` | 객체 attribute 분해 |
| **OR** | `case "premium" \| "vip":` | 합집합 |

### 1.3 ShopTracker 의 사용은 어떤 패턴인가

```python
match sub_ctx.tier:
    case "premium":      # ← Literal
        ...
    case "basic":        # ← Literal
        ...
    case _:              # ← Wildcard
        ...
```

가장 단순한 *literal + wildcard*. 다른 언어의 switch 와 거의 동일한 형태.

→ ShopTracker 가 이 단순 형태에 머무는 이유 : tier 가 str 이고 분기가 명확. 더 복잡한 패턴 (Class, Mapping) 은 도메인이 더 복잡할 때.

---

## §2 더 강력한 패턴 — 구조 분해

### 2.1 Sequence 패턴

```python
def parse_command(args: list[str]):
    match args:
        case []:
            return "no command"
        case ["create", name]:
            return f"create {name}"
        case ["delete", *names]:
            return f"delete many: {names}"
        case [cmd, *rest]:
            return f"unknown: {cmd}, args={rest}"
```

- `[]` : 빈 리스트
- `["create", name]` : 정확히 2 원소, 첫째가 "create", 둘째를 `name` 에 바인딩
- `["delete", *names]` : 첫째가 "delete", 나머지 모두 `names` 에 (튜플 unpacking 같은 *)
- `[cmd, *rest]` : 첫째 `cmd`, 나머지 `rest`

이건 if/elif 로 풀려면 길다 :

```python
if not args:
    return "no command"
elif len(args) == 2 and args[0] == "create":
    name = args[1]
    return f"create {name}"
elif len(args) >= 1 and args[0] == "delete":
    names = args[1:]
    ...
```

→ match 가 *구조 + 바인딩* 을 한 줄에.

### 2.2 Mapping 패턴

```python
def handle_event_payload(payload: dict):
    match payload:
        case {"type": "order_created", "id": str(order_id)}:
            return f"order {order_id}"
        case {"type": "payment", "amount": int(amount)} if amount > 0:
            return f"payment {amount}"
        case {"type": event_type}:
            return f"unknown type: {event_type}"
        case _:
            return "no type"
```

- `{"type": "order_created", "id": str(order_id)}` : key 가 "type" 이고 값이 정확히 "order_created", "id" 가 있고 값이 str → str 의 *내용* 을 `order_id` 에 바인딩
- `if amount > 0` : guard
- 추가 키가 있어도 OK (subset 매칭)

→ JSON / dict 처리에 자연스러움. ShopTracker 의 이벤트 detail (dict) 을 분기할 때 쓸 수 있음.

### 2.3 Class 패턴 (가장 강력)

```python
from dataclasses import dataclass

@dataclass
class Move:
    direction: str
    distance: int

@dataclass
class Stop:
    pass

@dataclass
class Turn:
    angle: int

def describe(action):
    match action:
        case Move(direction="north", distance=d):
            return f"{d}m 북"
        case Move(direction=dir, distance=d) if d > 100:
            return f"{dir}으로 멀리 ({d}m)"
        case Stop():
            return "정지"
        case Turn(angle=a) if -45 < a < 45:
            return f"살짝 {a}도"
        case Turn(angle=a):
            return f"크게 {a}도"
```

이게 *구조적 패턴 매칭의 진짜 가치*. 객체의 타입 + attribute 를 한 번에 매칭.

ShopTracker 에 적용해보면 :

```python
def explain_payment(payment):
    match payment.status:
        case PaymentStatus.APPROVED:
            return f"승인 ({payment.transaction_id})"
        case PaymentStatus.REJECTED:
            return "거절"
        case PaymentStatus.PENDING:
            return "처리중"
```

또는 이벤트 분기 :

```python
async def log_event(event):
    match event:
        case OrderCreatedEvent(order_id=oid, total_amount=amt):
            logger.info("주문 생성", order_id=oid, amount=amt)
        case PaymentApprovedEvent(payment_id=pid, applied_discount_type=t):
            logger.info("결제 승인", payment_id=pid, discount_type=t)
        case PaymentRejectedEvent(reason=r):
            logger.warn("결제 거절", reason=r)
```

이건 isinstance + getattr 체인보다 훨씬 깔끔.

### 2.4 OR 패턴

```python
match tier:
    case "premium" | "vip":
        return high_tier_policy
    case "basic" | "standard":
        return mid_tier_policy
    case _:
        return default_policy
```

`|` 로 합집합. 단, OR 패턴 안에서는 *이름 바인딩 불가* (양쪽이 다른 변수면 안 됨) :

```python
case Point(x=x) | Line(start=x):    # ❌ x 가 양쪽 다른 의미
    ...
```

### 2.5 Capture vs Literal 의 함정

```python
PREMIUM = "premium"
match tier:
    case PREMIUM:        # ← Literal? Capture?
        ...
```

답 : **Capture**. Python 의 match 는 *대문자로 시작하지 않는 이름* 을 capture 로 본다. 즉 `PREMIUM` 이 `tier` 의 값을 *바인딩하는 새 변수* 가 되어 *항상 매칭*. 의도와 정반대.

해결 :
1. **점 기호 사용** : `case constants.PREMIUM:` — dotted name 은 literal
2. **모듈 / 클래스 attribute** : `case MyConsts.PREMIUM:` — 같은 이유
3. **Enum 사용** : `case Tier.PREMIUM:` — Class 패턴으로 인식 (대문자라서)

ShopTracker 가 단순 str literal (`"premium"`) 을 쓰는 이유 : 이 함정을 피함.

---

## §3 ShopTracker 의 match 사용을 어떻게 확장할 수 있나

### 3.1 새 등급 추가 (예 : VIP)

```python
match sub_ctx.tier:
    case "vip":                              # ← 새 케이스 추가
        return SubscriptionDiscountPolicy(Decimal("0.20"), "vip_subscription")
    case "premium":
        return SubscriptionDiscountPolicy(Decimal("0.10"), "premium_subscription")
    case "basic":
        return SubscriptionDiscountPolicy(Decimal("0.05"), "basic_subscription")
    case _:
        return NoDiscountPolicy()
```

새 `SubscriptionDiscountPolicy` 인스턴스 추가만 하면 됨. **handler 수정 0**. 03 §5 의 정책 주입 효과.

### 3.2 OR 로 묶기 (예 : 비활성 구독은 모두 NoDiscount)

```python
match (sub_ctx.tier, sub_ctx.is_active):
    case ("premium", True):
        return SubscriptionDiscountPolicy(Decimal("0.10"), "premium_subscription")
    case ("basic", True):
        return SubscriptionDiscountPolicy(Decimal("0.05"), "basic_subscription")
    case (_, False):                         # ← 비활성이면 무조건 NoDiscount
        return NoDiscountPolicy()
    case _:
        return NoDiscountPolicy()
```

튜플 매칭 + wildcard. is_active 가 False 면 어떤 tier 든 NoDiscount.

→ 현재 ShopTracker 의 SubscriptionContext 폴백 로직 (`if sub is None or not sub.is_active(): return guest`) 은 Provider 안에서 처리. match 안에서 한꺼번에 처리하는 변형도 가능.

### 3.3 Class 패턴으로 SubscriptionContext 자체 분기

```python
match sub_ctx:
    case SubscriptionContext(tier="premium", is_active=True):
        return PremiumDiscountPolicy()
    case SubscriptionContext(is_active=False):
        return NoDiscountPolicy()
    case SubscriptionContext(tier=t):
        return tier_to_policy(t)
```

장점 : SubscriptionContext 의 구조를 직접 분기.
단점 : 세 단계 if 가 더 명확할 때도 있음. 가독성 트레이드오프.

ShopTracker 가 단순 `tier` 만 보는 이유 : 분기 차원이 1 개 (tier) — 더 복잡해지기 전엔 단순.

---

## §4 함정

### 4.1 Capture 의 함정 (§2.5 재확인)

```python
TIER_PREMIUM = "premium"

match user_tier:
    case TIER_PREMIUM:       # 의도: "premium" 과 비교
        ...                  # 실제: 무조건 매칭됨, TIER_PREMIUM 가 user_tier 에 바인딩
```

→ Linter (mypy, pylint, ruff) 가 경고 — IDE 가 잡아줌. 그러나 모르고 쓰면 **모든 케이스가 첫 case 로 들어감**.

해결 :
- `match.case Constants.TIER_PREMIUM:` (dotted)
- `if user_tier == TIER_PREMIUM:` (그냥 if)
- 직접 literal: `case "premium":`

### 4.2 fall-through 없음

C/C++ switch 와 달리 Python match 는 *fall-through* 안 한다 :

```c
switch (tier) {
    case "premium":
        printf("premium");
        // break 없으면 다음 case 도 실행
    case "basic":
        printf("basic");
}
```

```python
match tier:
    case "premium":
        print("premium")
        # 다음 case 로 안 흐름. 자동 break.
    case "basic":
        print("basic")
```

→ Python 은 매칭된 첫 case 만 실행. C 의 함정이 없음. (장점)

### 4.3 case 순서 중요

```python
match tier:
    case _:                      # ← 와일드카드가 위에 있음
        return default
    case "premium":              # ← 영원히 도달 X (dead code)
        return premium
```

`_` 는 항상 매칭 → 그 아래는 dead code. mypy 등이 경고하지만 런타임은 조용히 통과.

→ **wildcard 는 항상 마지막**.

### 4.4 case _ 누락 시 None 반환

```python
def policy_for(tier: str):
    match tier:
        case "premium":
            return P()
        case "basic":
            return B()
    # 매칭 안 되면 함수가 끝남 → None 반환
```

Python 의 함수는 명시적 return 없으면 None. match 의 모든 case 가 실패하면 함수 끝.

→ **명시적 `case _:`** 또는 함수 마지막에 `raise` 또는 `return default`.

ShopTracker 는 `case _: return NoDiscountPolicy()` 로 처리. 명시적.

### 4.5 패턴 안에서 표현식 못 씀

```python
threshold = 100
match amount:
    case x if x > threshold:    # ✅ guard 안에서는 OK
        ...
    case > threshold:            # ❌ syntax error — 패턴은 표현식이 아님
        ...
```

패턴은 *문법적 형태* 만. 표현식 (함수 호출, 비교, 산술) 은 못 씀. 표현식 결과로 분기하려면 if guard.

### 4.6 dict 매칭의 subset 동작

```python
match data:
    case {"name": name}:
        # data 가 {"name": "alice"} 든 {"name": "alice", "age": 20} 든 매칭됨
        ...
```

**추가 키가 있어도 매칭** = subset 매칭. 정확 매칭이 필요하면 :

```python
case {"name": name, **rest} if not rest:    # rest 가 비어야
    ...
```

또는 길이 비교 가드.

### 4.7 list 매칭의 모호성

```python
match args:
    case [x, *rest]:
        ...
```

`x = args[0], rest = args[1:]`. 빈 리스트는 매칭 안 됨 — `case []` 로 따로.

---

## §5 다른 언어와의 비교

### 5.1 Java switch (역사)

```java
// Java 17 expression switch
String description = switch (tier) {
    case "premium" -> "10% off";
    case "basic" -> "5% off";
    default -> "no discount";
};
```

차이 :
- Java 21 부터 Pattern matching for switch (Class 패턴 비슷한 것 도입).
- Python 의 match 가 더 일찍, 더 단순하게 들어옴 (3.10 == 2021).

### 5.2 Rust match

```rust
match tier {
    Tier::Premium => Discount::new(0.10),
    Tier::Basic => Discount::new(0.05),
    _ => Discount::none(),
}
```

차이 :
- Rust 는 *exhaustive* 검사 — 모든 케이스 다루지 않으면 컴파일 에러.
- Python 은 그런 검사 없음 (3.10 의 한계). mypy 의 `match-statement-must-be-exhaustive` 는 일부 경고 가능하지만 표준 X.

### 5.3 Scala match

```scala
tier match {
  case "premium" => Discount(0.10)
  case "basic" => Discount(0.05)
  case _ => Discount.None
}
```

거의 같다. Python 의 PEP 634 가 Scala 영향을 많이 받음.

### 5.4 Haskell / OCaml

```haskell
discountFor :: String -> Double
discountFor "premium" = 0.10
discountFor "basic" = 0.05
discountFor _ = 0.0
```

함수 정의 자체에 패턴. Python 은 함수 안에서만.

### 5.5 JavaScript / TypeScript

```typescript
function discountFor(tier: Tier): Discount {
    switch (tier) {
        case 'premium': return new Discount(0.10);
        case 'basic': return new Discount(0.05);
        default: return Discount.none;
    }
}
```

JS / TS 는 여전히 단순 switch. Class 패턴 매칭 같은 건 없음 (2025 시점). TC39 의 pattern matching proposal 이 Stage 1 에 있음.

---

## §6 가독성 가이드

### 6.1 if/elif 가 더 나은 경우

- 조건이 *연속 비교* 일 때 (`if 0 < x < 100`).
- 조건이 *복잡한 표현식* 일 때 (`if user.is_admin and order.total > 1000`).
- 분기가 2~3 개일 때 (오버헤드 < 단순함).

### 6.2 match 가 더 나은 경우

- 같은 변수의 *값 분기* 가 4 개 이상.
- *구조 분해* 가 필요할 때 (dict / 튜플 / dataclass).
- *exhaustive* 의도가 명시적일 때 (모든 case + wildcard).

### 6.3 ShopTracker 의 선택

```python
match sub_ctx.tier:
    case "premium": return PremiumDiscountPolicy()
    case "basic": return BasicDiscountPolicy()
    case _: return NoDiscountPolicy()
```

- 같은 변수 (`sub_ctx.tier`) 분기
- 케이스 3 개
- exhaustive (wildcard)

→ if/elif 보다 *살짝* 더 깔끔. 큰 차이는 아님. 일관성을 위해 모든 정책 분기를 match 로 통일.

---

## §7 학습 포인트 (한 줄 요약)

1. **match-case = Python 3.10+ 의 구조적 패턴 매칭** — 단순 switch 가 아니다.
2. **7 종 패턴** : Literal, Capture, Wildcard, Sequence, Mapping, Class, OR.
3. **case 의 변수 이름은 capture** — dotted name (`Const.X`) 만 literal 로 인식.
4. **fall-through 없음** — 첫 매칭만 실행, C 의 break 함정 없음.
5. **wildcard `_` 는 마지막** — 위에 두면 dead code.
6. **`case _:` 누락 시 함수가 None 반환** — 명시적 default.
7. **subset 매칭 (dict)** — 추가 키 OK, 정확 매칭은 guard.
8. **exhaustive 검사 없음** — Rust / Haskell 과 다름. 명시적 default 로 보완.
9. **ShopTracker 사용은 단순 literal + wildcard** — 정책 분기는 1 차원.
10. **확장 (VIP 등급, 비활성 구독 처리) 시 OR / 튜플 / Class 패턴 활용 가능**.

---

## 추가 참고

- PEP 634 — Structural Pattern Matching: Specification: https://peps.python.org/pep-0634/
- PEP 635 — 동기 (motivation): https://peps.python.org/pep-0635/
- PEP 636 — 튜토리얼: https://peps.python.org/pep-0636/
- ShopTracker 의 사용처 : `di_container.py` 의 `discount_policy`, `shipping_fee_policy`
- 03 — DI 정책 주입 — match 가 사용되는 맥락
- 다음 글 : `concepts/04-object-model.md` (Python 의 객체/타입 시스템 — match 가 가능한 이유)
