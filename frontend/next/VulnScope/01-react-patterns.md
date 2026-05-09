# 01. React 핵심 패턴

> 이 문서가 다루는 것: React hook 의 본질, server vs client component, Suspense, ref vs state — VulnScope 가 사용한 모든 React 핵심 패턴의 **아이디어**.
> 전제: useState/useEffect 본 적은 있음. 의도와 함정은 잘 모름.

---

## §0. React 의 mental model

### 0.1 React 는 함수다
```typescript
function MyComponent({ name }: { name: string }) {
    return <div>Hello, {name}</div>;
}
```
**핵심**: 컴포넌트 = 입력 (props) → 출력 (JSX) 함수. **순수 함수 가 ideal**.

같은 props 면 같은 결과. side effect 안 함 (render 중에).

### 0.2 render 는 매번 일어남
React 는 props/state 변경 감지 → 함수 재호출 → 새 JSX → 비교 (reconciliation) → 변한 곳만 DOM 업데이트.

```typescript
function Counter() {
    const [n, setN] = useState(0);
    console.log("rendered");           // ← state 변경 시마다 출력
    return <button onClick={() => setN(n + 1)}>{n}</button>;
}
```

→ **함수 내부 변수는 매 render 새로 만들어짐**. closure.

### 0.3 closure 의 함정 (가장 핵심)
```typescript
function Bad() {
    const [n, setN] = useState(0);

    useEffect(() => {
        const id = setInterval(() => {
            console.log(n);     // ❌ 항상 0! (mount 시점의 n 만 봄)
            setN(n + 1);         // ❌ 항상 0+1=1
        }, 1000);
        return () => clearInterval(id);
    }, []);                       // 빈 deps → 한 번만 setup
}
```

setInterval 의 callback 은 **mount 시점의 `n` 을 closure 로 capture**. state 가 바뀌어도 callback 안의 `n` 은 그대로.

해결법:
- **functional update**: `setN(prev => prev + 1)` — closure 우회.
- **deps 추가**: `useEffect(..., [n])` — n 변경 시 effect 재설정.
- **useRef**: 최신 값 보관 (다음 §3 참조).

→ **render → 새 closure → 옛 변수 capture** 가 모든 hook 함정의 root cause.

이걸 알면 React 80% 이해.

---

## §1. Hooks 의 본질

### 1.1 useState — render 간 값 보존

```typescript
const [count, setCount] = useState(0);
```

**아이디어**:
- 함수 호출이 매 render 새로 시작 → 일반 변수는 보존 X.
- React 가 component instance 마다 array 형태로 state 저장.
- `useState` 호출 순서로 index 매칭.

**왜 호출 순서?**
React 가 hook 의 이름이 아닌 **호출 순서** 로 state slot 찾음. 그래서:
- ❌ `if (cond) useState(0)` — 호출 순서 깨짐.
- ✅ 항상 컴포넌트 top level 에서 호출.

### 1.2 useEffect — render 후 side effect

```typescript
useEffect(() => {
    fetch("/api/data").then(...);
    return () => { /* cleanup */ };
}, [deps]);
```

**아이디어**:
- 렌더 함수는 순수해야 함 (side effect X).
- side effect (fetch, subscription, DOM 직접 수정) 는 render 끝난 후 실행.
- `deps` 배열로 "재실행 조건" 명시.
- return function = cleanup (다음 effect 실행 전 또는 unmount 시).

**deps 의미**:
- `[]` — 한 번만 (mount).
- `[a, b]` — a 또는 b 변경 시.
- 생략 — 매 render (보통 잘못된 패턴).

**핵심 함정 — closure**: §0.3 참조. deps 빠뜨리면 stale read.

### 1.3 useRef — render 와 무관한 mutable container

```typescript
const ref = useRef(0);

useEffect(() => {
    setInterval(() => {
        ref.current++;        // 변경해도 re-render 안 됨
        console.log(ref.current);  // 항상 최신
    }, 1000);
}, []);
```

**아이디어**:
- `{ current: T }` 같은 객체. component 동안 동일 인스턴스.
- 변경해도 re-render 트리거 X.
- 매 render 에서 같은 객체 → closure 로 capture 해도 항상 최신 값.

**용도**:
1. **DOM 노드 참조**: `<input ref={inputRef} />`.
2. **closure 우회**: setInterval 안에서 최신 state 읽기.
3. **mount 여부 추적**: `useRef(false)` + `useEffect` 에서 set.

VulnScope 의 사용:
```typescript
// useScanStream.ts
const lastSeqRef = useRef(0);

useEffect(() => {
    es.addEventListener("finding", (e) => {
        const seq = Number(e.lastEventId);
        if (seq <= lastSeqRef.current) return;    // dedupe — closure 안 해도 최신
        lastSeqRef.current = Math.max(lastSeqRef.current, seq);
    });
}, [scanId]);
```
→ `lastSeqRef.current` 가 listener 안에서도 최신. state 였으면 closure stale.

### 1.4 useMemo — 비싼 계산 cache

```typescript
const sevCounts = useMemo(() => {
    const counts = { critical: 0, ... };
    for (const f of findings) { ... }
    return counts;
}, [findings]);
```

**아이디어**:
- 매 render 마다 같은 계산 반복 → 비용.
- deps 변경 안 됐으면 이전 결과 재사용.

**남용 주의**: 단순 계산엔 useMemo overhead 가 더 클 수 있음. 측정 후 사용.

### 1.5 useCallback — 함수 reference 안정화

```typescript
const handleClick = useCallback(() => { ... }, [deps]);
```

**아이디어**:
- 함수도 매 render 새로 생성 → 자식에 props 로 넘기면 자식 re-render.
- useCallback = useMemo 의 함수 버전.

**남용 주의**: 자식이 React.memo 안 쓰면 무의미.

### 1.6 Custom hooks — 로직 재사용

```typescript
function useScanStream(scanId: string): ScanStreamState {
    const [events, setEvents] = useState([]);
    useEffect(() => { /* SSE setup */ }, [scanId]);
    return { events, ... };
}
```

**아이디어**:
- hook 호출 규칙 만족 (top level + 컴포넌트/hook 안에서만) → 어디서든 가능.
- `use` prefix 컨벤션.
- 로직 + state 묶음. 컴포넌트 추출과 다른 차원의 재사용.

VulnScope 의 custom hooks:
- `useScanStream(scanId)` — SSE 구독.
- `useScanResult(scanId)` — 3개 useQuery 합성.
- `useFindingDetail(findingId)`.
- `useTargets()` / `useScans({...})` / `useProfiles()` — TanStack Query wrapper.

**핵심**: 비즈니스 로직 컴포넌트에서 분리. 컴포넌트는 view 만.

---

## §2. Server vs Client Component (Next.js 13+ App Router)

### 2.1 default 가 server component
```typescript
// app/page.tsx
export default function HomePage() {
    return <div>Hello</div>;        // server-rendered
}
```

**효과**:
- HTML 만 응답 (JS 번들 X).
- 빠른 초기 로드.
- 직접 DB/API 호출 가능 (서버 코드).
- `useState`, `useEffect` 등 hook 사용 X.

### 2.2 `"use client"` directive
```typescript
"use client";

import { useState } from "react";

export function LoginForm() {
    const [email, setEmail] = useState("");    // OK
    return <input ... />;
}
```

**효과**:
- 클라이언트로 hydrate (JS 번들 포함).
- hooks 사용 가능.
- onClick, onChange 등 인터랙션 가능.
- 자식 컴포넌트도 자동 client (boundary).

### 2.3 핵심 통찰 — boundary

```
[server component (page.tsx)]
   │
   ├─► [server component] (그대로 server)
   │
   └─► [client component (LoginForm)]
           │
           └─► [child component] ← 자동 client
```

**규칙**:
- server → server: 자유.
- server → client: OK. client component 가 boundary.
- client → server: **불가** (보통). client 가 server fetch 하려면 별도 메커니즘.

### 2.4 VulnScope 의 적용

```typescript
// app/(app)/scans/[scanId]/page.tsx — server component
export default async function ScanResultPage({ params }: { params: Promise<{ scanId: string }> }) {
    const { scanId } = await params;
    return <ScanResult scanId={scanId} />;        // ScanResult 는 "use client"
}

// features/scan/components/ScanResult.tsx — client component
"use client";
export function ScanResult({ scanId }) {
    const { scan, ... } = useScanResult(scanId);    // hook
    // ...
}
```

**왜 분리?**:
- Page (server) = routing + parameter 추출 + initial load 가능.
- ScanResult (client) = TanStack Query useQuery 같은 hooks 사용.

### 2.5 함정
- "use client" 컴포넌트 import 시 그 자식들 자동 client. 의도 안 하면 번들 증가.
- async params (Next.js 15+) — `params: Promise<{ ... }>`. await 필수.

### 2.6 직접 구현한다면
원리: **two-pass rendering**.
1. Server 단: server component HTML + client component placeholder.
2. Client 단: hydrate (placeholder 에 JS 붙임).

직접 구현 시 (예: Vite + custom loader):
- Vite plugin 으로 `"use client"` 마커 감지.
- server pass: client component 를 placeholder div 로 (data attribute 에 props).
- client pass: placeholder 찾아 React.hydrate 호출.

**복잡도 폭증** — Next.js 가 이걸 다 해줌. 보통 직접 구현 X.

---

## §3. Suspense + 비동기 boundary

### 3.1 Suspense 의 의미
```typescript
<Suspense fallback={<Spinner />}>
    <ChildThatMightSuspend />
</Suspense>
```

**아이디어**: 자식이 아직 준비 안 됐다 (fetch 중, lazy load 중) → fallback 표시. 준비되면 자동 swap.

### 3.2 VulnScope 의 사용
```typescript
// app/(auth)/login/page.tsx
<Suspense fallback={<div>Loading...</div>}>
    <LoginForm />
</Suspense>
```

**왜?** `LoginForm` 안의 `useSearchParams()` 가 client hydration 중 suspend. Suspense 가 그 boundary.

### 3.3 핵심 통찰
**Suspense 는 비동기 boundary** — "이 안에 어떤 것이 비동기로 준비 중이면 fallback".

데이터 fetch 도 가능 (React 19 use() hook):
```typescript
const data = use(fetchPromise);    // suspend until resolved
```

### 3.4 직접 구현한다면
React 의 Suspense 는 **componentDidCatch + throw promise** 메커니즘:
- 자식이 throw promise → 부모 Suspense 가 catch → fallback 렌더 → promise resolve 시 retry.

직접 구현:
```typescript
class MySuspense extends React.Component {
    state = { promise: null };

    componentDidCatch(error) {
        if (error instanceof Promise) {
            this.setState({ promise: error });
            error.then(() => this.setState({ promise: null }));
        }
    }

    render() {
        if (this.state.promise) return this.props.fallback;
        return this.props.children;
    }
}
```

원리만. React 가 정밀 처리 (priority, hydration).

---

## §4. 비제어 vs 제어 컴포넌트

### 4.1 controlled (제어)
```typescript
const [value, setValue] = useState("");
<input value={value} onChange={(e) => setValue(e.target.value)} />
```
- 매 keystroke 마다 setValue → re-render.
- React state 가 single source of truth.

### 4.2 uncontrolled (비제어)
```typescript
const ref = useRef<HTMLInputElement>(null);
<input ref={ref} defaultValue="" />
// 값 읽기:
const value = ref.current?.value;
```
- DOM 이 source of truth.
- React state 없음 → re-render 없음 (성능).

### 4.3 RHF 의 선택
RHF 는 비제어 + ref subscription.

```typescript
const { register } = useForm();
<input {...register("email")} />
// register 가 ref + onBlur + onChange 자동
```

→ keystroke 마다 re-render 0. 폼 전체가 가벼움. (자세히는 [03-form-patterns.md](03-form-patterns.md))

### 4.4 핵심 통찰
**re-render 비용 vs single source of truth** 의 trade-off.
- 비제어 = 빠름 + DOM 의존.
- 제어 = React 일관 + re-render 비용.

---

## §5. 패턴 매트릭스

| 문제 | 패턴 |
|---|---|
| render 간 값 보존 | useState |
| side effect | useEffect + cleanup + deps |
| re-render 없이 mutable | useRef |
| 비싼 계산 cache | useMemo |
| 함수 reference 안정 | useCallback |
| 로직 재사용 | custom hook |
| server-rendered + 빠른 로드 | server component |
| interactivity | "use client" |
| 비동기 boundary | Suspense |
| 폼 re-render 회피 | uncontrolled (RHF) |

---

## §6. 학습 포인트

1. **render = 함수 호출** — closure 가 모든 hook 함정의 root.
2. **useState 의 호출 순서 의존** — top level 만.
3. **useEffect deps 빠뜨리면 stale read** — eslint-plugin-react-hooks 강제.
4. **useRef = "render 와 무관한 mutable container"** — closure 우회 + DOM ref.
5. **useMemo/useCallback 남용 X** — 측정 후.
6. **server vs client boundary** — "use client" = client subtree 시작.
7. **async params (Next.js 15+)** — Promise + await.
8. **Suspense = 비동기 boundary** — fallback 자동.
9. **uncontrolled = re-render 0** — RHF 의 핵심 의도.
10. **custom hook 으로 로직 분리** — 컴포넌트는 view 만.

### 추가 참고
- React 공식 문서: https://react.dev/learn
- Dan Abramov 의 [A Complete Guide to useEffect](https://overreacted.io/a-complete-guide-to-useeffect/) — closure 함정.
- Kent C. Dodds 의 [How to use React Context effectively](https://kentcdodds.com/blog/how-to-use-react-context-effectively).
