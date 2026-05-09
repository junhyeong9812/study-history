# 10. Browser Storage + Network

> 이 문서가 다루는 것: localStorage / sessionStorage / IndexedDB / Cache API / Cookies + fetch + CORS + HTTP caching.

---

## §0. 왜 알아야?

- 데이터 어디 저장할지 (UX, 보안).
- API 호출의 정확한 동작 (CORS, preflight).
- HTTP cache 활용.

---

## §1. Storage 옵션 비교

| | localStorage | sessionStorage | Cookie | IndexedDB | Cache API |
|---|---|---|---|---|---|
| 크기 | 5MB | 5MB | 4KB | 무제한 (수GB) | 무제한 |
| 형식 | string | string | string | object | Response |
| 만료 | 영구 | 탭 닫힘 | 명시 | 명시 | 명시 |
| 서버로 자동 전송 | X | X | ✅ | X | X |
| 동기/비동기 | sync | sync | sync | async | async |
| 용도 | 작은 설정 | 임시 (탭) | 인증/세션 | 큰 데이터 | offline cache |

---

## §2. localStorage / sessionStorage

### 2.1 사용
```javascript
localStorage.setItem("theme", "dark");
const theme = localStorage.getItem("theme");
localStorage.removeItem("theme");
localStorage.clear();
```

→ string 만. object 는 JSON.stringify/parse.

### 2.2 sync = main thread 차단
- 큰 데이터 read/write = freeze.
- 5MB 풀 차면 throw.

### 2.3 origin scope
- 같은 protocol + host + port 만 공유.
- subdomain 격리 (`api.x.com` 와 `x.com` 다름).

### 2.4 storage event
다른 탭/윈도우에서 localStorage 변경 시 trigger:
```javascript
window.addEventListener("storage", (e) => {
    console.log(e.key, e.oldValue, e.newValue);
});
```

→ 같은 tab 의 변경엔 안 trigger. 다른 tab 만.

### 2.5 sessionStorage
- 같지만 탭 단위 격리.
- 탭 닫으면 사라짐.
- form draft 등에.

### 2.6 함정
- private browsing 에서 X 또는 in-memory.
- string 만 → object 매번 JSON.

VulnScope 미사용 (cookie 기반 세션).

---

## §3. Cookie

### 3.1 특징
- HTTP 표준. 서버 ↔ 클라 자동.
- 매 요청에 자동 첨부 (도메인/path 일치).
- 4KB 제한.

### 3.2 attributes
```
Set-Cookie: session=abc; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=86400
```

- **HttpOnly** — JS 접근 X (`document.cookie` 안 보임). XSS 방어.
- **Secure** — HTTPS 만.
- **SameSite** — `Strict` / `Lax` / `None`. CSRF 방어.
- **Path / Domain** — scope.
- **Max-Age / Expires** — 만료.

### 3.3 SameSite 의미
- `Strict` — 모든 cross-site 요청 cookie 안 보냄. 가장 엄격.
- `Lax` — top-level GET 만 허용 (link 클릭). cross-site POST 차단.
- `None` — 모두 허용 (Secure 필수).

VulnScope 의 `vulnscope_session` cookie:
```
HttpOnly + SameSite=Lax + Path=/
```

→ XSS 방어 + CSRF 1차 방어.

### 3.4 JS 에서 cookie 읽기/쓰기
```javascript
document.cookie = "key=value; path=/";
console.log(document.cookie);    // "key=value; other=..."  (HttpOnly 제외)
```

→ 매우 원시적 API. 라이브러리 권장 (`js-cookie` 등).

VulnScope 는 cookie 직접 안 만짐 — 백엔드/BFF 가 Set-Cookie 응답.

### 3.5 보안 patterns 는 H 카테고리 + study/sse-libraries 참조.

---

## §4. IndexedDB

### 4.1 정의
브라우저 안의 NoSQL DB. async + 큰 데이터 + structured.

### 4.2 raw API 어려움
```javascript
const request = indexedDB.open("myDB", 1);
request.onsuccess = (e) => {
    const db = e.target.result;
    const tx = db.transaction("users", "readwrite");
    const store = tx.objectStore("users");
    store.add({ id: 1, name: "Alice" });
};
```

→ event-based + verbose. 실제론 `idb` 같은 wrapper.

### 4.3 활용
- 큰 데이터 (수MB+).
- offline-first apps.
- 복잡한 query.

### 4.4 라이브러리
- `idb` — Promise wrapper.
- `Dexie` — high-level ORM.
- `localForage` — localStorage-like API.

VulnScope 미사용 (서버 의존).

---

## §5. Cache API (Service Worker)

### 5.1 정의
HTTP Response 객체 자체를 cache. service worker 와 짝.

```javascript
const cache = await caches.open("v1");
await cache.put(request, response);
const cached = await cache.match(request);
```

### 5.2 활용
- offline PWA.
- network-first / cache-first / stale-while-revalidate 전략.

VulnScope 미사용 (PWA 아님).

---

## §6. fetch API — modern HTTP 클라

### 6.1 기본
```javascript
const res = await fetch("/api/scans", {
    method: "GET",
    headers: { "Content-Type": "application/json" },
    credentials: "include",          // cookie 첨부
    body: JSON.stringify(data),       // POST 등
});

if (!res.ok) throw new Error(`status ${res.status}`);
const data = await res.json();
```

### 6.2 credentials
- `"omit"` — cookie 안 보냄.
- `"same-origin"` — 같은 origin 만.
- `"include"` — 항상 (cross-origin 도). VulnScope 의 default.

### 6.3 Response 메서드
- `.text()` / `.json()` / `.blob()` / `.arrayBuffer()` / `.formData()`.
- 한 번만 호출 가능 (stream consume).

### 6.4 streaming response
```javascript
const reader = res.body.getReader();
while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    // value = Uint8Array chunk
}
```

→ SSE / 큰 다운로드.

### 6.5 AbortController
```javascript
const controller = new AbortController();
const promise = fetch(url, { signal: controller.signal });
controller.abort();    // 취소
```

→ React 의 useEffect cleanup 에서.

### 6.6 vs XHR / axios
- XHR — old. event 기반.
- axios — 라이브러리. interceptor 풍부. 무거움.
- fetch — modern, native, 가벼움. interceptor 없음 (직접).

VulnScope = fetch + openapi-fetch.

---

## §7. CORS

### 7.1 same-origin policy
브라우저: 다른 origin 의 응답을 JS 가 read 못 함 (보안).

origin = protocol + host + port.

```
https://app.com  vs  https://api.com   ← cross-origin
https://app.com  vs  http://app.com    ← cross-origin (protocol)
https://app.com  vs  https://app.com:8080  ← cross-origin (port)
```

### 7.2 CORS 가 해결
서버가 응답 헤더에 "허용" 명시:
```
Access-Control-Allow-Origin: https://app.com
Access-Control-Allow-Credentials: true
```

→ 브라우저가 JS 에 응답 노출.

### 7.3 simple request
- GET / HEAD / POST.
- 기본 헤더만 (Accept, Content-Type:text/plain 등).
- → preflight 없음. 바로 request.

### 7.4 preflight (OPTIONS)
- 위 외 (DELETE, PUT, custom header) → preflight 먼저.
- 브라우저가 OPTIONS 요청 → 서버가 Access-Control-Allow-Methods/Headers 응답.
- 통과해야 실제 request.

### 7.5 credentials + CORS
- `credentials: "include"` → preflight 도 cookie 보냄.
- 서버 응답에 `Access-Control-Allow-Credentials: true` 필요.
- `Access-Control-Allow-Origin: *` 와 credentials 동시 X (specific origin 명시).

### 7.6 CORS 회피 = BFF
VulnScope: frontend 와 백엔드 origin 다름 → BFF (Next.js) 가 same-origin proxy. CORS 회피.

---

## §8. HTTP Caching

### 8.1 Cache-Control
```
Cache-Control: max-age=3600, public
```

- `max-age=N` — N 초 동안 fresh.
- `public` / `private` — CDN / browser only.
- `no-cache` — 매번 validate (ETag 사용).
- `no-store` — 캐시 X.

### 8.2 ETag (validation)
서버:
```
ETag: "abc123"
```
다음 요청 시 클라:
```
If-None-Match: "abc123"
```
- 같으면 → 304 Not Modified (body 없음).
- 다르면 → 200 + 새 body.

VulnScope 의 `/uploads/{id}/download` 가 sha256 = ETag.

### 8.3 Last-Modified
```
Last-Modified: Mon, 01 May 2026 ...
If-Modified-Since: Mon, 01 May 2026 ...
```

ETag 보다 덜 정확 (초 단위).

### 8.4 cache 우선순위
1. memory cache (탭 안).
2. disk cache.
3. service worker.
4. HTTP server.

→ 빠른 단계부터.

---

## §9. WebSocket

### 9.1 정의
양방향 long-lived connection. text + binary.

```javascript
const ws = new WebSocket("wss://example.com/socket");
ws.onmessage = (e) => console.log(e.data);
ws.send("hello");
ws.close();
```

### 9.2 vs SSE
- SSE: 단방향 (server → client). HTTP 기반. 자동 재연결.
- WebSocket: 양방향. 별도 protocol (ws://). 재연결 직접.

VulnScope = SSE. WebSocket 미사용.

---

## §10. 학습 포인트

1. **storage 5종** — 크기/만료/sync/scope 다름.
2. **localStorage = sync, 5MB**. 큰 데이터 X.
3. **Cookie = 자동 전송**. HttpOnly + SameSite 보안.
4. **IndexedDB = async, 큰 데이터**. 라이브러리 (idb, Dexie).
5. **Cache API = Response 자체 cache**. PWA 용.
6. **fetch credentials 3종** — omit/same-origin/include.
7. **AbortController = fetch cancel**. useEffect cleanup.
8. **CORS = 다른 origin 응답 허용**. preflight (OPTIONS).
9. **BFF = CORS 회피** (same-origin proxy).
10. **HTTP cache = max-age + ETag**. 304 Not Modified.

### 추가 참고
- MDN Storage: https://developer.mozilla.org/en-US/docs/Web/API/Web_Storage_API
- MDN CORS: https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
- HTTP Caching: https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching
