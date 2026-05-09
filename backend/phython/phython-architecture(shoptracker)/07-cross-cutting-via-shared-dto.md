# 07 — 모듈 간 데이터 공유 (Shared DTO + DI 중개)

> Payments 가 "이 고객이 Premium 인가?" 를 알아야 할인을 적용할 수 있다. Subscriptions 모듈에 그 정보가 있다. Payments 가 Subscriptions 를 import 하면 결합이 생기고, 안 하면 정보를 못 얻는다. 이 딜레마를 해결하는 ShopTracker 의 패턴이 *얇은 DTO 를 shared 에 두고 DI 컨테이너가 중개* 하는 방식이다. `SubscriptionContext` 가 그 표본.

---

## §0 문제 정의 — 횡단 데이터의 함정

모듈 간 *호출* (Command) 은 04 장의 EventBus 로 해결됐다. 그런데 *읽기* 는 어떨까?

```python
# Payments 가 결제 처리 시 Premium 할인 여부를 알아야 함
class PaymentHandler:
    async def on_order_created(self, event: OrderCreatedEvent):
        # 어떻게 customer 의 subscription tier 를 알지?
        ???
        if tier == "premium":
            discount = order_total * Decimal("0.1")
        ...
```

가능한 옵션들 :

### 옵션 A — Payments 가 Subscriptions 를 import

```python
from app.subscriptions.application.queries import GetSubscriptionByCustomer
from app.subscriptions.domain.entities import Subscription

class PaymentHandler:
    def __init__(self, sub_query_handler: GetSubscriptionHandler):
        ...
    async def on_order_created(self, event):
        sub: Subscription = await self.sub_query.handle(...)
        if sub.tier == "premium": ...
```

문제 :
- **방향 결합** — Payments → Subscriptions.
- Payments 테스트에 Subscriptions 의 mock 필요.
- Subscriptions 의 entity 변경이 Payments 빌드를 깬다.

### 옵션 B — 이벤트로 통보

Subscriptions 가 변할 때마다 SubscriptionUpdatedEvent 를 발행하고 Payments 가 자기 캐시에 저장.

문제 :
- Payments 가 Subscription 캐시를 자체 관리 (복잡, 실수 위험).
- 이벤트 누락 시 stale.
- 사실 *읽기 시점* 의 진실을 알고 싶을 뿐인데 너무 무거움.

### 옵션 C — Shared DTO + DI 중개 (★ ShopTracker)

```
shared/subscription_context.py:
    @dataclass(frozen=True)
    class SubscriptionContext:
        customer_name: str
        tier: str
        is_active: bool

DI Container:
    @provide(scope=REQUEST)
    async def get_subscription_context(
        self,
        request: Request,
        sub_query: GetSubscriptionByCustomer,
    ) -> SubscriptionContext:
        # 요청에서 customer_name 추출
        # Subscriptions 모듈에서 조회
        # SubscriptionContext 로 변환하여 반환

Payments:
    class PaymentHandler:
        def __init__(self, ..., context: SubscriptionContext):
            self._context = context   # ← Subscriptions 를 모름
```

장점 :
- Payments 는 `SubscriptionContext` (shared) 만 import. Subscriptions 모듈 무관.
- Subscriptions 도 Payments 를 모름.
- **DI 컨테이너가 중개자** — 둘 사이의 다리 역할을 컨테이너가 짊어짐.

---

## §1 본질 메커니즘 — Shared Kernel + Anti-Corruption Layer

### 1.1 Shared Kernel (DDD)

> 여러 Bounded Context 가 *공유하는 작은 모델*. 변경 시 모든 컨텍스트의 합의 필요.

shared 폴더는 이 Shared Kernel. 단, **얇아야 함** — 너무 두꺼워지면 모든 모듈이 shared 를 통해 결합.

ShopTracker 의 shared 구성 :
```
shared/
├── event_bus.py            (인프라)
├── events.py               (계약 — 모든 이벤트 타입)
├── value_objects.py        (Money — 도메인 원시값)
├── subscription_context.py (★ 횡단 read DTO)
├── di_container.py         (인프라)
└── ...
```

### 1.2 Anti-Corruption Layer (ACL)

> *외부 (다른 컨텍스트) 의 모델을 그대로 들이지 않고, 우리 컨텍스트의 용어로 번역하는 계층.*

DI Provider 의 변환 단계 (`Subscription` entity → `SubscriptionContext` DTO) 가 ACL 역할 :

```
Subscriptions Domain        Shared DTO            Payments
┌──────────────┐           ┌──────────────┐
│ Subscription │  변환     │ Subscription │  주입
│  - id        │ ─────►   │ Context      │ ──────►  PaymentHandler
│  - tier      │           │  - tier      │
│  - expires_at│           │  - is_active │
│  - history[] │           │  - customer  │
│  - ...       │           └──────────────┘
└──────────────┘
   복잡한 entity         얇은 read DTO
```

ACL 의 효과 :
- Payments 는 Subscription 의 *모든* 필드를 보지 않음. 필요한 3 개만.
- Subscription 에 새 필드 추가 시 → SubscriptionContext 는 영향 없음.
- 이름 / 단위 / 표현이 달라져도 변환 단계에서 흡수.

---

## §2 ShopTracker 코드 정독

### 2.1 SubscriptionContext

```python
# shared/subscription_context.py:18-29
from dataclasses import dataclass

@dataclass(frozen=True)
class SubscriptionContext:
    customer_name: str
    tier: str          # "none", "basic", "premium"
    is_active: bool

    @classmethod
    def guest(cls, customer_name: str = "guest") -> "SubscriptionContext":
        """미구독 고객용 기본값."""
        return cls(customer_name=customer_name, tier="none", is_active=False)
```

읽어보기 :

- `frozen=True` — 불변. 한 요청 안에서 컨텍스트가 바뀌지 않게 보장.
- `customer_name: str`, `tier: str`, `is_active: bool` — **3 개 필드만**. Subscription entity 의 expires_at, history 등은 노출 X.
- `tier: str` — Enum 이 아닌 str. 이유 :
  - Enum 은 모듈 간 공유 시 import 가 필요. shared 에 두면 OK 지만 더 무거움.
  - 단, 타입 안전성은 떨어짐 — "primium" 같은 오타 컴파일 시 못 잡음.
  - 트레이드오프 : 학습 단계라 단순 str 채택.
- `@classmethod guest()` — **Null Object 패턴**. 구독 없는 고객을 None 으로 다루지 않고 "tier='none', is_active=False" 인 *유효한 컨텍스트* 로 다룸. Payments 는 None 체크 없이 항상 컨텍스트를 받음.

### 2.2 Provider — DI 컨테이너 중개

ShopTracker 의 `di_container.py` 에서 (Phase 2 도입 예정 지점) :

```python
# 의도적 예시 — 실제 구현은 phase 2
class PaymentsProvider(Provider):
    @provide(scope=Scope.REQUEST)
    async def subscription_context(
        self,
        request: Request,                       # FastAPI Request 주입
        sub_query: SubscriptionByCustomerQuery,
    ) -> SubscriptionContext:
        customer_name = request.path_params.get("customer_name") \
                     or request.query_params.get("customer_name") \
                     or "guest"
        try:
            sub = await sub_query.handle(customer_name=customer_name)
            return SubscriptionContext(
                customer_name=sub.customer_name,
                tier=sub.tier.value,           # Enum → str
                is_active=sub.is_active(),
            )
        except SubscriptionNotFoundError:
            return SubscriptionContext.guest(customer_name)
```

핵심 :

- `scope=REQUEST` — **요청마다 한 번** 만들어지고 같은 요청 안에서는 재사용. 한 요청 처리 중에 N 번 조회되지 않음.
- `request: Request` — 컨텍스트 추출의 출처. JWT, header, path param 등 어디서든.
- `sub_query: SubscriptionByCustomerQuery` — Subscriptions 모듈의 Query handler. **이 Provider 가 두 모듈의 다리**.
- `try/except SubscriptionNotFoundError` — 없으면 guest. ACL 에서 "외부 도메인의 예외" 도 흡수.
- 반환 타입 `SubscriptionContext` — Payments 가 받는 타입. 이게 계약.

### 2.3 사용처 — PaymentHandler

```python
# 의도적 예시
class PaymentHandler:
    def __init__(
        self,
        repo: PaymentRepositoryProtocol,
        event_bus: EventBus,
        subscription: SubscriptionContext,    # ← Subscriptions 를 모름
        pg: PaymentGateway,
    ):
        self._repo = repo
        self._event_bus = event_bus
        self._subscription = subscription
        self._pg = pg

    async def on_order_created(self, event: OrderCreatedEvent) -> None:
        # 할인 정책 — context 만 본다
        discount_rate = self._discount_for(self._subscription.tier)
        final = event.total_amount * (Decimal("1") - discount_rate)

        result = await self._pg.charge(amount=final, ...)
        ...

    @staticmethod
    def _discount_for(tier: str) -> Decimal:
        return {
            "premium": Decimal("0.10"),
            "basic":   Decimal("0.05"),
            "none":    Decimal("0"),
        }.get(tier, Decimal("0"))
```

`PaymentHandler` 는 :
- import 에 `from app.subscriptions...` 없음.
- 생성자에 `SubscriptionContext` 만 받음.
- **모듈 간 결합 0**.

### 2.4 양쪽이 바뀔 때

| 변화 | Payments 영향 |
|---|---|
| Subscription 에 `discount_history` 필드 추가 | **무영향** — context 에 안 들어감 |
| Subscription tier 에 "enterprise" 추가 | context.tier 에 추가 → Payments 가 알게 됨 (의도적) |
| Subscription entity 가 dict 기반 → dataclass 로 리팩 | Provider 의 변환만 수정 — Payments 무영향 |
| Subscription 이 다른 마이크로서비스로 분리 | Provider 가 HTTP 호출로 바뀜 — Payments 무영향 |

이게 ACL 의 진짜 가치 — *외부의 변화가 우리에게 새지 않는다*.

---

## §3 직접 구현 — 미니 ACL

```python
# 시나리오 : Reports 모듈이 User 의 정보를 살짝 알아야 함

# 1. 외부 모듈 (Users)
class User:
    id: UUID
    name: str
    email: str
    encrypted_password: str
    created_at: datetime
    last_login_at: datetime
    address: Address
    payment_methods: list[PaymentMethod]
    # ... 25 개 필드

# 2. Shared DTO (Reports 가 보고 싶어하는 *최소* 정보)
@dataclass(frozen=True)
class UserContext:
    id: UUID
    display_name: str       # name 을 가공 (예 : "Alice (verified)")
    is_verified: bool       # last_login_at 으로 파생

# 3. Provider (ACL)
@provide(scope=Scope.REQUEST)
async def user_context(self, request: Request, user_query: UserQuery) -> UserContext:
    user_id = current_user_id_from_jwt(request)
    user = await user_query.find_by_id(user_id)
    return UserContext(
        id=user.id,
        display_name=f"{user.name}" + (" (verified)" if user.is_verified() else ""),
        is_verified=user.is_verified(),
    )

# 4. Reports 사용
class ReportHandler:
    def __init__(self, user_ctx: UserContext, ...):
        self._user_ctx = user_ctx
    async def generate(self):
        title = f"Report for {self._user_ctx.display_name}"
        ...
```

Reports 는 25 필드의 User 를 모르고 3 필드의 UserContext 만 안다.

---

## §4 함정

### 4.1 DTO 비대화

`SubscriptionContext` 가 처음엔 3 필드였는데 점점 늘어 20 필드가 되면 ACL 의 의미 상실. 사실상 entity 의 복사본. 신호가 보이면 :
- 진짜로 다 필요한가? 컨텍스트 분리 (`SubscriptionBillingContext`, `SubscriptionUiContext`).
- 아니면 Subscriptions 모듈 자체를 더 잘게 쪼갤 신호.

### 4.2 Provider 가 무거워짐

매 요청마다 Provider 가 DB 조회 → N+1 문제 가능성. 같은 요청 내 캐싱은 `scope=REQUEST` 로 자동 (한 요청에 한 번만 만들어짐). 요청 간 캐싱 (예 : Redis) 은 별도.

### 4.3 변환 누락

Subscription 에 새 tier "enterprise" 가 생겼는데 Provider 의 매핑을 업데이트 안 하면 → Payments 가 "enterprise" 를 못 봄. 컴파일 시 못 잡음.

해결 :
- Tier 를 Enum 으로 (shared 에 정의) → 매핑 누락 시 런타임 에러.
- 통합 테스트로 흐름 검증.

### 4.4 Shared 가 Subscription 도메인을 import

```python
# anti-pattern — shared/subscription_context.py 에서
from app.subscriptions.domain.entities import Subscription   # ← NO

@dataclass
class SubscriptionContext:
    @classmethod
    def from_subscription(cls, sub: Subscription) -> "SubscriptionContext":
        ...
```

shared 가 Subscriptions 모듈을 import 하면 의존이 *역방향* 으로 생김 (다른 모듈은 shared 를 import → 결국 Subscriptions 도 import 한 셈).

해결 : 변환 로직은 **Provider 안** (Subscriptions 모듈을 import 해도 OK 인 곳) 에서. shared 의 DTO 는 순수.

### 4.5 Null Object vs Optional

`SubscriptionContext.guest()` 는 Null Object. 대안은 `SubscriptionContext | None`. 비교 :

| | Null Object (`guest()`) | Optional (`None`) |
|---|---|---|
| 사용 측 | `if ctx.tier == "none"` | `if ctx is None` |
| 안전성 | 항상 객체, NPE 없음 | None 체크 누락 위험 |
| 표현력 | "guest 도 유효한 사용자" | "구독이 없음" 이 명시적 |

ShopTracker 는 Null Object 채택 — Payments 가 `None` 체크 없이 흐름을 짤 수 있게.

### 4.6 Context 가 stale

요청 시작 시점에 Provider 가 만든 context 가 요청 처리 중에 사용자가 구독을 업그레이드 → context 는 옛 값. 보통 OK (한 요청은 시작 시점의 진실로 처리), 단 명시해야.

---

## §5 다른 패턴과의 비교

### 5.1 vs DDD Bounded Context Map

DDD 의 *Context Map* 패턴 들 :
- **Shared Kernel** : 작은 공유 모델 — ShopTracker 의 shared/.
- **Customer/Supplier** : 한 컨텍스트가 다른 쪽에 종속.
- **Anticorruption Layer** : 외부를 우리 용어로 변환 — Provider 가 ACL.
- **Open Host Service** : 공식 API 제공 — REST / gRPC.
- **Published Language** : 합의된 메시지 포맷 — events.py.

ShopTracker 는 Shared Kernel + ACL 조합.

### 5.2 vs gRPC / REST

마이크로서비스라면 모듈 호출 = HTTP. SubscriptionContext 는 그 응답 DTO. ShopTracker 의 패턴은 *모놀리스에서 마이크로서비스로의 점진 진화 경로* 를 그대로 닮음 — Provider 의 내부 호출만 HTTP 호출로 바꾸면 마이크로서비스 모드.

### 5.3 vs GraphQL Federation

GraphQL Federation 은 여러 서비스의 schema 를 합쳐 한 client 에 노출. 각 entity 의 필드를 어느 서비스가 책임지는지 메타데이터로 표현. ShopTracker 의 SubscriptionContext 는 backend-internal 버전의 같은 사상 — *각 도메인이 자기 데이터 책임* + *외부는 얇은 view 만*.

---

## §6 테스트 전략

### 6.1 PaymentHandler 단위 — context 를 fake 로

```python
async def test_premium_gets_10_percent_discount():
    handler = PaymentHandler(
        repo=FakePaymentRepo(),
        event_bus=FakeBus(),
        subscription=SubscriptionContext(
            customer_name="alice",
            tier="premium",
            is_active=True,
        ),
        pg=FakePG(approve=True),
    )
    await handler.on_order_created(OrderCreatedEvent(
        order_id=uuid4(), customer_name="alice",
        total_amount=Decimal("100"), items_count=1,
        timestamp=datetime.now(UTC),
    ))
    saved = handler._repo.saved[0]
    assert saved.discount_amount == Decimal("10")
```

- Subscriptions 모듈 mock 필요 없음.
- 다양한 tier 케이스 = context 만 바꿔 주입.

### 6.2 Provider 통합 테스트

```python
async def test_subscription_context_provider_returns_premium():
    container = make_container([SubscriptionsProvider(), TestPaymentsProvider()])
    seed_subscription(customer_name="alice", tier="premium")

    async with container() as request_container:
        ctx = await request_container.get(SubscriptionContext)
        assert ctx.tier == "premium"
        assert ctx.is_active is True


async def test_subscription_context_provider_returns_guest_when_missing():
    container = make_container([SubscriptionsProvider(), TestPaymentsProvider()])

    async with container() as request_container:
        ctx = await request_container.get(SubscriptionContext)
        assert ctx.tier == "none"
        assert ctx.is_active is False
```

ACL 동작 자체를 검증.

---

## §10 학습 포인트 (한 줄 요약)

1. **모듈 간 *호출* 은 EventBus, *읽기* 는 Shared DTO + DI 중개**.
2. **Shared Kernel** : shared 폴더는 *얇은* 공유 모델만. 두꺼워지면 결합 폭발.
3. **Anti-Corruption Layer** : Provider 의 변환 단계가 외부 모델을 우리 용어로 번역.
4. **DTO 는 frozen + 최소 필드** — entity 의 부분집합. 노출 면적을 의도적으로 줄임.
5. **Null Object** (`guest()`) — None 대신 유효한 기본 컨텍스트로 흐름 단순화.
6. **scope=REQUEST** — 같은 요청 안에서 1 회만 조회, 자동 캐싱.
7. **변환 로직은 Provider 안** — shared 가 다른 모듈을 import 하지 않게.
8. **DTO 가 비대화하면 컨텍스트 분리** 또는 모듈 재설계 신호.
9. **마이크로서비스 진화 경로** — Provider 의 내부 호출을 HTTP/gRPC 로 바꾸면 그대로 분산화.
10. **테스트가 단순** — Payments 단위 테스트에 Subscription mock 불필요.

---

## 추가 참고

- Eric Evans, *Domain-Driven Design*, Context Map 챕터
- Vaughn Vernon, *Implementing DDD*, Anti-Corruption Layer
- Martin Fowler, *Bounded Context*, https://martinfowler.com/bliki/BoundedContext.html
- ShopTracker 다음 글 : `08-aggregate-state-machine.md` (Order 의 상태머신)
