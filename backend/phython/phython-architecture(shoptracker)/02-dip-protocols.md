# 02. DIP + Python Protocol

> 이 문서가 다루는 것: Dependency Inversion Principle, Python `Protocol` 의 본질, structural typing vs nominal typing, ABC 와 비교.

---

## §0. DIP 가 무엇인가?

### 0.1 SOLID 의 D

**Dependency Inversion Principle**:
> 상위 모듈이 하위 모듈에 의존하면 안 된다. 둘 다 추상화에 의존해야 한다.
> 추상화는 구체에 의존하면 안 된다. 구체가 추상화에 의존해야 한다.

### 0.2 naive (DIP 위반)
```python
# command_handler.py
from infrastructure.repository import SQLAlchemyOrderRepository    # ❌ 구체 import

class CreateOrderHandler:
    def __init__(self):
        self._repo = SQLAlchemyOrderRepository(...)               # ❌ 직접 인스턴스화
```

**문제**:
- handler 가 SQLAlchemy 직접 의존.
- 테스트 시 SQLAlchemy mock 또는 진짜 DB 필요.
- DB 교체 시 handler 수정.

### 0.3 DIP 적용
```python
# command_handler.py
from app.orders.domain.interfaces import OrderRepositoryProtocol    # ✅ 추상화 import

class CreateOrderHandler:
    def __init__(self, repo: OrderRepositoryProtocol):              # ✅ Protocol 의존
        self._repo = repo
```

→ handler 가 Protocol 만 알음. 구체는 DI 컨테이너 (03 doc) 가 주입.

---

## §1. Python `Protocol` — structural typing

### 1.1 Java/C# 의 nominal typing
```java
// Java
interface OrderRepository { void save(Order o); }
class JpaOrderRepository implements OrderRepository { ... }    // 명시적 implements
```

→ `implements` 키워드 명시 필수. 컴파일러가 검증.

### 1.2 Python `Protocol` = structural typing
```python
from typing import Protocol

class OrderRepositoryProtocol(Protocol):
    async def save(self, order: Order) -> None: ...

# 어디에도 "implements" 명시 X
class SQLAlchemyOrderRepository:
    async def save(self, order: Order) -> None:
        # ...
```

→ `SQLAlchemyOrderRepository` 가 같은 시그니처의 `save` 메서드 가짐 → **자동으로 Protocol 충족**.

→ static type checker (mypy/pyright) 가 정적 검증.

### 1.3 본질
**"같은 모양이면 같은 타입"** = duck typing 의 type-safe 버전.

> "If it walks like a duck and quacks like a duck, it's a duck."

Python 의 일반 duck typing 은 runtime 확인. Protocol 은 **compile time (정적 분석)** 확인.

### 1.4 ShopTracker 의 Protocol 들

```python
# orders/domain/interfaces.py
class OrderRepositoryProtocol(Protocol):
    async def save(self, order: Order) -> None: ...
    async def find_by_id(self, order_id: UUID) -> Order | None: ...
    async def update(self, order: Order) -> None: ...

class OrderReadRepositoryProtocol(Protocol):
    async def find_by_id(self, order_id: UUID) -> Order | None: ...
    async def list_orders(self, ...) -> list[Order]: ...
    async def count_orders(self, ...) -> int: ...

# shared/event_bus.py
class EventBus(Protocol):
    async def publish(self, event: object) -> None: ...
    def subscribe(self, event_type: type, handler: Callable) -> None: ...
```

→ 모두 `Protocol` 상속. `implements` 키워드 X. 구현체가 자동 충족.

---

## §2. ABC vs Protocol

### 2.1 ABC (Abstract Base Class) — Python 의 nominal typing
```python
from abc import ABC, abstractmethod

class OrderRepositoryABC(ABC):
    @abstractmethod
    async def save(self, order: Order) -> None: ...

class SQLAlchemyOrderRepository(OrderRepositoryABC):    # 명시 상속
    async def save(self, order: Order) -> None:
        # ...
```

→ Java interface 비슷. 명시 상속 + abstract method 강제.

### 2.2 Protocol vs ABC

| | Protocol | ABC |
|---|---|---|
| 명시 상속 | 불필요 | 필수 |
| 검증 시점 | static (mypy) | runtime (실수 시 TypeError) |
| duck typing | ✅ | ❌ |
| 다형성 | structural | nominal |
| 다중 충족 | 자유 | 다중 상속 가능 (복잡) |
| third-party 호환 | ✅ (소스 수정 X) | ❌ |

### 2.3 언제 ABC?
- runtime 에 "이게 정말 충족하나?" 검증 필요.
- 명시적 계약 강조 (Java 스타일).
- abstract method + concrete method 같이.

### 2.4 언제 Protocol?
- structural typing.
- third-party 클래스도 자동 충족.
- 가벼운 interface.
- ShopTracker 는 모두 Protocol.

---

## §3. Protocol 의 advanced features

### 3.1 `runtime_checkable`
```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class OrderRepositoryProtocol(Protocol):
    async def save(self, order: Order) -> None: ...

# 사용
isinstance(repo, OrderRepositoryProtocol)    # runtime check (느림)
```

→ runtime `isinstance` 가능. but **느림** + 메서드 시그니처 검증 X (이름만).

ShopTracker 는 미사용 — 정적 검증으로 충분.

### 3.2 Protocol 의 default 메서드
```python
class Greeter(Protocol):
    name: str

    def greet(self) -> str:
        return f"Hello, {self.name}"     # default implementation
```

→ Protocol 도 default 메서드 가능. 충족 클래스가 override 가능.

### 3.3 generic Protocol
```python
from typing import Protocol, TypeVar

T = TypeVar("T")

class Repository(Protocol[T]):
    async def save(self, entity: T) -> None: ...
    async def find_by_id(self, id: UUID) -> T | None: ...
```

→ `Repository[Order]`, `Repository[Subscription]` 등 generic.

ShopTracker 는 도메인별로 Protocol 분리 (`OrderRepositoryProtocol`, `SubscriptionRepositoryProtocol`) — generic 안 씀.

### 3.4 callable Protocol
```python
class Logger(Protocol):
    def __call__(self, message: str) -> None: ...

def use_logger(log: Logger):
    log("hello")    # log() 호출
```

→ callable 자체가 Protocol.

---

## §4. ShopTracker 의 DIP 적용 흐름

### 4.1 Handler → Protocol → Adapter

```
[CreateOrderHandler]
   │  의존
   ▼
[OrderRepositoryProtocol]  ← domain layer
   │  자동 충족
   ▼
[SQLAlchemyOrderRepository]  ← infrastructure layer
```

handler 코드:
```python
class CreateOrderHandler:
    def __init__(self, repo: OrderRepositoryProtocol, event_bus: EventBus):
        self._repo = repo                              # Protocol 의존
        self._event_bus = event_bus

    async def handle(self, command: CreateOrderCommand) -> UUID:
        # repo.save() 호출 — Protocol 의 시그니처만 안다
        # 실제로는 SQLAlchemyOrderRepository.save() 가 실행
        await self._repo.save(order)
```

### 4.2 DI 가 연결
```python
# di_container.py
class OrdersProvider(Provider):
    @provide(scope=Scope.REQUEST)
    def order_repository(self, session: AsyncSession) -> SQLAlchemyOrderRepository:
        return SQLAlchemyOrderRepository(session)        # 구체 인스턴스

    @provide(scope=Scope.REQUEST)
    def create_order_handler(
        self, repo: SQLAlchemyOrderRepository, event_bus: EventBus
    ) -> CreateOrderHandler:
        return CreateOrderHandler(repo, event_bus)        # Protocol 자리에 구체 주입
```

→ 03-di-and-dishka.md 에서 깊이.

### 4.3 테스트 — Fake 주입
```python
class FakeOrderRepository:
    """Protocol 충족하는 in-memory 구현."""
    def __init__(self):
        self._store = {}

    async def save(self, order: Order) -> None:
        self._store[order.id] = order

    async def find_by_id(self, order_id: UUID) -> Order | None:
        return self._store.get(order_id)

    async def update(self, order: Order) -> None:
        self._store[order.id] = order

# 테스트
async def test_create_order():
    repo = FakeOrderRepository()        # Protocol 충족 (메서드 시그니처 동일)
    bus = FakeEventBus()
    handler = CreateOrderHandler(repo, bus)
    # ...
```

→ Fake 가 SQLAlchemy 모름. handler 도 SQLAlchemy 모름. **테스트 격리**.

---

## §5. Protocol 의 함정

### 5.1 시그니처 mismatch
```python
class OrderRepositoryProtocol(Protocol):
    async def save(self, order: Order) -> None: ...

class WrongRepository:
    async def save(self, order_data: dict) -> None: ...    # ❌ 타입 다름
```

→ static type checker (mypy/pyright) 가 잡음. runtime 안 씀.

### 5.2 attribute Protocol
```python
class HasName(Protocol):
    name: str

class Animal:
    def __init__(self, name: str):
        self.name = name

# Animal 이 HasName 자동 충족
```

→ 메서드 외 attribute 도 Protocol 가능.

### 5.3 너무 큰 Protocol
```python
class GodRepository(Protocol):    # ❌ 책임 너무 많음
    async def save(...): ...
    async def find_by_id(...): ...
    async def list(...): ...
    async def search(...): ...
    async def export_csv(...): ...
    async def email_admin(...): ...
```

→ Interface Segregation Principle (ISP) 위반. 분리 권장.

ShopTracker 는 Read/Write 분리 (CQRS 약식).

### 5.4 mypy/pyright strict 활용
```toml
# pyproject.toml
[tool.mypy]
strict = true

[tool.pyright]
typeCheckingMode = "strict"
```

→ Protocol 충족 안 하면 즉시 에러.

---

## §6. 다른 언어 비교

### 6.1 Go interface
```go
type Repository interface {
    Save(ctx context.Context, order Order) error
}

// 구현 — implements 키워드 X
func (r *SqlRepository) Save(ctx context.Context, order Order) error { ... }
```

→ Python Protocol 과 거의 같음. Go 도 structural typing.

### 6.2 TypeScript interface
```typescript
interface Repository {
    save(order: Order): Promise<void>;
}

// 구현 — implements 옵션
class SqlRepository implements Repository { ... }
class DuckSqlRepository { save(...) { ... } }    // implements 없어도 OK
```

→ TypeScript 도 structural typing 기본 + 명시적 `implements` 옵션.

### 6.3 Java interface
```java
interface Repository {
    void save(Order order);
}

class JpaRepository implements Repository { ... }    // implements 필수
```

→ nominal. ShopTracker 의 Protocol 과 다름.

### 6.4 Rust trait
```rust
trait Repository {
    fn save(&self, order: Order);
}

impl Repository for SqlRepository { ... }            // explicit impl
```

→ Java 와 비슷한 nominal.

---

## §7. ShopTracker 가 사용한 모든 Protocol

| Protocol | 위치 | 충족 클래스 |
|---|---|---|
| `EventBus` | `shared/event_bus.py` | `InMemoryEventBus` |
| `OrderRepositoryProtocol` | `orders/domain/interfaces.py` | `SQLAlchemyOrderRepository` |
| `OrderReadRepositoryProtocol` | 동일 | 동일 |
| `SubscriptionRepositoryProtocol` | `subscriptions/domain/interfaces.py` | `SQLAlchemySubscriptionRepository` |

**v0.2 후보**:
- Phase 2: `PaymentGatewayProtocol` (Fake/Real Gateway 교체).
- Phase 3: `ShippingProviderProtocol`.
- Phase 4: `TrackingStoreProtocol`.

---

## §8. 학습 포인트

1. **DIP** — 추상화 의존, 구체 의존 X.
2. **Protocol = Python interface** — but `implements` 키워드 X.
3. **structural typing** — 같은 모양이면 같은 타입.
4. **mypy/pyright 가 정적 검증** — runtime 비용 0.
5. **ABC 는 nominal**, Protocol 은 structural.
6. **runtime_checkable** = isinstance 가능 (느림, 시그니처 X).
7. **generic Protocol** = `Protocol[T]`.
8. **Read/Write 분리 = ISP** — CQRS 약식.
9. **Fake repository = Protocol 의 진가** — 테스트 격리.
10. **함정** = 시그니처 mismatch (mypy 가 잡음), 너무 큰 Protocol (ISP 위반).

### 추가 참고
- PEP 544: https://peps.python.org/pep-0544/
- mypy Protocol: https://mypy.readthedocs.io/en/stable/protocols.html
- Robert C. Martin, *Clean Architecture* — DIP.
- 다음: [03-di-and-dishka.md](03-di-and-dishka.md) — DI 컨테이너로 연결.
