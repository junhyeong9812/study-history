# SSE (Server-Sent Events) 라이브러리 정리

> 작성일: 2026-05-02
> 목적: SSE 표준 + 백엔드(Java/Spring) / 프론트엔드(브라우저/Node) 의 사용 가능한 라이브러리/API 종합. VulnScope 가 무엇을 골랐고 왜인지 + 대안들.

---

## 0. SSE 표준 자체

**HTML5 Server-Sent Events** — W3C/WHATWG 표준.

### Wire format (text/event-stream)
```
id: 5
event: finding
data: {"findingId":"...","severity":"HIGH"}

id: 6
event: phase
data: {"phase":"checking"}

```

- `id:` — 이벤트 ID. 클라가 끊겼다 재연결 시 `Last-Event-ID` 헤더로 자동 보냄.
- `event:` — 이벤트 타입. 클라가 `addEventListener("phase", ...)` 로 매칭.
- `data:` — 페이로드 (텍스트). 여러 줄 가능 (`\n` 으로 join).
- `:` 로 시작하면 주석.
- `retry: 3000` — 재연결 간격 (ms) 힌트.
- 이벤트 구분자 = **빈 줄**.

### HTTP 요구사항
- response: `Content-Type: text/event-stream`
- `Cache-Control: no-cache`
- `Connection: keep-alive`
- HTTP/1.1 chunked 또는 HTTP/2 stream.

### 자동 동작 (브라우저 EventSource)
- 연결 끊기면 자동 재시도.
- 마지막 받은 `id:` 를 `Last-Event-ID` 헤더로 자동 첨부 → 서버가 resume 가능.
- JSON 직접 파싱 안 함 (data 는 raw string. 클라가 `JSON.parse`).

### 한계
- **단방향만** (server→client). 클라→서버는 별도 fetch.
- **HTTP/1.1 connection 6개 cap** (도메인당). 다중 SSE 시 HTTP/2 권장.
- WebSocket 보다 단순. 양방향 필요하면 WS.

---

## 1. 백엔드 (Java/Spring) — 라이브러리 옵션

### 1.1 Spring Web MVC `SseEmitter` ★ VulnScope 가 사용 중

**위치**: `org.springframework.web.servlet.mvc.method.annotation.SseEmitter` (Spring Web).
**사용처**: `ScanStreamController.stream`.

```java
@RestController
@RequestMapping("/scans")
public class ScanStreamController {

    @GetMapping(value = "/{id}/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter stream(@PathVariable UUID id,
                             @RequestHeader(name = "Last-Event-ID", required = false) Long lastEventId) {
        SseEmitter emitter = new SseEmitter(0L);  // timeout 0 = 무한
        // ... backfill + subscribe
        emitter.onCompletion(...);
        emitter.onTimeout(...);
        emitter.onError(...);
        return emitter;
    }

    private void send(SseEmitter emitter, StoredEvent event) {
        emitter.send(SseEmitter.event()
            .id(String.valueOf(event.seq()))
            .name(event.type())
            .data(event.payload()));     // Jackson 자동 직렬화
    }
}
```

**장점**:
- Spring Web MVC 표준. 추가 의존 0.
- 타임아웃/콜백 (onCompletion/onTimeout/onError) 자동 라이프사이클.
- Jackson 자동 직렬화.
- Last-Event-ID 헤더는 `@RequestHeader` 로 직접 받음.

**단점**:
- **Servlet thread 1개 점유** (Tomcat 일반 모드). 동시 SSE 수만큼 thread 필요.
- 가상 스레드 (Spring Boot 3.2+) 활성 시 thread 비용 거의 0 → 해결.
- 양방향 X (SSE 표준 한계).

**왜 선택**: Spring 표준 + 단순 + 가상 스레드 친화. WebFlux 도입 비용 회피.

### 1.2 Spring WebFlux `Flux<ServerSentEvent<T>>` (대안)

**위치**: `org.springframework.web.reactive.*` (Spring WebFlux).

```java
@RestController
public class ScanStreamController {

    @GetMapping(value = "/scans/{id}/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<Object>> stream(@PathVariable UUID id) {
        return scanEventService.streamEvents(id)
            .map(event -> ServerSentEvent.<Object>builder()
                .id(String.valueOf(event.seq()))
                .event(event.type())
                .data(event.payload())
                .build());
    }
}
```

**장점**:
- Reactive — 진짜 non-blocking. Netty 기반.
- backpressure 지원 (subscriber 가 slow 면 producer 가 throttle).
- 무한 스트림 자연 표현.

**단점**:
- WebFlux 도입 시 **전체 스택 reactive 화** 권장 (controller 만 일부 도입은 어색).
- Spring MVC + JPA 같은 blocking 코드와 mix 어려움.
- 학습 곡선 (Reactor — Mono/Flux/operators).
- 가상 스레드가 도입된 후 WebFlux 의 가치 일부 감소.

**왜 안 선택**: VulnScope 는 MVC + JPA(준비) + 가상 스레드 — Reactive 까지 갈 이유 약함.

### 1.3 Spring Modulith Events (`@ApplicationModuleListener`)

**역할**: 직접 SSE 생성 X. **도메인 이벤트 → SSE 게이트웨이** 의 다리 (outbox 기반).

```java
@ApplicationModuleListener  // = @Async + @TransactionalEventListener(AFTER_COMMIT) + outbox
void onFindingRecorded(FindingRecorded event) {
    realtimeChannel.emit(channel, "finding", payload);
}
```

**장점**: at-least-once + 트랜잭션 영속 보장. event publisher → outbox → dispatcher → SSE.
**단점**: outbox 테이블 + dispatcher 워커 추가.
**참고**: `study/domain-events-and-outbox.md`. VulnScope v0.1 은 미사용 (WorkerExecutor 가 직접 emit).

### 1.4 Servlet 직접 (low-level)

**언제**: Spring 안 쓰는 환경 또는 ultra-fine 제어.

```java
@WebServlet("/sse")
public class RawSseServlet extends HttpServlet {
    protected void doGet(HttpServletRequest req, HttpServletResponse res) throws IOException {
        res.setContentType("text/event-stream");
        res.setCharacterEncoding("UTF-8");
        AsyncContext async = req.startAsync();
        async.setTimeout(0);
        PrintWriter writer = res.getWriter();
        // 직접 "id: ...\nevent: ...\ndata: ...\n\n" 작성
    }
}
```

**장점**: 의존 0.
**단점**: 모든 wire format 직접. 테스트 어려움. Spring 의 SseEmitter 가 같은 일을 헬퍼로 제공.

### 1.5 Project Reactor + Sinks (양방향 fan-out)

**언제**: 한 sink 에 publish → 여러 SSE 클라이언트 fan-out.

```java
private final Sinks.Many<StoredEvent> sink = Sinks.many().multicast().onBackpressureBuffer();

// publish
sink.tryEmitNext(event);

// SSE endpoint
@GetMapping(produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<StoredEvent> stream() {
    return sink.asFlux();
}
```

**장점**: pub/sub 패턴이 라이브러리 차원에서.
**단점**: WebFlux 와 짝. Spring MVC 와 mix 어색.

### 1.6 외부 라이브러리

- **`okhttp-sse`** — 클라이언트 측 SSE. 서버 X.
- **`oasis-open/sse-eventsource`** — 자바 클라용. 서버 X.
- 백엔드 서버용 **전용 SSE 라이브러리** 는 사실상 없음 — Spring 의 SseEmitter / Flux 가 표준.

---

## 2. 프론트엔드 (브라우저) — 라이브러리 옵션

### 2.1 Browser native `EventSource` ★ VulnScope 가 사용 중

**위치**: 브라우저 빌트인. 의존 0.
**사용처**: `useScanStream.ts`.

```typescript
const es = new EventSource("/api/scans/123/stream", { withCredentials: true });

es.addEventListener("finding", (e) => {
    const data = JSON.parse(e.data);
    const seq = Number(e.lastEventId);
    // ...
});

es.onopen = () => setConnected(true);
es.onerror = () => setConnected(false);  // 자동 재연결 시도

// cleanup
es.close();
```

**장점**:
- **자동 재연결** (지수 백오프).
- **자동 Last-Event-ID 첨부** — 재연결 시 마지막 받은 `id:` 를 헤더로 보냄.
- 표준 — 추가 의존 0.
- TypeScript 기본 타입 지원.

**단점**:
- **`withCredentials` 외 헤더 추가 불가** — 예: 커스텀 인증 토큰을 헤더로 못 보냄. cookie 기반 세션이라 OK 했지만 Bearer Token 환경엔 부적합.
- POST/PUT 불가 — GET 만.
- 일부 환경 (Node SSR, fetch interceptor 필요) 에서 한계.

**왜 선택**: VulnScope 는 BFF cookie 기반 세션 → 헤더 추가 불필요. 가장 단순.

### 2.2 `microsoft/fetch-event-source` ★ 인기 대안

**위치**: npm `@microsoft/fetch-event-source` (또는 fork 들).

```typescript
import { fetchEventSource } from '@microsoft/fetch-event-source';

await fetchEventSource('/api/sse', {
    method: 'POST',                       // ★ POST 가능!
    headers: {
        'Authorization': `Bearer ${token}`,  // ★ 커스텀 헤더 가능
        'Content-Type': 'application/json',
    },
    body: JSON.stringify({...}),
    onmessage(ev) { ... },
    onclose() { ... },
    onerror(err) { throw err; },           // throw 안 하면 자동 재시도
});
```

**장점**:
- `fetch` API 위에 구축 → **헤더/메서드/body 자유**.
- 재연결 정책 직접 제어 (onerror throw 여부).
- Bearer token 인증 SSE 가능.

**단점**:
- 의존 추가 (~10KB).
- 자동 재연결 정책 직접 작성.

**언제**: 헤더 인증 (Bearer/JWT) 필수 + SSE.

### 2.3 `eventsource` (Node 폴리필)

**위치**: npm `eventsource` (Node.js 환경 — SSR/test).

```typescript
import EventSource from 'eventsource';
const es = new EventSource(url, { headers: { 'Cookie': '...' } });
```

**장점**: Node 에서 SSE 클라이언트 가능 (브라우저 EventSource 가 없는 환경).
**단점**: Node 전용. 브라우저는 native 가 더 가벼움.
**언제**: SSR / proxy / 테스트 환경에서 SSE 소비.

### 2.4 `eventsource-parser`

**위치**: npm `eventsource-parser`. low-level parser only.

```typescript
import { createParser } from 'eventsource-parser';

const parser = createParser((event) => {
    if (event.type === 'event') {
        console.log(event.data, event.id);
    }
});

const response = await fetch('/api/sse');
const reader = response.body!.getReader();
while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    parser.feed(new TextDecoder().decode(value));
}
```

**장점**: parsing 만. 어떤 transport (fetch, WebSocket bridge) 와도 결합.
**단점**: 재연결/Last-Event-ID 직접 구현.
**언제**: ChatGPT 스트리밍 같은 SSE 응답 파싱.

### 2.5 `sse.js`

**위치**: npm `sse.js`. EventSource 의 POST 지원 변형.

**장점**: POST + body + 헤더 SSE. EventSource API 와 비슷.
**단점**: 유지보수 다소 stale. fetch-event-source 가 더 활발.

### 2.6 React Query + custom transport (수동 통합)

직접적인 SSE 라이브러리는 아니지만, useQuery 와 SSE 결합 패턴:

```typescript
function useScanStream(scanId: string) {
    const queryClient = useQueryClient();
    useEffect(() => {
        const es = new EventSource(`/api/scans/${scanId}/stream`);
        es.addEventListener("finding", (e) => {
            queryClient.setQueryData(["scan", scanId, "findings"], (old) => [...old, parsed]);
        });
        return () => es.close();
    }, [scanId, queryClient]);
}
```

**장점**: SSE 가 캐시 invalidate. UI 자동 업데이트.

---

## 3. 비교표 — VulnScope 의 결정

| | VulnScope 선택 | 대안 | 선택 이유 |
|---|---|---|---|
| 백엔드 SSE | Spring `SseEmitter` | WebFlux `Flux<ServerSentEvent>` | MVC + 가상 스레드로 충분. WebFlux 학습/통합 비용 회피. |
| 프론트 SSE | Browser `EventSource` | `@microsoft/fetch-event-source` | BFF cookie 인증 → 헤더 추가 불필요. native 의 자동 재연결 + Last-Event-ID 활용. |
| 발행 → SSE | WorkerExecutor 가 `realtime.emit` 직접 | Spring Modulith outbox listener | v0.1 단순함. v0.2 신뢰성 필요시 outbox. |

---

## 4. 실전 함정 (전 라이브러리 공통)

### 4.1 프록시/CDN 의 buffering
nginx 기본은 response buffering → SSE 가 멈춤. 설정 필요:
```nginx
location /api/ {
    proxy_buffering off;
    proxy_cache off;
    proxy_set_header Connection '';
    proxy_http_version 1.1;
    chunked_transfer_encoding on;
}
```
Cloudflare/CDN: SSE 지원 확인.

### 4.2 HTTP/2 권장
HTTP/1.1 의 도메인당 6 connection cap → 다중 SSE 시 부족. HTTP/2 면 multiplexing.

### 4.3 idle timeout
load balancer/proxy 의 idle timeout (기본 60초~5분) 보다 짧은 주기로 keep-alive 보내야 함:
```java
emitter.send(SseEmitter.event().comment("keepalive"));   // 30초마다
```
VulnScope 는 워커가 phase 마다 emit 하므로 자연 keep-alive.

### 4.4 message ordering vs concurrency
한 EventSource 의 event listener 는 single-threaded (브라우저). 백엔드 동시 publish 가 client 에 순서 보장 X (HTTP/1.1 는 보장, HTTP/2 도 stream 내 순서 보장). 시퀀스 (`id:`) 활용 권장.

### 4.5 EventSource 의 retry: 헤더
서버가 `retry: 3000` 발행하면 클라가 3초 간격 재연결. 명시 안 하면 브라우저 default (Chrome 3초).

### 4.6 발신자측 close 시 클라이언트 처리
서버가 emitter.complete() 호출하면 EventSource 가 자동 재연결 시도. 의도적 종료면 클라가 close() 명시.

---

## 5. SSE vs 대안 비교

### SSE vs WebSocket
| | SSE | WebSocket |
|---|---|---|
| 방향 | 단방향 (server→client) | 양방향 |
| 프로토콜 | HTTP (표준 GET) | ws:// (HTTP upgrade) |
| 자동 재연결 | ✅ (브라우저) | ❌ (직접 구현) |
| Last-Event-ID resume | ✅ | ❌ (직접) |
| 프록시/방화벽 친화 | ✅ (HTTP) | ⚠️ (일부 차단) |
| 메시지 형식 | 텍스트 | 텍스트 + binary |
| 라이브러리 | 표준 EventSource | 표준 WebSocket / socket.io |
| 사용처 | 알림/로그/진행 표시 | 채팅/게임/RTC |

→ 단방향 push 에는 **SSE 가 단순+안정**. VulnScope 의 scan progress 가 정확히 fit.

### SSE vs Polling (long-poll / short-poll)
| | SSE | Long-poll | Short-poll |
|---|---|---|---|
| 지연 | 즉시 | 즉시 | interval 만큼 |
| 서버 부하 | 낮음 (1 connection 유지) | 중 (요청마다 connection) | 높음 (반복 요청) |
| 클라이언트 코드 | 단순 (EventSource) | 중간 | 단순 |

→ VulnScope 는 SSE + polling fallback (`findByScanSince`) 둘 다 제공.

### SSE vs gRPC streaming
- gRPC: HTTP/2 + protobuf. 양방향 + binary + 강한 schema.
- SSE: HTTP + 텍스트. 단방향 + JSON.
- 브라우저 직접 gRPC = grpc-web (extra proxy 필요). SSE 는 native.

---

## 6. VulnScope 의 SSE 흐름 다이어그램

```
[Worker (Phase 07 WorkerExecutor)]
   │  realtime.emit(channel, "finding", payload)
   ▼
[RealtimeChannel.emit (facade)]
   ├─► Store.append(channel, ...) → seq 부여 + retention
   └─► Bus.publish(event)
         │
         ▼ (subscribers loop)
       [emitter.send(id=seq, event=type, data=payload)]
         │
         ▼ HTTP chunked response
       [Browser EventSource]
         │
         ▼ addEventListener("finding", ...)
       [useScanStream hook → setEvents]
         │
         ▼
       [ScanLiveDashboard render]


[Browser 끊김 후 재연결]
   │  GET /scans/.../stream + Last-Event-ID: <last seq>
   ▼
[ScanStreamController]
   ├─► Store.readSince(channel, lastSeq, 1000) → backfill emit
   └─► Bus.subscribe → 이후 라이브
```

---

## 7. 학습 포인트

1. **SSE = HTTP + text/event-stream**. 단순. WebSocket 보다 진입장벽 낮음.
2. **id: + event: + data: + 빈 줄** wire format.
3. **Last-Event-ID 자동 첨부** (브라우저 EventSource) — resume 의 핵심.
4. **Spring SseEmitter** = MVC 표준. 가상 스레드와 잘 맞음.
5. **WebFlux Flux<ServerSentEvent>** = Reactive 풀스택일 때.
6. **Native EventSource** = 가장 단순. cookie 인증 OK.
7. **fetch-event-source** = 헤더 자유 + POST 가능. JWT 환경.
8. **eventsource-parser** = parser only. fetch + manual stream 결합.
9. **proxy_buffering off** — nginx 등 SSE 멈춤 방지.
10. **idle timeout** 보다 짧게 keep-alive (`comment` event).

---

## 8. 추가 참고

- HTML Living Standard SSE: https://html.spec.whatwg.org/multipage/server-sent-events.html
- Spring Web MVC SseEmitter docs: https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-async.html#mvc-ann-async-sse
- Spring WebFlux SSE: https://docs.spring.io/spring-framework/reference/web/webflux/reactive-spring.html
- @microsoft/fetch-event-source: https://github.com/Azure/fetch-event-source
- eventsource-parser: https://github.com/rexxars/eventsource-parser
- 본 프로젝트의 SSE 학습: `docs/plans/2026-04-30/02-first-slice/learned/08-sse-gateway.md`
- 본 프로젝트의 EventSource hook: `docs/plans/2026-04-30/02-first-slice/learned/09e-scan-stream.md`
