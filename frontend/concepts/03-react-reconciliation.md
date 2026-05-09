# 03. React Reconciliation + Fiber

> 이 문서가 다루는 것: Virtual DOM, diffing, render phase, commit phase, key prop, Fiber 아키텍처. React 가 화면을 어떻게 효율적으로 업데이트하는지.

---

## §0. 왜 reconciliation 이 필요한가?

### 0.1 naive (직접 DOM 조작)
```typescript
document.getElementById("count").textContent = String(count);
```

**문제**:
- DOM 조작은 비쌈 (layout/repaint).
- 여러 컴포넌트가 같은 DOM 노드 만지면 충돌.
- 모든 변경을 직접 추적 = 복잡 + bug.

### 0.2 React 의 통찰
**Virtual DOM** = 화면을 "JS 객체 tree" 로 표현. 변경 시:
1. 새 Virtual DOM 생성.
2. 이전 Virtual DOM 과 비교 (diff).
3. 변한 부분만 실제 DOM 업데이트.

→ **render 는 함수 호출 (싸다), 실제 DOM 업데이트만 최소화**.

### 0.3 reconciliation = "이전 vs 새 비교"
```
이전 tree:                      새 tree:
<div>                            <div>
    <h1>Hello</h1>                  <h1>Hello, World!</h1>    ← 변경
    <p>Click</p>                    <p>Click</p>
</div>                           </div>
```

→ React 가 `<h1>` 의 textContent 만 업데이트. 나머지는 그대로.

---

## §1. Virtual DOM (React Element)

### 1.1 JSX 의 정체
```typescript
<div className="card">
    <h1>Hello</h1>
</div>
```

→ JSX 컴파일 후:
```typescript
React.createElement("div", { className: "card" },
    React.createElement("h1", null, "Hello")
);
```

→ 결국 평범한 JS 객체:
```typescript
{
    type: "div",
    props: { className: "card", children: [
        { type: "h1", props: { children: "Hello" } }
    ]}
}
```

이게 **React Element** = Virtual DOM 의 노드.

### 1.2 React Element 의 특징
- 가벼운 JS 객체.
- 만들기 싸 (단순 object literal).
- 실제 DOM 노드 X.

→ 매 render 마다 새 tree 만들어도 부담 적음. 비교 후 진짜 DOM 만 업데이트.

---

## §2. Diffing 알고리즘

### 2.1 일반 graph diff = O(N³)
두 tree 의 최소 변경 알고리즘은 매우 비쌈.

### 2.2 React 의 휴리스틱 = O(N)
**가정 1 — type 다르면 통째로 교체**:
```
이전: <div><Counter /></div>
새:   <span><Counter /></span>
```
→ `<div>` → `<span>` 이라 Counter 도 unmount + mount. 자식 state 잃음.

**가정 2 — list 는 key 로 매칭**:
```
이전: [<Item id="A" />, <Item id="B" />]
새:   [<Item id="B" />, <Item id="A" />]
```
- key 없으면 React 가 위치 기반으로 비교 → 둘 다 props 변경으로 처리.
- key 있으면 React 가 reorder 로 인식 → A/B 의 state 보존.

### 2.3 key 의 의미
```typescript
{items.map(item => <Item key={item.id} {...item} />)}
```

**규칙**:
- key 가 안정 (id 기반) → reorder 시 state 보존.
- key 가 index → reorder 시 잘못된 매칭 (state mismatch).

**잘못된 패턴**:
```typescript
{items.map((item, i) => <Item key={i} {...item} />)}    // ❌ index 사용
```

→ list 변경 시 모든 컴포넌트가 props 변경으로 인식. state 잃거나 잘못 매칭.

### 2.4 함정
- 같은 컴포넌트 type + 다른 위치 → state 다른 컴포넌트 것 받음.
- key 가 stable 하지 않음 (예: `Math.random()`) → 매번 unmount/mount → state 잃음.

---

## §3. Render phase vs Commit phase

### 3.1 두 phase
**render phase (interruptible)**:
- 컴포넌트 함수 호출.
- React Element tree 생성.
- diff 계산.
- → 결과는 메모리만. DOM 안 건듦.

**commit phase (sync, uninterruptible)**:
- DOM 실제 업데이트.
- ref 갱신.
- useLayoutEffect 실행.
- → 짧고 sync. 중단 X.

### 3.2 왜 분리?
**render 가 비싸도 commit 은 짧음**:
- render 중 더 급한 일 (사용자 입력) 발생 → render abort, 다시 시작.
- commit 은 항상 끝. 일관 상태 보장.

**Concurrent rendering 의 본질**:
- render 가 interruptible.
- 매 5ms 마다 brake (Scheduler).
- 다른 일 처리.

### 3.3 render 가 여러 번 호출됨
```typescript
function MyComp() {
    console.log("render");        // 같은 props/state 라도 두 번 출력 가능
    return <div>...</div>;
}
```

Strict Mode + Concurrent 에서 가능.

→ **render 는 idempotent + side-effect free**.

### 3.4 commit 시점에만 effect 실행
useEffect 는 commit 후. 그래서 render 가 abort 되어도 effect 안 실행.

useState 도 schedule 만, 다음 render 에 반영.

---

## §4. Fiber 아키텍처 (React 16+)

### 4.1 Fiber 가 무엇인가
**기존 (React 15 까지)**: render 가 recursive — 한 번 시작하면 끝까지 동기.
- 큰 tree 에선 메인 스레드 블록 → 사용자 입력 끊김.

**Fiber (React 16+)**: render 를 unit (Fiber node) 단위로 쪼갬. 각 단위 후 break point.

### 4.2 Fiber node
각 컴포넌트마다 Fiber object:
```typescript
{
    type: Counter,          // 컴포넌트
    props: { ... },
    state: { count: 0 },
    return: parentFiber,
    sibling: nextFiber,
    child: childFiber,
    alternate: prevFiber,   // 이전 render 의 Fiber
    // ...
}
```

→ linked list 구조. tree 처럼 children/sibling 연결.

### 4.3 work loop
```
while (workInProgress && !shouldYield()) {
    workInProgress = performUnitOfWork(workInProgress);
}
// shouldYield() — 5ms 지났거나 더 급한 일 있으면 true
// performUnitOfWork — 다음 Fiber 처리 (render)
```

→ 각 Fiber 처리 후 yield. 다른 일 처리 후 다시 시작.

### 4.4 priority
React 18+ 에선 update priority (Lane) 도입:
- urgent (사용자 입력) — 즉시.
- transition — non-urgent (useTransition).
- idle — 대기.

→ Concurrent 의 본질.

### 4.5 직접 알 필요 없음
보통 React 사용자는 Fiber 의 디테일 모름. 단:
- "render 가 interruptible" 메커니즘 = Fiber.
- Concurrent 가능 = Fiber.

---

## §5. Reconciliation 의 실전 함정

### 5.1 conditional render 시 state 잃음
```typescript
{cond ? <Counter /> : null}
```

cond 가 true → false → true 변할 때마다:
- false 시 unmount → state 잃음.
- true 다시 시 mount → state 0 부터.

→ state 보존 원하면 항상 mount + 표시만 숨김:
```typescript
<div style={{ display: cond ? "block" : "none" }}>
    <Counter />
</div>
```

### 5.2 같은 type, 다른 위치
```typescript
{cond ? (
    <div>
        <Counter />
    </div>
) : (
    <div>
        <span>placeholder</span>
        <Counter />
    </div>
)}
```

cond 변경 시 Counter 의 위치가 바뀜:
- 첫 번째 child → 두 번째 child.
- React 가 위치 기반으로 비교 → 다른 컴포넌트 취급 → state 잃음.

해결: key 추가
```typescript
<Counter key="counter" />
```

→ React 가 같은 컴포넌트 인식.

### 5.3 list reorder 시 state 잃음
```typescript
items.map(i => <Counter id={i.id} count={i.count} />)
```

reorder 시:
- key 없음 → 위치 기반 매칭 → state mismatch.
- key={i.id} → 정확한 매칭 → state 유지.

### 5.4 component vs JSX
```typescript
// ❌ render 함수 안에 컴포넌트 정의
function Parent() {
    function Child() { ... }       // render 마다 새 함수
    return <Child />;
}
```

→ Child 가 매번 새 type → 매번 unmount/mount → state 잃음.

해결: top level 정의.

---

## §6. memo / useMemo / useCallback (재 visit)

### 6.1 React.memo
```typescript
const MemoCounter = React.memo(Counter);
```

→ props shallow compare. 같으면 re-render skip.

### 6.2 useMemo / useCallback 와 짝
```typescript
function Parent() {
    const handleClick = useCallback(() => { ... }, []);    // stable reference
    const data = useMemo(() => compute(...), [deps]);     // stable reference

    return <MemoChild onClick={handleClick} data={data} />;
}
```

→ MemoChild 가 React.memo 면 props reference 안 바뀌어 re-render 안 됨.

### 6.3 함정
- React.memo 안 쓰면 useCallback/useMemo 무의미.
- shallow compare 라 깊은 객체 변경 못 잡음.

---

## §7. concurrent rendering 의 의미

### 7.1 기본 시나리오
```
사용자가 input 에 빠르게 타이핑
  ↓
매 keystroke 마다 setQuery → re-render
  ↓
큰 list filter (수천 row) 매번
  ↓
input 끊김 (메인 스레드 블록)
```

### 7.2 useTransition / useDeferredValue 적용
```typescript
const [isPending, startTransition] = useTransition();

function handleChange(e) {
    setQuery(e.target.value);                   // urgent
    startTransition(() => {
        setFiltered(filter(data, e.target.value));   // non-urgent
    });
}
```

→ filter 가 무거워도 input 안 끊김. React 가 input update 우선.

### 7.3 핵심
**Concurrent = render 를 우선순위로 schedule**. urgent 먼저, non-urgent 나중에 (또는 abort + 재시작).

---

## §8. Server Component 의 reconciliation

### 8.1 RSC tree
Next.js 13+ App Router 의 server component:
- server 가 Element tree 직렬화 (RSC payload).
- client 가 deserialize → reconciliation.
- client component 만 hydrate (interactive).

### 8.2 핵심
**server component 는 reconciliation 의 한쪽 끝 (server-rendered)**. client 가 받아서 normal reconciliation.

상세는 06-routing-patterns.md 참조.

---

## §9. 직접 구현 (mini Virtual DOM)

```typescript
type VNode = { type: string; props: any; children: VNode[] };

function createElement(type: string, props: any, ...children: VNode[]): VNode {
    return { type, props, children };
}

function render(vnode: VNode, container: HTMLElement) {
    const el = document.createElement(vnode.type);
    Object.entries(vnode.props || {}).forEach(([k, v]) => {
        el.setAttribute(k, v as string);
    });
    vnode.children.forEach(c => render(c, el));
    container.appendChild(el);
}

function diff(oldVNode: VNode, newVNode: VNode, dom: HTMLElement) {
    if (oldVNode.type !== newVNode.type) {
        // 통째 교체
        dom.parentElement?.replaceChild(createDom(newVNode), dom);
        return;
    }
    // props diff
    Object.entries(newVNode.props || {}).forEach(([k, v]) => {
        if (oldVNode.props[k] !== v) dom.setAttribute(k, v as string);
    });
    // children diff (key 기반 매칭 생략)
    newVNode.children.forEach((c, i) => {
        diff(oldVNode.children[i], c, dom.children[i] as HTMLElement);
    });
}
```

원리만. React 는 Fiber + key + concurrent + ... 매우 정밀.

---

## §10. 패턴 매트릭스

| 문제 | 패턴 |
|---|---|
| 효율적 DOM 업데이트 | Virtual DOM + diff |
| diff O(N) | type/key 휴리스틱 |
| list state 보존 | stable key (id) |
| interruptible render | Fiber + Scheduler |
| commit 일관성 | render/commit 분리 |
| 자식 re-render 방지 | React.memo + useCallback/useMemo |
| 무거운 update 미루기 | useTransition |

---

## §11. 학습 포인트

1. **Virtual DOM = JS 객체 tree**. 만들기 싸.
2. **diff = O(N) 휴리스틱** — type 다르면 통째로, list 는 key 매칭.
3. **key 안정성 = state 보존** — index X, id 사용.
4. **render phase = interruptible**, commit phase = sync.
5. **Fiber = render 의 unit + linked list** — interruptible 의 기반.
6. **render idempotent** — Strict Mode/Concurrent 에서 여러 번 호출 가능.
7. **conditional render → unmount → state 잃음** — display none 으로 보존.
8. **same type 다른 위치 → state mismatch** — key 명시.
9. **함수 안 컴포넌트 정의 X** — 매번 새 type.
10. **React.memo + useCallback/useMemo 짝** — props reference 안정.

### 추가 참고
- React Reconciliation 공식: https://react.dev/learn/preserving-and-resetting-state
- Lin Clark, [Inside React's Fiber Implementation](https://www.youtube.com/watch?v=ZCuYPiUIONs)
- Andrew Clark, [React Fiber Architecture](https://github.com/acdlite/react-fiber-architecture)
