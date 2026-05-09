# F. 인프라 구현 패턴

> 실제 구현 시 마주치는 디테일들 — storage key, dedupe, ETag, SSE wire, Comparator combinator. 11개 패턴.
> 전제: B (데이터 구조), E (인프라 추상화) 이해.

---

## F1. Storage key 설계 (org/날짜 파티션 + UUID 파일명)

### 무엇인가
파일/객체 저장 시 키 (path) 를 의미 있게 구성. tenant 격리 + 파일시스템 성능.

### 코드
```java
// StoreUploadService.computeStorageKey
private String computeStorageKey(OrgId orgId, UploadId id) {
    LocalDate today = LocalDate.ofInstant(clock.instant(), ZoneOffset.UTC);
    return String.format("uploads/%s/%04d/%02d/%02d/%s.bin",
        orgId.value(),                   // ① org 분리
        today.getYear(),
        today.getMonthValue(),
        today.getDayOfMonth(),           // ② 날짜 파티션
        id.value());                     // ③ UUID 파일명
}
// → "uploads/<org-uuid>/2026/05/02/<file-uuid>.bin"
```

### 왜 이 구조?

**①  org 분리** — `uploads/<org>/...`:
- Multi-tenant 격리. 다른 org 의 파일이 한 디렉토리에 안 섞임.
- 백업/삭제 시 org 단위 작업 쉬움 (`rm -rf uploads/<orgId>/`).

**② 날짜 파티션** — `2026/05/02/`:
- **파일시스템 성능**: 한 디렉토리에 수십만 파일 누적 시 listing/lookup 느려짐 (특히 ext4 의 dir_index 한계).
- 날짜로 분산 → 한 디렉토리 = 하루치 파일 → 일정 size.

**③ UUID 파일명**:
- 충돌 0 (random UUID).
- 확장자 `.bin` 일률 — 콘텐츠 타입은 메타로.

### 핵심 아이디어
**S3 hot partition 회피 + tenant scoping**. AWS S3 도 같은 패턴 권장 (key prefix 분산).

### 함정
- timezone 일관성 — UTC 사용 (서버 timezone 영향 받지 X).
- 옛 날짜 파티션 정리 (cleanup) 정책 별도 필요.

### 대안
- hash 기반 분산 (`uploads/<sha256[0:2]>/...`) — Git/S3 표준. dedupe 시 일관 키.
- 단일 디렉토리 — 작은 시스템 OK, 큰 시스템 X.

---

## F2. SHA-256 dedupe (org-scoped)

### 무엇인가
같은 콘텐츠 (sha256 동일) 의 두 번째 upload 는 새로 저장 안 함. 기존 UploadId 반환.

### 코드
```java
// StoreUploadService.store
public Upload store(UploadInput input) {
    Sha256 sha = Sha256.ofBytes(input.content());
    Optional<Upload> existing = repository.findByOrgAndSha256(input.orgId(), sha);
    if (existing.isPresent()) {
        return existing.get();        // ← dedupe! 새 저장 X
    }
    // ... 신규 저장
}
```

### 왜 필요한가
- 같은 finding 의 evidence 가 여러 번 첨부 (재스캔 등) → 같은 raw HTTP 응답.
- 매번 새로 저장하면 디스크 낭비.

### Org-scoped 인 이유
- **다른 org 의 dedupe 안 함** — 같은 콘텐츠라도 격리.
- privacy: org A 가 어떤 파일 가졌는지 org B 가 추론 못 함.

### 핵심 아이디어
**content-addressable storage** + **tenant 격리**. Git 의 object 저장과 비슷하지만 tenant scope 추가.

### 함정
- sha256 collision (사실상 0) — 가정.
- byte 1개 다르면 다른 sha → dedupe 안 됨. semantic dedupe 와 다름.

### 대안
- 해시 안 하고 매번 저장 — 단순. 디스크 낭비.
- 전역 dedupe (org 무관) — privacy 위반.

---

## F3. ETag = sha256 (자연스러운 cache 키)

### 무엇인가
HTTP ETag 헤더 = sha256 해시. 클라이언트 cache + 304 Not Modified 응답.

### 코드
```java
// UploadController.download
return ResponseEntity.ok()
    .header(HttpHeaders.CONTENT_TYPE, ds.meta().contentType())
    .header(HttpHeaders.CONTENT_LENGTH, String.valueOf(ds.meta().sizeBytes()))
    .header(HttpHeaders.ETAG, "\"" + ds.meta().sha256() + "\"")
    .body(new InputStreamResource(ds.content()));
```

### 왜 sha256?
- 콘텐츠 식별자. 같은 콘텐츠 = 같은 ETag.
- 클라이언트가 `If-None-Match: "<sha256>"` 헤더 보내면 서버는 304 Not Modified 만 반환 (body 안 보냄). 대역폭 절약.

### 핵심 아이디어
**content hash 가 자연스러운 ETag**. 별도 ETag 생성 로직 불필요.

### 전문 용어 사전
- **ETag** (Entity Tag): HTTP cache 검증 헤더.
- **strong ETag**: 콘텐츠 byte 일치. `"<value>"`.
- **weak ETag**: semantic 일치 (byte 다를 수 있음). `W/"<value>"`. VulnScope 는 strong.

### 함정
- ETag 따옴표 (`"..."`) 필수 — 그렇지 않으면 invalid. Spring 자동 처리 X (직접 따옴표 추가).

---

## F4. base64url no-padding (URL/Cookie 안전)

### 무엇인가
일반 base64 (`+/=`) 대신 URL 안전 변형 (`-_`, padding 제거).

### 코드
```java
// TokenService
private static String base64Url(byte[] bytes) {
    return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
}
```

### 왜 필요한가
일반 base64:
- `+` → URL 에서 space 로 해석. 별도 encoding (`%2B`) 필요.
- `/` → URL path separator. 잘못 해석.
- `=` padding → URL query value 에서 충돌.

base64url:
- `+` → `-`, `/` → `_`, padding 제거 (또는 `%3D` 인코딩).
- URL/Cookie 그대로 사용 가능.

### 핵심 아이디어
**JWT 의 컨벤션** (RFC 7515 §2). 토큰을 URL 에 박아도 깨끗.

### 전문 용어 사전
- **base64url**: RFC 4648 §5. URL-safe alphabet.
- **padding**: base64 가 길이 4의 배수 만들기 위한 `=` 채움. URL 에 거추장.

---

## F5. SSE: Last-Event-ID resume + Store backfill + Bus subscribe

### 무엇인가
SSE 클라이언트가 끊겼다 재연결 시 Last-Event-ID 헤더로 끊긴 시점 알림. 서버가 그 이후 이벤트 backfill + 새 이벤트 실시간 전달.

### 코드
```java
// ScanStreamController.stream
@GetMapping(value = "/{id}/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter stream(@PathVariable UUID id,
                         @RequestHeader(name = "Last-Event-ID", required = false) Long lastEventId) {
    SseEmitter emitter = new SseEmitter(0L);
    ChannelKey channel = ChannelKey.of("scan/" + id);
    long since = lastEventId == null ? 0 : lastEventId;

    // ① 백필
    for (StoredEvent event : store.readSince(channel, since, BACKFILL_LIMIT)) {
        send(emitter, event);
    }

    // ② 구독
    RealtimeBus.Subscription subscription = bus.subscribe(channel, e -> send(emitter, e));

    emitter.onCompletion(subscription::close);
    emitter.onTimeout(subscription::close);
    emitter.onError(t -> subscription.close());

    return emitter;
}

private void send(SseEmitter emitter, StoredEvent event) {
    try {
        emitter.send(SseEmitter.event()
            .id(String.valueOf(event.seq()))    // ← Last-Event-ID 의 키
            .name(event.type())
            .data(event.payload()));
    } catch (IOException e) {
        emitter.completeWithError(e);
    }
}
```

### 흐름
1. 클라 첫 접속 → since=0 → 모든 이벤트 backfill (보통 0개).
2. 라이브 이벤트 → bus.publish → emitter.send.
3. 클라 끊김 (네트워크) → 자동 재연결 (브라우저 EventSource).
4. 재연결 요청 → `Last-Event-ID: <마지막 받은 seq>`.
5. 서버: store.readSince(채널, lastSeq, 1000) → 끊긴 동안 이벤트 backfill.
6. 라이브 구독 재개.

→ **이벤트 누락 0** (retention window 안에 있으면).

### 핵심 아이디어
**"읽은 위치" 를 클라가 들고 있고 서버가 read** — Kafka consumer offset 모델 차용.

### 함정
- backfill 양 너무 많으면 (수만 개) 응답 지연. limit 필수.
- retention window 밖이면 누락. 클라가 polling fallback (`findByScanSince`) 또는 새 scan trigger.

상세는 `study/sse-libraries.md` 참조.

---

## F6. SseEmitter 라이프사이클 콜백 (memory leak 방지)

### 무엇인가
SseEmitter 의 종료/타임아웃/오류 콜백 모두 등록 → 구독 자동 해제.

### 코드
```java
emitter.onCompletion(subscription::close);   // 정상 종료
emitter.onTimeout(subscription::close);       // 타임아웃
emitter.onError(t -> subscription.close());   // 에러
```

### 왜 3개 모두?
- **onCompletion**: 서버가 emitter.complete() 호출 또는 클라이언트 명시 close.
- **onTimeout**: emitter timeout (VulnScope 는 0 = 무한, 그래도 콜백 등록).
- **onError**: I/O 오류 (클라이언트 끊김 등).

→ **세 경로 중 하나라도 누락하면 leak**. Bus.subscribers Set 에 dead subscriber 누적.

### 핵심 아이디어
**resource lifecycle 의 모든 종료 경로 처리**. defensive programming.

### 함정
- 콜백 1개라도 빠뜨리면 silent leak — production 에서만 발견 (메모리 OOM).
- 콜백 안에서 추가 close 호출 시 idempotent 보장 (`set.remove` 가 idempotent).

---

## F7. Comparator combinator (정렬)

### 무엇인가
Java 8 Comparator 의 함수형 메서드 — `comparing`, `thenComparing`, `reversed`, `nullsLast`, `comparingLong`.

### 어디서 쓰나 (5종 패턴)

#### 단일 키 내림차순
```java
// InMemoryScanRepository
.sorted(Comparator.comparing(Scan::createdAt, Comparator.reverseOrder()))
```

#### primitive long (박싱 회피)
```java
// InMemoryFindingRepository
.sorted(Comparator.comparingLong(Finding::seq))
```

#### 자연 순서
```java
// InMemoryEvidenceRepository
.sorted(Comparator.comparing(Evidence::createdAt))     // asc
```

#### nullsLast + thenComparing
```java
// InMemoryTargetRepository
.sorted(Comparator
    .comparing(Target::lastUsedAt, Comparator.nullsLast(Comparator.reverseOrder()))
    .thenComparing(Target::createdAt, Comparator.reverseOrder()))
```

#### enum ordinal + 보조 키
```java
// InMemoryProfileRepository
.sorted(Comparator
    .comparing((Profile p) -> p.kind().ordinal())     // SYSTEM=0 먼저
    .thenComparing(Profile::name))                     // 같은 kind 내 알파벳
```

### 핵심 아이디어
**combinator 로 복잡한 정렬 조립** — 작은 building block 조합.

상세는 `study/comparator.md` 참조.

---

## F8. In-memory pagination (전체 fetch + sublist)

### 무엇인가
filter 적용 + slice + Page wrap. v0.1 한정. v0.2 DB 면 LIMIT/OFFSET push down.

### 코드
```java
// ScanQueryService.list
@Override
public Page<Scan> list(OrgId orgId, ScanFilter filter, PageRequest pageRequest) {
    List<Scan> all = repository.findAllByOrg(orgId).stream()
        .filter(filter::matches)
        .toList();
    long total = all.size();
    int from = Math.min(pageRequest.offset(), all.size());
    int to = Math.min(from + pageRequest.size(), all.size());
    List<Scan> slice = all.subList(from, to);
    return Page.of(slice, pageRequest.page(), pageRequest.size(), total);
}
```

### 왜 OK in-memory?
- 데이터 N 작음 (수십~수백 scan).
- 매 page 호출이 전체 fetch — 비효율이지만 작은 N 에 OK.

### v0.2 DB 전환
- 같은 interface (`Page<Scan> list(...)`).
- 구현만 SQL `LIMIT ? OFFSET ?` 으로 변경.
- application/service 코드 수정 0.

### 함정
- N 큰 환경에서 OOM 위험 — 확장 한계 인지.
- Math.min 으로 offset 보호 (page 너무 큰 값 → ArrayIndexOutOfBoundsException 회피).

---

## F9. Polling fallback (`findByScanSince`)

### 무엇인가
SSE 안 되는 환경 (corporate proxy 등) 위한 polling endpoint. since 파라미터로 마지막 받은 seq 이후만.

### 코드
```java
// FindingRepository
List<Finding> findByScanSince(ScanId scanId, long sinceSeq, int limit);

// FindingController
@GetMapping("/scans/{scanId}/findings")
public List<FindingResponse> listByScan(@PathVariable UUID scanId,
                                        @RequestParam(required = false) Long since,
                                        @RequestParam(defaultValue = "100") int limit) {
    ScanId sid = ScanId.of(scanId);
    List<Finding> findings = since == null
        ? findingQuery.findByScan(sid)
        : findingQuery.findByScanSince(sid, since, limit);
    return findings.stream().map(FindingResponse::fromDomain).toList();
}
```

### 사용 (frontend)
```
GET /api/scans/<id>/findings?since=5
→ seq > 5 인 finding 만 반환
```

### 핵심 아이디어
**SSE + polling 둘 다 제공**. 환경 호환성.

### 함정
- polling 간격 짧으면 서버 부하. 권장 5초+.
- since 누락 시 전체 반환 — 큰 list.

---

## F10. catch-all + status mapping (controller 단)

### 무엇인가
도메인 예외 → HTTP status 매핑. `@RestControllerAdvice` 가 모든 컨트롤러 catch.

### 코드
```java
// scan/presentation/ScanExceptionHandler.java
@RestControllerAdvice(basePackageClasses = ScanController.class)
class ScanExceptionHandler {

    @ExceptionHandler(TargetNotAccessible.class)
    public ResponseEntity<ProblemDetail> notAccessible(TargetNotAccessible e) {
        return GlobalExceptionHandler.problem(HttpStatus.FORBIDDEN,
            "target-not-accessible", "Target Not Accessible", e.getMessage());
    }

    @ExceptionHandler(ScanAlreadyRunning.class)
    public ResponseEntity<ProblemDetail> alreadyRunning(ScanAlreadyRunning e) {
        return GlobalExceptionHandler.problem(HttpStatus.CONFLICT,
            "scan-already-running", "Scan Already Running", e.getMessage());
    }
    // ...
}
```

### 왜 모듈별?

`basePackageClasses = ScanController.class` → scan presentation 만 적용. 다른 모듈의 같은 예외 (있다면) 와 분리.

### 핵심 아이디어
**도메인 예외 → HTTP status 매핑이 controller layer 의 책임**. service/domain 은 HTTP 모름.

### ProblemDetail (RFC 7807)
표준 에러 body:
```json
{
  "type": "scan-already-running",
  "title": "Scan Already Running",
  "status": 409,
  "detail": "a scan is already running for target <uuid>"
}
```

### 함정
- 모듈 간 같은 예외 (예: shared 의 일반 예외) 처리는 GlobalExceptionHandler 에 모음.
- Hibernate 의 `EntityNotFoundException` 같은 인프라 예외도 매핑 필요.

---

## F11. ResponseCookie type-safe builder

### 무엇인가
Spring 의 ResponseCookie 빌더. 직접 Set-Cookie 헤더 작성보다 안전.

### 코드
```java
// AuthLoginController.login
ResponseCookie cookie = ResponseCookie.from(AuthFilter.COOKIE_NAME, token)
    .httpOnly(true)              // JS 접근 차단
    .secure(false)               // dev — HTTPS only false
    .sameSite("Lax")             // CSRF 방어
    .path("/")
    .maxAge(Duration.ofSeconds(tokenService.ttlSeconds()))
    .build();

return ResponseEntity.ok()
    .header(HttpHeaders.SET_COOKIE, cookie.toString())
    .body(...);
```

### 왜 builder?
- raw `"vulnscope_session=...; HttpOnly; SameSite=Lax; Path=/; Max-Age=86400"` 직접 작성 → escape/format 실수 가능.
- builder 가 escape + 표준 format 자동.

### 핵심 아이디어
**type-safe builder = 잘못된 헤더 생성 방지**.

상세 보안 의미는 H 카테고리.

---

## §∞. 정리

| 문제 | 패턴 |
|---|---|
| 파일 저장 키 설계 | F1 org/날짜 파티션 |
| 콘텐츠 dedupe | F2 sha256 + org-scoped |
| HTTP cache | F3 ETag = sha256 |
| URL 안전 인코딩 | F4 base64url no-padding |
| SSE resume | F5 Last-Event-ID + readSince + subscribe |
| SSE leak 방지 | F6 emitter 콜백 3종 |
| 정렬 조합 | F7 Comparator combinator |
| in-memory pagination | F8 전체 fetch + sublist |
| SSE fallback | F9 polling (since 파라미터) |
| 도메인 예외 → HTTP | F10 @ExceptionHandler + ProblemDetail |
| Cookie type-safe | F11 ResponseCookie builder |

### 추가 참고
- `study/comparator.md` — F7 상세
- `study/sse-libraries.md` — F5/F6 상세
- RFC 7807 ProblemDetail — F10
- RFC 4648 base64url — F4
