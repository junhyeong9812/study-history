# Spring ApplicationEvents

> 같은 JVM 안의 모듈 간 통신 메커니즘. 메시지 브로커 없이 발행/구독.

## 개념

`ApplicationEventPublisher`로 이벤트를 발행하면 Spring이 등록된 리스너들에게 전달. 발행자와 구독자는 서로를 모르고, **이벤트 객체**만 공유한다 → 모듈 간 의존성 역전.

이 프로젝트의 의도: Orders가 Payments를 모르고, Payments가 Shipping을 모름. 모두 `shared/events` 의 record만 안다.

## 발행

```java
// orders/application/service/OrderCommandService.java
private final ApplicationEventPublisher eventPublisher;
...
eventPublisher.publishEvent(new OrderCreatedEvent(
    order.getId().value(), customerName,
    order.getTotalAmount().amount(), items.size(), Instant.now()
));
```

`ApplicationEventPublisher`는 Spring이 자동 주입. `ApplicationContext` 자체가 이걸 구현한다.

## 구독

### 동기 (`@EventListener`)

```java
@Component
public class X {
    @EventListener
    public void on(OrderCreatedEvent event) { ... }
}
```
- **호출 스레드에서 즉시 실행**.
- 트랜잭션 내부 → 함께 롤백.
- 예외가 발행자까지 전파됨.

### 트랜잭션 경계 인지 (`@TransactionalEventListener`)

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void on(OrderCreatedEvent event) { ... }
```
- `AFTER_COMMIT`(기본): 트랜잭션이 **커밋된 후** 실행. 결제 같은 부수 효과를 안전하게 분리.
- `AFTER_ROLLBACK`, `BEFORE_COMMIT`, `AFTER_COMPLETION` 도 가능.

### 비동기 + 지속성 (`@ApplicationModuleListener`) — Spring Modulith

이 프로젝트가 사용하는 형태:
```java
// payments/application/eventhandler/PaymentEventHandler.java
@ApplicationModuleListener
public void on(OrderCreatedEvent event) {
    processPaymentUseCase.processPayment(...);
}
```

내부적으로 다음 4가지를 묶어준다:
1. `@TransactionalEventListener(phase = AFTER_COMMIT)` — 발행자 트랜잭션 커밋 후
2. `@Async` — 별도 스레드에서 실행
3. `@Transactional(propagation = REQUIRES_NEW)` — 리스너 자체 트랜잭션
4. **이벤트 지속성** — `event_publication` 테이블에 저장, 실패 시 재시도

자세한 동작은 `spring-modulith.md` 참고.

## 이 프로젝트의 이벤트 흐름

```
OrderCreated   →  PaymentEventHandler.on()    → processPayment
                  TrackingEventHandler.on()   → 타임라인 기록

PaymentApproved → OrderEventHandler.on()      → Order PAID
                  ShippingEventHandler.on()    → createShipment
                  TrackingEventHandler.on()   → 타임라인 기록

PaymentRejected → OrderEventHandler.on()      → Order CANCELLED
                  TrackingEventHandler.on()   → FAILED 단계

ShipmentCreated → OrderEventHandler.on()      → Order SHIPPING
                  TrackingEventHandler.on()   → 타임라인 기록

ShipmentStatusChanged → TrackingEventHandler.on() → DELIVERED 단계 갱신

SubscriptionActivated → TrackingEventHandler.on() → 로그
```

여러 모듈이 같은 이벤트를 동시에 구독해도 각자 독립적으로 처리됨.

## 주의사항

1. **이벤트 record는 immutable로**. 변경 가능한 필드를 두면 동시 처리에서 사고 남.
2. **이벤트 = "사실의 통보"**. 명령(command)이 아님. `CalculateDiscountEvent`(X) → `OrderCreatedEvent`(O).
3. **인메모리 이벤트는 JVM 재시작 시 사라짐**. 결제 처리 중간에 죽으면 이벤트 유실 → Modulith의 지속성 기능 필요.
4. **순환 이벤트 주의**. A 이벤트 → B 핸들러 → C 이벤트 → A 핸들러 → … 무한 루프.

## FastAPI 대응

이 프로젝트의 FastAPI 버전이 `InMemoryEventBus`를 직접 구현했던 이유 → Python 표준에 동등 기능 없음. Spring은 이게 프레임워크에 내장.

| FastAPI 버전 | Spring |
|--------------|--------|
| `EventBus.publish()` | `ApplicationEventPublisher.publishEvent()` |
| `EventBus.subscribe(Type, handler)` | `@EventListener` / `@ApplicationModuleListener` |
| 비동기 | `asyncio.create_task` 직접 | Spring Async/Modulith 자동 |
| 영속성 | 직접 outbox 구현 | Modulith `event_publication` 자동 |
