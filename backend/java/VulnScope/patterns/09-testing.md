# I. 테스트 패턴

> 단위/통합/비동기/SSE 테스트의 11개 패턴.

---

## §0. 테스트의 mental model

**좋은 테스트의 특징** (Kent Beck):
- **fast** — 빠름 (단위 < 100ms).
- **isolated** — 다른 테스트 영향 X.
- **repeatable** — 매번 같은 결과.
- **self-validating** — assert 실패 자동.
- **timely** — 코드 직전/직후 작성.

**flaky test** (random fail) 가 나쁜 이유:
- "통과하면 OK, 실패하면 retry" — 신뢰 0.
- 점점 무시됨 → silent regression.

VulnScope 의 11개 패턴이 이 5 특성 보장.

---

## I1. `Clock.fixed` (시간 결정성)

### 무엇인가
실제 system time 대신 고정된 Clock 사용. 시간 의존 테스트가 매번 같은 결과.

### 코드
```java
private static final Clock T0 = Clock.fixed(
    Instant.parse("2026-04-30T00:00:00Z"),
    ZoneOffset.UTC
);

@Test
void request_creates_queued_scan_with_initial_state() {
    Scan scan = Scan.request(id, ORG, TARGET, PROFILE, T0);
    assertThat(scan.createdAt()).isEqualTo(T0.instant());    // 매번 정확히 같음
}
```

### 왜 필요한가

**naive (`Clock.systemUTC()`)**:
```java
Scan scan = Scan.request(..., Clock.systemUTC());
// 매 실행 시 다른 시각.
// assertThat(scan.createdAt()).isEqualTo(...) — 무엇과 비교?
```

→ flaky. `Instant.now()` 도 마찬가지.

**fixed Clock**:
- 매번 같은 instant.
- assert 가 정확히 가능.
- `T0.instant().plusSeconds(60)` 으로 "1분 후" 시뮬레이션.

### 핵심 아이디어
**시간을 명시 의존 (Clock 주입) → 테스트에서 통제 가능**. dependency injection 의 한 적용.

### 함정
- production 코드도 `Clock.systemUTC()` 직접 호출 X — 모든 시간 의존이 Clock 인자 받음.
- TimeZone 명시 (`ZoneOffset.UTC`) — server timezone 무관.

### 대안
- Mockito `mockStatic(Instant.class).when(...)` — 가능하지만 복잡.
- 직접 timestamp 인자 — Clock 추상화가 더 일반적.

---

## I2. AssertJ fluent + `assertThatThrownBy`

### 무엇인가
AssertJ 의 fluent matcher. 가독성 + 풍부한 검증.

### 코드
```java
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

assertThat(scan.status()).isEqualTo(ScanStatus.QUEUED);
assertThat(events).hasSize(1);
assertThat(events.get(0).getClass().getSimpleName()).isEqualTo("ScanRequested");

// 예외 검증
assertThatThrownBy(() -> done.markRunning(T0))
    .isInstanceOf(ScanCannotTransition.class)
    .hasMessageContaining("from DONE to RUNNING");
```

### 왜 AssertJ?

**JUnit 의 `assertEquals`**:
```java
assertEquals(ScanStatus.QUEUED, scan.status());    // 순서 헷갈림
assertEquals(expected, actual);                     // expected 가 먼저인지 actual 이 먼저인지
```

**AssertJ fluent**:
```java
assertThat(scan.status()).isEqualTo(ScanStatus.QUEUED);
// "actual is equal to expected" — 자연 영어
```

장점:
- 자연 문장 순서.
- chain (`.isNotNull().hasSize(3).contains(x)`).
- 풍부한 매처 (`containsExactly`, `extracting`, `filteredOn` 등).
- 예외 검증 (`assertThatThrownBy`).

### 핵심 아이디어
**테스트 코드도 production 코드 — 가독성 중요**. AssertJ 가 더 읽기 쉬움.

---

## I3. Anonymous inner class stub (Mockito 회피)

### 무엇인가
단순 boolean 인터페이스는 anonymous class 가 Mockito 보다 명확.

### 코드
```java
// TriggerScanServiceTest
private TargetQuery stubTargetQuery() {
    return new TargetQuery() {
        @Override
        public Optional<Target> findById(TargetId id) { return Optional.empty(); }
        @Override
        public List<Target> recentByOrg(OrgId orgId, int limit) { return List.of(); }
        @Override
        public boolean isOwnedBy(TargetId id, OrgId orgId) {
            return id.equals(OWNED_TARGET) && orgId.equals(ORG);    // 명시 logic
        }
    };
}
```

### 비교

**Mockito**:
```java
TargetQuery targetQuery = mock(TargetQuery.class);
when(targetQuery.isOwnedBy(OWNED_TARGET, ORG)).thenReturn(true);
when(targetQuery.isOwnedBy(any(), any())).thenReturn(false);
```
- 의존: Mockito 라이브러리.
- 의도: when/then 으로 표현. 약간 마법 같음.

**Anonymous stub**:
- 의존 0.
- 명시 logic — `id.equals(OWNED_TARGET) && orgId.equals(ORG)`. Java 코드.
- 인터페이스 모든 메서드 구현 (compile time 강제).

### 핵심 아이디어
**단순 인터페이스 + 명확한 logic 은 stub > mock**. mock 은 복잡한 verify 필요할 때.

### 함정
- 인터페이스 메서드 많으면 anonymous class boilerplate.
- 호출 횟수 검증 (`verify(mock, times(2))`) 필요하면 Mockito 가 편함.

---

## I4. `@TempDir` (filesystem 테스트 격리)

### 무엇인가
JUnit 5 의 임시 디렉토리. 자동 생성 + 종료 후 정리.

### 코드
```java
class LocalFileObjectStorageTest {

    @TempDir Path tempDir;        // ← 자동 주입

    @Test
    void put_creates_file_under_root() throws IOException {
        LocalFileObjectStorage storage = new LocalFileObjectStorage(tempDir.toString());
        storage.put("uploads/abc.bin", new ByteArrayInputStream("hello".getBytes()));

        Path expected = tempDir.resolve("uploads/abc.bin");
        assertThat(Files.exists(expected)).isTrue();
    }
}
```

### 왜 필요한가
- 테스트마다 다른 디렉토리 → 격리.
- 종료 시 자동 cleanup → leak 없음.
- 동시 실행 안전.

### 핵심 아이디어
**filesystem 테스트도 격리 가능**. JUnit 의 헬퍼.

---

## I5. `@SpringBootTest + @AutoConfigureMockMvc` (통합 테스트)

### 무엇인가
전체 Spring 컨텍스트 + MockMvc 자동 주입. 진짜 네트워크 안 탐.

### 코드
```java
@SpringBootTest
@AutoConfigureMockMvc
class ScanControllerTest {

    @Autowired private MockMvc mvc;
    @Autowired private RecordFindingService recordService;    // service 직접 주입도 가능

    @Test
    void trigger_scan_returns_201() throws Exception {
        mvc.perform(post("/scans").cookie(session).contentType(...).content("..."))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.status").value("QUEUED"));
    }
}
```

### 왜 필요한가
- 모든 layer (Controller + Service + Repository + Filter) 통합 검증.
- 실제 Spring DI + Filter chain 동작.
- 가짜 HTTP request — 빠름 (네트워크 X).

### 함정
- `@SpringBootTest` 는 무거움 (전체 context). 단위 테스트엔 `@WebMvcTest` 가 더 가벼움.
- VulnScope 는 in-memory repository 라 전체 context 띄워도 OK.

---

## I6. Cross-module fixture (controller test 가 다른 endpoint 호출)

### 무엇인가
ScanControllerTest 가 fixture 만들 때 target endpoint 호출.

### 코드
```java
private String registerTarget(String url) throws Exception {
    MvcResult res = mvc.perform(post("/targets")
            .cookie(session)
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                {"type":"WEB","value":"%s"}
                """.formatted(url)))
        .andExpect(status().isCreated())
        .andReturn();
    String location = res.getResponse().getHeader("Location");
    return location.substring("/targets/".length());
}

// 사용
@Test
void trigger_scan_with_owned_target_returns_201_queued() throws Exception {
    String targetId = registerTarget("https://scan-trigger.example.com");
    mvc.perform(post("/scans").content(...).cookie(session))
        .andExpect(status().isCreated());
}
```

### 왜?
- 진짜 통합 — target 등록 + scan trigger 두 흐름이 실제로 동작하는지 검증.
- 데이터 일관성 (target 의 orgId == 인증 사용자 orgId) 자동 보장.

### 핵심 아이디어
**통합 테스트의 진가 = 모듈 간 흐름 검증**. fixture 도 endpoint 통해서 만들면 진짜 e2e.

### 함정
- target endpoint 가 깨지면 scan 테스트도 같이 깨짐 — 진짜 의존성. 의도된 trade-off.

---

## I7. `TestAuthSupport.loginAsAdmin` (테스트 헬퍼)

### 무엇인가
모든 controller 테스트가 `@BeforeEach` 에서 호출. cookie 반환.

### 코드
```java
@BeforeEach
void login() throws Exception {
    session = TestAuthSupport.loginAsAdmin(mvc);
}

// 사용
mvc.perform(post("/scans").cookie(session)...).andExpect(...);
```

### 왜?
- 모든 테스트에 인증 fixture — boilerplate 제거.
- 한 곳에서 admin login 로직 관리.

---

## I8. Awaitility (비동기 polling)

### 무엇인가
비동기 결과를 polling 으로 검증. 함수형 표현.

### 코드
```java
import static org.awaitility.Awaitility.await;

@Test
void worker_eventually_completes_or_fails_after_trigger() {
    var scan = triggerScanService.trigger(org, target.id(), STANDARD_PROFILE);

    await().atMost(Duration.ofSeconds(30)).until(() -> {
        ScanStatus status = scanQuery.findById(scan.id()).map(Scan::status).orElse(null);
        return status == ScanStatus.DONE || status == ScanStatus.FAILED;
    });

    Scan finalScan = scanQuery.findById(scan.id()).orElseThrow();
    assertThat(finalScan.status()).isIn(ScanStatus.DONE, ScanStatus.FAILED);
}
```

### 왜 Thread.sleep 안 씀?

**naive**:
```java
trigger(...);
Thread.sleep(5000);    // ❌ 매번 5초 대기
assertThat(scan.status()).isIn(DONE, FAILED);
```
- 5초 < 실제 → flaky (테스트 실패).
- 5초 > 실제 → 느린 테스트.

**Awaitility**:
- polling — 결과 도달 즉시 진행.
- atMost timeout — 무한 대기 X.
- 표현 functional.

### 핵심 아이디어
**비동기 = "언제 완료될지 모르지만, 일정 시간 안에" 패턴**. polling + timeout 조합.

### 전문 용어 사전
- **eventually consistent**: 즉시는 X, 결국 일관 도달.
- **polling**: 주기적 확인.

---

## I9. MockEventSource (frontend SSE hook 테스트)

### 무엇인가
브라우저 `EventSource` 를 mock 으로 교체. hook 의 모든 동작 격리 검증.

### 코드
```typescript
class MockEventSource {
    static instances: MockEventSource[] = [];
    url: string;
    onopen: (() => void) | null = null;
    onerror: (() => void) | null = null;
    closed = false;
    private listeners: Map<string, Listener[]> = new Map();

    constructor(url: string, init?: { withCredentials?: boolean }) {
        this.url = url;
        MockEventSource.instances.push(this);
    }

    addEventListener(type: string, listener: Listener) {
        const arr = this.listeners.get(type) ?? [];
        arr.push(listener);
        this.listeners.set(type, arr);
    }

    emit(type: string, data: string, lastEventId = "") {
        const event = new MessageEvent(type, { data, lastEventId });
        this.listeners.get(type)?.forEach(l => l(event));
    }

    close() { this.closed = true; }
}

beforeEach(() => {
    MockEventSource.instances = [];
    (globalThis as any).EventSource = MockEventSource;
});

it("dedupes events with seq <= lastSeq on reconnect", async () => {
    const { result } = renderHook(() => useScanStream("s1"));
    const es = MockEventSource.instances[0];

    act(() => {
        es.emit("finding", JSON.stringify({ title: "A" }), "1");
        es.emit("finding", JSON.stringify({ title: "B" }), "2");
        es.emit("finding", JSON.stringify({ title: "B-dup" }), "2");    // 중복
        es.emit("finding", JSON.stringify({ title: "C" }), "3");
    });

    await waitFor(() => expect(result.current.events).toHaveLength(3));    // 4 X
});
```

### 왜?
- 진짜 SSE 서버 안 띄움 — fast.
- hook 의 dedupe / lifecycle 정확히 검증.
- `globalThis.EventSource` 교체 — Node 환경 (jest jsdom) 에 EventSource 자체 없음.

### 핵심 아이디어
**globalThis 교체로 브라우저 API stub**. test 에서만 적용.

---

## I10. RHF + Zod schema 단위 테스트

### 무엇인가
폼 검증 룰의 모든 케이스 (success/fail/path/message).

### 코드
```typescript
import { LoginSchema } from "./schemas";

describe("LoginSchema", () => {
    it("accepts valid email and password", () => {
        const result = LoginSchema.safeParse({ email: "admin", password: "admin" });
        expect(result.success).toBe(true);
    });

    it("rejects empty email", () => {
        const result = LoginSchema.safeParse({ email: "", password: "admin" });
        expect(result.success).toBe(false);
        if (!result.success) {
            expect(result.error.issues[0].path).toEqual(["email"]);
            expect(result.error.issues[0].message).toBe("이메일을 입력하세요");
        }
    });
});
```

### 왜?
- 검증 룰이 폼 동작의 핵심.
- `.safeParse` 는 throw 안 함 → 결과 객체 검증 가능.

### `.safeParse` vs `.parse`
- `.parse(input)` — 실패 시 throw.
- `.safeParse(input)` — `{ success: true, data } | { success: false, error }` 반환. 테스트 친화.

---

## I11. RecordingPublisher pattern (제거됨, 학습용)

### 무엇인가
이전 ApplicationEventPublisher mock — 발행 검증.

### 제거된 코드 (예전엔 있었음)
```typescript
private static final class RecordingPublisher implements ApplicationEventPublisher {
    final List<Object> events = new ArrayList<>();
    @Override public void publishEvent(Object event) { events.add(event); }
}

@Test
void register_publishes_event() {
    service.register(...);
    assertThat(events.events).hasSize(1);
    assertThat(events.events.get(0).getClass().getSimpleName()).isEqualTo("TargetRegistered");
}
```

### 왜 제거?
4개 이벤트 (ScanCompleted/ScanFailed/FindingRecorded/TargetRegistered) listener 0 → 발행 자체 무효 → 제거 → 발행 검증 테스트도 제거.

상세는 `study/domain-events-and-outbox.md`.

### 학습 가치
**이벤트 발행 검증 패턴 자체는 유효** — VulnScope 는 listener 없는 이벤트 제거 결정으로 같이 사라짐. 다른 프로젝트에서는 활용 가능.

---

## §∞. 정리

| 문제 | 패턴 |
|---|---|
| 시간 결정성 | I1 Clock.fixed |
| 가독성 + 매처 | I2 AssertJ + assertThatThrownBy |
| Mockito 회피 | I3 anonymous stub |
| filesystem 격리 | I4 @TempDir |
| 통합 테스트 | I5 @SpringBootTest + MockMvc |
| 모듈 간 흐름 | I6 cross-module fixture |
| 인증 boilerplate | I7 TestAuthSupport.loginAsAdmin |
| 비동기 검증 | I8 Awaitility |
| frontend SSE | I9 MockEventSource (globalThis 교체) |
| Zod schema | I10 .safeParse + issues |
| 이벤트 발행 검증 | I11 RecordingPublisher (제거됨) |
