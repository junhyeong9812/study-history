# G. HTTP / REST API 패턴

> REST endpoint 디자인 결정들. 9개 패턴.

---

## G1. Cookie httpOnly + sameSite=Lax

### 무엇인가
- **httpOnly**: JS 가 cookie 못 읽음 (`document.cookie` 에 안 보임).
- **sameSite=Lax**: cross-site POST 에 cookie 안 보냄. cross-site GET 은 OK (link 클릭 등).

### 코드
```java
ResponseCookie.from(COOKIE_NAME, token)
    .httpOnly(true)
    .sameSite("Lax")
    .path("/")
    .maxAge(Duration.ofSeconds(ttl))
    .build();
```

### 왜 이 조합?
- **httpOnly** = XSS 방어. 공격자 JS 가 cookie 탈취 불가.
- **sameSite=Lax** = CSRF 1차 방어. 다른 사이트의 form POST 가 cookie 안 첨부.

### 전문 용어 사전
- **XSS** (Cross-Site Scripting): 공격자 JS 가 페이지에 주입.
- **CSRF** (Cross-Site Request Forgery): 사용자 cookie 로 다른 사이트가 요청 위조.
- **sameSite**: `Strict` (모든 cross-site 차단) / `Lax` (top-level GET 허용) / `None` (cross-site 허용, secure 필수).

VulnScope 가 `Lax` 인 이유: link 클릭 navigation (top-level GET) 시 cookie 첨부 = 정상 UX. cross-site POST 만 차단.

### 함정
- `secure=false` 는 dev 만. production HTTPS 면 `true`.
- `sameSite=None` 사용 시 `secure=true` 필수 (브라우저 강제).

H 카테고리에서 보안 깊이 다룸.

---

## G2. 422 vs 400 분리

### 무엇인가
- **400 Bad Request**: 파싱 실패 (잘못된 JSON, 누락 필드).
- **422 Unprocessable Entity**: 파싱 OK 지만 의미 위반.

### 코드
```java
@ExceptionHandler(InvalidTargetValue.class)   // type-value 불일치 (의미 위반)
public ResponseEntity<ProblemDetail> invalid(InvalidTargetValue e) {
    return GlobalExceptionHandler.problem(HttpStatus.UNPROCESSABLE_ENTITY,    // 422
        "invalid-target-value", "Invalid Target Value", e.getMessage());
}
```

### 예시
- `400`: `{"type": "INVALID"}` — `type` 이 enum 값 아님 (Spring Jackson 이 자동 400).
- `422`: `{"type": "IP", "value": "https://example.com"}` — type 은 enum OK, value 도 String OK, 하지만 IP type 에 URL 못 들어감 (의미 위반).

### 왜 분리?
- 클라이언트 에러 메시지 구분.
- monitoring/alerting 분리 가능.

### 출처
RFC 4918 §11.2 (WebDAV). 일반화되어 REST 에서 표준.

---

## G3. RFC 7807 ProblemDetail

### 무엇인가
표준 에러 응답 body. type/title/status/detail 4필드.

### 코드
```java
// shared/error/ProblemDetail.java
public record ProblemDetail(
    String type,
    String title,
    int status,
    String detail
) {}
```

### 응답 예
```json
{
  "type": "scan-already-running",
  "title": "Scan Already Running",
  "status": 409,
  "detail": "a scan is already running for target 12345..."
}
```

### 왜 표준?
- 클라이언트가 `type` 으로 에러 분류 가능 (status code 는 너무 broad).
- `title` 은 사람 읽음. `detail` 은 디버깅.

### 핵심 아이디어
**구조화된 에러 = 클라이언트 처리 가능**. 평문 메시지는 parse 어려움.

### 함정
- type 은 URI 권장 (RFC) — VulnScope 는 단순 string. 작은 시스템에 OK.
- 보안 민감 정보 (스택 트레이스, 내부 path) 누출 X.

### 출처
RFC 7807.

---

## G4. Nested resource URL

### 무엇인가
부모-자식 관계를 URL 에 표현.

### 예시
- `/scans/{scanId}/findings` — scan 의 findings.
- `/scans/{scanId}/findings/{findingId}` — 특정 finding.
- `/findings/{findingId}/evidence` — finding 의 evidence.
- `/uploads/{id}/download` — upload 의 download action.
- `/scans/{id}/stream` — scan 의 SSE stream.
- `/scans/{id}/summary` — scan 의 summary sub-resource.

### 왜 nested?
- 관계 표현이 URL 에 자연.
- 권한 체크가 쉬워짐 (scan 의 ownership = 그 안 finding 의 ownership).

### 함정
- 너무 깊은 nesting (`/a/b/c/d/e`) 권장 X. 2단계 정도.
- nested vs flat — 둘 다 valid. 컨벤션 일관.

---

## G5. Pagination envelope `{data, meta}`

### 무엇인가
list 응답이 raw array 가 아닌 `{data: [...], meta: {...}}` 구조.

### 코드
```java
// ScanPageResponse
public record ScanPageResponse(
    List<ScanResponse> data,
    PageMeta meta
) {
    public record PageMeta(int page, int size, long totalItems, int totalPages) {}
}
```

### 응답 예
```json
{
  "data": [
    {"id": "...", "status": "DONE", ...},
    ...
  ],
  "meta": {
    "page": 0,
    "size": 20,
    "totalItems": 142,
    "totalPages": 8
  }
}
```

### 왜 envelope?
- 페이지 정보 (총 개수, 다음 페이지 여부) 포함.
- frontend 가 일관 처리 (`data` 항상 존재, `meta` 항상 페이지 정보).
- 향후 `links` 추가 (HATEOAS) 자연.

### 함정
- 단건 응답에는 envelope 안 씀 — overkill.
- 모든 list endpoint 에 envelope 일관 사용 (mixed 면 frontend 혼란).

### 출처
JSON:API spec, OpenAPI 컨벤션.

---

## G6. Spring auto query parameter binding

### 무엇인가
Instant ISO-8601, enum, primitive 등 자동 파싱.

### 코드
```java
// ScanController.list
@GetMapping
public ScanPageResponse list(
    @RequestParam(required = false) UUID target,
    @RequestParam(required = false) Instant from,         // ISO-8601 → Instant
    @RequestParam(required = false) Instant to,
    @RequestParam(required = false) ScanStatus status,    // "DONE" → enum
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size
) { ... }
```

### 사용 예
```
GET /scans?from=2026-04-30T00:00:00Z&status=DONE&page=0&size=20
→ Spring 이 자동 파싱
```

### 왜 편한가
- 직접 `String → Instant.parse` 같은 boilerplate 0.
- 잘못된 값 (예: status=INVALID) → Spring 자동 400 응답.

### 핵심 아이디어
**framework default 활용** — Spring 의 `Converter` registry.

### 함정
- 잘못된 값 시 default error 가 그리 친절하지 X. 필요 시 `@ExceptionHandler(MethodArgumentTypeMismatchException.class)` 로 custom.

---

## G7. ResponseEntity + Optional 패턴

### 무엇인가
Optional 의 `.map().orElseGet()` 으로 200 vs 404 분기.

### 코드
```java
// ScanController.get
@GetMapping("/{id}")
public ResponseEntity<ScanResponse> get(@PathVariable UUID id) {
    return scanQuery.findById(ScanId.of(id))
        .map(s -> ResponseEntity.ok(ScanResponse.fromDomain(s)))
        .orElseGet(() -> ResponseEntity.notFound().build());
}
```

### 왜 패턴화?
- 모든 GET by id 에 동일.
- if/else 보다 functional.

### 함정
- `orElse()` 와 `orElseGet()` 차이: 전자는 항상 평가, 후자는 lazy. 매번 새 ResponseEntity 생성이라 `orElseGet` 권장.

---

## G8. JDK `HttpClient` (외부 의존 X)

### 무엇인가
Java 11+ 표준 HTTP 클라이언트. Apache HttpClient/OkHttp 같은 외부 의존 불필요.

### 코드
```java
// HttpHeaderProbe
public HttpHeaderProbe() {
    this.client = HttpClient.newBuilder()
        .followRedirects(HttpClient.Redirect.NORMAL)
        .connectTimeout(Duration.ofSeconds(10))
        .build();
}

@Override
public ProbeResult fetch(String url) throws Exception {
    HttpRequest request = HttpRequest.newBuilder(URI.create(url))
        .GET()
        .timeout(Duration.ofSeconds(15))
        .header("User-Agent", "VulnScope/0.1 ...")
        .build();
    HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
    // ...
}
```

### 왜 표준?
- 의존 0.
- Apache HttpClient 와 비슷한 기능.
- HTTP/2 지원.
- 가상 스레드 친화 (blocking send + virtual thread = 효율).

### 함정
- 일부 고급 기능 (proxy, advanced retry) 은 OkHttp 가 더 풍부.
- VulnScope 의 단순 GET 에는 표준 충분.

### 출처
JEP 321 (Java 11).

---

## G9. SSE wire format (`id:`, `event:`, `data:`)

### 무엇인가
Server-Sent Events 표준 텍스트 format.

### 코드 (Spring SseEmitter)
```java
emitter.send(SseEmitter.event()
    .id(String.valueOf(event.seq()))    // → "id: 5"
    .name(event.type())                  // → "event: finding"
    .data(event.payload()));             // → "data: {...}" (Jackson 직렬화)
```

### Wire format
```
id: 5
event: finding
data: {"findingId":"...","severity":"HIGH"}

```
- `id:` — Last-Event-ID 의 키.
- `event:` — type. 클라가 `addEventListener(type, ...)` 매칭.
- `data:` — payload. JSON 직렬화 (Spring 자동).
- 빈 줄로 이벤트 구분.

### 핵심 아이디어
**HTML5 표준 + Spring 의 builder 가 boilerplate 자동**.

상세는 `study/sse-libraries.md`.

---

## §∞. 정리

| 문제 | 패턴 |
|---|---|
| Cookie 보안 | G1 httpOnly + sameSite |
| status 정확도 | G2 422 vs 400 |
| 표준 에러 | G3 ProblemDetail (RFC 7807) |
| 관계 URL | G4 nested resource |
| pagination | G5 {data, meta} envelope |
| query parameter | G6 Spring auto binding |
| Optional → 응답 | G7 .map().orElseGet() |
| HTTP 클라이언트 | G8 JDK HttpClient |
| SSE wire | G9 SseEmitter event builder |
