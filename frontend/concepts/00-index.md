# Frontend 기본 개념 학습서

> 작성일: 2026-05-02
> 대상: VulnScope 코드를 보면서 "이건 React 의 무엇이지?", "브라우저 안에서 어떻게 동작하지?" 묻게 되는 사람.
> 각 doc 가 독립 — 관심사대로 읽기.

---

## 카테고리 (13 docs)

| # | 파일 | 주제 | 키워드 |
|---|---|---|---|
| 00 | (이 파일) | Index | 학습 순서 + 카테고리 |
| 01 | [01-react-component-lifecycle.md](01-react-component-lifecycle.md) | React 컴포넌트 라이프사이클 | mount/update/unmount, Strict Mode, Concurrent rendering |
| 02 | [02-react-hooks-deep.md](02-react-hooks-deep.md) | Hooks 모두 깊이 | useState/useEffect/useRef/useMemo/useCallback/useContext/useReducer/useLayoutEffect/useTransition/useDeferredValue/useId/useSyncExternalStore/use |
| 03 | [03-react-reconciliation.md](03-react-reconciliation.md) | Reconciliation + Fiber | Virtual DOM, diffing, render phase, commit phase, key, scheduler |
| 04 | [04-javascript-core.md](04-javascript-core.md) | JavaScript 핵심 | closure, prototype, this binding, async/await, Promise, generator |
| 05 | [05-javascript-proxy-reflect.md](05-javascript-proxy-reflect.md) | Modern JS — Proxy, Reflect | Symbol, WeakMap/WeakSet, ESM vs CJS |
| 06 | [06-typescript-types.md](06-typescript-types.md) | TypeScript type system | generic, conditional/mapped/template literal type, narrowing, discriminated union, satisfies |
| 07 | [07-browser-event-loop.md](07-browser-event-loop.md) | Event Loop | microtask vs macrotask, requestAnimationFrame, async timing |
| 08 | [08-browser-lifecycle.md](08-browser-lifecycle.md) | Page Lifecycle | DOMContentLoaded, load, beforeunload, Visibility API, Page Lifecycle API |
| 09 | [09-browser-rendering.md](09-browser-rendering.md) | Critical Rendering Path | parse, layout, paint, composite, layer, GPU, repaint vs reflow |
| 10 | [10-browser-storage-network.md](10-browser-storage-network.md) | Storage + Network | localStorage/sessionStorage/IndexedDB/Cache API/Cookies + fetch + CORS + HTTP caching |
| 11 | [11-browser-observers.md](11-browser-observers.md) | Web Observers | Intersection/Mutation/Resize Observer + MediaQueryList |
| 12 | [12-performance-web-vitals.md](12-performance-web-vitals.md) | Performance | LCP, INP, CLS, code splitting, lazy loading, React 최적화 |

---

## 학습 순서 추천

### 처음 React 인 사람
**01 → 02 → 03 → 07 → 09 → 나머지**
- 01/02/03: React 의 본질.
- 07: 비동기 timing 의 기본.
- 09: 브라우저가 화면 그리는 과정.

### React 알지만 깊이 모르는 사람
**03 (Reconciliation) → 02 (hooks deep) → 07 (event loop) → 11 (observers) → 12 (perf)**

### 브라우저 동작 궁금한 사람
**07 → 08 → 09 → 10 → 11**

### 인터뷰 준비
**04 (JS core) → 02 (hooks) → 07 (event loop) → 03 (reconciliation) → 09 (rendering) → 06 (TS)**

### 성능 튜닝
**12 → 09 → 03 → 02 (메모이제이션 정확히)**

---

## 각 doc 의 일관 구조

```
§0. 이 개념이 무엇이고 왜 알아야 하나
§1. 본질적 메커니즘 (어떻게 동작하나)
§2. 코드 예제 + VulnScope 의 활용
§3. 함정 + 흔한 오해
§4. 직접 구현하면 (가능한 부분만)
§5. 관련 개념 / 더 읽기
§6. 학습 포인트 (한 줄 요약 10개)
```

---

## 다른 study 문서

- [../00-index.md](../00-index.md) — frontend 패턴/디자인 문서.
- [../../patterns/00-index.md](../../patterns/00-index.md) — 백엔드 + 공통 패턴.
- [../../sse-libraries.md](../../sse-libraries.md) — SSE 라이브러리.
