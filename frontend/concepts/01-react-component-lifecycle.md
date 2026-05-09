# 01. React 컴포넌트 라이프사이클

> 이 문서가 다루는 것: React 컴포넌트가 mount/update/unmount 되는 과정, Strict Mode 의 더블 mount, Concurrent rendering 의 의미.
> 이걸 모르면: useEffect 가 두 번 실행되는 이유 모름, cleanup 안 함 → memory leak, render 가 commit 과 다른 phase 라는 것 모름.

---

## §0. lifecycle 이 무엇이고 왜 중요한가?

### 0.1 lifecycle = "컴포넌트의 생애"
React 컴포넌트는 다음 단계를 거침:
1. **mount** — 처음 DOM 에 추가됨.
2. **update** — props/state 변경 시 다시 render.
3. **unmount** — DOM 에서 제거됨.

```typescript
function Counter() {
    console.log("render");
    useEffect(() => {
        console.log("mount or update effect");
        return () => console.log("cleanup");
    });
    return <div>...</div>;
}
```

### 0.2 왜 알아야 하나?
- **resource lifecycle** — subscription/timer 는 mount 시 setup, unmount 시 cleanup.
- **performance** — 불필요한 update 줄이기.
- **mental model** — render 가 매번 일어남, side effect 는 render 후.

VulnScope 의 useScanStream 이 정확히 이 lifecycle 활용:
```typescript
useEffect(() => {
    const es = new EventSource(...);          // mount 시 connect
    return () => es.close();                  // unmount 시 close
}, [scanId]);
```

---

## §1. Class component 시대 (legacy 참고)

### 1.1 lifecycle methods (deprecated 일부)
```typescript
class Counter extends React.Component {
    componentDidMount() { /* mount 후 */ }
    componentDidUpdate(prevProps) { /* update 후 */ }
    componentWillUnmount() { /* unmount 직전 */ }
    shouldComponentUpdate(nextProps, nextState) { /* update 결정 */ }
    render() { return <div>...</div>; }
}
```

### 1.2 함수형 + hook 으로 대체
- `componentDidMount` + `componentDidUpdate` + `componentWillUnmount` = `useEffect`.
- `shouldComponentUpdate` = `React.memo` + `useMemo`.

### 1.3 핵심
**class 는 거의 안 씀**. 모든 새 코드는 함수 + hook. legacy 이해용.

VulnScope 도 100% 함수 컴포넌트.

---

## §2. 함수 컴포넌트의 render

### 2.1 render = 함수 호출
```typescript
function Counter({ count }) {
    return <div>{count}</div>;
}
```

React 가:
1. props 받아 컴포넌트 함수 호출.
2. 반환값 (JSX = React Element) 받음.
3. 이전 render 의 결과와 비교 (reconciliation).
4. 변한 부분만 DOM 업데이트.

매 render 마다 함수 새로 호출 → 변수 새로 만들어짐. 그래서 useState/useRef 가 React 내부 store 를 사용.

### 2.2 render 트리거 조건
- **mount** — 처음 추가.
- **state 변경** — `setState`.
- **props 변경** — 부모가 다른 props 전달.
- **context 변경** — useContext 의 value 변경.
- **부모 re-render** — 부모가 re-render 면 자식도 (memo 안 했으면).

### 2.3 render = pure function 권장
React 가 가정:
- 같은 props → 같은 결과.
- side effect 없음 (DOM 직접 수정, fetch, console.log 등 X — render 중에는).
- side effect 는 useEffect 안에 분리.

**왜?** Concurrent rendering 에서 render 가 여러 번 호출되거나 abort 될 수 있음. side effect 가 render 안에 있으면 중복.

---

## §3. mount → update → unmount

### 3.1 mount
```
1. 컴포넌트 함수 호출 (render)
2. React 가 결과를 DOM 에 추가
3. useLayoutEffect 실행 (sync, 화면 그리기 전)
4. 화면 그리기 (paint)
5. useEffect 실행 (async, paint 후)
```

### 3.2 update
```
1. props/state/context 변경 트리거
2. 컴포넌트 함수 호출 (재 render)
3. 이전 결과와 비교 (reconciliation)
4. 변한 부분만 DOM 업데이트
5. useLayoutEffect cleanup → 새 useLayoutEffect (deps 변경 시)
6. useEffect cleanup → 새 useEffect (deps 변경 시, paint 후)
```

### 3.3 unmount
```
1. 컴포넌트 제거 결정 (조건부 render or 부모 제거)
2. 자식 unmount 먼저 (children 부터)
3. useLayoutEffect cleanup
4. DOM 제거
5. useEffect cleanup
```

### 3.4 핵심 통찰
**cleanup 함수가 unmount 시 자동 호출**. memory leak 방지의 핵심.

```typescript
useEffect(() => {
    const id = setInterval(...);
    return () => clearInterval(id);    // unmount 시 자동
}, []);
```

→ cleanup 빠뜨리면 컴포넌트 사라져도 timer 살아 있음. setState 호출 시 "Can't perform a React state update on an unmounted component" 경고.

VulnScope 의 useScanStream 이 정확히 이거:
```typescript
return () => es.close();    // unmount 시 EventSource 닫음
```

---

## §4. Strict Mode 의 더블 mount (Dev 전용)

### 4.1 Strict Mode 란
```typescript
// app/layout.tsx
<React.StrictMode>
    <App />
</React.StrictMode>
```

React 18+ Strict Mode 는 **dev 환경에서 의도적으로 mount → unmount → mount** 두 번 실행.

```
1. Counter mount
2. useEffect 실행
3. (즉시) cleanup
4. Counter unmount
5. Counter 다시 mount
6. useEffect 다시 실행
```

### 4.2 왜 이렇게 짜증나게?
**bug 조기 발견**:
- cleanup 안 한 effect 가 두 번 setup → 명확히 보임.
- 옛 코드의 implicit assumption (mount 1회) 발견.

특히 React 18+ 의 `<Offscreen />` (가시성 토글) 또는 Suspense + Server Components 환경에서 컴포넌트가 진짜로 mount/unmount/mount 될 수 있음. Strict Mode 가 이 시나리오 dev 에 미리 시뮬.

### 4.3 영향
- useEffect 가 두 번 실행 → cleanup 도 두 번 → resource 둘 다 해제됨 (cleanup 잘 짰으면 OK).
- API fetch 가 두 번 → backend 부담 (단, dev 만).
- 초기화 횟수 카운트 등의 코드는 두 번 실행 → bug 노출.

### 4.4 production 에선?
**Strict Mode 효과 0**. production 빌드에선 더블 mount 안 함.

→ dev 에서만 짜증나는 보호 장치.

### 4.5 함정
- ref 로 "mount 했음" 표시 후 fetch 하는 패턴 (fetch 두 번 회피) → Strict Mode 회피지만 진짜 마운트되면 fetch 안 됨. 안티패턴.
- 정답: cleanup 잘 짜고 fetch 두 번 OK 로 만들기 (또는 dedupe).

---

## §5. Concurrent Rendering (React 18+)

### 5.1 기본
React 18 부터 **render 가 interruptible**:
- React 가 render 시작.
- 더 급한 일 (사용자 입력 등) 발생 → render 중단.
- 급한 일 처리 → render 다시 시작 (또는 abort).

→ "render 가 1번 = commit 1번" 보장 X.

### 5.2 Concurrent 의 의도
- **반응성**: 무거운 컴포넌트 render 중에도 UI 가 input 에 반응.
- **자동 batching**: 여러 setState 가 1번 render.
- **Transitions**: "급하지 않은 update" 표시 (`useTransition`).

### 5.3 함정 — render 가 여러 번 호출됨
```typescript
function Counter() {
    let counter = 0;        // ❌ render 안에서 변수 변경 위험
    counter++;
    console.log(counter);
}
```
→ Concurrent 에서 render 가 abort + 재시작 시 counter 가 의미 없음.

→ render 는 **pure function** 권장. side effect 는 useEffect.

### 5.4 commit phase
**render** = 함수 호출 + 가상 결과 계산 (interruptible).
**commit** = DOM 실제 업데이트 + ref 갱신 + lifecycle (uninterruptible, sync).

useEffect 는 commit 후 (paint 후) async. useLayoutEffect 는 commit 후 paint 전 sync.

---

## §6. useLayoutEffect vs useEffect

### 6.1 차이
| | useLayoutEffect | useEffect |
|---|---|---|
| 실행 시점 | DOM 업데이트 후, paint 전 | paint 후 |
| 동기/비동기 | sync (블로킹) | async |
| 사용처 | DOM measure, 깜빡임 방지 | 일반 side effect |

### 6.2 깜빡임 (flicker) 시나리오
```typescript
// useEffect — 깜빡임 가능
useEffect(() => {
    document.title = `Count: ${count}`;
});
// 1. count 0 → render → paint (title "Count: 0")
// 2. useEffect 실행 → title "Count: 1" → 다시 paint
// 사용자가 잠깐 0 봄.

// useLayoutEffect — 깜빡임 없음
useLayoutEffect(() => {
    document.title = `Count: ${count}`;
});
// 1. count 0 → render → DOM update → useLayoutEffect (title "Count: 1") → paint
// 사용자는 1 만 봄.
```

### 6.3 일반 권장
**useEffect 를 default**. useLayoutEffect 는 깜빡임 문제 있을 때만.

VulnScope 는 useLayoutEffect 미사용 — 모든 effect 가 일반 useEffect.

---

## §7. 컴포넌트 라이프사이클 전체 흐름 (요약)

### 7.1 mount
```
[render]                     ← function 호출, JSX 반환
  ↓
[diff & DOM update]          ← 처음이라 모든 노드 추가
  ↓
[useLayoutEffect]            ← sync, paint 전
  ↓
[paint]                      ← 화면에 그림
  ↓
[useEffect]                  ← async, paint 후
```

### 7.2 update
```
[setState / props change]
  ↓
[render]                     ← function 다시 호출
  ↓
[reconciliation]             ← 이전과 비교
  ↓
[DOM update (변한 부분만)]
  ↓
[useLayoutEffect cleanup]    ← deps 변경 시
[useLayoutEffect]            ← 새 effect
  ↓
[paint]
  ↓
[useEffect cleanup]          ← deps 변경 시
[useEffect]                  ← 새 effect
```

### 7.3 unmount
```
[조건부 render → 빠짐]
  ↓
[자식 unmount 먼저]
  ↓
[useLayoutEffect cleanup]
  ↓
[DOM 제거]
  ↓
[useEffect cleanup]
```

---

## §8. VulnScope 의 lifecycle 활용 예

### 8.1 useScanStream
```typescript
useEffect(() => {
    const es = new EventSource(...);
    es.addEventListener("finding", listener);
    es.onopen = ...; es.onerror = ...;

    return () => es.close();
}, [scanId]);
```

- mount: EventSource open + listener 등록.
- scanId 변경: cleanup (close) + 새 effect (new EventSource).
- unmount: close.

→ resource lifecycle 명확.

### 8.2 ScanLiveDashboard 의 자동 navigate
```typescript
useEffect(() => {
    if (done || failed) {
        const id = setTimeout(() => router.push(`/scans/${scanId}`), 1500);
        return () => clearTimeout(id);
    }
}, [done, failed, router, scanId]);
```

- done/failed 가 true 가 되면 1.5초 후 navigate.
- 중간에 다른 변화 (혹은 unmount) 시 cleanup 으로 timer cancel.

→ 중복 navigate 방지.

---

## §9. 함정 + 흔한 오해

### 9.1 "useEffect 가 1번만 실행되어야 한다"
- Strict Mode 에선 dev 시 두 번. cleanup 잘 짜면 OK.
- production 에선 1번.
- "useEffect 두 번 실행 = bug" 가정 X.

### 9.2 "render 안에서 fetch 해도 되나?"
**X**. render 는 pure function. fetch 는 side effect → useEffect.

### 9.3 "setState 가 즉시 반영되나?"
**X**. setState 는 schedule 만. 다음 render 에 반영. closure 안의 state 는 옛 값.

### 9.4 "useEffect 의 deps 빠뜨려도 되나?"
**X**. eslint-plugin-react-hooks 가 강제. deps 빠뜨리면 stale closure 문제.

### 9.5 "componentWillMount 같은 hook 있나?"
**X**. 함수 컴포넌트는 mount 전 hook 없음. 첫 render 가 곧 mount.

---

## §10. 학습 포인트

1. **render = 함수 호출, 매번**. pure function 권장.
2. **mount/update/unmount** 3단계. useEffect cleanup 이 unmount 시 자동.
3. **Strict Mode = dev 더블 mount**. cleanup 잘 짜면 OK.
4. **Concurrent rendering** = render interruptible, side effect X.
5. **commit phase** = render 후 sync, DOM 갱신 + useLayoutEffect.
6. **useLayoutEffect = sync (paint 전)**, useEffect = async (paint 후).
7. **cleanup 안 함 = memory leak** (timer, subscription, EventSource).
8. **deps 변경 시 cleanup → 새 effect**.
9. **자식 unmount 먼저** (children 부터 cleanup).
10. **production 은 Strict Mode 효과 0** — dev 만.

### 추가 참고
- React 공식: https://react.dev/learn/lifecycle-of-reactive-effects
- Dan Abramov, [A Complete Guide to useEffect](https://overreacted.io/a-complete-guide-to-useeffect/)
- React 18 Concurrent: https://react.dev/blog/2022/03/29/react-v18
