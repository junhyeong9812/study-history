# E. 외부 인프라 추상화

> 이 문서가 다루는 것: 파일 시스템/HTTP/SSE/이벤트 같은 외부 시스템을 도메인이 모르게 감추는 기법. 6개 패턴.
> 전제: D 카테고리 (헥사고날) 의 port/adapter 이해.

---

## §0. 추상화의 의미 — 왜 도메인이 인프라를 모르게 하나?

도메인 ("스캔이 발견된 finding 의 증거") 와 인프라 ("/data/uploads/2026/05/01/abc.bin 파일") 는 **다른 변화 속도**:
- 도메인: 비즈니스 룰 — 자주 안 바뀜.
- 인프라: 기술 디테일 — 자주 바뀜 (파일 → S3 → CDN → ...).

**도메인이 인프라 직접 의존**:
- 인프라 변경 → 도메인 같이 변경. 도메인 안정성 깨짐.

**도메인이 추상화 의존**:
- 인프라 변경 → 어댑터만 변경. 도메인 무영향.

E 카테고리는 이 추상화의 6가지 실전 패턴.

---

## E1. Probe / ScanModule port 분리 (Strategy)

### 무엇인가
"HTTP fetch" (Probe) 와 "취약점 점검" (ScanModule) 을 별도 interface 로 분리.

### 어디서 쓰나
```java
// scanengine/domain/Probe.java
public interface Probe {
    ProbeResult fetch(String url) throws Exception;
}

// scanengine/domain/ScanModule.java
public interface ScanModule {
    ModuleId id();
    ModuleResult check(ProbeResult probeResult, String targetUrl);
}
```

```java
// 구현
@Component
public class HttpHeaderProbe implements Probe { ... }

@Component
public class MissingSecurityHeadersModule implements ScanModule { ... }
```

```java
// WorkerExecutor 가 둘 다 사용
ProbeResult probeResult = probe.fetch(url);                  // ① fetch
ModuleResult moduleResult = headersModule.check(probeResult, url);  // ② check
```

### 왜 분리?

**naive (한 인터페이스)**:
```java
public interface ScanProbe {
    ModuleResult run(String url);    // fetch + check 합쳐짐
}
```
**문제**:
- 새 점검 모듈 추가 시 (예: SQL injection 검사) HTTP fetch 다시 해야 함. 같은 URL 에 여러 모듈 적용 불가.
- fetch 와 check 가 다른 책임 — 분리해야 SRP.

**분리**:
- fetch 1번 → 모듈 N개 적용 가능. 비용 절감.
- 각 모듈이 같은 ProbeResult 받음.
- 새 점검 모듈 추가가 간단 (Probe 변경 0).

### 핵심 아이디어
**Strategy pattern** — 여러 알고리즘을 인터페이스로 표현, 동일 입력에 다른 처리.

`ScanModule` 의 다양한 구현 (현재 1개, v0.2 에 SQL injection / XSS 등 추가) 이 같은 ProbeResult 를 다르게 점검.

### 전문 용어 사전
- **Strategy pattern** (GoF): 알고리즘 family 를 interface 로 캡슐화. 런타임 선택.
- **SRP** (Single Responsibility Principle): 한 클래스 = 한 책임.

### 함정
- ProbeResult 가 너무 크면 (전체 응답 body) 메모리 부담. VulnScope 는 4KB cap.
- 모듈마다 다른 fetch 가 필요하면 (POST + body 등) Probe interface 확장 필요.

### 대안
- 합쳐진 ScanProbe — 모듈 N개에 fetch N번. 비용.
- Visitor pattern — 모듈이 ProbeResult 를 visit. 본질 비슷.

### 출처
GoF Design Patterns — Strategy.

---

## E2. Store + Bus 분리 (영속 vs 즉시 fanout)

### 무엇인가
실시간 이벤트의 **영속** (Store) 과 **즉시 전파** (Bus) 를 별도 interface 로.

### 어디서 쓰나
```java
// shared/realtime/RealtimeStore.java
public interface RealtimeStore {
    StoredEvent append(ChannelKey channel, String type, Object payload);
    List<StoredEvent> readSince(ChannelKey channel, long since, int limit);  // resume 용
    long latestSeq(ChannelKey channel);
    void close(ChannelKey channel);
}

// shared/realtime/RealtimeBus.java
public interface RealtimeBus {
    Subscription subscribe(ChannelKey channel, Consumer<StoredEvent> subscriber);
    void publish(StoredEvent event);

    @FunctionalInterface
    interface Subscription extends AutoCloseable {
        @Override void close();
    }
}
```

### 왜 분리?

**Store 의 역할**:
- 영속 + seq 부여 + retention.
- 클라이언트 재연결 시 backfill (`readSince`).
- 신뢰성 모델: at-least-once (영속이라 leak 없음).

**Bus 의 역할**:
- 즉시 fanout. 새 이벤트가 모든 active subscriber 에 전달.
- 영속 X. 메모리에서만 동작.
- 신뢰성 모델: at-most-once (subscriber 가 publish 시점에 없으면 놓침).

**합치면**:
- "영속 + 실시간" 한 시스템 = 복잡도. 어느 쪽 부하가 critical 인지 모호.

**분리하면**:
- Store 의 retention/storage 정책 vs Bus 의 fanout 정책 독립적으로 발전.
- VulnScope 는 in-memory 둘 다지만, v0.2 에 Store=PostgreSQL/Bus=Redis Pub/Sub 식 분리 자연.

### 핵심 아이디어
**Kafka 의 log + consumer 분리 모델** 차용. log = 영속, consumer = read.

### 합성은 어떻게?
RealtimeChannel facade (E3) 가 Store + Bus 합성.

### 전문 용어 사전
- **at-most-once delivery**: 0번 또는 1번. 중복 X, 누락 가능.
- **at-least-once delivery**: 1번 또는 그 이상. 누락 X, 중복 가능.
- **exactly-once delivery**: 정확히 1번. 가장 어려움 (분산 환경 거의 불가능, idempotent receiver 로 흉내).

### 함정
- Store 와 Bus 의 신뢰성 모델 명시 약속 필요. 헷갈리면 사용 측 잘못된 가정.
- 영속만 있으면 polling, Bus 만 있으면 resume 불가 — 둘 다 필요.

### 대안
- 합쳐진 single interface — 단순 prototype.
- 외부 message broker (Kafka, NATS) — 분산 환경.

### 출처
- Kafka Design (LinkedIn paper).
- VulnScope ADR 0007 (Store + Bus 결정).

---

## E3. RealtimeChannel facade (Store + Bus 합성)

### 무엇인가
Store 와 Bus 를 한 메서드 (`emit`) 로 묶음. 호출자가 순서 신경 X.

### 어디서 쓰나
```java
// shared/realtime/RealtimeChannel.java
@Component
public class RealtimeChannel {

    private final RealtimeStore store;
    private final RealtimeBus bus;

    public RealtimeChannel(RealtimeStore store, RealtimeBus bus) {
        this.store = store;
        this.bus = bus;
    }

    public StoredEvent emit(ChannelKey channel, String type, Object payload) {
        StoredEvent event = store.append(channel, type, payload);   // ① 먼저 영속
        bus.publish(event);                                          // ② 그 다음 발행
        return event;
    }
}
```

### 왜 Store 먼저?

**bus 먼저, store 나중**:
```
1. bus.publish(event) → subscriber 가 이벤트 받음
2. store.append(...) → 실패 (디스크 full 등)
3. 클라이언트 재연결 → store.readSince → 그 이벤트 없음
4. → 클라이언트가 이벤트 누락 인지 못 함 (이미 받았다고 생각)
```

**store 먼저, bus 나중**:
```
1. store.append(event) → 영속 성공
2. bus.publish(event) → subscriber 받음
   (만약 publish 실패해도) 클라이언트 재연결 시 readSince 로 받음
```

→ **Store first 가 일관성 보장**. consistent ordering.

### 핵심 아이디어
**Facade pattern** — 두 시스템의 합성을 단일 표면으로.

호출자 (`WorkerExecutor.realtime.emit(...)`) 는 Store/Bus 둘이 있다는 것조차 모름. 한 줄 호출.

### 함정
- store.append 성공 + bus.publish 실패 케이스: 영속됐으니 readSince 로 결국 도달. eventually consistent.
- store.append 실패 시 bus.publish 안 함 — 일관성. 호출자에 예외 전파.

### 대안
- 호출자가 직접 store.append + bus.publish — 매번 두 줄, 순서 실수 가능.
- Spring `ApplicationEventPublisher` + outbox listener — 신뢰성 강함, 복잡도 큼.

### 출처
GoF Facade pattern.

---

## E4. `Subscription` = `AutoCloseable` + functional interface

### 무엇인가
구독 해제 (`unsubscribe`) 를 객체 반환 + close 메서드로. 람다 표현 가능.

### 어디서 쓰나
```java
// shared/realtime/RealtimeBus.java
public interface RealtimeBus {
    Subscription subscribe(ChannelKey channel, Consumer<StoredEvent> subscriber);

    @FunctionalInterface              // ← 단일 메서드 = lambda 가능
    interface Subscription extends AutoCloseable {
        @Override void close();        // ← AutoCloseable 의 close. throws 제거
    }
}
```

```java
// InMemoryRealtimeBus 구현
@Override
public Subscription subscribe(ChannelKey channel, Consumer<StoredEvent> subscriber) {
    subscribers.computeIfAbsent(channel, k -> ConcurrentHashMap.newKeySet()).add(subscriber);
    return () -> {                                          // ← lambda Subscription
        Set<Consumer<StoredEvent>> set = subscribers.get(channel);
        if (set != null) set.remove(subscriber);
    };
}
```

```java
// 사용 (ScanStreamController)
RealtimeBus.Subscription subscription = bus.subscribe(channel, e -> send(emitter, e));

emitter.onCompletion(subscription::close);    // 종료 시 unsubscribe
emitter.onTimeout(subscription::close);
emitter.onError(t -> subscription.close());
```

### 왜 이 디자인?

**naive (subscribe + unsubscribe 별도 메서드)**:
```java
public interface RealtimeBus {
    void subscribe(ChannelKey channel, Consumer<...> subscriber);
    void unsubscribe(ChannelKey channel, Consumer<...> subscriber);
}
```
**문제**:
- 호출자가 subscriber lambda 를 변수로 들고 있어야 unsubscribe 가능.
- 같은 lambda 식별성 (`equals`) 어려움.

**Subscription 반환**:
- 호출 결과 객체 = 구독 핸들. 그것을 close 하면 해제.
- subscriber lambda 보관 불필요.

**AutoCloseable + functional interface**:
- `try-with-resources` 호환 — 자동 close.
- lambda 로 표현 가능 (`() -> { ... }`).

### 핵심 아이디어
**RxJava 의 `Disposable` 패턴**. 구독을 first-class object 로.

### 전문 용어 사전
- **Disposable / Subscription**: reactive 라이브러리의 구독 핸들 표준 이름.
- **AutoCloseable**: Java 7 try-with-resources 기반 인터페이스. close 자동 호출.
- **@FunctionalInterface**: 단일 abstract 메서드 인터페이스. lambda 표현 가능. 컴파일러가 강제.

### 함정
- subscribe 결과 무시 → 영원히 unsubscribe 안 됨. Memory leak.
- close 한 후 중복 close — idempotent 권장 (지금 코드는 set.remove 가 idempotent).

### 대안
- subscriber 객체에 unsubscribe 메서드 — naive.
- weak reference — GC 가 알아서 정리. 의도 불명확.

### 출처
- RxJava Disposable 패턴.
- Java 7 try-with-resources (JEP 153).

---

## E5. AutoCloseable wrapper (스트림 라이프사이클 명시)

### 무엇인가
InputStream + 메타데이터를 묶고 AutoCloseable 구현. 호출자에게 close 책임 명시.

### 어디서 쓰나
```java
// upload/application/dto/DownloadStream.java
public final class DownloadStream implements AutoCloseable {

    private final UploadView meta;
    private final InputStream content;

    public DownloadStream(UploadView meta, InputStream content) {
        this.meta = meta;
        this.content = content;
    }

    public UploadView meta() { return meta; }
    public InputStream content() { return content; }

    @Override
    public void close() throws IOException {
        content.close();
    }
}
```

### 사용
```java
// UploadController.download
Optional<DownloadStream> stream = uploadQuery.download(UploadId.of(id));
if (stream.isEmpty()) return ResponseEntity.notFound().build();

DownloadStream ds = stream.get();
// ... 권한 체크
if (!uploadOrg.equals(currentOrg)) {
    try { ds.close(); } catch (Exception ignored) {}    // ← leak 방지
    return ResponseEntity.status(403).build();
}

return ResponseEntity.ok()
    .body(new InputStreamResource(ds.content()));    // Spring 이 자동 close
```

### 왜 wrap?

**naive (분리 반환)**:
```java
public interface UploadQuery {
    Optional<UploadView> findById(UploadId id);
    InputStream openContent(UploadId id);    // ❌ 별도 호출
}
```
**문제**:
- 두 호출 시점이 다름 — 사이에 race 가능.
- 호출자가 두 번 호출 + 결과 묶기.

**wrap**:
```java
public interface UploadQuery {
    Optional<DownloadStream> download(UploadId id);    // 묶음
}
```
- meta + stream 같이 가져옴 (consistent snapshot).
- AutoCloseable 명시 → leak 위험 감소.

### 핵심 아이디어
**resource 라이프사이클을 타입으로 표현**. AutoCloseable 이 "이 객체는 close 필요" 신호.

### try-with-resources

```java
try (DownloadStream ds = uploadQuery.download(id).orElseThrow()) {
    // ... 사용
}    // 자동 close
```

VulnScope 의 controller 는 try-with 안 씀 (Spring 이 ResponseEntity body 처리 시 자동 close 보장). 권한 거부 시는 명시 close.

### 전문 용어 사전
- **resource leak**: file descriptor / DB connection 등 close 안 해서 누적. 시스템 한계 도달 시 장애.
- **idempotent close**: 여러 번 close OK. close 한 후 다시 close 해도 안전.

### 함정
- close 시 IOException — wrapper 가 throw 또는 wrap.
- 호출자가 받고 안 닫으면 leak. 패턴 강제 X (Java 가 GC 마지막에 close 하지만 늦음).

### 대안
- Function callback (`uploadQuery.withDownload(id, ds -> { ... })`) — 호출자가 수동 close 안 함. Java 의 일반 패턴 X.
- Reactive `Mono<DataBuffer>` — backpressure + 자동 lifecycle. WebFlux 환경.

### 출처
- AutoCloseable: Java 7 (JEP 153).
- try-with-resources idiom.

---

## E6. Spring `ApplicationEventPublisher` (in-process pub/sub)

### 무엇인가
Spring 의 표준 이벤트 시스템. publisher + listener decoupling.

### 어디서 쓰나
```java
// scan/application/service/TriggerScanService.java
private final ApplicationEventPublisher events;

public Scan trigger(...) {
    Scan saved = repository.save(scan);
    events.publishEvent(new ScanRequested(saved.id(), ...));    // ← publish
    return saved;
}
```

```java
// scanengine/application/service/WorkerExecutor.java
@EventListener
@Async
public void on(ScanRequested event) {
    execute(event);
}
```

### 왜 필요한가
**naive (publisher 가 listener 직접 호출)**:
```java
public Scan trigger(...) {
    Scan saved = repository.save(scan);
    workerExecutor.execute(saved.id());    // ❌ scan → scanengine 직접 의존
    return saved;
}
```
**문제**:
- scan 모듈이 scanengine 모듈 import. cross-module 결합 폭증.
- 새 listener (notification) 추가 시 scan 코드 변경.

**publishEvent**:
- scan 모듈은 `ScanRequested` 만 알면 됨. listener 모름.
- listener 추가/제거 자유.

### 핵심 아이디어
**Observer pattern + Spring 자동 라우팅**.

publisher 가 `events.publishEvent(obj)` 하면 Spring 이 등록된 모든 `@EventListener(of obj's type)` 호출.

### 신뢰성 한계
**default 동작**:
- 동기 (publish 스레드에서 listener 즉시 실행) — `@Async` 명시해야 비동기.
- 트랜잭션 통합 X — `@TransactionalEventListener(AFTER_COMMIT)` 또는 outbox.
- listener 실패 시 재시도 X. **at-most-once delivery**.

→ 진짜 신뢰성 (at-least-once + DB 영속) 필요하면 Spring Modulith outbox + `@ApplicationModuleListener`. `study/domain-events-and-outbox.md` 참조.

### 전문 용어 사전
- **ApplicationEventPublisher**: Spring 의 발행자.
- **@EventListener**: 구독 메서드 어노테이션.
- **@TransactionalEventListener**: 트랜잭션 phase (BEFORE_COMMIT, AFTER_COMMIT, AFTER_ROLLBACK) 별 호출.
- **Observer pattern** (GoF): subject + observer. publish/subscribe 의 일반 형태.

### 함정
- 동기 default — 느린 listener 가 publisher 응답 시간 영향.
- 같은 클래스 안에서 self-call (publishEvent 후 자기 @EventListener) → Spring proxy 우회 → 동기 실행. on() vs execute() 분리 필요 (가상 스레드 환경에서 자세히).

### 대안
- 직접 의존 — 결합 폭증.
- 외부 message queue (Kafka) — 분산.
- ScanRequested 만 활성, 다른 4개 제거 — VulnScope 의 결정 (`study/domain-events-and-outbox.md`).

### 출처
- Spring Framework 4.2+ `@EventListener`.
- VulnScope 의 결정: `study/domain-events-and-outbox.md`.

---

## §∞. 정리

### 패턴 매트릭스

| 문제 | 패턴 |
|---|---|
| fetch + check 분리 | E1 Probe + ScanModule (Strategy) |
| 영속 + 실시간 분리 | E2 Store + Bus |
| 두 시스템 합성 | E3 RealtimeChannel facade |
| 구독 핸들 | E4 Subscription (AutoCloseable + lambda) |
| 스트림 라이프사이클 | E5 DownloadStream (AutoCloseable wrapper) |
| in-process pub/sub | E6 ApplicationEventPublisher |

### 학습 추천 순서

1. **§0 추상화의 의미** — 동기.
2. **E1 Strategy** — 가장 표준 GoF 패턴.
3. **E2 Store + Bus** — Kafka 의 log+consumer 차용.
4. **E3 Facade** — 합성.
5. **E4/E5 AutoCloseable** — 라이프사이클 명시.
6. **E6 ApplicationEventPublisher** — Spring 표준 (한계 인지).

### 추가 참고
- GoF Design Patterns (Strategy, Facade, Observer, Adapter).
- Project Reactor Disposable 패턴.
- Kafka Design 문서.
- VulnScope ADR 0007 (Store + Bus 결정).
- `study/domain-events-and-outbox.md`.
