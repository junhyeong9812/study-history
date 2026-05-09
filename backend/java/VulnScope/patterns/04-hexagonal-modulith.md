# D. 헥사고날 + Modulith

> 이 문서가 다루는 것: 도메인이 인프라/프레임워크에 의존하지 않게 하는 아키텍처 + Spring Modulith 의 모듈 경계 강제. 8개 패턴.
> 전제: C 카테고리 (DDD) 의 Aggregate / Repository port 이해.
> 보충: `study/modulith-cross-module-dependency.md` (cross-module 의존 사상).

---

## §0. 헥사고날 아키텍처가 무엇인가?

### 0.1 layered architecture 의 한계

전통적 layered (Presentation / Service / DAO / DB):
```
Controller → Service → DAO → DB
```

**문제**:
- Service 가 DAO (`SqlSessionFactory`, `EntityManager`) 직접 의존.
- 도메인 객체가 ORM 어노테이션 (`@Entity`, `@Column`) 으로 오염.
- DB 교체 (MySQL → MongoDB) 면 도메인 코드도 같이 변경.
- 테스트 시 DB 띄워야 (단위 테스트도).

### 0.2 헥사고날 (Ports & Adapters)

Alistair Cockburn (2005) 의 제안:

```
   ┌─────────────────────────────────┐
   │           Application           │
   │  ┌───────────────────────────┐  │
   │  │       Domain (core)        │  │
   │  │   - Aggregate              │  │
   │  │   - Domain Logic           │  │
   │  │                            │  │
   │  │  inbound port ←──┐  ┌──→ outbound port
   │  └──────────────────┼──┼─────┘  │
   │                     │  │        │
   │  inbound adapter ←──┘  └──→ outbound adapter
   │  (Controller)              (Repository impl, External API client)
   └─────────────────────────────────┘
```

**핵심 원칙**:
1. **도메인은 외부 모름** — Spring/JPA/HTTP 어노테이션 0.
2. **port = interface** — 도메인이 외부와 통신하는 추상화.
3. **adapter = 구현** — port 의 구체 구현 (인프라 layer).
4. **의존 방향**: 모든 화살표가 도메인 가리킴 (역전).

### 0.3 VulnScope 의 layering

각 도메인 모듈 (target/scan/finding/upload/evidence/profile/scanengine):
```
presentation/    ← inbound adapter (Controller, Request/Response)
application/
  api/           ← inbound port (Command/Query interface)
  dto/           ← input/output DTO
  service/       ← orchestration (도메인 + repository 조립)
domain/          ← Aggregate + Value Object + Repository port + 예외 + 이벤트
infrastructure/  ← outbound adapter (InMemoryRepository, ObjectStorage 구현 등)
```

ArchUnit 룰이 강제 (J 카테고리 참조).

---

## D1. Hexagonal layering — 의존 방향 단방향

### 무엇인가
도메인 → application → infrastructure / presentation 순서. 도메인이 가장 안. 다른 layer 가 도메인 의존, 도메인은 다른 layer 의존 X.

### 어디서 쓰나
모든 도메인 모듈. 검증은 `back/src/test/java/com/vulnscope/architecture/ArchitectureTest.java`:
```java
@ArchTest
static final ArchRule domain_must_not_depend_on_outside =
    noClasses().that().resideInAPackage("..domain..")
        .should().dependOnClassesThat().resideInAnyPackage(
            "..application..",
            "..presentation..",
            "..infrastructure..",
            "org.springframework.boot..",
            "org.springframework.web..",
            "org.springframework.context..",
            "org.springframework.stereotype..",
            "com.fasterxml.jackson.."
        );
```

### 검증 결과
- `Scan` (`scan/domain/Scan.java`) 의 import 검사:
  ```java
  import com.vulnscope.shared.kernel.OrgId;       // ✅ shared 만
  import java.time.Clock;                          // ✅ JDK
  ```
  Spring 어노테이션 0. JPA 어노테이션 0. infrastructure 의존 0.

### 왜 단방향?

**양방향 의존 시**:
- service 가 DAO impl 의존 → DB 교체 시 service 수정.
- domain 이 Spring 의존 → Spring 없는 환경에서 도메인 못 씀 (테스트, 다른 컨테이너).

**단방향 (안 → 밖)**:
- 도메인 = 비즈니스 룰만. 외부 변경 무영향.
- 외부 = 도메인 호출. 도메인 변경 시만 외부 영향 받음.

### 핵심 아이디어
**의존 역전 (Dependency Inversion)** — 상위 layer 가 하위 layer 의 abstraction 의존.

도메인 = "상위" (비즈니스 의미). 인프라 = "하위" (기술 디테일). 일반적으로 service → DB 처럼 "위에서 아래로" 의존하는데, 헥사고날은 거꾸로 — 도메인이 abstraction 정의, 인프라가 구현 (구현이 abstraction 의존).

### 전문 용어 사전
- **DIP (Dependency Inversion Principle)**: SOLID 의 D. 추상화 의존.
- **inversion of control (IoC)**: control flow 가 framework 에서 호출. Spring DI 가 표준 예.
- **stable abstraction**: 도메인 interface 가 자주 안 바뀜. 인프라가 자주 바뀜. stable 한 쪽에 의존.

### 함정
- 도메인이 무심코 Spring/Jackson 의존 → ArchUnit 안 잡으면 silent.
- "도메인 layer 에는 어노테이션 0" 컨벤션 강제 필요.

### 대안
- Layered (전통) — 빠른 prototype 에 OK. 장기 유지보수 약함.
- Onion architecture (Jeffrey Palermo) — 헥사고날의 변형. 본질 동일.
- Clean Architecture (Uncle Bob) — 더 일반화된 형태.

### 출처
- Alistair Cockburn, *Hexagonal Architecture* (2005).
- Robert C. Martin, *Clean Architecture* (2017).

---

## D2. Port-Adapter (`ObjectStorage` + `LocalFileObjectStorage`)

### 무엇인가
도메인 layer 에 `interface` (port), 인프라 layer 에 구현 (adapter).

### 어디서 쓰나
```java
// upload/domain/ObjectStorage.java (port)
public interface ObjectStorage {
    void put(String key, InputStream content) throws IOException;
    InputStream get(String key) throws IOException;
    boolean exists(String key);
    void delete(String key) throws IOException;
}
```

```java
// upload/infrastructure/storage/LocalFileObjectStorage.java (adapter)
@Component
public class LocalFileObjectStorage implements ObjectStorage {
    private final Path root;

    public LocalFileObjectStorage(@Value("${app.upload.root:./data}") String root) {
        this.root = Path.of(root).toAbsolutePath().normalize();
    }

    @Override
    public void put(String key, InputStream content) throws IOException {
        // ... actual file write
    }
    // ...
}
```

### 사용
```java
// StoreUploadService — port 의존, 구현 모름
@Service
public class StoreUploadService {
    private final ObjectStorage storage;     // ← interface

    public StoreUploadService(ObjectStorage storage, ...) {
        this.storage = storage;              // Spring DI 가 LocalFile 주입
    }

    public Upload store(UploadInput input) {
        storage.put(key, content);           // 어디 저장하는지 모름
    }
}
```

### 왜 필요한가

**v0.1 → v0.2 시나리오**:
- v0.1: LocalFile (파일 시스템).
- v0.2: S3 (클라우드 스토리지).

**naive (도메인이 구현 직접 의존)**:
```java
@Service
public class StoreUploadService {
    private final LocalFileObjectStorage storage;    // ❌
    // ...
}
```
→ S3 도입 시 service 코드 변경. 모든 호출 site.

**Port-Adapter**:
```java
// 새 adapter 추가
@Component
public class S3ObjectStorage implements ObjectStorage {
    private final S3Client client;
    @Override public void put(...) { client.putObject(...); }
    // ...
}

// LocalFileObjectStorage 비활성화 (또는 profile 분리)
// service 코드 변경 0.
```

### 핵심 아이디어
**도메인이 외부 시스템을 interface 로 추상화**. 구현 교체 가능. 테스트에서 in-memory mock 가능.

### 전문 용어 사전
- **port**: interface. 도메인이 외부와 통신하는 contract.
- **adapter**: port 의 구현. 인프라 디테일.
- **outbound port**: 도메인이 외부 호출 (`Repository`, `ObjectStorage`).
- **inbound port**: 외부가 도메인 호출 (`ScanCommand`, `ScanQuery`). controller 가 이걸 호출.

### 함정
- port interface 가 외부 시스템 디테일 노출 (예: `S3PutObjectResponse`) → 추상화 깨짐. port 는 도메인 어휘로.
- 구현 1개만 있으면 interface 무의미? — 테스트 mock + 미래 교체 옵션.

### 대안
- 직접 구현 의존 — 단순 prototype.
- Spring Data JPA `JpaRepository` — interface 만 작성, Spring 자동 구현. 본질 같음.

### 출처
- Hexagonal Architecture: Cockburn 원문.
- Adapter pattern: GoF Design Patterns.

---

## D3. CQRS (`*Command` + `*Query` interface 분리)

### 무엇인가
쓰기 (Command) 와 읽기 (Query) 를 별도 interface 로 분리.

### 어디서 쓰나
모든 도메인 모듈의 `application/api`:
- `ScanCommand` + `ScanQuery`
- `TargetCommand` + `TargetQuery`
- `FindingCommand` + `FindingQuery`
- `UploadCommand` + `UploadQuery`
- `EvidenceCommand` + `EvidenceQuery`
- `ProfileQuery` (read-only)

```java
// scan/application/api/ScanCommand.java
public interface ScanCommand {
    Scan trigger(OrgId orgId, TargetId targetId, ProfileId profileId);
    void markRunning(ScanId id);
    void markCompleted(ScanId id, ScanSummary summary);
    void markFailed(ScanId id, String reason);
}

// scan/application/api/ScanQuery.java
public interface ScanQuery {
    Optional<Scan> findById(ScanId id);
    List<Scan> findByOrg(OrgId orgId, int limit);
    Page<Scan> list(OrgId orgId, ScanFilter filter, PageRequest pageRequest);
}
```

### Controller 가 둘 다 의존
```java
@RestController
public class ScanController {
    private final ScanCommand scanCommand;
    private final ScanQuery scanQuery;

    public ScanController(ScanCommand c, ScanQuery q) {
        this.scanCommand = c;
        this.scanQuery = q;
    }

    @PostMapping
    public ... trigger(...) { Scan s = scanCommand.trigger(...); }    // 쓰기

    @GetMapping("/{id}")
    public ... get(...) { return scanQuery.findById(...); }            // 읽기
}
```

### 왜 분리?

**naive (단일 ScanService)**:
```java
public interface ScanService {
    Scan trigger(...);                  // 쓰기
    void markRunning(...);              // 쓰기
    Optional<Scan> findById(...);       // 읽기
    List<Scan> list(...);                // 읽기
}
```
**문제**:
- read-only 가 필요한 컴포넌트도 write 메서드 보임. 의존성 명확 X.
- read 와 write 가 다른 신뢰성 모델 (write 는 정합성, read 는 eventually consistent OK) 인데 같이 묶임.

**CQRS**:
- 의도 명확: "이 컨트롤러는 ScanQuery 만 필요" → ScanCommand 의존 X.
- 권한/신뢰성 분리 가능 (예: Query 만 캐싱).
- 단위 테스트가 mock 작음.

### 핵심 아이디어
**Command Query Separation** (Bertrand Meyer, 1980년대) — 메서드는 "값 변경" 하거나 "값 반환" 둘 중 하나만, 둘 다 X.
**CQRS** (Greg Young) 는 이걸 interface 레벨로 확장.

### CQRS 의 strong vs weak

**Strong CQRS**: 별도 read DB + write DB + event sync. 매우 복잡.
**Weak CQRS** (VulnScope): 같은 DB, interface 만 분리. 의도 명시 + 약간의 격리.

VulnScope 는 weak. 큰 시스템 (수십만 RPS) 에선 strong 도 검토.

### 핵심 아이디어
**의도 명시 + 미래 분리 옵션**.

### 전문 용어 사전
- **CQS** (Command Query Separation): 메서드 레벨 분리. Bertrand Meyer.
- **CQRS** (Command Query Responsibility Segregation): interface/모델 레벨 분리. Greg Young.
- **eventual consistency**: read 와 write DB 간 sync 지연 허용. strong CQRS 에서 일반적.

### 함정
- trigger 가 `Scan` 반환 (도메인 객체) — 읽기 의미도 있음. 엄격 CQRS 면 void + 별도 query 필요. VulnScope 는 실용 (location header 발급에 id 필요).
- 두 interface 의 구현이 같은 service 일 수 있음 — facade pattern 으로 합성 (D4).

### 대안
- 단일 service interface — 단순. read-only 컴포넌트 의존 명확 X.
- Strong CQRS + ES (Event Sourcing) — 큰 시스템.

### 출처
- Bertrand Meyer, *Object-Oriented Software Construction* (1988) — CQS.
- Greg Young, *CQRS Documents* (2010+).

---

## D4. Facade (CommandService 가 여러 service 묶음)

### 무엇인가
하나의 interface 가 여러 내부 service 를 합성. 외부는 단일 interface 만 봄.

### 어디서 쓰나
```java
// ScanCommandService — package-private facade
@Service
class ScanCommandService implements ScanCommand {

    private final TriggerScanService triggerService;
    private final ScanLifecycleService lifecycle;

    ScanCommandService(TriggerScanService trigger, ScanLifecycleService lifecycle) {
        this.triggerService = trigger;
        this.lifecycle = lifecycle;
    }

    @Override
    public Scan trigger(OrgId orgId, TargetId targetId, ProfileId profileId) {
        return triggerService.trigger(orgId, targetId, profileId);    // 위임
    }

    @Override
    public void markRunning(ScanId id) {
        lifecycle.markRunning(id);
    }

    @Override
    public void markCompleted(ScanId id, ScanSummary summary) {
        lifecycle.markCompleted(id, summary);
    }

    @Override
    public void markFailed(ScanId id, String reason) {
        lifecycle.markFailed(id, reason);
    }
}
```

### 왜 필요한가
- `TriggerScanService`: 트리거 + cross-module 검증 + 이벤트 발행. 무거움.
- `ScanLifecycleService`: 단순 상태 전환. 가벼움.
- 두 service 가 책임 다름 → 분리 권장.
- 하지만 **외부는 단일 interface (`ScanCommand`)** 만 봄.

→ Facade 가 두 service 를 합성하여 단일 interface 구현.

### 핵심 아이디어
**책임 분리 (내부) + 단순한 외부 표면**. GoF Facade pattern.

### 전문 용어 사전
- **Facade pattern** (GoF): 복잡한 subsystem 을 단순 interface 로 wrap.
- **package-private** (Java): no modifier. 같은 package 내부만 접근. facade 구현 자체는 외부 안 봄.

### 함정
- facade 가 단순 위임만 하면 무의미? — interface 통한 외부 노출이 필요하면 의미.
- facade 가 비즈니스 로직 추가 X. 위임만. 비즈니스는 위임받는 service 가.

### 대안
- 단일 큰 service — 책임 분리 X.
- 외부가 두 service 직접 의존 — interface 표면 폭증.

### 출처
- GoF Design Patterns.
- 대규모 Spring 코드의 흔한 패턴.

---

## D5. Modulith `@NamedInterface` (출입구 명시)

### 무엇인가
Spring Modulith 의 어노테이션. "이 패키지는 다른 모듈이 import 가능한 출입구" 명시.

### 어디서 쓰나
모든 도메인 모듈의 다음 패키지:
```java
// scan/application/api/package-info.java
@org.springframework.modulith.NamedInterface("api")
package com.vulnscope.scan.application.api;

// scan/application/dto/package-info.java
@org.springframework.modulith.NamedInterface("dto")
package com.vulnscope.scan.application.dto;

// scan/domain/package-info.java
@org.springframework.modulith.NamedInterface("domain")
package com.vulnscope.scan.domain;

// scan/domain/event/package-info.java
@org.springframework.modulith.NamedInterface("events")
package com.vulnscope.scan.domain.event;
```

### 왜 필요한가

**Modulith 의 default**: 모든 모듈이 internal — 다른 모듈 import 차단.

**`@NamedInterface`**: "이 패키지만 외부 출입구" 라고 화이트리스트.

**효과**:
- ✅ `import com.vulnscope.scan.application.api.ScanCommand;` (다른 모듈에서) — OK.
- ❌ `import com.vulnscope.scan.application.service.TriggerScanService;` — 차단 (NamedInterface 아님).
- ❌ `import com.vulnscope.scan.infrastructure.persistence.InMemoryScanRepository;` — 차단.

→ **internal 직접 import 금지**. 강제 검증은 `ApplicationModules.verify()`.

### 핵심 아이디어
**모듈러 모놀리스 = 단일 deployable + 모듈 경계**. 마이크로서비스의 디렉토리 분리 흉내.

### Modulith 가 보장하는 것
1. **internal 침범 차단**: 다른 모듈의 service/infrastructure 직접 import X.
2. **순환 의존 차단**: A → B → A 형태 금지.
3. **명시적 의존 (옵션)**: `@ApplicationModule(allowedDependencies = ...)` 으로 모듈별 허용 의존 명시.

### 전문 용어 사전
- **Spring Modulith**: Spring 의 모듈러 모놀리스 지원 라이브러리.
- **named interface**: 모듈의 공식 출입구.
- **internal package**: NamedInterface 아닌 패키지. 같은 모듈만 접근.

### 함정
- `@NamedInterface` 어노테이션을 패키지 신규 생성 시 빠뜨리면 외부에서 import 안 됨. silent fail (compile error).
- 한 모듈의 internal 이 다른 모듈에 노출되면 결합 폭증 — 그래서 default 가 internal.

### 대안
- Java 9 모듈 시스템 (`module-info.java`) — 더 강력. 학습 곡선 큼. Modulith 가 더 가볍고 Spring 친화.
- 직접 ArchUnit 룰 — 가능. Modulith 가 표준화.

### 출처
- Spring Modulith: https://docs.spring.io/spring-modulith/reference/
- VulnScope ADR 0001 (Modulith 선택 근거).

---

## D6. Cross-module call via interface only

### 무엇인가
다른 모듈 호출 시 항상 interface (`*Query`, `*Command`) 만 의존. 도메인 객체/구현 직접 X.

### 어디서 쓰나
```java
// scan/application/service/TriggerScanService.java
import com.vulnscope.target.application.api.TargetQuery;       // ← interface
import com.vulnscope.profile.application.api.ProfileQuery;     // ← interface
// import com.vulnscope.target.infrastructure.persistence.InMemoryTargetRepository;  // ❌ X

@Service
public class TriggerScanService {
    private final TargetQuery targetQuery;       // interface 의존
    private final ProfileQuery profileQuery;
    // ...

    public Scan trigger(OrgId orgId, TargetId targetId, ProfileId profileId) {
        if (!targetQuery.isOwnedBy(targetId, orgId)) { ... }              // cross-module
        if (!profileQuery.existsAndVisibleTo(profileId, orgId)) { ... }   // cross-module
    }
}
```

### scanengine 의 8 의존
```java
// WorkerExecutor — 가장 많은 cross-module 의존
private final ScanCommand scanCommand;        // scan
private final TargetQuery targetQuery;        // target
private final TargetCommand targetCommand;    // target
private final FindingCommand findingCommand;  // finding
private final EvidenceCommand evidenceCommand; // evidence
private final Probe probe;                     // 자기 도메인
private final ScanModule headersModule;        // 자기 도메인
private final RealtimeChannel realtime;        // shared
```

### 왜 필요한가
**naive (구현 직접 import)**:
```java
import com.vulnscope.target.application.service.RegisterTargetService;
```
**문제**:
- service class 의 메서드 시그니처 변경 시 모든 호출자 영향.
- internal 침범 — Modulith 차단됨.

**interface 만**:
- `TargetQuery` 의 시그니처가 contract. 구현 변경 자유.
- Modulith 가 허용 (NamedInterface).

### 핵심 아이디어
**모듈 간 결합 = interface 한 줄**. 헥사고날의 inbound port 가 cross-module 출입구로도 작동.

자세한 사상은 `study/modulith-cross-module-dependency.md` 참조.

### 함정
- 다른 모듈의 도메인 객체 (`Scan`, `Target`) 를 controller 까지 전달 가능 — `domain/package-info.java` 에 `@NamedInterface("domain")` 부착해서 노출. VulnScope 의 정책.
- interface 를 자주 변경하면 cross-module 영향. interface 안정성 = 약속.

### 대안
- 모듈 간 도메인 이벤트만 (interface 호출 X) — 더 분리. 하지만 즉시 응답 필요한 케이스 (예: trigger 시 권한 검증) 어려움.
- 마이크로서비스 + REST 호출 — 진짜 분리. 운영 비용 폭증.

### 출처
`study/modulith-cross-module-dependency.md`.

---

## D7. CommandLineRunner 로 startup seed

### 무엇인가
Spring Boot 의 startup hook. 모든 빈 초기화 후 실행. 시드 데이터 적재.

### 어디서 쓰나
```java
// profile/infrastructure/persistence/SystemProfileSeed.java
@Component
class SystemProfileSeed implements CommandLineRunner {

    static final ProfileId QUICK_ID = ProfileId.of(UUID.fromString("00000000-0000-0000-0000-00000000000a"));
    static final ProfileId STANDARD_ID = ProfileId.of(UUID.fromString("00000000-0000-0000-0000-00000000000b"));
    static final ProfileId DEEP_ID = ProfileId.of(UUID.fromString("00000000-0000-0000-0000-00000000000c"));

    private final InMemoryProfileRepository repository;

    SystemProfileSeed(InMemoryProfileRepository repository) {
        this.repository = repository;
    }

    @Override
    public void run(String... args) {
        repository.save(Profile.system(
            QUICK_ID, "Quick", "Read-only checks · ~2m",
            1, 10, List.of("headers.missing")
        ));
        repository.save(Profile.system(STANDARD_ID, "Standard", ..., 3, 20, List.of(...)));
        repository.save(Profile.system(DEEP_ID, "Deep", ..., 5, 50, List.of(...)));
    }
}
```

### 왜 필요한가
v0.1 in-memory → 매 부팅 시 데이터 비어있음. system profile (Quick/Standard/Deep) 가 항상 있어야 ScanController 의 `DEFAULT_PROFILE` 가 동작.

### CommandLineRunner vs @PostConstruct

| | CommandLineRunner | @PostConstruct |
|---|---|---|
| 시점 | 모든 빈 초기화 후 (application context refresh 끝) | 자기 빈 초기화 직후 |
| 의존성 | 모든 빈 사용 가능 | 자기 의존 빈만 보장 |
| 인자 | command-line args | 없음 |

→ seed 처럼 다른 빈 (`InMemoryProfileRepository`) 의존하면 CommandLineRunner 가 안전.

### 핵심 아이디어
**"부팅 후 한 번 실행" hook**. data 시드 / health check / migration trigger.

### 전문 용어 사전
- **CommandLineRunner / ApplicationRunner**: Spring Boot startup hook. 후자는 `ApplicationArguments` 받음.
- **@PostConstruct**: JSR-250. 빈 자체 초기화. 다른 빈 사용 위험.

### 함정
- 여러 CLR 등록 시 실행 순서 미정 — `@Order` 로 명시.
- 예외 throw 시 application 부팅 실패. seed 실패는 부팅 실패가 맞음 (data 없으면 동작 X).

### 대안
- Spring Data SQL `data.sql` — DB 시드. v0.1 에 DB 없어 못 씀.
- Liquibase/Flyway — DB migration + seed. v0.2 후보.

### 출처
Spring Boot 의 표준 hook.

---

## D8. Sentinel UUID (DB 없는 시기 fixed reference)

### 무엇인가
의미 있는 fixed UUID. 코드 곳곳에서 hardcoded 매핑.

### 어디서 쓰나
```java
// shared/security/Credentials.java
public static final UUID FIXED_USER_ID = UUID.fromString("00000000-0000-0000-0000-000000000001");
public static final UUID FIXED_ORG_ID  = UUID.fromString("00000000-0000-0000-0000-000000000002");

// SystemProfileSeed
static final ProfileId QUICK_ID    = ProfileId.of(UUID.fromString("00000000-0000-0000-0000-00000000000a"));
static final ProfileId STANDARD_ID = ProfileId.of(UUID.fromString("00000000-0000-0000-0000-00000000000b"));
static final ProfileId DEEP_ID     = ProfileId.of(UUID.fromString("00000000-0000-0000-0000-00000000000c"));

// ScanController.DEFAULT_PROFILE — STANDARD_ID 와 일치
private static final ProfileId DEFAULT_PROFILE = ProfileId.of(
    UUID.fromString("00000000-0000-0000-0000-00000000000b")    // ← STANDARD_ID
);
```

### 왜 필요한가
**v0.1 in-memory** → 매 부팅 시 random UUID 생성하면 다른 곳에서 참조 깨짐.

예: ScanController 의 `DEFAULT_PROFILE` 이 random 이면 SystemProfileSeed 가 만든 STANDARD profile 과 불일치. UUID 안 맞으면 `existsAndVisibleTo` false → InvalidProfile 에러.

**Sentinel UUID** 로 고정하면:
- SystemProfileSeed 가 `STANDARD_ID` 로 시드.
- ScanController 가 `DEFAULT_PROFILE` 로 같은 UUID 참조.
- 일치 보장.

### 의미 있는 패턴
- `0000-...0001` = user
- `0000-...0002` = org
- `0000-...000a` = quick profile
- `0000-...000b` = standard profile
- `0000-...000c` = deep profile

→ "0001 = 사용자, 0002 = 조직, 000a/b/c = profile" 같은 약속.

### 핵심 아이디어
**기억 가능한 sentinel** 로 의미 명시. `0000-...` 패턴은 random 안 만들어짐 (확률 거의 0) → "이건 시드 데이터" 식별.

### 전문 용어 사전
- **sentinel value**: 특수 의미를 가진 값. 예: `-1` 이 "없음" 표시.
- **fixture data**: 테스트/시드용 고정 데이터.

### 함정
- v0.2 DB 도입 시 random UUID 가 자연 — sentinel 패턴 의미 없어짐. lookup 으로 대체.
- production 에서 sentinel UUID 노출 시 정보 누출 가능 (예측 가능). v0.1 dev 한정 OK.

### 대안
- 매 부팅 random + lookup table — 빠른 구현엔 sentinel 이 더 단순.
- DB 영속 + 보존 — v0.2 의 정공법.

### 출처
- Sentinel value 패턴: 일반 프로그래밍 관용.
- VulnScope 의 v0.1 in-memory 한정 결정.

---

## §∞. 정리

### 패턴 매트릭스

| 문제 | 패턴 |
|---|---|
| 도메인 ↔ 인프라 분리 | D1 hexagonal layering |
| 외부 시스템 추상화 | D2 port-adapter |
| 읽기/쓰기 분리 | D3 CQRS interface |
| 여러 service 묶기 | D4 facade |
| 모듈 출입구 강제 | D5 NamedInterface |
| cross-module 호출 | D6 interface only |
| startup 시드 | D7 CommandLineRunner |
| in-memory fixed reference | D8 sentinel UUID |

### 학습 추천 순서

1. **§0 layered 의 한계** — 헥사고날 의 동기.
2. **D1 layering** + **D2 port-adapter** — 헥사고날 의 본질.
3. **D5 NamedInterface** — Modulith 의 핵심.
4. **D6 cross-module** — 모듈 간 의존의 실전.
5. **D3 CQRS** — interface 분리 패턴.
6. **D4 facade** — interface 합성.
7. **D7/D8** — startup + sentinel (실전 트릭).

### 추가 참고
- Alistair Cockburn, *Hexagonal Architecture* (2005).
- Robert C. Martin, *Clean Architecture* (2017).
- Spring Modulith reference: https://docs.spring.io/spring-modulith/reference/
- VulnScope ADR 0001 — Modulith vs microservices 결정.
- `study/modulith-cross-module-dependency.md` — D6 상세.
- `study/domain-events-and-outbox.md` — 이벤트 기반 cross-module 통신.
