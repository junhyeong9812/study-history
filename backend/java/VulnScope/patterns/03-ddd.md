# C. DDD 도메인 모델링

> 이 문서가 다루는 것: Eric Evans 의 Domain-Driven Design 핵심 패턴들 — Aggregate, Value Object, invariant, state machine, domain event. VulnScope 의 6개 도메인 모듈에 모두 적용. 11개 패턴.
> 전제: Java record 기본. 객체지향 (encapsulation) 의 의미.

---

## §0. DDD 가 무엇이고 왜 중요한가?

### 0.1 anemic domain model 의 문제

전형적인 잘못된 패턴:
```java
// "Anemic Domain Model" — 도메인이 단순 data holder
public class Scan {
    private UUID id;
    private ScanStatus status;
    private Instant createdAt;
    // getters / setters 만 ...
}

@Service
public class ScanService {
    public void markRunning(Scan scan) {
        if (scan.getStatus() != ScanStatus.QUEUED) {
            throw new IllegalStateException(...);
        }
        scan.setStatus(ScanStatus.RUNNING);    // ← 누구나 set
        scan.setStartedAt(Instant.now());
    }
}
```

**문제**:
- `scan.setStatus(...)` 를 누구나 호출. invariant (QUEUED → RUNNING 만 허용) 가 service 한 군데에만. 다른 서비스가 잘못 호출하면 깨짐.
- Scan 은 데이터 가방 (data bag). "scan 이 무엇을 할 수 있는가" 가 코드에 안 보임.
- 비즈니스 로직이 service 에 흩어짐.

Martin Fowler 가 이걸 [Anemic Domain Model anti-pattern](https://martinfowler.com/bliki/AnemicDomainModel.html) 이라 부름.

### 0.2 Rich Domain Model

**해결**: 도메인 객체에 행동 + invariant 보장.

```java
public record Scan(ScanId id, ScanStatus status, ...) {

    public Scan markRunning(Clock clock) {
        if (!status.canTransitionTo(ScanStatus.RUNNING)) {
            throw new ScanCannotTransition(status, RUNNING);  // 도메인이 거부
        }
        return new Scan(id, RUNNING, ...);                   // 새 인스턴스
    }
}
```

**효과**:
- Scan 자신이 invariant 책임. service 가 잘못 호출 → 도메인이 막음.
- "Scan 이 할 수 있는 행동" 이 메서드 시그니처에 명시.
- service 는 조립만 (orchestration).

### 0.3 DDD 의 핵심 빌딩 블록

Eric Evans 의 책 *Domain-Driven Design* (2003) 의 패턴들:
- **Aggregate**: 일관성 경계가 있는 객체 묶음 (보통 1개 root + 자식들).
- **Aggregate Root**: 외부에서 접근 가능한 진입점.
- **Value Object**: ID 없는 의미 단위 (`Sha256`, `TargetValue`).
- **Repository**: Aggregate 를 영속/조회하는 추상화.
- **Domain Service**: 도메인 행동인데 단일 Aggregate 에 안 들어가는 것.
- **Domain Event**: 도메인 사실의 발생 통지.
- **Bounded Context**: 한 모델이 일관된 의미를 가지는 경계.

VulnScope 는 이 패턴들을 모두 사용.

---

## C1. Aggregate Root (record)

### 무엇인가
도메인 객체. 일관성 경계의 진입점. 외부는 Aggregate Root 만 참조 (자식 직접 X).

VulnScope 의 6개:
- `Scan`, `Finding`, `Target`, `Upload`, `Evidence`, `Profile`.

### 어디서 쓰나
```java
// scan/domain/Scan.java
public record Scan(
    ScanId id,
    OrgId orgId,
    TargetId targetId,
    ProfileId profileId,
    ScanStatus status,
    ScanSummary summary,
    Instant createdAt,
    Instant startedAt,
    Instant finishedAt
) {
    public static Scan request(...) { ... }
    public Scan markRunning(Clock clock) { ... }
    public Scan markCompleted(...) { ... }
    public Scan markFailed(Clock clock) { ... }
}
```

### 왜 record?

**불변성 (immutability)**:
- record 는 mutator 없음 (setter X). 모든 필드 final.
- 한번 만들면 변경 X. 동시성 자동 안전.

**equals/hashCode/toString 자동**:
- 모든 필드 사용한 표준 구현.
- `Scan a == Scan b` 가 가능 (값 비교).

**state transition 메서드** (markRunning 등):
- 새 record 인스턴스 반환 (immutable update).
- 기존 인스턴스는 그대로 — 다른 스레드가 들고 있어도 안전.

### 핵심 아이디어
**Aggregate = 비즈니스 트랜잭션의 단위**. 한 번에 일관성 보장해야 하는 데이터 묶음.

예: Scan 의 `status` + `summary` + `finishedAt` 은 함께 변경 (markCompleted) — 따로 변경되면 invariant 깨짐. 그래서 같은 Aggregate.

### 전문 용어 사전
- **Aggregate**: 일관성 경계 안의 객체들. 한 트랜잭션 = 한 Aggregate 변경.
- **Aggregate Root**: Aggregate 의 진입점. 외부는 root 만 참조.
- **invariant**: 항상 true 여야 하는 조건. 예: "Scan 의 startedAt 은 createdAt 이후".
- **immutable**: 변경 불가. 동시성 + 추론 단순.

### 함정
- Aggregate 가 너무 크면 (수십 필드) — 분리 후보. SRP 위반.
- 두 Aggregate 동시 변경은 별도 트랜잭션 + 도메인 이벤트로 결합 (eventual consistency).

### 대안
- 일반 class + setter — anemic model. DDD 위반.
- Lombok `@Value` — Java 14 미만에선 비슷한 의미. record 가 표준.

### 출처
Eric Evans, *Domain-Driven Design* (2003) — Chapter 6 Aggregates.

---

## C2. Value Object (record + 검증)

### 무엇인가
ID 없는 도메인 객체. 값 자체가 의미. 같은 값이면 같은 객체.

VulnScope:
- `Sha256(String hex)` — 콘텐츠 해시.
- `TargetValue(String raw)` — URL/IP 문자열.
- `ChannelKey(String value)` — SSE 채널.
- `ScanId/TargetId/.../ProfileId` — UUID wrapper (식별자지만 value object 형식).
- `ModuleId(String value)` — 모듈 이름.

### 어디서 쓰나
```java
// shared/kernel/Sha256.java
public record Sha256(String hex) {
    public Sha256 {
        Objects.requireNonNull(hex, "hex");
        if (hex.length() != 64 || !hex.matches("[0-9a-f]{64}")) {
            throw new IllegalArgumentException("sha256 hex must be 64 lowercase hex chars");
        }
    }

    public static Sha256 of(String hex) { return new Sha256(hex); }
    public static Sha256 ofBytes(byte[] bytes) { ... }
}
```

### 왜 필요한가

**naive (primitive obsession)**:
```java
public Upload(... String sha256) { ... }     // ❌

storage.findByHash("invalid hex");           // ❌ silent. 잘못된 형식이 통과
```

**Value object**:
```java
public Upload(... Sha256 sha256) { ... }     // ✅ 타입으로 강제

storage.findByHash(Sha256.of("invalid"));    // ❌ throw at parse time
```

→ **잘못된 형식이 시스템에 들어올 수 없음**. 모든 entry point 에서 검증.

### 핵심 아이디어
**의미 있는 값은 타입으로 표현**. 단순 String/long 은 의미 없음 + 검증 없음.

Eric Evans 의 핵심 통찰:
- 도메인 어휘 (`Sha256`, `TargetValue`) 가 코드에 직접 등장.
- 잘못된 값 차단을 type system 에 위임.

### 전문 용어 사전
- **Value Object**: ID 없음. equals 가 모든 필드 비교. 변경되면 새 인스턴스.
- **primitive obsession**: 기본 타입 (String, long) 만 사용. 의미 손실 + 검증 부재.
- **smart constructor** (Haskell 용어): 검증된 instance 만 만드는 생성자. compact constructor 가 같은 의미.

### 차이: Value Object vs Entity
- **Entity** (Aggregate Root): ID 로 식별. ID 같으면 같은 객체 (속성 다르더라도).
- **Value Object**: 값으로 식별. 모든 속성 같으면 같은 객체.

→ ID `0000-...0001` 인 Scan 은 status 가 어떻든 같은 Scan. ID 다른 Scan 은 다른 객체 (속성 같아도).
→ `Sha256("a3f5...")` 는 어디서 만들든 같은 Sha256.

### 함정
- Value Object 인데 mutable 필드 — 외부 변경 시 hash 깨짐. record + immutable 강제.
- equals 깨면 HashMap 키로 못 씀.

### 대안
- Lombok `@Value` (Java 14 미만).
- 직접 클래스 + equals/hashCode — boilerplate.
- record 가 표준.

### 출처
Eric Evans, *DDD* — Chapter 5 Value Objects.

---

## C3. Compact constructor 검증

### 무엇인가
record 의 특수 constructor. canonical constructor 호출 전 검증/정규화.

### 어디서 쓰나
```java
// Finding 의 compact constructor
public record Finding(
    FindingId id,
    OrgId orgId,
    ScanId scanId,
    long seq,
    ModuleId moduleId,
    Severity severity,
    OwaspId owaspId,
    String title,
    String location,
    String param,
    FindingStatus status,
    Instant firstSeenAt,
    Instant lastSeenAt
) {

    public Finding {
        Objects.requireNonNull(id, "id");
        Objects.requireNonNull(orgId, "orgId");
        Objects.requireNonNull(scanId, "scanId");
        Objects.requireNonNull(moduleId, "moduleId");
        Objects.requireNonNull(severity, "severity");
        Objects.requireNonNull(title, "title");
        Objects.requireNonNull(location, "location");
        Objects.requireNonNull(status, "status");
        Objects.requireNonNull(firstSeenAt, "firstSeenAt");
        Objects.requireNonNull(lastSeenAt, "lastSeenAt");
        if (seq < 1) throw new IllegalArgumentException("seq must be >= 1");
        if (title.isBlank()) throw new IllegalArgumentException("title must not be blank");
    }
    // ...
}
```

### 왜 필요한가
**모든 생성 경로** (정적 팩토리, 외부 new, JSON deserialize, reflection) 가 compact constructor 거침.

검증 한 곳 → 모든 instance 가 valid 보장 (invariant).

### 문법 디테일
```java
public Finding {                       // 파라미터 시그니처 없음 (canonical 과 동일)
    // 검증/정규화 코드
    // 명시적 this.field = ... 안 함 — 자동
}
```

vs 일반 constructor:
```java
public Finding(FindingId id, ... long seq, ...) {
    this.id = id;                      // 명시
    this.seq = seq;
    // ...
}
```

### 핵심 아이디어
**모든 instance 가 valid 라는 type-level 보장**. invalid Finding 이 시스템에 못 들어옴.

### 전문 용어 사전
- **canonical constructor**: 모든 필드 받는 표준 생성자. record 가 자동 생성.
- **compact constructor**: 시그니처 없는 검증/정규화 전용. canonical 호출 전.
- **invariant**: 항상 true 인 조건. constructor 검증 = invariant 의 보장.

### 함정
- compact constructor 안에서 `this` 사용 X (필드 아직 할당 전).
- 정규화 가능 (`modules = List.copyOf(modules);` 같은) — 람다 안에서 파라미터 재할당.

### 대안
- 정적 팩토리 + private 생성자 — 더 strict. 단점: 외부에서 `new` 호출 차단 어려움 (record 는 자동 public canonical).
- factory + assertion — runtime 비용 동일. compact 가 표준.

### 출처
JEP 395 Records (Java 14).

---

## C4. Static factory method

### 무엇인가
도메인 객체 생성을 명시 메서드로. `new Scan(...)` 직접 호출 대신 `Scan.request(...)`.

### 어디서 쓰나
| 도메인 | 정적 팩토리 |
|---|---|
| Scan | `request(...)` — QUEUED 로 시작 |
| Finding | `record(...)` — OPEN + firstSeen=lastSeen=now |
| Target | `register(...)` — lastUsedAt=null |
| Upload | `create(...)` — uploadedAt 자동 |
| Evidence | `link(...)` — createdAt 자동 |
| Profile | `system(...)` 또는 `custom(...)` — kind 분기 |

```java
// Scan.java
public static Scan request(ScanId id, OrgId orgId, TargetId targetId, ProfileId profileId, Clock clock) {
    return new Scan(id, orgId, targetId, profileId,
        ScanStatus.QUEUED, null, clock.instant(), null, null);
}
```

### 왜 필요한가

**naive new**:
```java
new Scan(id, orgId, targetId, profileId, QUEUED, null, clock.instant(), null, null);
```
- 의도 안 보임 (왜 QUEUED 인지, null 들이 무엇인지).
- 매번 boilerplate.

**static factory**:
```java
Scan.request(id, orgId, targetId, profileId, clock);
```
- 의도 명시 (`request` = "신규 트리거").
- 보일러 감춤.
- Profile 의 `system()` vs `custom()` 처럼 **다중 팩토리** 가능.

### 핵심 아이디어
**Effective Java Item 1 — "Consider static factory methods instead of constructors"**.

장점:
1. **이름 있음** — `Scan.request` vs `new Scan`. 의도 명시.
2. **새 인스턴스 매번 안 만들어도 됨** — caching 가능 (`Boolean.valueOf` 등).
3. **하위 타입 반환 가능** — 인터페이스 반환, 구현 숨김.
4. **다양한 의도 표현** — `Profile.system()`, `Profile.custom()` 둘 다.

### 전문 용어 사전
- **factory method**: 생성을 메서드로 wrap. GoF Factory Method pattern.
- **named constructor idiom** (C++): 같은 의도. C++ 에선 더 강력.

### 함정
- `new` 와 정적 팩토리 둘 다 허용 시 일관성 깨짐. 컨벤션: 정적 팩토리만 사용.
- record 는 canonical constructor 가 항상 public — `new` 직접 호출 가능. 컨벤션 강제 어려움.

### 대안
- 일반 생성자만 — 위 단점들.
- Builder pattern — 필드 많고 optional 많을 때. record 의 정적 팩토리 + overloading 으로 대부분 해결.

### 출처
Joshua Bloch, *Effective Java* — Item 1.

---

## C5. State machine in enum

### 무엇인가
상태 전이 규칙을 enum 메서드로. 새 상태 추가 시 컴파일러가 누락 알림.

### 어디서 쓰나
```java
// scan/domain/ScanStatus.java
public enum ScanStatus {
    QUEUED,
    RUNNING,
    DONE,
    FAILED,
    CANCELED;

    public boolean isTerminal() {
        return this == DONE || this == FAILED || this == CANCELED;
    }

    public boolean canTransitionTo(ScanStatus next) {
        return switch (this) {
            case QUEUED -> next == RUNNING || next == CANCELED || next == FAILED;
            case RUNNING -> next == DONE || next == FAILED || next == CANCELED;
            case DONE, FAILED, CANCELED -> false;
        };
    }
}
```

### 왜 필요한가

**naive (Map 기반)**:
```java
Map<ScanStatus, Set<ScanStatus>> TRANSITIONS = Map.of(
    QUEUED, Set.of(RUNNING, CANCELED, FAILED),
    RUNNING, Set.of(DONE, FAILED, CANCELED)
);

boolean canTransitionTo(ScanStatus from, ScanStatus to) {
    return TRANSITIONS.getOrDefault(from, Set.of()).contains(to);
}
```
**문제**:
- runtime 데이터. 검증 약함.
- 새 상태 (예: PAUSED) 추가 시 Map 누락 — runtime 까지 안 잡힘.

**enum + switch expression**:
```java
return switch (this) {
    case QUEUED -> ...
    case RUNNING -> ...
    case DONE, FAILED, CANCELED -> false;
};
```
**효과**:
- switch expression 은 **exhaustive** — 모든 case 처리 안 하면 컴파일 에러.
- 새 상태 추가 시 즉시 모든 switch 가 컴파일 에러 → 누락 X.
- 데이터가 아니라 코드 — refactor 도구 활용 가능.

### 핵심 아이디어
**enum 의 다형성 + switch expression 의 exhaustive check**. type-driven design.

### 전문 용어 사전
- **switch expression** (Java 14+): switch 가 값 반환. break 없음. exhaustive 강제.
- **exhaustive**: 모든 case 처리. 누락 시 컴파일 에러.
- **state machine**: 유한한 상태 + 전이 규칙. FSM (Finite State Machine).

### 함정
- enum 안에서 추상 메서드 + 각 case override 도 가능. switch 보다 OO 스타일. VulnScope 는 switch 가 더 단순해서 사용.
- 모든 transition 의 시점 (createdAt → startedAt 같은 타임스탬프) 도 invariant — Aggregate 가 보장.

### 대안
- if/else 체인 — 누락 검증 약함.
- State pattern (GoF) — 각 상태가 별도 클래스. 복잡한 행동 분기에 좋음. 단순 boolean 검증엔 enum 충분.
- 외부 library (Spring State Machine, Stateless4j) — 복잡한 워크플로우용. VulnScope 는 5상태라 over-engineering.

### 출처
- enum methods: Joshua Bloch, *Effective Java* — Item 38.
- switch expression: JEP 361 (Java 14).
- State pattern: GoF Design Patterns.

---

## C6. State transition method on Aggregate (immutable)

### 무엇인가
Aggregate 가 자기 invariant 를 책임. transition 메서드 호출 → 검증 후 새 인스턴스 반환.

### 어디서 쓰나
```java
// Scan.markRunning
public Scan markRunning(Clock clock) {
    if (!status.canTransitionTo(ScanStatus.RUNNING)) {
        throw new ScanCannotTransition(status, ScanStatus.RUNNING);
    }
    return new Scan(id, orgId, targetId, profileId,
        ScanStatus.RUNNING, null, createdAt, clock.instant(), null);
}
```

### 사용 흐름
```java
// service
Scan scan = repository.findById(id).orElseThrow();
Scan running = scan.markRunning(clock);    // 새 인스턴스
repository.save(running);                  // 영속

// 원본 scan 은 그대로 (불변).
// 다른 스레드가 들고 있어도 안전.
```

### 왜 필요한가

**잘못된 패턴 (mutable + setter)**:
```java
public class Scan {
    private ScanStatus status;
    public void setStatus(ScanStatus s) { this.status = s; }    // ❌
}

// 호출 측
scan.setStatus(RUNNING);            // 누구나 호출. invariant 검증 X.
scan.setStatus(DONE);               // RUNNING 안 거치고도 DONE? OK?
```

**올바른 패턴 (immutable + transition)**:
```java
Scan running = scan.markRunning(clock);    // 검증 통과해야 RUNNING
Scan done = running.markCompleted(summary, clock);  // RUNNING → DONE 만
```

→ 도메인 자체가 룰 강제. service 가 잘못 호출 → 즉시 예외.

### 핵심 아이디어
**"Tell, don't ask"** — 객체에 명령 (markRunning) 하지 상태 물어서 분기 X.

```java
// ❌ ask
if (scan.getStatus() == QUEUED) scan.setStatus(RUNNING);

// ✅ tell
scan = scan.markRunning(clock);    // 객체가 알아서 검증
```

### 전문 용어 사전
- **Tell, don't ask**: 객체지향 원칙. 객체에게 행동 요청, 내부 상태 묻지 X.
- **immutable update**: 변경 시 새 인스턴스. functional 스타일.
- **persistent data structure**: immutable + 효율적 partial copy. Clojure 의 vector 등.

### 함정
- 매 transition 시 새 인스턴스 → GC 부담. 일반적으론 무시 가능 (record 가 작음). high-throughput 시 lock 기반 mutable 고려.
- transition 결과를 받아서 save 안 하면 (`scan.markRunning(c)` 결과 무시) silent bug. 컨벤션: 항상 변수에 받아 save.

### 대안
- Mutable Aggregate — 동시성 위험 + invariant 검증 분산.
- Event Sourcing — 모든 transition 을 이벤트로 기록. 복잡 (별도 doc 주제).

### 출처
- Tell don't ask: Andy Hunt & Dave Thomas, *Pragmatic Programmer*.
- Immutable Aggregate: Vaughn Vernon, *Implementing DDD*.

---

## C7. Domain exception (RuntimeException 상속)

### 무엇인가
도메인 룰 위반을 명시 예외 클래스로. ExceptionHandler 가 HTTP status 매핑.

### 어디서 쓰나
8개 도메인 예외:
- `ScanCannotTransition`, `ScanAlreadyRunning`, `TargetNotAccessible`, `InvalidProfile` (scan)
- `InvalidTargetValue`, `TargetAlreadyExists` (target)
- `FindingCannotTransition` (finding)
- `UploadStorageFailure` (upload)

```java
// scan/domain/ScanAlreadyRunning.java
public class ScanAlreadyRunning extends RuntimeException {
    public ScanAlreadyRunning(TargetId targetId) {
        super("a scan is already running for target " + targetId.value());
    }
}
```

```java
// presentation/ScanExceptionHandler.java
@ExceptionHandler(ScanAlreadyRunning.class)
public ResponseEntity<ProblemDetail> alreadyRunning(ScanAlreadyRunning e) {
    return GlobalExceptionHandler.problem(HttpStatus.CONFLICT,
        "scan-already-running", "Scan Already Running", e.getMessage());
}
```

### 왜 RuntimeException?

**checked vs unchecked**:
- **checked** (`Exception` 직접 상속) — 호출자가 catch 또는 throws 명시 강제.
- **unchecked** (`RuntimeException` 상속) — 강제 X.

도메인 예외 = unchecked 권장:
- 도메인 룰 위반은 보통 **버그 또는 클라이언트 잘못된 입력** — service/controller 가 일반 처리.
- 매 메서드에 throws 적으면 boilerplate.
- ExceptionHandler 가 HTTP status 매핑 → controller 깨끗.

### 왜 일반 예외 (`IllegalStateException`) 안 씀?

**일반 예외**:
```java
throw new IllegalStateException("scan already running");
```
- ExceptionHandler 가 catch 정밀하게 못 함. 모든 ISE 가 같은 status.

**도메인 예외**:
```java
throw new ScanAlreadyRunning(targetId);
```
- 정확한 catch (`@ExceptionHandler(ScanAlreadyRunning.class)`).
- HTTP status 정확히 (409 Conflict).
- 메시지도 도메인 의미.

### 핵심 아이디어
**도메인 어휘를 예외에도 적용**. exception type 이 도메인 fact.

### 전문 용어 사전
- **checked exception**: catch 또는 throws 강제. Java 만의 특이.
- **unchecked exception** (RuntimeException): 강제 X. 대부분 언어의 표준.
- **exception translation**: 낮은 layer 의 예외를 높은 layer 의 의미로 wrap. 예: IOException → UploadStorageFailure.

### 함정
- 너무 많은 도메인 예외 — type explosion. 비슷한 룰은 묶기 (예: 모든 transition 위반 → 단일 `CannotTransition` enum 인자).
- exception driven control flow — 정상 흐름에 throw/catch 사용 X. 비싸고 의도 흐림.

### 대안
- Result type (`Result<T, E>`) — Rust 스타일. Java 표준 X. vavr 라이브러리.
- 도메인 예외 + ExceptionHandler 가 Java/Spring 표준.

### 출처
- Joshua Bloch, *Effective Java* — Item 70 (use checked exceptions for recoverable, unchecked for programming errors).
- Eric Evans, *DDD* — domain exception 의 의미.

---

## C8. Domain event (record)

### 무엇인가
도메인에서 발생한 사실 (fact) 을 record 로 표현. 다른 모듈이 listen.

VulnScope 의 유일 활성 이벤트:
- `ScanRequested` — TriggerScanService 발행, WorkerExecutor 구독.

(다른 4개는 listener 0 이라 제거됨 — `study/domain-events-and-outbox.md`)

### 어디서 쓰나
```java
// scan/domain/event/ScanRequested.java
public record ScanRequested(
    ScanId scanId,
    OrgId orgId,
    TargetId targetId,
    Instant requestedAt
) {
}
```

```java
// 발행
events.publishEvent(new ScanRequested(saved.id(), saved.orgId(), saved.targetId(), saved.createdAt()));

// 구독
@EventListener
@Async
public void on(ScanRequested event) {
    execute(event);
}
```

### 왜 필요한가

**naive (직접 호출)**:
```java
public Scan trigger(...) {
    Scan scan = ...;
    repository.save(scan);
    workerExecutor.execute(scan.id());    // ❌ scan 모듈이 worker 모듈 직접 의존
    return scan;
}
```
**문제**:
- scan 모듈이 worker 모듈 import. 결합도.
- 새 listener (notification, audit) 추가 시 scan 모듈 변경.

**Domain event**:
```java
events.publishEvent(new ScanRequested(...));   // scan 모듈 → ScanRequested 만 알면 됨
```
- publisher 가 listener 모름. decoupling.
- 새 listener 추가 시 publisher 코드 변경 0.

### 핵심 아이디어
**Observer pattern + 도메인 어휘**. 이벤트 = 도메인의 fact.

### 전문 용어 사전
- **domain event**: 도메인에서 일어난 일을 알리는 객체. past tense (`ScanRequested`, 명사형 X).
- **event-driven architecture**: 이벤트 기반 통신. 강한 결합 회피.
- **at-most-once / at-least-once / exactly-once delivery**: 이벤트 전달 보장 수준.

### 함정
- listener 없는 이벤트 발행 = silent no-op. VulnScope 가 4개 제거한 이유.
- 동기 listener 가 publish 스레드에서 실행 — 느린 listener 가 publish 느리게.
- transaction 통합 X (default) — `@TransactionalEventListener` 또는 outbox.

### 대안
- 직접 의존 — 결합 폭증.
- 외부 메시지 큐 (Kafka) — cross-process. in-process 면 over-engineering.

### 출처
- Vaughn Vernon, *Implementing DDD* — Chapter 8.
- `study/domain-events-and-outbox.md` — outbox 패턴 + listener 없는 이벤트 결정.

---

## C9. Repository port (interface in domain layer)

### 무엇인가
Aggregate 영속/조회 추상화. 도메인 layer 에 interface, infrastructure layer 에 구현.

### 어디서 쓰나
```java
// domain/ScanRepository.java
public interface ScanRepository {
    Scan save(Scan scan);
    Optional<Scan> findById(ScanId id);
    List<Scan> findByTarget(TargetId targetId);
    boolean hasRunningForTarget(TargetId targetId);
}
```

```java
// infrastructure/persistence/InMemoryScanRepository.java (구현)
@Component
public class InMemoryScanRepository implements ScanRepository {
    private final ConcurrentMap<UUID, Scan> byId = new ConcurrentHashMap<>();
    // ...
}
```

```java
// application/service/ — interface 의존
@Service
public class TriggerScanService {
    private final ScanRepository repository;        // interface!
    public TriggerScanService(ScanRepository repository) {
        this.repository = repository;               // Spring DI
    }
}
```

### 왜 필요한가

**naive (구현 직접 의존)**:
```java
@Service
public class TriggerScanService {
    private final InMemoryScanRepository repository;    // ❌
}
```
**문제**:
- 도메인이 인프라 (in-memory) 의존. v0.2 DB 전환 시 도메인 코드 수정.
- 테스트에서 mock 어려움.

**interface 의존**:
- 도메인이 "Scan 영속/조회" 만 알고 어떻게 (DB/in-memory/file) 모름.
- 구현 교체 시 도메인 변경 0.

### 핵심 아이디어
**의존 역전 원칙 (Dependency Inversion Principle, DIP)** — 상위 layer 가 하위 layer 의 abstraction (interface) 에 의존, 구체 구현 X.

### 전문 용어 사전
- **DIP**: SOLID 의 D. 추상화 의존.
- **port (in hexagonal)**: 도메인이 외부와 통신하는 interface. inbound (controller call) / outbound (repository call).
- **adapter (in hexagonal)**: port 의 구현. 인프라 레이어.

### 함정
- repository interface 가 너무 큼 → split (CQRS 의 Command/Query 분리도 한 방법).
- 구현 X 인 interface 만 두면 무의미. test 용이라도 1개 구현 필수.

### 대안
- Spring Data JPA 의 `JpaRepository` — interface 만 정의, Spring 이 구현 자동 생성. v0.2 후보.
- 직접 구현 — 단순.

### 출처
- 헥사고날: Alistair Cockburn, *Hexagonal Architecture*.
- DIP: Robert C. Martin, *Clean Architecture*.

---

## C10. Linker aggregate (의미 부여 + 두 도메인 연결)

### 무엇인가
두 다른 도메인을 연결하면서 의미 (kind) 부여하는 작은 Aggregate. 콘텐츠 자체는 다른 모듈.

### 어디서 쓰나
```java
// evidence/domain/Evidence.java
public record Evidence(
    EvidenceId id,
    FindingId findingId,    // ← finding 모듈
    EvidenceKind kind,       // ← 의미 (HAR/RAW_HTTP/SCREENSHOT/...)
    UploadId uploadId,       // ← upload 모듈
    Instant createdAt
) {
    public static Evidence link(...) { ... }
}
```

### 왜 필요한가
**naive (Finding 안에 evidence list)**:
```java
public record Finding(... List<Upload> evidences) { ... }    // ❌
```
**문제**:
- finding 이 upload 의존 — 모듈 경계 깨짐.
- evidence 의 kind 같은 메타 어디 둘지 모호.

**Linker aggregate**:
- evidence 가 두 모듈 (finding, upload) 의 식별자만 보유.
- kind = evidence 만의 의미 추가.
- finding 모듈은 evidence 모름. upload 모듈도 evidence 모름. evidence 만 둘 다 알음.

### 핵심 아이디어
**M:N 관계 + 메타데이터 = 별도 Aggregate**. RDBMS 의 join table 의 객체지향 대응.

### 전문 용어 사전
- **linker** (또는 association class): 두 entity 의 관계 자체가 의미를 갖는 객체.
- **junction table** (RDBMS): M:N 의 SQL 표현.

### 함정
- Aggregate 너무 작아 보일 수 있음 — but 의미 명확 + 모듈 분리에 결정적.
- 양쪽 모듈이 Evidence 직접 의존 X — 자기 식별자만 알면 됨.

### 대안
- Finding 내부 list — 모듈 결합. v0.1 회피.
- Many-to-many in single Aggregate — RDBMS 컨벤션 따른 모델. DDD 에선 derived.

### 출처
- Eric Evans, *DDD* — Chapter 5 (associations).
- VulnScope 의 evidence 가 정확한 적용 사례.

---

## C11. Visibility 정책 (도메인 메서드)

### 무엇인가
권한/visibility 룰을 도메인 메서드에 둠. service/controller 가 정책 모름.

### 어디서 쓰나
```java
// profile/domain/Profile.java
public boolean isVisibleTo(OrgId viewerOrg) {
    return kind == ProfileKind.SYSTEM
        || (orgId != null && orgId.equals(viewerOrg));
}
```

사용:
```java
// ProfileQueryService.existsAndVisibleTo
return repository.findById(id)
    .map(p -> p.isVisibleTo(orgId))
    .orElse(false);
```

또는 repository:
```java
// InMemoryProfileRepository.findVisibleTo
return byId.values().stream()
    .filter(p -> p.isVisibleTo(orgId))
    .sorted(...)
    .toList();
```

### 왜 필요한가
**naive (service 에 룰)**:
```java
public List<Profile> findVisibleTo(OrgId orgId) {
    return repository.findAll().stream()
        .filter(p -> p.kind() == SYSTEM || p.orgId().equals(orgId))
        .toList();
}
```
**문제**:
- 룰이 service 에. 다른 곳 (예: existsAndVisibleTo) 에서 같은 룰 다시 작성 → 중복 + drift.
- profile 의 "보이는가" 는 profile 자신의 의미 — 도메인 책임.

**도메인 메서드**:
- 룰 1군데. 일관성.
- 도메인 어휘 (`isVisibleTo`) 가 사용처에서 자연스럽게 읽힘.

### 핵심 아이디어
**도메인 룰은 도메인 객체 안에**. service 는 조립만.

### 전문 용어 사전
- **specification pattern** (DDD): 도메인 룰을 객체로 캡슐화. `Specification<Profile>.isSatisfiedBy(profile)` 형태. VulnScope 는 단순화 (메서드 직접).
- **policy**: 행동 규칙. domain policy = 도메인 객체에 위임.

### 함정
- 도메인 메서드가 외부 의존 (DB lookup 등) 필요하면 곤란 — domain service 로 분리.
- VulnScope 의 isVisibleTo 는 self-contained (Profile 의 필드만 봄) — 도메인에 자연.

### 대안
- Spring Security `@PreAuthorize` — annotation 기반. 외부 의존.
- Specification 객체 — 복잡한 조합 룰에 적합. 단순 boolean 엔 메서드 충분.

### 출처
- DDD Specification: Eric Evans, *DDD* — Chapter 9.
- "Tell don't ask": C6 참조.

---

## §∞. 정리

### 패턴 매트릭스

| 문제 | 패턴 |
|---|---|
| 일관성 단위 | C1 Aggregate Root |
| 의미 있는 값 | C2 Value Object |
| 모든 instance valid 보장 | C3 compact constructor |
| 생성 의도 명시 | C4 static factory |
| 상태 전이 룰 | C5 enum state machine |
| invariant 보장 | C6 transition method (immutable) |
| 도메인 룰 위반 표현 | C7 domain exception |
| cross-module 알림 | C8 domain event |
| 영속 추상화 | C9 repository port |
| M:N + 메타 | C10 linker aggregate |
| 권한 룰 | C11 visibility 메서드 |

### 학습 추천 순서

1. **§0 anemic vs rich** — 가장 중요한 mental model 전환.
2. **C1 Aggregate** + **C2 Value Object** — DDD 의 두 빌딩 블록.
3. **C3/C4 compact constructor + static factory** — 보장 + 의도.
4. **C5/C6 state machine + transition** — 도메인의 행동.
5. **C7/C8 exception + event** — 도메인 fact 표현.
6. **C9 repository** — 헥사고날 입문.
7. **C10/C11 linker + visibility** — DDD 의 미세함.

### 추가 참고
- Eric Evans, *Domain-Driven Design* (2003) — bible.
- Vaughn Vernon, *Implementing DDD* (2013) — 실전.
- Martin Fowler, [Anemic Domain Model](https://martinfowler.com/bliki/AnemicDomainModel.html).
- Joshua Bloch, *Effective Java* — Item 1 (factory), Item 17 (immutability), Item 50 (defensive copy), Item 70 (exceptions).
