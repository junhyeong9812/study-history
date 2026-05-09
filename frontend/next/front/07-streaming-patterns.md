# 07. 실시간 스트리밍 패턴

> 이 문서가 다루는 것: EventSource, useEffect cleanup, ref-based dedupe, automatic reconnection — 실시간 데이터 흐름의 본질.

---

## §0. real-time 의 다양한 접근

| 방식 | 방향 | 라이브러리 | 사용처 |
|---|---|---|---|
| **Polling** | 단방향 | fetch | 단순. 간격 trade-off. |
| **Long-polling** | 단방향 | fetch with hang | 즉시성 + 단순. 비효율. |
| **SSE** | 단방향 (server→client) | EventSource | push 알림, 진행 표시. |
| **WebSocket** | 양방향 | WebSocket / Socket.IO | 채팅, 게임. |

VulnScope = SSE (live scan progress).

이 doc 은 **SSE 를 React hook 으로 통합하는 패턴** 의 본질.

---

## §1. EventSource — 브라우저 native SSE 클라이언트

### 1.1 기본 사용
```typescript
const es = new EventSource("/api/scans/123/stream", { withCredentials: true });

es.addEventListener("finding", (event) => {
    const data = JSON.parse(event.data);
    console.log(data, event.lastEventId);
});

es.onopen = () => console.log("connected");
es.onerror = () => console.log("disconnected");

// cleanup
es.close();
```

### 1.2 자동 동작
**EventSource 가 무료로 해주는 것**:
1. **자동 재연결** — 끊기면 지수 백오프로 재시도.
2. **Last-Event-ID 자동 첨부** — 마지막 받은 `id:` 를 헤더로 보냄.
3. **표준 wire format 파싱** — `id:`, `event:`, `data:` 구조 자동.

**naive (fetch streaming)**:
- 재연결 직접 구현.
- Last-Event-ID 직접 추적.
- chunk 파싱 직접.

→ EventSource 가 SSE 의 표준 클라이언트.

### 1.3 한계
- **헤더 추가 불가** (`withCredentials` 외) — Bearer token 헤더 못 보냄.
- **POST 불가** — GET 만.
- VulnScope 는 cookie 인증 → 헤더 추가 불필요. OK.

대안: `@microsoft/fetch-event-source` — fetch 위에 SSE. 헤더 자유.

---

## §2. React hook 으로 wrap (useScanStream)

### 2.1 코드
```typescript
"use client";

import { useEffect, useRef, useState } from "react";

export type StreamEventType = "phase" | "finding" | "done" | "failed" | "log";

export interface StreamEvent {
    type: StreamEventType;
    seq: number;
    payload: unknown;
}

export interface ScanStreamState {
    connected: boolean;
    events: StreamEvent[];
    lastSeq: number;
    done: boolean;
    failed: boolean;
}

export function useScanStream(scanId: string): ScanStreamState {
    const [connected, setConnected] = useState(false);
    const [events, setEvents] = useState<StreamEvent[]>([]);
    const [done, setDone] = useState(false);
    const [failed, setFailed] = useState(false);
    const lastSeqRef = useRef(0);                                 // (1) ref!

    useEffect(() => {                                              // (2) lifecycle
        const es = new EventSource(`/api/scans/${scanId}/stream`, {
            withCredentials: true,
        });

        const append = (type: StreamEventType, raw: MessageEvent) => {
            const seq = Number(raw.lastEventId) || 0;
            if (seq > 0 && seq <= lastSeqRef.current) return;     // (3) dedupe
            lastSeqRef.current = Math.max(lastSeqRef.current, seq);
            let payload: unknown = raw.data;
            try { payload = JSON.parse(raw.data); } catch {}
            setEvents((prev) => [...prev, { type, seq, payload }]);
            if (type === "done") setDone(true);
            if (type === "failed") setFailed(true);
        };

        es.addEventListener("phase", (e) => append("phase", e as MessageEvent));
        es.addEventListener("finding", (e) => append("finding", e as MessageEvent));
        es.addEventListener("done", (e) => append("done", e as MessageEvent));
        es.addEventListener("failed", (e) => append("failed", e as MessageEvent));
        es.addEventListener("log", (e) => append("log", e as MessageEvent));

        es.onopen = () => setConnected(true);
        es.onerror = () => setConnected(false);

        return () => {
            es.close();                                            // (4) cleanup
        };
    }, [scanId]);

    return { connected, events, lastSeq: lastSeqRef.current, done, failed };
}
```

### 2.2 핵심 패턴 4가지
1. **useRef for dedupe** (`lastSeqRef`) — re-render 안 트리거하면서 최신 값 유지.
2. **useEffect for setup** — mount 시 EventSource 생성, dependency `[scanId]` 변경 시 재설정.
3. **dedupe logic** — `seq <= lastSeq` 면 무시. 재연결 backfill 의 중복 방지.
4. **cleanup return** — unmount 시 `es.close()`. memory leak 방지.

---

## §3. Why useRef for dedupe? (중요)

### 3.1 만약 useState 였다면
```typescript
const [lastSeq, setLastSeq] = useState(0);

useEffect(() => {
    es.addEventListener("finding", (e) => {
        const seq = Number(e.lastEventId);
        if (seq <= lastSeq) return;        // ← BUG! lastSeq 가 stale
        setLastSeq(seq);
    });
    return () => es.close();
}, [scanId]);                              // lastSeq 빠짐
```

**문제**:
- listener closure 가 mount 시점의 `lastSeq` (= 0) capture.
- setLastSeq 호출해도 listener 안의 lastSeq 는 여전히 0.
- 모든 이벤트가 dedupe 우회.

### 3.2 deps 에 lastSeq 추가?
```typescript
useEffect(() => { ... }, [scanId, lastSeq]);
```
- 매 lastSeq 변경 시 useEffect 재실행 → EventSource 재생성 → 재연결 → 모든 backfill 다시.
- 무한 loop / 비효율.

### 3.3 useRef 가 정답
```typescript
const lastSeqRef = useRef(0);

useEffect(() => {
    es.addEventListener("finding", (e) => {
        const seq = Number(e.lastEventId);
        if (seq <= lastSeqRef.current) return;    // ← 항상 최신!
        lastSeqRef.current = Math.max(...);
    });
}, [scanId]);    // lastSeqRef 는 deps 에 없어도 됨 (불변)
```

**아이디어**:
- `lastSeqRef.current` 는 동일 객체 reference. listener 가 그걸 read.
- 매번 최신 값.
- deps 에 안 들어감 → useEffect 재실행 X.

### 3.4 핵심 통찰
**closure-trapped state vs mutable container**.
- state: render 마다 새 값 → closure 안에 stale.
- ref: 매번 같은 객체 → 안에 mutable. 항상 최신.

**규칙**:
- **렌더링에 필요한 값** = useState (변경 시 re-render 보장).
- **로직 안에서만 최신 값 필요** = useRef (re-render 무관).

---

## §4. useEffect cleanup — memory leak 방지

### 4.1 cleanup 의 중요성
```typescript
return () => {
    es.close();
};
```

**이거 빠뜨리면?**:
- `useScanStream` 사용 컴포넌트 unmount → 다음 마운트 → 새 EventSource.
- 옛 EventSource 안 닫힘 → 백그라운드에서 계속 listen.
- subscriber 누적 → memory leak + 네트워크 리소스 점유.

### 4.2 cleanup 시점
- unmount 시.
- deps 변경 시 이전 effect cleanup (새 effect 실행 전).

```typescript
useEffect(() => {
    const es = new EventSource(`/api/scans/${scanId}/stream`);
    // ...
    return () => es.close();
}, [scanId]);

// scanId 가 "A" → "B" 변경 시:
// 1. cleanup ("A" 의 EventSource close)
// 2. 새 effect ("B" 의 EventSource open)
```

### 4.3 핵심 통찰
**setup → cleanup 짝**. resource 라이프사이클 명시.

useEffect 가 React 의 lifecycle 통합.

### 4.4 함정
- cleanup 안에서 async 안 됨 (return 함수가 sync 만).
- 비동기 cancel 필요하면 AbortController 패턴.

---

## §5. Stale-while-error reconnect

### 5.1 EventSource 의 자동 재연결
```typescript
es.onerror = () => setConnected(false);
```

`onerror` 가 호출되어도 EventSource 는 **자동으로 재연결 시도**. close() 호출 안 하면 계속 재시도.

### 5.2 끊김 흐름
1. 네트워크 끊김 → server 가 connection close.
2. EventSource onerror 트리거 → state 업데이트 (UI: "DISCONNECTED").
3. EventSource 가 자동 재시도 (지수 백오프, 보통 3초).
4. 성공 → onopen → state 업데이트 (UI: "CONNECTED" 또는 "RUNNING").
5. 재연결 시 Last-Event-ID 자동 첨부 → 서버가 backfill.

### 5.3 핵심 통찰
**EventSource 의 자동 재연결 + Last-Event-ID = 무손실 streaming**. retention window 안에 있으면 누락 0.

---

## §6. SSE 가 막히는 환경 — polling fallback

### 6.1 corporate proxy 의 buffering
일부 proxy 가 response 를 buffer → SSE 가 멈춤. 해결:
- 서버 nginx 에 `proxy_buffering off`.
- 클라이언트 fallback (polling).

### 6.2 VulnScope 의 fallback endpoint
```typescript
// /api/scans/{scanId}/findings?since=N
const findings = await listFindings(scanId, since);
```

→ 클라이언트가 "마지막 받은 seq" 기억하고 polling.

### 6.3 핵심 통찰
**SSE + polling 둘 다 제공**. 환경 호환성.

VulnScope 는 SSE 우선, polling 은 별도 endpoint 로 가능.

---

## §7. 자동 navigate (done/failed 후)

### 7.1 useEffect 로 navigate
```typescript
useEffect(() => {
    if (done || failed) {
        const id = setTimeout(() => router.push(`/scans/${scanId}`), 1500);
        return () => clearTimeout(id);
    }
}, [done, failed, router, scanId]);
```

### 7.2 핵심 패턴
- 외부 state (done/failed) 감지 → side effect (navigate).
- setTimeout 로 1.5초 지연 (사용자가 "done" 메시지 보게).
- cleanup 으로 timer cancel (중복 navigate 방지).

### 7.3 통찰
**state 변화 → side effect** 는 useEffect 의 표준 패턴. cleanup 이 중복 효과 방지.

---

## §8. 패턴 매트릭스

| 문제 | 패턴 |
|---|---|
| SSE wire format 파싱 + 자동 재연결 | EventSource native |
| React lifecycle 통합 | useEffect setup/cleanup |
| dedupe (재연결 backfill 중복) | useRef + closure-safe |
| state 변화 → side effect | useEffect with deps |
| memory leak 방지 | cleanup 함수 |
| 환경 호환 | polling fallback |
| 자동 navigate (terminal 상태 후) | useEffect + setTimeout + cleanup |

---

## §9. 학습 포인트

1. **EventSource = native SSE**. 재연결 + Last-Event-ID 자동.
2. **헤더 추가 불가** — cookie 환경에 OK. JWT 면 fetch-event-source.
3. **closure-trapped state** — listener 안의 state 가 stale.
4. **useRef = mutable container** — 항상 최신, re-render 무관.
5. **cleanup 짝** — setup → cleanup. memory leak 방지.
6. **deps 변경 시 cleanup → 새 effect** — sequence 보장.
7. **자동 재연결 + Last-Event-ID = 무손실**.
8. **polling fallback** — corporate proxy 환경.
9. **자동 navigate** — useEffect + setTimeout + cleanup.
10. **state vs ref** = 렌더링 필요 vs 로직 만 필요.

### 추가 참고
- HTML5 SSE: https://html.spec.whatwg.org/multipage/server-sent-events.html
- 본 프로젝트 SSE 학습: `study/sse-libraries.md`
- React useEffect 가이드: https://overreacted.io/a-complete-guide-to-useeffect/
