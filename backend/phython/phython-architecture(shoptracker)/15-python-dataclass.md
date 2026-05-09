# 15 — Python dataclass 의 모든 것

> ShopTracker 의 모든 도메인 / 이벤트 / DTO / Command / Query 가 `@dataclass` 다. 30 줄짜리 클래스가 5 줄로 줄고, `__init__` / `__repr__` / `__eq__` 의 boilerplate 가 자동. 이 글은 dataclass 의 옵션과 함정을 정독.

---

## §0 동기 — Boilerplate 의 종말

```python
# pre-dataclass (verbose)
class Order:
    def __init__(self, id, customer_name, items, status, ...):
        self.id = id
        self.customer_name = customer_name
        ...
    def __repr__(self):
        return f"Order(id={self.id}, ...)"
    def __eq__(self, other):
        if not isinstance(other, Order): return False
        return (self.id, ...) == (other.id, ...)
    def __hash__(self):
        return hash((self.id, ...))
```

```python
# with dataclass
@dataclass
class Order:
    id: UUID
    customer_name: str
    items: list[OrderItem]
    status: OrderStatus
```

`__init__`, `__repr__`, `__eq__` 모두 자동.

---

## §1 기본 사용

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int

p = Point(1, 2)
print(p)             # Point(x=1, y=2)
p == Point(1, 2)     # True
```

PEP 557 (Python 3.7) 부터 표준 라이브러리.

---

## §2 옵션

### 2.1 frozen=True — 불변

```python
@dataclass(frozen=True)
class Money:
    amount: Decimal
    currency: str

m = Money(Decimal("100"), "KRW")
m.amount = Decimal("200")    # FrozenInstanceError
```

- `__setattr__` 을 막아 불변성 강제.
- `__hash__` 자동 생성 (기본 dataclass 는 `eq=True` 면 hash 가 None 됨).
- ShopTracker : Money, 모든 이벤트, Command/Query DTO 가 frozen.

### 2.2 eq=True (default) — 동등성

기본으로 `__eq__` 자동. False 로 두면 id 비교 (object 기본).

### 2.3 order=True — 비교 가능

```python
@dataclass(order=True)
class Version:
    major: int
    minor: int
    patch: int

Version(1, 2, 3) < Version(1, 2, 4)   # True
```

`__lt__`, `__le__`, `__gt__`, `__ge__` 자동.

### 2.4 init=False — 자동 init 비활성

```python
@dataclass(init=False)
class Foo:
    x: int
    def __init__(self, raw: str):     # 직접 작성
        self.x = int(raw)
```

### 2.5 repr=False — 자동 repr 비활성

비밀 정보가 있으면 직접 작성.

### 2.6 slots=True (3.10+)

```python
@dataclass(slots=True)
class Point:
    x: int
    y: int
```

- `__slots__ = ("x", "y")` 자동.
- 메모리 ↓, 속성 추가 불가.
- ShopTracker 의 `Money` 는 dataclass 가 아니지만 같은 효과를 위해 `__slots__` 직접.

### 2.7 kw_only=True (3.10+)

```python
@dataclass(kw_only=True)
class Order:
    id: UUID
    customer_name: str

Order(id=..., customer_name="alice")    # OK
Order(uuid4(), "alice")                  # TypeError - keyword 만
```

순서 의존 버그 차단. 큰 dataclass 에 추천.

---

## §3 default / default_factory

```python
@dataclass
class Foo:
    x: int = 0                              # 단순 default
    items: list[int] = field(default_factory=list)   # mutable
```

- **mutable default 직접 금지** :
  ```python
  @dataclass
  class Bad:
      items: list[int] = []   # ValueError - mutable default 금지
  ```
- `field(default_factory=list)` 로 매 인스턴스마다 새 list.

---

## §4 field — 더 정교한 제어

```python
from dataclasses import field

@dataclass
class Foo:
    public: int = 0
    secret: str = field(default="", repr=False)        # repr 에 안 보임
    cache: dict = field(default_factory=dict, compare=False)  # eq 비교 제외
    extra: int = field(init=False, default=0)          # init 인자 안 받음
```

- `repr=False` : `__repr__` 에 제외.
- `compare=False` : `__eq__` 에 제외.
- `init=False` : 생성자 인자에서 제외.
- `metadata={...}` : 외부 도구용 (예 : marshmallow 가 사용).

---

## §5 `__post_init__`

```python
@dataclass
class Email:
    value: str

    def __post_init__(self):
        if "@" not in self.value:
            raise ValueError("invalid email")
```

- `__init__` 후 자동 호출.
- 검증 / 파생 필드 계산에 사용.
- frozen 일 때는 `object.__setattr__(self, "field", value)` 로 우회.

```python
@dataclass(frozen=True)
class Coord:
    x: int
    y: int
    distance: float = field(init=False)

    def __post_init__(self):
        object.__setattr__(self, "distance", (self.x**2 + self.y**2) ** 0.5)
```

---

## §6 ShopTracker 코드 분석

### 6.1 도메인 entity

```python
@dataclass
class Order:
    id: UUID
    customer_name: str
    items: list[OrderItem]
    status: OrderStatus
    total_amount: Money
    created_at: datetime
    updated_at: datetime
```

- 평범한 mutable dataclass — `mark_paid` 가 status 를 수정해야 하므로.
- 단, `items` 가 mutable list — 외부가 `order.items.append(...)` 가능. 학습 단순화.

### 6.2 이벤트

```python
@dataclass(frozen=True)
class OrderCreatedEvent:
    order_id: UUID
    customer_name: str
    total_amount: Decimal
    items_count: int
    timestamp: datetime
```

- `frozen=True` — 사실은 변하지 않음.
- 자동 `__eq__`, `__hash__` — set / dict 키로 사용 가능 (idempotency 체크 용).

### 6.3 Command / Query

```python
@dataclass(frozen=True)
class CreateOrderCommand:
    customer_name: str
    items: list[OrderItemDTO]
```

- frozen — 핸들러가 수정 못 하게.
- `items: list[...]` — frozen 이라도 *list 자체는* 여전히 mutable. 진정한 불변은 `tuple` 이지만 ShopTracker 는 list 채택 (간결).

### 6.4 Subscription Context

```python
@dataclass(frozen=True)
class SubscriptionContext:
    customer_name: str
    tier: str
    is_active: bool

    @classmethod
    def guest(cls, customer_name: str = "guest") -> "SubscriptionContext":
        return cls(customer_name=customer_name, tier="none", is_active=False)
```

- frozen + classmethod factory (Null Object 패턴, 07 장).

---

## §7 함정

### 7.1 frozen 의 미묘한 한계

```python
@dataclass(frozen=True)
class Foo:
    items: list[int]

f = Foo([1, 2])
f.items = [3, 4]            # FrozenInstanceError
f.items.append(3)           # ✓ 작동! list 자체는 mutable
```

→ 진정한 불변은 `tuple`, `frozenset`, immutable 라이브러리.

### 7.2 mutable default 잊기

```python
@dataclass
class Foo:
    items: list = []        # ValueError at class definition
    items: list = field(default_factory=list)  # OK
```

dataclass 는 이걸 *컴파일 시 에러* 로 만들어 줌. 다행.

### 7.3 상속 시 default 문제

```python
@dataclass
class Base:
    x: int = 0

@dataclass
class Child(Base):
    y: int           # ← TypeError: 부모에 default 가 있는 필드 다음에 default 없는 필드 못 옴
```

해결 :
- `kw_only=True` (3.10+) — keyword 인자라 순서 무관.
- 또는 child 도 default 갖게.

### 7.4 dataclass + ABC

```python
@dataclass
class Foo(ABC):
    x: int
    @abstractmethod
    def f(self): ...

Foo(1)              # 인스턴스화 가능 (abstract 무시)
```

→ `__init_subclass__` 트릭 또는 직접 가드.

### 7.5 dataclass 와 Pydantic

Pydantic v2 의 `dataclasses.dataclass` 통합 :

```python
from pydantic.dataclasses import dataclass as pyd_dataclass

@pyd_dataclass
class Foo:
    x: int
```

런타임 검증까지 — `Foo("not int")` → ValidationError. 표준 dataclass 는 안 잡음.

ShopTracker 는 표준 dataclass — 도메인은 *프레임워크 무관* 원칙.

### 7.6 hash 의 함정

```python
@dataclass
class Foo:
    x: list

set([Foo([1, 2])])      # TypeError - unhashable
```

list 필드가 있으면 hash 불가. set 키로 쓰려면 tuple 또는 hashable 필드만.

---

## §8 dataclass 대안

| 도구 | 특징 |
|---|---|
| `@dataclass` | 표준, 무프레임워크 |
| `attrs` | dataclass 의 모태. 더 풍부한 기능 (validators, converters). |
| `pydantic.BaseModel` | 검증 + 직렬화 + JSON Schema. HTTP layer 에 |
| `msgspec.Struct` | 매우 빠른 직렬화 + 타입 검증 |
| `NamedTuple` | tuple 기반. 작고 불변. 메서드 정의 가능. |
| `TypedDict` | dict 기반. 타입 힌트만, 런타임 검증 X. |

ShopTracker 는 도메인 = `@dataclass`, HTTP = `pydantic.BaseModel`. 분리.

---

## §10 학습 포인트 (한 줄 요약)

1. **dataclass 의 본질** : `__init__` / `__repr__` / `__eq__` 자동.
2. **frozen=True** = 불변. ShopTracker 의 이벤트 / Command / VO.
3. **`field(default_factory=list)`** — mutable default 의 정답.
4. **`__post_init__`** — 검증 / 파생 필드.
5. **slots=True (3.10+)** — 메모리 + 속성 차단 자동.
6. **kw_only=True (3.10+)** — 순서 의존 버그 차단.
7. **frozen 도 list 자체는 mutable** — 진정한 불변은 tuple.
8. **상속 시 default 순서** — kw_only 로 해결.
9. **도메인은 표준 dataclass**, **HTTP 는 Pydantic** — 경계 분리.
10. **dataclass 는 클래스 단위 boilerplate 제거기** — 작은 클래스 많이 만드는 도메인 코드와 짝.

---

## 추가 참고

- PEP 557 : https://peps.python.org/pep-0557/
- attrs : https://www.attrs.org/
- ShopTracker 다음 글 : `16-python-async-await.md`
