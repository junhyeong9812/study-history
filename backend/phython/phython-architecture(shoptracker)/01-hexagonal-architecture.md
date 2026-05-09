# 01. Hexagonal Architecture (Ports & Adapters)

> 이 문서가 다루는 것: ShopTracker 의 4-layer 구조 (domain ← application ← infrastructure ← presentation), port/adapter 의 본질, 도메인이 프레임워크를 모르는 의미.
> 전제: Python 기본.

---

## §0. 왜 헥사고날?

### 0.1 layered architecture 의 한계

전통적 layered:
```
Controller → Service → DAO → DB
```

**문제**:
- Service 가 DAO (`SQLAlchemy`) 직접 의존.
- 도메인 객체가 ORM 어노테이션으로 오염 (`Column`, `relationship`).
- DB 교체 (PostgreSQL → MongoDB) → 도메인 코드도 수정.
- 테스트 시 DB 띄워야 함.

### 0.2 Cockburn 의 헥사고날 (2005)

```
   ┌─────────────────────────────────┐
   │           Application           │
   │  ┌───────────────────────────┐  │
   │  │       Domain (core)        │  │
   │  │   - Entity                 │  │
   │  │   - Domain Logic           │  │
   │  │                            │  │
   │  │  inbound port ←──┐  ┌──→ outbound port
   │  └──────────────────┼──┼─────┘  │
   │                     │  │        │
   │  inbound adapter ←──┘  └──→ outbound adapter
   │  (FastAPI router)         (SQLAlchemy Repository, Event Bus impl)
   └─────────────────────────────────┘
```

**원칙**:
1. domain 은 외부 모름 (FastAPI/SQLAlchemy import 0).
2. **port** = interface (Python: `Protocol`).
3. **adapter** = 구현 (infrastructure layer).
4. 의존 방향 = 모든 화살표가 domain 가리킴.

### 0.3 ShopTracker 의 layering

각 도메인 모듈 (`orders`, `subscriptions`):
```
presentation/    ← inbound adapter (FastAPI router, Pydantic schemas)
application/     ← orchestration (handler 가 도메인 + repository 조립)
  commands.py    — input DTO
  queries.py
  command_handlers.py / query_handlers.py
domain/          ← Entity + Value Object + Protocol + 도메인 예외
  entities.py    — Order, OrderItem (Aggregate Root)
  value_objects.py — OrderStatus
  interfaces.py  — OrderRepositoryProtocol (port)
  exceptions.py
infrastructure/  ← outbound adapter (SQLAlchemy)
  models.py      — ORM Model (DB 표현)
  mappers.py     — Entity ↔ Model 변환
  repository.py  — Protocol 의 구현 (Adapter)
```

---

## §1. 본질적 원칙 — domain 이 외부 모름

### 1.1 domain entity 의 import
```python
# src/app/orders/domain/entities.py
from dataclasses import dataclass
from datetime import datetime, UTC
from uuid import UUID, uuid4

from app.shared.value_objects import Money
from app.orders.domain.value_objects import OrderStatus
from app.orders.domain.exceptions import InvalidOrderError, InvalidStatusTransition
```

→ 모두 **stdlib 또는 같은 도메인 모듈**. FastAPI/SQLAlchemy/Pydantic 0.

이게 헥사고날의 시각적 검증.

### 1.2 왜 중요?

**naive (도메인이 SQLAlchemy 의존)**:
```python
# ❌ Order entity 가 ORM Model 이기도 함
class Order(Base):
    __tablename__ = "orders"
    id = Column(UUID, primary_key=True)
    status = Column(String)
```

문제:
- Order 인스턴스 생성 시 SQLAlchemy session 필요.
- 단위 테스트 = DB 띄워야.
- Order 의 비즈니스 룰 + DB 매핑이 한 클래스에 섞임.
- DB 교체 시 도메인 코드도 수정.

**ShopTracker (분리)**:
```python
# domain/entities.py — 순수 데이터 + 행동
@dataclass
class Order:
    id: UUID
    customer_name: str
    items: list[OrderItem]
    status: OrderStatus
    # ...
    @classmethod
    def create(cls, ...): ...
    def mark_paid(self): ...

# infrastructure/models.py — ORM Model (별개)
class OrderModel(Base):
    __tablename__ = "orders"
    id = Column(String, primary_key=True)
    # ...

# infrastructure/mappers.py — 변환
def order_to_model(order: Order) -> OrderModel: ...
def model_to_order(model: OrderModel) -> Order: ...
```

→ Order 단위 테스트 시 DB 0. SQLAlchemy 교체 시 mappers + repository 만 변경.

---

## §2. Port = `Protocol`

### 2.1 코드
```python
# src/app/orders/domain/interfaces.py
from typing import Protocol
from uuid import UUID
from app.orders.domain.entities import Order

class OrderRepositoryProtocol(Protocol):
    """쓰기용 Repository (Output Port).

    domain 이 외부 (DB) 에 요구하는 인터페이스.
    실제 구현은 infrastructure 의 SQLAlchemyOrderRepository.
    """
    async def save(self, order: Order) -> None: ...
    async def find_by_id(self, order_id: UUID) -> Order | None: ...
    async def update(self, order: Order) -> None: ...

class OrderReadRepositoryProtocol(Protocol):
    """읽기용 Repository (CQRS 약식)."""
    async def find_by_id(self, order_id: UUID) -> Order | None: ...
    async def list_orders(self, customer_name: str | None = None, ...) -> list[Order]: ...
    async def count_orders(self, ...) -> int: ...
```

### 2.2 핵심 통찰
**Protocol = Java interface 의 Python 버전. but `implements` 키워드 X (duck typing)**.

같은 메서드 시그니처 가지면 자동 충족. Java 면 `class X implements OrderRepositoryProtocol` 명시 필요. Python 은 그냥 메서드 매칭.

→ 02-dip-protocols.md 에서 깊이 다룸.

### 2.3 왜 Read/Write 분리?
- Command handler (`CreateOrderHandler`) 는 `OrderRepositoryProtocol` 만 의존 (save, find, update).
- Query handler (`GetOrderHandler`) 는 `OrderReadRepositoryProtocol` 만 (find, list, count).
- Phase 1 은 같은 구현체가 둘 다 충족 → CQRS 약식.
- 나중에 read DB 분리 가능 (구현 교체).

---

## §3. Adapter = 구현체

### 3.1 코드
```python
# src/app/orders/infrastructure/repository.py
from sqlalchemy.ext.asyncio import AsyncSession

class SQLAlchemyOrderRepository:
    """OrderRepositoryProtocol + OrderReadRepositoryProtocol 동시 충족."""

    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def save(self, order: Order) -> None:
        model = order_to_model(order)        # Entity → ORM Model
        self._session.add(model)
        await self._session.flush()

    async def find_by_id(self, order_id: UUID) -> Order | None:
        model = await self._session.get(OrderModel, str(order_id))
        if model is None:
            return None
        return model_to_order(model)         # ORM Model → Entity
    # ... list_orders, count_orders, update
```

### 3.2 핵심
- SQLAlchemy 의존이 여기에 격리.
- domain 의 `Order` 는 그대로.
- `mapper` 가 두 모델 간 변환 책임 (10-mapper-orm-domain.md).

### 3.3 explicit `implements` 없음
```python
class SQLAlchemyOrderRepository:    # ← Protocol 명시 X
    async def save(...): ...
```

Python 은 duck typing — 메서드 시그니처가 맞으면 자동 충족. mypy/pyright 가 정적 검증.

---

## §4. application layer — orchestration

### 4.1 코드
```python
# src/app/orders/application/command_handlers.py
class CreateOrderHandler:
    def __init__(self, repo: OrderRepositoryProtocol, event_bus: EventBus) -> None:
        self._repo = repo                    # Protocol 의존
        self._event_bus = event_bus          # 또 다른 Protocol

    async def handle(self, command: CreateOrderCommand) -> UUID:
        # 1. command DTO → 도메인 객체
        items = [OrderItem(...) for dto in command.items]

        # 2. 도메인 entity 생성 (검증은 entity 가)
        order = Order.create(customer_name=command.customer_name, items=items)
        order.mark_payment_pending()

        # 3. 영속
        await self._repo.save(order)

        # 4. 이벤트 발행
        await self._event_bus.publish(OrderCreatedEvent(...))
        return order.id
```

### 4.2 핵심
- application = "유즈케이스" 단위 orchestration.
- domain entity 의 행동 호출 + repository 호출 + event 발행.
- 도메인 룰 자체는 entity 가 강제 (`Order.create()` 가 검증).

---

## §5. presentation = inbound adapter

### 5.1 코드 (FastAPI router)
```python
# src/app/orders/presentation/router.py
from fastapi import APIRouter, Depends
from dishka.integrations.fastapi import FromDishka

router = APIRouter(prefix="/orders", tags=["orders"])

@router.post("/", status_code=201)
async def create_order(
    request: CreateOrderRequest,            # Pydantic schema
    handler: FromDishka[CreateOrderHandler],
) -> CreateOrderResponse:
    command = CreateOrderCommand(           # Pydantic → application DTO
        customer_name=request.customer_name,
        items=[...],
    )
    order_id = await handler.handle(command)
    return CreateOrderResponse(order_id=order_id)
```

### 5.2 핵심
- FastAPI/Pydantic 의존이 여기에 격리.
- application 의 handler 호출만.
- request DTO (Pydantic) → command DTO (dataclass) 변환.

→ FastAPI 교체 (예: Litestar) 시 router/schemas 만 변경.

---

## §6. ShopTracker 의 디렉토리 구조 의미

```
src/app/orders/
├── domain/                    ★ 핵심: 외부 의존 0
│   ├── entities.py             — Order, OrderItem
│   ├── value_objects.py        — OrderStatus (enum + state machine)
│   ├── exceptions.py           — InvalidOrderError, InvalidStatusTransition
│   └── interfaces.py           — Protocol (port)
├── application/               유즈케이스
│   ├── commands.py             — CreateOrderCommand (input DTO)
│   ├── queries.py              — ListOrdersQuery
│   ├── command_handlers.py     — CreateOrderHandler.handle()
│   ├── query_handlers.py
│   └── event_handlers.py       — 다른 모듈의 이벤트 수신
├── infrastructure/            외부 시스템 어댑터
│   ├── models.py               — ORM Model (SQLAlchemy)
│   ├── mappers.py              — Entity ↔ Model 변환
│   └── repository.py           — Protocol 구현
└── presentation/              HTTP 표면
    ├── router.py               — FastAPI APIRouter
    └── schemas.py              — Pydantic Request/Response
```

**의존 방향**:
- `presentation → application → domain`
- `infrastructure → domain` (Protocol 충족)
- `domain` 은 어디도 의존 X.

---

## §7. 테스트 친화성

### 7.1 domain unit test (DB 0)
```python
# tests/unit/test_order_entity.py
def test_order_create_with_valid_items():
    items = [OrderItem("상품A", quantity=2, unit_price=Money(Decimal("10000")))]
    order = Order.create(customer_name="홍길동", items=items)
    assert order.status == OrderStatus.CREATED
    assert order.total_amount == Money(Decimal("20000"))

def test_order_create_rejects_empty_items():
    with pytest.raises(InvalidOrderError):
        Order.create(customer_name="홍길동", items=[])
```

→ DB 없음. SQLAlchemy 없음. **순수 Python**.

### 7.2 application test (Fake Repository)
```python
class FakeOrderRepository:
    """Protocol 의 in-memory 구현. 테스트 전용."""
    def __init__(self):
        self._store: dict[UUID, Order] = {}

    async def save(self, order: Order) -> None:
        self._store[order.id] = order

    async def find_by_id(self, order_id: UUID) -> Order | None:
        return self._store.get(order_id)

    async def update(self, order: Order) -> None:
        self._store[order.id] = order

# 테스트
async def test_create_order_publishes_event():
    repo = FakeOrderRepository()
    bus = FakeEventBus()
    handler = CreateOrderHandler(repo, bus)
    # ...
    assert isinstance(bus.events[0], OrderCreatedEvent)
```

→ DB 없이 application logic 검증.

### 7.3 integration test (TestClient + 진짜 DB)
```python
# tests/integration/test_order_flow.py
async def test_create_order_endpoint(client: AsyncClient):
    response = await client.post("/orders/", json={...})
    assert response.status_code == 201
```

---

## §8. 직접 구현 (다른 언어 비교)

### 8.1 Spring (Java) 의 헥사고날
```java
// domain/OrderRepository.java
public interface OrderRepository {
    void save(Order order);
}

// infrastructure/JpaOrderRepository.java
@Component
class JpaOrderRepository implements OrderRepository { ... }
```

→ explicit `implements`. ShopTracker 는 Protocol 자동 매칭.

### 8.2 Clean Architecture (Uncle Bob)
- Entities (innermost)
- Use Cases
- Interface Adapters
- Frameworks & Drivers (outermost)

→ ShopTracker 와 본질 같음 (4 layer).

---

## §9. 함정

### 9.1 domain 에 Pydantic 사용
- Pydantic = framework. 도메인 X.
- ShopTracker 는 dataclass.

### 9.2 mapper 안 두고 ORM 자체를 도메인으로
- 단순 prototype 유혹. 장기에 도메인 결합.

### 9.3 application 이 너무 두꺼움
- 비즈니스 룰이 application 으로 새는 안티패턴.
- 룰은 domain entity 안에. application 은 조립만.

---

## §10. 학습 포인트

1. **domain 이 외부 모름** — FastAPI/SQLAlchemy import 0.
2. **port = Protocol, adapter = infrastructure 구현**.
3. **의존 방향 단방향** — 모두 domain 가리킴.
4. **Read/Write Protocol 분리** = CQRS 의 작은 형태.
5. **mapper 가 entity ↔ ORM 변환** — 도메인 격리의 핵심.
6. **application = orchestration** — 도메인 + repo + event 조립.
7. **presentation = inbound adapter** — FastAPI 격리.
8. **domain unit test = 순수 Python** — DB 0.
9. **Fake Repository 로 application 테스트** — Protocol 의 진가.
10. **Python `Protocol` = 자동 매칭** — Java 의 explicit implements 다름.

### 추가 참고
- Alistair Cockburn, *Hexagonal Architecture* (2005).
- Robert C. Martin, *Clean Architecture* (2017).
- 다음 doc: [02-dip-protocols.md](02-dip-protocols.md) — Protocol 깊이.
