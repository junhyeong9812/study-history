# 동시성 인-메모리 저장소 설계 가이드

> 자바에서 thread-safe하고 성능 좋은 in-memory 저장소를 만들 때 알아야 할 원칙들

---

## 목차

1. [왜 어려운가](#1-왜-어려운가)
2. [핵심 원칙](#2-핵심-원칙)
3. [동시성 도구 선택](#3-동시성-도구-선택)
4. [메모리 관리](#4-메모리-관리)
5. [API 설계](#5-api-설계)
6. [흔한 함정과 회피법](#6-흔한-함정과-회피법)
7. [테스트와 검증](#7-테스트와-검증)
8. [체크리스트](#8-체크리스트)

---

## 1. 왜 어려운가

### 동시성 버그의 특성

동시성 버그는 일반 버그와 다르게 매우 까다로워요:

- **재현이 어려움**: 타이밍에 따라 발생하므로 "내 PC에선 잘 되는데"가 흔함
- **테스트로 찾기 어려움**: 단위 테스트로는 잡히지 않음
- **운영 환경에서만 터짐**: 부하가 올라갈 때 처음 드러남
- **데이터 손상**: 락이 빠져서 데이터가 깨지면 복구가 어려움

그래서 **처음부터 잘 설계**해야 해요. 나중에 "동시성 좀 추가하자"는 거의 불가능.

### 인-메모리 저장소 특유의 도전

| 문제 | 설명 |
|---|---|
| **메모리 한계** | 프로세스 메모리는 한정적. 무한정 쌓이면 OOM |
| **재시작 시 소실** | 디스크 저장소가 아니므로 데이터 휘발성 |
| **GC 압력** | 객체가 많을수록 GC 부담 증가 |
| **단일 노드** | 분산 안 됨. 노드가 죽으면 데이터도 같이 |

---

## 2. 핵심 원칙

### 원칙 1: 불변성을 최대한 활용하라

**불변 객체는 동시성의 마법**이에요. 한 번 만들고 안 바꾸면 동기화가 필요 없음.

```java
// ❌ 가변 객체: 보호 필요
public class Event {
    private String type;
    private Object payload;
    public void setType(String t) { this.type = t; }
}

// ✅ 불변 객체 (record): 안전하게 공유 가능
public record StoredEvent(
    ChannelKey channel,
    long seq,
    Instant ts,
    String type,
    Object payload
) {}
```

저장소에 들어가는 **데이터 객체는 record나 final 클래스**로 만들어요.

### 원칙 2: 공유 상태를 최소화하라

스레드 간에 공유하는 변수가 적을수록 버그가 줄어요.

```java
// ❌ 한 큰 자료구조에 모든 게 묶임
class GlobalStore {
    Map<String, List<Event>> allEvents = new HashMap<>();
    // 모든 채널이 한 맵에 → 락 경합 심함
}

// ✅ 채널별로 격리
class ChannelStore {
    ConcurrentMap<ChannelKey, ChannelLog> logs;
    // 채널마다 독립적인 ChannelLog → 채널 A 작업이 채널 B 안 막음
}
```

### 원칙 3: 락보다는 동시성 자료구조

가능하면 직접 `synchronized`를 쓰지 말고, **이미 검증된 동시성 자료구조**를 쓰세요.

| 대신 쓰지 말아야 할 것 | 권장 |
|---|---|
| `synchronized HashMap` | `ConcurrentHashMap` |
| `Vector`, `Hashtable` (구식) | `ConcurrentHashMap`, `CopyOnWriteArrayList` |
| `synchronized` + `int` | `AtomicInteger`, `AtomicLong` |
| 수동 wait/notify | `BlockingQueue`, `CompletableFuture` |
| 직접 만든 큐 | `ConcurrentLinkedQueue/Deque`, `LinkedBlockingQueue` |

JDK가 제공하는 동시성 도구는 수년간 검증돼서 **직접 만드는 것보다 거의 항상 빠르고 안전**해요.

### 원칙 4: 일관성 모델을 명시적으로 결정하라

"얼마나 정확하면 충분한가?"를 의식적으로 정해야 해요.

| 일관성 수준 | 의미 | 비용 |
|---|---|---|
| **Strong (선형성)** | 모든 작업이 어떤 순간에 즉시 보임 | 매우 비쌈 |
| **Sequential** | 모든 스레드가 같은 순서로 봄 | 비쌈 |
| **Eventually consistent** | 결국 같아짐. 잠깐은 다를 수 있음 | 저렴 |
| **Weakly consistent** | 순회 중 변경된 게 보일 수도 안 보일 수도 | 매우 저렴 |

실시간 이벤트 시스템처럼 **"잠깐 어긋나도 괜찮은"** 도메인이라면 weakly consistent를 받아들이는 게 성능에 큰 도움이 돼요.

```java
// ConcurrentLinkedDeque의 size()는 O(n)이고 순회 중 정확하지 않을 수 있음
// 하지만 실시간 시스템에선 "대략 얼마나 쌓였는지"만 알면 충분
```

### 원칙 5: Lock-free를 추구하되 맹신하지 말라

Lock-free가 항상 빠른 건 아니에요:

- **경쟁이 적을 때**: lock-free가 빠름 (락 비용 없음)
- **경쟁이 심할 때**: CAS 재시도 폭증으로 오히려 느려질 수 있음
- **복잡한 변경**: 여러 필드를 함께 바꿔야 한다면 락이 더 명료할 때도 있음

**측정**하고 결정하세요. 직관은 종종 틀려요.

---

## 3. 동시성 도구 선택

### `ConcurrentHashMap`

**언제 쓰나**: 동시 접근이 필요한 키-값 저장소.

```java
ConcurrentMap<String, ChannelLog> logs = new ConcurrentHashMap<>();

// ✅ 원자적 "없으면 생성" - race 없음
ChannelLog log = logs.computeIfAbsent(key, k -> new ChannelLog());

// ✅ 원자적 업데이트
logs.compute(key, (k, v) -> v == null ? newLog() : updated(v));

// ❌ 두 단계로 나누면 race 발생
if (!logs.containsKey(key)) {           // 다른 스레드가 끼어들면
    logs.put(key, new ChannelLog());    // 두 번 만들어질 수 있음
}
```

**핵심 메서드**:
- `computeIfAbsent(k, fn)`: 없을 때만 생성 (원자적)
- `compute(k, fn)`: 항상 변환 적용 (원자적)
- `merge(k, v, fn)`: 기존 값과 새 값을 병합 (원자적)
- `putIfAbsent(k, v)`: 없을 때만 put (원자적)

### `ConcurrentLinkedQueue` / `ConcurrentLinkedDeque`

**언제 쓰나**: 락 없이 양쪽에서 추가/제거가 필요한 큐.

```java
Deque<Event> events = new ConcurrentLinkedDeque<>();

events.add(event);              // 끝에 추가
events.pollFirst();             // 머리에서 제거 (없으면 null)
events.peekFirst();             // 머리 들여다보기 (없으면 null)
```

**주의사항**:
- `size()`는 **O(n)**이에요. 자주 호출하면 비쌈
- 순회 중 변경에 대해 **약한 일관성** (예외는 안 나지만 정확한 스냅샷도 아님)
- 멤버 검사(`contains`)도 O(n)

### `BlockingQueue` 계열

**언제 쓰나**: 생산자-소비자 패턴. 큐가 비었을 때 자동으로 대기.

```java
BlockingQueue<Task> queue = new LinkedBlockingQueue<>(1000);

// 생산자
queue.put(task);                // 큐가 꽉 차면 대기
queue.offer(task, 1, SECONDS);  // 1초까지만 대기, 실패 시 false

// 소비자
Task t = queue.take();          // 비어있으면 대기
Task t = queue.poll(1, SECONDS); // 1초까지만 대기
```

**선택지**:
- `LinkedBlockingQueue`: 연결 리스트 기반. 용량 제한 가능
- `ArrayBlockingQueue`: 배열 기반. 용량 고정
- `SynchronousQueue`: 용량 0. 핸드오프 전용
- `PriorityBlockingQueue`: 우선순위 큐

### `Atomic*` 클래스

**언제 쓰나**: 단일 변수에 대한 원자적 read-modify-write.

```java
AtomicLong counter = new AtomicLong(0);

counter.incrementAndGet();              // ++counter
counter.getAndIncrement();              // counter++
counter.addAndGet(5);                   // counter += 5
counter.compareAndSet(expected, newVal); // 직접 CAS
counter.updateAndGet(x -> x * 2);       // 함수 적용
```

**고성능 변형 (Java 8+)**:
- `LongAdder`: 카운터에 최적화. 여러 셀에 분산해서 경쟁 줄임. 카운터 용도라면 `AtomicLong`보다 빠름
- `LongAccumulator`: 일반화된 누산기

```java
// 카운터로는 LongAdder가 훨씬 빠름 (경쟁 분산)
LongAdder adder = new LongAdder();
adder.increment();
long total = adder.sum();   // 정확한 합 (호출 시점)
```

### `volatile`

**언제 쓰나**: 단순한 플래그처럼 **쓰기 한 번/읽기 여러 번** 패턴.

```java
private volatile boolean closed = false;

// 한 스레드가 set
closed = true;

// 다른 스레드들이 즉시 봄
while (!closed) { ... }
```

**핵심 이해**:
- **가시성(visibility) 보장**: 한 스레드 변경을 다른 스레드가 즉시 봄
- **원자성 보장 아님**: `count++` 같은 건 안 됨 (Atomic 써야 함)
- **재정렬 방지**: 일부 컴파일러/CPU 재정렬을 막음

### `ReentrantLock` / `ReentrantReadWriteLock`

**언제 쓰나**: `synchronized`로는 부족한 정교한 락 제어.

```java
ReentrantLock lock = new ReentrantLock();

// 타임아웃 가능
if (lock.tryLock(100, MILLISECONDS)) {
    try { ... } finally { lock.unlock(); }
}

// Read/Write 분리 (읽기 많고 쓰기 적을 때)
ReadWriteLock rwLock = new ReentrantReadWriteLock();
rwLock.readLock().lock();   // 여러 스레드 동시 가능
rwLock.writeLock().lock();  // 배타적
```

**StampedLock (Java 8+)**: 더 빠른 read 최적화 (낙관적 읽기 지원).

### `CopyOnWriteArrayList`

**언제 쓰나**: **읽기 압도적으로 많고 쓰기 거의 없음** (예: 옵저버 리스트).

```java
List<Listener> listeners = new CopyOnWriteArrayList<>();

listeners.add(l);                      // 내부 배열 통째로 복사 (비쌈)
for (Listener l : listeners) { ... }   // 순회 중 변경 안전
```

**주의**: 쓰기가 잦으면 매번 전체 복사라 성능 박살. 읽기 99% 시나리오에만.

### 도구 선택 빠른 가이드

| 상황 | 추천 |
|---|---|
| 키-값 맵 | `ConcurrentHashMap` |
| 카운터 (높은 경쟁) | `LongAdder` |
| 카운터 (낮은 경쟁) | `AtomicLong` |
| 단순 플래그 | `volatile boolean` |
| FIFO 큐 (lock-free) | `ConcurrentLinkedQueue` |
| 생산자-소비자 (블로킹) | `LinkedBlockingQueue` |
| 양방향 큐 | `ConcurrentLinkedDeque` |
| 읽기 많고 쓰기 거의 없음 | `CopyOnWriteArrayList` |
| 정교한 락 제어 | `ReentrantLock` |
| 읽기/쓰기 분리 | `ReentrantReadWriteLock`, `StampedLock` |

---

## 4. 메모리 관리

인-메모리 저장소의 **가장 큰 적은 OOM(OutOfMemoryError)**이에요. 다음 4가지 정책을 반드시 고려하세요.

### 4.1 크기 제한 (Size Cap)

**최대 항목 수**를 고정하고 초과 시 오래된 것 제거.

```java
private static final int MAX_EVENTS = 10_000;

private void trim(Deque<Event> events) {
    while (events.size() > MAX_EVENTS) {
        events.pollFirst();
    }
}
```

**주의**: `ConcurrentLinkedDeque.size()`는 O(n)이라 매번 체크하면 비쌈. 자주 호출되는 곳이라면 별도 카운터를 두는 것도 고려.

### 4.2 시간 기반 보관 (TTL Retention)

**일정 시간 지난 항목 제거**.

```java
private static final Duration RETENTION = Duration.ofMinutes(60);

private void trimByTime(Deque<Event> events) {
    Instant cutoff = Instant.now().minus(RETENTION);
    while (true) {
        Event head = events.peekFirst();
        if (head == null || head.ts().isAfter(cutoff)) break;
        events.pollFirst();
    }
}
```

**팁**: 이벤트가 시간순이면 **머리부터만 검사**하면 됨 (앞이 가장 오래됨).

### 4.3 메모리 압박 시 비상 정리

JVM이 메모리 부족할 때 자동으로 줄이는 메커니즘.

#### `SoftReference` 활용
```java
Map<Key, SoftReference<Value>> cache = new ConcurrentHashMap<>();

// SoftReference는 GC가 메모리 부족 시 자동 회수
cache.put(key, new SoftReference<>(value));

Value v = cache.get(key).get();   // 회수됐으면 null
```

**주의**: 캐시에는 적합하지만 **신뢰성이 필요한 데이터엔 부적합**. 언제 사라질지 예측 불가.

#### `WeakReference`
참조가 끊기면 즉시 회수 가능. 약한 캐시용.

### 4.4 빈 컨테이너 정리

오래 안 쓴 채널/세션 자체를 제거하는 것도 잊지 마세요.

```java
@Scheduled(fixedDelay = 60_000)  // 1분마다
public void cleanupEmptyChannels() {
    Instant cutoff = Instant.now().minus(Duration.ofHours(1));
    logs.entrySet().removeIf(e -> {
        ChannelLog log = e.getValue();
        return log.events.isEmpty() && log.lastActivityBefore(cutoff);
    });
}
```

이걸 안 하면 **채널은 비어있어도 ChannelLog 객체는 영원히 남아** 메모리 누수가 됨.

### 4.5 모니터링 지표

운영 중 다음 지표를 노출하세요:
- 총 채널 수 (`logs.size()`)
- 총 이벤트 수 (각 채널 합)
- 가장 큰 채널의 이벤트 수
- 평균/최대 채널 나이
- trim으로 제거된 이벤트 수 (카운터)

Micrometer + Prometheus 조합이 무난.

---

## 5. API 설계

좋은 API는 **잘못 쓰기 어렵게** 만들어요.

### 5.1 인터페이스로 추상화

```java
public interface RealtimeStore {
    StoredEvent append(ChannelKey channel, String type, Object payload);
    List<StoredEvent> readSince(ChannelKey channel, long since, int limit);
    long latestSeq(ChannelKey channel);
    void close(ChannelKey channel);
}

@Component
class InMemoryRealtimeStore implements RealtimeStore { ... }

// 나중에 분산 환경 가면
@Component
class RedisRealtimeStore implements RealtimeStore { ... }
```

### 5.2 값 객체(Value Object)로 ID 표현

```java
// ❌ String을 그대로 쓰면 어떤 ID인지 헷갈림
void append(String channelId, String type, ...);

// ✅ 타입으로 분리: 컴파일러가 잘못된 사용 방지
public record ChannelKey(String value) {
    public ChannelKey {
        Objects.requireNonNull(value, "value");
        if (value.isBlank()) throw new IllegalArgumentException(...);
    }
}

void append(ChannelKey channel, String type, ...);
```

### 5.3 시퀀스 기반 페이징

폴링 패턴엔 **단조 증가하는 seq**를 쓰면 멱등하고 누락 없음.

```java
List<StoredEvent> readSince(ChannelKey channel, long since, int limit);
```

클라이언트는 마지막 seq를 기억하고 그 이후만 요청. 중복도 누락도 없음.

### 5.4 입력값 검증

생성자/메서드 진입 시 즉시 검증.

```java
public StoredEvent append(ChannelKey channel, String type, Object payload) {
    Objects.requireNonNull(channel, "channel");
    Objects.requireNonNull(type, "type");
    if (type.isBlank()) throw new IllegalArgumentException(...);
    // ...
}
```

### 5.5 에러 처리 정책 명시

```java
// 닫힌 채널에 쓰면? → 명확한 예외
if (log.closed) {
    throw new IllegalStateException("Channel is closed");
}

// 없는 채널 읽으면? → 빈 리스트 vs 예외 → 도메인에 맞게
if (log == null) return List.of();   // 이쪽이 보통 더 친절
```

### 5.6 불변 응답 반환

내부 자료구조를 노출하면 호출자가 망가뜨릴 수 있어요.

```java
// ❌ 내부 리스트 직접 반환
return log.events;   // 호출자가 .clear() 가능!

// ✅ 복사 또는 불변 뷰 반환
return List.copyOf(snapshot);
return Collections.unmodifiableList(new ArrayList<>(snapshot));
```

---

## 6. 흔한 함정과 회피법

### 함정 1: Check-Then-Act

```java
// ❌ 두 단계 사이에 race
if (!map.containsKey(key)) {
    map.put(key, value);
}

// ✅ 원자적 메서드
map.putIfAbsent(key, value);
map.computeIfAbsent(key, k -> compute());
```

### 함정 2: HashMap을 멀티스레드 환경에서 사용

```java
// ❌ HashMap은 thread-safe 아님
// → 데이터 깨짐, 무한 루프 가능 (Java 7에서 악명 높았음)
Map<K, V> map = new HashMap<>();

// ✅ ConcurrentHashMap 또는 외부 동기화
Map<K, V> map = new ConcurrentHashMap<>();
```

### 함정 3: `synchronized` 컬렉션의 순회

```java
List<T> list = Collections.synchronizedList(new ArrayList<>());

// ❌ 개별 메서드는 동기화되지만 순회는 외부에서 락 필요
for (T t : list) { ... }   // ConcurrentModificationException 가능

// ✅ 외부 동기화
synchronized (list) {
    for (T t : list) { ... }
}

// ✅✅ 또는 처음부터 동시 컬렉션
List<T> list = new CopyOnWriteArrayList<>();
```

### 함정 4: `volatile` 오해

```java
// ❌ volatile은 ++ 같은 복합 연산을 원자화하지 못함
private volatile int counter;
counter++;   // 여전히 race condition!

// ✅ 복합 연산엔 Atomic
private final AtomicInteger counter = new AtomicInteger();
counter.incrementAndGet();
```

### 함정 5: 락 안에서 외부 호출

```java
// ❌ 락 잡은 채로 시간 오래 걸리는 외부 호출
synchronized (this) {
    Result r = httpClient.send(...);   // 다른 스레드 다 막힘!
    cache.put(key, r);
}

// ✅ 외부 호출은 락 밖에서
Result r = httpClient.send(...);
synchronized (this) {
    cache.put(key, r);
}
```

### 함정 6: 락 순서 불일치 → 데드락

```java
// 스레드 A: lockA → lockB
// 스레드 B: lockB → lockA
// → 데드락 가능

// ✅ 항상 같은 순서로 락 획득
// 락 객체에 순서를 강제하는 헬퍼를 만들거나, 단일 락으로 단순화
```

### 함정 7: Compound Action 안 한 묶음

```java
// ❌ 두 작업을 별도로
if (events.size() > MAX) {
    events.pollFirst();   // 사이에 다른 스레드가 size를 바꿈
}

// ✅ 하나의 트랜잭션처럼 묶거나, 어차피 polling은 atomic이니 결과 받기
while (events.size() > MAX) {
    if (events.pollFirst() == null) break;   // 이미 비었으면 끝
}
```

### 함정 8: ABA 문제

```java
// CAS만으론 ABA 못 막음
// 참조나 포인터 다룰 땐 AtomicStampedReference 사용
```

### 함정 9: 메모리 가시성 무시

```java
// ❌ 이 boolean은 다른 스레드가 영원히 못 볼 수 있음
private boolean stopped = false;

// ✅ volatile 추가
private volatile boolean stopped = false;
```

### 함정 10: 무한 재시도

```java
// ❌ CAS 실패 시 무한 재시도 → CPU 폭주
while (!atomic.compareAndSet(prev, next)) {
    prev = atomic.get();
    next = ...;
}

// ✅ 백오프, 최대 시도 횟수, 또는 Atomic의 update 메서드
atomic.updateAndGet(x -> compute(x));   // JDK가 알아서 처리
```

---

## 7. 테스트와 검증

동시성 코드는 **일반 단위 테스트로 부족**해요.

### 7.1 스트레스 테스트

여러 스레드에서 동시에 호출.

```java
@Test
void concurrent_append_no_lost_events() throws Exception {
    int threads = 10;
    int perThread = 1000;
    ExecutorService pool = Executors.newFixedThreadPool(threads);
    CountDownLatch latch = new CountDownLatch(threads);

    for (int i = 0; i < threads; i++) {
        pool.submit(() -> {
            try {
                for (int j = 0; j < perThread; j++) {
                    store.append(channel, "type", j);
                }
            } finally {
                latch.countDown();
            }
        });
    }
    latch.await();

    // 모든 이벤트가 고유 seq를 가져야 함
    assertEquals(threads * perThread, store.latestSeq(channel));
}
```

### 7.2 jcstress (JCStress)

OpenJDK에서 만든 **동시성 테스트 프레임워크**. 가능한 인터리빙을 자동 탐색.

```java
@JCStressTest
@Outcome(id = "1, 1", expect = ACCEPTABLE, desc = "Both see update")
@State
public class MyTest {
    int x;
    @Actor public void writer() { x = 1; }
    @Actor public void reader(II_Result r) { r.r1 = x; r.r2 = x; }
}
```

### 7.3 정적 분석 도구

- **SpotBugs** + FindBugs concurrency plugin: 흔한 동시성 패턴 검출
- **ErrorProne**: `@GuardedBy`, `@ThreadSafe` 어노테이션 검증

### 7.4 어노테이션으로 의도 표현

```java
@ThreadSafe
public class InMemoryStore { ... }

@Immutable
public record StoredEvent(...) {}

@GuardedBy("this")
private List<Event> events;
```

`javax.annotation.concurrent` 또는 `net.jcip.annotations` 패키지. 도구가 검증해주고 사람도 읽기 쉬움.

### 7.5 TLA+ / 모델 체커

매우 중요한 시스템이라면 **수학적 검증**도 고려. AWS, Microsoft 같은 곳에서 쓰는 방법.

---

## 8. 체크리스트

설계 시 이 질문들에 답할 수 있어야 해요:

### 동시성
- [ ] 어떤 객체가 여러 스레드에서 접근되나?
- [ ] 그 객체는 thread-safe한가? (`@ThreadSafe`로 명시)
- [ ] 가변 상태는 어떻게 보호되나? (락? 동시 자료구조? Atomic?)
- [ ] Check-then-act 패턴이 있는가? 있다면 원자적인가?
- [ ] 락 순서가 일관되는가? (데드락 방지)
- [ ] 외부 호출이 락 안에서 일어나지 않는가?

### 메모리
- [ ] 데이터가 무한정 쌓일 수 있는가?
- [ ] 크기 제한이 있는가?
- [ ] 시간 기반 만료가 있는가?
- [ ] 빈 컨테이너 자체도 정리되는가?
- [ ] 메모리 사용량을 모니터링할 지표가 있는가?

### API
- [ ] 인터페이스로 추상화되어 있는가?
- [ ] ID는 값 객체로 타입 안전한가?
- [ ] 입력값이 검증되는가?
- [ ] 반환값은 불변/방어적 복사인가?
- [ ] 에러 시나리오가 명확히 처리되는가? (없는 키, 닫힌 리소스 등)

### 운영
- [ ] 메트릭이 노출되는가? (크기, 처리량, 오류율)
- [ ] 로그가 적절한가? (너무 많지도, 적지도 않게)
- [ ] graceful shutdown이 가능한가?
- [ ] 재시작 시 데이터 처리 정책이 있는가?

### 테스트
- [ ] 단일 스레드 동작 테스트 ✓
- [ ] 다중 스레드 스트레스 테스트 ✓
- [ ] 메모리 정책 테스트 (한도 초과, TTL) ✓
- [ ] 엣지 케이스 (빈 컨테이너, 닫힌 리소스) ✓

---

## 9. 참고할 패턴 모음

### 패턴 A: 채널별 격리

```java
ConcurrentMap<Key, ChannelLog> logs = new ConcurrentHashMap<>();

// 각 채널은 독립적으로 동작
ChannelLog log = logs.computeIfAbsent(key, k -> new ChannelLog());
```

### 패턴 B: 시퀀스 기반 폴링

```java
class ChannelLog {
    final AtomicLong nextSeq = new AtomicLong(1);
    final Deque<Event> events = new ConcurrentLinkedDeque<>();
}

long seq = log.nextSeq.getAndIncrement();   // 원자적 시퀀스 부여
```

### 패턴 C: Volatile 플래그로 상태 전이

```java
class Resource {
    volatile boolean closed = false;

    public void use() {
        if (closed) throw new IllegalStateException();
        // ...
    }

    public void close() {
        closed = true;
    }
}
```

### 패턴 D: Builder + Immutable

```java
public final class Event {
    private final long seq;
    private final Instant ts;
    // 모든 필드 final, 생성자에서만 할당
    // 한번 만들면 안전하게 공유 가능
}
```

### 패턴 E: 정기적 백그라운드 정리

```java
@Scheduled(fixedDelay = 60_000)
public void sweep() {
    Instant cutoff = Instant.now().minus(RETENTION);
    logs.entrySet().removeIf(e -> e.getValue().isStale(cutoff));
}
```

---

## 10. 마무리: 좋은 동시성 코드의 특징

좋은 동시성 코드는 다음을 만족해요:

1. **읽기 쉽다**: 누가 봐도 어떤 락이 무엇을 보호하는지 명확
2. **잘못 쓰기 어렵다**: API 자체가 사용자를 함정에서 막아줌
3. **측정된 성능**: 추측이 아닌 벤치마크 기반의 선택
4. **점진적으로 확장 가능**: 인-메모리 → Redis → Kafka로 갈 수 있는 인터페이스
5. **운영자 친화적**: 메트릭, 로그, 헬스 체크가 잘 갖춰짐

### 추천 도서

- **Java Concurrency in Practice** - Brian Goetz (필독서)
- **The Art of Multiprocessor Programming** - Maurice Herlihy
- **Designing Data-Intensive Applications** - Martin Kleppmann

### 핵심 한 줄

> **"동시성은 추가하는 게 아니라 처음부터 설계한다. 그리고 가능하면 직접 만들지 말고, JDK가 검증해놓은 도구를 쓴다."**
