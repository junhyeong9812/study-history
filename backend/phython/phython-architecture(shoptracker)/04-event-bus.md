# 04 — 이벤트 버스 (모듈 간 비동기 통신)

> Orders 가 Payments 를 직접 부르지 않고도, "주문 생성됨" 이라는 사실을 던지면 Payments / Shipping / Tracking 이 알아서 반응한다. 이걸 가능하게 하는 것이 이벤트 버스다. 이 글은 ShopTracker 의 `InMemoryEventBus` 코드를 한 줄씩 까보면서, 이벤트 기반 아키텍처가 *왜* 필요하고 *어떻게* 동작하며 *언제* Outbox / Kafka 같은 외부 인프라로 가야 하는지를 정리한다.

---

## §0 문제 정의 — 모듈 간 직접 호출의 비용

모듈러 모놀리스에서 가장 쉬운 잘못은 `OrderService` 안에서 `PaymentService` 를 직접 부르는 것이다.

```python
# anti-pattern
class OrderService:
    def __init__(self, payment_service: PaymentService, shipping_service: ShippingService):
        ...
    async def create_order(self, ...):
        order = Order.create(...)
        await self.repo.save(order)
        await self.payment_service.start_payment(order.id, order.total)   # ← 결합
        await self.shipping_service.prepare_shipment(order.id)            # ← 결합
        await self.tracking_service.start_tracking(order.id)              # ← 결합
```

이 코드의 문제 :

1. **방향 결합** — Orders 가 Payments / Shipping / Tracking 을 *알아야 한다*. import 가 발생하고, Orders 모듈이 다른 모듈을 흡수한다.
2. **확장 비용** — 새로운 모듈 (예: Marketing 이 "주문 시 쿠폰 지급") 이 추가될 때마다 Orders 코드를 수정해야 한다. **OCP 위반**.
3. **트랜잭션 경계 혼란** — payment 가 실패하면 order 도 롤백할 것인가? Orders 가 Payments 의 실패를 어떻게 해석할 것인가? 책임이 한 함수에 뒤섞인다.
4. **테스트 비용** — `OrderService` 를 테스트하려면 PaymentService / ShippingService / TrackingService 의 mock 을 모두 주입해야 한다.
5. **사이클 위험** — Payments 가 다시 Orders 의 무언가를 호출해야 한다면 순환 참조가 생긴다 (`from app.orders import ...` ↔ `from app.payments import ...`).

해법 : **간접 통신 (indirect communication)**. Orders 는 "이런 일이 일어났다" 라는 사실 (event) 만 던지고, 누가 그 사실을 듣는지는 알 필요가 없다. 듣는 쪽도 누가 던졌는지 알 필요가 없다.

이 패턴의 OOP 이름은 **Observer 패턴**이고, 시스템 레벨로 확장한 것이 **이벤트 기반 아키텍처 (Event-Driven Architecture, EDA)** 다.

---

## §1 본질 메커니즘 — Observer + Pub/Sub

### 1.1 Observer 패턴 (GoF)

> Subject 가 상태를 바꾸면 등록된 Observer 들에게 통보한다.

```
┌─────────┐   notify    ┌──────────┐
│ Subject │ ──────────► │ Observer │
└─────────┘             └──────────┘
     ▲ register
     │
┌──────────┐
│ Observer │
└──────────┘
```

이를 분산 시스템 스케일로 가져온 것이 Pub/Sub :

```
Publisher ─── event ───► [ Event Bus ] ─── event ───► Subscriber A
                                       └─── event ───► Subscriber B
                                       └─── event ───► Subscriber C
```

### 1.2 핵심 속성

| 속성 | 설명 |
|---|---|
| **Loose coupling** | Publisher 와 Subscriber 가 서로의 타입을 import 하지 않는다. 둘 다 *event 타입* 만 import. |
| **Many-to-many** | 한 이벤트에 여러 핸들러가 붙을 수 있다. |
| **Past tense (사실)** | 이벤트는 "OrderCreated" 처럼 과거형. 명령 (Command) 이 아니다. "DoPayment" 가 아니라 "OrderCreated → 너희가 결정해". |
| **Immutable** | 사실이므로 변하지 않는다 → `frozen=True dataclass`. |

### 1.3 Event vs Command — 자주 혼동되는 부분

| | Command | Event |
|---|---|---|
| 시제 | 명령형 (`CreateOrder`, `StartPayment`) | 과거형 (`OrderCreated`, `PaymentApproved`) |
| 수신자 | **하나** (특정 핸들러에게 시킴) | **0~N** (관심 있는 누구든) |
| 결과 | 실패하면 발신자가 알아야 함 | 발신자는 결과를 신경 쓰지 않음 |
| 의도 | "이거 해줘" | "이런 일 일어났음, 알아서들" |
| 결합 | 발신자가 수신자를 안다 | 모른다 |

Orders 가 `OrderCreatedEvent` 를 던지면 — Payments 가 안 받아도 Orders 는 *상관없다*. Orders 의 책임은 "주문이 생성됨" 을 알리는 것까지. (만약 Payments 가 받지 않으면 결제가 안 일어난다는 *비즈니스적* 문제가 있겠지만, *코드 흐름* 상 Orders 의 책임은 거기서 끝.)

---

## §2 ShopTracker 코드 정독 — `InMemoryEventBus`

ShopTracker 의 이벤트 버스는 **84 줄**짜리 단순한 메모리 구현이다. 학습용으로 의도적으로 작게 만들어졌다. 줄별로 읽어보자.

### 2.1 Protocol 정의 (인터페이스)

```python
# src/app/shared/event_bus.py:25-33
class EventBus(Protocol):
    """이벤트 버스 인터페이스.

    도메인 코드와 핸들러는 이 Protocol만 의존한다.
    InMemoryEventBus인지, RedisEventBus인지 모른다.
    """
    async def publish(self, event: object) -> None: ...
    def subscribe(self, event_type: type, handler: Callable) -> None: ...
```

- `class EventBus(Protocol)` — 02 장에서 본 structural typing. 도메인 코드는 이 `EventBus` 만 의존하면, 나중에 `RedisEventBus` / `KafkaEventBus` 로 바꿔도 도메인은 그대로다.
- `async def publish(...) -> None: ...` — 이벤트를 던진다. `event: object` 로 받는 이유는 **모든 이벤트 타입을 받아들이기 위해**. 타입 안전성은 핸들러 등록 시 `event_type: type` 으로 보장한다.
- `def subscribe(...)` — *동기* 함수. 단순히 dict 에 핸들러를 등록할 뿐이라 await 가 필요 없음.
- `handler: Callable` — 함수든 메서드든 callable 이면 다 OK.

> **DIP 적용 지점** : 이 `EventBus` Protocol 은 application layer 에 있어야 할까 shared 에 있어야 할까? ShopTracker 는 `shared/` 에 둔다 — 모든 모듈이 이 *계약* 을 알아야 하기 때문. 인프라 종속물 (`InMemoryEventBus` 구현체) 만 별도로 분리할 수도 있지만, 학습 프로젝트라 같은 파일에 둔다.

### 2.2 InMemoryEventBus — 상태

```python
# event_bus.py:36-46
class InMemoryEventBus:
    """메모리 기반 구현체."""

    def __init__(self) -> None:
        # defaultdict: 처음 보는 키도 자동으로 빈 리스트 생성
        self._handlers: dict[type, list[Callable]] = defaultdict(list)
```

- `_handlers: dict[type, list[Callable]]` — 키가 **이벤트 타입 자체** (클래스 객체) 다. 인스턴스가 아니라 클래스. Python 에서 클래스도 first-class object 라서 dict 의 키로 쓸 수 있다.
- `defaultdict(list)` — `self._handlers[OrderCreatedEvent].append(handler)` 를 처음 호출해도 KeyError 없이 빈 리스트가 자동 생성된다. 없으면 매번 :
  ```python
  if event_type not in self._handlers:
      self._handlers[event_type] = []
  self._handlers[event_type].append(handler)
  ```
  이렇게 두 줄을 써야 한다. defaultdict 은 이걸 한 줄로 만들어준다.
- 자료구조 선택 이유 : 한 이벤트에 N 개 핸들러 → `list`. 이벤트 타입은 dict 키 → O(1) 조회.

### 2.3 subscribe — 핸들러 등록

```python
# event_bus.py:48-59
def subscribe(self, event_type: type, handler: Callable) -> None:
    self._handlers[event_type].append(handler)
    logger.info(
        "event_subscribed",
        event_name=event_type.__name__,
        handler_name=handler.__qualname__,
    )
```

- `event_type.__name__` — 클래스의 *문자열* 이름. `OrderCreatedEvent.__name__ == "OrderCreatedEvent"`.
- `handler.__qualname__` — qualified name. 모듈 안에서의 전체 경로. `PaymentHandler.handle_order_created` 같은 형태로 어느 클래스의 메서드인지까지 보인다. `__name__` 은 `handle_order_created` 만 나옴.
- `structlog.info("event_subscribed", ...)` — 17 장에서 다룰 구조화 로깅. JSON 으로 찍혀서 grep 이 쉽다.
- **언제 호출되나?** : 보통 앱 시작 시점 (lifespan / startup) 에 한 번만. 매 요청마다 subscribe 하면 핸들러가 중복 등록된다 (이게 `InMemoryEventBus` 의 함정 중 하나).

### 2.4 publish — 사실 발행

```python
# event_bus.py:61-85
async def publish(self, event: object) -> None:
    event_type = type(event)
    handlers = self._handlers.get(event_type, [])
    logger.info(
        "event_published",
        event_name=event_type.__name__,
        handler_count=len(handlers),
    )
    for handler in handlers:
        try:
            await handler(event)
        except Exception as e:
            # ★ 핵심: 하나의 핸들러가 실패해도 다른 핸들러는 계속 실행
            logger.error(
                "event_handler_failed",
                event_name=event_type.__name__,
                handler_name=handler.__qualname__,
                error=str(e),
            )
```

한 줄씩 :

- `event_type = type(event)` — 이벤트 *인스턴스* 에서 *클래스* 를 꺼낸다. `type(OrderCreatedEvent(...)) is OrderCreatedEvent` → True. 이걸로 dict 를 조회.
- `self._handlers.get(event_type, [])` — `defaultdict` 라도 `get(key, default)` 를 쓰면 새 키가 *생기지 않는다*. publish 만으로 빈 리스트가 dict 에 박히는 부작용을 피하는 디테일.
- `await handler(event)` — 모든 핸들러를 *직렬* 로 호출. 첫 번째가 끝나야 두 번째가 시작된다. 병렬 처리는 아님 (의도적).
- `try / except Exception` — **격리 (fault isolation)**. 핸들러 하나가 터져도 다른 핸들러는 돈다. 결제 핸들러가 실패해도 알림 핸들러는 동작해야 한다는 비즈니스 요구.
- `except Exception` — `BaseException` 이 아니라 `Exception` 만 잡는다. `KeyboardInterrupt`, `SystemExit` 같은 시스템 레벨 신호는 통과시켜야 함.
- `logger.error(...)` — 에러를 *삼키지 않는다*. 로그를 남기고 처리는 계속.

> **이게 적절한가?** : 학습 프로젝트로서는 OK. 하지만 프로덕션이라면 "결제 실패가 로그만 남고 사라진다" 는 심각한 데이터 손실. 이 한계 때문에 §7 의 Outbox 가 필요해진다.

### 2.5 사용 예 — Phase 1 의 흐름

```python
# 앱 시작 시 (orders/main.py 등 startup hook)
event_bus.subscribe(OrderCreatedEvent, payment_handler.on_order_created)
event_bus.subscribe(PaymentApprovedEvent, shipping_handler.on_payment_approved)
event_bus.subscribe(PaymentApprovedEvent, tracking_handler.on_payment_approved)
event_bus.subscribe(PaymentRejectedEvent, order_handler.on_payment_rejected)

# 런타임 (CreateOrderHandler.handle 안에서)
order = Order.create(...)
await self.repo.save(order)
await self.event_bus.publish(OrderCreatedEvent(
    order_id=order.id,
    customer_name=order.customer_name,
    total_amount=order.total_amount,
    items_count=len(order.items),
    timestamp=datetime.now(),
))
# → payment_handler.on_order_created 가 호출됨
# → 결제 시도 → PaymentApprovedEvent 발행
# → shipping_handler / tracking_handler 가 동시에 깨어남 (실은 순차)
```

이 한 번의 publish 로 **payment → shipping → tracking 까지의 사슬이 자동으로 흐른다**. Orders 코드는 한 줄도 모른다.

### 2.6 이벤트 정의 (`shared/events.py`)

```python
# events.py:26-33
@dataclass(frozen=True)
class OrderCreatedEvent:
    order_id: UUID
    customer_name: str
    total_amount: Decimal
    items_count: int
    timestamp: datetime
```

- `@dataclass(frozen=True)` — **불변**. 이벤트는 사실이므로 변경 불가. Python 의 `frozen=True` 는 `__setattr__` 을 막아서 `event.order_id = ...` 를 런타임 에러로 만든다.
- `@dataclass` 가 자동 생성하는 것 :
  - `__init__(order_id, customer_name, ...)` — keyword-only 가 아니므로 positional 로 호출 가능
  - `__repr__` — 디버깅 시 `OrderCreatedEvent(order_id=..., customer_name=...)` 출력
  - `__eq__` — 필드별 비교. 테스트에서 `assert event == expected` 가능
  - `__hash__` — frozen=True 일 때만. dict 키 / set 원소로 사용 가능
- 필드는 **원시 값 + UUID + Decimal + datetime** 만. 도메인 객체 (Order entity) 를 그대로 넣지 않음. 이유 :
  1. 직렬화 가능해야 함 (Redis / Kafka 옮길 때).
  2. 수신자가 발신 모듈의 도메인 타입을 import 하면 결합이 다시 생긴다.
  3. 이벤트는 *스냅샷*. 발행 시점의 사실을 박제. 도메인 객체는 변하지만 이벤트는 안 변함.

### 2.7 모듈별 위치 — shared/ 에 두는 이유

```
src/app/
├── shared/
│   ├── event_bus.py     ← Protocol + InMemoryEventBus
│   └── events.py        ← 모든 이벤트 타입
├── orders/              ← OrderCreatedEvent 발행
├── payments/            ← OrderCreatedEvent 구독, PaymentApprovedEvent 발행
└── shipping/            ← PaymentApprovedEvent 구독
```

- `events.py` 가 shared 에 있어야 모든 모듈이 import 할 수 있다.
- 이 `shared/events.py` 가 *팽창* 하면? Phase 5 쯤 가면 50+ 이벤트가 한 파일에 모인다. 나누는 시점은 "모듈별로 의미 있게 묶을 수 있을 때". 예 : `shared/events/orders.py`, `shared/events/payments.py`. 단, **이벤트는 모듈 간 계약이라 모듈 *내부* 에 두면 안 된다** — 그러면 다른 모듈이 `from app.orders.events import OrderCreatedEvent` 를 해야 하고, 결국 의존이 생긴다.

---

## §3 직접 구현 — 80 줄짜리 미니 이벤트 버스

학습 목적으로 직접 만들어 보면서 어디가 까다로운지 체험.

### 3.1 v1 — 가장 단순한 구현

```python
from collections import defaultdict
from typing import Callable

class MiniEventBus:
    def __init__(self):
        self._handlers = defaultdict(list)

    def subscribe(self, event_type: type, handler: Callable):
        self._handlers[event_type].append(handler)

    async def publish(self, event):
        for handler in self._handlers[type(event)]:
            await handler(event)

# 사용
bus = MiniEventBus()
bus.subscribe(OrderCreatedEvent, lambda e: print(f"got: {e}"))
await bus.publish(OrderCreatedEvent(...))
```

이건 ShopTracker 코드와 본질적으로 같다. 차이점은 **격리**.

### 3.2 v2 — 핸들러 격리

```python
async def publish(self, event):
    for handler in self._handlers[type(event)]:
        try:
            await handler(event)
        except Exception:
            logger.exception("handler failed")
            # 계속 진행
```

격리를 안 하면 핸들러 A 가 터지면 B, C 가 호출되지 않는다. 이건 *비즈니스 결정*이다 — "결제 실패가 알림 발송도 막아야 하는가?" 보통은 No.

### 3.3 v3 — 병렬 처리 (asyncio.gather)

```python
import asyncio

async def publish(self, event):
    handlers = self._handlers[type(event)]
    results = await asyncio.gather(
        *(handler(event) for handler in handlers),
        return_exceptions=True   # 예외도 결과로 받음
    )
    for handler, result in zip(handlers, results):
        if isinstance(result, Exception):
            logger.error("handler failed", handler=handler.__qualname__, error=result)
```

장점 : I/O 가 많은 핸들러들이 동시에 돈다. 3 개 핸들러가 각자 100ms DB 호출이라면 직렬은 300ms, 병렬은 100ms.

단점 :
- 순서 보장 안 됨. 핸들러 간 순서 의존이 있으면 깨진다.
- DB connection pool 동시 점유.
- 디버깅이 더 복잡.

ShopTracker 가 직렬을 택한 이유 : 단순함 + 학습 의도. 프로덕션에서는 케이스 바이 케이스.

### 3.4 v4 — 데코레이터 기반 등록 (NestJS 스타일)

```python
class MiniEventBus:
    def __init__(self):
        self._handlers = defaultdict(list)

    def on(self, event_type):
        """데코레이터로 사용."""
        def decorator(func):
            self._handlers[event_type].append(func)
            return func
        return decorator

# 사용
bus = MiniEventBus()

@bus.on(OrderCreatedEvent)
async def handle_order_created(event):
    print(event)
```

장점 : 핸들러 정의와 등록이 한곳에 모인다. NestJS 의 `@OnEvent('order.created')` 와 같은 발상.
단점 : import 시점에 등록되므로 "테스트에서 핸들러를 빼고 싶다" 가 어려워진다 — DI 컨테이너 주입 방식이 더 testable.

ShopTracker 가 데코레이터를 안 쓰는 이유 : 명시적 등록 (startup 에서 한 번) 이 학습에 더 명확함 + Dishka 컨테이너에서 핸들러를 꺼내 등록하는 흐름이 깔끔함.

---

## §4 함정 (실제 부딪히는 문제들)

### 4.1 핸들러 중복 등록

테스트에서 `event_bus.subscribe(...)` 를 매번 호출하다가 핸들러가 N 번 등록되어 한 이벤트에 대해 같은 처리가 N 번 일어남. → subscribe 는 startup 한 번만. 테스트는 매번 새 EventBus 인스턴스를 만들어 주입.

```python
# 잘못
@pytest.fixture
def app():
    event_bus.subscribe(...)   # 테스트마다 누적
    return create_app()

# 옳음
@pytest.fixture
def event_bus():
    bus = InMemoryEventBus()
    bus.subscribe(...)
    return bus
```

### 4.2 동기 핸들러 mix

`InMemoryEventBus.publish` 는 `await handler(event)` 라 핸들러가 반드시 *코루틴 함수* 여야 한다. 동기 함수를 등록하면 :

```python
def sync_handler(event):
    print(event)

bus.subscribe(OrderCreatedEvent, sync_handler)
await bus.publish(OrderCreatedEvent(...))
# TypeError: object NoneType can't be used in 'await' expression
```

해결 : 핸들러를 무조건 `async def` 로 강제하거나, publish 에서 `inspect.iscoroutinefunction(handler)` 로 분기.

```python
import inspect

async def publish(self, event):
    for handler in self._handlers[type(event)]:
        result = handler(event)
        if inspect.iscoroutine(result):
            await result
```

ShopTracker 는 강제 방식 (모든 핸들러 async) 을 채택.

### 4.3 이벤트 vs 트랜잭션 — at-most-once 의 위험

```python
async def create_order_handler(...):
    async with session.begin():
        order = Order.create(...)
        await repo.save(order)            # ① DB INSERT
        await event_bus.publish(event)    # ② 이벤트
    # ③ commit
```

이 코드의 함정 :

- ② 가 ③ 전에 실행 → 핸들러 (Payment) 가 *아직 commit 안 된* order 를 조회하려 하면 보일까? 같은 session 이면 보이고, 다른 session 이면 안 보인다.
- ③ 의 commit 이 실패하면? 이벤트는 이미 발행됐고, 핸들러는 이미 결제 처리했을 수 있음. **상태 불일치**.
- ② 의 핸들러가 터져서 ③ 가 롤백되면? Order 가 사라진 채로 핸들러는 부분적으로 실행됨.

이게 인메모리 이벤트 버스의 근본 한계. 해결 :

1. **commit 후 발행** (after_commit hook) — SQLAlchemy event 로 `session.after_commit` 에 발행 큐를 비운다. 적어도 "DB 에 안 들어간 이벤트가 발행되는" 문제는 막음.
2. **Outbox 패턴** (§7) — 이벤트를 같은 트랜잭션에서 outbox 테이블에 INSERT 하고, 별도 워커가 발행. 진정한 at-least-once 보장.

ShopTracker 는 학습용이라 단순 publish. 프로덕션이면 둘 중 하나는 도입.

### 4.4 핸들러 실패가 사라진다

`except Exception → logger.error` 는 *조용한 실패*. 핸들러가 결제를 처리하다가 외부 PG 사 timeout 으로 터지면 → 로그만 남고 결제 안 됨 → 사용자는 주문 됐다고 봤는데 결제는 영영 안 됨.

해결 :
- **Retry** : tenacity / asyncio retry 로 N 번 시도.
- **DLQ (Dead Letter Queue)** : N 번 실패하면 별도 테이블/큐로 옮겨서 사람이 처리.
- **Saga 보상** : 결제 실패가 확정되면 OrderCancellation 이벤트 발행.

§6 의 Saga 글에서 다룸.

### 4.5 순환 이벤트 (이벤트 폭풍)

PaymentApproved → Shipping 핸들러 → ShipmentCreated 발행 → 어떤 핸들러가 또 다른 이벤트 → ... → 결국 OrderCreated 가 다시 발행되면 무한 루프.

해결 :
- 이벤트 그래프를 *DAG* 로 유지. 주기 (cycle) 가 생기지 않게 설계.
- 디버그 시 발행 chain 을 `correlation_id` 로 추적.
- 한 publish 안에서 같은 이벤트 타입이 N 회 이상 재발행되면 경고.

### 4.6 핸들러 등록 누락

새 모듈을 만들었는데 `subscribe(...)` 를 깜박해서 이벤트가 발행돼도 아무도 안 듣는 상태. 컴파일러가 못 잡음 (이벤트 타입은 `object` 로 받음).

해결 :
- startup 시 핸들러 등록 로그를 보고 점검.
- Integration test 에서 "OrderCreated 발행 → Payment 가 처리됨" 을 검증.
- IDE 의 "find usages" 로 이벤트 타입을 grep 하면 발행 / 구독 모두 보임.

---

## §5 다른 언어 / 프레임워크 비교

| 플랫폼 | 메커니즘 | 특징 |
|---|---|---|
| **Spring** | `ApplicationEventPublisher` + `@EventListener` | 동기 기본, `@Async` 로 비동기. transactional event 도 지원 (`@TransactionalEventListener(phase = AFTER_COMMIT)`). |
| **NestJS** | `EventEmitter2` + `@OnEvent('event.name')` | 문자열 키 기반. 와일드카드 (`order.*`) 지원. |
| **Node.js (raw)** | `EventEmitter` (built-in) | 동기 호출. 비동기는 직접 처리. |
| **Java (Axon)** | `EventBus` + `@EventHandler` | CQRS / ES 프레임워크. 영속화까지 통합. |
| **Go** | 내장 없음. channel 또는 watermill | channel 로 직접 구현 가능. watermill 라이브러리가 Pub/Sub 추상. |
| **Rust** | tokio broadcast channel | 멀티 consumer 채널. 컴파일 타임 타입 안전. |

ShopTracker 의 InMemoryEventBus 는 본질적으로 **Spring 의 ApplicationEventPublisher 동기 모드** 와 같은 모델.

> **Spring 과의 결정적 차이** : Spring 은 `@TransactionalEventListener` 로 "트랜잭션 commit 후" 에 핸들러를 호출하는 옵션이 있다. ShopTracker 의 InMemoryEventBus 는 그게 없으니 §4.3 의 함정에 그대로 노출된다.

---

## §6 인프라 진화 경로

학습 프로젝트 → 프로덕션으로 가면서 이벤트 버스가 어떻게 바뀌는지.

```
┌─ Stage 0 (단일 인스턴스, 학습) ─┐
│  InMemoryEventBus              │
│  - 단점: 인스턴스 죽으면 사라짐
│  - 단점: 멀티 인스턴스 동기 X
└────────────────────────────────┘
              ↓ 프로덕션화
┌─ Stage 1 (Outbox + Polling) ──┐
│  Outbox 테이블에 INSERT (같은 tx)
│  별도 워커가 polling 하여 발행 │
│  - 장점: at-least-once 보장
│  - 장점: 인프라 추가 거의 없음
└────────────────────────────────┘
              ↓ 멀티 인스턴스
┌─ Stage 2 (Redis Pub/Sub) ─────┐
│  publish → Redis CHANNEL
│  각 인스턴스가 구독             │
│  - 장점: 단순, 빠름
│  - 단점: 메시지 영속화 X (놓치면 끝)
└────────────────────────────────┘
              ↓ 신뢰성 / 순서 / 재처리
┌─ Stage 3 (Kafka / RabbitMQ) ──┐
│  파티션, 오프셋, 재처리, 영속화
│  - 장점: 본격 EDA
│  - 단점: 운영 복잡, 학습 비용
└────────────────────────────────┘
```

ShopTracker 가 Stage 0 에서 멈춘 이유 : *공부 목적*. Stage 1 (Outbox) 로 가는 시점이 "프로덕션 트래픽" 으로 적정.

---

## §7 Outbox 패턴 — 진정한 at-least-once

이벤트 버스의 본질적 문제는 **DB 트랜잭션과 발행이 별개의 작업** 이라는 점. 둘 다 성공하거나 둘 다 실패해야 하는데, 일반 publish 는 그걸 보장 못 함 (분산 트랜잭션 = 2PC 는 너무 무거움).

### 7.1 아이디어

> 이벤트를 발행하지 말고, *outbox 테이블* 에 같은 트랜잭션으로 INSERT 한다. 별도 프로세스가 outbox 를 polling 하여 진짜 발행한다.

```sql
CREATE TABLE outbox (
    id UUID PRIMARY KEY,
    event_type VARCHAR NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL,
    published_at TIMESTAMP NULL    -- NULL = 미발행
);
```

```python
# Order 저장과 outbox INSERT 가 같은 트랜잭션
async with session.begin():
    await session.execute(insert(OrderModel).values(...))
    await session.execute(insert(OutboxModel).values(
        id=uuid4(),
        event_type="OrderCreatedEvent",
        payload=json.dumps(asdict(event), default=str),
        created_at=datetime.now(),
    ))
# commit 시 둘 다 들어가거나 둘 다 안 들어감
```

```python
# 별도 워커
async def outbox_publisher():
    while True:
        async with session.begin():
            rows = await session.execute(
                select(OutboxModel)
                .where(OutboxModel.published_at.is_(None))
                .order_by(OutboxModel.created_at)
                .limit(100)
                .with_for_update(skip_locked=True)   # ← 멀티 워커 안전
            )
            for row in rows.scalars():
                try:
                    event = deserialize(row.event_type, row.payload)
                    await real_event_bus.publish(event)
                    row.published_at = datetime.now()
                except Exception:
                    logger.exception("publish failed, will retry")
        await asyncio.sleep(0.1)
```

### 7.2 보장

- **At-least-once** : 워커가 publish 후 published_at 업데이트 전에 죽으면 다음 polling 에서 다시 발행. 중복 발행 가능 → **핸들러는 idempotent 해야 함**.
- **순서** : `order by created_at` 으로 순서 보장. 단, 멀티 워커면 깨질 수 있음.
- **트랜잭션 atomicity** : DB INSERT 와 이벤트 발행이 한 트랜잭션이 됨.

### 7.3 ShopTracker 가 도입한다면

- `shared/outbox/` 모듈 추가
- `EventBus` Protocol 의 `OutboxEventBus` 구현 추가 (publish → outbox INSERT)
- 워커 lifespan 에서 시작
- 핸들러는 `event_id` 기준으로 처리 여부 기록 (idempotency)

지금 단계에선 학습 목표상 InMemory 로 충분하다 — **Outbox 가 필요한 이유** 를 이해하는 게 핵심.

---

## §8 테스트 전략

### 8.1 Unit — EventBus 자체

```python
import pytest
from app.shared.event_bus import InMemoryEventBus

@pytest.mark.asyncio
async def test_publish_calls_subscribed_handler():
    bus = InMemoryEventBus()
    received = []

    async def handler(event):
        received.append(event)

    bus.subscribe(str, handler)
    await bus.publish("hello")

    assert received == ["hello"]


@pytest.mark.asyncio
async def test_publish_isolates_handler_failure():
    bus = InMemoryEventBus()
    received = []

    async def bad(event): raise RuntimeError("boom")
    async def good(event): received.append(event)

    bus.subscribe(str, bad)
    bus.subscribe(str, good)
    await bus.publish("x")

    assert received == ["x"]   # bad 가 터져도 good 은 호출됨


@pytest.mark.asyncio
async def test_publish_with_no_subscribers_is_noop():
    bus = InMemoryEventBus()
    await bus.publish("nobody listens")   # 예외 없이 끝나야 함
```

### 8.2 Integration — 모듈 간 흐름

```python
@pytest.mark.asyncio
async def test_order_created_triggers_payment(test_container):
    # given
    order_handler = await test_container.get(CreateOrderHandler)
    payment_repo = await test_container.get(PaymentRepository)

    # when
    await order_handler.handle(CreateOrderCommand(...))

    # then — payment 가 자동으로 만들어졌는지
    payments = await payment_repo.list_by_order_id(...)
    assert len(payments) == 1
```

이 테스트는 EventBus 를 *진짜* 로 쓰면서 핸들러 등록까지 통합 검증. 03 장의 test container 패턴과 결합.

### 8.3 핸들러 자체 테스트

핸들러는 결국 함수 / 메서드. EventBus 를 거치지 않고 직접 호출하여 테스트 가능 → 가장 빠르고 신뢰성 높음.

```python
@pytest.mark.asyncio
async def test_payment_handler_creates_payment_on_order_created():
    handler = PaymentHandler(payment_repo=fake_repo, ...)
    event = OrderCreatedEvent(order_id=uuid4(), ...)

    await handler.on_order_created(event)

    assert len(fake_repo.saved) == 1
```

---

## §10 학습 포인트 (한 줄 요약)

1. **이벤트는 과거형 사실** — Command 가 아니다. 수신자는 0~N 명.
2. **결합 방향이 사라진다** — Publisher / Subscriber 가 서로 import 안 함. event 타입만 공유.
3. **이벤트는 불변** (frozen dataclass) — 사실은 변하지 않으므로.
4. **이벤트는 원시 값으로** — 도메인 객체 X. 직렬화 가능 + 결합 방지.
5. **defaultdict + Callable 리스트** — 이벤트 버스의 자료구조는 본질적으로 이게 전부.
6. **핸들러 격리 (try/except)** — 한 핸들러 실패가 다른 핸들러를 막지 않게.
7. **InMemoryEventBus 의 한계** : 트랜잭션 분리, at-most-once, 인스턴스 사이 동기 X.
8. **Outbox 패턴** : DB INSERT 와 이벤트 발행을 한 트랜잭션으로 묶기 위해 별도 테이블 + 워커 도입.
9. **at-least-once → idempotent 핸들러** : 중복 처리에도 결과가 같아야.
10. **인프라는 *나중* 에** : InMemory → Outbox → Redis → Kafka 순으로 *필요해질 때* 진화.

---

## 추가 참고

- Martin Fowler, *Event-Driven Architecture*, https://martinfowler.com/articles/201701-event-driven.html
- Chris Richardson, *Microservices Patterns* — Saga, Outbox 챕터
- Vaughn Vernon, *Implementing Domain-Driven Design* — Domain Event 챕터
- ShopTracker 다음 글 : `05-cqrs.md` (read/write 분리), `06-saga-cross-module.md` (이벤트 체인의 보상 거래)
