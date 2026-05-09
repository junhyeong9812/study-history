# 05 — CQRS (Command Query Responsibility Segregation)

> *"한 객체에 읽기와 쓰기 책임을 모두 맡기지 마라"* — Bertrand Meyer 의 CQS 원칙을 객체에서 *시스템 레벨* 로 끌어올린 것이 CQRS. ShopTracker 의 `command_handlers.py` / `query_handlers.py` 분리, `OrderRepositoryProtocol` / `OrderReadRepositoryProtocol` 분리는 모두 이 원칙의 실천이다. 이 글은 *왜* 분리하는지, *어디까지* 분리해야 하는지, *언제* 별도 read DB 까지 갈 때인지를 코드와 함께 정리.

---

## §0 문제 정의 — 한 함수가 너무 많은 일을 한다

```python
# anti-pattern
class OrderService:
    async def get_or_create_order(self, customer_name, items=None):
        if items is None:
            return await self.repo.find_by_customer(customer_name)
        else:
            order = Order.create(customer_name, items)
            await self.repo.save(order)
            await self.event_bus.publish(...)
            return order
```

이 함수의 문제 :

1. **읽기인지 쓰기인지 시그니처만 봐서는 모른다** — `items` 가 None 이면 읽기, 아니면 쓰기. 호출자가 부수효과를 예측할 수 없다.
2. **트랜잭션 경계가 분기마다 다르다** — 읽기는 read-only tx, 쓰기는 read-write tx 가 적절한데 같은 함수에선 강제할 수 없다.
3. **테스트가 두 배** — 읽기 케이스 / 쓰기 케이스 모두 검증해야 함.
4. **확장이 어렵다** — "주문 목록은 read replica 에서 읽고 싶다" → 함수를 갈라야 함.
5. **권한 / 캐싱 정책이 다르다** — 읽기는 공개, 쓰기는 인증 필수. 캐싱은 읽기에만. 한 함수에 섞여 있으면 미들웨어가 분기.

CQS (Command-Query Separation, Meyer 1988) :

> *모든 메서드는 상태를 바꾸는 Command 거나, 값을 반환하는 Query 둘 중 하나여야 한다. 둘 다는 안 된다.*

CQRS (Greg Young, 2010) 는 이 원칙을 시스템 아키텍처 차원으로 확장 :

> *읽기 모델과 쓰기 모델을 분리한다. 핸들러도 Repository 도 (필요하면 DB 까지) 분리.*

---

## §1 본질 메커니즘

### 1.1 CQRS 의 *스펙트럼*

CQRS 는 "0 or 1" 가 아니라 단계적이다.

```
Level 0 — 같은 객체에 read/write 메서드 (대부분의 CRUD 앱)
   │
Level 1 — Command 객체 / Query 객체 분리, 한 핸들러 클래스에 모두 (CQS 객체 레벨)
   │
Level 2 — Command 핸들러 / Query 핸들러 클래스 분리, 같은 Repository (★ ShopTracker 위치)
   │
Level 3 — Command Repository / Query Repository 인터페이스 분리, 구현체는 같은 DB
   │
Level 4 — Command DB / Query DB 분리 (read replica, denormalized view)
   │
Level 5 — Event Sourcing + Read Model Projection (full CQRS+ES)
```

각 단계의 비용은 기하급수. ShopTracker 는 **Level 2~3** 사이에서 의도적으로 멈춘다.

### 1.2 ShopTracker 가 채택한 분리

| 레이어 | Command 측 | Query 측 |
|---|---|---|
| DTO | `CreateOrderCommand`, `CancelOrderCommand` | `GetOrderQuery`, `ListOrdersQuery` |
| Handler | `CreateOrderHandler`, `CancelOrderHandler` | `GetOrderHandler`, `ListOrdersHandler` |
| Repository Protocol | `OrderRepositoryProtocol` (save/update/find_by_id) | `OrderReadRepositoryProtocol` (find_by_id/list/count) |
| 의존성 | Repo + **EventBus** | Repo only |
| 트랜잭션 | read-write | read-only (의도적) |
| 부수효과 | DB 변경, 이벤트 발행 | 없음 |

DB 와 구현체는 같은 클래스 (`SQLAlchemyOrderRepository`) 가 두 Protocol 모두를 만족한다 — Protocol 의 structural typing 덕분 (02 장).

### 1.3 *왜* 핸들러까지 분리하나

같은 클래스 안에 `create_order` / `get_order` / `list_orders` 를 다 두지 않는 이유 :

| 이유 | 효과 |
|---|---|
| **의존성 최소화** | Query 핸들러는 EventBus 를 import 하지 않음. 생성자에서도 받지 않음. |
| **변경 격리** | 조회 로직 수정 시 Command 핸들러를 안 건드림. 리뷰 범위 축소. |
| **테스트 단순** | Query 핸들러 테스트는 EventBus mock 이 필요 없음. |
| **권한 정책** | "읽기는 모두, 쓰기는 admin" 같은 정책을 클래스 단위로 적용. |
| **캐싱** | Query 핸들러 출력에만 `@cache` 데코레이터 부착. |

---

## §2 ShopTracker 코드 정독

### 2.1 Command DTO (`commands.py`)

```python
# orders/application/commands.py:24-28
@dataclass(frozen=True)
class CreateOrderCommand:
    """주문 생성 명령."""
    customer_name: str
    items: list[OrderItemDTO]
```

- `frozen=True` — Command 도 불변. 핸들러가 받은 후 수정 못 하게 막음.
- `customer_name: str`, `items: list[OrderItemDTO]` — **router** (FastAPI) 에서 들어온 Pydantic 모델을 dataclass 로 변환해서 전달. domain 이 Pydantic 을 모르게 차단.
- 비즈니스 로직 0. 단순 데이터.
- 명사형 + Command 접미사 (`CreateOrderCommand`) — "이렇게 해줘" 의 의도가 이름에 박혀 있음.

> 왜 Pydantic 모델 그대로 안 쓰는가? — Pydantic 은 *검증 라이브러리*. application layer 가 Pydantic 에 의존하면 인프라 / 프레젠테이션 / 도메인의 경계가 무너진다. dataclass 로 한 번 변환하면 이 경계가 명시적이 된다. 단점은 boilerplate. 트레이드오프.

### 2.2 Command Handler (`command_handlers.py`)

```python
# command_handlers.py:20-28 (CreateOrderHandler)
class CreateOrderHandler:
    def __init__(self, repo: OrderRepositoryProtocol, event_bus: EventBus) -> None:
        self._repo = repo
        self._event_bus = event_bus

    async def handle(self, command: CreateOrderCommand) -> UUID:
        ...
```

- 생성자 주입 : `repo` 와 `event_bus` 두 의존성. 둘 다 Protocol 타입.
- `handle(command)` — 단일 메서드. *유즈케이스 = 핸들러 클래스* 의 1:1 매핑. `handle_create / handle_update` 같이 한 클래스에 여러 핸들 메서드를 두지 않음. 이유 : 한 클래스의 책임을 한 유즈케이스로 좁힘.
- 반환 타입 `UUID` — 생성된 리소스의 식별자만 돌려줌. 전체 entity 를 돌려주지 않음 (도메인 객체가 application 경계 밖으로 새는 것을 부분적으로 막음).

```python
# command_handlers.py:30-61 (handle 본문)
async def handle(self, command: CreateOrderCommand) -> UUID:
    # 1. DTO → 도메인 변환
    items = [
        OrderItem(
            product_name=dto.product_name,
            quantity=dto.quantity,
            unit_price=Money(dto.unit_price),
        )
        for dto in command.items
    ]
    # 2. 엔티티 생성 (검증은 Order.create 가 수행)
    order = Order.create(customer_name=command.customer_name, itmes=items)
    # 3. 상태 전이
    order.mark_payment_pending()
    # 4. 저장
    await self._repo.save(order)
    # 5. 이벤트 발행
    await self.event_bus.publish(
        OrderCreatedEvent(...)
    )
    return order.id
```

핸들러의 표준 흐름 5 단계 :

1. **DTO → Domain** : `OrderItemDTO` (Decimal) → `OrderItem(unit_price=Money(...))`. Money 값 객체로 감싸면서 도메인의 invariant 가 적용됨 (09 장).
2. **Factory** : `Order.create(...)` 가 검증과 생성 책임. `Order(...)` 직접 호출 X.
3. **상태 전이** : `mark_payment_pending()` 같은 의도가 드러나는 메서드. 단순 setter 가 아님 (08 장 상태머신).
4. **Persist** : `self._repo.save(order)` — Protocol 메서드. 구현체는 모름.
5. **Event** : `event_bus.publish(...)` — 다른 모듈이 반응. 04 장.

> **버그 노트** : 코드에 `itmes=items` 오타 + `self.event_bus`(`self._event_bus` 가 정확) 가 있음. ShopTracker 는 학습 단계라 일부 버그 잔존. 5 단계 흐름의 골격은 정확.

### 2.3 Query DTO (`queries.py`)

```python
# orders/application/queries.py:11-23
@dataclass(frozen=True)
class GetOrderQuery:
    order_id: str

@dataclass(frozen=True)
class ListOrdersQuery:
    customer_name: str | None = None
    status: str | None = None
    page: int = 1
    size: int = 20
```

- Command 와 같은 frozen dataclass. 다만 의미는 *질문* — "이런 조건의 데이터 줘".
- 페이지네이션 (`page`, `size`) 이 Query DTO 에 포함 — 이런 상세 (페이지 크기, 정렬) 는 *읽기* 쪽에만 있는 관심사.
- 명사형 + Query 접미사. `Get` / `List` 동사 prefix 로 단건 / 다건 구분.

### 2.4 Query Handler (`query_handlers.py`)

```python
# query_handlers.py:28-37 (GetOrderHandler)
class GetOrderHandler:
    def __init__(self, repo: OrderReadRepositoryProtocol) -> None:
        self._repo = repo       # ★ EventBus 없음!

    async def handle(self, query: GetOrderQuery) -> Order:
        order = await self._repo.find_by_id(UUID(query.order_id))
        if order is None:
            raise OrderNotFoundError(query.order_id)
        return order
```

핵심 두 가지 :

1. **`OrderReadRepositoryProtocol` 의존** — 쓰기 쪽 Protocol 이 아닌 읽기 쪽. 같은 인스턴스가 둘을 다 구현해도, *타입* 은 분리해서 의존.
2. **EventBus 없음** — 생성자 시그니처에서부터 부수효과 없음이 보장.

```python
# query_handlers.py:40-58 (ListOrdersHandler)
class ListOrdersHandler:
    def __init__(self, repo: OrderReadRepositoryProtocol) -> None:
        self._repo = repo

    async def handle(self, query: ListOrdersQuery) -> PaginatedOrders:
        items = await self._repo.list_orders(
            customer_name=query.customer_name,
            status=query.status,
            page=query.page,
            size=query.size,
        )
        total = await self._repo.count_orders(
            customer_name=query.customer_name,
            status=query.status,
        )
        return PaginatedOrders(items=items, total=total, page=query.page, size=query.size)
```

- 두 번 호출 (`list_orders` + `count_orders`) — 페이지네이션의 흔한 패턴. 한 번에 같은 트랜잭션이면 일관성 보장. SQL 로는 `SELECT ... LIMIT` + `SELECT COUNT(*)` 두 쿼리.
- `PaginatedOrders` — Query handler 의 출력 DTO. 또 다른 dataclass.

### 2.5 Repository Protocol 분리

```python
# orders/domain/interfaces.py:21-25
class OrderRepositoryProtocol(Protocol):
    """쓰기용 Repository, Command 핸들러가 의존."""
    async def save(self, order: Order) -> None:...
    async def find_by_id(self, order_id: UUID) -> Order | None:...
    async def update(self, order: Order) -> None: ...

# interfaces.py:27-40
class OrderReadRepositoryProtocol(Protocol):
    """읽기용 Repository, Query 핸들러가 의존."""
    async def find_by_id(self, order_id: UUID) -> Order | None:...
    async def list_orders(self, customer_name=None, status=None, page=1, size=20) -> list[Order]:...
    async def count_orders(self, customer_name=None, status=None) -> int: ...
```

관찰 :

- `find_by_id` 는 **양쪽에 모두 있음** — 쓰기 핸들러도 "수정할 대상을 먼저 조회" 가 필요 (`CancelOrderHandler` 가 그렇게 사용). 중복이지만 의도적.
- `list_orders` / `count_orders` 는 **읽기에만 있음** — Command 가 목록을 보지 않음. (만약 본다면 그건 Query 의 책임을 침범한 것)
- `save` / `update` 는 **쓰기에만** — 읽기가 변경하지 않음.

> **DRY 와의 충돌** : `find_by_id` 가 두 Protocol 에 있는 것을 "DRY 위반" 으로 보고 base Protocol 을 만들고 싶을 수 있다. 하지만 그 base 가 한쪽만 변할 때 다른 쪽도 강제 영향을 받게 된다. **인터페이스의 DRY 는 신중히** — 인터페이스는 *분리* 가 기본 (ISP, Interface Segregation Principle).

### 2.6 같은 구현체가 두 Protocol 모두 만족

```python
# infrastructure/repository.py
class SQLAlchemyOrderRepository:
    """OrderRepositoryProtocol + OrderReadRepositoryProtocol 둘 다 구현."""

    async def save(self, order): ...           # ← write
    async def update(self, order): ...         # ← write
    async def find_by_id(self, order_id): ...  # ← both
    async def list_orders(self, ...): ...      # ← read
    async def count_orders(self, ...): ...     # ← read
```

Protocol 의 structural typing 덕분에 `SQLAlchemyOrderRepository` 는 둘 다 자동으로 만족 (`implements` 키워드 불필요). DI 컨테이너에서는 :

```python
@provide(scope=Scope.REQUEST)
def order_write_repo(self, session: AsyncSession) -> OrderRepositoryProtocol:
    return SQLAlchemyOrderRepository(session)

@provide(scope=Scope.REQUEST)
def order_read_repo(self, session: AsyncSession) -> OrderReadRepositoryProtocol:
    return SQLAlchemyOrderRepository(session)
```

같은 구현체를 두 Protocol 로 *각각* 등록. 향후 read replica 로 가면 `read_session` 을 주입하는 별도 구현체로 교체.

---

## §3 직접 구현 — 미니 CQRS

```python
# 1. Command + Query
@dataclass(frozen=True)
class CreateUserCommand:
    name: str
    email: str

@dataclass(frozen=True)
class GetUserQuery:
    user_id: int

# 2. Repo Protocol (분리)
class UserWriteRepo(Protocol):
    async def save(self, user: User) -> int: ...

class UserReadRepo(Protocol):
    async def find_by_id(self, user_id: int) -> User | None: ...

# 3. Handler
class CreateUserHandler:
    def __init__(self, repo: UserWriteRepo, bus: EventBus):
        self.repo, self.bus = repo, bus
    async def handle(self, cmd: CreateUserCommand) -> int:
        user = User.create(cmd.name, cmd.email)
        user_id = await self.repo.save(user)
        await self.bus.publish(UserCreated(user_id=user_id))
        return user_id

class GetUserHandler:
    def __init__(self, repo: UserReadRepo):
        self.repo = repo
    async def handle(self, q: GetUserQuery) -> User:
        return await self.repo.find_by_id(q.user_id)

# 4. Router
@router.post("/users")
async def create_user(
    body: CreateUserBody,
    handler: FromDishka[CreateUserHandler],
):
    user_id = await handler.handle(CreateUserCommand(body.name, body.email))
    return {"id": user_id}

@router.get("/users/{user_id}")
async def get_user(
    user_id: int,
    handler: FromDishka[GetUserHandler],
):
    user = await handler.handle(GetUserQuery(user_id))
    if user is None:
        raise HTTPException(404)
    return UserResponse.from_domain(user)
```

이 흐름의 신호 :

- POST handler 와 GET handler 가 *완전히 다른 클래스* 와 *다른 Protocol* 에 의존.
- Read 쪽은 EventBus 가 어디에도 안 보임.
- 각 핸들러는 책임이 단일.

---

## §4 함정

### 4.1 "쓰기 후 바로 읽기" 일관성

Command 가 끝난 직후 Query 가 결과를 보려면 :
- 같은 트랜잭션이면 OK.
- 다른 세션이면 read replica lag (Level 4+) 에서 stale 가능.

해결 :
- Command 가 *식별자만* 반환 + 클라이언트가 GET 으로 다시 확인.
- 또는 Command handler 가 Query handler 의 결과를 자체적으로 만들어서 반환 (CQRS 살짝 깸).

### 4.2 Read Repository 가 Domain 객체를 반환하는가

ShopTracker 의 `OrderReadRepositoryProtocol` 은 `Order` 도메인 객체를 반환한다. 이건 학습 단계라 OK 지만, 본격 CQRS 에서는 :

- Read 측은 **Read Model** (denormalized DTO) 을 반환.
- 예 : `OrderListItem(id, customer_name, total, status)` — entity 가 아닌 view.
- 이유 : Read 가 도메인 invariant 를 알 필요 없음. JOIN 으로 만든 결과를 도메인으로 매핑하는 비용이 큼.

ShopTracker 가 Order 를 그대로 쓰는 이유 : Phase 1 단순함. Read Model 도입은 phase 5 (CQRS) 의 목표.

### 4.3 Command 가 값을 너무 많이 반환

`CreateOrderHandler.handle` 이 `Order` entity 전체를 반환하면 :
- application layer 의 도메인 객체가 router 까지 노출.
- router 가 `Order.total_amount.amount` 같이 도메인을 직접 까볼 위험.

ShopTracker 처럼 `UUID` 만 반환하면 router 가 다시 GET 핸들러를 호출하거나, 응답 DTO 를 별도로 만들어야 함. 명시적이고 안전.

### 4.4 핸들러 클래스 폭발

유즈케이스마다 클래스가 늘면 50, 100 개씩 핸들러가 생긴다. 대응 :
- 모듈별 폴더 (`orders/application/command_handlers.py`, `payments/application/...`) 로 격리.
- 메디에이터 패턴 (예: Python `mediatr` 라이브러리) 으로 dispatch 자동화.
- 단순 CRUD 는 핸들러를 안 만들고 router 가 repo 를 직접 호출 (CQRS 를 *일관* 적용하지 않는 절충).

### 4.5 Command 안에서 Query 호출

```python
# anti-pattern
async def handle(self, cmd):
    if await self._query_handler.handle(...):   # ← 안 좋음
        ...
```

핸들러 간 호출은 결합을 만든다. 대안 :
- Command handler 가 Repo 를 통해 직접 조회 (자기가 필요한 것은 자기가).
- 또는 *공통 read* 를 application 의 별도 *service* 로 빼서 둘 다 의존.

### 4.6 Read 가 DB 외 자원을 호출

Query handler 가 외부 API 호출 (예: PG 사 잔고 조회) 를 하면 :
- 더 이상 부수효과 없는 "순수 읽기" 가 아님.
- 캐싱, 권한, 트랜잭션 가정이 깨짐.

대응 : 외부 호출은 별도 service / adapter 로 빼고, query handler 는 *DB 만* 본다는 규칙.

---

## §5 다른 언어 / 프레임워크 비교

| 환경 | CQRS 구현 |
|---|---|
| **Spring** | 보통 `@Service` 안에 read/write 메서드 혼재 (Level 0~1). 별도로 `XxxQueryService` / `XxxCommandService` 분리하는 팀도 있음. |
| **NestJS** | `@nestjs/cqrs` 모듈. `CommandBus` / `QueryBus` / `EventBus` 분리, `@CommandHandler` / `@QueryHandler` 데코레이터. |
| **.NET** | MediatR 라이브러리가 사실상 표준. `IRequest<TResponse>` 로 Command/Query 통일 인터페이스, `IRequestHandler` 로 처리. |
| **Java (Axon)** | `@CommandHandler` / `@QueryHandler` / `@EventSourcingHandler` 어노테이션. Event Sourcing 까지 통합. |
| **Go** | 라이브러리 의존 X. 패키지 분리 (`order/command`, `order/query`) 로 구조적 분리. |

ShopTracker 의 패턴은 **NestJS 의 `@nestjs/cqrs` 와 가장 유사** — Command/Query DTO + Handler 클래스 분리, 단 dispatcher (Bus) 는 안 두고 router 가 핸들러를 직접 주입받음.

---

## §6 언제 어디까지 갈 것인가 — 의사결정 가이드

| 상황 | 권장 Level |
|---|---|
| 단순 CRUD, 트래픽 적음, 팀 작음 | Level 0~1 (분리 X 또는 DTO 분리만) |
| 도메인이 풍부, 비즈니스 로직 명확, 모놀리스 | **Level 2~3 (ShopTracker)** |
| Read TPS ≫ Write TPS, 캐싱 / replica 필요 | Level 4 (DB 분리) |
| 감사 / 시간여행 / 복원 필요 | Level 5 (Event Sourcing) |

> **위로 올라갈수록 유지보수 비용 ↑**. 이유 없이 Level 5 까지 가면 *복잡성을 위한 복잡성*. ShopTracker 는 학습이라는 명분으로 Level 2~3 까지 유지하면서 *왜 그 이상이 필요한가* 를 체험하는 게 목적.

---

## §7 테스트 전략

### 7.1 Command handler

```python
@pytest.mark.asyncio
async def test_create_order_persists_and_publishes():
    repo = FakeOrderRepo()
    bus = FakeEventBus()
    handler = CreateOrderHandler(repo, bus)

    order_id = await handler.handle(CreateOrderCommand(
        customer_name="alice",
        items=[OrderItemDTO("a", 1, Decimal("100"))],
    ))

    assert order_id is not None
    assert len(repo.saved) == 1
    assert isinstance(bus.published[0], OrderCreatedEvent)
```

- Repo 와 EventBus 모두 fake.
- 검증 : 저장됐는지 + 이벤트 발행됐는지.

### 7.2 Query handler

```python
@pytest.mark.asyncio
async def test_get_order_returns_existing_order():
    repo = FakeOrderRepo(seed=[order])
    handler = GetOrderHandler(repo)

    result = await handler.handle(GetOrderQuery(str(order.id)))

    assert result == order
```

- EventBus 가 없음. Mock 도 불필요. 더 간단.

### 7.3 Read/Write 모순 검증

```python
async def test_command_changes_visible_to_query(test_container):
    create = await test_container.get(CreateOrderHandler)
    get = await test_container.get(GetOrderHandler)

    order_id = await create.handle(CreateOrderCommand(...))
    fetched = await get.handle(GetOrderQuery(str(order_id)))

    assert fetched.id == order_id
```

- Command → Query 흐름이 같은 트랜잭션 / 같은 DB 에서 일관됐는지 통합 검증.

---

## §10 학습 포인트 (한 줄 요약)

1. **CQS** : 메서드는 Command 또는 Query 둘 중 하나. 둘 다 X.
2. **CQRS** : 시스템 레벨로 확장 — 핸들러 / Repo / DB 까지 분리 가능.
3. **CQRS 는 스펙트럼** : Level 0 ~ 5. 필요한 만큼만.
4. **ShopTracker = Level 2~3** : 핸들러 클래스 + Repo Protocol 분리, DB 는 같음.
5. **Command 핸들러 = repo + EventBus** / **Query 핸들러 = repo only** — 의존성 시그니처에서부터 부수효과 차이가 보임.
6. **Repo Protocol 분리는 ISP** : `find_by_id` 중복 OK. 인터페이스 분리가 우선.
7. **같은 구현체로 두 Protocol 만족** : structural typing + DI 의 결합. read replica 갈 때 분리 쉬움.
8. **Command 는 식별자만 반환** : 도메인 객체가 application 밖으로 새는 것 차단.
9. **Read Model** (denormalized DTO) 은 Level 4+ 의 표지. ShopTracker 는 아직 entity 그대로.
10. **Event Sourcing 은 별 우주** : CQRS 와 자주 같이 쓰지만 분리해서 이해. 무리해서 같이 도입 X.

---

## 추가 참고

- Greg Young, *CQRS Documents*, https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf
- Martin Fowler, *CQRS*, https://martinfowler.com/bliki/CQRS.html
- Bertrand Meyer, *Object-Oriented Software Construction* (CQS 원전)
- ShopTracker 다음 글 : `06-saga-cross-module.md` (Command 가 다른 모듈을 트리거할 때의 보상 거래)
