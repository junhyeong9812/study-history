# 05. Modern JavaScript — Proxy, Reflect, Symbol

> 이 문서가 다루는 것: Proxy, Reflect, Symbol, WeakMap/WeakSet, ESM 등 modern JS 의 깊은 기능. Vue 3 reactivity, MobX, immer 같은 라이브러리의 기반.

---

## §0. 왜 알아야?

기본 React 사용엔 직접 안 쓰지만:
- **Vue 3 reactivity** — Proxy 기반.
- **MobX** — Proxy.
- **Immer** — Proxy.
- **TypeORM, Prisma** — Proxy 로 lazy loading.

→ 이걸 모르면 라이브러리가 마법으로 보임. 알면 직접 비슷한 것도 만들 수 있음.

---

## §1. Symbol — unique identifier

### 1.1 정의
```javascript
const sym = Symbol("description");
const sym2 = Symbol("description");
console.log(sym === sym2);    // false (각 Symbol 은 unique)
```

### 1.2 용도

#### 객체 key 로 (충돌 방지)
```javascript
const KEY = Symbol("internal");
obj[KEY] = "secret";

// 다른 라이브러리가 같은 KEY 만들 수 없음
```

#### well-known symbols (의미 있는 hook)
- `Symbol.iterator` — for-of 가능하게.
- `Symbol.asyncIterator` — for await-of.
- `Symbol.toPrimitive` — 형 변환 hook.

```javascript
class Range {
    [Symbol.iterator]() {
        let i = 0;
        return {
            next: () => i < 3 ? { value: i++, done: false } : { value: undefined, done: true }
        };
    }
}

for (const x of new Range()) console.log(x);    // 0, 1, 2
```

### 1.3 Symbol.for (registry)
```javascript
const a = Symbol.for("shared");
const b = Symbol.for("shared");
console.log(a === b);    // true (전역 registry)
```

### 1.4 함정
- Symbol key 는 `Object.keys` / `JSON.stringify` 에 안 보임.
- `Object.getOwnPropertySymbols()` 로 명시 접근.

---

## §2. Proxy — 객체 동작 가로채기

### 2.1 정의
객체의 기본 동작 (get, set, has, delete, ...) 을 가로채는 wrapper.

```javascript
const target = { name: "Alice", age: 30 };

const proxy = new Proxy(target, {
    get(obj, prop) {
        console.log(`getting ${prop}`);
        return obj[prop];
    },
    set(obj, prop, value) {
        console.log(`setting ${prop} = ${value}`);
        obj[prop] = value;
        return true;       // success
    }
});

proxy.name;            // "getting name" → "Alice"
proxy.name = "Bob";    // "setting name = Bob"
proxy.age;             // "getting age" → 30
```

### 2.2 traps (모든 hook)
- `get(obj, prop, receiver)` — property read.
- `set(obj, prop, value, receiver)` — property write.
- `has(obj, prop)` — `in` operator.
- `deleteProperty(obj, prop)` — `delete obj.prop`.
- `apply(target, thisArg, args)` — function call.
- `construct(target, args)` — `new`.
- `ownKeys(target)` — `Object.keys`.
- `getPrototypeOf(target)` — `Object.getPrototypeOf`.
- ... 13개 trap.

### 2.3 활용 예 — reactive object (Vue 3 의 핵심)
```javascript
const dependencies = new Map();   // prop → Set of effect functions
let activeEffect = null;

function reactive(obj) {
    return new Proxy(obj, {
        get(target, prop) {
            if (activeEffect) {
                const effects = dependencies.get(prop) || new Set();
                effects.add(activeEffect);
                dependencies.set(prop, effects);
            }
            return target[prop];
        },
        set(target, prop, value) {
            target[prop] = value;
            const effects = dependencies.get(prop);
            effects?.forEach(fn => fn());     // 변경 시 의존 effect 실행
            return true;
        }
    });
}

function effect(fn) {
    activeEffect = fn;
    fn();          // 호출 — get 이 dependencies 에 등록
    activeEffect = null;
}

const state = reactive({ count: 0 });
effect(() => console.log("count =", state.count));    // count = 0

state.count = 1;        // 자동: count = 1 출력
state.count = 5;        // count = 5
```

→ **Vue 3 reactivity 의 70% 가 이거**. ref/computed 는 추가.

### 2.4 활용 예 — validation wrapper
```javascript
const validated = new Proxy(user, {
    set(obj, prop, value) {
        if (prop === "age" && value < 0) throw new Error("age must be >= 0");
        if (prop === "email" && !value.includes("@")) throw new Error("invalid email");
        obj[prop] = value;
        return true;
    }
});

validated.age = -1;    // throw!
```

### 2.5 활용 예 — lazy loading
```javascript
function lazyLoad(loader) {
    let loaded = null;
    return new Proxy({}, {
        get(_, prop) {
            if (!loaded) loaded = loader();
            return loaded[prop];
        }
    });
}

const heavy = lazyLoad(() => loadHugeData());
heavy.foo;    // 처음 access 시 로드
```

### 2.6 함정
- Proxy 는 deep 안 됨 — nested 객체는 별도 wrap.
- 일부 environment 에서 비싼 — 매 access 마다 trap.
- 직접 동등성 (Proxy === Proxy 그 자체).

### 2.7 직접 활용
- 라이브러리 만들 때 매우 강력.
- 일반 앱 코드에선 거의 안 씀 (overkill).

---

## §3. Reflect — Proxy 의 짝

### 3.1 정의
객체 조작의 표준 API. Proxy trap 안에서 default 동작 호출.

```javascript
const proxy = new Proxy(target, {
    get(obj, prop, receiver) {
        console.log("intercept");
        return Reflect.get(obj, prop, receiver);    // ← default 동작
    }
});
```

### 3.2 왜 필요?
Proxy 의 trap 에서 "원래 동작" 호출하고 싶을 때:
- `obj[prop]` — receiver 등 미세 잃을 수 있음.
- `Reflect.get(obj, prop, receiver)` — 정확한 default.

### 3.3 메서드들
- `Reflect.get(target, prop, receiver)`
- `Reflect.set(target, prop, value, receiver)`
- `Reflect.has(target, prop)` — `in`.
- `Reflect.deleteProperty(target, prop)` — `delete`.
- `Reflect.apply(fn, thisArg, args)` — `fn.apply`.
- `Reflect.construct(Cls, args)` — `new Cls(...args)`.
- `Reflect.ownKeys(target)` — Symbol 포함 모든 own keys.

### 3.4 vs Object 메서드
| | Reflect | Object |
|---|---|---|
| `Reflect.has(o, "p")` | true/false | `"p" in o` |
| `Reflect.deleteProperty` | boolean 반환 | `delete` keyword |
| `Reflect.ownKeys` | Symbol 포함 | `Object.keys` (string 만) |

→ Reflect 가 더 일관, 보통 함수.

### 3.5 일반 사용 X
Proxy 짝궁. 직접 코드에선 거의 안 씀.

---

## §4. WeakMap / WeakSet (재 visit)

### 4.1 일반 Map 의 GC 문제
```javascript
const cache = new Map();
function trackUser(user) { cache.set(user, ...); }

let user = { name: "Alice" };
trackUser(user);
user = null;        // user 객체 다른 참조 X
// 하지만 cache.has(user) — 여전히 살아있음 (cache 가 strong reference)
```

→ memory leak.

### 4.2 WeakMap
key 가 weak — 다른 곳에서 참조 잃으면 자동 GC.

```javascript
const cache = new WeakMap();
let user = { name: "Alice" };
cache.set(user, ...);
user = null;
// 다음 GC cycle 에 cache 의 entry 도 자동 제거.
```

### 4.3 한계
- key 는 객체만 (primitive X).
- iteration X (`weak` 라 보장 못 함).
- size X.
- 정확한 entry 수 알 수 없음.

→ "객체에 부가 정보 attach" 용도. iteration 필요하면 일반 Map.

### 4.4 use case — private fields
```javascript
const _private = new WeakMap();

class Counter {
    constructor() {
        _private.set(this, { count: 0 });
    }
    increment() {
        _private.get(this).count++;
    }
    get count() {
        return _private.get(this).count;
    }
}
```

→ 외부에서 `_private` 모름 → 진짜 private. (현재는 `#privateField` 문법이 더 표준.)

### 4.5 use case — DOM node metadata
```javascript
const nodeData = new WeakMap();

function setDataFor(node, data) { nodeData.set(node, data); }
function getDataFor(node) { return nodeData.get(node); }

// node 가 DOM 에서 제거 → GC → nodeData 도 자동.
```

### 4.6 WeakSet
같은 의미. 중복 안 가짐. value 만.
```javascript
const visited = new WeakSet();
visited.add(node);
visited.has(node);    // true
```

---

## §5. ESM Deep — modules 의 완전한 이해

### 5.1 import 의 lazy nature
```javascript
import { foo } from "./bar.js";
```

→ 컴파일 타임에 정적 분석. circular dependency 가능 (bindings).

```javascript
// a.js
import { b } from "./b.js";
export const a = "A";

// b.js
import { a } from "./a.js";
export const b = "B-" + a;    // 'B-undefined' 가 아닌 동적 binding
```

→ ESM 은 binding (live reference). CommonJS 와 다름.

### 5.2 dynamic import
```javascript
const module = await import("./heavy.js");
module.runHeavy();
```

→ runtime 분석. code splitting 의 기반. Webpack/Vite 의 chunk 분리.

### 5.3 top-level await
```javascript
// module.js
const data = await fetchConfig();
export const config = data;
```

ESM module 의 top level 에서 await 가능 (Node 14+, browser ESM).

→ 모듈 초기화 자체가 비동기.

### 5.4 import maps (browser)
```html
<script type="importmap">
{
    "imports": {
        "react": "https://esm.sh/react@19"
    }
}
</script>
<script type="module">
    import React from "react";   // 위 매핑 따라
</script>
```

→ bundler 없이 browser 에서 직접 import 가능.

### 5.5 tree shaking
ESM 은 정적 분석 가능 → dead code 제거. CommonJS 어려움.

```javascript
// big-lib.js
export function used() { ... }
export function unused() { ... }     // 안 쓰면 번들 안 들어감

// app.js
import { used } from "./big-lib";
```

---

## §6. 기타 modern features (간단)

### 6.1 Optional chaining (`?.`)
```javascript
const street = user?.address?.street;     // null/undefined 안전
fn?.();    // fn 이 있으면 호출
arr?.[0];   // arr 가 있으면 인덱스
```

### 6.2 Nullish coalescing (`??`)
```javascript
const port = process.env.PORT ?? 3000;     // null/undefined 만 default
// vs ||
const port = process.env.PORT || 3000;     // falsy (0, "") 도 default — 위험
```

### 6.3 Logical assignment
```javascript
a ??= b;     // a 가 null/undefined 이면 b 할당
a ||= b;     // a 가 falsy 이면 b
a &&= b;     // a 가 truthy 이면 b
```

### 6.4 Numeric separators
```javascript
const million = 1_000_000;
const hex = 0xFF_FF_FF;
```

### 6.5 String.replaceAll
```javascript
"a.b.c".replaceAll(".", "-");    // "a-b-c"
// 이전엔 regex /\./g 필요했음.
```

### 6.6 Array.at(-1)
```javascript
[1, 2, 3].at(-1);    // 3 (마지막)
arr.at(-2);           // 뒤에서 두 번째
```

### 6.7 Object.hasOwn (vs hasOwnProperty)
```javascript
Object.hasOwn(obj, "key");    // 표준 + 안전
// obj.hasOwnProperty("key") 는 prototype 변조 시 위험
```

### 6.8 structuredClone (deep clone)
```javascript
const deep = structuredClone(obj);    // deep copy. JSON 보다 풍부 (Date, Map, Set 등).
```

### 6.9 Array grouping (ES2024)
```javascript
arr.group(item => item.category);
// → { category1: [...], category2: [...] }
```

---

## §7. 함정 + 흔한 오해

### 7.1 Proxy = magic 으로 보지 말 것
- 모든 동작이 trap 으로 가로채짐.
- 디버깅 어려움 (간접 호출).
- 단순 케이스엔 over-engineering.

### 7.2 Reflect 직접 호출 흔치 X
- Proxy trap 안에서만.
- 일반 코드는 `obj.prop` 직접.

### 7.3 Symbol 남용 X
- 일반 객체 key 는 string 권장.
- Symbol 은 충돌 회피 / well-known hook 용.

### 7.4 WeakMap iteration X
- size 도 모름.
- iteration 필요하면 일반 Map (단 leak 주의).

---

## §8. 학습 포인트

1. **Symbol = unique id**. 충돌 회피 + well-known hook (iterator).
2. **Proxy = 객체 동작 trap**. Vue/MobX/Immer 의 기반.
3. **Reflect = Proxy 의 default 동작**. 일관 API.
4. **WeakMap = key weak**. leak 방지.
5. **ESM = binding (live)** + tree shaking + dynamic import.
6. **top-level await** — module 초기화 비동기.
7. **`?.` optional chaining** — null safety.
8. **`??` nullish coalescing** — `||` 보다 정확.
9. **structuredClone** — deep clone 표준.
10. **Object.hasOwn** — `hasOwnProperty` 보다 안전.

### 추가 참고
- MDN Proxy: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy
- MDN Reflect: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Reflect
- Vue 3 Reactivity: https://vuejs.org/guide/extras/reactivity-in-depth.html
- ESM specification: https://tc39.es/ecma262/#sec-modules
