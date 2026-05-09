# Spring `TransactionSynchronization` — 트랜잭션 라이프사이클에 끼어들기

> `TransactionalEventPublisher`가 사용하는 Spring 트랜잭션 동기화 메커니즘을 풀어 정리한 학습 노트.
> "이벤트는 커밋 후에만 발행, 롤백 시 폐기"라는 계약이 어떻게 자바 코드 몇 줄로 구현되는가.

---

## 0. 분석 대상 코드

```java
// common/src/main/java/com/hybrid/common/event/TransactionalEventPublisher.java
package com.hybrid.common.event;

import org.springframework.stereotype.Component;
import org.springframework.transaction.support.TransactionSynchronization;
import org.springframework.transaction.support.TransactionSynchronizationManager;

@Component
public class TransactionalEventPublisher {

    private final InMemoryEventBus bus;

    public TransactionalEventPublisher(InMemoryEventBus bus) { this.bus = bus; }

    public void publish(DomainEvent event) {
        if (!TransactionSynchronizationManager.isSynchronizationActive()) {
            bus.publish(event);
            return;
        }
        TransactionSynchronizationManager.registerSynchronization(
                new TransactionSynchronization() {
                    @Override public void afterCommit() { bus.publish(event); }
                }
        );
    }
}
```

20줄짜리 코드 안에 **Spring 트랜잭션 추상의 핵심**이 들어있다. 하나씩 풀어보자.

---

## 1. 풀어야 할 문제

### 1.1 이중 쓰기(Dual Write) 함정

```java
@Transactional
public void createOrder(...) {
    Order order = Order.create(...);
    orderRepository.save(order);

    eventBus.publish(new OrderCreated(order.getId(), ...));   // ❌ 트랜잭션 커밋 전 발행
}
```

이렇게 짜면 두 가지 문제가 동시에 발생한다.

**문제 1: 커밋 안 됐는데 이벤트 발행됨**

```
1. order 저장 (DB에 INSERT, 아직 커밋 안 됨)
2. eventBus.publish(OrderCreated)
   → PaymentEventHandler가 호출
   → orderId로 order 조회 시도
   → ❌ 다른 트랜잭션에선 안 보임 (아직 커밋 전!)
```

**문제 2: 이벤트 발행 후 트랜잭션 롤백**

```
1. order 저장
2. eventBus.publish(OrderCreated) → 핸들러들 처리 완료
3. 후속 로직 예외 → 트랜잭션 롤백
4. order는 사라졌는데 이벤트는 이미 처리됨
   → 결제됐는데 주문이 없는 상태 ❌
```

### 1.2 해결의 직관

> **"이벤트는 트랜잭션이 커밋된 직후에만 발행한다. 롤백되면 폐기한다."**

이걸 어떻게 짜는가? 직접 구현하려면:

```java
// 안 좋은 방식 — 수동
public void createOrder(...) {
    List<DomainEvent> pendingEvents = new ArrayList<>();
    boolean committed = false;
    try {
        // 트랜잭션 시작
        order = Order.create(...);
        orderRepository.save(order);
        pendingEvents.add(new OrderCreated(...));
        // 트랜잭션 커밋
        committed = true;
    } finally {
        if (committed) {
            pendingEvents.forEach(eventBus::publish);
        }
    }
}
```

이걸 모든 메서드에 일일이 박을 순 없다. **Spring이 트랜잭션의 라이프사이클 훅을 제공**하니 그걸 쓰면 된다.

---

## 2. Spring 트랜잭션 추상 한 장 정리

```
┌────────────────────────────────────────────────────────────┐
│  PlatformTransactionManager  ← 트랜잭션을 시작/커밋/롤백     │
│  (구현: JpaTransactionManager, DataSourceTransactionManager)│
│                                                            │
│  사용자가 직접 부르거나 @Transactional이 대신 부른다         │
└────────────────┬───────────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────────┐
│  TransactionSynchronizationManager   ← 정적 유틸 클래스      │
│                                                            │
│  ThreadLocal로 다음을 보관:                                 │
│   - 현재 스레드에 활성 트랜잭션이 있는가?                    │
│   - 등록된 TransactionSynchronization 콜백 목록              │
│   - 바인딩된 리소스 (DataSource → Connection 등)            │
└────────────────┬───────────────────────────────────────────┘
                 │ registerSynchronization(...)
                 ▼
┌────────────────────────────────────────────────────────────┐
│  TransactionSynchronization (인터페이스)                    │
│                                                            │
│  beforeCommit / beforeCompletion / afterCommit /           │
│  afterCompletion / suspend / resume / flush                │
│                                                            │
│  사용자가 구현해서 트랜잭션 라이프사이클 훅에 코드를 끼워넣음 │
└────────────────────────────────────────────────────────────┘
```

세 가지가 협력하는 구조. **트랜잭션 매니저**가 트랜잭션을 운전하고, **매니저(정적 유틸)** 가 ThreadLocal로 상태를 추적하고, **Synchronization**이 그 라이프사이클의 특정 시점에 끼어들 콜백을 제공한다.

---

## 3. `TransactionSynchronizationManager`

### 3.1 정적 유틸 클래스

```java
public abstract class TransactionSynchronizationManager {
    public static boolean isSynchronizationActive();
    public static void registerSynchronization(TransactionSynchronization s);
    public static List<TransactionSynchronization> getSynchronizations();
    public static boolean isActualTransactionActive();
    public static String getCurrentTransactionName();
    public static Integer getCurrentTransactionIsolationLevel();
    public static boolean isCurrentTransactionReadOnly();
    public static void bindResource(Object key, Object value);
    public static Object getResource(Object key);
    // ...
}
```

전부 `static`. **ThreadLocal에 저장된 트랜잭션 컨텍스트를 조회/조작**한다.

### 3.2 ThreadLocal로 추적하는 이유

스레드별로 트랜잭션 상태가 다를 수 있기 때문. 톰캣의 요청 스레드 풀을 생각하면:

```
[Thread-1]  요청 A 처리 중 — 트랜잭션 진행 중 (UPDATE orders)
[Thread-2]  요청 B 처리 중 — 트랜잭션 진행 중 (INSERT payments)
[Thread-3]  요청 C 처리 중 — 트랜잭션 없음 (단순 GET)
```

각 스레드가 자기 트랜잭션을 가지므로 **ThreadLocal**이 정확한 자료구조다. `TransactionSynchronizationManager`는 이걸 꺼내 보여주는 정적 게이트웨이일 뿐.

### 3.3 핵심 메서드 두 개

이 프로젝트에서 직접 쓰는 건 두 개뿐이다.

```java
TransactionSynchronizationManager.isSynchronizationActive()
// 현재 스레드에 활성 트랜잭션이 있고, 동기화(synchronization)가 활성화돼 있는가?

TransactionSynchronizationManager.registerSynchronization(sync)
// 활성 트랜잭션의 라이프사이클에 끼어들 콜백을 등록
```

> `isSynchronizationActive`와 `isActualTransactionActive`는 미묘하게 다르다. 전자는 "동기화 인프라가 활성화돼 있는가", 후자는 "실제 트랜잭션이 시작됐는가". 보통은 전자만 확인해도 충분하다.

---

## 4. `TransactionSynchronization` 인터페이스

### 4.1 라이프사이클 메서드들

```java
public interface TransactionSynchronization {

    // 보류/재개 (Propagation.REQUIRES_NEW 등에서 발화)
    default void suspend() {}
    default void resume() {}

    // 데이터 강제 플러시 (JPA 등에서)
    default void flush() {}

    // === 커밋 직전 ===
    default void beforeCommit(boolean readOnly) {}

    // === 트랜잭션 종료 직전 (커밋이든 롤백이든 호출됨) ===
    default void beforeCompletion() {}

    // === 커밋 성공 직후 (롤백 시엔 호출 안 됨!) ===
    default void afterCommit() {}

    // === 트랜잭션 종료 후 (커밋이든 롤백이든 호출됨) ===
    default void afterCompletion(int status) {}
        // status: STATUS_COMMITTED(0), STATUS_ROLLED_BACK(1), STATUS_UNKNOWN(2)
}
```

전부 `default` 빈 메서드라 필요한 것만 오버라이드하면 된다.

### 4.2 라이프사이클 시간순

```
[트랜잭션 진행 중]
        │
        │ 사용자 코드 실행 (save, update 등)
        │
        ▼
[커밋 결정 시점]
        │
        ├─ ① beforeCommit(readOnly)        ← 커밋 직전, 검증·플러시 같은 것
        │
        ├─ ② beforeCompletion()            ← 종료 직전 (커밋/롤백 공통)
        │
        ▼
[DB에 실제 커밋 또는 롤백]
        │
        ├─ ③ (커밋 성공 시만) afterCommit()  ← ⭐ 우리가 쓰는 훅
        │
        ├─ ④ afterCompletion(status)       ← 종료 후 (커밋/롤백 공통)
        │
        ▼
[트랜잭션 끝, ThreadLocal 정리]
```

**핵심 포인트**:

| 훅 | 커밋 시 | 롤백 시 | 용도 |
|----|--------|--------|------|
| `beforeCommit` | ✅ | ❌ | 커밋 직전 검증, 마지막 플러시 |
| `beforeCompletion` | ✅ | ✅ | 리소스 정리 (JDBC connection 반환 등) |
| **`afterCommit`** | **✅** | **❌** | **커밋이 확정된 후에만 할 일** ← 이 프로젝트 |
| `afterCompletion(status)` | ✅ | ✅ | 종료 후 정리·알림 (status로 분기 가능) |

### 4.3 `afterCommit`이 특별한 이유

이름 그대로 **"커밋이 성공했을 때만"** 호출된다. 롤백되면 호출되지 않는다.

→ 이벤트 발행 같은 **"커밋 확정 후에만 일어나야 하는 일"** 에 정확히 맞는 훅.

---

## 5. `TransactionalEventPublisher` 한 줄씩 풀기

```java
public void publish(DomainEvent event) {
    if (!TransactionSynchronizationManager.isSynchronizationActive()) {
        bus.publish(event);                                         // ①
        return;
    }
    TransactionSynchronizationManager.registerSynchronization(
            new TransactionSynchronization() {
                @Override public void afterCommit() {                // ②
                    bus.publish(event);
                }
            }
    );
}
```

### 5.1 ① 트랜잭션 밖이면 즉시 발행

```java
if (!TransactionSynchronizationManager.isSynchronizationActive()) {
    bus.publish(event);
    return;
}
```

호출자가 `@Transactional` 컨텍스트 밖이라면 트랜잭션이 없다 → 기다릴 이유 없음 → 즉시 발행.

이건 **fail-safe 설계**다. 누군가 트랜잭션 없이 이 메서드를 호출해도 동작은 한다(다만 보장은 약해짐).

### 5.2 ② 트랜잭션 안이면 afterCommit에 예약

```java
TransactionSynchronizationManager.registerSynchronization(
    new TransactionSynchronization() {
        @Override public void afterCommit() {
            bus.publish(event);
        }
    }
);
```

이게 핵심이다. 풀면:

1. **`new TransactionSynchronization() { ... }`** — 익명 클래스로 동기화 콜백 객체 생성. `afterCommit`만 오버라이드.
2. **`registerSynchronization(...)`** — 현재 스레드의 트랜잭션 동기화 매니저에 콜백 등록.
3. 등록 즉시 실행되는 게 아니라 **트랜잭션 매니저가 커밋을 끝낸 직후에 자동으로 호출**된다.
4. 만약 트랜잭션이 롤백되면 `afterCommit`은 호출되지 않으므로 **이벤트는 발행되지 않는다**.

### 5.3 람다 vs 익명 클래스

`TransactionSynchronization`은 **추상 메서드가 여러 개라 함수형 인터페이스가 아니다** (모두 default지만 `@FunctionalInterface`가 아님). 따라서 람다로 못 짠다.

```java
// ❌ 안 됨
registerSynchronization(() -> bus.publish(event));

// ✅ 익명 클래스 (위 코드처럼)
registerSynchronization(new TransactionSynchronization() {
    @Override public void afterCommit() { bus.publish(event); }
});
```

자바 21+에서는 익명 클래스 대신 record 같은 패턴을 쓰기도 하지만, 여기선 익명 클래스가 가장 읽기 좋다.

### 5.4 `event` 변수가 어떻게 익명 클래스 안에서 보이는가

```java
public void publish(DomainEvent event) {       // 메서드 매개변수
    ...
    new TransactionSynchronization() {
        @Override public void afterCommit() {
            bus.publish(event);                // ← 외부 매개변수 참조
        }
    };
}
```

자바의 **effectively final 캡처**. 익명 클래스(또는 람다) 안에서 외부 지역변수를 참조하려면 그 변수가 사실상 final이어야 한다. `event`는 매개변수라 재할당 안 되니 OK.

내부적으로는 익명 클래스 인스턴스가 `event`의 참조를 자기 필드에 들고 있는다. 그래서 메서드가 종료되어도 GC되지 않고, afterCommit에서 사용 가능하다.

---

## 6. 시간순 시나리오

### 6.1 정상 커밋 시나리오

```java
@Transactional
void createOrder(...) {
    Order order = Order.create(...);
    repo.save(order);
    publisher.publish(new OrderCreated(order.getId(), ...));   // 콜백 등록만
    // 메서드 정상 종료 → 트랜잭션 커밋
}
```

```
[T+0]   @Transactional → 트랜잭션 시작 (begin)
        TransactionSynchronizationManager에 동기화 활성화

[T+1]   repo.save(order) → INSERT (아직 커밋 전)

[T+2]   publisher.publish(OrderCreated) 호출
        ├─ isSynchronizationActive() == true
        └─ registerSynchronization(콜백) — afterCommit에 예약

[T+3]   메서드 종료 → 트랜잭션 매니저가 커밋 시작
        ├─ 등록된 synchronization들의 beforeCommit 호출
        ├─ DB 커밋 성공
        ├─ ⭐ afterCommit 발화 → bus.publish(event) 실제 실행
        │     └─ EventStore.append + 핸들러 디스패치
        └─ afterCompletion(STATUS_COMMITTED) 호출

[T+4]   ThreadLocal 정리
```

### 6.2 롤백 시나리오

```java
@Transactional
void createOrder(...) {
    Order order = Order.create(...);
    repo.save(order);
    publisher.publish(new OrderCreated(order.getId(), ...));   // 콜백 등록만
    throw new RuntimeException("oops");
}
```

```
[T+0]   @Transactional → 트랜잭션 시작
[T+1]   repo.save(order) — INSERT
[T+2]   publisher.publish — afterCommit 콜백 등록만
[T+3]   RuntimeException 발생 → @Transactional이 롤백 결정
        ├─ beforeCompletion() 호출
        ├─ DB 롤백
        ├─ ❌ afterCommit() 호출 안 됨 (커밋이 안 됐으니까)
        └─ afterCompletion(STATUS_ROLLED_BACK) 호출

[T+4]   ThreadLocal 정리

결과: 이벤트는 발행되지 않음. EventStore에 append되지 않음.
```

이게 정확히 `TransactionalEventPublishingTest.트랜잭션_롤백_시_이벤트는_폐기된다` 가 검증하는 동작이다.

---

## 7. Spring의 더 높은 추상 — `@TransactionalEventListener`

같은 일을 어노테이션 기반으로 할 수 있다.

```java
@Component
class OrderCreatedHandler {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void on(OrderCreated event) {
        // 커밋 후에만 호출됨
    }
}
```

`phase` 옵션:

| Phase | 의미 |
|------|------|
| `BEFORE_COMMIT` | beforeCommit 시점 |
| `AFTER_COMMIT` | afterCommit 시점 (기본값) |
| `AFTER_ROLLBACK` | 롤백 후 |
| `AFTER_COMPLETION` | 커밋/롤백 무관하게 종료 후 |

### 7.1 그런데 이 프로젝트는 왜 직접 만들었나

세 가지 이유:

1. **명시성**. `@TransactionalEventListener`는 호출 흐름이 어노테이션에 숨어 보이지 않는다. 직접 publisher를 만들면 **"트랜잭션과 이벤트 발행이 어떻게 엮이는지"** 코드 한 곳에서 보인다.

2. **EventStore 결합**. 이 프로젝트의 핵심은 단순 pub/sub가 아니라 **Store + Bus 분리**(`event-store-vs-simple-bus.md` 참고). `@TransactionalEventListener`는 Spring `ApplicationEvent`를 전제로 하므로 우리 EventBus와 직접 결합되지 않는다.

3. **학습 가치**. `TransactionSynchronization`을 직접 다뤄봐야 Spring 트랜잭션의 작동 원리가 이해된다. Phase 2에서 Outbox로 가는 진화도 이 구조 위에서 자연스럽다.

### 7.2 비교

| 측면 | `@TransactionalEventListener` | 직접 구현 (`TransactionalEventPublisher`) |
|------|----------------------------|-----------------------------------------|
| 코드량 | 어노테이션 한 줄 | 클래스 하나 |
| 명시성 | 낮음 (마법) | 높음 (코드로 보임) |
| 유연성 | 적음 (Spring 표준 사건 모델) | 많음 (자기 모델 가능) |
| 학습 가치 | 적음 | 큼 |
| 운영 안정성 | Spring이 검증 | 직접 책임 |

규모 큰 프로젝트엔 보통 `@TransactionalEventListener`로 충분하다. 이 프로젝트는 학습 + 진화 가능성을 위해 직접 만든다.

---

## 8. 자주 만나는 함정

### 8.1 `afterCommit` 안에서 새 트랜잭션이 필요한 경우

```java
@Override public void afterCommit() {
    bus.publish(event);   // 이 안에서 또 DB 작업이 필요하면?
}
```

`afterCommit` 시점엔 **이미 트랜잭션이 끝나있다**. 여기서 DB를 또 건드리려면 새 트랜잭션이 필요하다. 보통 다른 빈에 `@Transactional(propagation = REQUIRES_NEW)`을 걸어 호출한다.

### 8.2 `afterCommit`에서 예외가 나면

`afterCommit`에서 예외가 발생해도 **이미 커밋된 DB는 롤백되지 않는다**. 커밋은 끝났다. 예외는 위로 전파될 수 있지만 트랜잭션 결과는 그대로.

→ 그래서 `afterCommit` 안에선 **방어적으로 try/catch + 로깅** 또는 **재시도 큐로 보내기** 같은 처리가 권장된다. 이 프로젝트는 `InMemoryEventBus.publish` 안에서 이미 try/catch로 핸들러 예외를 격리한다.

### 8.3 비동기 처리는 직접 안 됨

`afterCommit`은 **트랜잭션 매니저 스레드**에서 동기 실행된다. 여기서 무거운 일을 하면 응답이 느려진다. 비동기로 처리하려면:

```java
@Override public void afterCommit() {
    asyncExecutor.execute(() -> bus.publish(event));
}
```

다만 이 경우 **이벤트 처리 결과의 보장이 약해진다** (비동기는 또 다른 문제). 트레이드오프.

### 8.4 트랜잭션 밖에서 호출했을 때 동작이 바뀜

```java
publisher.publish(event);    // @Transactional 안: afterCommit에 예약
publisher.publish(event);    // @Transactional 밖: 즉시 발행
```

같은 메서드 호출이지만 컨텍스트에 따라 동작이 다르다. **호출자가 항상 트랜잭션 안에 있다고 강제하고 싶다면** 다음과 같이 바꿀 수도 있다:

```java
public void publish(DomainEvent event) {
    if (!TransactionSynchronizationManager.isSynchronizationActive()) {
        throw new IllegalStateException("publish must be called within a transaction");
    }
    ...
}
```

선택은 정책 문제. 이 프로젝트는 fail-safe(둘 다 동작)를 택했다.

---

## 9. Phase 2와의 연결 — Outbox 패턴으로의 진화

이 프로젝트의 Phase 2에선 같은 사상을 한 단계 진화시킨다.

| | Phase 1 | Phase 2 |
|--|---------|---------|
| 발행 시점 | `afterCommit` 콜백 | DB의 outbox 테이블에 INSERT |
| 보장 수단 | Spring 트랜잭션 동기화 | DB 트랜잭션 자체 |
| 외부 시스템 | 같은 JVM 내 핸들러 | Kafka 발행 (Relay가 폴링) |
| 신뢰성 | JVM 살아있는 동안만 | DB가 살아있는 한 영구 |

핵심 차이는 **"메모리 위 콜백"** vs **"DB 행"**. Phase 1의 `afterCommit`이 일종의 **인메모리 outbox**라고 볼 수 있다 — 콜백 큐가 ThreadLocal이라 휘발성이지만, 같은 트랜잭션 경계로 묶인다는 사상은 동일.

JVM 크래시까지 견디려면 outbox로 가야 한다. 그게 Phase 2의 동기.

---

## 10. 한 줄 요약

> **`TransactionSynchronizationManager`는 ThreadLocal로 트랜잭션 컨텍스트를 추적하는 정적 게이트웨이이고, `TransactionSynchronization`은 그 트랜잭션의 라이프사이클(`beforeCommit`, `afterCommit`, `afterCompletion` 등)에 끼어들 콜백 인터페이스다. `TransactionalEventPublisher`는 이 둘을 조합해 "이벤트는 커밋 후에만 발행, 롤백 시 폐기"라는 강한 일관성 계약을 짧은 코드로 구현한다.**

이 메커니즘은 Spring의 `@TransactionalEventListener`도 내부적으로 같은 걸 쓴다. 한 번 이해해두면 트랜잭션 관련 마법이 전부 보인다.

---

## 11. 함께 보면 좋은 자료

- `docs/study/event-store-vs-simple-bus.md` — Store + Bus 분리 사상
- `docs/study/testcontainers.md` — `TransactionalEventPublishingTest`가 이 동작을 어떻게 증명하는가
- `docs/phase-1-tdd.md` Step 4 — TDD 사이클로 본 구현 과정
- `docs/phase-2-tdd.md` Step 2 — Outbox 패턴으로 진화하는 부분
- Spring 공식: https://docs.spring.io/spring-framework/reference/data-access/transaction/event.html
- Spring 소스: `org.springframework.transaction.support.TransactionSynchronization` (인터페이스가 짧으니 직접 읽어보면 좋다)
