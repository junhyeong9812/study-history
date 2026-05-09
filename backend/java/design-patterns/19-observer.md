# 19 — Observer

> **분류**: 행위 패턴 (Behavioral)
> **한 줄**: 1:N 의존. 한 객체 변경 시 의존하는 객체들에게 *자동 통보*.

---

## 1. 의도 (Intent)

> "Define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically."

상태 변화의 *발신자* 와 *수신자* 를 분리. Pub/Sub 의 OOP 버전.

## 2. 문제 상황 (Motivation)

주가가 변하면 차트, 알람, 로그 모두 업데이트.

```java
// ❌ 발신자가 모든 수신자 알아야 함
class Stock {
    Chart chart;
    Alarm alarm;
    Logger logger;

    public void setPrice(double p) {
        this.price = p;
        chart.update(p);
        alarm.check(p);
        logger.log(p);
    }
}
// → 새 수신자 (Notification 등) 추가 시 Stock 수정.
```

해결 : Stock 은 *알림* 만 발행. 수신자는 등록.

## 3. 구조 (Structure)

```
Subject (interface)             Observer (interface)
  ┌────────────────┐             ┌──────────┐
  │ attach(o)      │ ─────────► │ update() │
  │ detach(o)      │             └──────────┘
  │ notify()       │                  ▲
  └────────┬───────┘                  │
           │                    ┌─────┴─────┐
   ConcreteSubject          ConcreteObserver1, 2, 3
   - state
   + setState() {
       state = ...
       notify()
     }
```

## 4. 참여자 (Participants)

- **Subject**: observer 등록 / 해제 / 통보 인터페이스.
- **ConcreteSubject**: 실제 상태 + observers 보관.
- **Observer**: update 인터페이스.
- **ConcreteObserver**: subject 의 변경에 반응.

## 5. Java 예제

### 5.1 정통 Observer

```java
import java.util.ArrayList;
import java.util.List;

// Observer interface
interface Observer<T> {
    void update(T value);
}

// Subject
class Stock {
    private final String symbol;
    private double price;
    private final List<Observer<Double>> observers = new ArrayList<>();

    public Stock(String symbol) { this.symbol = symbol; }

    public void subscribe(Observer<Double> o) { observers.add(o); }
    public void unsubscribe(Observer<Double> o) { observers.remove(o); }

    public void setPrice(double p) {
        this.price = p;
        notifyObservers();
    }

    private void notifyObservers() {
        for (Observer<Double> o : observers) {
            o.update(price);
        }
    }
}

// Concrete Observers
class ChartView implements Observer<Double> {
    @Override public void update(Double price) {
        System.out.println("Chart updated: $" + price);
    }
}

class PriceAlarm implements Observer<Double> {
    private final double threshold;
    public PriceAlarm(double threshold) { this.threshold = threshold; }
    @Override public void update(Double price) {
        if (price > threshold) System.out.println("ALARM: price > " + threshold);
    }
}

class StockLogger implements Observer<Double> {
    @Override public void update(Double price) {
        System.out.println("[LOG] price=" + price);
    }
}

// 사용
public class Main {
    public static void main(String[] args) {
        Stock stock = new Stock("AAPL");

        stock.subscribe(new ChartView());
        stock.subscribe(new PriceAlarm(150));
        stock.subscribe(new StockLogger());

        stock.setPrice(140);    // 차트, 로그
        stock.setPrice(160);    // 차트, 알람, 로그
    }
}
```

### 5.2 함수형 (Java 8+)

```java
public class StockEvents {
    private final List<Consumer<Double>> listeners = new ArrayList<>();

    public void subscribe(Consumer<Double> l) { listeners.add(l); }
    public void publish(double price) { listeners.forEach(l -> l.accept(price)); }
}

// 사용
StockEvents events = new StockEvents();
events.subscribe(p -> System.out.println("Chart: " + p));
events.subscribe(p -> { if (p > 150) System.out.println("ALARM"); });
events.publish(160);
```

람다 한 줄로 Observer 정의.

### 5.3 PropertyChangeSupport (Java 표준)

```java
import java.beans.PropertyChangeListener;
import java.beans.PropertyChangeSupport;

class Stock {
    private final PropertyChangeSupport pcs = new PropertyChangeSupport(this);
    private double price;

    public void addListener(PropertyChangeListener l) {
        pcs.addPropertyChangeListener("price", l);
    }

    public void setPrice(double p) {
        double old = this.price;
        this.price = p;
        pcs.firePropertyChange("price", old, p);
    }
}

// 사용
stock.addListener(e -> System.out.println("Changed: " + e.getNewValue()));
```

## 6. 변형 (Variants)

### 6.1 Push vs Pull

- **Push**: subject 가 데이터 *전달* (`update(price)`).
- **Pull**: subject 가 *알림만*, observer 가 필요 시 조회 (`update(); ... stock.getPrice()`).

Push 가 보통 단순. Pull 은 observer 가 자기 필요한 것만 가져갈 수 있음.

### 6.2 Filtered Subscription
```java
events.subscribe(price -> price > 150, listener);    // 조건부
```

### 6.3 Async Observer
```java
events.subscribe(price -> CompletableFuture.runAsync(() -> handle(price)));
```

별도 스레드에서 처리 — subject 는 빨리 반환.

### 6.4 Reactive Streams (RxJava, Project Reactor)
backpressure / 합성 / 에러 처리 표준화한 Observer 의 진화.

## 7. 함정 / 흔한 오해

### 7.1 메모리 누수
Observer 가 unsubscribe 안 되면 — Subject 가 살아있는 한 Observer 도 살아있음. WeakReference 또는 명시적 unsubscribe.

### 7.2 순환 통보
A.notify → B.update → B.notify → A.update → ... 무한 루프. 통보 깊이 제한 또는 그래프가 DAG 인지 검증.

### 7.3 동기 호출 / 시간
Observer.update() 가 느리면 Subject 가 블록. async 또는 timeout.

### 7.4 예외 격리
한 Observer 가 던진 예외가 다른 Observer 호출 막음 — try/catch 로 격리 (이벤트 버스의 fault isolation).

### 7.5 순서 의존
Observer 들의 호출 순서가 *예측 가능* 한가? List 면 등록 순서. Set 이면 미정.

## 8. 관련 패턴

- **Mediator (17)**: Mediator 가 observers 사용.
- **Command (14)**: 통보가 command 로 큐잉될 수 있음.
- **MVC**: Model 이 Subject, View 가 Observer.

## 9. 실무 사례

- Java `java.util.Observer` (deprecated since Java 9 — 한계 많음)
- `java.beans.PropertyChangeListener`
- Swing `ActionListener`, `MouseListener`
- RxJava / Reactor `Flux`, `Mono`, `Subject`
- Node.js `EventEmitter`
- Vue / React reactive (state 변경 → UI 자동)
- WebSocket 서버 (broadcast)
- Kafka consumer / Redis Pub/Sub (분산 observer)
- Spring `ApplicationListener`, `@EventListener`

```java
// Spring
@Component
public class OrderListener {
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        // 자동 호출
    }
}
```

> 본 ShopTracker 의 `EventBus` (study/04) 가 정확히 Observer 패턴의 이벤트 버스 변형.
