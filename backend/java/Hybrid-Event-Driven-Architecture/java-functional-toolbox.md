# `Consumer`와 함께 보는 자바 표준 라이브러리 도구들

> `InMemoryEventBus`의 `subscribe` / `publish` 두 메서드 안에 들어있는 자바 표준 라이브러리 요소들을 한 줄씩 풀어 정리한 학습 노트.
> 함수형 인터페이스, 제네릭, 동시성 컬렉션, null 안전 처리, 예외 격리가 모두 짧은 코드 안에 모여 있다.

---

## 0. 분석 대상 코드

```java
// common/src/main/java/com/hybrid/common/event/InMemoryEventBus.java

@Override
@SuppressWarnings("unchecked")
public <T extends DomainEvent> void subscribe(Class<T> type, Consumer<T> handler) {
    handlers.computeIfAbsent(type, k -> new CopyOnWriteArrayList<>())
            .add((Consumer<DomainEvent>) handler);
}

@Override
public void publish(DomainEvent event) {
    store.append(event);
    for (Consumer<DomainEvent> h : handlers.getOrDefault(event.getClass(), List.of())) {
        try { h.accept(event); }
        catch (Exception e) { log.error("handler failed for {}", event, e); }
    }
}
```

이 코드가 어떻게 동작하는지 이해하려면 다음 7개 도구의 역할을 알아야 한다.

| # | 도구 | 정체 |
|---|------|------|
| 1 | `Consumer<T>` | 함수형 인터페이스 (java.util.function) |
| 2 | `<T extends DomainEvent>` | 경계 있는 제네릭 메서드 |
| 3 | `Map.computeIfAbsent` | "없으면 만들고 있으면 가져온다" 원자적 연산 |
| 4 | `(Consumer<DomainEvent>) handler` 캐스트 | 제네릭 타입 소거 우회 |
| 5 | `@SuppressWarnings("unchecked")` | unchecked 경고 억제 |
| 6 | `Map.getOrDefault` + `List.of()` | null 안전한 조회 + 불변 빈 리스트 |
| 7 | `try/catch`로 감싼 `accept` | 핸들러 예외 격리 |

---

## 1. `Consumer<T>` — 함수형 인터페이스

`java.util.function` 패키지의 표준 함수형 인터페이스다.

```java
@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);                                       // 추상 메서드 1개
    default Consumer<T> andThen(Consumer<? super T> after) { ... }
}
```

핵심:

- **추상 메서드가 단 하나** (`accept`) → `@FunctionalInterface` 표시 가능 → 람다·메서드 레퍼런스로 바로 만들 수 있음.
- **입력만 받고 반환은 없다** (`void`). "값을 소비한다"는 뜻에서 Consumer.

```java
Consumer<String> printer = s -> System.out.println(s);
printer.accept("hello");                       // hello

Consumer<String> printer2 = System.out::println;   // 메서드 레퍼런스
```

이 프로젝트에선 `Consumer<DomainEvent>`가 **이벤트 핸들러**의 타입이다. "이벤트를 받아서 뭐든 처리하지만 반환값은 없다"는 계약이 정확히 들어맞는다.

### 1.1 함수형 인터페이스 가족

| 인터페이스 | 시그니처 | 용도 |
|-----------|---------|------|
| `Consumer<T>` | `T → void` | 받아서 처리 (반환 없음) |
| `Supplier<T>` | `() → T` | 만들어서 줌 (입력 없음) |
| `Function<T,R>` | `T → R` | 변환 |
| `Predicate<T>` | `T → boolean` | 판정 |
| `BiConsumer<T,U>` | `(T,U) → void` | 두 개 받아 처리 |
| `BiFunction<T,U,R>` | `(T,U) → R` | 두 개 받아 변환 |
| `Runnable` | `() → void` | 입출력 없음, 그냥 실행 |
| `Callable<V>` | `() → V (throws)` | 결과 반환 + 예외 가능 |

`Consumer`는 "이벤트 핸들러", "Observer", "콜백"의 표준화된 형태로 가장 자주 쓰인다.

### 1.2 직접 인터페이스 만들지 않는 이유

```java
// 옛날 방식
interface EventHandler<T> {
    void handle(T event);
}

// 표준 방식
Consumer<T> handler;
```

표준 함수형 인터페이스를 쓰면:

- 람다·메서드 레퍼런스가 그대로 호환됨.
- `andThen`, `compose` 같은 합성 메서드를 무료로 얻음.
- 자바 생태계 표준이라 외부 라이브러리와 호환됨.

도메인적으로 의미 있는 이름이 꼭 필요하지 않다면 표준을 쓰는 게 낫다.

---

## 2. `<T extends DomainEvent>` — 제네릭 메서드 (타입 안전 등록)

```java
public <T extends DomainEvent> void subscribe(Class<T> type, Consumer<T> handler) {
```

이게 왜 필요한가:

```java
bus.subscribe(OrderCreated.class, e -> { /* e의 타입이 OrderCreated로 추론됨 */ });
bus.subscribe(PaymentCompleted.class, e -> { /* e는 PaymentCompleted */ });

// 컴파일 오류 — 타입 불일치 잡힘
bus.subscribe(OrderCreated.class, (PaymentCompleted e) -> { ... });
```

`<T extends DomainEvent>`로 묶었기 때문에 **`Class<T>`와 `Consumer<T>`가 같은 T**여야 한다고 컴파일러가 강제한다. 핸들러 등록 시점에 잘못된 타입 매칭을 차단하는 게 목적.

### 2.1 만약 제네릭 없이 만들었다면

```java
// 안 좋은 설계
void subscribe(Class<?> type, Consumer<DomainEvent> handler) { ... }

// 사용처에서 매번 캐스팅 필요
bus.subscribe(OrderCreated.class, e -> {
    OrderCreated o = (OrderCreated) e;   // 매번 수동 캐스트
    ...
});
```

제네릭 메서드를 쓰면 **호출 측이 깔끔해지고, 타입 오류는 컴파일 타임에 잡힌다.**

---

## 3. `Map.computeIfAbsent` — "없으면 만들고 있으면 가져온다"

```java
handlers.computeIfAbsent(type, k -> new CopyOnWriteArrayList<>())
        .add((Consumer<DomainEvent>) handler);
```

이걸 `computeIfAbsent` 없이 쓰면 4줄이 된다:

```java
List<Consumer<DomainEvent>> list = handlers.get(type);
if (list == null) {
    list = new CopyOnWriteArrayList<>();
    handlers.put(type, list);
}
list.add(handler);
```

`computeIfAbsent` 동작:

```
1. type 키로 조회
2. 값이 있으면 그대로 반환
3. 값이 없으면 람다(k -> new CopyOnWriteArrayList<>()) 실행 → put → 반환
4. 어떤 경우든 List를 반환하므로 .add(handler)로 핸들러 추가
```

### 3.1 `ConcurrentHashMap`에서의 원자성

`ConcurrentHashMap.computeIfAbsent`는 **원자적**이다. 여러 스레드가 동시에 같은 키로 호출해도 람다는 **정확히 한 번만 실행**된다. 그래서 동일 이벤트 타입에 대한 첫 등록 시 List가 두 번 만들어지는 일이 없다.

> 학습 노트 `concurrent-collections.md`의 "ConcurrentHashMap의 버킷 락"이 정확히 여기서 작동한다.

### 3.2 비슷한 가족 메서드

| 메서드 | 동작 |
|-------|------|
| `computeIfAbsent(k, fn)` | 없을 때만 계산해서 넣음 |
| `computeIfPresent(k, fn)` | 있을 때만 계산해서 갱신 |
| `compute(k, fn)` | 무조건 계산해서 갱신 |
| `merge(k, value, fn)` | 없으면 value, 있으면 fn으로 합침 |
| `putIfAbsent(k, value)` | 없으면 value 넣음 (계산 없이) |

상황에 맞게 쓰면 if-else 분기를 한 줄로 줄일 수 있다.

---

## 4. `(Consumer<DomainEvent>) handler` — 안전하지 않은 캐스트

문제 상황:

- 들어온 `handler` 타입은 `Consumer<T>` (예: `Consumer<OrderCreated>`).
- 저장하는 List 타입은 `List<Consumer<DomainEvent>>`.
- `Consumer<OrderCreated>`를 `Consumer<DomainEvent>`로 직접 변환할 수 없다 — **자바 제네릭은 invariant**(불변성)이라 `Consumer<OrderCreated> ≠ Consumer<DomainEvent>`.

### 4.1 자바 제네릭의 invariance

```java
List<Object> list = new ArrayList<String>();    // ❌ 컴파일 오류
Consumer<DomainEvent> c = handler;              // ❌ Consumer<OrderCreated>는 다른 타입
```

`Object`가 `String`의 부모여도, `List<Object>`는 `List<String>`의 부모가 아니다.

### 4.2 타입 소거 (Type Erasure)

자바 제네릭은 컴파일 후 타입 정보가 사라지는 **타입 소거(erasure)** 모델이다. 런타임엔 둘 다 그냥 `Consumer`다.

```java
// 컴파일 전
Consumer<OrderCreated> c = ...;
Consumer<DomainEvent> d = ...;

// 컴파일 후 (런타임)
Consumer c = ...;
Consumer d = ...;
```

그래서 `(Consumer<DomainEvent>) handler` 캐스트는 **이론적으론 위험하지만 실제론 안전**하다 — `publish` 시점에 `event.getClass()` 키로 정확히 같은 타입의 핸들러만 꺼내 호출하기 때문.

### 4.3 더 안전한 대안

```java
// 옵션 1: TypedHandler 래퍼로 감싸기
record TypedHandler<T extends DomainEvent>(Class<T> type, Consumer<T> handler) {
    @SuppressWarnings("unchecked")
    void invoke(DomainEvent event) {
        if (type.isInstance(event)) handler.accept((T) event);
    }
}
```

phase-1-tdd.md의 Refactor 절에서 언급한 개선안이 이 방향이다. 지금은 단순함을 위해 unchecked 캐스트로 두고, 추상화는 필요해질 때 도입한다.

---

## 5. `@SuppressWarnings("unchecked")` — 컴파일러에게 책임 인수

```java
@SuppressWarnings("unchecked")
public <T extends DomainEvent> void subscribe(...) {
    ...
    .add((Consumer<DomainEvent>) handler);
}
```

자바 컴파일러는 `(Consumer<DomainEvent>) handler` 같은 unchecked 캐스트를 보면 경고를 띄운다. 이 어노테이션은 **"이 캐스트가 unchecked인 줄 알지만, 내가 책임지니 경고 끄라"** 는 표시.

### 5.1 자주 쓰는 카테고리

| 값 | 억제 대상 |
|-----|---------|
| `unchecked` | 제네릭 unchecked 캐스트/연산 |
| `rawtypes` | 원시 타입 사용 (`List` 같은 형태) |
| `deprecation` | `@Deprecated` 호출 |
| `unused` | 사용되지 않는 변수/메서드 |
| `serial` | `serialVersionUID` 누락 |
| `all` | 전부 (지양 — 너무 광범위) |

**가능한 한 좁은 범위(메서드 레벨)에 붙이고, 광범위(클래스 레벨)는 피한다.** 경고가 알려주는 진짜 문제를 가릴 수 있다.

---

## 6. `Map.getOrDefault` + `List.of()` — 안전한 기본값

```java
for (Consumer<DomainEvent> h : handlers.getOrDefault(event.getClass(), List.of())) {
```

- **`getOrDefault(key, default)`**: 키가 없으면 default를 반환. 우리 경우엔 빈 리스트.
- **`List.of()`**: **불변 빈 리스트** (Java 9+). 인스턴스가 캐시되어 매번 새로 만들지 않음.

이게 없었다면:

```java
List<Consumer<DomainEvent>> list = handlers.get(event.getClass());
if (list != null) {
    for (Consumer<DomainEvent> h : list) { ... }
}
```

`getOrDefault` + `List.of()` 조합으로 **null 체크 없이 안전하게 순회**한다. 빈 리스트면 for 루프가 한 번도 안 돌고 끝.

### 6.1 `Collections.emptyList()` vs `List.of()`

```java
Collections.emptyList()    // Java 1.5+, 캐시된 싱글톤
List.of()                  // Java 9+, 캐시된 싱글톤
```

기능적으론 동일. `List.of()`가 더 짧고 모던한 자바 스타일이고, `List.of(1,2,3)`처럼 값 있는 리스트도 같은 패밀리로 만들 수 있다는 게 장점.

### 6.2 `List.of()`의 특징

- **불변(immutable)**. `add`, `remove` 호출 시 `UnsupportedOperationException`.
- **null 거부**. `List.of(null)` 시 `NullPointerException`.
- **컴팩트**. 작은 사이즈 전용 최적화 인스턴스가 있음 (`List12`, `ListN` 등).

---

## 7. `try/catch`로 감싼 `accept` — 핸들러 예외 격리

```java
try { h.accept(event); }
catch (Exception e) { log.error("handler failed for {}", event, e); }
```

이유는 단순하다 — **한 핸들러의 예외가 다른 핸들러를 막지 않게** 격리한다. 한 구독자가 폭발해도 나머지는 정상 진행.

### 7.1 왜 `Exception`인가, `Throwable`이 아닌가

```java
try { ... }
catch (Exception e) { ... }       // 일반 예외만 캐치
// vs
catch (Throwable t) { ... }       // Error도 포함
```

`Throwable`까지 잡으면 `OutOfMemoryError`, `StackOverflowError` 같은 **JVM 차원의 치명적 오류도 삼켜버린다**. 보통 회복 불가능한 상황이라 잡지 않는 게 표준. `Exception`까지만 잡고 `Error`는 그대로 위로 올라가게 둔다.

### 7.2 더 정교한 정책의 가능성

이 프로젝트의 phase-1-tdd.md Step 3 Refactor 절에 언급된 개선안:

```java
@FunctionalInterface
interface EventErrorHandler {
    void handle(DomainEvent event, Consumer<DomainEvent> handler, Exception error);
}

// 기본: 로그
EventErrorHandler defaultPolicy = (e, h, ex) -> log.error(...);

// 또는: 메트릭 + 알림 + DLQ로 보내기
EventErrorHandler productionPolicy = (e, h, ex) -> {
    meter.counter("event.handler.failure").increment();
    deadLetterQueue.send(e);
    log.error(...);
};
```

지금은 "그냥 로그"가 충분하지만, 운영 환경에선 **에러 정책 자체가 정책 객체**가 되는 게 일반적.

---

## 8. 코드 흐름 다이어그램

### 8.1 subscribe 흐름

```
bus.subscribe(OrderCreated.class, paymentHandler::onOrderCreated)
                 │                          │
                 ▼                          ▼
       handlers (ConcurrentHashMap)    Consumer<OrderCreated>
                 │
                 ▼
       computeIfAbsent(OrderCreated.class, k -> new CopyOnWriteArrayList<>())
                 │
                 ├─ 키 없으면 → 새 CoW 리스트 만들어 put → 반환
                 └─ 키 있으면 → 기존 리스트 반환
                 │
                 ▼
              .add((Consumer<DomainEvent>) handler)
                 │
                 ▼
       handlers = {
           OrderCreated.class     → [paymentHandler::onOrderCreated],
           PaymentCompleted.class → [orderHandler::onPaymentCompleted, ...]
       }
```

### 8.2 publish 흐름

```
bus.publish(new OrderCreated(...))
        │
        ▼
   store.append(event)        ← Store에 영속화 (offset 부여)
        │
        ▼
   handlers.getOrDefault(OrderCreated.class, List.of())
        │
        ├─ 핸들러 있으면 → CopyOnWriteArrayList 반환
        └─ 없으면        → List.of() (불변 빈 리스트)
        │
        ▼
   for each h:
       h.accept(event)        ← Consumer.accept 호출
       (예외 시 로깅하고 다음 핸들러로 진행)
```

---

## 9. 도구 요약

| 도구 | 역할 | 이 코드에서의 사용 |
|------|------|------------|
| `Consumer<T>` | 입력만 받는 함수형 인터페이스 | 핸들러의 표준 타입 |
| `Consumer.accept(T)` | Consumer의 추상 메서드 | 핸들러 실행 |
| `<T extends X>` | 경계 있는 제네릭 메서드 | 타입 안전한 등록 |
| `Map.computeIfAbsent` | "없으면 만들고 가져온다" 원자적 연산 | 타입별 핸들러 리스트 lazy 초기화 |
| `Map.getOrDefault` | null 안전한 조회 | 핸들러 없는 타입을 빈 순회로 처리 |
| `List.of()` | 불변 빈 리스트 (Java 9+) | 기본값으로 빈 리스트 제공 |
| 제네릭 unchecked 캐스트 | 타입 소거 모델의 한계 우회 | 핸들러를 공통 타입 List에 저장 |
| `@SuppressWarnings("unchecked")` | unchecked 경고 억제 | 위 캐스트의 컴파일러 경고 끔 |
| `try/catch (Exception)` | 핸들러 예외 격리 | 한 핸들러 실패가 다른 핸들러를 막지 않게 |

---

## 10. 한 줄 요약

> **`Consumer<T>`는 자바의 "void 반환 핸들러" 표준 함수형 인터페이스이고, 이 코드는 그 핸들러들을 `ConcurrentHashMap` + `CopyOnWriteArrayList`에 타입별로 보관해뒀다가 publish 시 `getClass()`로 찾아 `accept`를 호출한다. `computeIfAbsent`·`getOrDefault`·`List.of()`는 nullness와 동시성 문제를 짧은 코드로 우회하는 자바 표준 라이브러리의 작은 도구들이다.**

이 짧은 메서드 두 개에 자바 함수형 프로그래밍, 제네릭, 동시성 컬렉션, null 안전, 예외 격리가 전부 들어있다. 표준 라이브러리 사용의 좋은 연습 사례다.

---

## 11. 함께 보면 좋은 자료

- `docs/study/concurrent-collections.md` — `ConcurrentHashMap`과 `CopyOnWriteArrayList`의 락 전략
- `docs/study/event-store-vs-simple-bus.md` — 이 핸들러 맵이 단순 버스가 아니라 Store와 짝을 이루는 이유
- `docs/phase-1-tdd.md` Step 3 — `InMemoryEventBus`의 TDD 구현 사이클
- Brian Goetz, *Java Concurrency in Practice* — `ConcurrentHashMap`과 함수형 합성의 안전한 사용
- *Effective Java* (Joshua Bloch) — Item 42~44 (람다와 함수형 인터페이스)
