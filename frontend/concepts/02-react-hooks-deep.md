# 02. React Hooks 모두 깊이

> 이 문서가 다루는 것: useState/useEffect/useRef/useMemo/useCallback/useContext/useReducer/useLayoutEffect/useTransition/useDeferredValue/useId/useSyncExternalStore/use 등 모든 hook 의 의도와 사용법.
> 전제: 01-react-component-lifecycle.md.

---

## §0. Hook 의 본질 — 왜 등장했나

### 0.1 class 시대의 문제
```typescript
class Counter extends React.Component {
    state = { count: 0 };
    componentDidMount() { /* subscribe */ }
    componentWillUnmount() { /* unsubscribe */ }
    componentDidUpdate(prev) {
        if (this.props.id !== prev.id) { /* re-subscribe */ }
    }
    render() { ... }
}
```

**문제**:
- subscribe/unsubscribe 가 3개 lifecycle method 에 흩어짐. 한 logic 의 코드가 분리.
- HOC (higher-order component) / render props 같은 재사용 패턴이 wrapper hell.
- this binding 문제.

### 0.2 Hook 의 통찰 (2018)
**한 hook 안에 setup + cleanup 묶음**:
```typescript
function Counter() {
    const [count, setCount] = useState(0);
    useEffect(() => {
        const sub = subscribe(props.id);
        return () => sub.unsubscribe();        // setup + cleanup 같은 곳
    }, [props.id]);
}
```

→ 한 logic = 한 hook. 재사용은 custom hook (`use*` 함수).

### 0.3 Hook 호출 규칙
1. **항상 top level 에서** — if/for/nested function 안 X.
2. **컴포넌트 또는 다른 hook 안에서만** — 일반 함수에서 X.

**왜?** React 가 hook 의 호출 순서로 state slot 매칭. 순서 깨지면 잘못된 state 받음.

```typescript
// ❌ if 안
if (cond) useState(0);
useState("");

// ✅ 항상 top level
useState(0);
const [v, setV] = cond ? useState("default") : [...];   // 안 됨. 다른 hook 으로 풀기
```

---

이제 13개 hook 모두.

---

## §1. useState — 기본

### 1.1 문법
```typescript
const [value, setValue] = useState(initial);
```

### 1.2 lazy initialization
```typescript
const [value, setValue] = useState(() => expensiveComputation());
//                                ↑ function — 처음만 호출
```

→ initial 이 비싼 계산이면 함수 형태로. 매 render 마다 호출 X.

### 1.3 functional update
```typescript
setCount(count + 1);            // ❌ closure 안의 count 가 stale 일 수 있음
setCount(prev => prev + 1);      // ✅ 항상 최신 prev
```

### 1.4 batching (React 18+)
```typescript
function handleClick() {
    setA(1);
    setB(2);
    setC(3);
    // → 1번 re-render (batch)
}
```

React 18+ 부터 setTimeout/Promise 안에서도 자동 batching.

### 1.5 함정
- setState 는 schedule 만. 즉시 state 변경 X.
- 객체 setState 시 spread 필요: `setUser({...user, name: "X"})`. 안 그러면 부분 업데이트 X (replace 아님).
- 같은 값 set 시 React 가 skip (Object.is 비교) — re-render 없음.

---

## §2. useEffect — side effect

### 2.1 문법
```typescript
useEffect(() => {
    // setup
    return () => {
        // cleanup (optional)
    };
}, [deps]);
```

### 2.2 deps 의 의미
- `[]` — mount 1번.
- `[a, b]` — a 또는 b 변경 시.
- 생략 — 매 render (보통 잘못).

### 2.3 cleanup 시점
- deps 변경 직전 (이전 effect cleanup).
- unmount 시.

### 2.4 closure 함정 (가장 큰)
```typescript
const [n, setN] = useState(0);

useEffect(() => {
    const id = setInterval(() => {
        setN(n + 1);     // ❌ n 이 mount 시점의 0 (closure)
    }, 1000);
    return () => clearInterval(id);
}, []);
// 결과: 1, 1, 1, ... (항상 0+1)
```

해결:
- functional update: `setN(prev => prev + 1)`.
- deps 추가: `[n]` (단, 매번 effect 재설정).
- ref: `nRef.current` 로 최신 값.

### 2.5 race condition (async)
```typescript
useEffect(() => {
    fetch(`/api/${id}`).then(r => r.json()).then(setData);
}, [id]);

// id 가 1 → 2 빠르게 변경 시:
// 1. id=1 fetch 시작
// 2. id=2 fetch 시작
// 3. id=2 응답 → setData
// 4. id=1 응답 (늦게) → setData → ❌ 옛 데이터 표시
```

해결: ignore flag + cleanup
```typescript
useEffect(() => {
    let cancelled = false;
    fetch(`/api/${id}`).then(r => r.json()).then(d => {
        if (!cancelled) setData(d);
    });
    return () => { cancelled = true; };
}, [id]);
```

또는 AbortController:
```typescript
useEffect(() => {
    const controller = new AbortController();
    fetch(`/api/${id}`, { signal: controller.signal })
        .then(r => r.json())
        .then(setData)
        .catch(e => { if (e.name !== "AbortError") console.error(e); });
    return () => controller.abort();
}, [id]);
```

VulnScope 는 TanStack Query 가 이거 자동.

---

## §3. useRef — mutable container

### 3.1 두 용도

#### (1) DOM 노드 참조
```typescript
const inputRef = useRef<HTMLInputElement>(null);
return <input ref={inputRef} />;

// 사용
inputRef.current?.focus();
```

#### (2) closure 우회 / 매 render 변경 안 됨
```typescript
const ref = useRef(0);
ref.current++;             // re-render 안 트리거
```

### 3.2 useState vs useRef
| | useState | useRef |
|---|---|---|
| 변경 시 re-render | 트리거 | X |
| 매 render 같은 reference | (매번 새 [val, setter]) | (current 같은 객체) |
| 용도 | rendering 에 영향 | rendering 무관 (timer id, latest value) |

### 3.3 VulnScope 사용
```typescript
// useScanStream — closure 우회
const lastSeqRef = useRef(0);

useEffect(() => {
    es.addEventListener("finding", (e) => {
        if (seq <= lastSeqRef.current) return;     // 항상 최신
        lastSeqRef.current = seq;
    });
}, [scanId]);
```

→ state 였으면 closure 안의 lastSeq 가 stale.

---

## §4. useMemo — 계산 cache

### 4.1 문법
```typescript
const sevCounts = useMemo(() => {
    const counts = { critical: 0, ... };
    for (const f of findings) { ... }
    return counts;
}, [findings]);
```

→ findings 변경 시만 재계산. 그 외 cache.

### 4.2 언제 사용
- **expensive computation**: 큰 list filter/sort.
- **referential equality**: 자식이 React.memo 인데 객체 props 가 매번 새 reference.

### 4.3 남용 주의
```typescript
const sum = useMemo(() => a + b, [a, b]);   // ❌ overhead 가 더 클 수 있음
```

→ 단순 계산엔 useMemo 자체 비용 (compare deps + 결과 보관) 이 더 큼.

### 4.4 핵심 통찰
**useMemo 는 hint, 보장 X**. React 가 cache 를 버릴 수 있음 (메모리 부족 등).

---

## §5. useCallback — 함수 reference 안정화

### 5.1 문법
```typescript
const handleClick = useCallback(() => { ... }, [deps]);
```

= `useMemo(() => fn, deps)`.

### 5.2 언제 사용
- 자식이 React.memo + onClick props.
- useEffect deps 에 함수 들어갈 때.

### 5.3 남용 주의
- 자식이 React.memo 안 하면 useCallback 무의미.
- 모든 함수에 useCallback = 코드 복잡도 + overhead.

### 5.4 useEvent (RFC, 미정)
"항상 최신 props/state 를 보는 안정 reference 함수" — useCallback 의 stale closure 문제 해결. React 19 에서 검토.

---

## §6. useContext — props drilling 회피

### 6.1 문법
```typescript
const ThemeContext = createContext<"light" | "dark">("light");

function App() {
    return (
        <ThemeContext.Provider value="dark">
            <Toolbar />
        </ThemeContext.Provider>
    );
}

function Toolbar() {
    const theme = useContext(ThemeContext);      // 자동
    return <button className={theme}>...</button>;
}
```

### 6.2 props drilling 이란
A → B → C → D 모든 layer 를 거쳐 D 에 props 전달. 중간 layer 가 그 props 모르고 그냥 통과.

→ Context 가 이걸 우회.

### 6.3 함정
- Context value 변경 시 모든 consumer re-render.
- 큰 객체를 한 Context 에 묶으면 불필요 re-render 폭증.
- 해결: split (theme + locale 별도 Context) 또는 selector (Zustand 등).

### 6.4 VulnScope 사용
- TanStack Query 의 QueryClientProvider — Provider 가 Context.
- Theme/locale 등 cross-cutting state.

VulnScope 는 직접 createContext 안 씀 (server state 만 관리하면 충분).

---

## §7. useReducer — 복잡 state machine

### 7.1 문법
```typescript
type Action = { type: "increment" } | { type: "set"; value: number };

function reducer(state: number, action: Action): number {
    switch (action.type) {
        case "increment": return state + 1;
        case "set": return action.value;
    }
}

const [state, dispatch] = useReducer(reducer, 0);

dispatch({ type: "increment" });
```

### 7.2 useState vs useReducer
- useState: 단순 값.
- useReducer: 복잡 state transition (action type 다양).

### 7.3 핵심 통찰
**Redux 의 작은 버전**. 컴포넌트 안에서 state machine.

VulnScope 는 useReducer 미사용 — useState 충분.

---

## §8. useLayoutEffect — sync side effect

### 8.1 vs useEffect
| | useLayoutEffect | useEffect |
|---|---|---|
| 시점 | DOM 업데이트 후, paint 전 | paint 후 |
| sync | sync (블로킹) | async |
| 용도 | DOM measure, scroll 위치 보정 | 일반 |

### 8.2 깜빡임 예
01-react-component-lifecycle.md §6 참조.

### 8.3 함정
- sync 라 느린 작업 X. paint 차단.
- SSR 시 warning (server 에 layout 없음). `useEffect` 또는 `useIsomorphicLayoutEffect`.

---

## §9. useTransition — non-urgent update (React 18+)

### 9.1 문법
```typescript
const [isPending, startTransition] = useTransition();

function handleSearch(query: string) {
    setQuery(query);                       // urgent (input 즉시)
    startTransition(() => {
        setSearchResults(filter(data, query));    // non-urgent
    });
}
```

### 9.2 의미
**"이 update 는 늦어도 OK"** 표시. React 가 더 급한 update (input typing 등) 우선.

→ 큰 list filter 같은 무거운 update 가 input 끊지 않음.

### 9.3 함정
- transition 안의 state update 만 영향. 외부 (urgent) 는 즉시.
- 외부 fetch 같은 async 에 효과 적음 — useDeferredValue 도 고려.

VulnScope 미사용 (작은 list).

---

## §10. useDeferredValue — value 의 transition

### 10.1 문법
```typescript
const deferredQuery = useDeferredValue(query);
// useEffect(() => { 무거운 작업 with deferredQuery }, [deferredQuery]);
```

### 10.2 useTransition 과 차이
- useTransition: action 수준 ("이 setState 는 deferred").
- useDeferredValue: value 수준 ("이 value 는 stale OK").

### 10.3 효과
input 빠르게 치는 동안 deferredQuery 는 옛 값 유지 → 무거운 effect 매 keystroke 안 돔.

---

## §11. useId — SSR-safe unique id

### 11.1 문법
```typescript
const id = useId();    // 안정된 unique id
return (
    <>
        <label htmlFor={id}>Email</label>
        <input id={id} />
    </>
);
```

### 11.2 왜 필요?
SSR + hydration 시 server/client 의 id 가 같아야 함. `Math.random()` 으로 생성하면 mismatch.

useId 가 React 의 tree position 기반 안정 id.

### 11.3 use case
- form label htmlFor.
- aria-* 속성.

VulnScope 미사용 (간단 폼).

---

## §12. useSyncExternalStore — 외부 store 통합

### 12.1 문법
```typescript
const value = useSyncExternalStore(
    subscribe,         // (callback) => unsubscribe
    getSnapshot,       // () => value
    getServerSnapshot? // SSR 용
);
```

### 12.2 용도
React 외부 store (Redux, Zustand, browser API) 를 React 와 동기화. tearing 방지.

### 12.3 예 — online status
```typescript
function useOnlineStatus() {
    return useSyncExternalStore(
        (cb) => {
            window.addEventListener("online", cb);
            window.addEventListener("offline", cb);
            return () => {
                window.removeEventListener("online", cb);
                window.removeEventListener("offline", cb);
            };
        },
        () => navigator.onLine,
        () => true  // SSR default
    );
}
```

### 12.4 VulnScope 미사용
TanStack Query 가 내부적으로 사용. 직접 호출 X.

---

## §13. use (React 19+)

### 13.1 문법
```typescript
function ScanResult({ scanId }: { scanId: string }) {
    const scan = use(fetchScan(scanId));   // suspend until resolved
    return <div>{scan.status}</div>;
}
```

### 13.2 의미
- Promise 직접 await 같은 효과.
- Suspense boundary 가 fallback.
- Context 도 받을 수 있음 (`use(MyContext)`).

### 13.3 useEffect 와 차이
- useEffect: render 후 fetch.
- use: render 중 suspend → Suspense fallback → resolve 후 render.

### 13.4 VulnScope
TanStack Query 가 이미 cache + Suspense 통합. 직접 use() 안 씀.

---

## §14. Custom Hooks — 로직 재사용

### 14.1 패턴
```typescript
function useScanStream(scanId: string) {
    const [events, setEvents] = useState([]);
    const lastSeqRef = useRef(0);

    useEffect(() => {
        const es = new EventSource(...);
        es.addEventListener("finding", ...);
        return () => es.close();
    }, [scanId]);

    return { events, ... };
}
```

### 14.2 규칙
- `use` prefix.
- hook 호출 규칙 (top level + 컴포넌트 안) 만족.
- 어디서든 호출 가능.

### 14.3 VulnScope 의 custom hooks
- `useScanStream` — SSE.
- `useScanResult`, `useScans`, `useFindingDetail`, `useTargets`, `useProfiles` — TanStack Query wrapper.

→ 비즈니스 logic + state 분리. 컴포넌트는 view.

---

## §15. 패턴 매트릭스

| 문제 | hook |
|---|---|
| 단순 state | useState |
| side effect | useEffect |
| DOM ref / mutable container | useRef |
| 비싼 계산 cache | useMemo |
| 함수 reference 안정 | useCallback |
| props drilling 회피 | useContext |
| 복잡 state machine | useReducer |
| sync side effect (paint 전) | useLayoutEffect |
| non-urgent update | useTransition |
| value 의 deferred | useDeferredValue |
| SSR-safe id | useId |
| 외부 store 통합 | useSyncExternalStore |
| Promise / Context (R19) | use |
| 로직 재사용 | custom hook |

---

## §16. 학습 포인트

1. **hook 호출 규칙** — top level + 컴포넌트/hook 안.
2. **useState lazy init** — 비싼 initial 은 함수.
3. **functional update** — `setX(prev => ...)` 가 closure 안전.
4. **useEffect closure 함정** — deps + functional update + ref.
5. **race condition** — cancelled flag 또는 AbortController.
6. **useRef = mutable container** — re-render 무관.
7. **useMemo 남용 X** — 비싼 계산만.
8. **useContext re-render 폭증** — split + memoize.
9. **useReducer = 컴포넌트 안 mini Redux**.
10. **useLayoutEffect = sync, useEffect = async** — 깜빡임만 후자, 일반은 전자.
11. **useTransition = action, useDeferredValue = value**.
12. **useSyncExternalStore = 외부 store 통합**. 라이브러리 내부.
13. **use (R19) = Promise/Context 직접**. Suspense 통합.
14. **custom hook = logic 재사용**. `use` prefix.

### 추가 참고
- React 공식 hooks: https://react.dev/reference/react/hooks
- Dan Abramov hooks 시리즈
- Patterns.dev Hooks: https://www.patterns.dev/posts/react-hooks
