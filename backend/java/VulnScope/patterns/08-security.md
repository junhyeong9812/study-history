# H. 보안

> web app 의 기본 보안 — 인증/인가/timing attack/path traversal/multi-tenant 격리. 7개 패턴.

---

## §0. 보안의 mental model

**공격자 가정**: 사용자는 모두 잠재적 공격자. 입력은 모두 신뢰 X. 시간/길이/byte 패턴 등 모든 관찰 가능 정보 활용.

**defense in depth**: 한 layer 가 뚫려도 다른 layer 가 막음. 다중 방어.

VulnScope 의 보안은 7개 layer.

---

## H1. AuthFilter (`OncePerRequestFilter`)

### 무엇인가
Spring Web Filter. 모든 요청이 controller 도달 전 가로채. 인증 검증.

### 코드
```java
@Component
public class AuthFilter extends OncePerRequestFilter {

    public static final String COOKIE_NAME = "vulnscope_session";

    private static final Set<String> PUBLIC_PATHS = Set.of(
        "/auth/login", "/auth/logout", "/actuator/health", "/actuator/info"
    );

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        String path = req.getRequestURI();
        if (isPublic(path)) { chain.doFilter(req, res); return; }

        String token = extractCookie(req);
        if (token == null) { res.sendError(401); return; }

        TokenService.TokenPayload payload;
        try {
            payload = tokenService.verify(token);
        } catch (TokenService.InvalidTokenException e) {
            res.sendError(401); return;
        }

        AuthenticatedPrincipal principal = new AuthenticatedPrincipal(
            payload.userId(), payload.orgId(), Set.of("MEMBER"));
        TenantContext.set(payload.orgId());          // ← per-request 컨텍스트
        req.setAttribute(PRINCIPAL_ATTR, principal);
        try {
            chain.doFilter(req, res);
        } finally {
            TenantContext.clear();                    // ← leak 방지
        }
    }
}
```

### OncePerRequestFilter 의 의미
Spring 의 base 클래스. **forwarded request (예: error page) 에 두 번 안 돌게 보장**. 일반 Filter 는 forward 시 다시 호출됨.

### 핵심 아이디어
**모든 보호 endpoint 의 cross-cutting concern 을 한 곳에**. controller 마다 인증 코드 0.

### 함정
- ThreadLocal clear 누락 → A9 참조.
- async 컨텍스트 (CompletableFuture, @Async) 로 ThreadLocal 자동 전파 X.

상세는 `01-concurrency.md` A8/A9, 본 doc H6 참조.

---

## H2. PUBLIC_PATHS whitelist (default deny)

### 무엇인가
명시적 허용 path 만 인증 bypass. 그 외 모두 인증 강제.

### 코드
```java
private static final Set<String> PUBLIC_PATHS = Set.of(
    "/auth/login", "/auth/logout", "/actuator/health", "/actuator/info"
);

private boolean isPublic(String path) {
    return PUBLIC_PATHS.contains(path);
}
```

### 왜 whitelist (not blacklist)?

**blacklist (잘못된 패턴)**:
```java
private static final Set<String> PROTECTED_PATHS = Set.of("/scans", "/findings", ...);
if (!PROTECTED_PATHS.contains(path)) chain.doFilter(...);   // ❌ 새 endpoint 추가 시 노출
```

새 endpoint (예: `/uploads`) 추가 시 PROTECTED_PATHS 갱신 잊으면 → 인증 없이 접근 가능 → **보안 사고**.

**whitelist**:
- unknown path 도 인증 강제.
- safe by default.

### 핵심 아이디어
**Secure by default**. 명시 허용만 통과.

### 전문 용어 사전
- **default deny**: 명시 허용 외 모두 차단. 보안 표준.
- **default allow**: 명시 차단 외 모두 통과. 위험.

---

## H3. Constant-time HMAC comparison (timing attack 방어)

### 무엇인가
바이트 비교 시 모든 바이트 검사 (early-return X). 시간 측정으로 비밀 추론 막음.

### 코드
```java
// TokenService
private static boolean constantTimeEquals(byte[] a, byte[] b) {
    if (a.length != b.length) return false;
    int result = 0;
    for (int i = 0; i < a.length; i++) {
        result |= a[i] ^ b[i];          // XOR 누적, early-return 없음
    }
    return result == 0;
}
```

### 왜 필요한가 — timing attack 시나리오

**naive (`Arrays.equals`)**:
```java
return Arrays.equals(expectedSig, providedSig);
// 내부적으로 첫 다른 바이트에서 즉시 false return
```

공격자가 측정 가능:
- providedSig 첫 byte = `0x00` → 다르면 0.001ms 만에 false 반환.
- providedSig 첫 byte = `0xa3` → 첫 byte 일치 → 두 번째 byte 비교 → 0.002ms.

→ 첫 byte 256개 중 정답 (`0xa3`) 만 약간 느림. **byte by byte brute force** 가능.

**constant-time**:
- 모든 byte 검사 (early return 없음).
- 시간이 비밀 의존 X.

### 핵심 아이디어
**side-channel 방어**. CPU 시간/cache/전력 등 관찰 가능한 모든 채널을 leak 방지.

### 전문 용어 사전
- **timing attack**: 응답 시간 측정으로 비밀 추론.
- **side-channel attack**: 정상 입출력 외의 채널 (시간, 전력, 소리 등) 활용.
- **constant-time**: 비밀 데이터에 의존 X 인 일정 시간.

### 함정
- compiler/JIT optimization 이 early-return 도입할 수 있음 — JIT 가 똑똑하면 위험. Java 의 단순 loop 는 보통 안전.
- HMAC 비교에는 `MessageDigest.isEqual` (Java 표준) 사용 가능 — 같은 의미.

### 대안
- `MessageDigest.isEqual(a, b)` — 표준. constant-time 보장.
- `Arrays.equals` — 일반 데이터에 OK, **secret 비교에 위험**.

### 출처
- 1996 Paul Kocher, *Timing Attacks on Implementations of Diffie-Hellman, RSA, DSS, and Other Systems*.
- OWASP Cryptographic Storage Cheat Sheet.

---

## H4. Path traversal defense (`startsWith(root)`)

### 무엇인가
파일 path 정규화 후 root 안인지 검증. `../` 우회 차단.

코드 + 상세는 `02-data-structures.md` B8 참조.

핵심:
```java
private Path resolve(String key) {
    Path resolved = root.resolve(key).normalize();
    if (!resolved.startsWith(root)) {
        throw new IllegalArgumentException("path traversal blocked: " + key);
    }
    return resolved;
}
```

### 왜 보안 critical?
공격자가 `key="../../../etc/passwd"` → 시스템 파일 읽기 가능 → 정보 노출.

### 출처
OWASP Path Traversal Cheat Sheet. CWE-22.

---

## H5. Multi-tenant orgId isolation

### 무엇인가
모든 데이터 access 가 현재 사용자 org 검증. cross-tenant leak 차단.

### 어디서

#### 1) Repository 의 `findByOrg*` 메서드
```java
// UploadRepository
Optional<Upload> findByOrgAndSha256(OrgId orgId, Sha256 sha256);

// TargetRepository
Optional<Target> findByOrgAndTypeAndValue(OrgId orgId, TargetType type, TargetValue value);
List<Target> findRecentByOrg(OrgId orgId, int limit);
```

→ 다른 org 의 데이터 자체를 못 봄.

#### 2) Aggregate 가 orgId 보유 + ownership check
```java
// scan/application/api/TargetQuery
boolean isOwnedBy(TargetId id, OrgId orgId);

// TriggerScanService — scan trigger 시 검증
if (!targetQuery.isOwnedBy(targetId, orgId)) {
    throw new TargetNotAccessible(targetId);
}
```

#### 3) Visibility 메서드
```java
// Profile.isVisibleTo
public boolean isVisibleTo(OrgId viewerOrg) {
    return kind == ProfileKind.SYSTEM
        || (orgId != null && orgId.equals(viewerOrg));
}
```

#### 4) Controller 단 권한 check
```java
// UploadController.download
UUID uploadOrg = ds.meta().orgId();
UUID currentOrg = TenantContext.currentOrgId();
if (!uploadOrg.equals(currentOrg)) {
    return ResponseEntity.status(403).build();
}
```

### 왜 모든 layer?

**defense in depth** — 한 layer 가 뚫려도 다른 layer 가 막음.

- Repository: orgId 안 받으면 query 자체 실패.
- Aggregate: orgId 필드 있어 검증 가능.
- Controller: TenantContext 로 마지막 검증.

### 핵심 아이디어
**multi-tenant SaaS 의 가장 critical 한 invariant** — "내 데이터만 보임".

### 함정
- 한 곳이라도 빠뜨리면 cross-tenant leak. **모든 entry point 검증**.
- ThreadLocal (TenantContext) leak 시 다른 사용자로 인지 가능 → A9 finally clear 필수.

### 출처
- SaaS multi-tenancy 표준.
- AWS Multi-Tenant SaaS storage strategies.

---

## H6. Controller 단 권한 check (TenantContext 사용처)

### 무엇인가
ThreadLocal (TenantContext) 사용 코드는 controller 에 모음. service 단 격리.

### 왜?
service 가 TenantContext 직접 호출 시:
- 단위 테스트에서 ThreadLocal set 필요 → 복잡.
- service 의 invariant 가 "ThreadLocal 에 set 된 값" 에 의존 → 결합.

**Controller 가 ThreadLocal 추출 후 service 에 명시 전달**:
```java
// ScanController
OrgId orgId = OrgId.of(TenantContext.currentOrgId());     // controller 가 추출
Scan scan = scanCommand.trigger(orgId, ...);              // service 에 명시 전달
```

### 핵심 아이디어
**implicit context (ThreadLocal) 는 boundary (controller) 에서만 사용**. 안쪽은 명시 인자.

### 예외: UploadController.download
download 가 `Optional<DownloadStream>` 받고 권한 check — service 가 권한 모름. 권한 check 가 controller 에. 의도된 디자인 — DownloadStream 이 메타 + 스트림 묶어서 service signature 단순.

---

## H7. Token secret 길이 검증 (weak secret 차단)

### 무엇인가
HMAC secret 이 너무 짧으면 부팅 실패.

### 코드
```java
// TokenService 생성자
public TokenService(@Value("${app.security.token.secret}") String secret, ...) {
    if (secret == null || secret.length() < 16) {
        throw new IllegalStateException("token secret must be at least 16 chars");
    }
    // ...
}
```

### 왜 필요한가
짧은 secret (`"abc"`) → brute force 가능. 모든 가능한 secret 시도 → token 위조.

16자 minimum 은 base level. production 권장 32+ random bytes.

### 핵심 아이디어
**fail-fast at startup** — 잘못된 설정으로 운영 X. 부팅 단계에 차단.

### 함정
- secret 을 코드에 hardcoded → git 노출. `${VULNSCOPE_TOKEN_SECRET:dev-secret}` 패턴 (env override).
- production 의 default value (`dev-secret-...`) 사용 시 운영자가 변경 안 했을 가능성 → 별도 monitoring.

### 출처
OWASP Authentication Cheat Sheet.

---

## §∞. 정리

| 문제 | 패턴 |
|---|---|
| 모든 요청 인증 | H1 AuthFilter (OncePerRequestFilter) |
| safe by default | H2 PUBLIC_PATHS whitelist |
| timing attack 방어 | H3 constant-time compare |
| path traversal | H4 normalize + startsWith |
| cross-tenant leak | H5 multi-tenant isolation (모든 layer) |
| ThreadLocal boundary | H6 controller 단 권한 |
| weak config | H7 secret 길이 startup 검증 |

### 학습 추천
보안은 catalog 보다 **mental model** 중요:
1. **default deny** (H2)
2. **defense in depth** (H5)
3. **fail-fast at startup** (H7)
4. **side-channel 인지** (H3)

위 4가지를 알면 새 보안 결정도 자연스럽게 됨.
