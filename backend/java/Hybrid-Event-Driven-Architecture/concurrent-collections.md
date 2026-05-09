# 동시성 컬렉션 — `CopyOnWriteArrayList`를 중심으로

> 이 프로젝트의 `InMemoryEventBus`가 사용하는 동시성 자료구조를 이해하기 위한 학습 노트.
> Java `java.util.concurrent` 컬렉션들이 내부적으로 어떤 락 전략으로 안전성을 만드는가에 초점.

---

## 1. `CopyOnWriteArrayList`란

자바 동시성 패키지(`java.util.concurrent`)에 있는 **스레드 세이프한 List 구현체**다. 이름 그대로 **"쓰기 시 복사(Copy-On-Write)"** 전략을 쓴다.

```
읽기 (get, iterator):  락 없음. 현재 배열 참조를 그대로 사용
쓰기 (add, set, remove): 내부 배열을 통째로 새로 복사 → 수정 → 참조 교체
```

### 1.1 `ArrayList`와 비교

| 구분 | ArrayList | CopyOnWriteArrayList |
|------|-----------|---------------------|
| 스레드 세이프 | ❌ | ✅ |
| 읽기 비용 | O(1) | O(1), 락 없음 → 매우 빠름 |
| 쓰기 비용 | O(1) amortized | **O(n)** — 매번 전체 배열 복사 |
| iterator 동작 | 수정 중이면 `ConcurrentModificationException` | 스냅샷 시점 기준, 예외 없음 |
| 메모리 | 한 벌 | 쓰기 순간 두 벌(잠깐) |

### 1.2 언제 쓰는가

**읽기가 압도적으로 많고 쓰기는 드문** 경우.

- 이벤트 핸들러 등록 같은 **구독자 리스트**: 등록은 한두 번, 발행 시마다 순회.
- Listener 패턴, 옵저버 패턴.
- 설정값처럼 거의 불변에 가까운 데이터.

**쓰기가 잦으면 절대 쓰면 안 된다.** 매 쓰기마다 전체 복사라 성능이 무너진다.

---

## 2. 내부 동작 — 락은 어디에 있는가

`CopyOnWriteArrayList`의 모든 쓰기 메서드는 **하나의 `ReentrantLock`** 을 잡고 들어간다.

```java
// JDK 소스 단순화
private transient volatile Object[] array;
private final ReentrantLock lock = new ReentrantLock();

public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();                          // 1. 락 먼저 잡는다
    try {
        Object[] es = getArray();         // 2. 락 잡은 뒤에 현재 배열을 읽는다
        int len = es.length;
        Object[] newElements = Arrays.copyOf(es, len + 1);
        newElements[len] = e;
        setArray(newElements);            // 3. 새 배열로 참조 교체
        return true;
    } finally {
        lock.unlock();
    }
}
```

순서가 핵심이다. **락 → 읽기 → 복사 → 쓰기 → 언락**.

---

## 3. 동시 쓰기는 어떻게 처리되는가

### 3.1 자주 하는 오해

> "T1과 T2가 동시에 빈 리스트에 add하면, 둘 다 빈 배열을 본 뒤 각자 복사해서 한 명이 덮어쓰는 거 아닌가?"

**그렇지 않다.** 락 안에서 배열을 읽기 때문에 **후행 스레드는 항상 선행 스레드의 결과를 본다.**

### 3.2 실제 타임라인

```
T1: add("A")                    T2: add("B")
─────────────────────           ─────────────────────
lock.lock() ✅ 획득
                                lock.lock() ⏳ 대기...
es = getArray() → []
copy → ["A"]
setArray(["A"])
lock.unlock()
                                lock.lock() ✅ 획득
                                es = getArray() → ["A"]   ← T1이 setArray한 결과
                                copy → ["A", "B"]
                                setArray(["A","B"])
                                lock.unlock()

결과: ["A", "B"]   ✅ 둘 다 살아남음
```

T2가 "빈 배열"을 본다는 가정이 틀린 지점은 — **T2는 락이 풀릴 때까지 대기하다가 락을 잡은 직후에야 배열을 읽기 때문에**, 그 시점에는 이미 T1의 setArray가 반영되어 있다.

### 3.3 보장의 근거 — volatile + happens-before

내부 배열 필드가 `volatile`이고, `lock.unlock()`과 `lock.lock()`은 **happens-before 관계**를 만든다(JMM 규칙).

```
T1: setArray(["A"]) → lock.unlock()
                         │
                         ▼ happens-before
T2:                   lock.lock() → getArray()  ← 반드시 ["A"]를 본다
```

CPU 캐시에 박혀서 못 보는 일은 없다. JVM 메모리 모델이 가시성을 보장한다.

### 3.4 만약 락이 없었다면

가설적으로 락이 없는 copy-on-write라면 정확히 우려한 시나리오가 발생한다:

```
T1: read [] → copy → ["A"] → setArray(["A"])
T2: read [] → copy → ["B"] → setArray(["B"])  ← T1의 결과를 덮어씀

결과: ["B"]   ❌ A 유실
```

이게 "락 없는 CoW"의 함정이라 `CopyOnWriteArrayList`는 **반드시 락**을 쓴다. 읽기는 락 없이 빠르게, 쓰기는 락 안에서 안전하게 — 비대칭 설계.

### 3.5 동시 쓰기의 진짜 비용

쓰기가 동시에 N번 들어오면:

```
순차로 처리됨 (락 직렬화)
+ 매 쓰기마다 배열 전체 복사 (O(n))
= 총 비용 O(N × n)
```

배열 크기가 커질수록, 동시 쓰기가 많을수록 **급격히 느려진다.** 그래서 "쓰기 가뭄" 워크로드에만 쓰라는 것이다.

---

## 4. 읽기는 어떻게 안전한가

읽기는 락을 잡지 않는다. 그래도 안전한 이유:

- 새 배열을 **다 만든 뒤** 한 번에 참조를 교체한다 (`setArray`).
- `array` 필드가 `volatile`이므로, 새 참조 쓰기는 다른 스레드에 즉시 보인다.
- 읽기 스레드는 **어느 시점의 일관된 스냅샷**을 본다. 반쯤 복사된 배열을 보는 일은 없다.

### 4.1 iterator의 특별한 동작

```java
List<String> list = new CopyOnWriteArrayList<>(List.of("A","B"));

Iterator<String> it = list.iterator();   // 이 시점 배열 ["A","B"] 스냅샷을 잡음
list.add("C");                            // 새 배열 ["A","B","C"]가 됨

while (it.hasNext()) System.out.println(it.next());
// 출력: A, B   (C는 안 보임 — iterator 생성 시점 스냅샷 기준)
```

iterator는 만들어진 **그 시점의 배열**을 끝까지 사용한다. 그래서 `ConcurrentModificationException`이 안 난다. 대신 iterator로 `remove()` 같은 수정은 **불가능** — `UnsupportedOperationException`을 던진다.

---

## 5. 다른 동시성 컬렉션의 락 전략

`java.util.concurrent` 컬렉션들은 **각자 다른 전략의 락을 내부에 숨기고 있다.** 사용자는 그냥 List, Map처럼 쓰지만 안에서는 락 전략이 제각각이다.

| 컬렉션 | 락 전략 | 특징 |
|--------|---------|------|
| `CopyOnWriteArrayList` | **단일 ReentrantLock** + 쓰기 시 배열 통째 복사 | 읽기 락 없음, 쓰기 직렬화 |
| `ConcurrentHashMap` | **버킷별 락(Java 8+)** + CAS | 다른 버킷 쓰기는 동시 진행 가능 |
| `ConcurrentLinkedQueue` | **락 자체가 없음 (lock-free)** + CAS만 사용 | 모든 연산이 비차단 |
| `Collections.synchronizedList` | **메서드 단위 synchronized** (객체 전체 락) | 가장 단순, 가장 거친 |
| `LinkedBlockingQueue` | **putLock + takeLock** 두 개 분리 | 생산자/소비자 동시 진행 가능 |
| `ArrayBlockingQueue` | **단일 락 + 두 Condition** | put/take 같은 락 공유 |

### 5.1 진화의 방향

거친 락 → 잘게 쪼갠 락 → 락 없음(CAS)으로 발전했다.

```
synchronized 전체    →   부분 락(스트라이핑)   →   CAS / lock-free
(Collections.sync~)      (ConcurrentHashMap)        (ConcurrentLinkedQueue)
       ↑                        ↑                          ↑
  단순/안전           동시성 ↑               경합 적을 때 최고
  경합 시 느림         락 관리 복잡            구현 어려움
```

`ConcurrentHashMap`은 특히 흥미롭다. **Java 7까지는 16개 segment 락**으로 쪼개다가, **Java 8부터는 버킷(bin) 단위 락 + CAS**로 더 잘게 쪼갰다. 같은 키만 안 건드리면 동시 쓰기가 거의 막히지 않는다.

### 5.2 `synchronized` vs `ReentrantLock`

JDK 내부 컬렉션이 옛날 `Vector`처럼 `synchronized`를 쓰지 않고 `ReentrantLock`을 쓰는 이유:

- `tryLock(timeout)`으로 **타임아웃** 가능
- `lockInterruptibly()`로 **인터럽트** 가능
- `new ReentrantLock(true)`로 **공정성(FIFO)** 옵션
- 여러 `Condition` 분리 가능 (대기/통지를 세분화)

`synchronized`는 단순하고 빠르지만 이런 세밀한 제어가 안 된다.

---

## 6. 이 프로젝트와의 연결

`InMemoryEventBus`는 두 종류의 동시성 컬렉션을 함께 쓴다.

```java
// common/src/main/java/com/hybrid/common/event/InMemoryEventBus.java
private final Map<Class<? extends DomainEvent>, List<Consumer<DomainEvent>>> handlers
        = new ConcurrentHashMap<>();

@Override
public <T extends DomainEvent> void subscribe(Class<T> type, Consumer<T> handler) {
    handlers.computeIfAbsent(type, k -> new CopyOnWriteArrayList<>())
            .add((Consumer<DomainEvent>) handler);
}
```

| 자료구조 | 선택 이유 |
|---------|---------|
| `ConcurrentHashMap` (handlers 맵) | 서로 다른 이벤트 타입에 대한 등록이 **동시에** 일어날 수 있어야 함. 버킷 락이 적합 |
| `CopyOnWriteArrayList` (핸들러 리스트) | 같은 타입 핸들러는 **등록은 가뭄(부팅 시)**, **순회는 폭우(매 publish)**. 정확히 CoW 워크로드 |

### 6.1 구체적인 안전성

```java
@Override
public void publish(DomainEvent event) {
    store.append(event);
    for (Consumer<DomainEvent> h : handlers.getOrDefault(event.getClass(), List.of())) {
        try { h.accept(event); }
        catch (Exception e) { log.error("handler failed for {}", event, e); }
    }
}
```

이 순회 도중에 다른 스레드가 `subscribe`를 호출해도:

- `ConcurrentModificationException` → **나지 않음** (CoW 스냅샷 iterator)
- 새로 등록된 핸들러 → **이번 publish엔 안 보이고, 다음 publish부터 보임**
- 자료 깨짐 → **없음** (락이 쓰기를 직렬화)

만약 `synchronizedList`로 했다면 순회 전체를 직접 `synchronized` 블록으로 감싸야 했고, 그 사이 `subscribe`가 막혔을 것이다.

---

## 7. 한 줄 요약

| 개념 | 요약 |
|------|------|
| `CopyOnWriteArrayList` | "읽기 폭발 + 쓰기 가뭄"에 최적. 읽기 락 없음, 쓰기는 단일 락 + 전체 복사 |
| 동시 쓰기 | **순차로 다 반영됨**. 락이 직렬화하므로 유실 없음 |
| 락 위치의 중요성 | `getArray()`를 락 **안**에서 호출하기 때문에 후행 스레드가 선행 결과를 본다 |
| 동시성 컬렉션 일반 | "보통 컬렉션 인터페이스 + 내부에 숨겨진 락/CAS 전략". 워크로드에 맞게 골라 써야 함 |

---

## 8. 더 읽을 거리

- Java Memory Model 명세 (JLS Chapter 17)
- Brian Goetz, *Java Concurrency in Practice*
- JDK 소스 코드 — `java.util.concurrent.CopyOnWriteArrayList`, `java.util.concurrent.ConcurrentHashMap`
- ConcurrentHashMap의 Java 7→8 진화: segment lock → bin-level CAS 리팩터링
