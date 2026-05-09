# 17 — Mediator

> **분류**: 행위 패턴 (Behavioral)
> **한 줄**: 객체 간 직접 참조 대신 *중재자* 가 통신을 조율.

---

## 1. 의도 (Intent)

> "Define an object that encapsulates how a set of objects interact. Mediator promotes loose coupling by keeping objects from referring to each other explicitly."

여러 객체가 서로 *직접* 알면 결합 폭발. 중재자가 *유일한 통신 채널*.

## 2. 문제 상황 (Motivation)

대화방 — 사용자 N 명. 각자 메시지를 모든 다른 사용자에게 보내야 한다.

```java
// ❌ 직접 참조
class User {
    private List<User> others;
    public void send(String msg) {
        for (User u : others) u.receive(msg);
    }
}
// → 사용자 N 명, 참조 N×(N-1) 개. 누가 누구를 아는지 추적 불가.
```

해결 : 중재자 (대화방) 가 모든 사용자를 알고, 메시지 전달 담당.

```java
class ChatRoom {                     // Mediator
    void send(String msg, User from) { ... }
}

class User {
    ChatRoom room;
    public void send(String msg) { room.send(msg, this); }
}
```

→ User 는 ChatRoom 만 안다. ChatRoom 만 모든 User 를 안다.

## 3. 구조 (Structure)

```
                Mediator (interface)
                   ┌──────────────┐
                   │ notify(c, e) │
                   └──────┬───────┘
                          │
              ConcreteMediator
                 - colleagues
                 + notify(...) {
                     /* coordinate */
                   }

   Colleague (abstract)         ConcreteColleague1, ConcreteColleague2
   - mediator: Mediator         (mediator 만 안다)
```

## 4. 참여자 (Participants)

- **Mediator**: 중재 인터페이스.
- **ConcreteMediator**: 동료들 보유 + 통신 조율.
- **Colleague**: Mediator 만 안다. 자기 일 + Mediator 통한 알림.

## 5. Java 예제

### 5.1 ChatRoom

```java
import java.util.ArrayList;
import java.util.List;

// Mediator
interface ChatMediator {
    void send(String message, User user);
    void register(User user);
}

// Concrete Mediator
class ChatRoom implements ChatMediator {
    private final List<User> users = new ArrayList<>();

    @Override public void register(User user) {
        users.add(user);
        user.setRoom(this);
    }

    @Override public void send(String message, User sender) {
        for (User u : users) {
            if (u != sender) u.receive(message, sender.getName());
        }
    }
}

// Colleague
abstract class User {
    protected ChatMediator room;
    protected final String name;

    public User(String name) { this.name = name; }
    public String getName() { return name; }
    public void setRoom(ChatMediator room) { this.room = room; }

    public abstract void send(String message);
    public abstract void receive(String message, String from);
}

// Concrete Colleagues
class TextUser extends User {
    public TextUser(String name) { super(name); }

    @Override public void send(String message) {
        System.out.println(name + " sends: " + message);
        room.send(message, this);
    }

    @Override public void receive(String message, String from) {
        System.out.println(name + " received from " + from + ": " + message);
    }
}

// 사용
public class Main {
    public static void main(String[] args) {
        ChatMediator room = new ChatRoom();

        User alice = new TextUser("Alice");
        User bob = new TextUser("Bob");
        User carol = new TextUser("Carol");

        room.register(alice);
        room.register(bob);
        room.register(carol);

        alice.send("Hi everyone");
        // Bob received from Alice: Hi everyone
        // Carol received from Alice: Hi everyone
    }
}
```

### 5.2 GUI Form Mediator

```java
// 폼의 위젯들이 서로의 상태를 신경쓰지 않게
interface DialogMediator {
    void widgetChanged(Widget w);
}

class LoginDialog implements DialogMediator {
    Button loginBtn;
    TextField username, password;
    Checkbox rememberMe;

    @Override public void widgetChanged(Widget w) {
        // 사용자명 / 비번 둘 다 채워졌으면 버튼 enable
        if (w == username || w == password) {
            loginBtn.setEnabled(
                !username.getText().isEmpty() && !password.getText().isEmpty()
            );
        }
        // 다른 위젯들의 협업 로직...
    }
}
```

위젯 간 직접 참조 대신 dialog 가 모든 상호작용 조율.

## 6. 변형 (Variants)

### 6.1 Event-driven mediator
Mediator 가 이벤트 버스 (04). 객체가 이벤트 발행 → mediator 가 다른 객체에 전달.

### 6.2 Mediator 의 분산
큰 시스템에서는 mediator 가 god object 가 됨. 도메인별로 분리.

## 7. 함정 / 흔한 오해

### 7.1 God Mediator
모든 로직을 mediator 에 — 결국 god object. mediator 는 *조율* 만, 비즈니스 로직은 colleague.

### 7.2 Facade (10) 와의 차이
- **Facade**: 클라이언트가 *서브시스템* 에 접근 단순화. 단방향 (client → subsystem).
- **Mediator**: 객체들 *서로* 통신 중재. 양방향.

### 7.3 Observer (19) 와의 차이
- **Mediator**: 정해진 객체들 사이의 *조율*. 일대일 또는 일대다 둘 다.
- **Observer**: 한 객체 변경에 대한 *통보*. 단방향 fire-and-forget.

### 7.4 결합이 colleague → mediator 로 이동
완전 제거가 아님. 단지 *집중*. 그래도 N×N 결합보다는 낫다.

## 8. 관련 패턴

- **Facade (10)**: 비슷하지만 다름.
- **Observer (19)**: mediator 가 observer 로 알림.
- **Singleton (05)**: mediator 가 singleton 인 경우.

## 9. 실무 사례

- Java `java.util.concurrent.locks.Condition` (Lock 이 mediator)
- Java Swing/AWT 의 dialog 가 위젯 mediator
- ATC (Air Traffic Control) — 비행기들이 직접 통신 X, ATC 거침
- Discord / Slack 서버 — 모든 메시지가 서버 거침
- Spring `ApplicationEventPublisher` — 이벤트 mediator 변형
- React `Context API` — 컴포넌트 간 props drilling 회피
- Redux `store` (action → reducer → state → UI)
- 메시지 브로커 (Kafka, RabbitMQ) — 분산 시스템의 mediator

> Event-driven 아키텍처는 본질적으로 mediator 의 산업화. 한 모듈이 다른 모듈을 직접 부르지 않고 *이벤트 버스 (mediator)* 를 거침.
