# 11 — Repository Pattern (도메인의 영속화 추상화)

> Repository 는 *컬렉션처럼 보이는* 영속화 인터페이스다. 도메인 코드가 "DB 에서 Order 가져와" 가 아니라 "주문 컬렉션에서 ID 로 찾아" 라고 말할 수 있게 해준다. ShopTracker 의 `OrderRepositoryProtocol` + `SQLAlchemyOrderRepository` 가 이 패턴의 표본.

---

## §0 문제 정의 — DB 코드가 도메인 안에 새 나오면

```python
# anti-pattern
class CreateOrderHandler:
    async def handle(self, command):
        async with engine.connect() as conn:
            result = await conn.execute(text("INSERT INTO orders ..."))
            order_id = result.lastrowid
        ...
```

문제 :
- 핸들러가 SQL / DB 연결을 안다.
- 테스트할 때 진짜 DB 필요.
- DB 교체가 핸들러 전체 수정.
- 트랜잭션 경계가 핸들러에 노출.

해결 : **Repository = 영속화 어댑터**. 핸들러는 `repo.save(order)` 라고만 말함.

---

## §1 본질 — Eric Evans 의 정의

> Repository 는 *모든 객체가 메모리 안에 있는 컬렉션* 인 것처럼 도메인이 영속화를 다루게 해주는 객체.

### 1.1 핵심 속성

1. **컬렉션 인터페이스** — `add`, `find`, `remove` 같은 컬렉션 동사.
2. **Aggregate 단위** — Repository 는 Aggregate Root 마다 하나. `OrderItemRepository` 가 별도로 없음 (Order 통해서만).
3. **도메인 객체 in/out** — DB 행이 아니라 도메인 entity.
4. **DB 무관 인터페이스** — 도메인에는 Protocol/interface만. 구현은 infrastructure.

### 1.2 ShopTracker 의 매핑

| 역할 | 위치 |
|---|---|
| Protocol (port) | `orders/domain/interfaces.py` |
| 구현 (adapter) | `orders/infrastructure/repository.py` |
| Mapper | `orders/infrastructure/mappers.py` |
| 의존자 | `orders/application/*_handlers.py` |

---

## §2 ShopTracker 코드 정독

### 2.1 Protocol 측

```python
# domain/interfaces.py
class OrderRepositoryProtocol(Protocol):
    async def save(self, order: Order) -> None: ...
    async def find_by_id(self, order_id: UUID) -> Order | None: ...
    async def update(self, order: Order) -> None: ...

class OrderReadRepositoryProtocol(Protocol):
    async def find_by_id(self, order_id: UUID) -> Order | None: ...
    async def list_orders(self, ...) -> list[Order]: ...
    async def count_orders(self, ...) -> int: ...
```

- 메서드 이름이 *컬렉션 동사* — `save`, `find_by_id`, `list`. SQL 동사 (`select`, `insert`) 아님.
- 인자/반환 모두 도메인 (`Order`, `UUID`). DB 행 / Row 아님.
- `Protocol` 이라 구현체가 명시적 implements 필요 없음 (02 장).

### 2.2 SQLAlchemy 구현

```python
# infrastructure/repository.py
class SQLAlchemyOrderRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session

    async def save(self, order: Order) -> None:
        model = order_to_model(order)
        self._session.add(model)
        await self._session.flush()
```

- 생성자에 `AsyncSession` 만 주입. session 은 DI 가 요청 단위로 관리.
- `save` = Mapper + session.add + flush. 책임 단순.
- `commit` 안 함 — DI 의 generator 가 트랜잭션 종결 (03 장).

### 2.3 find / update / list / count

(10 장 §2 참조 — Mapper 와 결합.)

핵심 패턴 :
- find : `session.get` → mapper → 도메인.
- update : 기존 model 가져와 부분 갱신. `session.merge` 보다 명시적.
- list : `select(...).where(...).order_by(...).limit/offset`. SQL 빌더 노출은 Repository 안에만.
- count : `select(func.count())`. 페이지네이션 total.

### 2.4 같은 구현체로 두 Protocol 만족

`SQLAlchemyOrderRepository` 가 `OrderRepositoryProtocol` 와 `OrderReadRepositoryProtocol` 둘 다 자동 만족 (structural typing). DI 컨테이너에서 두 Protocol 로 각각 등록 (05 장).

---

## §3 직접 구현 — InMemory

```python
class InMemoryOrderRepository:
    def __init__(self):
        self._store: dict[UUID, Order] = {}

    async def save(self, order: Order) -> None:
        self._store[order.id] = order

    async def find_by_id(self, order_id: UUID) -> Order | None:
        return self._store.get(order_id)

    async def update(self, order: Order) -> None:
        if order.id in self._store:
            self._store[order.id] = order

    async def list_orders(self, customer_name=None, status=None, page=1, size=20):
        items = list(self._store.values())
        if customer_name:
            items = [o for o in items if o.customer_name == customer_name]
        if status:
            items = [o for o in items if o.status.value == status]
        items.sort(key=lambda o: o.created_at, reverse=True)
        start = (page - 1) * size
        return items[start:start+size]

    async def count_orders(self, customer_name=None, status=None):
        items = list(self._store.values())
        if customer_name: items = [o for o in items if o.customer_name == customer_name]
        if status: items = [o for o in items if o.status.value == status]
        return len(items)
```

같은 Protocol → 같은 핸들러에 그대로 주입 가능. 테스트 / 데모 / 학습에 유용.

---

## §4 함정

### 4.1 Repository 가 SQL 빌더를 외부에 노출

```python
# anti-pattern
def find_orders_query(): return select(OrderModel).where(...)

# handler
stmt = repo.find_orders_query()
stmt = stmt.where(...)        # ← 핸들러가 SQL 조립
```

→ 인프라가 도메인까지 새 나옴. **Repository 의 메서드는 비즈니스 의도** (`find_pending_orders_for_customer`) 로 표현.

### 4.2 Generic Repository 의 함정

```python
class GenericRepo(Generic[T]):
    async def save(self, entity: T): ...
    async def find_by_id(self, id) -> T: ...
```

장점 : DRY. 단점 : 도메인별 *의미 있는* 쿼리 (`find_top_customers_this_month`) 가 들어갈 곳이 없어짐. ShopTracker 는 도메인별 Repository (Generic 안 씀).

### 4.3 Repository 가 너무 많은 메서드

`find_by_X`, `find_by_X_and_Y`, `find_by_X_or_Y`... 폭발. 신호 :
- *Specification 패턴* (조건 객체) 도입.
- 또는 Read Model 분리 (CQRS Level 4+).

### 4.4 Repository 가 트랜잭션을 commit

```python
# anti-pattern
async def save(self, order):
    self._session.add(model)
    await self._session.commit()    # ← 누가 트랜잭션 책임지나?
```

→ Repository 가 commit 하면 같은 핸들러에서 두 Repository 를 쓸 때 트랜잭션이 분리됨. **commit 은 더 위 레이어 (DI 의 session generator 또는 Unit of Work)**.

### 4.5 Repository in Repository

`OrderRepository` 가 `PaymentRepository` 를 알면 → 모듈 결합. Repository 는 *자기 모듈의 aggregate* 만 책임.

다른 모듈 데이터가 필요하면 application layer 에서 조립 (각 Repo 따로 호출).

### 4.6 N+1 문제

```python
orders = await repo.list_orders(...)
for o in orders:
    print(o.items)            # ← lazy loading 이면 매번 SELECT
```

해결 : `lazy="selectin"` 또는 명시적 eager loading (`options(selectinload(...))`).

ShopTracker 는 model 에 `lazy="selectin"` 명시 (10 장).

---

## §5 다른 환경

| 환경 | Repository |
|---|---|
| **Spring Data JPA** | `JpaRepository<Order, UUID>` 인터페이스만 정의 → 구현 자동 생성. |
| **NestJS + TypeORM** | `Repository<Order>` 제공, custom Repository 도 가능. |
| **Django ORM** | Manager 가 Repository 역할. `Order.objects.filter(...)`. Active Record 적. |
| **SQLAlchemy (전통)** | Repository 패턴 직접 구현. SA 가 Repository 자체를 제공하지 않음. |
| **Rust (sea-orm)** | ActiveModel 패턴. Repository 는 직접 작성. |

ShopTracker = SQLAlchemy 전통. 직접 작성으로 분리 명확.

---

## §6 Unit of Work 패턴

Repository 와 짝. *여러 Repository 의 변경을 한 트랜잭션으로 묶는* 패턴.

```python
class UnitOfWork:
    def __init__(self, session: AsyncSession):
        self.session = session
        self.orders = SQLAlchemyOrderRepository(session)
        self.payments = SQLAlchemyPaymentRepository(session)

    async def __aenter__(self): return self
    async def __aexit__(self, exc_type, exc, tb):
        if exc:
            await self.session.rollback()
        else:
            await self.session.commit()
        await self.session.close()

# 사용
async with UnitOfWork(session) as uow:
    await uow.orders.save(order)
    await uow.payments.save(payment)
# 둘 다 commit 또는 둘 다 rollback
```

ShopTracker 는 별도 Uow 안 두고 *DI 의 session generator* 가 같은 역할 (03 장 §2.3) — async session 이 요청 동안 살아있고 끝에 commit/rollback. 더 단순.

---

## §7 테스트 전략

### 7.1 InMemory 로 핸들러 단위 테스트

```python
async def test_create_order_handler():
    repo = InMemoryOrderRepository()
    bus = FakeEventBus()
    handler = CreateOrderHandler(repo, bus)
    await handler.handle(CreateOrderCommand(...))
    assert len(repo._store) == 1
```

진짜 DB 안 만짐.

### 7.2 SQL 통합 테스트 (testcontainers / sqlite)

```python
@pytest.fixture
async def test_session():
    engine = create_async_engine("sqlite+aiosqlite:///:memory:")
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    async with AsyncSession(engine) as session:
        yield session

async def test_repo_round_trip(test_session):
    repo = SQLAlchemyOrderRepository(test_session)
    order = Order.create(...)
    await repo.save(order)
    await test_session.commit()
    loaded = await repo.find_by_id(order.id)
    assert loaded.id == order.id
```

### 7.3 Contract 테스트 — 두 구현체가 같은 행동

```python
@pytest.mark.parametrize("repo_factory", [
    lambda: InMemoryOrderRepository(),
    lambda session: SQLAlchemyOrderRepository(session),
])
async def test_save_then_find(repo_factory):
    ...
```

InMemory 와 SQLAlchemy 가 *동일 Protocol* 을 *동일하게* 구현하는지 검증.

---

## §10 학습 포인트 (한 줄 요약)

1. **Repository = 컬렉션 인터페이스** — `save`, `find`, `list`. SQL 동사 X.
2. **Aggregate Root 단위** — OrderItemRepository 별도 없음.
3. **도메인 객체 in/out** — Row 가 아닌 entity.
4. **Protocol (port) + Impl (adapter)** — Hexagonal 의 정수.
5. **commit 은 Repository 책임 X** — Unit of Work 또는 DI session.
6. **Generic Repository 신중히** — 도메인 의미가 사라짐.
7. **N+1 회피** — `selectin` / 명시적 eager loading.
8. **InMemory 구현체 = 테스트의 친구** — 빠른 단위 테스트.
9. **Contract 테스트** — 모든 구현체가 같은 행동 보장.
10. **Repository in Repository 금지** — 모듈 결합. application 에서 조립.

---

## 추가 참고

- Eric Evans, *DDD*, Repository
- Martin Fowler, *PoEAA*, Repository / Unit of Work
- ShopTracker 다음 글 : `12-fastapi-router-pydantic.md` (HTTP layer)
