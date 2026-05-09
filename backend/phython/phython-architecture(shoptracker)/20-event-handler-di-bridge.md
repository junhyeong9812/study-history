# 20 — 이벤트 핸들러와 DI 컨테이너 다리 놓기

> 03 장은 DI 가 HTTP 요청에 어떻게 의존성을 주입하는지를 다뤘다. 04 장은 EventBus 가 어떻게 모듈 간 결합 없이 통신하는지를 다뤘다. 그러나 *둘이 만나는 지점* — 이벤트 핸들러가 의존성을 어떻게 받는가 — 는 두 문서 어느 곳에서도 본격적으로 다뤄지지 않는다. 이 글은 ShopTracker 의 `main.py` 의 `register_event_handlers` 가 왜 그렇게 생겼는지, 어댑터 패턴과 REQUEST 스코프 수동 처리가 왜 필요한지를 정리한다.

---

## §0 문제 정의 — 이벤트 핸들러의 의존성 딜레마

### 0.1 평범한 핸들러의 모습

ShopTracker 의 `payments/application/event_handlers.py` :

```python
async def handle_order_created(
    event: OrderCreatedEvent,
    handler: ProcessPaymentHandler,
) -> None:
    await handler.handle(
        ProcessPaymentCommand(
            order_id=str(event.order_id),
            amount=event.total_amount,
            customer_name=event.customer_name,
            method="credit_card",
        )
    )
```

핸들러 함수는 `(event, dependencies...)` 시그니처. `ProcessPaymentHandler` 같은 의존성을 *주입받는다*.

### 0.2 EventBus 의 시그니처

`shared/event_bus.py` 의 `subscribe` 와 `publish` :

```python
def subscribe(self, event_type: type, handler: Callable) -> None:
    self._handlers[event_type].append(handler)

async def publish(self, event: object) -> None:
    ...
    for handler in handlers:
        await handler(event)        # ← (event,) 한 개만 넘긴다
```

EventBus 는 **`(event,)` 한 개만 넘기는 callable** 만 알고 있다. 의존성 주입 같은 건 모른다 — 알 필요도 없다 (그게 단순함의 핵심).

### 0.3 충돌

```python
event_bus.subscribe(OrderCreatedEvent, handle_order_created)
# ...
await event_bus.publish(OrderCreatedEvent(...))
# → handle_order_created(event)  ← 인자 1개만 넘김
# → 함수는 (event, handler: ProcessPaymentHandler) 를 기대
# → TypeError: missing 1 required positional argument: 'handler'
```

핸들러 함수는 의존성을 받고 싶은데, EventBus 는 의존성을 모른다. **누군가가 EventBus 와 핸들러 함수 사이에서 의존성을 채워넣어야 한다.**

### 0.4 추가 문제 — Dishka REQUEST 스코프

가장 단순한 해결 — "DI 컨테이너에서 의존성을 꺼내자" — 가 *그냥 되지 않는다*.

```python
# ❌ 작동하지 않는 시도
async def adapter(event: OrderCreatedEvent):
    handler = await container.get(ProcessPaymentHandler)
    # → RuntimeError: REQUEST scope is not active
    await handle_order_created(event, handler)
```

이유 : `ProcessPaymentHandler` 는 REQUEST 스코프. Dishka 의 REQUEST 스코프는 **HTTP 요청에 묶여서 자동으로 열린다**. FastAPI + `setup_dishka(container, app)` 가 미들웨어로 REQUEST 스코프를 매 요청마다 열고 닫아준다.

그런데 이벤트 핸들러는 `await event_bus.publish(...)` 로 *명령형 코드 흐름*에서 호출된다 :
- HTTP 요청 안에서 호출될 수도 있고 (커맨드 핸들러가 publish)
- HTTP 바깥일 수도 있고 (cron job, startup hook, CLI)
- 같은 HTTP 요청이라도 *다른* REQUEST 스코프가 필요할 수도 있고 (다른 트랜잭션 경계)

→ 이벤트 핸들러가 동작하려면 **REQUEST 스코프를 어떻게든 마련해야 한다**.

이 두 문제 — 시그니처 불일치 + REQUEST 스코프 — 가 함께 풀려야 한다. 그 답이 **어댑터 + 컨텍스트 매니저** 패턴.

---

## §1 본질적 메커니즘

### 1.1 두 책임의 분리

```
[핸들러 함수]              [어댑터]              [EventBus]
(event, deps...)  ←────  (event,)  ────►  publish(event)
   ▲                        │
   │ 호출                    │ 의존성 꺼내기
   │                        ▼
   └───────────  [DI 컨테이너 (REQUEST 스코프)]
```

- **핸들러 함수**는 비즈니스 로직만 안다. 의존성을 *생성자처럼* 받는다.
- **어댑터**는 EventBus 와 핸들러 함수 사이의 변환기. `(event,)` → `(event, deps...)`.
- **DI 컨테이너**는 의존성의 실제 인스턴스를 안다.

이 분리가 깨지면 (예 : 핸들러 함수가 직접 컨테이너에 손을 뻗으면) 핸들러 자체가 컨테이너를 알게 되어 다시 결합이 생긴다.

### 1.2 컨텍스트 매니저로 REQUEST 스코프 열기

Dishka 의 `AsyncContainer` 를 *호출하면* REQUEST 스코프가 열린다 :

```python
container = create_container()      # AsyncContainer 인스턴스 — APP 스코프

# REQUEST 스코프 열기 (수동)
async with container() as request_container:    # ← container() 호출
    handler = await request_container.get(ProcessPaymentHandler)
    # ... 사용 ...
# 여기서 자동으로 닫힘 — session commit 또는 rollback 처리
```

핵심 :
- `container()` 호출 = 새 REQUEST 스코프 인스턴스 만들기
- `async with` 진입 = 스코프 시작
- `request_container.get(X)` = 해당 스코프 안에서 X 인스턴스 얻기 (의존 그래프 자동 해결)
- `async with` 종료 = 스코프 닫힘 + cleanup (예 : session.commit / rollback / close)

이게 03 장 §2.1 의 async generator 패턴의 *수동 호출 버전*. FastAPI 가 매 요청마다 자동으로 해주던 것을, 우리가 직접 한다.

### 1.3 어댑터의 역할

어댑터는 클로저 (closure) 로 만든다 — `container` 와 `event_bus` 를 캡처하는 작은 함수 :

```python
async def on_order_created(event: OrderCreatedEvent) -> None:
    async with container() as request_container:
        handler = await request_container.get(ProcessPaymentHandler)
        await payments_handle_order_created(event, handler)
#       ▲                                  ▲       ▲
#       │                                  │       │
#       └ 핸들러 함수 (실제 비즈니스 로직)    │       │
#                                          │       │
#                          (event, handler) ←─────── 시그니처 변환
```

어댑터의 4 책임 :
1. EventBus 의 `(event,)` 시그니처 받기
2. REQUEST 스코프 열기
3. 컨테이너에서 의존성 꺼내기
4. 진짜 핸들러 함수 호출 + 인자 채워넣기

어댑터 코드는 **비즈니스 로직을 한 줄도 포함하지 않는다**. 변환만 한다.

---

## §2 ShopTracker 코드 정독 — `register_event_handlers`

ShopTracker 의 `main.py` 는 Phase 2 부터 모든 라우팅을 한곳에서 등록한다. Phase 4 시점의 최종 모습 :

```python
def register_event_handlers(event_bus: EventBus, container) -> None:
    """모든 모듈의 이벤트 라우팅을 한 곳에서 등록.

    이 함수만이 "어떤 이벤트가 어떤 핸들러로 가는지" 안다.
    각 모듈은 publish/subscribe 만 알 뿐, 라우팅 정보는 모른다.
    """

    # === Phase 2 ===
    async def on_order_created(event: OrderCreatedEvent) -> None:
        async with container() as rc:
            handler = await rc.get(ProcessPaymentHandler)
            await payments_handle_order_created(event, handler)

    async def on_payment_approved_orders(event: PaymentApprovedEvent) -> None:
        async with container() as rc:
            repo = await rc.get(SQLAlchemyOrderRepository)
            await orders_handle_payment_approved(event, repo)

    async def on_payment_rejected(event: PaymentRejectedEvent) -> None:
        async with container() as rc:
            repo = await rc.get(SQLAlchemyOrderRepository)
            await orders_handle_payment_rejected(event, repo)

    # === Phase 3 ===
    async def on_payment_approved_shipping(event: PaymentApprovedEvent) -> None:
        async with container() as rc:
            fee_policy = await rc.get(ShippingFeePolicy)
            repo = await rc.get(SQLAlchemyShipmentRepository)
            eb = await rc.get(EventBus)
            await shipping_handle_payment_approved(event, fee_policy, repo, eb)

    async def on_shipment_created(event: ShipmentCreatedEvent) -> None:
        async with container() as rc:
            repo = await rc.get(SQLAlchemyOrderRepository)
            await orders_handle_shipment_created(event, repo)

    async def on_shipment_status_changed(event: ShipmentStatusChangedEvent) -> None:
        async with container() as rc:
            repo = await rc.get(SQLAlchemyOrderRepository)
            await orders_handle_shipment_delivered(event, repo)

    # === Phase 4 (Tracking 누적) ===
    async def on_order_created_tracking(event: OrderCreatedEvent) -> None:
        async with container() as rc:
            repo = await rc.get(SQLAlchemyTrackingRepository)
            await tracking_handle_order_created(event, repo)

    # ... 나머지 4 개 tracking 어댑터 ...

    # === 등록 ===
    event_bus.subscribe(OrderCreatedEvent, on_order_created)
    event_bus.subscribe(OrderCreatedEvent, on_order_created_tracking)        # 누적
    event_bus.subscribe(PaymentApprovedEvent, on_payment_approved_orders)
    event_bus.subscribe(PaymentApprovedEvent, on_payment_approved_shipping)  # 누적
    event_bus.subscribe(PaymentApprovedEvent, on_payment_approved_tracking)  # 누적
    event_bus.subscribe(PaymentRejectedEvent, on_payment_rejected)
    event_bus.subscribe(PaymentRejectedEvent, on_payment_rejected_tracking)
    event_bus.subscribe(ShipmentCreatedEvent, on_shipment_created)
    event_bus.subscribe(ShipmentCreatedEvent, on_shipment_created_tracking)
    event_bus.subscribe(ShipmentStatusChangedEvent, on_shipment_status_changed)
    event_bus.subscribe(ShipmentStatusChangedEvent, on_shipment_status_changed_tracking)
```

### 2.1 lifespan 에서의 호출

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    container = app.state.dishka_container
    event_bus = await container.get(EventBus)        # APP 스코프
    register_event_handlers(event_bus, container)
    yield
    await container.close()
```

**왜 lifespan 에서 한 번만 호출하는가?**
- `event_bus.subscribe` 는 dict 에 append. 호출할 때마다 누적된다.
- 매 요청마다 호출하면 핸들러가 N 회 등록되어 한 이벤트에 같은 처리가 N 번 일어난다 (04 §4.1 함정).
- 앱 시작 시 한 번만이 정답.

**왜 `event_bus = await container.get(EventBus)` 인가?**
- EventBus 는 APP 스코프 (모든 요청이 같은 인스턴스 공유 — subscriber 정보가 유지되어야 함).
- APP 스코프는 컨테이너 직접 호출 (`container.get`) 으로 얻을 수 있다 (REQUEST 스코프와 달리 별도 컨텍스트 매니저 필요 없음).

### 2.2 단일 진입점의 가치

이 한 함수가 *모든 라우팅의 유일한 진리원* 이다 :

| 누가 안다 | 무엇을 |
|---|---|
| `register_event_handlers` | 어떤 이벤트 → 어떤 핸들러 |
| 각 모듈의 핸들러 함수 | 자기 비즈니스 로직 |
| EventBus | 등록된 핸들러 목록 (이벤트 종류별) |
| Provider | 의존성 생성 방법 |

새 핸들러 추가 시 :
1. 핸들러 함수 작성 (해당 모듈 안)
2. 어댑터 작성 + subscribe (`register_event_handlers` 안 — 한 곳)
3. (필요시) Provider 에 의존성 추가

**라우팅이 흩어지지 않는다.** "이 이벤트가 어디서 처리되지?" 는 한 함수만 보면 된다.

### 2.3 어댑터를 외부 함수로 빼지 않는 이유

```python
# 대안 — 어댑터를 모듈 함수로
# payments/application/event_handlers.py
def make_order_created_adapter(container):
    async def adapter(event):
        async with container() as rc:
            handler = await rc.get(ProcessPaymentHandler)
            await handle_order_created(event, handler)
    return adapter

# main.py
event_bus.subscribe(OrderCreatedEvent, make_order_created_adapter(container))
```

이렇게도 짤 수 있다. ShopTracker 가 *클로저로 register 안에 두는* 이유 :

- 라우팅 정보가 흩어지지 않음 (장점이자 §2.2 의 핵심)
- main.py 만 보면 와이어링 전체가 보임
- 단점 : `register_event_handlers` 함수가 길어짐 (Phase 4 면 어댑터 11 개)

규모가 더 커지면 (예 : 어댑터 30 개) 모듈별로 분리할 가치가 생긴다 — 그 시점이 *언제* 인가는 §4.5 함정 절에서 다룸.

---

## §3 직접 구현 — 미니 어댑터

원리를 손으로 짜보면서 본질을 본다.

### 3.1 v1 — 어댑터 한 개 수동 작성

```python
from contextlib import asynccontextmanager

class MiniContainer:
    """Dishka 흉내. 매 호출마다 새 'request scope' 반환."""

    @asynccontextmanager
    async def __call__(self):
        scope = {}      # 단순화: dict 가 scope
        try:
            yield MiniRequestContainer(scope, self._providers)
        finally:
            for v in scope.values():
                if hasattr(v, "close"):
                    await v.close()

class MiniRequestContainer:
    def __init__(self, scope, providers):
        self._scope = scope
        self._providers = providers

    async def get(self, type_):
        if type_ not in self._scope:
            self._scope[type_] = await self._providers[type_]()
        return self._scope[type_]


# 사용
container = MiniContainer()
container.register(OrderRepository, lambda: SqlOrderRepository(...))
container.register(ProcessPaymentHandler, lambda: ProcessPaymentHandler(...))


async def handle_order_created(event, handler):
    """진짜 핸들러 — 의존성 받음."""
    await handler.process(event)


async def adapter(event):
    """어댑터 — EventBus 가 부르는 (event,) 시그니처."""
    async with container() as rc:
        handler = await rc.get(ProcessPaymentHandler)
        await handle_order_created(event, handler)


event_bus.subscribe(OrderCreatedEvent, adapter)
```

→ Dishka 가 해주는 일이 사실은 이 정도. 컨텍스트 매니저 + dict.

### 3.2 v2 — 어댑터 자동 생성 (메타프로그래밍)

매번 손으로 어댑터 짜는 게 귀찮으니 헬퍼 :

```python
def make_adapter(container, handler_func):
    """핸들러 함수의 시그니처를 보고 자동으로 어댑터 만들기."""
    import inspect
    sig = inspect.signature(handler_func)
    # 첫 매개변수는 event, 나머지는 의존성
    dep_params = list(sig.parameters.values())[1:]

    async def adapter(event):
        async with container() as rc:
            deps = [await rc.get(p.annotation) for p in dep_params]
            await handler_func(event, *deps)

    return adapter

event_bus.subscribe(
    OrderCreatedEvent,
    make_adapter(container, handle_order_created),
)
```

장점 : `register_event_handlers` 가 `make_adapter` 호출만 모인 짧은 함수가 됨.
단점 :
- type hint 에 의존 — string-typed import 가 못 따라옴
- 디버깅 시 추적 불편 (자동 생성 코드)
- ShopTracker 는 학습 의도로 *명시적 어댑터* 채택

### 3.3 v3 — 데코레이터 등록 (NestJS 스타일)

```python
# payments/application/event_handlers.py
@on_event(OrderCreatedEvent)
async def handle_order_created(event, handler: ProcessPaymentHandler):
    ...
```

이런 식의 데코레이터는 **import 시점에 등록** 이라 :
- 등록 위치가 흩어짐 (각 모듈이 자기 등록)
- 테스트에서 일부 핸들러만 빼고 등록하기 어려움
- 하지만 라우팅 코드가 줄어듦

ShopTracker 가 안 쓰는 이유 : 명시적 등록 (`register_event_handlers`) 이 **라우팅 가시성** 면에서 우위. 학습 자료로서도 "어디서 등록되지?" 가 한 곳에 모임.

---

## §4 함정

### 4.1 핸들러 안에서 직접 컨테이너 호출

```python
# ❌ 안티패턴
async def handle_order_created(event, container):    # 컨테이너 자체를 받음
    async with container() as rc:
        handler = await rc.get(ProcessPaymentHandler)
        ...
```

핸들러가 컨테이너를 알게 되면 :
- 핸들러 단위 테스트가 컨테이너를 만들어야 함 (무거움)
- 핸들러 코드에 인프라 (Dishka) import → 도메인 결합

→ 어댑터에만 컨테이너 의존. 핸들러는 의존성 *결과* 만 받는다.

### 4.2 같은 REQUEST 스코프 재사용 시도

```python
# ❌ 같은 publish 안에서 모든 핸들러가 한 스코프 공유?
async with container() as rc:
    for handler in handlers:
        await handler(event, rc)
```

이렇게 하면 :
- 한 핸들러의 session commit 이 다른 핸들러에 영향
- 한 핸들러 실패 시 다른 핸들러도 같은 session 으로 묶임 (rollback 전염)
- 격리 깨짐

→ **각 어댑터가 독립 스코프**. session 격리가 핸들러 fault isolation 의 한 축.

### 4.3 컨테이너를 클로저로 캡처하는 함정

```python
def register_event_handlers(event_bus, container):
    container_ref = container       # 의도적 변수

    async def on_order_created(event):
        async with container_ref() as rc:
            ...

    event_bus.subscribe(OrderCreatedEvent, on_order_created)
# register 함수가 끝나도 on_order_created 는 container 를 클로저로 잡고 있음.
# container 가 GC 되지 않음 — 의도적.
# 단, 테스트에서 container 를 새로 만들면 옛 핸들러가 옛 container 를 잡고 있음.
# 매 테스트마다 새 event_bus + 새 register 호출이 정답.
```

테스트 누수 방지는 §21 (테스트 인프라) 에서 다룸.

### 4.4 어댑터가 예외를 삼킨다?

```python
async def on_order_created(event):
    async with container() as rc:
        handler = await rc.get(ProcessPaymentHandler)
        await handle_order_created(event, handler)
        # 예외 발생 시 — 어떻게 되나?
```

흐름 :
1. `handle_order_created` 가 예외 발생
2. `async with container()` 가 cleanup (session.rollback)
3. 어댑터가 예외를 다시 raise
4. EventBus 의 `try/except Exception` 이 잡아서 `logger.error`
5. 다른 핸들러는 계속 실행 (격리)

→ 어댑터 자체는 예외를 *추가로* 처리하지 않는다. EventBus 의 격리에 맡김.

만약 어댑터에서 try/except 를 추가하면 EventBus 의 로그가 어댑터의 로그로 덮여서 디버깅 어려워짐. **격리의 책임은 한 곳** (EventBus).

### 4.5 register_event_handlers 가 길어질 때

Phase 4 시점 어댑터 11 개. Phase 8 쯤이면 30+ 가 될 수 있다. 분리 시점 :

| 어댑터 수 | 권장 |
|---|---|
| ~10 | `main.py` 안 register 함수 한 개 |
| 10~30 | 모듈별로 `register_payment_handlers`, `register_shipping_handlers` 함수 분리 + main 에서 호출 |
| 30+ | 모듈별 등록 모듈 + 자동 등록 메커니즘 검토 (데코레이터, naming convention scan) |

ShopTracker 는 단계 1 단계. 분리는 *문제가 생긴 후* 에.

### 4.6 동기 핸들러를 등록

```python
def sync_handler(event):           # ❌ async 가 아님
    print(event)

event_bus.subscribe(OrderCreatedEvent, sync_handler)
# publish 호출 시:
# await sync_handler(event)
# → TypeError: object NoneType can't be used in 'await' expression
```

ShopTracker 의 EventBus 는 모든 핸들러가 코루틴 함수임을 가정. 04 §4.2 참조.

### 4.7 lifespan 의 호출 누락

`register_event_handlers(event_bus, container)` 를 lifespan 에서 호출하지 않으면 :
- subscribe 가 한 번도 안 일어남
- publish 시 `_handlers.get(event_type, []) == []` → 핸들러 0 개
- `event_published, handler_count=0` 로그만 남고 *조용히 아무 일도 안 일어남*

가장 미묘한 함정. 통합 테스트에서 "결제가 자동 생성됐나?" 를 검증하면 잡힘.

---

## §5 다른 환경과의 비교

### 5.1 Spring (Java)

```java
@Component
public class PaymentEventHandler {
    @Autowired
    private ProcessPaymentService service;

    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void handleOrderCreated(OrderCreatedEvent event) {
        service.process(event);          // 의존성은 @Autowired 필드
    }
}
```

차이 :
- Spring 은 **AOP proxy** 가 어댑터 역할. `@Autowired` 필드는 컨테이너가 채움.
- `@TransactionalEventListener(phase = AFTER_COMMIT)` 가 ShopTracker 가 못 하는 *commit 후 발행* 보장.
- ShopTracker 의 명시적 어댑터는 Spring 의 *어떻게 동작하는지를 한 줄 한 줄 보여주는 것* 에 가깝다.

### 5.2 NestJS (TypeScript)

```typescript
@Injectable()
export class PaymentEventHandler {
  constructor(private readonly service: ProcessPaymentService) {}

  @OnEvent('order.created')
  async handleOrderCreated(event: OrderCreatedEvent) {
    await this.service.process(event);
  }
}
```

차이 :
- 데코레이터 + 클래스 메서드. NestJS DI 가 인스턴스를 만들고 메서드를 등록.
- 이벤트 키가 *문자열* (`'order.created'`) — 와일드카드 (`'order.*'`) 지원.
- ShopTracker 는 *타입 자체* 를 키로 — 더 type-safe 하지만 와일드카드 X.

### 5.3 .NET (MediatR)

```csharp
public class PaymentEventHandler : INotificationHandler<OrderCreatedEvent>
{
    private readonly IProcessPaymentService _service;
    public PaymentEventHandler(IProcessPaymentService service)
    {
        _service = service;
    }
    public async Task Handle(OrderCreatedEvent notification, CancellationToken token)
    {
        await _service.Process(notification);
    }
}
```

차이 :
- MediatR 가 컨테이너에서 핸들러 인스턴스를 만들고 호출.
- 핸들러는 *클래스 + 인터페이스 구현*. 함수 X.
- ShopTracker 는 함수 + 어댑터. 더 가벼움.

### 5.4 Temporal / Cadence

```python
@workflow.defn
class OrderWorkflow:
    @workflow.run
    async def run(self, order_id):
        await workflow.execute_activity(process_payment, order_id)
        await workflow.execute_activity(create_shipment, order_id)
```

Temporal 은 **워크플로우 자체가 영속화** 되어 어댑터 / 의존성 주입 같은 개념이 다른 차원.

→ 비교의 의미 : ShopTracker 는 학습용 *최소 구현*. Spring/NestJS/Temporal 같은 프로덕션 도구들이 어떤 문제를 해결하는지 이해하는 출발점.

---

## §6 학습 포인트 (한 줄 요약)

1. **어댑터 = 시그니처 변환** — EventBus 의 `(event,)` ↔ 핸들러의 `(event, deps...)`.
2. **REQUEST 스코프는 수동 처리** — `async with container() as rc:` 로 매 어댑터가 독립 스코프.
3. **`register_event_handlers` = 라우팅 단일 진리원** — main.py 만 보면 모든 라우팅이 보인다.
4. **lifespan 한 번만** — subscribe 누적이라 매 요청 호출 X.
5. **APP 스코프는 `container.get` 직접** — REQUEST 만 컨텍스트 매니저 필요.
6. **각 어댑터 독립 스코프** — session 격리 = fault isolation.
7. **핸들러는 컨테이너를 모른다** — 어댑터에만 컨테이너 의존.
8. **예외는 EventBus 가 격리** — 어댑터에서 try/except 추가 X.
9. **클로저로 caputre** — `register_event_handlers` 안에서 `container` 를 클로저로 잡음.
10. **분리 시점은 30 어댑터쯤** — 그 전엔 한 함수가 가시성 더 좋음.

---

## 추가 참고

- 03 — DI + Dishka : 정책 주입과 REQUEST 스코프의 자동 처리
- 04 — EventBus : subscribe / publish 의 본질
- 06 — Saga : 어댑터를 통과하는 이벤트 사슬
- Dishka context manager: https://dishka.readthedocs.io/en/stable/usage.html
- Spring `@TransactionalEventListener` : commit 후 발행 (ShopTracker 가 갖지 못한 보장)
- 다음 글 : `21-test-infrastructure-patterns.md` — 통합 테스트에서 어댑터 등록을 어떻게 다루는가
