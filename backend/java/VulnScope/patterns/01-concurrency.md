# A. 동시성 (Concurrency)

> 이 문서가 다루는 것: VulnScope 에서 멀티 스레드 환경의 race condition 을 어떻게 회피했는지. 12개 패턴.
> 전제 지식: Java 기본 + 스레드 개념 (Thread.start). lock/atomic 처음이면 §0 부터.

---

## §0. 동시성의 기본 — 왜 어려운가?

### 0.1 race condition 이란?

두 스레드가 같은 데이터를 동시에 읽고 쓸 때, **연산이 인터리빙(interleave) 되면 결과가 망가지는 현상**.

```java
// counter 가 0 인 상태에서 두 스레드가 동시에 counter++ 호출
Thread A: int tmp = counter;  // tmp = 0
Thread B: int tmp = counter;  // tmp = 0  ← 같은 값 읽음
Thread A: counter = tmp + 1;  // counter = 1
Thread B: counter = tmp + 1;  // counter = 1  ← 1번만 증가됨!
```

**기대**: counter = 2. **실제**: counter = 1. 한 increment 가 사라짐 → **lost update**.

`counter++` 는 사실 **3개 명령** (read → add → write) 의 묶음. 중간에 다른 스레드가 끼어들면 망가짐. 이런 묶음을 **non-atomic operation** 이라 부름.

### 0.2 atomic 이란?

**더 작은 단위로 쪼개지지 않는 연산**. 중간에 다른 스레드가 끼어들 수 없음.

`AtomicLong.incrementAndGet()` 은 read+add+write 를 **CPU 레벨 원자 명령** (CAS) 으로 처리 → atomic.

### 0.3 CAS (Compare-And-Swap)

CPU 의 atomic 명령. "현재 값이 X 이면 Y 로 바꿔라" 를 한 번에 수행.

```
CAS(memory, expected, new):
    if memory == expected:
        memory = new
        return true
    else:
        return false
```

x86 의 `LOCK CMPXCHG` 명령. 하드웨어 보장이라 lock 없이도 race-free.

`AtomicLong.incrementAndGet()` 은 내부적으로:
```
loop:
    long current = get();
    long next = current + 1;
    if (CAS(value, current, next)) return next;
    goto loop;        // 다른 스레드가 먼저 바꿨으면 재시도
```

이게 **lock-free** 의 본질 — 락 없이도 원자성 보장.

### 0.4 lock 의 비용

`synchronized` 또는 `ReentrantLock` 은 atomic 보다 무거움:
- 다른 스레드 wait → context switch → OS 호출.
- 락 경쟁 시 CPU cache miss.
- atomic 은 보통 lock 없이 동작 (uncontended path).

→ 가능하면 atomic, 어려우면 lock.

### 0.5 메모리 가시성 (visibility) — happens-before

CPU 마다 캐시가 있어, 스레드 A 가 변수 쓴 값이 스레드 B 의 캐시에 즉시 안 보임. **stale read** 발생.

```java
// Thread A
data = computeExpensive();
ready = true;          // ← 이 write 가 다른 스레드에 보일까?

// Thread B
if (ready) {           // ← 위에서 ready=true 봤어도
    use(data);          // data 가 아직 stale 일 수 있음 (캐시)
}
```

**해결**: `volatile`, `synchronized`, atomic 클래스 등이 **happens-before** 보장. 한 스레드의 write 가 다른 스레드의 read 에 보이는 순서 약속.

이게 Java Memory Model (JMM, JSR-133) 의 핵심.

---

이제 VulnScope 의 12개 동시성 패턴.

---

## A1. `ConcurrentHashMap` — 멀티 스레드 안전 Map

### 무엇인가
JDK 표준 Map. 여러 스레드가 동시에 put/get 해도 race 없음. **lock-free read** + **fine-grained lock write** (segment 단위).

### 어디서 쓰나
모든 `InMemory*Repository` (Target/Scan/Finding/Upload/Evidence/Profile) + `InMemoryRealtimeStore`/`Bus`/`ControlBus`.

```java
// InMemoryScanRepository.java
private final ConcurrentMap<UUID, Scan> byId = new ConcurrentHashMap<>();

public Scan save(Scan scan) {
    byId.put(scan.id().value(), scan);   // 스레드 A
    return scan;
}

public Optional<Scan> findById(ScanId id) {
    return Optional.ofNullable(byId.get(id.value()));   // 스레드 B 동시
}
```

**race 없음**: HTTP request 처리 스레드 (controller) + worker 가상 스레드가 동시 접근해도 안전.

### 왜 필요한가
일반 `HashMap` 은 멀티 스레드 시 위험:
- get/put 동시 → 무한 loop (Java 7 이전 가끔)
- write 중 read → ArrayIndexOutOfBoundsException
- 대부분 silent corruption (데이터가 사라짐)

`Collections.synchronizedMap(new HashMap<>())` 은 모든 op 에 객체 lock — read 도 lock 대기.

`ConcurrentHashMap` 은 다름:
- read: lock 없음 (volatile 만 사용).
- write: bucket 단위 lock (Java 7) 또는 CAS + 일부 lock (Java 8+).
- 결과: read 는 거의 무료, write 는 충돌 minimal.

### 핵심 아이디어
**"읽기 빈도 >> 쓰기 빈도" 가정 + lock 분할**. 전체 Map 에 lock 1개 거는 게 아니라, "이 키 영역" 만 lock.

Java 8 부터는 **CAS 기반 + 충돌 시만 synchronized** — 평균 케이스에 lock 거의 없음.

### 전문 용어 사전
- **lock-free**: 락 없이 atomic 명령 (CAS) 만으로 동시성 보장.
- **fine-grained lock**: 큰 객체 전체에 lock 거는 게 아니라, 작은 단위 (bucket, segment) 에 lock.
- **uncontended**: 같은 자원 두 스레드가 동시에 안 만지는 상태. atomic/CAS 가 가장 빠른 path.

### 함정
- `compute*` 류 메서드 (computeIfAbsent, merge) 는 lock 거는 case 있음. callback 안에서 같은 map 의 다른 키 access 시 deadlock 위험.
- `keySet`, `values`, `entrySet` 의 iterator 는 **weakly consistent** — iteration 중 다른 스레드의 변경 일부만 보일 수 있음. 강한 snapshot 원하면 `new ArrayList<>(map.values())`.

### 대안
| | ConcurrentHashMap | synchronizedMap | HashMap |
|---|---|---|---|
| read 동시성 | ✅ lock 없음 | ❌ lock | ⚠️ unsafe |
| write 동시성 | ✅ fine-grained | ❌ 전체 lock | ❌ corruption |
| iterator | weakly consistent | fail-fast (수정 시 예외) | fail-fast |
| 메모리 | 약간 큼 | 작음 | 작음 |

### 출처/아이디어
Doug Lea, Brian Goetz 등이 작성한 JSR-166 (`java.util.concurrent`). Java 5 도입. Brian Goetz 의 책 "Java Concurrency in Practice" (2006) 가 표준 참조.

---

## A2. `ConcurrentMap.computeIfAbsent` + 새 인스턴스 — atomic "처음만 생성"

### 무엇인가
"키가 없으면 람다 실행해서 만든 값을 put, 있으면 기존 값 반환" 을 **하나의 atomic 연산** 으로.

### 어디서 쓰나

#### 1) per-scan 시퀀스 발급 (가장 중요)
```java
// InMemoryFindingRepository.nextSeqForScan
@Override
public long nextSeqForScan(ScanId scanId) {
    return seqByScan
        .computeIfAbsent(scanId.value(), k -> new AtomicLong(0))   // ← (1)
        .incrementAndGet();                                         // ← (2)
}
```

(1): scanId 처음 들어오면 `AtomicLong(0)` 새로 만들고 put. 이미 있으면 기존 그대로.
(2): 그 AtomicLong 에 증가 (atomic).

→ **두 단계 모두 thread-safe**.

#### 2) per-channel 로그 생성
```java
// InMemoryRealtimeStore.append
ChannelLog log = logs.computeIfAbsent(channel, key -> new ChannelLog());
```

### 왜 필요한가
naive (잘못된) 패턴:
```java
// ❌ race condition
if (!map.containsKey(k)) {
    map.put(k, new AtomicLong(0));
}
return map.get(k).incrementAndGet();
```

두 스레드가 동시에 같은 scanId 처음 호출 시:
1. 스레드 A: containsKey → false
2. 스레드 B: containsKey → false  ← 둘 다 false 봤음
3. 스레드 A: put(new AtomicLong(0))
4. 스레드 B: put(new AtomicLong(0))  ← 덮어씀! 이전 increment 사라짐
5. ...

→ 시퀀스 1, 2, 1, 2... 같은 중복 발생.

`computeIfAbsent` 는 **bucket 레벨 lock 으로 람다 1회만 실행 보장**. 두 스레드 동시 호출 시 한 명만 람다 실행, 나머지는 그 결과 받음.

### 핵심 아이디어
**"check-then-act" 가 atomic 으로 합쳐져야 안전**. 별도 호출은 race. 표준 라이브러리가 합친 메서드 제공.

### 전문 용어 사전
- **check-then-act**: "조건 검사 → 액션" 패턴. 두 단계 사이에 다른 스레드가 끼어들면 race. atomic 합치기 필수.
- **bucket-level lock**: ConcurrentHashMap 이 충돌 시 해당 키의 bucket 만 lock. 다른 키는 영향 없음.

### 함정
- 람다 안에서 같은 map 의 다른 키 접근 → deadlock 위험. 람다는 단순 인스턴스 생성만.
- 람다 안에서 예외 던지면 put 안 됨 + 호출자에 전파.

### 대안
- `putIfAbsent(k, v)` — 기존 값 있으면 기존 반환. 단점: 매번 새 인스턴스 생성 후 버려짐 (낭비).
- `merge(k, v, BiFunction)` — 기존 값 있으면 합치기. 다른 용도 (counter 합산 등).
- `computeIfAbsent` 가 **인스턴스 생성을 1회로 제한** 하므로 가장 효율적.

### 출처
Java 8 Stream/Lambda 도입 시 ConcurrentMap 에 추가된 메서드들. JEP 155.

---

## A3. `AtomicLong.incrementAndGet` — race-free 시퀀스

### 무엇인가
`long` 변수에 atomic 증가/감소/CAS 제공하는 wrapper.

### 어디서 쓰나
```java
// InMemoryRealtimeStore.ChannelLog
final AtomicLong nextSeq = new AtomicLong(1);

@Override
public StoredEvent append(...) {
    long seq = nextSeq.getAndIncrement();   // 1, 2, 3, ... atomic
    return new StoredEvent(channel, seq, ...);
}
```

### 왜 필요한가
`long counter; counter++;` 는 **3 명령** (read, add, write). race 가능 (§0.1 참조).

`synchronized` 로 감싸도 되지만:
```java
synchronized (this) {
    counter++;
}
```
→ lock 비용. 단순 증가에 과함.

`AtomicLong.incrementAndGet()` 은 CAS loop:
```
do {
    long current = get();
    long next = current + 1;
} while (!compareAndSet(current, next));
return next;
```
- lock 없음 (uncontended 시 매우 빠름).
- 충돌 시 재시도 (CAS 가 false 면 loop).
- CPU 의 LOCK CMPXCHG 명령으로 hardware 보장.

### 핵심 아이디어
**Compare-And-Swap (CAS) 로 lock 없이 atomic 보장**. lock-free 알고리즘의 가장 단순한 예.

### 전문 용어 사전
- **CAS (Compare-And-Swap)**: "현재 값이 X 이면 Y 로 바꿔라" 를 atomic 으로. CPU hardware 명령.
- **lock-free**: 락 없이 동시성 보장. 한 스레드가 죽어도 다른 스레드가 진행 가능.
- **wait-free**: lock-free 보다 강함. 모든 스레드가 유한 단계에 완료 보장. Atomic 은 보통 lock-free (CAS 재시도) 까지.

### 함정
- 여러 변수 묶어서 atomic 보장 X. 그러려면 `AtomicReference<MyRecord>` + CAS 또는 lock.
- `i++` 의 atomic 만 보장. `if (counter > 100) counter++;` 같은 conditional 은 별도 lock 필요.

### 대안
- `LongAdder` (Java 8+) — 매우 high contention 카운터에 더 빠름. 내부적으로 cell 분산. 단점: get() 이 O(cells).
- `synchronized` — 단순한 단일 정수 증가에는 atomic 이 더 빠름.

### 출처
Java 5 `java.util.concurrent.atomic` 패키지. AtomicInteger/Long/Reference 등.

---

## A4. `ConcurrentLinkedDeque` — lock-free FIFO + tail trim

### 무엇인가
양방향 큐 (deque). lock-free. tail append + head pollFirst 가 안전.

### 어디서 쓰나
```java
// InMemoryRealtimeStore.ChannelLog
final Deque<StoredEvent> events = new ConcurrentLinkedDeque<>();

// append (worker 가 emit)
events.add(event);                  // tail

// trim (retention 정리)
while (log.events.size() > MAX) {
    log.events.pollFirst();         // head
}

// readSince (SSE 클라이언트 backfill)
for (StoredEvent event : log.events) {  // weakly-consistent iterator
    if (event.seq() > since) ...
}
```

### 왜 필요한가
SSE 의 worker emit 은 **very high throughput**. `LinkedList + synchronized` 면 모든 op 에 lock 비용.

`ConcurrentLinkedDeque` 는:
- add (tail append): CAS 기반 lock-free.
- pollFirst (head remove): CAS 기반 lock-free.
- iterator: weakly consistent — 시작 시점의 snapshot 비슷한 view, 중간 변경 일부 반영 가능.

### 핵심 아이디어
**Michael-Scott non-blocking queue** 알고리즘. tail/head 포인터 CAS 로 갱신.

### 전문 용어 사전
- **Michael-Scott queue**: 1996년 Michael & Scott 의 lock-free queue 알고리즘. 모든 lock-free 큐의 표준 참조.
- **weakly consistent iterator**: iteration 중 다른 스레드의 변경이 일부만 보일 수 있음. ConcurrentModificationException 안 던짐 (fail-fast 와 반대).

### 함정
- `size()` 가 O(N) — 전체 노드 순회. 빈번 호출 비용 큼. VulnScope 는 trim 시만 호출.
- 정확한 snapshot 원하면 `new ArrayList<>(deque)` 복사.

### 대안
- `LinkedBlockingDeque` — capacity 지정 가능 + blocking offer/take. lock 기반.
- `ArrayBlockingQueue` — 고정 size + circular buffer. 단방향만.
- VulnScope 는 retention 으로 size 관리 → ConcurrentLinkedDeque 가 적합.

---

## A5. `ConcurrentHashMap.newKeySet()` — thread-safe Set view

### 무엇인가
ConcurrentHashMap 의 keySet 을 thread-safe Set 으로 노출. 별도 Set 구현 없이 표준 헬퍼.

### 어디서 쓰나
```java
// InMemoryRealtimeBus.subscribers
private final ConcurrentMap<ChannelKey, Set<Consumer<StoredEvent>>> subscribers
    = new ConcurrentHashMap<>();

@Override
public Subscription subscribe(ChannelKey channel, Consumer<StoredEvent> subscriber) {
    subscribers.computeIfAbsent(channel, k -> ConcurrentHashMap.newKeySet())
        .add(subscriber);
    return () -> subscribers.get(channel).remove(subscriber);
}
```

→ subscribers Map 의 value 가 thread-safe Set. add/remove 안전.

### 왜 필요한가
`Collections.synchronizedSet(new HashSet<>())` 도 가능하지만:
- 모든 op (read 포함) 에 lock.
- iterator 사용 시 외부 synchronized 블록 필요 (안 그러면 ConcurrentModificationException).

`ConcurrentHashMap.newKeySet()` 은 ConcurrentHashMap 의 모든 장점 (lock-free read + fine-grained write) 그대로.

### 핵심 아이디어
**Set 도 결국 "값이 boolean 인 Map"** — ConcurrentHashMap 을 그대로 활용.

### 함정
- iterator 가 weakly consistent — publish loop 중 unsubscribe 해도 즉시 반영 X. VulnScope 는 OK (다음 publish 사이클에 반영).

### 대안
- `CopyOnWriteArraySet` — read 가 매우 많고 write 거의 없을 때. write 마다 전체 복사 (수십 명 subscribe 환경).
- `Collections.synchronizedSet` — 모든 op lock.

---

## A6. `volatile boolean` — cross-thread 가시성

### 무엇인가
`volatile` 키워드. 변수 read/write 가 **happens-before** 보장. 캐시 안 거치고 main memory 접근.

### 어디서 쓰나
```java
// InMemoryRealtimeStore.ChannelLog
private static final class ChannelLog {
    final AtomicLong nextSeq = new AtomicLong(1);
    final Deque<StoredEvent> events = new ConcurrentLinkedDeque<>();
    volatile boolean closed = false;        // ← 여기
}

// thread A (scan terminal 시 close)
log.closed = true;

// thread B (append)
@Override
public StoredEvent append(...) {
    if (log.closed) {                       // ← 즉시 보임
        throw new IllegalStateException("Channel ... is closed");
    }
    ...
}
```

### 왜 필요한가
일반 `boolean closed = false;` 면:
- thread A 의 `closed = true` 가 thread B 의 캐시에 늦게 도달 (또는 영영 안 도달).
- thread B 가 stale `false` 를 계속 봄 → close 후에도 append 성공 → 채널 무결성 깨짐.

`volatile` 은:
- write: 모든 CPU 캐시 invalidate + main memory 즉시 반영.
- read: 캐시 거치지 않고 main memory 직접.
- happens-before: write 이전 모든 작업이 read 이후 모든 작업에 보임.

### 핵심 아이디어
**Java Memory Model (JMM) 의 volatile 보장**. CPU/JVM 의 reordering 과 cache 를 wider barrier 로 강제.

### 전문 용어 사전
- **happens-before**: A → B 순서가 보장된다는 약속. JMM 의 핵심 개념.
- **memory barrier (fence)**: CPU 명령 reordering 차단 + 캐시 flush. volatile read/write 가 barrier 삽입.
- **stale read**: 다른 스레드가 이미 write 한 값을 못 보고 옛 값 read.
- **JMM (Java Memory Model)**: JSR-133 (Java 5+). multi-thread 메모리 의미 표준.

### 함정
- volatile 은 **단일 read/write 만 atomic 보장**. `volatile counter; counter++;` 는 여전히 race (3 명령). atomic class 또는 lock 필요.
- 객체 reference 는 volatile 가능 — 객체 자체 변경은 별도 동기화 필요.

### 대안
- `AtomicBoolean` — get/set + CAS 가능. boolean flag 만 필요하면 volatile 충분.
- `synchronized` 블록 — happens-before 보장 + atomic 묶음. 무거움.

### 출처
JSR-133 Java Memory Model. Brian Goetz 의 "Java Concurrency in Practice" Chapter 3.

---

## A7. `synchronized` 블록 — thread-unsafe API 보호

### 무엇인가
객체 monitor 를 lock 으로 사용. 같은 객체 lock 잡은 스레드 1개만 블록 진입.

### 어디서 쓰나
```java
// TokenService.signedHmac
private final Mac mac;     // javax.crypto.Mac — thread-unsafe!

private byte[] signedHmac(byte[] input) {
    synchronized (mac) {                  // mac 객체 lock
        return mac.doFinal(input);
    }
}
```

### 왜 필요한가
`Mac.doFinal` 은 **thread-unsafe** (JCA 명세). 두 스레드 동시 호출 시 internal state 깨짐 → 잘못된 HMAC.

옵션:
1. **synchronized** (현재) — 단순. lock 비용 있지만 token verify 빈도 그리 안 높음.
2. **ThreadLocal Mac** — 각 스레드 자기 인스턴스. lock 제거. 메모리 비용.
3. **매 호출 새 Mac.getInstance** — init 비용 큼 (key 재설정).

VulnScope 는 단순함 우선 → synchronized.

### 핵심 아이디어
**외부 라이브러리의 thread-safety 명세 확인 후 최소 비용 보호**. unsafe 면 lock, safe 면 그대로.

### 전문 용어 사전
- **monitor**: Java 객체마다 가진 implicit lock. synchronized(obj) 가 그 lock 잡음.
- **mutual exclusion (mutex)**: 같은 자원에 한 스레드만 접근 보장. lock 의 본질.
- **reentrant**: 같은 스레드가 같은 lock 다시 잡아도 OK. Java synchronized 는 reentrant.

### 함정
- lock 안에서 외부 메서드 호출 (특히 다른 lock 잡는 메서드) → **deadlock** 위험.
- lock 안에서 오래 걸리는 작업 (I/O 등) → 다른 스레드 모두 block.
- `synchronized (this)` 또는 `synchronized (someField)` — lock 객체 명시.

### 대안
- `ReentrantLock` — try-lock, timeout, fair 옵션 등 풍부. synchronized 보다 무거운 API.
- `ReadWriteLock` — read 동시 허용 + write 단독. read 빈도 높을 때.
- `StampedLock` — optimistic read 가능. read 99% case 빠름.

---

## A8. `ThreadLocal<T>` — per-request 컨텍스트

### 무엇인가
스레드마다 별도의 변수 저장소. 같은 코드가 다른 스레드에서 다른 값 read.

### 어디서 쓰나
```java
// shared/security/TenantContext.java
public final class TenantContext {

    private static final ThreadLocal<UUID> CURRENT_ORG = new ThreadLocal<>();

    public static void set(UUID orgId) { CURRENT_ORG.set(orgId); }

    public static UUID currentOrgId() {
        UUID orgId = CURRENT_ORG.get();
        if (orgId == null) throw new IllegalStateException("No tenant set on current thread");
        return orgId;
    }

    public static void clear() { CURRENT_ORG.remove(); }
}
```

```java
// AuthFilter — set + clear
TenantContext.set(payload.orgId());
try { chain.doFilter(req, res); }
finally { TenantContext.clear(); }

// Controller — read
OrgId orgId = OrgId.of(TenantContext.currentOrgId());
```

### 왜 필요한가
multi-tenant SaaS 에서 모든 service/controller 가 "현재 사용자 org 가 누구인지" 알아야 함.

옵션:
1. **모든 메서드 시그니처에 OrgId 추가** — boilerplate 폭증. `service.list(orgId, page, size)` 매번.
2. **ThreadLocal 컨텍스트** (현재) — 한 번 set, 어디서나 read. 시그니처 깔끔.

### 핵심 아이디어
**cross-cutting concern (横断 관심사) 을 implicit context 로** — 모든 layer 에 명시 전달 안 해도 됨.

Spring Security 의 `SecurityContextHolder`, MDC (logging), Spring Transaction 등이 모두 ThreadLocal.

### 전문 용어 사전
- **cross-cutting concern**: 여러 layer 에 걸쳐 필요한 관심사 (auth, logging, transaction). AOP 의 대상.
- **implicit context**: 명시적 인자 없이 "지금 컨텍스트" 로 접근.

### 함정
- **leak 위험** — 다음 패턴 (A9) 참조.
- 가상 스레드 환경 OK — 각 가상 스레드가 자기 ThreadLocal 가짐.
- `ThreadLocal.get()` 빈번 호출 시 약간 비용 (HashMap lookup 비슷).

### 대안
- 인자 명시 전달 — 시그니처 폭증.
- Spring `RequestScope` 빈 — Spring 의존, 비슷한 의미.
- ScopedValue (Java 21+ preview) — ThreadLocal 의 leak 안전 후속. 가상 스레드 친화. 안정 시 마이그레이션 권장.

---

## A9. ThreadLocal `clear()` in `finally` — leak 방지

### 무엇인가
ThreadLocal 사용 후 명시적으로 비우기. servlet thread pool 재사용 시 다른 사용자에 leak 안 되게.

### 어디서 쓰나
```java
// AuthFilter.doFilterInternal
TenantContext.set(payload.orgId());
try {
    chain.doFilter(req, res);
} finally {
    TenantContext.clear();          // ← 핵심
}
```

### 왜 필요한가
**servlet thread pool 의 스레드는 재사용** — request 끝나도 죽지 않고 다음 request 받음.

```
Request 1 (org=Alice) — Thread #5 처리:
    set(Alice)
    ... controller ...
    [응답 전송]
    [clear() 안 함]  ← BAD

Request 2 (org=Bob) — Thread #5 재사용:
    [AuthFilter 가 새로 set(Bob) 호출 전이라면]
    Controller 가 currentOrgId() → Alice 봄!  ← 보안 사고
```

→ **Alice 의 데이터를 Bob 이 봄**. cross-tenant 정보 노출.

`finally { clear(); }` 는 예외 발생해도 무조건 실행 보장.

### 핵심 아이디어
**ThreadLocal 사용 시 try-finally 는 의무**. 한 줄 빠뜨리면 보안 사고.

### 전문 용어 사전
- **thread leak (memory)**: ThreadLocal entry 가 GC 안 되어 메모리 누적. `WeakReference` 로 키는 GC 되지만 값은 leak.
- **information leak**: 위 시나리오. cross-thread/tenant 정보 노출.

### 함정
- async 작업으로 ThreadLocal 전파 시 별도 처리 필요. CompletableFuture, ExecutorService 등은 ThreadLocal 자동 전파 X.
- ScopedValue (Java 21+ preview) 는 자동 cleanup → 이 함정 회피.

### 대안 없음
ThreadLocal 사용 시 finally clear 가 표준. 안 하면 사고.

---

## A10. `@Async` + `@EnableAsync` + 가상 스레드

### 무엇인가
- `@Async`: 메서드를 별도 스레드에서 실행하라고 Spring 에 표시.
- `@EnableAsync`: 어플리케이션 레벨 활성화.
- 가상 스레드 (`spring.threads.virtual.enabled=true`): blocking I/O 가 OS 스레드 점유 안 함.

### 어디서 쓰나
```java
// VulnscopeApplication
@SpringBootApplication
@EnableAsync                        // ← 활성
public class VulnscopeApplication { ... }
```

```java
// scanengine/application/service/WorkerExecutor.java
@EventListener
@Async                              // ← async 실행
public void on(ScanRequested event) {
    execute(event);
}

public void execute(ScanRequested event) {
    // ... HTTP fetch (blocking)
    // ... DB save
    // ... 시간 걸리는 작업
}
```

### 왜 필요한가

**문제**: `POST /scans` (trigger) 가 응답까지 worker 끝까지 기다리면 사용자 응답 30초+ 걸림.

**해결**:
1. trigger → DB save → `publishEvent(ScanRequested)` → 즉시 응답 (1초 미만).
2. 별도 스레드에서 worker 가 실제 fetch + 점검.

`@Async` 없으면 `@EventListener` 가 publish 호출 스레드에서 동기 실행 → trigger 가 worker 끝까지 대기.

### 가상 스레드 (Project Loom)

전통적 OS 스레드 (platform thread) 는:
- 비싸 (각각 1MB stack 등).
- blocking I/O (HTTP fetch 15초) 동안 OS 스레드 점유 → 다른 작업 못 함.
- 동시 worker 100개 = OS 스레드 100개 (heavy).

**가상 스레드** (Java 21+) 는:
- 매우 가벼움 (KB 단위).
- blocking I/O 시 OS 스레드 (carrier) 양보 → 다른 가상 스레드가 carrier 사용.
- 동시 worker 1만 개도 OK.

`spring.threads.virtual.enabled=true` 면:
- Tomcat 요청 처리 → 가상 스레드.
- `@Async` → 가상 스레드.
- `@Scheduled` → 가상 스레드.

### 핵심 아이디어
**비동기 실행 + 가상 스레드 = blocking 코드를 그대로 두면서 효율적 동시성**. reactive (WebFlux) 학습 비용 회피.

### 전문 용어 사전
- **Project Loom**: Java 의 가상 스레드 도입 프로젝트. JEP 444 (Java 21).
- **carrier thread**: 가상 스레드를 실제로 실행하는 OS 스레드. 가상 스레드 blocking 시 carrier 양보.
- **continuation**: 가상 스레드의 실행 상태 (stack frames). park 시 저장, resume 시 복원.
- **pinning**: 가상 스레드가 carrier 양보 못 하는 상태. synchronized 블록 안에서 발생 (Java 21 한정 — 24+ 에서 해소 진행).

### 함정
- `@Async` 메서드를 같은 클래스 안에서 자가 호출 시 proxy 우회 → 동기 실행. `on()` 과 `execute()` 분리 필수.
- `synchronized` 안에서 blocking I/O → 가상 스레드 pinning. JDK 24+ 또는 ReentrantLock 사용.
- ThreadLocal 은 가상 스레드도 지원. 단 ScopedValue 가 가상 스레드 친화.

### 대안
- WebFlux Reactive — 진짜 non-blocking. 학습 곡선 큼.
- ExecutorService 직접 — boilerplate.
- @Async + virtual threads = 균형.

### 출처
JEP 444 Virtual Threads (Java 21). Spring Boot 3.2+ 자동 통합.

---

## A11. `@EventListener` — in-process pub/sub

### 무엇인가
Spring 의 이벤트 시스템. `ApplicationEventPublisher.publishEvent(event)` 호출 시 등록된 listener 자동 호출.

### 어디서 쓰나
```java
// publisher
events.publishEvent(new ScanRequested(...));

// listener (다른 클래스)
@EventListener
@Async
public void on(ScanRequested event) {
    execute(event);
}
```

### 왜 필요한가
publisher 가 listener 모름 (decoupling). 새 listener 추가 시 publisher 코드 변경 X.

VulnScope 의 `ScanRequested` 만 활성 listener 있음 (WorkerExecutor). 다른 4개 (ScanCompleted/Failed/FindingRecorded/TargetRegistered) 는 listener 0 이라 제거됨 — `study/domain-events-and-outbox.md` 참조.

### 핵심 아이디어
**Observer pattern + Spring 자동 라우팅**. cross-cutting 통신.

### 함정
- 동기 실행이 default — listener 가 publish 스레드에서 즉시 실행. `@Async` 명시해야 비동기.
- 트랜잭션 통합 X (default). `@TransactionalEventListener(AFTER_COMMIT)` 로 commit 후 실행 가능.
- 신뢰성 약함. listener 실패 시 재시도 X. **at-most-once delivery**.

### 대안
- 직접 Strategy/Observer — Spring 표준 활용 X.
- Spring Modulith `@ApplicationModuleListener` — async + AFTER_COMMIT + outbox 자동.
- 메시지 큐 (Kafka/RabbitMQ) — cross-process.

### 출처
Spring Framework 4.2+ `@EventListener`. 그 전엔 `ApplicationListener<E>` 인터페이스 구현.

---

## A12. catch-all + markFailed — worker hang 방지

### 무엇인가
async worker 의 마지막 try-catch. 어떤 예외든 잡아 도메인 상태를 "FAILED" 로 표시.

### 어디서 쓰나
```java
// WorkerExecutor.execute
try {
    scanCommand.markRunning(scanId);
    // ... HTTP fetch
    // ... module check
    // ... finding record
    // ... evidence store
    scanCommand.markCompleted(scanId, summary);
    realtime.emit(channel, "done", ...);
} catch (Exception e) {
    scanCommand.markFailed(scanId, e.getMessage());
    realtime.emit(channel, "failed", Map.of("reason",
        e.getMessage() == null ? "unknown" : e.getMessage()));
}
```

### 왜 필요한가
worker 는 외부 의존 많음 (HTTP, parser, DB). 어떤 예외든 가능:
- `IOException` (network)
- `ConnectException` (host unreachable)
- `NullPointerException` (parsing edge case)
- `OutOfMemoryError` (큰 응답)

catch 없으면:
- async 스레드의 unhandled exception → 그냥 사라짐 (Future 안 받으면).
- scan 은 영원히 QUEUED 또는 RUNNING 상태.
- 사용자는 SSE 에서 아무 이벤트 안 받음. UI 가 "Loading..." 영원.

catch-all + markFailed 면:
- DB 영속: status=FAILED, finishedAt 기록.
- SSE: failed 이벤트 → 클라가 "Failed: <reason>" 표시 + 자동 navigate.

### 핵심 아이디어
**비동기 워커는 fail-safe 종착지가 있어야 함**. 어떤 예외든 도메인 invariant 유지.

### 전문 용어 사전
- **fail-safe**: 실패해도 시스템이 의미 있는 상태로 종착. silent fail 회피.
- **stuck (hang)**: 진행 안 됨 + 종료도 안 됨. 가장 곤란한 상태.
- **observable failure**: 실패가 외부 (UI, monitoring) 에 노출됨. 처리 가능.

### 함정
- `catch (Exception e)` 는 `Error` 안 잡음 (OOM, StackOverflow). 위험. `catch (Throwable t)` 가 진짜 catch-all 이지만 거의 안 씀 (JVM 죽으면 어차피 처리 못 함).
- 잡고 swallow 만 하면 안 됨 — 반드시 도메인 상태 + monitoring (log) 둘 다.

### 대안
- 각 단계에 try-catch — verbose. 한 군데 catch-all 이 더 깔끔.
- `CompletableFuture.exceptionally(...)` — Future 사용 시.
- circuit breaker (Hystrix/Resilience4j) — 외부 호출만 wrap. 본질 다름.

---

## §∞. 정리

### 패턴 매트릭스

| 문제 | 패턴 |
|---|---|
| 멀티 스레드 Map | A1 ConcurrentHashMap |
| "처음만 생성" race-free | A2 computeIfAbsent + atomic 인스턴스 |
| race-free 카운터 | A3 AtomicLong |
| lock-free 큐 | A4 ConcurrentLinkedDeque |
| thread-safe Set | A5 newKeySet() |
| 단순 flag visibility | A6 volatile |
| thread-unsafe API 보호 | A7 synchronized 블록 |
| per-request 컨텍스트 | A8 ThreadLocal |
| ThreadLocal leak 방지 | A9 finally clear |
| async 비동기 실행 | A10 @Async + 가상 스레드 |
| in-process pub/sub | A11 @EventListener |
| async fail-safe | A12 catch-all + markFailed |

### 학습 추천 순서

1. **§0 동시성 기본** — race condition 이 무엇인지 모르면 나머지 무의미.
2. **A3 AtomicLong** + **A1 ConcurrentHashMap** — atomic 의 본질.
3. **A2 computeIfAbsent** — 두 단계 합치기의 의미.
4. **A6 volatile** — JMM happens-before 입문.
5. **A8/A9 ThreadLocal + clear** — multi-tenant SaaS 의 표준.
6. **A10 @Async + 가상 스레드** — Java 21 의 새 시대.
7. **A11/A12** — Spring 통합 + fail-safe.

### 추가 참고
- Brian Goetz 외, "Java Concurrency in Practice" — bible.
- JSR-166 (java.util.concurrent), JSR-133 (Memory Model).
- JEP 444 Virtual Threads.
- Doug Lea 의 [util.concurrent 문서](http://gee.cs.oswego.edu/dl/concurrency-interest/).
