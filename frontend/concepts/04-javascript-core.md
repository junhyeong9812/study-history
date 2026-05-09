# 04. JavaScript 핵심

> 이 문서가 다루는 것: closure, prototype, this binding, async/await, Promise, generator, iterator. React/TypeScript 이해의 토대.

---

## §0. 왜 JS 핵심을 알아야?

React 는 JS 위에 build. hook 의 closure 함정, async 흐름, Promise chain, prototype-based class — 모두 JS 핵심.

JS 모르면 React 도 매번 마법.

---

## §1. Closure

### 1.1 정의
함수가 정의될 때 그 환경 (스코프) 을 capture. 함수가 외부 스코프 변수에 접근 가능.

### 1.2 예
```javascript
function makeCounter() {
    let count = 0;          // 외부 변수
    return function() {
        count++;             // closure 가 count 참조
        return count;
    };
}

const counter = makeCounter();
counter();    // 1
counter();    // 2
counter();    // 3
```

`makeCounter` 가 끝나도 `count` 는 살아있음. inner 함수가 capture.

### 1.3 활용
- 모듈 패턴 (private 변수).
- callback 에서 외부 변수 사용.
- 함수형 프로그래밍 (currying, partial application).

### 1.4 React 의 closure 함정
```typescript
function Counter() {
    const [n, setN] = useState(0);
    useEffect(() => {
        setInterval(() => {
            setN(n + 1);     // n 이 closure 안에서 stale
        }, 1000);
    }, []);
}
```

`useEffect` 람다가 mount 시점의 `n=0` capture. 매번 0+1=1.

해결: functional update, ref, deps.

### 1.5 핵심 통찰
**closure = "이 함수가 생성된 환경" 의 snapshot**. 함수 호출 시점의 환경 X.

---

## §2. Prototype

### 2.1 JS 의 객체 모델
JS 는 prototype-based. 모든 객체가 다른 객체 (prototype) 를 참조.

```javascript
const arr = [1, 2, 3];
arr.map(x => x * 2);
// arr 자신엔 map 없음. Array.prototype.map 사용.
// arr.__proto__ === Array.prototype
```

### 2.2 prototype chain
```javascript
arr → Array.prototype → Object.prototype → null
```

`arr.toString()` 호출 시:
1. arr 자신에 toString 있나? No.
2. Array.prototype 에 있나? Yes (배열 형식).
3. → 호출.

### 2.3 class 도 prototype
```javascript
class Animal {
    constructor(name) { this.name = name; }
    sayHello() { return `Hello, ${this.name}`; }
}

const a = new Animal("Cat");
a.sayHello();
// a.__proto__ === Animal.prototype
// Animal.prototype.sayHello 호출
```

→ class 는 syntax sugar. 내부는 prototype.

### 2.4 React 와 prototype
거의 직접 안 만남. 단:
- React.Component 가 class (prototype 기반).
- 함수 컴포넌트는 prototype 무관.

---

## §3. this binding

### 3.1 this 의 4 규칙
JS 의 `this` 는 호출 방식에 따라 다름:

#### 1) default
```javascript
function f() { return this; }
f();    // → window (browser) or undefined (strict mode)
```

#### 2) implicit (메서드 호출)
```javascript
const obj = {
    name: "Alice",
    greet() { return this.name; }
};
obj.greet();    // "Alice" (this = obj)
```

#### 3) explicit (call/apply/bind)
```javascript
function greet() { return this.name; }
greet.call({ name: "Bob" });    // "Bob"
greet.apply({ name: "Bob" });   // "Bob"
const bound = greet.bind({ name: "Bob" });
bound();                         // "Bob"
```

#### 4) new
```javascript
function Animal(name) { this.name = name; }
const a = new Animal("Cat");    // this = 새 객체
```

### 3.2 arrow function 은 다름
```javascript
const obj = {
    name: "Alice",
    greet: () => this.name        // ❌ this 가 obj 아님
};
obj.greet();    // undefined (또는 window.name)
```

→ arrow function 은 **lexical this** (정의 시점의 this).

### 3.3 React 와 this
- 함수 컴포넌트는 `this` 무관.
- class 컴포넌트는 `this.setState` etc — bind 필요했음 (legacy).
- arrow function 으로 자동 bind.

### 3.4 함정
- callback 으로 메서드 전달 시 `this` 잃음:
```javascript
button.onclick = obj.greet;     // this = button (or undefined)
button.onclick = () => obj.greet();    // 또는
button.onclick = obj.greet.bind(obj);   // 또는 bind
```

---

## §4. Promise

### 4.1 정의
미래에 완료될 비동기 작업 표현.

```javascript
const promise = new Promise((resolve, reject) => {
    setTimeout(() => {
        if (Math.random() > 0.5) resolve("success");
        else reject(new Error("fail"));
    }, 1000);
});

promise.then(value => console.log(value))
       .catch(err => console.error(err));
```

### 4.2 3 state
- **pending**: 진행 중.
- **fulfilled**: resolve 호출됨.
- **rejected**: reject 호출됨.

state 한 번 정해지면 변경 X.

### 4.3 chain
```javascript
fetch("/api")
    .then(r => r.json())
    .then(data => process(data))
    .catch(err => console.error(err));
```

각 then 의 반환값이 다음 then 으로. error 는 catch 로 떨어짐.

### 4.4 microtask queue
Promise 의 then/catch callback 은 **microtask queue** 에 들어감.

```javascript
console.log("1");
Promise.resolve().then(() => console.log("2"));
console.log("3");
// 출력: 1 → 3 → 2
// (microtask 는 현재 stack 끝난 후, 다음 macrotask 전)
```

자세한 event loop 는 `07-browser-event-loop.md`.

### 4.5 Promise.all / Promise.race / Promise.allSettled
```javascript
// 모두 성공해야 resolve
const [a, b, c] = await Promise.all([fetchA(), fetchB(), fetchC()]);

// 가장 빨리 settle 된 것 (성공 또는 실패)
const winner = await Promise.race([slow, fast]);

// 모두 settle (성공/실패 무관)
const results = await Promise.allSettled([...]);
// results = [{ status: "fulfilled", value }, { status: "rejected", reason }, ...]
```

---

## §5. async / await

### 5.1 문법
```javascript
async function fetchData() {
    const res = await fetch("/api");
    const data = await res.json();
    return data;
}
```

→ Promise chain 의 syntax sugar.

### 5.2 동작
```javascript
async function f() {
    const x = await somePromise;
    return x + 1;
}
// = 
function f() {
    return somePromise.then(x => x + 1);
}
```

### 5.3 try/catch
```javascript
async function f() {
    try {
        const x = await mayFail();
        return x;
    } catch (err) {
        console.error(err);
        throw err;
    }
}
```

### 5.4 async 함수는 항상 Promise 반환
```javascript
async function f() { return 42; }
f();    // Promise<42>, not 42
```

### 5.5 함정
- forEach + await — forEach 가 await 안 기다림:
```javascript
items.forEach(async (item) => {
    await fetch(...);    // 다음 iteration 안 기다림
});
```
해결: `for...of` + await:
```javascript
for (const item of items) {
    await fetch(...);
}
```

---

## §6. Iterator + Generator

### 6.1 iterator protocol
객체가 next() 메서드 가지면 iterator. `{ value, done }` 반환.

```javascript
const iter = {
    i: 0,
    next() {
        if (this.i < 3) return { value: this.i++, done: false };
        return { value: undefined, done: true };
    }
};

for (const x of { [Symbol.iterator]() { return iter; } }) {
    console.log(x);    // 0, 1, 2
}
```

### 6.2 Symbol.iterator
객체에 `[Symbol.iterator]()` 가 있으면 iterable. for-of 가능.

### 6.3 Generator function
```javascript
function* counter() {
    yield 1;
    yield 2;
    yield 3;
}

const gen = counter();
gen.next();    // { value: 1, done: false }
gen.next();    // { value: 2, done: false }
gen.next();    // { value: 3, done: false }
gen.next();    // { value: undefined, done: true }
```

→ `function*` 가 자동으로 iterator 생성. `yield` 가 next 의 결과.

### 6.4 활용
- 무한 sequence 표현.
- async iteration (`for await...of`).
- Redux-saga 같은 비동기 흐름 라이브러리.

VulnScope 미사용. 라이브러리 내부에서.

---

## §7. ES6+ 의 주요 syntax

### 7.1 Arrow function
```javascript
const add = (a, b) => a + b;
const greet = name => `Hello, ${name}`;
```
- lexical this.
- 짧음.
- 단점: this/arguments/new.target 없음.

### 7.2 Spread / Rest
```javascript
// Spread
const arr2 = [...arr1, 4, 5];
const obj2 = { ...obj1, name: "X" };
fn(...args);

// Rest
function fn(...args) { ... }
const [first, ...rest] = arr;
const { a, ...rest } = obj;
```

### 7.3 Destructuring
```javascript
const { name, age } = user;
const [x, y] = pair;
const { name = "default" } = user;     // default value
const { name: userName } = user;        // rename
```

### 7.4 Template literals
```javascript
`Hello, ${name}! You have ${count} messages.`

// tagged
function tag(strings, ...values) { ... }
tag`Hello, ${name}!`
```

### 7.5 Optional chaining + nullish coalescing
```javascript
const street = user?.address?.street;       // ?. (chain)
const port = process.env.PORT ?? 3000;     // ?? (null/undefined 만)
```

### 7.6 Object shorthand
```javascript
const name = "Alice";
const user = { name };    // = { name: name }

const obj = {
    greet() { ... }       // = greet: function() { ... }
};
```

---

## §8. Module (ESM)

### 8.1 import / export
```javascript
// math.js
export function add(a, b) { return a + b; }
export const PI = 3.14;
export default function multiply(a, b) { return a * b; }

// main.js
import multiply, { add, PI } from "./math.js";
```

### 8.2 dynamic import
```javascript
const module = await import("./math.js");
module.add(1, 2);
```

→ lazy load. code splitting.

### 8.3 ESM vs CommonJS
| | ESM (import/export) | CommonJS (require/module.exports) |
|---|---|---|
| 표준 | ES2015+ | Node 초기 |
| 비동기 | async (top-level await) | sync |
| static | 정적 분석 가능 | 동적 require |
| tree shaking | YES | 어려움 |

**modern**: ESM 권장. Node 도 점진 전환.

VulnScope (Next.js) 는 ESM.

---

## §9. WeakMap / WeakSet

### 9.1 일반 Map 의 GC 문제
```javascript
const cache = new Map();
function trackUser(user) { cache.set(user, ...); }

// user 객체를 어디선가 잃어도 cache 에 남음 → GC 안 됨 → leak.
```

### 9.2 WeakMap
key 가 GC 가능. 다른 곳에서 참조 잃으면 자동 제거.

```javascript
const cache = new WeakMap();
cache.set(user, ...);
// user 다른 참조 잃으면 cache 에서도 자동 제거.
```

### 9.3 한계
- key 만 weak. value 는 strong.
- iteration X (weak 라 보장 못 함).
- size X.

→ 정확히 "객체에 부가 정보 attach + leak 방지" 용도.

---

## §10. 함정 모음

### 10.1 == vs ===
```javascript
0 == "0"      // true (강제 변환)
0 === "0"     // false (엄격)
```
→ **항상 ===** 사용.

### 10.2 NaN
```javascript
NaN === NaN    // false!
isNaN(NaN)     // true
Number.isNaN(NaN)    // true (안전)
```

### 10.3 typeof null
```javascript
typeof null    // "object" (역사적 bug)
typeof undefined    // "undefined"
```

### 10.4 falsy values
- `false`, `0`, `""`, `null`, `undefined`, `NaN`.
- 다른 모든 값 truthy (빈 객체 `{}` 도 truthy!).

### 10.5 floating point
```javascript
0.1 + 0.2    // 0.30000000000000004 (IEEE 754)
```
→ 금융 계산 시 BigInt 또는 cents 로 정수.

---

## §11. 학습 포인트

1. **closure = 정의 시점 환경 capture**. 호출 시점 환경 X.
2. **prototype = 객체 → 객체 → null chain**. class 는 sugar.
3. **this 의 4 규칙** + arrow function 은 lexical this.
4. **Promise = 비동기 결과**, 3 state.
5. **microtask vs macrotask** — Promise 는 microtask.
6. **async/await = Promise sugar**.
7. **forEach + await 함정** — for-of 사용.
8. **iterator/generator** — 무한 sequence + 비동기 흐름.
9. **Spread/Rest, destructuring, optional chaining** — modern syntax.
10. **ESM vs CJS** — modern 은 ESM.
11. **WeakMap = key GC 가능** — leak 방지.
12. **항상 ===** + falsy values 인지.

### 추가 참고
- MDN JavaScript: https://developer.mozilla.org/en-US/docs/Web/JavaScript
- Kyle Simpson, *You Don't Know JS* 시리즈.
- Lydia Hallie, [JavaScript Visualized](https://dev.to/lydiahallie/javascript-visualized-the-javascript-engine-4cdf).
