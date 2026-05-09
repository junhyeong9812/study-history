# 21 — 테스트 인프라 패턴 (Hexagonal + 이벤트 기반에서의 실전)

> 13 장은 테스트의 *철학* — 4 단 피라미드, Fake vs Mock, 시간/랜덤 격리 — 을 다뤘다. 이 글은 그 철학을 ShopTracker 의 통합 테스트에 *실제로 적용할 때 부딪히는 패턴들* 을 다룬다 : conftest 의 누적 갱신, 비결정적 외부 시스템 (FakeGateway random) 처리, 비동기 이벤트 결과 폴링, 테스트 격리. phase7_integration_tests.md 의 6 개 통합 테스트가 어떤 *기술* 위에 서 있는지를 보여준다.

---

## §0 문제 정의 — 학습용 통합 테스트의 어려움

### 0.1 좋은 통합 테스트의 조건

- **결정적** — 같은 입력에 같은 결과. 100 번 돌려도 99 번 통과 / 1 번 실패는 통과 0 번과 같다.
- **빠름** — 단위 테스트보다 느리지만 한 케이스 100ms 이내.
- **격리됨** — 다른 테스트의 부산물이 영향 X.
- **가독성** — `test_premium_subscriber_gets_10_percent_discount` 만 봐도 의도가 명확.

### 0.2 ShopTracker 의 통합 테스트가 도전하는 것

phase7 의 6 개 테스트가 검증하는 것 :
- 주문 → 결제 자동 생성 (이벤트 기반)
- 결제 → 배송 자동 생성 (Saga 한 단계 진행)
- 구독 등급별 정책 주입 (헤더 → DB → DTO → 정책 분기)
- 결제 거절 시 보상 흐름 (역방향 이벤트)
- Tracking 타임라인 기록 (모든 이벤트 누적)

이걸 검증하려면 :
- conftest 가 phase4 시점의 모든 의존성 (Orders + Subscriptions + Payments + Shipping + Tracking) 을 알아야 함
- TestProvider 가 prod 의 5 개 Provider 를 흉내내야 함
- FakeGateway 의 random 90/10 을 어떻게든 다뤄야 함
- 이벤트가 비동기로 흐르는 동안 결과를 *기다려야* 함
- 매 테스트가 깨끗한 상태로 시작해야 함

이 패턴들이 정리되지 않으면 conftest 가 미궁이 되고 테스트가 flaky 해진다.

### 0.3 이 글이 다루는 다섯 패턴

1. **TestProvider 누적 갱신** — phase 진행에 따라 어떻게 확장하나
2. **비결정적 외부 처리** — FakeGateway random 두 가지 패턴 (재시도 + monkeypatch)
3. **비동기 이벤트 결과 폴링** — 이벤트 처리가 끝났는지 어떻게 아나
4. **테스트 격리** — fixture scope, 핸들러 누적, 전역 상태
5. **TestProvider vs ProdProvider** — 어디까지 같게 할 것인가

---

## §1 TestProvider 누적 갱신 패턴

### 1.1 Phase 1 시점의 TestProvider — 시작점

```python
# tests/conftest.py (Phase 1)
class TestProvider(Provider):
    @provide(scope=Scope.REQUEST)
    async def session(self) -> AsyncSession:
        async with session_factory() as session:
            try:
                yield session
                await session.commit()
            except Exception:
                await session.rollback()
                raise

    @provide(scope=Scope.APP)
    def test_event_bus(self) -> EventBus:
        return event_bus     # 픽스처에서 만든 인스턴스

    # Orders
    @provide(scope=Scope.REQUEST)
    def order_repo(self, session: AsyncSession) -> SQLAlchemyOrderRepository:
        return SQLAlchemyOrderRepository(session)

    @provide(scope=Scope.REQUEST)
    def create_order(
        self, repo: SQLAlchemyOrderRepository, eb: EventBus,
    ) -> CreateOrderHandler:
        return CreateOrderHandler(repo, eb)

    # ... 나머지 Order/Subscription handlers ...
```

### 1.2 Phase 2 시점에 추가되는 것

```python
# === Phase 2 추가 ===
@provide(scope=Scope.REQUEST)
async def subscription_context(
    self,
    request: Request,
    repo: SQLAlchemySubscriptionRepository,
) -> SubscriptionContext:
    customer_name = request.headers.get("X-Customer-Name", "guest")
    sub = await repo.find_active_by_customer(customer_name)
    if sub is None or not sub.is_active():
        return SubscriptionContext.guest(customer_name)
    return SubscriptionContext(
        customer_name=customer_name,
        tier=sub.tier.value,
        is_active=True,
    )

@provide(scope=Scope.REQUEST)
def payment_repo(self, session: AsyncSession) -> SQLAlchemyPaymentRepository:
    return SQLAlchemyPaymentRepository(session)

@provide(scope=Scope.REQUEST)
def payment_gateway(self) -> FakePaymentGateway:
    return FakePaymentGateway()

@provide(scope=Scope.REQUEST)
def discount_policy(self, sub_ctx: SubscriptionContext) -> DiscountPolicy:
    match sub_ctx.tier:
        case "premium":
            return SubscriptionDiscountPolicy(Decimal("0.10"), "premium_subscription")
        case "basic":
            return SubscriptionDiscountPolicy(Decimal("0.05"), "basic_subscription")
        case _:
            return NoDiscountPolicy()

@provide(scope=Scope.REQUEST)
def process_payment(
    self,
    repo: SQLAlchemyPaymentRepository,
    gateway: FakePaymentGateway,
    policy: DiscountPolicy,
    eb: EventBus,
) -> ProcessPaymentHandler:
    return ProcessPaymentHandler(repo, gateway, policy, eb)
```

### 1.3 누적 갱신이 *왜* 자연스러운 패턴인가

| 대안 | 단점 |
|---|---|
| **Phase 마다 새 TestProvider 클래스** | 누가 어떤 Phase 인지 매번 결정 — 테스트가 phase 전환에 흔들림 |
| **Phase 2 부터는 prod 의 PaymentsProvider 그대로 import 해서 쓰기** | DB / config 등 prod 환경 의존이 침투. 테스트 격리 깨짐 |
| **누적 (★ ShopTracker)** | 테스트는 항상 "최종 상태" 를 검증. phase 진행과 무관하게 안정적 |

학습 의도는 phase 별 *진행* 인데, 테스트는 phase 와 무관하게 *현재 상태* 를 검증해야 한다. 그래서 conftest 는 누적.

### 1.4 누적의 비용

- TestProvider 가 *길어진다*. Phase 4 면 메서드 20 개 이상.
- prod 의 Provider 5 개를 한 클래스에 모은 형태 — 책임이 흩어짐.

규모가 더 커지면 prod 처럼 분리할 수 있다 :

```python
# 큰 규모 시
class TestAppProvider(Provider): ...
class TestOrdersProvider(Provider): ...
class TestPaymentsProvider(Provider): ...

container = make_async_container(
    TestAppProvider(), TestOrdersProvider(), TestPaymentsProvider(),
)
```

ShopTracker 는 학습 단계라 단일 TestProvider — 한 클래스만 보면 되는 단순함.

---

## §2 ShopTracker 의 conftest.py — 핵심 흐름

### 2.1 4 단 fixture 구조

```python
@pytest.fixture
async def async_engine():
    """SQLite in-memory + Base.metadata.create_all."""
    engine = create_async_engine("sqlite+aiosqlite://", echo=False)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    await engine.dispose()


@pytest.fixture
async def session_factory(async_engine):
    return async_sessionmaker(async_engine, class_=AsyncSession, ...)


@pytest.fixture
async def event_bus():
    """매 테스트마다 새 인스턴스 — 핸들러 누적 방지."""
    return InMemoryEventBus()


@pytest.fixture
async def async_client(session_factory, event_bus):
    """TestProvider + register_event_handlers + httpx AsyncClient."""
    test_app = FastAPI()
    test_app.include_router(orders_router)
    test_app.include_router(subs_router)
    test_app.include_router(payments_router)
    test_app.include_router(shipping_router)
    test_app.include_router(tracking_router)

    container = make_async_container(TestProvider())
    setup_dishka(container, test_app)

    # === 이벤트 핸들러 등록 (prod main.py 와 같은 코드) ===
    async def on_order_created(event):
        async with container() as rc:
            handler = await rc.get(ProcessPaymentHandler)
            await payments_handle_order_created(event, handler)

    # ... 나머지 어댑터 11 개 ...

    event_bus.subscribe(OrderCreatedEvent, on_order_created)
    # ... 나머지 subscribe ...

    async with AsyncClient(
        transport=ASGITransport(app=test_app),
        base_url="http://test",
    ) as client:
        yield client

    await container.close()
```

### 2.2 fixture 의존 그래프

```
async_engine
    │
    ▼
session_factory ───► event_bus
    │                   │
    └─► async_client ◄──┘
            │
            ▼
        통합 테스트 함수
```

`async_client` 가 모든 인프라를 묶어주는 진입점. 통합 테스트는 `async def test_x(async_client):` 만 받으면 됨.

### 2.3 매 테스트가 깨끗한 상태인 이유

- `async_engine` : 매 테스트마다 새 SQLite 메모리 DB → drop_all + create_all → 데이터 0
- `event_bus` : 매 테스트마다 새 인스턴스 → 등록된 핸들러 0
- `async_client` 안에서 register_event_handlers 다시 실행 → 깨끗한 등록

→ pytest fixture scope 가 기본 `function` 이라서 자동으로 격리됨. **fixture scope 를 `module` 이나 `session` 으로 바꾸면 격리 깨짐** (§4.1 함정).

---

## §3 비결정적 외부 처리 — FakeGateway random

### 3.1 문제

ShopTracker 의 `FakePaymentGateway` :

```python
class FakePaymentGateway:
    async def process(self, payment) -> GatewayResult:
        await asyncio.sleep(0.1)
        if random.random() < 0.9:           # 90% 승인
            return GatewayResult(success=True, ...)
        return GatewayResult(success=False, ...)
```

10% 확률로 거절. 테스트가 :
- 결제 *승인* 을 기대하는데 거절되면 → assertion 실패
- 결제 *거절* 을 검증하고 싶은데 매번 승인되면 → 테스트 의미 없음

**같은 테스트가 어떤 날은 통과 어떤 날은 실패** — flaky test 의 전형.

### 3.2 패턴 A — 재시도 + skip

```python
@pytest.mark.asyncio
async def test_premium_subscriber_gets_10_percent_discount(async_client):
    await _create_subscription(async_client, "프리미엄", "premium")

    for _ in range(5):
        order_id = await _create_order_for(async_client, "프리미엄", 100000)
        payment = (await async_client.get(
            f"/api/v1/payments/order/{order_id}")).json()
        if payment["status"] == "approved":
            break
    else:
        pytest.skip("FakeGateway 5회 거절 — 매우 드문 케이스")

    assert payment["discount_amount"] == 10000
```

작동 원리 :
- 90% 승인이면 5 회 시도 시 모두 거절 확률 = `0.1**5 = 0.00001` (10 만 분의 1)
- skip 은 *fail 이 아니라 건너뜀* — CI 의 빨간불 X
- 의도 : "이 테스트는 결제 승인 케이스를 검증하는데, 운 나쁘게 5 번 다 거절되면 이번 라운드는 검증 불가, 그러나 fail 도 아님"

언제 쓰나 :
- **승인 흐름** 을 검증하고 싶을 때 (대다수 테스트)
- 결제 거절 자체는 다른 테스트가 다룰 때

장점 : 코드 단순. monkey patch 없음.
단점 : 정말 운 나쁘면 skip → 그날의 검증 누락 (10 만 분의 1 이지만 *0 이 아님*).

### 3.3 패턴 B — monkeypatch 로 강제

```python
from unittest.mock import patch

@pytest.mark.asyncio
async def test_payment_rejection_cancels_order(async_client):
    """FakeGateway 를 거절로 강제."""

    async def always_reject(self, payment) -> GatewayResult:
        return GatewayResult(
            success=False, transaction_id=None, message="강제 거절 (테스트)")

    with patch(
        "app.payments.infrastructure.fake_gateway.FakePaymentGateway.process",
        always_reject,
    ):
        order_resp = await async_client.post("/api/v1/orders/", json={...})
        order_id = order_resp.json()["id"]

        # 거절이 보장되니 결과 검증
        order = (await async_client.get(f"/api/v1/orders/{order_id}")).json()
        assert order["status"] == "cancelled"
```

작동 원리 :
- `unittest.mock.patch` 가 클래스 메서드를 *임시로* 교체
- `with` 블록 안에서만 적용. 끝나면 원래대로
- `always_reject` 는 self 를 받음 — 메서드 패치라서 self 가 인자

언제 쓰나 :
- **거절 흐름** (보상 트랜잭션) 을 검증할 때
- 특정 상황을 *반드시* 만들어야 하는 테스트

장점 : 결정적. 100% 보장.
단점 :
- 패치 경로 (`app.payments.infrastructure.fake_gateway.FakePaymentGateway.process`) 가 *문자열* 이라 IDE refactoring 이 안 따라옴
- 클래스 위치 변경 시 패치 깨짐
- `with` 블록 들여쓰기로 코드 깊어짐

### 3.4 패턴 C (대안) — TestProvider 에서 다른 Gateway 주입

```python
class AlwaysApprovingGateway:
    async def process(self, payment) -> GatewayResult:
        return GatewayResult(success=True, transaction_id="txn_test", message="ok")

# 일부 테스트에서만 다른 Provider 사용 — 복잡
```

이걸 쓰려면 fixture 단위로 TestProvider 를 다르게 만들어야 함. ShopTracker 는 **재시도 + monkeypatch 두 패턴이 단순해서** 이걸 안 씀. 하지만 randomness 가 더 많은 곳 (예 : 추천 시스템) 이면 Provider 분기가 더 깔끔할 수 있음.

### 3.5 어떤 패턴을 언제

| 검증 의도 | 권장 패턴 | 이유 |
|---|---|---|
| 결제 승인 흐름 (대다수) | 재시도 + skip | 코드 단순, 90% 확률이라 거의 한 번에 통과 |
| 결제 거절 흐름 (소수) | monkeypatch | 거절 강제 필요. 10% 확률에 의존하면 안 됨 |
| Tracking 타임라인 (승인 전제) | 재시도 + skip | 결제 거절되면 배송이 안 만들어져 검증 불가 → skip |
| 보상 흐름 (FAILED phase) | monkeypatch | 거절 보장 필요 |

phase7 의 6 개 테스트 중 1 개 (`test_saga_failure`) 만 monkeypatch, 나머지 5 개는 재시도. 이 비율이 일반적.

---

## §4 비동기 이벤트 결과 폴링

### 4.1 문제

```python
# 주문 생성 (이벤트 발행됨)
order_resp = await async_client.post("/api/v1/orders/", json={...})
order_id = order_resp.json()["id"]

# 결제가 자동 생성됐는지 확인 — 그런데 *언제* 끝났지?
payment = await async_client.get(f"/api/v1/payments/order/{order_id}")
# → 200 OK? 아직 처리 중? 핸들러가 실패?
```

InMemoryEventBus 는 `await event_bus.publish(...)` 가 모든 핸들러를 *순차로* await 한다. 그래서 publish 가 끝나면 모든 핸들러도 끝났다 — 이론상.

하지만 실전에서 :
- HTTP POST 가 200 으로 돌아와도 publish 가 핸들러까지 *완료* 했는지 한 번 더 확인하는 게 안전
- 핸들러가 별도 트랜잭션이라 *DB 에 commit 됐는지* 까지 확인 필요
- 폴링은 더 무거운 EventBus (Outbox, Redis Pub/Sub) 도입 시 그대로 유효한 패턴

### 4.2 폴링 헬퍼 — phase7 의 `_wait_for_shipment`

```python
async def _wait_for_shipment(client, order_id: str, max_attempts: int = 10):
    """배송이 생성될 때까지 폴링. 비동기 이벤트 처리를 기다린다."""
    for _ in range(max_attempts):
        resp = await client.get(f"/api/v1/shipping/order/{order_id}")
        if resp.status_code == 200:
            return resp.json()
    return None
```

특징 :
- `max_attempts=10` — 적당한 상한. 무한 루프 방지.
- `if resp.status_code == 200` — 생성됐으면 반환.
- 못 찾으면 `None` 반환 → 호출자가 `pytest.fail` / `pytest.skip` 결정.
- **sleep 이 없음** — InMemoryEventBus 라서 publish 가 끝나면 모든 게 끝남. Outbox 라면 sleep 필요.

### 4.3 sleep 추가 시점

분산 EventBus (Outbox + 별도 워커) 로 가면 :

```python
async def _wait_for_shipment(client, order_id, max_attempts=20, sleep_ms=50):
    for _ in range(max_attempts):
        resp = await client.get(f"/api/v1/shipping/order/{order_id}")
        if resp.status_code == 200:
            return resp.json()
        await asyncio.sleep(sleep_ms / 1000)
    return None
```

`sleep` 의 함정 :
- 너무 짧으면 (10ms) → CPU 낭비
- 너무 길면 (1초) → 테스트 느려짐
- 권장 : 50~100ms × 10~20 회 = 최대 1~2 초.

### 4.4 polling vs event-based wait

```python
# polling (위 패턴)
for _ in range(10):
    if (await client.get(...)).status_code == 200:
        return
    await asyncio.sleep(0.05)

# event-based (대안)
event_received = asyncio.Event()
async def hook(_event):
    event_received.set()
event_bus.subscribe(ShipmentCreatedEvent, hook)

await client.post(...)
await asyncio.wait_for(event_received.wait(), timeout=2.0)
```

차이 :
- polling 은 *결과의 유무* 를 확인 (DB 조회). 핸들러 실패 시 영원히 안 옴.
- event-based 는 *이벤트의 도착* 을 확인. 핸들러 안에서 추가 작업 (예 : DB commit) 이 안 끝났을 수 있음.

ShopTracker 가 polling 을 택한 이유 : 통합 테스트는 *최종 상태 (DB)* 를 검증하는 게 의도. event 도착만으로는 불충분.

---

## §5 테스트 격리

### 5.1 fixture scope = function (기본)

pytest fixture scope :
- `function` : 매 테스트마다 새로 (기본값, ShopTracker 모든 fixture)
- `class` : 같은 클래스 안 테스트들 공유
- `module` : 같은 .py 안 테스트들 공유
- `session` : 전체 pytest 실행 동안 공유

ShopTracker 가 `function` 인 이유 :
- DB 가 깨끗 — 데이터 누수 X
- EventBus 가 깨끗 — 핸들러 누적 X
- container 가 깨끗 — 옛 인스턴스 잡고 있지 않음

비용 : 매 테스트마다 SQLite 메모리 DB + create_all → 케이스당 +50ms 정도. 통합 테스트 100 개면 +5 초. 학습 단계엔 OK.

### 5.2 격리가 깨지는 흔한 패턴

```python
# ❌ 전역 EventBus
event_bus = InMemoryEventBus()           # 모듈 레벨

@pytest.fixture
def setup():
    event_bus.subscribe(...)             # 매 테스트 누적!
    yield
```

해결 : `event_bus` fixture 자체로 만들어 매 테스트 새 인스턴스.

```python
# ❌ scope=session 의 container
@pytest.fixture(scope="session")
async def container():
    return make_async_container(...)
# 100 개 테스트가 같은 컨테이너 = 같은 session_factory = 같은 connection pool.
# 한 테스트의 connection 누수가 다른 테스트로 전염.
```

해결 : `function` scope. 비싸지만 안전.

### 5.3 import 시점 부작용

```python
# ❌ 어떤 모듈을 import 만 해도 핸들러가 등록됨
# payments/__init__.py
event_bus.subscribe(OrderCreatedEvent, ...)  # import 시 실행

# 테스트가 payments 를 import → 등록됨 → 다른 테스트에 영향
```

ShopTracker 는 **명시적 register_event_handlers** 라서 이 문제가 없다. 데코레이터 기반 (NestJS 스타일) 으로 가면 import side effect 가 생긴다.

→ 데코레이터 등록의 함정. ShopTracker 의 명시적 패턴이 테스트 격리에도 유리.

### 5.4 SubscriptionContext 의 헤더 의존

```python
@provide(scope=Scope.REQUEST)
async def subscription_context(
    self, request: Request, repo: SQLAlchemySubscriptionRepository,
) -> SubscriptionContext:
    customer_name = request.headers.get("X-Customer-Name", "guest")
    ...
```

테스트가 헤더 안 넘기면 → "guest" → 미구독 → NoDiscountPolicy.

이게 *원하는 default* 인지 검토 :
- 미구독 검증 테스트 : 헤더 안 넘김 → guest → OK ✅
- premium 검증 : 헤더 넘김 → premium ✅
- 헤더 오타 ("X-Custmoer-Name") → guest → 미묘한 fail

→ 통합 테스트 헬퍼에서 헤더를 *명시적으로* 넘기는 패턴을 쓰면 디버깅이 쉬움.

```python
async def _create_order_for(client, customer_name, amount):
    return await client.post(
        "/api/v1/orders/",
        json={...},
        headers={"X-Customer-Name": customer_name},   # 항상 명시
    )
```

phase7 의 헬퍼가 이 패턴을 따른다.

---

## §6 TestProvider vs ProdProvider

### 6.1 같게 두는 부분

| 요소 | Test | Prod | 의미 |
|---|---|---|---|
| Repository 구현 | `SQLAlchemy*Repository` | 동일 | DB 연동 *진짜로* 검증 |
| Mapper | 동일 | 동일 | round-trip 검증 |
| Handler | 동일 | 동일 | 비즈니스 로직 검증 |
| EventBus | `InMemoryEventBus` | 동일 (Phase 0) | 이벤트 흐름 검증 |
| FakeGateway | `FakePaymentGateway` | 동일 (Phase 0) | 외부 API 흉내 |
| Discount/Shipping policy | match 분기 동일 | 동일 | 정책 주입 검증 |

**왜 같게 두나?** : 통합 테스트의 가치는 *prod 에 가까운 환경* 에서 검증하는 것. 너무 다르면 통합 테스트가 prod 동작을 보장하지 않음.

### 6.2 다르게 두는 부분

| 요소 | Test | Prod | 이유 |
|---|---|---|---|
| DB | SQLite in-memory | PostgreSQL | 속도 + 격리. PostgreSQL 특화 기능 (JSONB, ARRAY) 안 쓸 때만 OK |
| Engine pool | SQLite 기본 | asyncpg pool | 속도 |
| structlog renderer | console | json (prod) | 가독성 |
| (선택) Gateway | AlwaysApprovingGateway | FakePaymentGateway | 결정성 — but ShopTracker 는 monkeypatch 채택 |

### 6.3 SQLite 의 함정

`shipping/infrastructure/models.py` 에 PostgreSQL JSONB 같은 게 있으면 SQLite 에서 안 됨. ShopTracker 는 의도적으로 :
- `events_json: Mapped[str] = mapped_column(Text, default="[]")` — JSONB 대신 Text + json.dumps
- DateTime, Numeric 등 SQLite 가 지원하는 타입만 사용

→ "테스트와 prod 의 DB 가 다를 수 있다" 는 제약이 prod 의 모델 설계까지 영향. 학습 단계엔 OK, 본격 PostgreSQL 기능 (배열, JSONB query, 윈도우 함수) 이 필요해지면 testcontainers 로 진짜 PostgreSQL 사용 검토.

---

## §7 함정 모음

### 7.1 핸들러 누적 — 가장 흔한 flaky 원인

`event_bus` fixture 가 `function` scope 가 아니거나, register 가 fixture 외부에서 호출되면 — 테스트가 진행될수록 핸들러 N 배. 한 OrderCreated 이벤트가 N 번 처리되어 결제도 N 개 생성. → assertion 실패.

→ `event_bus` 는 *항상* function scope + register 도 *항상* fixture 안.

### 7.2 trans_id / random 의 변동

테스트가 `assert payment["transaction_id"] == "txn_xxx"` 같은 식으로 예측 불가능한 값을 검증하면 매번 다름.

→ presence 만 검증 (`assert payment["transaction_id"] is not None`) 하거나, monkeypatch 로 고정.

### 7.3 datetime 의 변동

`processed_at` 같은 시간 필드. 테스트가 정확한 값을 비교하면 깨짐.

→ 13 §4.4 의 Clock Protocol 주입. ShopTracker 는 Phase 1 단계라 안 함 — 시간 정확도가 중요해지면 도입.

### 7.4 fixture 의존 순서

```python
@pytest.fixture
async def async_client(session_factory, event_bus):
    ...
```

`session_factory` 가 먼저 생성되고 `event_bus` 가 그 다음. 그런데 `event_bus` 가 `session_factory` 를 *읽지* 않아도 의존 명시는 명확하게.

### 7.5 async fixture 의 함정

```python
# pytest-asyncio mode 가 'auto' 가 아니면 — 모든 async 테스트에 @pytest.mark.asyncio 필요
[tool.pytest.ini_options]
asyncio_mode = "auto"
```

ShopTracker 는 auto 가 권장.

### 7.6 register 누락

`async_client` fixture 안에서 register_event_handlers 호출이 빠지면 — 핸들러 0 개 → publish 해도 아무 일 X → 통합 테스트가 `payment 가 생성되지 않음` 으로 실패. 디버깅 어려움.

→ fixture 의 마지막 줄에 `assert len(event_bus._handlers) > 0` 같은 sanity check 도 가능 (학습용).

### 7.7 monkeypatch 경로의 미묘함

```python
# ❌ 정의처를 패치
patch("app.payments.infrastructure.fake_gateway.FakePaymentGateway.process", ...)
# 만약 다른 곳에서 `from app.payments.infrastructure.fake_gateway import FakePaymentGateway`
# 후 이미 `gateway = FakePaymentGateway()` 인스턴스를 만들어두었다면 — 패치가 늦음.
```

ShopTracker 는 DI 가 매 요청마다 `FakePaymentGateway()` 호출 → 패치가 잘 적용. 그러나 인스턴스를 startup 에 만들어 두는 패턴이면 패치 못함.

→ DI 의 lazy / per-request 생성이 monkeypatch 친화적.

---

## §8 다른 환경과의 비교

| 환경 | 통합 테스트 패턴 |
|---|---|
| **Spring Boot** | `@SpringBootTest` + H2 in-memory + `@MockBean` 으로 외부 교체 |
| **NestJS** | `Test.createTestingModule()` + `overrideProvider` |
| **Django** | `TestCase` + sqlite + `mock.patch` |
| **Rails** | RSpec request specs + factory_bot + database_cleaner |
| **Go** | testing + dockertest 로 진짜 PostgreSQL 컨테이너 |
| **Python (ShopTracker)** | pytest + Dishka TestProvider + sqlite + monkey patch |

공통 패턴 :
1. **DI 컨테이너 교체** — Mock/Fake/in-memory 주입
2. **DB 격리** — in-memory 또는 컨테이너 + reset
3. **외부 API 격리** — Mock 또는 Fake
4. **fixture scope = test** — 격리 우선

ShopTracker 의 패턴은 Python 표준이며 다른 언어로 옮겨도 거의 그대로 적용 가능.

---

## §9 학습 포인트 (한 줄 요약)

1. **TestProvider 누적 갱신** — phase 진행마다 메서드 추가, 한 클래스로 단일 진리.
2. **fixture scope = function** — 매 테스트 새 DB / event_bus / container.
3. **재시도 + skip** = 승인 흐름 검증의 단순한 패턴 (90% 확률에 의존).
4. **monkeypatch** = 거절 흐름 강제 (보상 트랜잭션 검증).
5. **폴링 헬퍼** = 비동기 이벤트 결과 대기. InMemory 면 sleep 불필요.
6. **헤더 명시** — `X-Customer-Name` 을 항상 헬퍼에서 넘김 → 디버깅 쉬움.
7. **TestProvider ≈ ProdProvider** — DB / log / Gateway 만 다르고 핸들러 / 매퍼 / 정책은 동일.
8. **import side effect 회피** — 명시적 register 가 데코레이터보다 테스트 격리에 유리.
9. **SQLite 의 한계 인지** — PostgreSQL 특화 기능 쓰기 전엔 OK.
10. **flaky 의 첫 의심은 핸들러 누적** — fixture scope 와 register 위치 점검.

---

## 추가 참고

- 13 — 4 단 테스트 피라미드 + Fake/Mock 차이
- 03 — Dishka TestProvider 패턴
- 04 — InMemoryEventBus 의 publish/subscribe 메커니즘
- 20 — 어댑터 + REQUEST 스코프 수동 처리
- pytest-asyncio docs : https://pytest-asyncio.readthedocs.io/
- unittest.mock.patch : https://docs.python.org/3/library/unittest.mock.html#patch
- 다음 글 : `22-python-match-case.md` — 정책 분기의 핵심 문법
