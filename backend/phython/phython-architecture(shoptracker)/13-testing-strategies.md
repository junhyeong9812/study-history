# 13 — 테스트 전략 (Hexagonal 아키텍처에서의 4 단계)

> 좋은 아키텍처는 *테스트가 빠르고 명확하다*. ShopTracker 의 분리 (도메인 / application / infrastructure / presentation) 는 각 레이어를 *서로 다른 방식과 속도로* 테스트할 수 있게 해준다. 이 글은 어떤 테스트가 어디에 속하고 무엇을 검증해야 하는지를 정리.

---

## §0 테스트 피라미드 in Hexagonal

```
            ┌─────────────────┐
            │   E2E (HTTP)    │  ← 적게, 비싸게, 느리게 (10 ~ 100 ms / 케이스)
            └─────────────────┘
         ┌────────────────────────┐
         │  Integration (DB / 외부)│  ← 중간 (10 ~ 50 ms)
         └────────────────────────┘
      ┌──────────────────────────────┐
      │  Application (Handler + Fake) │  ← 많이 (1 ~ 5 ms)
      └──────────────────────────────┘
   ┌──────────────────────────────────────┐
   │  Domain (Entity / VO 단위)            │  ← 매우 많이 (< 1 ms)
   └──────────────────────────────────────┘
```

각 층이 *전 층의 무엇을 신뢰할 수 있게 하는지* 가 핵심.

---

## §1 4 가지 레이어 별 테스트

### 1.1 Domain Test — 가장 빠르고 가장 많이

대상 :
- `Order.create` 의 검증 규칙
- `OrderStatus.can_transition_to` 의 매핑
- `Money.add` / `subtract` / 음수 차단

특징 :
- DB / HTTP / DI 컨테이너 / 비동기 — 전부 *없음*.
- 순수 Python 단위 테스트.
- 1 케이스 < 1 ms.
- 100 ~ 500 케이스가 보통.

```python
def test_money_add():
    assert Money(Decimal("100")).add(Money(Decimal("50"))) == Money(Decimal("150"))

def test_order_create_rejects_empty_items():
    with pytest.raises(InvalidOrderError):
        Order.create("alice", [])

def test_paid_cannot_be_cancelled():
    o = Order.create(...)
    o.mark_payment_pending()
    o.mark_paid()
    with pytest.raises(InvalidStatusTransition):
        o.cancel()
```

핵심 가치 : **비즈니스 규칙의 명세서**. 이 테스트들이 *문서* 역할.

### 1.2 Application Test — Handler + Fake Repo

대상 :
- `CreateOrderHandler.handle` 이 repo.save 를 부르는가
- `CancelOrderHandler` 가 OrderCancelled 이벤트를 발행하는가
- Query handler 가 PaginatedOrders 를 잘 만드는가

특징 :
- DB X (InMemoryRepo 사용).
- EventBus 도 fake.
- async 테스트 (`pytest-asyncio`).
- 1 케이스 1 ~ 5 ms.

```python
@pytest.mark.asyncio
async def test_create_order_handler_persists_and_publishes():
    repo = InMemoryOrderRepository()
    bus = FakeEventBus()
    handler = CreateOrderHandler(repo, bus)

    order_id = await handler.handle(CreateOrderCommand(
        customer_name="alice",
        items=[OrderItemDTO("a", 1, Decimal("100"))],
    ))

    assert order_id in repo._store
    assert any(isinstance(e, OrderCreatedEvent) for e in bus.published)
```

Fake 객체 만들기가 핵심 — Repository Protocol 의 InMemory 구현은 한 번 만들어두고 모든 테스트에서 재사용.

### 1.3 Integration Test — 진짜 DB / 진짜 EventBus

대상 :
- SQLAlchemy Repository 가 실제로 INSERT / SELECT 를 잘 하는가
- Mapper 의 round-trip 이 무손실인가
- 이벤트 흐름 (Choreography Saga) 이 끝까지 흐르는가

특징 :
- 진짜 DB (sqlite in-memory 또는 testcontainers PostgreSQL).
- 진짜 EventBus + 진짜 핸들러 등록.
- 1 케이스 10 ~ 50 ms.

```python
@pytest.fixture
async def session():
    engine = create_async_engine("sqlite+aiosqlite:///:memory:")
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    async with AsyncSession(engine) as s:
        yield s

@pytest.mark.asyncio
async def test_repository_saves_and_loads(session):
    repo = SQLAlchemyOrderRepository(session)
    o = Order.create("alice", [OrderItem("a", 2, Money(Decimal("100")))])
    await repo.save(o)
    await session.commit()
    loaded = await repo.find_by_id(o.id)
    assert loaded.id == o.id
    assert loaded.total_amount.amount == Decimal("200")
```

**Test Container** (03 장 §6) 활용 :

```python
@pytest.fixture
async def test_container():
    container = make_async_container(
        AppProvider(),
        OrdersProvider(),
        TestDbProvider(),     # in-memory sqlite
    )
    yield container
    await container.close()

@pytest.mark.asyncio
async def test_create_order_flow(test_container):
    async with test_container() as request_container:
        handler = await request_container.get(CreateOrderHandler)
        order_id = await handler.handle(CreateOrderCommand(...))
        # 그리고 다시 조회
        get_handler = await request_container.get(GetOrderHandler)
        order = await get_handler.handle(GetOrderQuery(str(order_id)))
        assert order.customer_name == "alice"
```

### 1.4 E2E (HTTP) Test — TestClient

대상 :
- HTTP status code, JSON 형식
- Pydantic 검증이 422 로 떨어지는지
- 도메인 예외 → 4xx 매핑

특징 :
- FastAPI TestClient. 진짜 HTTP 호출 시뮬.
- 진짜 DB.
- 1 케이스 50 ~ 200 ms.
- *마지막 보루* — 핵심 시나리오만.

```python
def test_create_and_get_order(client):
    create = client.post("/api/v1/orders/", json={
        "customer_name": "alice",
        "items": [{"product_name": "a", "quantity": 1, "unit_price": 100}],
    })
    assert create.status_code == 201
    order_id = create.json()["id"]

    fetch = client.get(f"/api/v1/orders/{order_id}")
    assert fetch.status_code == 200
    assert fetch.json()["customer_name"] == "alice"
```

---

## §2 ShopTracker 의 테스트 폴더 구조 (권장)

```
tests/
├── unit/
│   ├── domain/
│   │   ├── test_order.py
│   │   ├── test_money.py
│   │   └── test_order_status.py
│   └── application/
│       ├── test_create_order_handler.py
│       └── test_cancel_order_handler.py
├── integration/
│   ├── test_repository.py
│   ├── test_mapper_roundtrip.py
│   └── test_saga_flow.py
├── e2e/
│   └── test_orders_api.py
└── conftest.py    # 공통 fixture (session, container, client)
```

- `unit/` 은 빠르게 (전체 < 1 초).
- CI 에서 `pytest tests/unit -x` 를 watch — 즉시 피드백.
- `integration/` + `e2e/` 는 느림 — PR 시점.

---

## §3 Fake / Stub / Mock 의 차이

| 종류 | 용도 |
|---|---|
| **Fake** | 단순화된 진짜 구현 (InMemoryRepo). 호출 검증 없이 행동. |
| **Stub** | 정해진 값을 반환. "이 호출엔 이거 줘" 식. |
| **Mock** | 호출 검증까지 (`assert mock.called`). |
| **Spy** | 진짜 호출 + 호출 기록. |

ShopTracker 는 *Fake 위주* — Protocol 기반이라 InMemoryRepo / FakeEventBus / FakePG 가 자연스러움. `unittest.mock.Mock` 의 마법은 거의 안 씀.

```python
class FakeEventBus:
    def __init__(self):
        self.published: list = []
    async def publish(self, event):
        self.published.append(event)
    def subscribe(self, event_type, handler):
        pass
```

10 줄짜리 Fake 가 mock 보다 *읽기 좋음*.

---

## §4 함정

### 4.1 단위 테스트에 진짜 DB

도메인 테스트가 DB 띄우면 → 100 케이스 × 100ms = 10 초. 피드백 느려짐. → DB 없이 도메인을 테스트할 수 있게 *분리* (10 장).

### 4.2 모든 케이스를 E2E 로

100 개 검증을 다 HTTP 통해서 → 너무 느림 + 실패 메시지가 모호 ("422" 만 보고 어디가 틀린지 추적).
→ Pydantic 검증 자체는 unit, HTTP 매핑은 E2E 한두 개.

### 4.3 Fake 가 비대

InMemoryRepo 가 실제 DB 의 모든 동작 (lazy loading, cascade, transaction isolation) 을 시뮬하려고 하면 → 더 이상 Fake 가 아니라 *또 다른 DB*. 단순함을 잃음.
→ Fake 는 *Protocol 의 행동* 만 충실히. 현실의 DB 동작은 Integration 으로.

### 4.4 시간 의존

`datetime.now()` 가 핸들러에서 호출되면 테스트 마다 값이 다름.
→ `Clock` Protocol 주입 (정책 주입 패턴, 03 장).

```python
class Clock(Protocol):
    def now(self) -> datetime: ...

class FixedClock:
    def __init__(self, t): self._t = t
    def now(self): return self._t
```

### 4.5 랜덤 의존

UUID 생성이 핸들러에서 → 테스트가 비결정적.
→ `IdGenerator` Protocol 주입.

### 4.6 테스트 간 상태 누출

전역 EventBus 에 핸들러 누적, 전역 DI 컨테이너 등.
→ fixture scope 를 "function" 으로. 매 테스트마다 새 컨테이너.

### 4.7 비동기 테스트의 함정

`pytest-asyncio` 의 mode 설정 누락 → 테스트가 silently skip.

```ini
# pytest.ini 또는 pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

`@pytest.mark.asyncio` 안 붙여도 auto.

---

## §5 다른 환경

| 환경 | 도구 |
|---|---|
| **Python** | pytest, pytest-asyncio, hypothesis (property-based), testcontainers |
| **Java** | JUnit 5, Mockito, Testcontainers |
| **TS** | Jest / Vitest, supertest |
| **Go** | testing 표준 패키지, testify, dockertest |

ShopTracker 의 패턴 (Protocol + Fake) 은 모든 언어에 적용 가능.

---

## §6 Hypothesis (Property-Based Testing) — 추천

도메인 invariant 를 *수많은 랜덤 입력* 으로 검증.

```python
from hypothesis import given, strategies as st
from decimal import Decimal

@given(
    st.decimals(min_value=0, max_value=10**9, allow_nan=False, allow_infinity=False),
    st.decimals(min_value=0, max_value=10**9, allow_nan=False, allow_infinity=False),
)
def test_money_add_commutative(a, b):
    assert Money(a).add(Money(b)) == Money(b).add(Money(a))

@given(st.decimals(min_value=0, max_value=10**9))
def test_money_add_zero_identity(a):
    assert Money(a).add(Money(Decimal("0"))) == Money(a)
```

10 ~ 100 케이스를 자동 생성. 엣지 케이스 (0, max, min) 자동 탐색.

---

## §10 학습 포인트 (한 줄 요약)

1. **테스트 피라미드** : domain (많이) → app (많이) → integration (중간) → e2e (적게).
2. **도메인 테스트는 < 1 ms** — DB / HTTP / async 없이.
3. **Application 테스트는 Fake 의존** — InMemoryRepo + FakeEventBus.
4. **Integration 은 진짜 DB** — testcontainers / sqlite in-memory.
5. **E2E 는 핵심 시나리오만** — 비싸고 모호.
6. **Fake > Mock** — Protocol 기반이라 직접 작성이 깔끔.
7. **시간 / 랜덤 / 외부 API 는 정책 주입** — Clock / IdGen / Gateway 추상.
8. **fixture scope = function** — 테스트 간 격리.
9. **Hypothesis 로 invariant 자동 검증** — domain VO 에 강력.
10. **테스트 이름이 곧 명세** — `test_paid_cannot_be_cancelled` 가 비즈니스 규칙.

---

## 추가 참고

- pytest docs : https://docs.pytest.org/
- Hypothesis : https://hypothesis.readthedocs.io/
- testcontainers-python : https://testcontainers-python.readthedocs.io/
- ShopTracker 다음 글 : `14-python-typing.md` (타입 힌트 / Protocol / Generic)
