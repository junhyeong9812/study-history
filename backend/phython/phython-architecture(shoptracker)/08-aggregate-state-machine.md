# 08 — Aggregate Root + 상태 머신 (도메인 invariant 의 보호)

> `Order` 의 status 가 `CREATED → PAYMENT_PENDING → PAID → SHIPPING → DELIVERED` 로만 흐르고, `PAID → CANCELLED` 가 절대 불가능하도록 *도메인 객체 자신* 이 강제한다. 이게 ShopTracker 의 `Order.cancel()` 한 줄이 보호하는 invariant 다. 이 글은 Aggregate Root + 상태머신 + Factory method 가 어떻게 *비즈니스 규칙을 코드 레벨에서 강제* 하는지를 정리.

---

## §0 문제 정의 — Anemic Model 의 함정

### 0.1 흔한 anti-pattern : Anemic Domain Model

```python
# anti-pattern
@dataclass
class Order:
    id: UUID
    status: str
    total: Decimal
    items: list

# Service 가 모든 로직을 가짐
class OrderService:
    def cancel(self, order: Order):
        if order.status == "paid":
            raise CannotCancelPaidOrder()
        order.status = "cancelled"

    def mark_paid(self, order: Order):
        if order.status != "payment_pending":
            raise InvalidTransition()
        order.status = "paid"
```

문제 :
- 비즈니스 규칙 (어떤 상태 전이가 가능한가) 이 **Service 에 흩어짐**.
- Service A 는 검증, Service B 는 잊고 직접 `order.status = "paid"` 로 바꿈 → 상태 깨짐.
- 도메인 객체는 데이터 컨테이너에 불과. 객체지향의 캡슐화 실종.
- 테스트 시 *각 service 별로* 검증을 다시 깔아야.

이게 Martin Fowler 가 *Anemic Domain Model* 이라 부른 안티 패턴.

### 0.2 해결의 방향 — Tell, Don't Ask

> *객체에게 데이터를 묻고 외부가 결정하지 말고, 객체에게 행위를 시켜라.*

```python
# 좋은 도메인
order.cancel()                 # ← 객체가 알아서 검증 + 전이
order.mark_paid()              # ← 마찬가지

# 외부에서 직접 못 함
order.status = "paid"          # ← 막혀야 함 (private 또는 setter 봉쇄)
```

도메인 객체가 **자기 invariant 를 자기가 지킨다**. 그게 OOP 의 본질.

---

## §1 본질 메커니즘

### 1.1 Aggregate Root (DDD)

> **하나 이상의 객체로 구성된 일관성 단위**. 외부는 Aggregate Root 를 통해서만 내부 객체에 접근. 트랜잭션의 단위이자 invariant 의 단위.

ShopTracker 의 `Order` 가 Aggregate Root :
- `Order` 가 `OrderItem` 들을 보유
- 외부는 `OrderItem` 을 직접 만들거나 수정하지 않음 (개념상)
- `Order` 메서드를 통해서만 변경
- 저장 / 조회 단위 = Order (하나의 Order 와 그 items 가 한 트랜잭션에서 함께 저장)

```
┌─ Aggregate Root: Order ─┐
│  id, customer_name      │
│  status (state machine) │
│  total_amount           │
│  ┌─ OrderItem ─┐        │ ← 외부 접근 차단 (Order 통해서만)
│  │  product    │        │
│  │  quantity   │        │
│  │  unit_price │        │
│  └─────────────┘        │
└─────────────────────────┘
```

### 1.2 Factory Method

생성자 (`__init__`) 는 *복원* 용 (DB 에서 불러올 때) 이고, *새 생성* 은 별도 factory method (`create`) 로 :

| | `__init__` (또는 dataclass 의 자동) | `Order.create(...)` |
|---|---|---|
| 용도 | DB → 객체 복원 (이미 검증된 데이터) | 새 객체 생성 (검증 필요) |
| 검증 | X (이미 검증됨) | O (모든 invariant) |
| ID | 외부에서 부여 (DB 의 ID) | uuid4() 자동 생성 |
| 시점 | infrastructure 에서 호출 | application 에서 호출 |

분리 이유 : DB 복원 시까지 검증을 돌리면 *과거의* 잘못된 데이터가 못 올라옴. 또 검증 비용도 낭비.

### 1.3 상태 머신 (State Machine)

> 객체가 가질 수 있는 상태와, 상태 사이의 *허용된 전이* 를 명시.

```
       CREATED
        / \
       /   \
  PAYMENT  CANCELLED
  PENDING ─────► (terminal)
   / \
  /   \
 PAID  CANCELLED
  │     ▲
  ▼     X (불가)
SHIPPING
  │
  ▼
DELIVERED
(terminal)
```

`PAID → CANCELLED` 가 막힌다는 것은 **결제된 주문은 코드 레벨에서 절대 취소될 수 없다** 는 강한 보장. 운영 실수, 새 개발자, 잘못된 핸들러 — 모두 이 한 줄에 막힘.

---

## §2 ShopTracker 코드 정독

### 2.1 OrderStatus — 상태 정의 + 전이 규칙

```python
# orders/domain/value_objects.py
class OrderStatus(str, Enum):
    CREATED = "created"
    PAYMENT_PENDING = "payment_pending"
    PAID = "paid"
    SHIPPING = "shipping"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

    def can_transition_to(self, target: "OrderStatus") -> bool:
        return target in _VALID_TRANSITIONS.get(self, set())


_VALID_TRANSITIONS: dict[OrderStatus, set[OrderStatus]] = {
    OrderStatus.CREATED:         {OrderStatus.PAYMENT_PENDING, OrderStatus.CANCELLED},
    OrderStatus.PAYMENT_PENDING: {OrderStatus.PAID, OrderStatus.CANCELLED},
    OrderStatus.PAID:            {OrderStatus.SHIPPING},   # ← CANCELLED 없음!
    OrderStatus.SHIPPING:        {OrderStatus.DELIVERED},
    OrderStatus.DELIVERED:       set(),
    OrderStatus.CANCELLED:       set(),
}
```

설계 디테일 :

- **`(str, Enum)` 다중 상속** : `OrderStatus.PAID == "paid"` → True. JSON 직렬화 시 자동으로 문자열. DB 에 string 으로 저장 가능. Pydantic 도 자동 처리.
- **`can_transition_to`** : 도메인 객체에 검증 책임을 *위임* 받는 Enum 의 메서드. 단순 데이터가 아니라 *행위* 가 있는 객체.
- **`_VALID_TRANSITIONS`** : 모듈 레벨 dict. private (밑줄 prefix). 변경되지 않는 *상수*.
- **`set()` (빈 집합)** = terminal state. `DELIVERED` / `CANCELLED` 에서는 어디로도 못 감.
- **dict.get(self, set())** : 만에 하나 enum 에 새 값을 추가하고 매핑을 잊으면 빈 집합 → 어디로도 못 감 → 안전한 기본값. 단, 그건 버그라서 별도로 잡아야 (예 : 테스트로 모든 enum 값이 매핑에 있는지 검증).

### 2.2 Order.create — Factory + 검증

```python
# orders/domain/entities.py:48-85
@classmethod
def create(cls, customer_name: str, items: list[OrderItem]) -> "Order":
    # 규칙 1: 고객 이름 필수
    if not customer_name or not customer_name.strip():
        raise InvalidOrderError("고객 이름이 비어있습니다")

    # 규칙 2: 최소 1개 항목
    if not items:
        raise InvalidOrderError("주문 항목이 비어있습니다")

    # 규칙 3: 각 항목의 수량 > 0, 가격 > 0
    for item in items:
        if item.quantity <= 0:
            raise InvalidOrderError(f"수량이 0 이하입니다: {item.product_name}")
        if not item.unit_price.is_positive:
            raise InvalidOrderError(f"가격이 0 이하입니다: {item.product_name}")

    # 총액 자동 계산
    total = items[0].subtotal
    for item in items[1:]:
        total = total.add(item.subtotal)

    now = datetime.now(UTC)
    return cls(
        id=uuid4(),
        customer_name=customer_name.strip(),
        items=items,
        status=OrderStatus.CREATED,
        total_amount=total,
        created_at=now,
        updated_at=now,
    )
```

읽어보기 :

- `@classmethod` + `cls` — `Order.create(...)` 형태로 호출. 인스턴스가 없을 때 호출하는 패턴.
- **검증 → 계산 → 생성** 의 3 단 구조.
  - 검증 : 입력의 정합성. 실패 시 `InvalidOrderError` (도메인 예외).
  - 계산 : `total_amount` 자동 계산 — 호출자가 잘못 넘길 여지를 없앰.
  - 생성 : 이 시점에 `cls(...)` 로 dataclass 생성. ID 자동 부여, 초기 상태 `CREATED`.
- `customer_name.strip()` — 입력 정규화. "  alice  " → "alice". 도메인의 표준 표현.
- `total = items[0].subtotal; for item in items[1:]: total = total.add(...)` — Money 가 불변 값 객체라 `+=` 가 새 객체 만들고 재할당.
- `now = datetime.now(UTC)` — `created_at`, `updated_at` 둘 다 같은 시각. UTC 명시.

> **함정** : 코드에 `Order.create(customer_name=..., itmes=items)` 오타가 command_handlers.py 에 있음 (`itmes`). Phase 2 진입 전에 잡혀야 할 버그.

### 2.3 Order._transition_to — 상태 전이의 게이트

```python
# entities.py:87-92
def _transition_to(self, target: OrderStatus) -> None:
    """상태 전이 공통 로직. 규칙 위반 시 예외 발생."""
    if not self.status.can_transition_to(target):
        raise InvalidStatusTransition(self.status.value, target.value)
    self.status = target
    self.updated_at = datetime.now(UTC)
```

핵심 :

- `_transition_to` (밑줄) — **private**. 외부에서 직접 호출 안 함. 자기 자신의 `mark_*` / `cancel` 메서드만 호출.
- **이 한 곳에만 검증 로직** — 모든 상태 전이가 여기를 통과. DRY.
- `updated_at` 갱신도 같은 곳에서 — 까먹을 여지 없음.
- `raise InvalidStatusTransition` — 도메인 예외. infrastructure 예외 (DB error 등) 와 분리.

### 2.4 의도가 드러나는 메서드 이름

```python
# entities.py:94-112
def mark_payment_pending(self) -> None:
    self._transition_to(OrderStatus.PAYMENT_PENDING)

def mark_paid(self) -> None:
    self._transition_to(OrderStatus.PAID)

def mark_shipping(self) -> None:
    self._transition_to(OrderStatus.SHIPPING)

def mark_delivered(self) -> None:
    self._transition_to(OrderStatus.DELIVERED)

def cancel(self) -> None:
    self._transition_to(OrderStatus.CANCELLED)
```

이 메서드들의 가치 :

- **이름이 비즈니스 동사** — `mark_paid`, `cancel`. setter (`set_status`) 가 아님.
- 호출 코드가 *비즈니스 흐름* 으로 읽힘 :
  ```python
  order = Order.create(...)
  order.mark_payment_pending()
  await repo.save(order)
  # ... payment 처리 ...
  order.mark_paid()
  ```
- 새 상태 추가 시 *메서드 추가* — 인터페이스가 진화함을 가시화.

### 2.5 Aggregate Root 의 invariant 강제

```python
@dataclass
class Order:
    items: list[OrderItem]
    total_amount: Money
```

**문제** : 외부가 `order.items.append(new_item)` 으로 직접 추가하면 `total_amount` 와 안 맞음.

ShopTracker Phase 1 은 이 부분이 *완벽히 막혀 있지 않음* — Python 의 `dataclass` 가 기본적으로 mutable 이라 `order.items.append(...)` 가 가능. 학습 단계라 의도적 단순.

본격적으로 보호하려면 :

```python
@dataclass
class Order:
    _items: list[OrderItem]   # private

    @property
    def items(self) -> tuple[OrderItem, ...]:   # 불변 view
        return tuple(self._items)

    def add_item(self, item: OrderItem) -> None:
        if self.status != OrderStatus.CREATED:
            raise CannotAddItemAfterCreation()
        self._items.append(item)
        self.total_amount = self._calculate_total()
```

→ 외부는 `add_item` 을 통해서만, invariant (status 체크 + total 재계산) 가 자동.

---

## §3 직접 구현 — 미니 상태머신

```python
from enum import Enum

class TicketStatus(str, Enum):
    OPEN = "open"
    IN_PROGRESS = "in_progress"
    RESOLVED = "resolved"
    CLOSED = "closed"
    REOPENED = "reopened"

    def can_transition_to(self, target):
        return target in _VALID.get(self, set())

_VALID = {
    TicketStatus.OPEN:        {TicketStatus.IN_PROGRESS, TicketStatus.CLOSED},
    TicketStatus.IN_PROGRESS: {TicketStatus.RESOLVED, TicketStatus.OPEN},
    TicketStatus.RESOLVED:    {TicketStatus.CLOSED, TicketStatus.REOPENED},
    TicketStatus.CLOSED:      set(),
    TicketStatus.REOPENED:    {TicketStatus.IN_PROGRESS},
}

@dataclass
class Ticket:
    id: UUID
    status: TicketStatus
    title: str

    @classmethod
    def create(cls, title: str) -> "Ticket":
        if not title.strip():
            raise InvalidTicketError("title required")
        return cls(id=uuid4(), status=TicketStatus.OPEN, title=title.strip())

    def _transition_to(self, target: TicketStatus):
        if not self.status.can_transition_to(target):
            raise InvalidStatusTransition(self.status.value, target.value)
        self.status = target

    def start(self):    self._transition_to(TicketStatus.IN_PROGRESS)
    def resolve(self):  self._transition_to(TicketStatus.RESOLVED)
    def close(self):    self._transition_to(TicketStatus.CLOSED)
    def reopen(self):   self._transition_to(TicketStatus.REOPENED)
```

테스트 :

```python
def test_cannot_close_in_progress_directly():
    t = Ticket.create("bug")
    t.start()
    with pytest.raises(InvalidStatusTransition):
        t.close()           # IN_PROGRESS → CLOSED 직접 불가

def test_resolve_then_close():
    t = Ticket.create("bug")
    t.start()
    t.resolve()
    t.close()
    assert t.status == TicketStatus.CLOSED
```

규칙 변경이 *테이블 한 줄* (`_VALID`) 에 모임. 새 규칙 추가가 한 줄.

---

## §4 함정

### 4.1 Enum 추가 시 매핑 누락

`OrderStatus.REFUNDED` 를 추가했는데 `_VALID_TRANSITIONS` 에 매핑을 안 넣으면 → REFUNDED 에서 어디로도 못 감 (terminal 처럼 동작). 컴파일 시 못 잡음.

해결 : 테스트로 *모든 enum 값이 매핑에 있는지* 검증.

```python
def test_all_statuses_mapped():
    for status in OrderStatus:
        assert status in _VALID_TRANSITIONS
```

### 4.2 dataclass 의 가변성

`@dataclass` 는 mutable. 외부가 `order.status = "paid"` 직접 변경 가능 — 검증 우회.

해결 :
- `frozen=True` + `dataclasses.replace` 로 새 인스턴스. 단, items 같은 list 는 여전히 mutable.
- `__post_init__` 에서 list → tuple 변환. tuple 은 immutable.
- 또는 setattr 을 막는 sentinel.

학습 단계라 ShopTracker 는 둠. *팀 컨벤션* 으로 직접 변경 금지하는 것도 한 방법 (lint rule).

### 4.3 상태 전이 시 부수효과

`mark_paid` 호출 시 "결제 완료 알림 메일" 도 보내야 한다면? :

옵션 1 — 도메인이 직접 호출 : 인프라 의존 → hexagonal 깨짐.
옵션 2 — **도메인 이벤트 기록** : `order._domain_events.append(OrderPaidEvent(...))` → 저장 후 dispatcher 가 event_bus 에 publish.
옵션 3 — Application handler 가 도메인 메서드 호출 후 직접 publish — ShopTracker 의 현재 방식.

ShopTracker 는 단순함을 위해 옵션 3 채택. 도메인 이벤트 패턴이 더 정통이지만 boilerplate 가 늘어남.

### 4.4 동시성

같은 Order 에 동시에 두 요청이 `mark_paid` 호출 → 첫 호출 성공, 두 번째는 `PAID → PAID` 시도 → `InvalidStatusTransition`. 이게 *바람직한 멱등성 부재* 인가?

대응 :
- DB 레벨에 optimistic lock (version 컬럼) → 두 번째 update 가 실패.
- 또는 `mark_paid` 가 이미 PAID 면 silently no-op (멱등) — 단, 의도적 결정.

ShopTracker 는 strict 한 상태머신 채택 (이미 PAID 면 예외). 사가에서 at-least-once 와 결합되는 부분은 §06 의 §4.3 참조.

### 4.5 Factory 가 기형적으로 커짐

검증이 100 줄 넘어가면 Factory 가 비대. 분리 신호 :
- **Specification 패턴** : 검증을 별도 객체로.
- **Builder 패턴** : 단계적 생성 (체이닝).
- 또는 도메인을 더 잘게 쪼갤 신호.

### 4.6 상태 + 데이터의 부분 결합

상태가 `PAID` 일 때만 `paid_at: datetime` 이 의미 있고, 그 외에는 None. dataclass 에 `paid_at: datetime | None` 으로 두면 invariant 흐림.

해결 : **State 패턴** (각 상태가 다른 클래스). 단 무겁다. ShopTracker 는 Optional 로 단순화.

---

## §5 다른 언어와 비교

### 5.1 Java + Spring + JPA

```java
@Entity
public class Order {
    @Id private UUID id;
    @Enumerated(EnumType.STRING) private OrderStatus status;

    public void markPaid() {
        if (!status.canTransitionTo(OrderStatus.PAID)) {
            throw new InvalidStatusTransition(status, OrderStatus.PAID);
        }
        this.status = OrderStatus.PAID;
        this.updatedAt = Instant.now();
    }
}
```

같은 패턴. Java 는 `private` 이 진짜로 막아주므로 캡슐화가 더 강함. Python 은 _underscore = 컨벤션.

### 5.2 Rust

```rust
enum OrderStatus { Created, PaymentPending, Paid, ... }

struct Order { status: OrderStatus, ... }

impl Order {
    pub fn mark_paid(&mut self) -> Result<(), InvalidTransition> {
        match self.status {
            OrderStatus::PaymentPending => { self.status = OrderStatus::Paid; Ok(()) }
            _ => Err(InvalidTransition),
        }
    }
}
```

Rust 의 `match` 가 모든 enum 값을 망라 검증 (compiler error if not exhaustive) → 매핑 누락 컴파일 시 잡힘. **Python 의 dict-based 보다 강함**.

### 5.3 TypeScript — Discriminated Union

```ts
type OrderState =
  | { kind: "created"; createdAt: Date }
  | { kind: "paid"; paidAt: Date }
  | { kind: "shipped"; trackingNumber: string };
```

상태 + 그 상태에서만 의미 있는 데이터를 type-level 로 묶음. PAID 일 때만 `paidAt` 이 존재. *부분 결합 문제* 를 우아하게 해결.

Python 은 이런 type-level 표현이 약함 (PEP 695 의 `type` 문법으로 일부 가능하지만 mypy 지원 제한적).

### 5.4 XState (TS / JS 생태계)

상태머신 라이브러리. 상태/이벤트/전이를 객체로 표현. 시각화 도구 제공.

Python 은 `transitions` 라이브러리가 비슷. ShopTracker 는 직접 dict 로 구현 — 학습 목적.

---

## §6 테스트 전략

### 6.1 모든 전이 검증

```python
@pytest.mark.parametrize("from_state,to_state,expected", [
    (CREATED, PAYMENT_PENDING, True),
    (CREATED, CANCELLED, True),
    (CREATED, PAID, False),
    (PAID, CANCELLED, False),    # ★ 핵심 invariant
    (PAID, SHIPPING, True),
    (DELIVERED, CANCELLED, False),
])
def test_transitions(from_state, to_state, expected):
    assert from_state.can_transition_to(to_state) == expected
```

### 6.2 Factory 검증

```python
def test_create_rejects_empty_customer():
    with pytest.raises(InvalidOrderError):
        Order.create("", [item()])

def test_create_rejects_zero_quantity():
    bad = OrderItem("a", 0, Money(Decimal("100")))
    with pytest.raises(InvalidOrderError):
        Order.create("alice", [bad])

def test_create_calculates_total():
    items = [
        OrderItem("a", 2, Money(Decimal("100"))),  # 200
        OrderItem("b", 1, Money(Decimal("50"))),   # 50
    ]
    o = Order.create("alice", items)
    assert o.total_amount.amount == Decimal("250")
    assert o.status == OrderStatus.CREATED
```

### 6.3 도메인 메서드

```python
def test_cannot_cancel_after_paid():
    o = Order.create(...)
    o.mark_payment_pending()
    o.mark_paid()
    with pytest.raises(InvalidStatusTransition):
        o.cancel()           # ★ 핵심 invariant 의 코드 표현
```

이 테스트가 *비즈니스 규칙의 명세서* 역할. 테스트 이름이 곧 규칙.

---

## §10 학습 포인트 (한 줄 요약)

1. **Anemic Model 회피** : 도메인 객체가 자기 invariant 를 자기가 지킨다.
2. **Aggregate Root** : 외부는 root 를 통해서만 내부 entity 변경. 트랜잭션의 단위.
3. **Factory method** (`Order.create`) : 새 생성용. `__init__` 은 DB 복원용.
4. **상태 머신** : 상태 + 허용된 전이를 명시. dict + Enum 으로 단순 표현.
5. **`_transition_to` private** : 모든 전이의 단일 게이트. DRY + 검증 누락 방지.
6. **의도가 드러나는 메서드** (`mark_paid`, `cancel`) — setter 가 아님.
7. **Tell, Don't Ask** : `order.status = "paid"` 가 아니라 `order.mark_paid()`.
8. **Enum 매핑 누락은 테스트로 잡기** — Python 은 컴파일러가 못 잡음.
9. **부수효과는 도메인 이벤트로** — 도메인이 인프라를 직접 호출 X.
10. **테스트가 명세** : `test_cannot_cancel_after_paid` 같은 이름 자체가 규칙 문서.

---

## 추가 참고

- Eric Evans, *Domain-Driven Design*, Aggregate / Factory 챕터
- Martin Fowler, *Anemic Domain Model*, https://martinfowler.com/bliki/AnemicDomainModel.html
- Vaughn Vernon, *Implementing DDD*, Aggregate 설계 챕터
- ShopTracker 다음 글 : `09-value-object-money.md` (Money 의 불변성)
