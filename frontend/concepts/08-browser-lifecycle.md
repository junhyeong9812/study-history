# 08. Browser Page Lifecycle

> 이 문서가 다루는 것: DOMContentLoaded, load, beforeunload, Visibility API, Page Lifecycle API. 페이지가 로드/숨김/종료되는 과정.

---

## §0. 왜 알아야?

- 분석/저장 timing 을 정확히 (페이지 떠나기 전 데이터 보내기).
- 백그라운드 탭 처리 (timer 멈춤, 음악 일시정지).
- SPA navigation 과 다름 (full page navigation).

---

## §1. 페이지 로드 단계

### 1.1 timing
```
1. HTML 다운로드 시작
2. HTML parsing → DOM 생성
3. CSS / JS 파싱 + 적용
4. DOMContentLoaded ★         ← DOM 준비
5. images / iframes 로드
6. load ★                       ← 모든 자원 완료
```

### 1.2 DOMContentLoaded vs load
| | DOMContentLoaded | load |
|---|---|---|
| 시점 | DOM tree 완성 | 모든 sub-resource 완료 (이미지 등) |
| 빠르기 | 빠름 | 느림 |
| 사용 | DOM 조작 가능 | 이미지 size 등 필요 |

### 1.3 listen
```javascript
document.addEventListener("DOMContentLoaded", () => {
    console.log("DOM ready");
});

window.addEventListener("load", () => {
    console.log("All loaded");
});
```

### 1.4 React 와 무관 (대부분)
React 는 자체 lifecycle. DOMContentLoaded 직접 listen 거의 X.

단, third-party script 통합 시 활용.

---

## §2. beforeunload — 페이지 떠나기 직전

### 2.1 사용
```javascript
window.addEventListener("beforeunload", (e) => {
    if (hasUnsavedChanges) {
        e.preventDefault();
        e.returnValue = "";    // 일부 브라우저
    }
});
```

→ "정말 나가시겠습니까?" 다이얼로그 (브라우저 default 메시지).

### 2.2 함정
- 메시지 custom 불가 (브라우저 default 만).
- 너무 자주 사용 시 사용자 짜증.
- mobile 일부 미지원.

### 2.3 unload (deprecated)
- 페이지 진짜 떠난 후. async X (sync 만).
- 모바일 보통 안 호출 (background → 종료).
- → **사용 X**. visibilitychange + pagehide 권장.

---

## §3. Page Visibility API

### 3.1 정의
탭 보임/숨김 감지.

```javascript
document.addEventListener("visibilitychange", () => {
    if (document.hidden) {
        console.log("hidden — pause");
        // 음악 멈춤, timer 멈춤, websocket 끊기 등
    } else {
        console.log("visible — resume");
    }
});

console.log(document.visibilityState);   // "visible" | "hidden" | "prerender"
```

### 3.2 활용
- 백그라운드 탭의 timer 멈춤 (CPU 절약).
- 비디오 일시정지.
- WebSocket reconnect.
- TanStack Query 의 refetchOnWindowFocus 가 사용 (실은 focus event).

### 3.3 함정
- visibility 변경 시점에 fetch 가능 (page 가 닫히지 않음).
- pagehide / unload 와 다름 — visibility 는 hide 일 뿐.

---

## §4. Page Lifecycle API (modern)

### 4.1 5 state
- **active** — 보임 + focus.
- **passive** — 보임 + 다른 윈도우/탭 focus.
- **hidden** — 안 보임 (탭 숨김, 최소화).
- **frozen** — 백그라운드, 브라우저가 freeze (CPU/메모리 절약).
- **terminated** — 종료.

### 4.2 transition events
```javascript
document.addEventListener("freeze", () => { /* 자원 해제 */ });
document.addEventListener("resume", () => { /* 자원 재할당 */ });

window.addEventListener("pagehide", (e) => {
    if (e.persisted) {
        // bfcache 에 들어감 (back/forward 캐시)
    } else {
        // 진짜 종료 직전
    }
});

window.addEventListener("pageshow", (e) => {
    if (e.persisted) {
        // bfcache 에서 복원
    }
});
```

### 4.3 활용
- pagehide: 마지막 데이터 저장 (analytics, draft 등).
- freeze: WebSocket 끊기, timer 정리.
- pageshow + persisted: state 복원.

### 4.4 navigator.sendBeacon
unload 시점에 fetch 신뢰 X. sendBeacon 권장.

```javascript
window.addEventListener("pagehide", () => {
    navigator.sendBeacon("/api/analytics", JSON.stringify(data));
});
```

→ 비동기 + 페이지 떠나도 보장.

---

## §5. bfcache (back/forward cache)

### 5.1 정의
브라우저가 페이지 전체 (DOM + state + JS heap) 캐시. 뒤로가기/앞으로가기 시 즉시 복원 (no re-load).

### 5.2 disable 조건
- `unload` listener 있으면.
- WebSocket / IndexedDB transaction 열려 있으면.
- `Cache-Control: no-store`.

### 5.3 효과
- 즉시 복원 (UX 매우 빠름).
- but state 가 옛 거. fresh 필요하면 pageshow 에서 갱신.

### 5.4 listen
```javascript
window.addEventListener("pageshow", (e) => {
    if (e.persisted) {
        // bfcache 복원 — 데이터 새로고침 등
    }
});
```

---

## §6. SPA navigation 과 차이

### 6.1 traditional navigation
- URL 변경 → 서버 fetch → HTML 교체 → DOMContentLoaded / load 다시.

### 6.2 SPA (Next.js, React Router)
- URL 변경 → JS 가 컴포넌트 swap → DOM event 안 트리거.
- react-router 의 `useNavigate`, Next 의 `router.push` 가 history.pushState 사용.

→ DOMContentLoaded 등은 처음 1회만.

### 6.3 Next.js 의 navigation event
```typescript
// app/template.tsx — 매 navigation 마다 mount
export default function Template({ children }) {
    useEffect(() => {
        // navigation
    }, []);
    return <>{children}</>;
}
```

또는 Next.js 의 `usePathname`:
```typescript
const pathname = usePathname();
useEffect(() => {
    // pathname 변경
}, [pathname]);
```

---

## §7. document.readyState

### 7.1 값
- `"loading"` — DOM 아직 parsing 중.
- `"interactive"` — DOMContentLoaded 시점. DOM 준비.
- `"complete"` — load 시점. 모든 자원 완료.

### 7.2 사용
```javascript
if (document.readyState === "loading") {
    document.addEventListener("DOMContentLoaded", init);
} else {
    init();    // 이미 ready
}
```

---

## §8. iframe lifecycle

### 8.1 iframe 도 별도 페이지
- iframe 안의 document 는 자체 DOMContentLoaded / load.
- parent 의 load 는 모든 iframe load 후.

### 8.2 cross-origin iframe
- parent 가 iframe 안 접근 X (Same-origin policy).
- postMessage 로 통신.

VulnScope 는 iframe 미사용.

---

## §9. service worker 와 lifecycle

### 9.1 service worker 가 있다면
- 페이지 닫혀도 살아있음 (background).
- network request intercept.
- push notification.
- install / activate / fetch event.

VulnScope 미사용 (PWA 아님).

---

## §10. 학습 포인트

1. **DOMContentLoaded** = DOM 준비. **load** = 모든 자원.
2. **beforeunload** = 떠나기 직전 confirm. 신중히.
3. **Visibility API** = 탭 보임/숨김.
4. **Page Lifecycle API** = active/passive/hidden/frozen/terminated.
5. **navigator.sendBeacon** = 종료 시 데이터 보내기.
6. **bfcache** = 즉시 복원. unload listener 면 disable.
7. **SPA navigation** = DOMContentLoaded 트리거 X. 직접 router event.
8. **document.readyState** = 현재 단계 query.
9. **iframe = 별도 lifecycle**.
10. **service worker = 페이지 무관 lifecycle**.

### 추가 참고
- MDN Page Lifecycle: https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API
- web.dev, [Page Lifecycle API](https://developer.chrome.com/blog/page-lifecycle-api/)
- web.dev, [bfcache](https://web.dev/articles/bfcache)
