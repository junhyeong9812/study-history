# 07. Browser Event Loop

> 이 문서가 다루는 것: JS 의 single-threaded + non-blocking 모델, microtask vs macrotask, requestAnimationFrame, 비동기 timing 의 본질.
> 이걸 모르면: setTimeout 0 vs Promise 의 차이 모름, "왜 useEffect 가 paint 후" 모름, 비동기 race condition 디버깅 못 함.

---

## §0. 왜 알아야?

### 0.1 JS 는 single-threaded
- 한 번에 한 작업.
- 비싼 동기 작업 = UI 블록.
- but fetch, setTimeout, event listener 같은 비동기는 어떻게?

### 0.2 답: Event Loop
브라우저/Node 가 JS 의 비동기 처리 메커니즘. queue + loop.

이걸 알면:
- React 의 batching 이해.
- Promise 와 setTimeout timing 이해.
- React 의 render 가 어떻게 interrupt 되는지.

---

## §1. JS 실행 모델

### 1.1 Call Stack
```javascript
function a() { b(); }
function b() { c(); }
function c() { console.log("hi"); }
a();
```

실행 시 stack:
```
| c() |
| b() |
| a() |
```

c 끝 → pop → b 끝 → pop → a 끝 → pop → empty.

### 1.2 single thread = 한 stack
JS 엔진은 한 스레드. Call Stack 1개.

비싼 함수 = stack 에 오래 남음 = UI freeze.

### 1.3 Web APIs
브라우저가 제공:
- `setTimeout`, `setInterval`
- `fetch`, `XMLHttpRequest`
- `addEventListener`
- DOM API
- `requestAnimationFrame`
- ... 등

이들은 **JS 엔진 외부 (브라우저 native code)** 에서 동작. JS 스레드 안 막음.

---

## §2. Event Loop 의 본질

### 2.1 흐름
```
1. Call Stack 에 동기 코드 실행
2. 비동기 작업 (setTimeout 등) → Web API 에 위임 → 별도 thread 에서 처리
3. 비동기 완료 → Callback Queue 에 push
4. Call Stack 비면 → Queue 에서 callback pop → Stack 으로
5. (반복)
```

### 2.2 시각화
```
[Call Stack]               [Web APIs]            [Callback Queue]
    |                          |                       |
   실행                       타이머/fetch              완료된 callback
    ↓                          ↓                       ↑
   pop                        push                    push
                                                       ↑
                          (Event Loop)                 |
                          비면 가져옴 ───────────────────┘
```

### 2.3 macrotask 의 예
- `setTimeout`, `setInterval`
- `setImmediate` (Node)
- DOM event (click, etc)
- I/O (network response)

### 2.4 microtask 의 예
- `Promise.then/catch/finally`
- `queueMicrotask`
- `MutationObserver`

---

## §3. macrotask vs microtask

### 3.1 핵심 규칙
**Call Stack 비면**:
1. 모든 microtask 처리 (queue 비울 때까지).
2. macrotask 1개 처리.
3. 다시 모든 microtask.
4. macrotask 1개.
5. ...

→ **microtask 가 macrotask 보다 우선**. 그리고 macrotask 사이마다 microtask 다 비움.

### 3.2 예
```javascript
console.log("1");
setTimeout(() => console.log("2"), 0);          // macrotask
Promise.resolve().then(() => console.log("3")); // microtask
console.log("4");

// 출력:
// 1
// 4
// 3   ← microtask 먼저
// 2   ← macrotask 다음
```

### 3.3 더 복잡한 예
```javascript
console.log("A");
setTimeout(() => {
    console.log("B");
    Promise.resolve().then(() => console.log("C"));
}, 0);
Promise.resolve().then(() => {
    console.log("D");
    setTimeout(() => console.log("E"), 0);
});
console.log("F");

// 출력: A, F, D, B, C, E
```

해석:
1. A (sync)
2. setTimeout 등록 (macro queue)
3. Promise.then 등록 (micro queue)
4. F (sync)
5. Stack 비움 → microtask 처리 → D
6. D 안의 setTimeout 등록
7. micro 비움 → macrotask 1개 → B
8. B 안의 Promise.then 등록
9. micro 처리 → C
10. macrotask → E

### 3.4 핵심 통찰
**Promise.then 은 즉시 실행 X**. microtask queue 에. setTimeout 0 보다 먼저, sync 보다 나중.

→ `setTimeout(fn, 0)` 는 "다음 macrotask 에" 의 의미. "0 ms 후" 가 아님.

### 3.5 함정 — microtask 무한 loop
```javascript
function loop() {
    Promise.resolve().then(loop);
}
loop();
// → microtask queue 가 영원히 안 비움 → macrotask 영원히 안 됨 → UI freeze
```

→ microtask 안에서 microtask 큐잉은 위험.

---

## §4. requestAnimationFrame (rAF)

### 4.1 정의
다음 paint 직전 callback 실행. 60fps = 16.67ms 마다.

```javascript
function animate() {
    // 위치 업데이트
    element.style.left = `${x++}px`;
    requestAnimationFrame(animate);
}
requestAnimationFrame(animate);
```

### 4.2 setTimeout 과 차이
- `setTimeout(fn, 16)` — 16ms 후 실행. paint 와 안 맞음.
- `requestAnimationFrame(fn)` — paint 와 sync. 부드러움.

### 4.3 timing
event loop 안에서 rAF 의 위치:
```
1. macrotask 처리
2. microtask 처리
3. (paint 시점) rAF callback
4. paint
5. 다음 cycle
```

→ rAF 가 paint 직전 callback. layout/paint 이 일관 timing.

### 4.4 활용
- 부드러운 animation (CSS transition 안 쓸 때).
- DOM measure 후 update 동기화.
- React 의 Concurrent rendering 이 사용 (5ms break point).

### 4.5 cancelAnimationFrame
```javascript
const id = requestAnimationFrame(fn);
cancelAnimationFrame(id);
```

---

## §5. requestIdleCallback

### 5.1 정의
브라우저가 idle (할 일 없음) 일 때 callback 실행. 우선순위 매우 낮음.

```javascript
requestIdleCallback((deadline) => {
    while (deadline.timeRemaining() > 0 && tasks.length) {
        doWork(tasks.pop());
    }
}, { timeout: 5000 });
```

### 5.2 활용
- 우선순위 낮은 작업 (analytics, logging, prefetch).
- React 의 옛 Scheduler 가 사용 (현재 MessageChannel + setTimeout 조합).

### 5.3 한계
- Safari 미지원.
- `timeout` 으로 강제 deadline 설정.

---

## §6. queueMicrotask

### 6.1 정의
직접 microtask queue 에 callback 추가.

```javascript
queueMicrotask(() => console.log("micro"));
```

= `Promise.resolve().then(() => console.log("micro"))` 와 같음.

### 6.2 차이
- `queueMicrotask` 가 더 직접적. Promise allocation 안 함.
- 라이브러리 내부에서 사용 (외부 API 의 sync/async 일관 시점 만들기).

---

## §7. React 의 batching

### 7.1 React 18+ automatic batching
```typescript
function handleClick() {
    setA(1);
    setB(2);
    setC(3);
    // → 1번 re-render (모든 setState batch)
}

setTimeout(() => {
    setA(1);
    setB(2);
    // → React 18+ 부터 1번 re-render (이전엔 2번)
}, 0);

await fetch(...);
setA(1);
setB(2);
// → 1번 re-render
```

### 7.2 메커니즘
- React 가 setState 호출들을 microtask 로 묶음.
- 모든 setState 끝나고 1번 render.

→ unnecessary re-render 회피.

### 7.3 강제 sync
```typescript
import { flushSync } from "react-dom";

flushSync(() => {
    setA(1);
});
// → 즉시 render + DOM update 끝남.
setB(2);
// → 다음 batch.
```

→ DOM measure 같은 타이밍 critical 시.

---

## §8. async / await 의 timing

### 8.1 핵심
```javascript
async function f() {
    console.log("1");
    await Promise.resolve();
    console.log("2");
}
console.log("A");
f();
console.log("B");

// 출력:
// A
// 1
// B   ← await 가 microtask 로 분리
// 2
```

→ `await` = "여기서 sync 종료, microtask 로 나머지". `then` 과 본질 같음.

### 8.2 함정
```javascript
async function withTimer() {
    console.log("start");
    await new Promise(r => setTimeout(r, 0));    // macrotask 기다림
    console.log("end");
}
```

`setTimeout 0` 이 macrotask → microtask 다 비운 후 → 그 다음 timing.

---

## §9. Long Task + 5ms 규칙

### 9.1 Long Task = 50ms+
브라우저 perf API 가 정의. 50ms 이상 sync 작업.

→ UI freeze 인지됨.

### 9.2 React Concurrent 의 5ms
React Scheduler 가 5ms 마다 yield (`shouldYield()` true). Long Task 회피.

→ render 가 길어도 사용자 입력 받음.

### 9.3 직접 측정
```javascript
const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
        console.log("Long task:", entry.duration, "ms");
    }
});
observer.observe({ entryTypes: ["longtask"] });
```

---

## §10. VulnScope 의 timing 활용

### 10.1 useEffect 의 timing
- render → paint → useEffect (async, microtask).
- useLayoutEffect — render → DOM update → useLayoutEffect (sync) → paint.

### 10.2 자동 navigate
```typescript
useEffect(() => {
    if (done || failed) {
        const id = setTimeout(() => router.push(...), 1500);
        return () => clearTimeout(id);
    }
}, [done, failed]);
```

`setTimeout` 으로 1.5초 후 navigate. cleanup 으로 cancel.

### 10.3 SSE 의 비동기 흐름
EventSource event 가 macrotask. listener 안의 setState 가 React batch.

---

## §11. 직접 구현 (mini event loop)

```javascript
const macroQueue = [];
const microQueue = [];

function setTimeout(fn, ms) {
    setTimeout_native(() => macroQueue.push(fn), ms);
}

function queueMicrotask(fn) {
    microQueue.push(fn);
}

function loop() {
    while (true) {
        // 1. microtask 모두 처리
        while (microQueue.length) {
            const fn = microQueue.shift();
            fn();
        }
        // 2. macrotask 1개
        if (macroQueue.length) {
            const fn = macroQueue.shift();
            fn();
        }
    }
}
```

원리. browser/Node 는 더 복잡 (rAF, idle, timer, I/O 등 다양 source).

---

## §12. 학습 포인트

1. **single thread + Web APIs** — 비동기는 외부 위임.
2. **Event Loop = Stack 비면 Queue 처리**.
3. **microtask 우선** — Promise > setTimeout 0.
4. **macrotask 사이마다 microtask 다 비움**.
5. **rAF = paint 직전** — animation 표준.
6. **requestIdleCallback** — 우선순위 낮은.
7. **React 18+ automatic batching** — setState 묶음.
8. **flushSync** — 강제 sync.
9. **await = microtask 로 분리** — sync 끝.
10. **Long Task = 50ms+**, React Concurrent 가 5ms break.

### 추가 참고
- Jake Archibald, [In The Loop](https://www.youtube.com/watch?v=cCOL7MC4Pl0)
- Philip Roberts, [What the heck is the event loop anyway?](https://www.youtube.com/watch?v=8aGhZQkoFbQ)
- MDN Event Loop: https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop
