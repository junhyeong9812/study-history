# 03. DI + Dishka + 정책 주입

> 이 문서가 다루는 것: Dishka DI 컨테이너의 본질, Provider 분리, Scope (APP/REQUEST), 정책 주입의 의미.
> 전제: 02-dip-protocols.md 의 Protocol.

---

## §0. DI 가 무엇인가?

### 0.1 manual instantiation 의 문제
```python
# ❌ 매번 새로 생성
async def create_order_endpoint(request: ...):
    engine = create_engine(...)
    session_factory = sessionmaker(engine)
    session = session_factory()
    repo = SQLAlchemyOrderRepository(session)
    bus = InMemoryEventBus()
    handler = CreateOrderHandler(repo, bus)
    # ...
```

**문제**:
- 매 endpoint 마다 boilerplate.
- session lifecycle 직접 관리 (commit/rollback/close).
- bus singleton 보장 X (매번 새 인스턴스 = 이벤트 발행 분리됨).
- 테스트 시 인스턴스 교체 어려움.

### 0.2 DI 의 통찰
**컨테이너가 객체 생성 + lifecycle + 주입 자동**:
- 의존 그래프 선언 (handler 가 repo 의존, repo 가 session 의존, ...).
- 컨테이너가 순서대로 생성.
- scope 별 lifetime (request 마다 새 session, app 동안 1 bus).
- 테스트 시 provider 교체.

### 0.3 ShopTracker 의 선택 — Dishka

**Dishka** (러시아어 "공") = Python 의 modern DI 라이브러리. async 친화 + scope 명확.

대안:
- `dependency-injector` — old, sync 위주.
- FastAPI 의 `Depends` — 간단하지만 컨테이너 X.
- Wireup — 가벼운 대안.

ShopTracker = Dishka. async + scope + Provider 분리 강력.

---

## §1. Dishka 의 핵심 — Provider + Scope

### 1.1 Provider
```python
from dishka import Provider, Scope, provide

class AppProvider(Provider):
    @provide(scope=Scope.APP)
    def config(self) -> AppConfig:
        return get_config()

    @provide(scope=Scope.APP)
    def event_bus(self) -> EventBus:
        return InMemoryEventBus()
```

`@provide(scope=...)` 데코레이터 = "이 메서드가 X 타입 인스턴스 생성".

### 1.2 Scope 의 의미

| Scope | lifetime | 예 |
|---|---|---|
| `APP` | 어플리케이션 동안 1개 (singleton) | config, event_bus, engine |
| `REQUEST` | HTTP 요청마다 새 인스턴스 | session, repository, handler |

→ session 은 매 요청마다 새로 (commit/rollback 격리). event_bus 는 1개 (subscriber 공유).

### 1.3 의존 그래프 자동 해석
```python
@provide(scope=Scope.APP)
def engine(self, config: AppConfig) -> AsyncEngine:
    return create_engine(config)
#                        ↑
# config 가 필요 — 컨테이너가 자동으로 위 config() 호출 결과 주입.
```

→ 매개변수 타입 hint 가 의존 선언. Dishka 가 그래프 따라 생성.

### 1.4 ShopTracker 의 전체 의존 그래프

```
[AppProvider]
   config: AppConfig                           (APP)
   engine: AsyncEngine ← config                (APP)
   session_factory: async_sessionmaker ← engine  (APP)
   event_bus: EventBus                         (APP)

   session: AsyncSession ← session_factory     (REQUEST, async generator)

[OrdersProvider]
   order_repository ← session                   (REQUEST)
   create_order_handler ← order_repository, event_bus    (REQUEST)
   get_order_handler ← order_repository         (REQUEST)
   list_orders_handler ← order_repository       (REQUEST)

[SubscriptionsProvider]
   subscription_repository ← session            (REQUEST)
   create_subscription_handler ← subscription_repository, event_bus  (REQUEST)
   ...
```

요청 흐름:
1. POST /orders/ 도착.
2. Dishka 가 REQUEST scope 시작.
3. handler 필요 → handler 의존 (repo, event_bus) 생성.
4. repo 의존 (session) 생성.
5. handler 호출.
6. 응답 후 REQUEST scope 종료 → session commit/close.

---

## §2. async session — generator 기반 lifecycle

### 2.1 코드
```python
@provide(scope=Scope.REQUEST)
async def session(
    self, factory: async_sessionmaker[AsyncSession]
) -> AsyncSession:
    async with factory() as session:
        try:
            yield session                  # ← yield 로 인스턴스 제공
            await session.commit()         # ← 정상 종료 시 commit
        except Exception as e:
            await session.rollback()       # ← 예외 시 rollback
            raise
```

### 2.2 핵심 통찰
**async generator 가 lifecycle hook 표현**:
- `yield session` 위 = setup.
- yield = 인스턴스 사용 (handler 안에서).
- `yield` 아래 = teardown (commit/rollback).

→ Python 의 `async with` + generator = 자동 lifecycle 관리.

### 2.3 Spring 비교
- Spring 의 `@Transactional` = AOP proxy 가 commit/rollback.
- Dishka 의 generator = 코드로 명시.

---

## §3. Provider 분리 — 모듈별 격리

### 3.1 ShopTracker 의 3 Provider
```python
class AppProvider(Provider):       # 공통 (config, engine, session, bus)
    ...

class OrdersProvider(Provider):    # orders 모듈
    ...

class SubscriptionsProvider(Provider):  # subscriptions 모듈
    ...

def create_container() -> AsyncContainer:
    return make_async_container(
        AppProvider(), OrdersProvider(), SubscriptionsProvider(),
    )
```

### 3.2 의도
- 모듈별 책임 분리.
- 새 모듈 추가 시 새 Provider 클래스 생성.
- Provider 간 의존 OK (예: OrdersProvider 가 AppProvider 의 session 의존).

### 3.3 v0.2 확장
```python
class PaymentsProvider(Provider):
    @provide(scope=Scope.APP)
    def payment_gateway(self) -> PaymentGatewayProtocol:
        return FakePaymentGateway()        # 또는 RealPaymentGateway()
    # ...

# main.py
make_async_container(
    AppProvider(), OrdersProvider(), SubscriptionsProvider(),
    PaymentsProvider(),                    # 추가
)
```

→ 새 모듈 = 새 Provider. 기존 코드 변경 0.

---

## §4. FastAPI 와 통합 — `FromDishka`

### 4.1 router 코드
```python
from dishka.integrations.fastapi import FromDishka

@router.post("/")
async def create_order(
    request: CreateOrderRequest,
    handler: FromDishka[CreateOrderHandler],    # ← Dishka 가 자동 주입
) -> CreateOrderResponse:
    order_id = await handler.handle(...)
    return CreateOrderResponse(order_id=order_id)
```

### 4.2 동작
1. FastAPI 가 endpoint 호출.
2. `FromDishka[CreateOrderHandler]` 발견 → Dishka container 에 요청.
3. Dishka 가 의존 그래프 따라 생성:
   - session ← factory.
   - order_repository ← session.
   - create_order_handler ← order_repository, event_bus.
4. handler 인스턴스 주입.
5. endpoint 함수 실행.
6. 응답 후 session commit + close.

### 4.3 FastAPI 의 `Depends` 와 비교

| | FastAPI Depends | Dishka FromDishka |
|---|---|---|
| 의존 선언 | `Depends(get_db)` | type hint (`FromDishka[X]`) |
| scope | request 만 | APP / REQUEST 등 |
| 컨테이너 | X (함수 chain) | container 객체 |
| async generator | OK | OK |
| 모듈 분리 | 어려움 | Provider 분리 자연 |

→ 작은 앱 = `Depends` 충분. 큰 앱 = Dishka 같은 컨테이너.

---

## §5. 정책 주입 — 같은 코드, 다른 행동

### 5.1 시나리오 (Phase 2 예정)
"Premium 구독자는 결제에 10% 할인 + 배송비 무료. Basic 은 5% + 배송비 절반."

같은 결제 흐름인데 정책이 다름.

### 5.2 naive (조건부 코드)
```python
# ❌ handler 안에 분기
async def handle(self, command):
    sub = await get_subscription(command.customer)
    if sub.tier == "premium":
        discount = total * 0.1
        shipping = 0
    elif sub.tier == "basic":
        discount = total * 0.05
        shipping = base_shipping / 2
    else:
        discount = 0
        shipping = base_shipping
```

**문제**:
- 정책 추가 (Gold tier 등) 시 handler 수정.
- 정책 변경 시 handler 수정.
- 정책 테스트가 handler 와 결합.

### 5.3 정책 객체 + DI
```python
class DiscountPolicy(Protocol):
    def calculate(self, total: Money) -> Money: ...

class NoneDiscount:
    def calculate(self, total: Money) -> Money: return Money(Decimal("0"))

class BasicDiscount:
    def calculate(self, total: Money) -> Money: return total.apply_rate(Decimal("0.05"))

class PremiumDiscount:
    def calculate(self, total: Money) -> Money: return total.apply_rate(Decimal("0.1"))

# Provider 가 구독 tier 따라 정책 인스턴스 결정
class PaymentsProvider(Provider):
    @provide(scope=Scope.REQUEST)
    def discount_policy(
        self, sub_context: SubscriptionContext
    ) -> DiscountPolicy:
        match sub_context.tier:
            case "premium": return PremiumDiscount()
            case "basic": return BasicDiscount()
            case _: return NoneDiscount()

# handler 는 정책만 의존
class ProcessPaymentHandler:
    def __init__(self, repo, policy: DiscountPolicy):
        self._policy = policy

    async def handle(self, command):
        discount = self._policy.calculate(total)
        # ...
```

### 5.4 효과
- 정책 추가 = 새 클래스 + Provider 분기.
- handler 변경 0.
- 각 정책 단위 테스트 가능.

→ Strategy pattern + DI 결합. ShopTracker 의 핵심 학습 목표 #3.

---

## §6. test 시 Container 교체

### 6.1 test container
```python
# tests/conftest.py
class FakeOrdersProvider(Provider):
    @provide(scope=Scope.REQUEST)
    def order_repository(self) -> FakeOrderRepository:
        return FakeOrderRepository()

    @provide(scope=Scope.REQUEST)
    def create_order_handler(
        self, repo: FakeOrderRepository, event_bus: EventBus
    ) -> CreateOrderHandler:
        return CreateOrderHandler(repo, event_bus)

@pytest.fixture
async def test_container():
    return make_async_container(
        AppProvider(), FakeOrdersProvider(),    # Fake 로 교체
    )
```

### 6.2 효과
- 진짜 DB 0.
- handler 의 Protocol 의존이 자동으로 Fake 로 매칭.

---

## §7. Dishka 의 다른 기능

### 7.1 generic provider
```python
T = TypeVar("T")

class GenericProvider(Provider, Generic[T]):
    @provide
    def service(self) -> Service[T]:
        return Service()
```

### 7.2 component grouping
- 같은 type 여러 인스턴스 (예: 두 DB) → component label.

### 7.3 lazy initialization
- 사용 시점에 생성 (REQUEST scope 가 자연 lazy).

### 7.4 cleanup hooks
- Provider 의 generator 가 yield 후 코드 = cleanup.
- session 의 commit/rollback/close.

---

## §8. 다른 DI 라이브러리 비교

### 8.1 Spring (Java)
```java
@Component
class CreateOrderService {
    @Autowired
    private OrderRepository repository;
}
```
→ 어노테이션 기반 자동 등록. Dishka 의 명시 Provider 와 다름.

### 8.2 FastAPI Depends
```python
def get_repo(session: AsyncSession = Depends(get_session)):
    return SqlAlchemyOrderRepository(session)

@router.post("/")
async def create_order(repo: SqlAlchemyOrderRepository = Depends(get_repo)):
    ...
```
→ 함수 기반 chain. 작은 앱 적합.

### 8.3 NestJS (TypeScript)
```typescript
@Injectable()
class CreateOrderService {
    constructor(@Inject('OrderRepository') private repo: OrderRepository) {}
}

@Module({
    providers: [{ provide: 'OrderRepository', useClass: SqlOrderRepository }],
})
export class OrdersModule { }
```
→ Spring 비슷. 어노테이션 + 모듈 시스템.

### 8.4 dependency-injector (Python)
```python
class Container(containers.DeclarativeContainer):
    config = providers.Configuration()
    repo = providers.Singleton(SqlAlchemyOrderRepository, session=...)
```
→ Dishka 와 비슷. but sync 위주, async 약함.

---

## §9. 함정

### 9.1 Scope 잘못
- session 을 APP scope 로 → 모든 request 가 같은 session 공유 → race + transaction 격리 깨짐.
- repository 를 APP 으로 → 같은 session 재사용 → 같은 문제.

→ DB session 은 항상 REQUEST.

### 9.2 circular dependency
- A → B, B → A 의존. Dishka 가 감지 못 하고 stack overflow.
- 해결: 한쪽이 다른 쪽 의존 끊기 또는 lazy.

### 9.3 Provider 너무 많은 책임
- 한 Provider 에 100 메서드 → 가독성 X.
- 모듈별 분리 (OrdersProvider, PaymentsProvider).

### 9.4 직접 인스턴스화
- 코드 어딘가에서 `CreateOrderHandler(repo, bus)` 직접 호출 → DI 우회. 원칙 깨짐.
- DI 컨테이너만 통하기.

---

## §10. 학습 포인트

1. **DI = 컨테이너가 lifecycle 관리** — manual 0.
2. **Dishka Provider + Scope** = lifetime 명시 (APP/REQUEST).
3. **async generator session** = yield + try/except commit/rollback.
4. **Provider 분리 = 모듈별** — 새 모듈 = 새 Provider.
5. **`FromDishka[X]` type hint** = FastAPI 와 통합.
6. **정책 주입** = Protocol + Provider 분기 = Strategy + DI.
7. **test container** = Provider 교체 (Fake).
8. **session 은 항상 REQUEST scope** — 격리.
9. **circular dependency 회피** — 한쪽 끊기.
10. **직접 인스턴스화 금지** — 컨테이너만.

### 추가 참고
- Dishka: https://github.com/reagento/dishka
- Robert C. Martin, *Clean Architecture* — DIP + DI.
- FastAPI Depends: https://fastapi.tiangolo.com/tutorial/dependencies/
- 다음: [04-event-bus.md](04-event-bus.md) — 모듈 간 통신.
