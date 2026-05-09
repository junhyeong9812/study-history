# B. 데이터 구조 + 인덱싱

> 이 문서가 다루는 것: in-memory 저장소를 RDBMS 처럼 효율적으로 (O(1) lookup) 만드는 기법 + 안전한 입력 정규화. 9개 패턴.
> 전제: A 카테고리 (동시성) 의 ConcurrentHashMap 이해.

---

## §0. 왜 in-memory 도 "인덱싱" 이 필요한가?

VulnScope v0.1 은 DB 안 씀. 모든 데이터 `ConcurrentHashMap<UUID, Aggregate>` 에 저장.

**naive 검색**:
```java
public Optional<Upload> findByOrgAndSha256(OrgId orgId, Sha256 sha) {
    return byId.values().stream()
        .filter(u -> u.orgId().equals(orgId) && u.sha256().equals(sha))
        .findFirst();
}
```
→ N 개 upload 라면 O(N). 1만 개면 1만 번 비교. dedupe 매 호출 시 비용 큼.

**인덱싱**:
```java
private final ConcurrentMap<DedupeKey, UUID> byOrgSha = new ConcurrentHashMap<>();

public Optional<Upload> findByOrgAndSha256(OrgId orgId, Sha256 sha) {
    UUID id = byOrgSha.get(new DedupeKey(orgId.value(), sha.hex()));
    return id == null ? Optional.empty() : Optional.ofNullable(byId.get(id));
}
```
→ O(1). HashMap lookup 두 번.

이게 RDBMS 의 secondary index 흉내. v0.2 DB 도입 시 같은 의미의 SQL `CREATE INDEX` 로 자연 전환.

---

## B1. Two-index repository — primary + secondary

### 무엇인가
`byId` (primary, ID → 객체) + `byKey` (secondary, 다른 키 → ID) 두 인덱스 동시 유지.

### 어디서 쓰나

#### 1) UploadRepository — sha256 dedupe
```java
// InMemoryUploadRepository
private final ConcurrentMap<UUID, Upload> byId = new ConcurrentHashMap<>();
private final ConcurrentMap<DedupeKey, UUID> byOrgSha = new ConcurrentHashMap<>();

@Override
public Upload save(Upload upload) {
    byId.put(upload.id().value(), upload);
    byOrgSha.put(
        new DedupeKey(upload.orgId().value(), upload.sha256().hex()),
        upload.id().value()
    );
    return upload;
}

@Override
public Optional<Upload> findByOrgAndSha256(OrgId orgId, Sha256 sha256) {
    UUID id = byOrgSha.get(new DedupeKey(orgId.value(), sha256.hex()));
    return id == null ? Optional.empty() : Optional.ofNullable(byId.get(id));
}

private record DedupeKey(UUID orgId, String sha256) {}
```

#### 2) TargetRepository — (org, type, value) unique
```java
// InMemoryTargetRepository
private final ConcurrentMap<UUID, Target> byId;
private final ConcurrentMap<UniqueKey, UUID> uniqueIndex;

@Override
public Target save(Target target) {
    UniqueKey key = new UniqueKey(target.orgId().value(), target.type(), target.value().raw());
    UUID prior = uniqueIndex.get(key);
    if (prior != null && !prior.equals(target.id().value())) {
        throw new IllegalStateException("duplicate target: " + key);  // 두 번째 방어선
    }
    byId.put(target.id().value(), target);
    uniqueIndex.put(key, target.id().value());
    return target;
}

private record UniqueKey(UUID orgId, TargetType type, String value) {}
```

### 왜 필요한가
- **primary index** (byId): 단건 조회 + 중복 키 검사.
- **secondary index** (byKey): 비-id 기반 조회 (dedupe, ownership 등) 를 O(1) 로.

stream filter (O(N)) 보다 O(1). 데이터 늘어도 lookup 시간 일정.

### 핵심 아이디어
**RDBMS 의 secondary index 를 in-memory 로 흉내**. v0.2 DB 면 `CREATE UNIQUE INDEX` 한 줄로 같은 의미.

```sql
-- v0.2 SQL 등가물
CREATE UNIQUE INDEX upload_org_sha ON upload (org_id, sha256);
```

### 두 인덱스 일관성
**save 시 두 인덱스 모두 갱신**. 빠뜨리면 silent corruption (한쪽엔 있고 한쪽엔 없음).

```java
public Upload save(Upload upload) {
    byId.put(...);          // ① primary
    byOrgSha.put(...);      // ② secondary
    return upload;
}
```

→ 두 put 사이에 다른 스레드가 read 하면 일시적 inconsistent. ConcurrentHashMap 이 individual op 만 atomic 보장. **truly atomic 이 필요하면 lock 필요** (VulnScope 는 race window 무시 가능 판단).

### 함정
- delete 시 두 인덱스 모두 삭제 안 하면 dangling reference (byKey 가 없는 byId 가리킴).
- update 시 secondary key 변경되면 (예: target value 수정) 옛 entry 남음. v0.1 은 update 안 함.

### 대안
- 단일 인덱스 + stream filter — 작은 N 에 OK. 큰 N 에 부적합.
- **Indexed in-memory 라이브러리** (Apache Commons collections-IndexedCollection) — 표준 X, 학습 비용. VulnScope 는 직접.

### 출처
RDBMS 설계 패턴 (Codd, 1970년대). secondary index 는 1960년대 IBM IMS 부터 표준.

---

## B2. Nested record as composite key

### 무엇인가
여러 필드 묶은 record 를 Map 의 key 로 사용. record 가 equals/hashCode 자동 생성.

### 어디서 쓰나
```java
// InMemoryUploadRepository
private record DedupeKey(UUID orgId, String sha256) {}

// InMemoryTargetRepository
private record UniqueKey(UUID orgId, TargetType type, String value) {}
```

### 왜 필요한가

**naive 1**: String concat
```java
String key = orgId + "|" + sha256;     // ❌
map.put(key, ...);
```
**문제**: 충돌 가능 (`"A|B|C"` vs `"A|B" + "|C"`). 파싱 어려움. `|` 문자 입력에 취약.

**naive 2**: 직접 클래스
```java
class DedupeKey {                       // ❌ boilerplate
    final UUID orgId;
    final String sha256;
    @Override public boolean equals(Object o) { ... }
    @Override public int hashCode() { ... }
    @Override public String toString() { ... }
    // 생성자, getter ...
}
```
**문제**: 30줄 boilerplate. 필드 추가 시 equals/hashCode 같이 수정.

**record (Java 14+)**:
```java
private record DedupeKey(UUID orgId, String sha256) {}
```
**효과**:
- equals/hashCode/toString 자동.
- accessor (`orgId()`, `sha256()`) 자동.
- 불변 보장.
- 한 줄 정의.

### 핵심 아이디어
**equals/hashCode 가 정확해야 HashMap 키로 안전**. record 는 모든 필드 사용한 표준 구현 자동 → 휴먼 에러 0.

### 전문 용어 사전
- **value-based equality**: 객체 동일성 (==) 이 아닌 필드 값 비교 (equals). HashMap 키는 value-based 필요.
- **canonical equals**: 모든 필드 사용한 equals. record 가 자동 생성하는 것.

### 함정
- record 의 필드 중 mutable 객체가 있으면 (예: `List`) 외부 변경 시 hashCode 깨짐. `List.copyOf` (B5) 로 보호.
- nested record 는 enclosing class 의 private 라도 reflection access 시 노출. 보안 민감 데이터는 별도.

### 대안
- Java 14 미만이면 Lombok `@Value` 또는 직접 작성.
- `Map.Entry<K,V>` — 두 필드 묶기 가능. 의미 X (값 의미 안 살아남).
- Tuple library — 의존 추가. record 가 표준.

### 출처
JEP 395 Records (Java 14 standard).

---

## B3. UUID value object → ConcurrentMap 키

### 무엇인가
도메인 식별자 (`ScanId`, `TargetId` 등) 는 record + UUID. 하지만 Map 의 키로는 raw `UUID` 사용.

### 어디서 쓰나
```java
// InMemoryScanRepository
private final ConcurrentMap<UUID, Scan> byId = new ConcurrentHashMap<>();

@Override
public Scan save(Scan scan) {
    byId.put(scan.id().value(), scan);     // ScanId → UUID (.value())
    return scan;
}

@Override
public Optional<Scan> findById(ScanId id) {
    return Optional.ofNullable(byId.get(id.value()));   // ScanId → UUID
}
```

### 왜 raw UUID?

**옵션 1 — `ScanId` 를 키로 직접**:
```java
ConcurrentMap<ScanId, Scan> byId;       // 가능
byId.put(scan.id(), scan);
```
- `ScanId` 가 record 라 equals/hashCode 자동. 작동.
- 단점: cross-module 에서 같은 UUID 참조 시 `ScanId` import 필요. 다른 모듈이 raw UUID 만 가지면 변환 필요.

**옵션 2 — raw UUID** (현재):
- Map 내부는 raw UUID. 도메인 타입 안 새어나옴.
- cross-module 호출 시 UUID 만 받아도 lookup 가능.
- VulnScope 의 controller 가 `@PathVariable UUID id` 로 받음 → 바로 lookup.

### 핵심 아이디어
**도메인 타입 안전성 (compile time)** + **저장 타입 단순성 (runtime)** 분리.

호출 site 에서는 `ScanId` 로 타입 안전. lookup 시 `.value()` 로 raw UUID 추출.

### 트레이드오프
- compile-time: `ScanId` 가 `TargetId` 와 섞이지 않게 막음.
- runtime: Map 키는 raw UUID — 두 record 를 매번 비교하지 않음 (.value() 한 번 추출).

### 함정
- 직접 raw UUID 로 lookup 시 (도메인 타입 우회) 위험. 컨벤션: 항상 `ScanId.of(...).value()` 또는 record accessor 사용.

### 대안
- 모든 곳에서 `ScanId` 사용 — 더 strict, 더 verbose.
- 모든 곳에서 raw UUID — 타입 안전성 0.
- 현재가 균형.

---

## B4. `Optional.ofNullable` — null → empty

### 무엇인가
nullable 값을 `Optional<T>` 로 wrap. `null` → `Optional.empty()`, non-null → `Optional.of(value)`.

### 어디서 쓰나
모든 `findById`:
```java
@Override
public Optional<Scan> findById(ScanId id) {
    return Optional.ofNullable(byId.get(id.value()));
}
```

### 왜 필요한가
**naive**:
```java
public Scan findById(ScanId id) {
    return byId.get(id.value());     // null 가능
}

// 호출 측
Scan scan = repo.findById(id);
scan.markRunning(...);                // NullPointerException 가능!
```

**Optional 강제**:
```java
public Optional<Scan> findById(ScanId id) { ... }

// 호출 측 — 컴파일러가 .map() 또는 .orElseThrow() 강제
Scan scan = repo.findById(id).orElseThrow();
scan.markRunning(...);
```

→ NPE 가 **API signature 에 명시**. 호출자가 빠뜨릴 수 없음.

### 핵심 아이디어
**"null 가능" 을 타입으로 표현**. Java 의 모든 reference 가 implicit nullable 인 문제를 wrap 으로 해결.

### 전문 용어 사전
- **null safety**: null 로 인한 NPE 예방. Kotlin 의 `T?` 처럼 강제하면 compile time 에 잡힘. Java 는 Optional 로 권장.
- **monad** (수학): Optional 은 monad. `map`, `flatMap` 으로 chain. functional 스타일.

### 사용 패턴
```java
// pattern 1: orElseGet
return scanQuery.findById(ScanId.of(id))
    .map(s -> ResponseEntity.ok(ScanResponse.fromDomain(s)))
    .orElseGet(() -> ResponseEntity.notFound().build());

// pattern 2: orElseThrow
Scan scan = repository.findById(id).orElseThrow();   // NoSuchElementException

// pattern 3: ifPresent
repo.findById(id).ifPresent(scan -> ...);
```

### 함정
- **`Optional` 을 필드에 저장 X** — 직렬화 문제. 메서드 반환 타입 전용.
- **`Optional<List<T>>` 안 함** — 빈 List 반환이 더 자연.
- **`Optional.get()` 직접 호출 X** — null safety 의미 파괴. orElseThrow / orElseGet 사용.

### 대안
- `@Nullable` 어노테이션 (JSR-305 등) — 정적 분석. runtime 강제 X.
- Kotlin `T?` — 진짜 compile time 강제.
- Optional 이 Java 의 표준.

### 출처
Java 8 (`java.util.Optional`). 영감: Haskell `Maybe`, Scala `Option`.

---

## B5. `List.copyOf` — defensive copy / immutable wrap

### 무엇인가
List 를 불변 복사본으로 wrap. 외부 변경이 record 내부에 영향 안 미침.

### 어디서 쓰나
```java
// Profile record 의 compact constructor
public record Profile(
    ProfileId id,
    OrgId orgId,
    String name,
    ProfileKind kind,
    String description,
    int crawlDepth,
    int rateRps,
    List<String> modules
) {
    public Profile {
        // ... validation
        modules = List.copyOf(modules);    // ← defensive copy + immutable
    }

    public static Profile system(...) {
        return new Profile(..., modules);
    }
}
```

### 왜 필요한가

**naive**:
```java
public record Profile(..., List<String> modules) {}

// 호출 측 (악의적 또는 실수)
List<String> mutable = new ArrayList<>(List.of("a", "b"));
Profile p = new Profile(..., mutable);
mutable.add("c");                  // ❌ Profile 내부 list 변경됨!
mutable.clear();                    // ❌ Profile.modules() 도 비어버림
```

→ record 의 "불변" 약속이 **참조 공유로 깨짐**. equals/hashCode 도 같이 깨짐 (List 가 hash 계산에 들어가면).

**`List.copyOf`**:
```java
public Profile {
    modules = List.copyOf(modules);    // 불변 복사
}
```
- 호출자가 mutable list 넘겨도 record 안에는 immutable copy.
- 외부 변경 영향 0.
- record 진짜 불변.

### 핵심 아이디어
**"불변" 은 reference 가 아니라 contents 까지**. 반드시 copy + immutable wrap.

### 전문 용어 사전
- **defensive copy**: 외부 입력을 내부에 그대로 들고 있지 않고 복사. 신뢰 경계.
- **structural immutability**: list/map 의 element 까지 변경 안 됨. shallow vs deep.

### 함정
- `List.copyOf` 는 **shallow copy** — list 자체는 immutable, element 는 그대로 (mutable 객체면 element 변경 가능).
- record 내부 사용 시 `modules.add(x)` 호출 시 `UnsupportedOperationException`.
- `Map.copyOf`, `Set.copyOf` 도 같은 패턴.

### 대안
- `Collections.unmodifiableList(new ArrayList<>(modules))` — 명시적이지만 verbose.
- `ImmutableList.copyOf(modules)` — Guava. 외부 의존.
- Java 10+ 면 `List.copyOf` 가 표준.

### 출처
Effective Java (3rd ed) Item 50 "Make defensive copies when needed".

---

## B6. Retention trim — 개수 + 시간 두 정책

### 무엇인가
in-memory 큐의 메모리 보호. "최대 N 개" + "마지막 T 시간 이내" 두 조건 모두 통과한 것만 유지.

### 어디서 쓰나
```java
// InMemoryRealtimeStore.trim
private static final int MAX_EVENTS_PER_CHANNEL = 10_000;
private static final Duration RETENTION = Duration.ofMinutes(60);

private void trim(ChannelLog log) {
    // 1) 개수 cap
    while (log.events.size() > MAX_EVENTS_PER_CHANNEL) {
        log.events.pollFirst();
    }
    // 2) 시간 cap
    Instant cutoff = Instant.now().minus(RETENTION);
    while (true) {
        StoredEvent head = log.events.peekFirst();
        if (head == null || head.ts().isAfter(cutoff)) break;
        log.events.pollFirst();
    }
}
```

### 왜 두 정책?

**개수만**: scan 1번이 1만 개 finding 발견하면 그게 유일한 backfill window. 다음 scan 의 이벤트 수신 못 함.

**시간만**: scan 이 1초에 100만 finding 발생시키면 메모리 폭발 (60분 retention 보장 위해).

**둘 다**: 메모리 보호 (개수 cap) + 의미 있는 윈도우 (시간 cap). 둘 중 빠른 것이 trim 트리거.

### 핵심 아이디어
**Kafka 의 retention 정책 (size + time) 차용**. 다른 message system 도 같은 두 axis.

### 전문 용어 사전
- **retention**: 데이터 보관 기간/크기.
- **TTL (time-to-live)**: 시간 기반 retention.
- **eviction policy**: 추방 정책. LRU/LFU/FIFO 등.

### 함정
- trim 을 매 append 시 동기 호출 — append 가 약간 느려짐. async background trim 도 가능 (스케줄러).
- VulnScope 의 trim 은 단순 — append 시점 즉시. 작은 데이터에 OK.

### 대안
- `Caffeine` library — sophisticated cache (LRU + size + time + weight). VulnScope 는 표준 라이브러리만.
- Redis sorted set + ZRANGEBYSCORE — 서버 분리 시.

### 출처
Apache Kafka 의 log retention 정책. Apache Cassandra 의 TTL.

---

## B7. HTTP header lowercase 정규화

### 무엇인가
HTTP 헤더 이름을 lowercase 로 통일. 비교 시 case 무관.

### 어디서 쓰나
```java
// HttpHeaderProbe.normalizeHeaders
private static Map<String, String> normalizeHeaders(Map<String, List<String>> raw) {
    Map<String, String> out = new HashMap<>(raw.size());
    for (Map.Entry<String, List<String>> e : raw.entrySet()) {
        out.put(e.getKey().toLowerCase(), String.join(", ", e.getValue()));   // ← lowercase
    }
    return out;
}

// ProbeResult.hasHeader
public boolean hasHeader(String name) {
    return headers.containsKey(name.toLowerCase());     // ← lowercase 비교
}
```

### 왜 필요한가
**HTTP 표준**: 헤더 이름은 case-insensitive (RFC 7230 §3.2).

```http
X-Frame-Options: DENY
x-frame-options: DENY      ← 같은 헤더
X-FRAME-OPTIONS: DENY      ← 같은 헤더
```

서버는 `X-Frame-Options` 보낼 수도 `x-frame-options` 보낼 수도. case 가 random.

`HashMap.containsKey("x-frame-options")` 는 case-sensitive. 정규화 안 하면 헤더 detect 못 함.

### 핵심 아이디어
**HTTP 표준의 case-insensitive 를 코드 한 곳에서 정규화**. 비교 site 마다 .toLowerCase 호출 X.

### 함정
- `equalsIgnoreCase` 도 가능하지만 매번 비용. 정규화 한 번이 효율.
- HTTP/2 는 strict lowercase 권장. 자연 정렬됨.

### 대안
- `TreeMap` + `String.CASE_INSENSITIVE_ORDER` comparator — 자동. 약간 느림.
- Apache `CaseInsensitiveMap` — 의존 추가.

### 출처
RFC 7230 §3.2. HTTP/1.1 표준.

---

## B8. Path normalization + startsWith — path traversal 방어

### 무엇인가
사용자 입력 path 를 정규화하고 root 안에 있는지 검증. `../` 로 root 밖 못 나가게.

### 어디서 쓰나
```java
// LocalFileObjectStorage
private final Path root;       // absolute, normalized

public LocalFileObjectStorage(@Value("${app.upload.root:./data}") String root) {
    this.root = Path.of(root).toAbsolutePath().normalize();
    //                ↑ 시작 시 정규화 (한 번)
}

private Path resolve(String key) {
    Path resolved = root.resolve(key).normalize();    // ← normalize
    if (!resolved.startsWith(root)) {                 // ← 검증
        throw new IllegalArgumentException("path traversal blocked: " + key);
    }
    return resolved;
}
```

모든 file op (`put`, `get`, `exists`, `delete`) 가 `resolve(key)` 거침.

### 왜 필요한가

**공격 시나리오**:
```
사용자가 storage key 에 "../../../etc/passwd" 입력
→ root.resolve("../../../etc/passwd") = "/abs/data/../../../etc/passwd"
→ normalize() = "/etc/passwd"
→ Files.newInputStream("/etc/passwd")
→ 시스템 파일 읽힘 → 사고
```

**방어**:
```
resolved = "/etc/passwd"
resolved.startsWith("/abs/data") → false
→ IllegalArgumentException → 차단
```

### 핵심 아이디어
**`normalize()` 후 `startsWith(root)` 검증** — `..` 처리된 후에도 root 안인지 확인.

### 전문 용어 사전
- **path traversal (directory traversal)**: `../` 로 의도된 디렉토리 밖 접근. OWASP Top 10 의 단골.
- **canonical path**: symlink/`..` 모두 풀린 진짜 path. `Path.normalize` 가 `..`/`.` 처리. `Path.toRealPath` 가 symlink 까지.

### 함정
- `Path.normalize` 만 하고 startsWith 검증 안 하면 우회 가능.
- `Path.toRealPath` 는 file 존재 시만 작동. write 전이면 못 씀.
- Windows 의 `\` vs Unix 의 `/` — `Path.of` 가 OS 추상화.
- symlink 공격 (root 안에 root 밖 가리키는 symlink) — `toRealPath` 또는 `Files.isSameFile` 추가 검증.

### 대안
- regex 차단 (`/\.\.\//`) — 우회 가능 (`%2e%2e%2f`, double encoding 등). 권장 X.
- canonical path 비교 — Java 의 표준.

### 출처
OWASP Path Traversal Cheat Sheet. CWE-22.

---

## B9. SHA-256 + base64url

### 무엇인가
- **SHA-256**: 256-bit cryptographic hash. 콘텐츠 무결성 + 식별.
- **base64url**: URL/Cookie 안전한 인코딩 (`+/=` 대신 `-_`).

### 어디서 쓰나

#### Sha256 (콘텐츠 dedupe)
```java
// shared/kernel/Sha256.java
public static Sha256 ofBytes(byte[] bytes) {
    try {
        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        byte[] hash = digest.digest(bytes);
        return new Sha256(HexFormat.of().formatHex(hash));     // → "a3f5b8..."
    } catch (NoSuchAlgorithmException e) {
        throw new IllegalStateException("SHA-256 not available", e);
    }
}
```

```java
// StoreUploadService.store
Sha256 sha = Sha256.ofBytes(input.content());
Optional<Upload> existing = repository.findByOrgAndSha256(input.orgId(), sha);
if (existing.isPresent()) return existing.get();    // dedupe!
```

#### base64url (token 인코딩)
```java
// TokenService
private static String base64Url(byte[] bytes) {
    return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
}

// payload + sig 구조
return payloadB64 + "." + base64Url(sig);    // "eyJhbGc...".sig...
```

### 왜 SHA-256?

**해시 함수 의 두 용도**:
1. **콘텐츠 식별 (content-addressable)**: 같은 내용 → 같은 해시 → dedupe.
2. **무결성 검증 (integrity)**: 전송 후 해시 재계산 → 일치하면 변조 X.
3. **HMAC 의 base**: 서명에 사용.

SHA-256 은:
- 256 bit 출력 → 충돌 확률 사실상 0 (생일 공격 2^128).
- JCA 표준 (`MessageDigest.getInstance("SHA-256")`) → 외부 라이브러리 X.
- 빠름 (CPU SHA-NI 명령 hardware 가속).

### 왜 base64url?

**일반 base64**: `+/=` 문자 사용.
- `+` → URL 에서 space 로 해석 (`%2B` 인코딩 필요).
- `/` → URL path separator.
- `=` → URL query value 의 padding 충돌.
→ URL/Cookie 에 그대로 못 씀.

**base64url**: `+` → `-`, `/` → `_`, `=` 제거 (no padding).
→ URL/Cookie 안전. JWT 표준.

### 핵심 아이디어
- **content-addressable storage**: 데이터의 식별자가 데이터 자체의 해시. Git, IPFS 같은 시스템의 기반.
- **base64url 은 JWT 의 컨벤션**. RFC 7515 §2.

### 전문 용어 사전
- **cryptographic hash**: 단방향 함수. 입력 → 고정 길이 해시. 같은 입력 → 같은 해시. 다른 입력 → (사실상) 다른 해시.
- **collision**: 다른 두 입력이 같은 해시 — 공격 (collision attack). SHA-256 은 현재 안전.
- **MAC (Message Authentication Code)**: 메시지 + secret → 서명. HMAC 는 hash 기반 MAC.
- **HexFormat**: Java 17+ 표준 hex 인코딩. `byte[] → String "ab12cd..."`.

### 함정
- **password 해싱에 SHA-256 직접 사용 X** — 빠르므로 brute force 쉬움. bcrypt/argon2 사용. SHA-256 은 콘텐츠 해시 + HMAC 용도.
- base64 != 암호화 — 단순 인코딩. 누구나 디코딩 가능.

### 대안
- MD5/SHA-1 — 충돌 발견됨. 보안 용도 X. 빠른 checksum 만.
- BLAKE2/BLAKE3 — SHA-256 보다 빠름. Java 표준 X.
- xxHash — 비암호화 빠른 해시. 보안 용도 X.

### 출처
- SHA-256: NIST FIPS 180-4 (Secure Hash Standard).
- base64url: RFC 4648 §5.
- HexFormat: JEP 356 (Java 17).

---

## §∞. 정리

### 패턴 매트릭스

| 문제 | 패턴 |
|---|---|
| O(N) 검색 → O(1) | B1 two-index repository |
| 여러 필드 키 | B2 nested record (DedupeKey, UniqueKey) |
| 도메인 타입 vs Map 키 | B3 raw UUID 추출 |
| null 가능 반환 | B4 Optional.ofNullable |
| 외부 list 변경 격리 | B5 List.copyOf |
| in-memory 메모리 보호 | B6 retention (size + time) |
| HTTP header case-insensitive | B7 lowercase 정규화 |
| 파일 path traversal | B8 normalize + startsWith |
| 콘텐츠 dedupe / token 인코딩 | B9 SHA-256 + base64url |

### 학습 추천 순서

1. **§0 인덱싱 의 의미** — RDBMS 안 써도 의미 있음.
2. **B4 Optional** — 매일 쓰는 패턴.
3. **B1/B2 two-index + nested record** — in-memory 의 핵심.
4. **B5 defensive copy** — 불변의 진짜 의미.
5. **B8 path traversal** — 보안 기본.
6. **B6 retention** — 큐 메모리 관리.
7. **B7/B9 정규화 + 해시** — 표준 친숙.

### 추가 참고
- Effective Java (3rd ed) — Item 17 (Minimize mutability), Item 50 (defensive copy).
- OWASP Cheat Sheets (Path Traversal, Cryptographic Storage).
- RFC 7230 (HTTP/1.1), RFC 4648 (base64).
