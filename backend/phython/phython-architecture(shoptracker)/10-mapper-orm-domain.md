# 10 — Mapper (Entity ↔ ORM Model 분리)

> 도메인의 `Order` 와 SQLAlchemy 의 `OrderModel` 은 *완전히 별개의 두 클래스* 다. 이 둘 사이에 `mappers.py` 가 변환을 담당한다. 왜 한 클래스로 안 합치고 *번거로운 변환 코드* 를 두는가? 이 글은 그 트레이드오프와 ShopTracker 의 매퍼 코드를 정독한다.

---

## §0 문제 정의 — Active Record vs Data Mapper

### 0.1 옵션 A — Active Record (Django ORM, Rails AR 스타일)

```python
class Order(Base):
    __tablename__ = "orders"
    id = Column(...)
    status = Column(...)

    def cancel(self):
        if self.status == "paid":
            raise ...
        self.status = "cancelled"
        self.save()
```

도메인 + ORM 한 클래스. **편하다**. 작은 프로젝트에서 빠름.

문제 :
- ORM (SQLAlchemy / Django) 에 도메인이 종속. 인프라 교체 불가.
- 테스트가 DB 의존 (도메인 메서드 검증에도 DB 필요).
- 컬럼 선언과 비즈니스 로직이 한 파일에 섞임.
- ORM 의 lazy loading / session 라이프사이클이 도메인 동작에 새 나옴.

### 0.2 옵션 B — Data Mapper (Hibernate, JPA, ShopTracker)

```
Order (domain)         OrderModel (ORM)
   ↑                       ↑
   └─── Mapper ────────────┘
   변환만 담당
```

도메인은 *어떤 DB 도 모름*. ORM 은 *비즈니스 로직 모름*. Mapper 가 둘 사이의 다리.

비용 :
- **boilerplate** — 매핑 코드를 손으로 짜야 함.
- 한 필드 추가 시 두 곳 (Entity + Model) + Mapper 까지 수정.
- 처음에 손이 많이 감.

이득 :
- 도메인 테스트가 DB 무관 (순수 단위 테스트).
- DB 스키마 변경이 도메인 로직 안 건드림.
- 도메인 객체에 ORM 어노테이션 / 메타데이터가 없음 — 깨끗.
- 다른 storage (Mongo, Redis, S3) 로의 교체가 *Mapper 만 갈아끼우면* 됨.

ShopTracker 는 학습 목적으로 옵션 B. 비용 알면서도 채택.

---

## §1 본질 메커니즘 — Hexagonal 의 분리 원칙

### 1.1 두 세계

| | Domain Entity | ORM Model |
|---|---|---|
| 위치 | `domain/entities.py` | `infrastructure/models.py` |
| 부모 클래스 | (없음, 순수 dataclass) | `Base` (SQLAlchemy declarative) |
| 메서드 | 비즈니스 (mark_paid, cancel) | (없음, 순수 데이터) |
| 필드 타입 | Money, OrderStatus, UUID | str, float, datetime |
| import | 표준 라이브러리만 | `sqlalchemy` |
| 변경 트리거 | 비즈니스 규칙 변화 | DB 스키마 변화 |

이 둘이 **서로 독립적으로 변할 수 있어야 한다** — 그게 분리의 본질.

### 1.2 Mapper 의 역할

```
order_to_model(order: Order) -> OrderModel    [저장 시]
model_to_order(model: OrderModel) -> Order    [조회 시]
```

추가로 :
- 타입 변환 (UUID ↔ str, Decimal ↔ float, Enum ↔ str, Money ↔ (amount, currency))
- 컬렉션 변환 (List[OrderItem] ↔ List[OrderItemModel])

---

## §2 ShopTracker 코드 정독

### 2.1 OrderModel — ORM 측

```python
# infrastructure/models.py
class OrderModel(Base):
    __tablename__ = "orders"

    id: Mapped[str] = mapped_column(String(36), primary_key=True, ...)
    customer_name: Mapped[str] = mapped_column(String(100))
    status: Mapped[str] = mapped_column(String(20), default="created")
    total_amount: Mapped[float] = mapped_column(Numeric(12, 2))
    currency: Mapped[str] = mapped_column(String(3), default="KRW")
    created_at: Mapped[datetime] = mapped_column(DateTime, default=...)
    updated_at: Mapped[datetime] = mapped_column(DateTime, default=...)

    items: Mapped[list["OrderItemModel"]] = relationship(
        back_populates="order", cascade="all, delete-orphan", lazy="selectin",
    )
```

특징 :

- **`Mapped[T]` + `mapped_column(...)`** : SQLAlchemy 2.0 의 typed declarative 스타일.
- **`String(36)`** : UUID 의 문자열 표현 길이. PostgreSQL 이라면 `UUID` 타입 사용 가능 (더 효율). SQLite 호환 위해 String.
- **`Numeric(12, 2)`** : 정밀 소수. 12 자리 중 소수점 2 자리. 금액 저장의 정석.
- **`status: str`** — Enum 이 아니라 string 으로 저장. DB 가 Enum 을 알 필요 없음. 마이그레이션 시 새 값 추가도 편함.
- **`items: relationship(...)`** :
  - `back_populates="order"` : 양방향 — `model.items` ↔ `item.order`.
  - `cascade="all, delete-orphan"` : Order 삭제 시 items 도 삭제. orphan = 부모를 잃은 자식 자동 정리.
  - `lazy="selectin"` : N+1 방지. Order 를 조회하면 자동으로 items 도 한 번에 로드 (`SELECT * FROM order_items WHERE order_id IN (...)`).

### 2.2 OrderItemModel

```python
class OrderItemModel(Base):
    __tablename__ = "order_items"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    order_id: Mapped[str] = mapped_column(String(36), ForeignKey("orders.id"), nullable=False)
    product_name: Mapped[str] = mapped_column(String(200))
    quantity: Mapped[int] = mapped_column(Integer)
    unit_price: Mapped[float] = mapped_column(Numeric(12, 2))
    currency: Mapped[str] = mapped_column(String(3), default="KRW")

    order: Mapped["OrderModel"] = relationship(back_populates="items")
```

- `id: int + autoincrement` : 도메인의 `OrderItem` 에는 *id 가 없음*. 도메인이 식별자 신경 안 쓰는 의도. DB 가 자기 PK 만 자동 부여.
- `order_id: ForeignKey("orders.id")` : 외래키. 부모 삭제 시 cascade.
- 도메인 `OrderItem` 에 `currency` 필드 직접 없음 (Money 안에 있음). 모델은 풀어서 저장.

### 2.3 order_to_model — 도메인 → ORM

```python
# infrastructure/mappers.py:22-41
def order_to_model(order: Order) -> OrderModel:
    return OrderModel(
        id=str(order.id),                                # UUID → str
        customer_name=order.customer_name,
        status=order.status.value,                       # Enum → str
        total_amount=float(order.total_amount.amount),   # Decimal → float
        currency=order.total_amount.currency,
        created_at=order.created_at,
        updated_at=order.updated_at,
        items=[
            OrderItemModel(
                product_name=item.product_name,
                quantity=item.quantity,
                unit_price=float(item.unit_price.amount),
                currency=item.unit_price.currency,
            )
            for item in order.items
        ],
    )
```

변환 포인트 :

| 도메인 | ORM | 변환 이유 |
|---|---|---|
| `UUID` | `str` | DB 호환성 (SQLite UUID 미지원) |
| `OrderStatus` (Enum) | `str` (`.value`) | DB Enum 의존 회피 |
| `Money(Decimal, currency)` | `float` + `str` | 두 컬럼으로 분해 |
| `list[OrderItem]` | `list[OrderItemModel]` | 각각 변환 |

> **`float(Decimal)` 의 위험** : 부동소수점 오차. KRW 100 처럼 정수면 OK, 소수점 있으면 `Numeric` 컬럼이라도 들어가는 과정에서 float 단계 거침. 이상적으로는 `Decimal` 그대로 SQLAlchemy 가 처리하게 (사실 SQLAlchemy 2.0 + Numeric 컬럼은 Decimal 그대로 전달 가능). ShopTracker 코드는 명시적으로 `float()` — 학습 단순화 + sqlite 호환.

### 2.4 model_to_order — ORM → 도메인 (복원)

```python
# mappers.py:44-63
def model_to_order(model: OrderModel) -> Order:
    items = [
        OrderItem(
            product_name=item.product_name,
            quantity=item.quantity,
            unit_price=Money(Decimal(str(item.unit_price)), item.currency),
        )
        for item in model.items
    ]
    return Order(
        id=UUID(model.id),                              # str → UUID
        customer_name=model.customer_name,
        items=items,
        status=OrderStatus(model.status),                # str → Enum
        total_amount=Money(Decimal(str(model.total_amount)), model.currency),
        created_at=model.created_at,
        updated_at=model.updated_at,
    )
```

핵심 :

- **`Order(...)` 직접 호출** — `Order.create(...)` 가 *아님*. Factory 의 검증을 우회. 이유 : 이미 *과거에 검증을 통과해 저장된 데이터*. 다시 검증하면 옛 데이터에 새 규칙을 적용해서 실패할 수도 있음 (08 장).
- **`Decimal(str(model.total_amount))`** : `Decimal(float)` 직접 변환은 부동소수점 오차를 그대로 끌고 옴. `str` 을 거치면 `Decimal("100.00")` 처럼 깨끗.
  ```python
  Decimal(0.1)        # Decimal('0.1000000000000000055511151231257827021181583404541015625')
  Decimal(str(0.1))   # Decimal('0.1')
  ```
- **`UUID(model.id)`** : str → UUID. 잘못된 형식이면 ValueError → 도메인이 복원 못 함 (DB 손상의 표지).
- **`OrderStatus(model.status)`** : str → Enum. DB 에 알 수 없는 값 ("foo") 이 있으면 ValueError. 마이그레이션 시 enum 값 변경 주의.

### 2.5 SQLAlchemyOrderRepository — Mapper 사용 흐름

```python
# infrastructure/repository.py
async def save(self, order: Order) -> None:
    model = order_to_model(order)
    self._session.add(model)
    await self._session.flush()

async def find_by_id(self, order_id: UUID) -> Order | None:
    model = await self._session.get(OrderModel, str(order_id))
    if model is None:
        return None
    return model_to_order(model)
```

Repository = Mapper + Session 작업.

- `save` : 도메인 → 모델 → session.add → flush.
- `find_by_id` : session.get → 모델 → 도메인 반환.
- `flush` vs `commit` : flush 는 SQL 발행만. commit 은 트랜잭션 종결. Repository 는 flush 까지, commit 은 **DI 의 session generator** (03 장) 가 관리.

### 2.6 update 의 미묘한 차이

```python
# repository.py:44-52
async def update(self, order: Order) -> None:
    model = await self._session.get(OrderModel, str(order.id))
    if model is None:
        return
    model.status = order.status.value
    model.total_amount = float(order.total_amount.amount)
    model.updated_at = order.updated_at
    await self._session.flush()
```

`save` 와 다르게 *기존 model 을 가져와 부분 갱신*. 이유 :

- ORM 의 dirty checking 활용. 변경된 필드만 UPDATE 발행 (`UPDATE orders SET status=?, ... WHERE id=?`).
- `session.merge(new_model)` 도 옵션이지만 cascade 동작이 까다로움.
- **단점** : 갱신 가능한 필드를 손으로 나열해야 함. status / total / updated_at 만 적었음 — items 변경은 반영 안 됨. ShopTracker 의 update 는 *상태 변경* 만 가정.

---

## §3 직접 구현 — 다른 storage 로 교체

### 3.1 InMemory 구현체 (테스트용)

```python
class InMemoryOrderRepository:
    def __init__(self):
        self._store: dict[UUID, Order] = {}

    async def save(self, order: Order) -> None:
        self._store[order.id] = order

    async def find_by_id(self, order_id: UUID) -> Order | None:
        return self._store.get(order_id)

    async def update(self, order: Order) -> None:
        self._store[order.id] = order
```

- Mapper 불필요 — 도메인 객체를 그대로 보관.
- 같은 Protocol 만족 → DI 컨테이너에서 SQLAlchemy 구현체와 교체.
- 테스트가 더 빠름 (DB 의존 X).

### 3.2 MongoDB 구현체

```python
class MongoOrderRepository:
    def __init__(self, collection):
        self._coll = collection

    async def save(self, order: Order) -> None:
        await self._coll.insert_one(order_to_document(order))

    async def find_by_id(self, order_id: UUID) -> Order | None:
        doc = await self._coll.find_one({"_id": str(order_id)})
        return document_to_order(doc) if doc else None

def order_to_document(order: Order) -> dict:
    return {
        "_id": str(order.id),
        "customer_name": order.customer_name,
        "status": order.status.value,
        "total_amount": str(order.total_amount.amount),  # str 로 정밀 보존
        "currency": order.total_amount.currency,
        "items": [...],
        "created_at": order.created_at,
        "updated_at": order.updated_at,
    }
```

도메인은 1 줄도 안 바뀜. Mapper + Repository 만 새로 작성.

---

## §4 함정

### 4.1 도메인 객체에 ORM 어노테이션 섞기

```python
# anti-pattern
@dataclass
class Order(Base):           # ← ORM Base 상속 + dataclass
    id: Mapped[str] = mapped_column(...)
    status: Mapped[str] = mapped_column(...)

    def cancel(self): ...
```

ORM 메타데이터가 도메인에 박힘 → Active Record. *분리* 가 깨짐. ShopTracker 의 `Order` 는 SQLAlchemy 모름.

### 4.2 Mapper 안에 비즈니스 로직

```python
# anti-pattern
def order_to_model(order: Order) -> OrderModel:
    if order.status == OrderStatus.PAID:
        order.notify_finance()    # ← 비즈니스 로직 X
    return OrderModel(...)
```

Mapper 는 *순수 변환*. 부수효과 / 검증 / 비즈니스 로직 X. 이게 깨지면 변환 추적이 어려워짐.

### 4.3 동일 객체 두 번 변환 (identity map 누락)

같은 Order 를 두 번 조회 → 두 개의 다른 Python 객체가 생성. ORM 의 *identity map* (같은 PK 는 같은 model 인스턴스) 은 SQLAlchemy 가 보장하지만, 그 후 Mapper 가 매번 새 Order 생성하면 identity map 의 이점 상실.

해결 :
- 도메인 entity 도 cache (Repository 안에서 dict).
- 또는 *같은 트랜잭션 내 같은 ID 는 같은 entity* 라는 invariant 를 Repository 가 보장.

ShopTracker 는 단순화 — 매번 새 entity 생성. 단일 요청 안에서 한 번만 조회한다고 가정.

### 4.4 Lazy loading 이 도메인에 새 나옴

ORM model 을 도메인까지 그대로 들고 가면 (변환 안 하고), 도메인 메서드 안에서 `model.items` 접근 시 lazy loading 트리거 → DB 호출 발생 → 도메인이 DB 의존. ShopTracker 는 model_to_order 에서 *전체 구조를 메모리로 옮김* — 그 후 도메인 메서드는 DB 무관.

### 4.5 Mapper 가 비대 (필드 100 개)

해결 :
- 자동화 라이브러리 (`mashumaro`, `pydantic` 의 `model_validate`, `dataclass-mapper`).
- 단, 자동화는 마법이라 디버깅 어려움. 학습은 손으로.

### 4.6 update 의 부분 갱신 위험

ShopTracker 의 update 는 status / total / updated_at 만 반영. items 변경이 반영 안 됨. 만약 도메인이 items 를 바꾸면 → DB 와 어긋남.

해결 :
- update 가 모든 필드를 반영 (boilerplate ↑).
- 또는 *도메인 수준에서 items 변경 금지* (Order.create 후 불변 의도).

---

## §5 다른 환경과 비교

| 환경 | 분리 정도 |
|---|---|
| **Java + JPA** | Entity 가 Active Record 에 가까움 (Hibernate). 명시적 분리는 별도 DTO 매핑. |
| **C# + EF Core** | Entity = ORM model 이 기본. POCO + Fluent API 로 메타데이터 분리 가능. |
| **Django ORM** | Model = Active Record. 분리하려면 별도 도메인 클래스 + 변환 (드뭄). |
| **Rust (sqlx, sea-orm)** | Repository 패턴 + 명시적 변환이 자연스러움. |
| **Go (sqlx, gorm)** | gorm 은 Active Record 적, sqlx 는 매핑 명시. |

ShopTracker 의 패턴은 **Java DDD 책 (Vernon 등) 의 정통 Hexagonal** 과 같음. SQLAlchemy 가 충분히 유연해서 가능.

---

## §6 테스트 전략

### 6.1 Mapper 단위 테스트

```python
def test_order_to_model_preserves_fields():
    order = Order.create("alice", [OrderItem("a", 2, Money(Decimal("100")))])
    model = order_to_model(order)
    assert model.customer_name == "alice"
    assert model.status == "created"
    assert model.total_amount == 200.0
    assert len(model.items) == 1

def test_model_to_order_recovers_entity():
    model = OrderModel(
        id="11111111-1111-1111-1111-111111111111",
        customer_name="alice",
        status="paid",
        total_amount=200.0, currency="KRW",
        created_at=datetime.now(UTC),
        updated_at=datetime.now(UTC),
        items=[OrderItemModel(product_name="a", quantity=2, unit_price=100.0, currency="KRW")],
    )
    order = model_to_order(model)
    assert order.customer_name == "alice"
    assert order.status == OrderStatus.PAID
    assert order.total_amount.amount == Decimal("200")
    assert len(order.items) == 1

def test_round_trip():
    o1 = Order.create("alice", [OrderItem("a", 2, Money(Decimal("100")))])
    o2 = model_to_order(order_to_model(o1))
    assert o1.customer_name == o2.customer_name
    assert o1.status == o2.status
    assert o1.total_amount == o2.total_amount
```

특히 **round-trip** 테스트가 중요 — 변환의 무손실성 검증.

### 6.2 도메인 테스트는 DB 무관

```python
def test_order_cancel_updates_status():
    order = Order.create("alice", [OrderItem("a", 1, Money(Decimal("100")))])
    order.cancel()
    assert order.status == OrderStatus.CANCELLED
    # ↑ DB 호출 없음. 1ms 안에 끝남.
```

이게 분리의 가장 큰 이득 — *단위 테스트가 DB 를 안 만짐*.

### 6.3 Repository 통합 테스트

```python
async def test_save_and_load(test_session):
    repo = SQLAlchemyOrderRepository(test_session)
    order = Order.create(...)
    await repo.save(order)
    loaded = await repo.find_by_id(order.id)
    assert loaded == order   # __eq__ 가 정의되어 있어야
```

여기는 진짜 DB 사용 (테스트용 SQLite in-memory).

---

## §10 학습 포인트 (한 줄 요약)

1. **Active Record vs Data Mapper** — ShopTracker 는 후자, 분리의 이득을 본다.
2. **도메인은 ORM 모름** — `entities.py` 에 `import sqlalchemy` 없음.
3. **ORM 은 비즈니스 모름** — `models.py` 에 메서드 (cancel 등) 없음.
4. **Mapper 는 순수 변환** — 검증 / 비즈니스 / 부수효과 X.
5. **`Decimal(str(float))`** — float 부동소수점 오차 회피의 표준 관용구.
6. **`UUID ↔ str`, `Enum ↔ str.value`** — DB 호환성 위해 풀어 저장.
7. **`Order(...)` 직접 호출 (factory 우회)** — 복원 시 검증 없이.
8. **lazy="selectin"** — N+1 방지. Order 조회 시 items 함께.
9. **storage 교체가 Mapper + Repo 만 갈아끼우면 됨** — InMemory / Mongo / Redis.
10. **Round-trip 테스트** — `model_to_order(order_to_model(x)) == x` 검증.

---

## 추가 참고

- Martin Fowler, *PoEAA* — Data Mapper 패턴
- SQLAlchemy 2.0 docs : https://docs.sqlalchemy.org/en/20/orm/
- Vaughn Vernon, *Implementing DDD* — Repository / Persistence 챕터
- ShopTracker 다음 글 : `11-repository-pattern.md` (Repository 의 책임 범위)
