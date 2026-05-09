# 20 — State

> **분류**: 행위 패턴 (Behavioral)
> **한 줄**: 객체의 *내부 상태에 따라 행동이 바뀜*. 조건문 대신 *상태 클래스*.

---

## 1. 의도 (Intent)

> "Allow an object to alter its behavior when its internal state changes. The object will appear to change its class."

상태 머신을 if/switch 대신 *클래스* 로 표현.

## 2. 문제 상황 (Motivation)

주문 (Order) 의 동작이 상태에 따라 달라짐 — `cancel()` 이 PENDING 상태에선 가능, PAID 면 불가.

```java
// ❌ 모든 메서드가 if/switch 로 가득
class Order {
    String status;

    public void cancel() {
        if (status.equals("PENDING")) {
            status = "CANCELLED";
        } else if (status.equals("PAID")) {
            throw new IllegalStateException("Cannot cancel paid order");
        } else if (status.equals("SHIPPED")) {
            throw new IllegalStateException("Cannot cancel shipped order");
        }
    }

    public void pay() {
        if (status.equals("PENDING")) {
            status = "PAID";
        } else if ...
    }
    // ... 메서드마다 같은 분기 반복
}
```

상태 추가 시 *모든 메서드* 의 분기 수정.

해결 : 상태별 클래스. 메서드 호출이 현재 state 객체에 위임.

## 3. 구조 (Structure)

```
Context (Order)
   ┌──────────────────┐
   │ - state: State   │
   │ + request() {    │
   │     state.handle()│
   │ }                │
   │ + setState(s)    │
   └──────────────────┘

State (interface)
   ┌──────────┐
   │ handle() │
   └──────┬───┘
          │
   ┌──────┴───────┬───────────┐
   │              │           │
StatePending   StatePaid   StateShipped
```

## 4. 참여자 (Participants)

- **Context**: state 보유. 행동을 state 에 위임.
- **State**: 상태별 인터페이스.
- **ConcreteState**: 특정 상태의 행동 + 상태 전이.

## 5. Java 예제

### 5.1 Order State Machine

```java
// Context
class Order {
    private OrderState state;
    private final String id;

    public Order(String id) {
        this.id = id;
        this.state = new PendingState();
    }

    public void setState(OrderState state) { this.state = state; }
    public String getId() { return id; }

    // Context 의 행동은 state 에 위임
    public void pay() { state.pay(this); }
    public void ship() { state.ship(this); }
    public void deliver() { state.deliver(this); }
    public void cancel() { state.cancel(this); }
    public String currentState() { return state.getClass().getSimpleName(); }
}

// State interface
interface OrderState {
    void pay(Order order);
    void ship(Order order);
    void deliver(Order order);
    void cancel(Order order);
}

// Concrete States
class PendingState implements OrderState {
    @Override public void pay(Order order) {
        System.out.println("Order " + order.getId() + " paid");
        order.setState(new PaidState());
    }
    @Override public void ship(Order order) {
        throw new IllegalStateException("Pay first");
    }
    @Override public void deliver(Order order) {
        throw new IllegalStateException("Pay first");
    }
    @Override public void cancel(Order order) {
        System.out.println("Order " + order.getId() + " cancelled");
        order.setState(new CancelledState());
    }
}

class PaidState implements OrderState {
    @Override public void pay(Order order) {
        throw new IllegalStateException("Already paid");
    }
    @Override public void ship(Order order) {
        System.out.println("Order " + order.getId() + " shipped");
        order.setState(new ShippedState());
    }
    @Override public void deliver(Order order) {
        throw new IllegalStateException("Ship first");
    }
    @Override public void cancel(Order order) {
        throw new IllegalStateException("Cannot cancel paid order");
    }
}

class ShippedState implements OrderState {
    @Override public void pay(Order order) { throw new IllegalStateException(); }
    @Override public void ship(Order order) { throw new IllegalStateException(); }
    @Override public void deliver(Order order) {
        System.out.println("Order " + order.getId() + " delivered");
        order.setState(new DeliveredState());
    }
    @Override public void cancel(Order order) {
        throw new IllegalStateException("Cannot cancel shipped");
    }
}

class DeliveredState implements OrderState {
    @Override public void pay(Order o) { throw new IllegalStateException(); }
    @Override public void ship(Order o) { throw new IllegalStateException(); }
    @Override public void deliver(Order o) { throw new IllegalStateException(); }
    @Override public void cancel(Order o) { throw new IllegalStateException(); }
}

class CancelledState implements OrderState {
    @Override public void pay(Order o) { throw new IllegalStateException(); }
    @Override public void ship(Order o) { throw new IllegalStateException(); }
    @Override public void deliver(Order o) { throw new IllegalStateException(); }
    @Override public void cancel(Order o) { throw new IllegalStateException(); }
}

// 사용
public class Main {
    public static void main(String[] args) {
        Order order = new Order("001");
        order.pay();        // Pending → Paid
        order.ship();       // Paid → Shipped
        order.deliver();    // Shipped → Delivered
        // order.cancel(); → IllegalStateException
    }
}
```

각 state 클래스가 *자기 상태에서 가능한 전이* 만 정의. 상태 추가 시 새 클래스만 추가.

### 5.2 Enum 기반 단순 버전

```java
enum OrderStatus {
    PENDING {
        @Override public OrderStatus pay() { return PAID; }
        @Override public OrderStatus cancel() { return CANCELLED; }
    },
    PAID {
        @Override public OrderStatus ship() { return SHIPPED; }
    },
    SHIPPED {
        @Override public OrderStatus deliver() { return DELIVERED; }
    },
    DELIVERED, CANCELLED;

    public OrderStatus pay() { throw new IllegalStateException(); }
    public OrderStatus ship() { throw new IllegalStateException(); }
    public OrderStatus deliver() { throw new IllegalStateException(); }
    public OrderStatus cancel() { throw new IllegalStateException(); }
}

class Order {
    private OrderStatus status = OrderStatus.PENDING;
    public void pay() { status = status.pay(); }
    // ...
}
```

→ 클래스 수가 줄어듦. 단, enum 안에 로직이 많아지면 복잡.

## 6. 변형 (Variants)

### 6.1 State 객체를 Singleton 으로
상태가 *상태 없음* (stateless) 이면 같은 인스턴스 공유.

```java
class PendingState {
    private static final PendingState INSTANCE = new PendingState();
    public static PendingState getInstance() { return INSTANCE; }
    private PendingState() {}
    ...
}
```

### 6.2 State Table
```java
Map<Pair<State, Event>, State> transitions = Map.of(
    Pair.of(PENDING, PAY), PAID,
    Pair.of(PAID, SHIP), SHIPPED,
    ...
);
```

전이 규칙을 *데이터* 로. 코드 변경 없이 외부 설정 가능.

### 6.3 Statechart (계층적 / 동시 상태)
UML statechart, state machine library (Spring Statemachine).

## 7. 함정 / 흔한 오해

### 7.1 Strategy (21) 와의 차이
- **State**: 객체의 상태에 따라 *자기 행동* 변경. State 가 *전이* 도 결정.
- **Strategy**: 알고리즘 *교체*. 외부에서 결정.
- 코드는 같아 보일 수 있음. 의도가 다름.

### 7.2 상태 폭발
상태 N 개 × 이벤트 M 개 = N×M 메서드. 큰 시스템은 statechart 도구로.

### 7.3 Context 와 State 의 결합
State 가 Context 의 모든 정보 알아야 할 때 — Context 가 자기 자신을 인자로 넘기거나, State 가 Context 참조 보유.

### 7.4 전이 검증 누락
"Pending 에서 ship 호출 시 어떻게?" 같은 모든 조합 검증 안 하면 미정의 행동.

## 8. 관련 패턴

- **Strategy (21)**: 형태 비슷.
- **Singleton (05)**: stateless state 객체.
- **Flyweight (11)**: state 인스턴스 공유.
- **Memento (18)**: state history 저장.

## 9. 실무 사례

- 주문 상태 (PENDING / PAID / SHIPPED / DELIVERED / CANCELLED)
- 결제 상태 (PENDING / APPROVED / REJECTED / REFUNDED)
- TCP connection 상태 (CLOSED / LISTEN / SYN_SENT / ESTABLISHED / ...)
- Java `Thread.State` (NEW, RUNNABLE, BLOCKED, WAITING, ...)
- 게임 캐릭터 (IDLE / WALKING / ATTACKING / DEAD)
- 비디오 플레이어 (PLAYING / PAUSED / STOPPED)
- HTTP request lifecycle (CONNECTED / SENT / RECEIVED / CLOSED)
- Spring Statemachine, Akka FSM

> ShopTracker 의 `OrderStatus.can_transition_to()` 가 enum 기반 state machine 의 단순 버전.
