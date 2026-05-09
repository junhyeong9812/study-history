# 06 — Saga (모듈 간 분산 트랜잭션과 보상 거래)

> Orders → Payments → Shipping → Tracking 으로 이어지는 흐름은 한 트랜잭션으로 묶을 수 없다. 각 단계는 다른 모듈, 다른 책임이고 (그리고 결국 다른 DB / 외부 API 일 수도 있고), `BEGIN; ... COMMIT;` 으로 감쌀 수가 없다. 이 한계를 다루는 패턴이 **Saga**. ShopTracker 는 이벤트 기반 (Choreography) 사가를 통해 이 흐름을 구현한다.

---

## §0 문제 정의 — 분산 트랜잭션의 불가능성

### 0.1 ACID 의 한계

단일 DB 안에서는 :

```python
async with session.begin():
    await order_repo.save(order)
    await inventory_repo.decrement(items)
    await user_repo.deduct_points(user_id, points)
# 셋 다 commit 되거나, 셋 다 rollback
```

이건 ACID 의 atomicity 가 자동으로 처리한다.

하지만 ShopTracker 의 흐름 :

```
1. Order 저장 (orders DB)
2. 결제 PG 사 호출 (외부 API)
3. Payment 저장 (payments DB)
4. Shipment 생성 (shipping DB, 외부 배송사 API 호출)
5. Tracking 시작 (tracking DB)
```

이 5 단계는 :
- 다른 모듈 (다른 책임 경계)
- 외부 시스템 (PG, 배송사) 호출 포함
- 다른 DB 일 수 있음 (마이크로서비스로 분해 시)
- **하나의 트랜잭션으로 묶을 수 없다**

전통 해법 인 **2PC (Two-Phase Commit)** 는 :
- 모든 참여자가 prepare → commit 단계를 거침
- 한 명이라도 실패하면 모두 rollback
- **단점 : 너무 느림, coordinator 죽으면 행 (block)**, 외부 API 는 보통 지원 안 함

→ 분산 환경에서 ACID 트랜잭션은 *실용적으로 불가능*.

### 0.2 Saga 패턴 — Hector Garcia-Molina (1987)

> 긴 트랜잭션을 *작은 로컬 트랜잭션의 시퀀스* 로 쪼개고, 실패 시 *보상 (compensating) 트랜잭션* 으로 되돌린다.

```
T1 → T2 → T3 → T4 → T5    (정상 흐름)
     │
실패 시 ↓
T1 ← C2 ← C3              (보상 흐름)
```

각 Ti 는 **로컬 트랜잭션** (단일 DB 에서 ACID). 전체로 보면 ACID 가 깨지지만, 비즈니스 레벨에서 *최종적으로 일관된 상태* 로 수렴.

ShopTracker 의 정상 흐름 :

```
Order.PENDING → Payment.APPROVED → Shipment.CREATED → Tracking.STARTED
```

실패 케이스 :

```
Order.PENDING → Payment.REJECTED → (보상) Order.CANCELLED
```

---

## §1 본질 메커니즘

### 1.1 두 가지 사가 스타일

| | Choreography (안무) | Orchestration (오케스트레이션) |
|---|---|---|
| 제어 | 분산 — 각 모듈이 이벤트를 받고 다음 이벤트를 발행 | 중앙 — Orchestrator 가 모든 단계를 명령 |
| 결합 | 약함 (이벤트 계약만 공유) | Orchestrator 가 모든 모듈을 알아야 함 |
| 추적 | 어려움 (분산 trace 필요) | 쉬움 (한곳에서 봄) |
| 복잡도 | 낮음 (모듈 단순) | 높음 (Orchestrator 자체가 stateful) |
| 적합 | 단계 < 5, 모듈 적음 | 단계 ≥ 5, 분기 많음 |

ShopTracker 는 **Choreography** 채택. 이유 : 04 장의 EventBus 가 이미 있고, 모듈 간 결합 최소화가 학습 목표.

### 1.2 Choreography 사가의 작동 (ShopTracker)

```
[Orders]                    [Payments]              [Shipping]            [Tracking]
   │                            │                       │                     │
CreateOrder                     │                       │                     │
   ├── save Order               │                       │                     │
   └── publish OrderCreated ───►│                       │                     │
                                │                       │                     │
                                ├── 외부 PG 호출       │                     │
                                ├── save Payment        │                     │
                                └── publish PaymentApproved ────────────────► │
                                                        │                     │
                                                        ├── save Shipment     │
                                                        └── publish ShipmentCreated
                                                                              │
                                                                              ├── start tracking

# 실패 케이스
[Orders]                    [Payments]
   │                            │
publish OrderCreated ──────────►│
                                ├── 외부 PG 호출 → 거절
                                └── publish PaymentRejected ───►[Orders]
                                                                  │
                                                                  ├── order.cancel() (보상)
                                                                  └── publish OrderCancelled
```

각 모듈은 **자기 다음에 누가 오는지 모른다**. 자기가 받을 이벤트와 발행할 이벤트만 안다.

### 1.3 보상 트랜잭션 (Compensating Transaction)

T1, T2, T3 의 보상이 C1, C2, C3 라면 :

| 일반 트랜잭션 | 보상 |
|---|---|
| Order 생성 | Order 취소 (소프트 삭제, 상태만 CANCELLED) |
| Payment 승인 | Payment 환불 (refund) — *로직적으로* 되돌림. 실제 DB 행은 보존. |
| Shipment 생성 | Shipment 취소 — 배송사에 취소 API 호출. |
| 재고 차감 | 재고 복구 |

**보상은 단순 rollback 이 아니다** :
- DB 의 row 를 지우지 않음. 감사를 위해 *역방향 행위* 를 새로 기록.
- 외부 API 는 더더욱 — PG 사 결제 승인 후 "안 했던 것으로 해줘" 가 아니라 *환불 API* 를 호출.
- **idempotent** 해야 함 (재시도 안전).

---

## §2 ShopTracker 의 사가 설계

### 2.1 정상 흐름의 이벤트 사슬

```
1. CreateOrderHandler.handle
   ├── Order.create() + repo.save()           [로컬 tx in orders]
   └── publish OrderCreatedEvent

2. PaymentHandler.on_order_created (Payments 모듈)
   ├── 구독 정보 조회 (cross-module read - subscription_context)
   ├── 외부 PG 호출 (가짜 PG: 90% 승인)
   ├── Payment.create() + repo.save()         [로컬 tx in payments]
   └── publish PaymentApprovedEvent

3. ShippingHandler.on_payment_approved (Shipping 모듈)
   ├── 구독 등급별 배송비 계산
   ├── Shipment.create() + repo.save()        [로컬 tx in shipping]
   └── publish ShipmentCreatedEvent

4. TrackingHandler.on_shipment_created (Tracking 모듈)
   ├── 가짜 배송사 API → 트래킹 번호
   └── 추적 정보 저장                         [로컬 tx in tracking]
```

각 단계 :
- **로컬 트랜잭션** (단일 DB 안에서 ACID)
- **이벤트 발행** (다음 단계 트리거)
- 다음 단계가 누구인지 모름

### 2.2 보상 흐름 — Payment 거절

```python
# events.py:80-86
@dataclass(frozen=True)
class PaymentRejectedEvent:
    """결제 거절 시 발행. Orders가 구독하여 주문을 자동 취소."""
    payment_id: UUID
    order_id: UUID
    reason: str
    timestamp: datetime
```

```python
# orders/application/event_handlers.py (phase 2 추가 예정)
class OrderEventHandler:
    def __init__(self, repo, event_bus):
        self._repo = repo
        self._event_bus = event_bus

    async def on_payment_rejected(self, event: PaymentRejectedEvent) -> None:
        order = await self._repo.find_by_id(event.order_id)
        if order is None:
            return
        order.cancel()                         # ← 보상 트랜잭션
        await self._repo.update(order)
        await self._event_bus.publish(
            OrderCancelledEvent(
                order_id=order.id,
                reason=f"payment_rejected: {event.reason}",
                timestamp=datetime.now(UTC),
            )
        )
```

핵심 :
- Payments 가 OrderCancelled 를 발행하지 않는다 — Payments 는 결제만 안다.
- Orders 가 PaymentRejected 를 *듣고* 자기 자신을 취소.
- "실패 → 보상" 도 이벤트로 흐름.

### 2.3 누가 누구를 듣는지 매핑

| 이벤트 발행자 | 이벤트 | 구독자 | 의미 |
|---|---|---|---|
| Orders | OrderCreated | Payments | "결제 시작해" |
| Payments | PaymentApproved | Shipping, Tracking | "배송 / 추적 준비" |
| Payments | PaymentRejected | **Orders** | "주문 취소해" (보상) |
| Shipping | ShipmentCreated | Tracking | "추적 시작" |
| Shipping | ShipmentStatusChanged | Orders | "주문 상태 갱신" |

**역방향 이벤트** (Payments → Orders) 가 보상 채널. 결합 방향이 Phase 1 에선 한 방향이지만 보상이 들어오면 양방향이 됨.

---

## §3 직접 구현 — 미니 사가

```python
# 시나리오 : 회원가입 → 환영 메일 → 무료 쿠폰 발급
# 메일 실패 시 회원 활성화 취소

# 1. 정상 흐름
async def on_user_registered(event: UserRegistered):
    try:
        await send_welcome_mail(event.email)
        await event_bus.publish(WelcomeMailSent(user_id=event.user_id))
    except SmtpError as e:
        await event_bus.publish(WelcomeMailFailed(user_id=event.user_id, reason=str(e)))

async def on_welcome_mail_sent(event: WelcomeMailSent):
    coupon = await coupon_repo.create(user_id=event.user_id, type="welcome_5pct")
    await event_bus.publish(CouponIssued(coupon_id=coupon.id, user_id=event.user_id))

# 2. 보상
async def on_welcome_mail_failed(event: WelcomeMailFailed):
    user = await user_repo.find_by_id(event.user_id)
    user.deactivate()                          # 보상
    await user_repo.update(user)
    await event_bus.publish(UserDeactivated(user_id=user.id, reason=event.reason))

# 3. 등록
event_bus.subscribe(UserRegistered, on_user_registered)
event_bus.subscribe(WelcomeMailSent, on_welcome_mail_sent)
event_bus.subscribe(WelcomeMailFailed, on_welcome_mail_failed)
```

이 구조의 이점 :
- 각 핸들러는 5~10 줄.
- 책임이 단일.
- 새 단계 추가 (예 : 쿠폰 발급 후 push 알림) 시 핸들러 한 개만 추가.

이 구조의 비용 :
- 한 회원가입의 *전체 흐름* 을 한곳에서 볼 수가 없음. 각 핸들러를 따라가며 머릿속에서 조립.
- 디버깅에 trace ID 필수.

---

## §4 함정

### 4.1 보상이 항상 가능한가

"메일을 이미 보냈는데 어떻게 *안 보낸 것* 으로 만들 수 있나?" 보낼 수 없다 — *사과 메일* 을 보낼 수 있을 뿐.

→ **보상은 비즈니스 의미상의 되돌림**, 기술적 rollback 이 아님. 설계 시 "이 단계가 실패하면 사용자에게 어떻게 사과할 것인가" 를 비즈니스가 정의해야 함.

### 4.2 보상의 보상은 어떻게

C2 (Payment 환불) 도 외부 API 호출. 환불이 실패하면? 환불의 보상 = "운영자에게 알림" + retry queue + 수동 처리.

→ 사가는 "**최종 일관성**" 만 보장. 일정 시간 후 사람이 개입해야 할 수도 있다는 것을 인정.

### 4.3 동시성 — 같은 Order 에 두 이벤트

`OrderCancelled` 이벤트가 발행되는 동안 사용자가 직접 `cancel` API 를 호출. 둘 다 `order.cancel()` 시도 :
- 첫 호출 : `PENDING → CANCELLED` ✅
- 두 번째 : `CANCELLED → CANCELLED` 시도, 상태머신 (08 장) 이 `InvalidStatusTransition` 으로 거절

→ **상태머신이 멱등성을 일부 제공**. 단, `repo.update` 에서 optimistic locking (version 컬럼) 도 같이 두면 안전.

### 4.4 이벤트 순서 역전

PaymentApproved 와 PaymentRejected 가 같은 결제에 대해 둘 다 발행되면? (예 : 결제 시도 두 번, 한 번 승인, 한 번 거절)
→ 핸들러가 마지막 이벤트만 정답으로 보면 잘못된 상태. 해결 :
- `payment_id` 를 ID 로 사용 — 다른 결제 = 다른 사가.
- 한 결제는 한 번만 처리되도록 idempotency key.

### 4.5 At-least-once × 핸들러 idempotency

EventBus 는 at-least-once (특히 Outbox 도입 시) → 같은 이벤트가 두 번 도착할 수 있음.

```python
async def on_order_created(event: OrderCreatedEvent):
    # 첫 호출 : Payment 생성
    # 두 번째 호출 : 이미 Payment 있는데 또 생성? → 중복!

    existing = await self._payment_repo.find_by_order_id(event.order_id)
    if existing is not None:
        return                              # ← idempotent 처리
    ...
```

→ 모든 핸들러는 "이미 처리된 이벤트인가?" 를 먼저 체크.

### 4.6 사가 상태의 가시성

지금 어느 Order 가 어느 단계에 있는가? Choreography 에선 단일 view 가 없음 :
- Orders DB 에 `status = PAYMENT_PENDING` 이지만 Payments 가 죽어서 진행 안 됨
- Shipping 까지 갔는데 Tracking 만 실패

대응 :
- 각 모듈의 도메인 상태 + 로그를 종합하는 **Saga Tracking View** (Read Model) 를 별도로 유지 → 사실상 Orchestration 으로 진화.
- 또는 Distributed Tracing (OpenTelemetry) 로 한 요청의 전체 흐름을 trace.

### 4.7 시간 — 사가가 끝나지 않으면

PaymentApproved 가 영영 안 오면 Order 는 영원히 `PAYMENT_PENDING`. 해결 :
- TTL — 일정 시간 (예 : 30 분) 후 Orders 가 자동 cancel.
- Timeout 이벤트 (cron + DB 스캔 → publish OrderTimeout).
- 이건 Orchestrator 가 하기 더 자연스러움.

---

## §5 Orchestration 으로 진화

흐름이 복잡해지면 Choreography 의 분산 추적이 부담. 이때 **Orchestrator** 도입 :

```python
class OrderSagaOrchestrator:
    """Order 의 사가를 한곳에서 명령."""

    async def execute(self, command: CreateOrderCommand):
        order_id = await self.orders_svc.create(command)
        try:
            payment = await self.payments_svc.charge(order_id, ...)
            shipment = await self.shipping_svc.create(order_id, payment.amount)
            await self.tracking_svc.start(shipment.id)
        except PaymentRejected as e:
            await self.orders_svc.cancel(order_id, reason=str(e))
        except ShippingFailed as e:
            await self.payments_svc.refund(payment.id)
            await self.orders_svc.cancel(order_id, reason=str(e))
```

장점 :
- 흐름이 한곳에 — 읽기 쉬움.
- 보상 로직이 명시적.

단점 :
- Orchestrator 가 모든 모듈을 import — 결합 ↑.
- Orchestrator 자체의 stateful 성 (현재 어디까지 진행했는지) 을 어딘가에 저장해야 (Saga state DB).

ShopTracker 가 Choreography 인 이유 : Phase 1~3 의 흐름이 단순 (Order → Payment → Shipping → Tracking 순차). 분기가 많아지면 Phase 5+ 에서 Orchestrator 도입 검토.

---

## §6 다른 솔루션과의 비교

| 환경 | 사가 도구 |
|---|---|
| **Spring** | `@Transactional` (단일 DB), 분산은 Spring Cloud + Camunda / Axon |
| **NestJS** | `@nestjs/cqrs` + Saga 데코레이터 (`@Saga`) |
| **.NET** | MassTransit Saga 상태머신 |
| **Java** | Camunda BPMN, Axon Saga |
| **Python** | 라이브러리 표준 X. Celery 워크플로우, Temporal, Cadence |
| **Temporal** | 사가 / 워크플로우 전용 — 코드를 그냥 sequential 하게 짜면 알아서 영속화/재시도/보상 |

ShopTracker 는 EventBus 만 쓰는 *직접 구현* 사가. Temporal 같은 워크플로우 엔진이 *생산 환경의 정답* 인 경우가 많지만, 학습 목적상 직접 짜본다.

---

## §7 테스트 전략

### 7.1 단위 — 핸들러 하나

```python
async def test_order_handler_cancels_on_payment_rejected():
    repo = FakeOrderRepo(seed=[order_pending])
    bus = FakeEventBus()
    handler = OrderEventHandler(repo, bus)

    await handler.on_payment_rejected(
        PaymentRejectedEvent(
            payment_id=uuid4(),
            order_id=order_pending.id,
            reason="잔액 부족",
            timestamp=datetime.now(UTC),
        )
    )

    updated = repo.saved[0]
    assert updated.status == OrderStatus.CANCELLED
    assert any(isinstance(e, OrderCancelledEvent) for e in bus.published)
```

### 7.2 통합 — 사가 전체 흐름

```python
async def test_happy_path_creates_order_payment_shipment(test_container):
    create_order = await test_container.get(CreateOrderHandler)
    payment_repo = await test_container.get(PaymentReadRepo)
    shipment_repo = await test_container.get(ShipmentReadRepo)

    order_id = await create_order.handle(CreateOrderCommand(...))

    # 이벤트 흐름이 끝날 때까지 대기 (in-memory 라 즉시 끝남)
    await asyncio.sleep(0)

    assert (await payment_repo.find_by_order_id(order_id)) is not None
    assert (await shipment_repo.find_by_order_id(order_id)) is not None


async def test_payment_rejection_cancels_order(test_container, force_pg_reject):
    create_order = await test_container.get(CreateOrderHandler)
    order_repo = await test_container.get(OrderReadRepo)

    order_id = await create_order.handle(CreateOrderCommand(...))
    await asyncio.sleep(0)

    order = await order_repo.find_by_id(order_id)
    assert order.status == OrderStatus.CANCELLED
```

이 통합 테스트는 사가의 *비즈니스 정합* 을 검증.

### 7.3 보상의 idempotency

```python
async def test_cancel_twice_is_safe():
    handler = OrderEventHandler(...)
    event = PaymentRejectedEvent(...)

    await handler.on_payment_rejected(event)
    await handler.on_payment_rejected(event)   # 두 번째 — 안전해야 함

    # 단일 OrderCancelled 만 발행됐어야 (또는 두 번이어도 다운스트림이 idempotent 면 OK)
```

---

## §10 학습 포인트 (한 줄 요약)

1. **분산 트랜잭션은 실용적으로 불가능** — 2PC 는 너무 비쌈.
2. **Saga = 로컬 tx 시퀀스 + 보상** — ACID 포기, 최종 일관성 채택.
3. **Choreography (이벤트)** vs **Orchestration (중앙)** — 단계 적으면 전자, 많으면 후자.
4. **보상은 rollback 아님** — 비즈니스 의미상의 되돌림 (환불, 사과 메일, 상태 변경).
5. **모든 핸들러는 idempotent** — at-least-once 환경에서 중복 도착 대비.
6. **상태머신이 사가의 안전장치** — 잘못된 보상 호출이 invariant 위반으로 막힘.
7. **사가 상태 가시성** : 분산 trace 또는 별도 read model 없으면 디버깅 지옥.
8. **Timeout 처리 필수** — 핸들러가 안 와도 사가가 영원히 걸리지 않게.
9. **ShopTracker = Choreography** — EventBus + 핸들러로 단순 구현. 복잡해지면 Temporal / Camunda 검토.
10. **테스트 전략** : 단위 (핸들러) + 통합 (전체 흐름) + idempotency 별도 검증.

---

## 추가 참고

- Hector Garcia-Molina, *Sagas* (1987 원 논문) — https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf
- Chris Richardson, *Microservices Patterns*, Chapter 4 — Saga
- Temporal docs : https://docs.temporal.io/
- ShopTracker 다음 글 : `07-cross-cutting-via-shared-dto.md` (모듈 간 *읽기* 데이터 공유 — `subscription_context.py`)
