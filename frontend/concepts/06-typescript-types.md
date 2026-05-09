# 06. TypeScript Type System

> 이 문서가 다루는 것: TypeScript 의 핵심 — generic, conditional types, mapped types, template literal types, narrowing, discriminated union, type vs interface, satisfies. VulnScope 가 사용한 타입 패턴.

---

## §0. 왜 TypeScript?

### 0.1 JS 의 한계
- 타입 0. runtime 까지 에러 안 잡힘.
- IDE 자동완성 약함.
- refactoring 위험.

### 0.2 TS 의 통찰
- compile 타임 검증.
- IDE 풍부한 자동완성.
- runtime 비용 0 (TS → JS, 타입 정보 제거).

### 0.3 함정
- "any" 남용 → 타입 의미 잃음.
- 너무 정교한 타입 → 가독성 ↓.
- runtime 안전은 별도 (Zod 같은 검증).

---

## §1. 기본 — primitive + object

### 1.1 primitive
```typescript
let s: string = "hello";
let n: number = 42;
let b: boolean = true;
let u: undefined = undefined;
let nl: null = null;
let big: bigint = 100n;
let sym: symbol = Symbol("x");
```

### 1.2 array / tuple
```typescript
let arr: number[] = [1, 2, 3];
let arr2: Array<number> = [1, 2, 3];

let pair: [string, number] = ["age", 30];        // tuple — 고정 길이/타입
```

### 1.3 object / interface / type
```typescript
type User = { name: string; age: number };
interface User2 { name: string; age: number }

const u: User = { name: "Alice", age: 30 };
```

### 1.4 함수
```typescript
function add(a: number, b: number): number { return a + b; }
const add: (a: number, b: number) => number = (a, b) => a + b;
```

### 1.5 union / intersection
```typescript
type Status = "queued" | "running" | "done" | "failed";    // union
type WithId = User & { id: string };                        // intersection (둘 다 만족)
```

---

## §2. type vs interface

### 2.1 거의 같음
```typescript
type A = { x: number };
interface B { x: number; }
```

### 2.2 차이
| | type | interface |
|---|---|---|
| primitive alias | type Age = number; | (불가) |
| union/intersection | type X = A | B; | (불가) |
| extend | A & B | extends A |
| declaration merging | (불가) | 같은 이름 여러 번 — merge |

### 2.3 권장
**type 을 default**. interface 는 declaration merging 필요할 때 (drd third-party 확장 등).

VulnScope 는 거의 type 사용.

---

## §3. Generic — type 의 함수

### 3.1 기본
```typescript
function identity<T>(x: T): T { return x; }
identity<string>("hello");    // T = string
identity(42);                   // 추론
```

`<T>` = type parameter. 호출 시 결정.

### 3.2 generic interface / type
```typescript
type Box<T> = { value: T };
type StringBox = Box<string>;       // { value: string }

interface Repository<T> {
    save(item: T): T;
    findById(id: string): T | null;
}
```

### 3.3 constraints
```typescript
function getId<T extends { id: string }>(obj: T): string {
    return obj.id;
}

getId({ id: "abc", name: "x" });    // OK
getId({ name: "x" });                 // ❌ id 없음
```

`extends` = "T 가 이 shape 만족해야".

### 3.4 default type
```typescript
type Container<T = string> = { value: T };
const c: Container = { value: "hello" };    // T default = string
```

### 3.5 multiple
```typescript
function pair<A, B>(a: A, b: B): [A, B] { return [a, b]; }
pair(1, "x");    // [number, string]
```

### 3.6 React 의 generic
```typescript
const [count, setCount] = useState<number>(0);
const ref = useRef<HTMLInputElement>(null);
const { data } = useQuery<ScanView>({ ... });
```

→ 라이브러리가 generic 으로 정의. 호출 시 type 명시 또는 추론.

---

## §4. Type narrowing

### 4.1 typeof guard
```typescript
function fn(x: string | number) {
    if (typeof x === "string") {
        x.toUpperCase();    // OK — narrowed to string
    } else {
        x.toFixed(2);        // OK — number
    }
}
```

### 4.2 in guard
```typescript
type Cat = { meow: () => void };
type Dog = { bark: () => void };

function fn(animal: Cat | Dog) {
    if ("meow" in animal) animal.meow();
    else animal.bark();
}
```

### 4.3 instanceof guard
```typescript
function fn(err: unknown) {
    if (err instanceof Error) {
        console.log(err.message);
    }
}
```

### 4.4 user-defined type guard
```typescript
function isString(x: unknown): x is string {
    return typeof x === "string";
}

function fn(x: unknown) {
    if (isString(x)) {
        x.toUpperCase();    // OK
    }
}
```

`x is string` = type predicate. true 반환 시 x 가 string.

### 4.5 control flow analysis
```typescript
function fn(x: string | null) {
    if (x === null) return;
    x.toUpperCase();    // null 제외됨
}
```

TS 가 자동으로 narrowing.

### 4.6 VulnScope 의 narrowing
```typescript
const { data, error } = await client.GET(...);
if (error) {
    return;
}
data.id;    // OK — error 제외 후 data non-null
```

---

## §5. Discriminated union

### 5.1 패턴
```typescript
type Result<T> =
    | { status: "success"; data: T }
    | { status: "error"; error: Error };

function fn(r: Result<User>) {
    if (r.status === "success") {
        r.data.name;        // OK
    } else {
        r.error.message;    // OK
    }
}
```

→ 공통 필드 (`status`) 로 분기. TS 가 narrowing 자동.

### 5.2 강력한 활용
```typescript
type ScanState =
    | { kind: "queued" }
    | { kind: "running"; startedAt: Date }
    | { kind: "done"; finishedAt: Date; summary: Summary }
    | { kind: "failed"; reason: string };

function describe(s: ScanState): string {
    switch (s.kind) {
        case "queued": return "Queued";
        case "running": return `Running since ${s.startedAt}`;
        case "done": return `Done with ${s.summary.total} findings`;
        case "failed": return `Failed: ${s.reason}`;
    }
}
```

→ 각 case 의 추가 필드 자동 narrowing. exhaustive switch.

### 5.3 exhaustive check
```typescript
function describe(s: ScanState): string {
    switch (s.kind) {
        case "queued": return "...";
        // ... 일부 빠뜨림
        default:
            const _exhaustive: never = s;     // ← never 할당으로 검증
            throw new Error(_exhaustive);
    }
}
```

빠뜨린 case 가 있으면 `s` 가 그 type 남아있음 → never 할당 실패 → compile error.

---

## §6. Conditional types

### 6.1 기본
```typescript
type IsString<T> = T extends string ? true : false;
type A = IsString<string>;       // true
type B = IsString<number>;        // false
```

### 6.2 distributive
union 에 conditional 적용 시 각각:
```typescript
type ToArray<T> = T extends any ? T[] : never;
type X = ToArray<string | number>;    // string[] | number[]
```

### 6.3 infer (extract)
```typescript
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
type A = ReturnType<() => string>;    // string
```

`infer R` = "R 이라는 type 변수 추출".

### 6.4 활용 — 라이브러리 타입
```typescript
type Awaited<T> = T extends Promise<infer U> ? U : T;
type X = Awaited<Promise<string>>;    // string
```

TS 표준에 포함됨 (`Awaited<T>`).

---

## §7. Mapped types

### 7.1 기본
```typescript
type Partial<T> = { [K in keyof T]?: T[K] };
type Required<T> = { [K in keyof T]-?: T[K] };
type Readonly<T> = { readonly [K in keyof T]: T[K] };
type Pick<T, K extends keyof T> = { [P in K]: T[P] };
```

→ 위 모두 TS 표준.

### 7.2 활용
```typescript
type User = { name: string; age: number; email: string };
type UserUpdate = Partial<User>;       // 모든 필드 optional
type UserKey = keyof User;              // "name" | "age" | "email"
type UserName = Pick<User, "name">;     // { name: string }
type UserNoEmail = Omit<User, "email">; // { name: string; age: number }
```

### 7.3 keyof 와 인덱스 access
```typescript
type Keys = keyof User;    // "name" | "age" | "email"
type Name = User["name"];  // string
```

### 7.4 직접 mapped type
```typescript
type Stringify<T> = { [K in keyof T]: string };
type S = Stringify<User>;   // { name: string; age: string; email: string }
```

---

## §8. Template literal types

### 8.1 기본
```typescript
type Greeting = `Hello, ${string}!`;
const g: Greeting = "Hello, World!";    // OK
```

### 8.2 union 조합
```typescript
type Color = "red" | "blue";
type Size = "sm" | "lg";
type Class = `${Color}-${Size}`;
// = "red-sm" | "red-lg" | "blue-sm" | "blue-lg"
```

### 8.3 활용 — API endpoint 타입
```typescript
type Method = "GET" | "POST" | "DELETE";
type Endpoint = "/users" | "/scans";
type Route = `${Method} ${Endpoint}`;
// = "GET /users" | "POST /users" | ... 6개
```

### 8.4 string manipulation
TS 표준:
- `Uppercase<T>`, `Lowercase<T>`, `Capitalize<T>`, `Uncapitalize<T>`.
- `${infer X}-${infer Y}` 같은 split.

---

## §9. as const (literal narrowing)

### 9.1 기본
```typescript
const x = "hello";              // type: string
const y = "hello" as const;     // type: "hello" (literal)

const arr = [1, 2, 3];          // type: number[]
const arr2 = [1, 2, 3] as const; // type: readonly [1, 2, 3]
```

### 9.2 활용
```typescript
const SEVERITIES = ["ALL", "CRITICAL", "HIGH", "MEDIUM", "LOW", "INFO"] as const;
type SeverityFilter = (typeof SEVERITIES)[number];
// = "ALL" | "CRITICAL" | ... | "INFO"
```

→ array literal → tuple → union type. VulnScope 의 ScanResult 가 사용.

---

## §10. satisfies (TS 4.9+)

### 10.1 문제 — 타입 검증 vs 추론 동시 X
```typescript
type Color = "red" | "blue";
const colors: Color[] = ["red", "blue"];
// colors 의 타입이 Color[] — narrow type 잃음.

const colors = ["red", "blue"];
// 검증 X — "yellow" 추가해도 통과.
```

### 10.2 satisfies
```typescript
const colors = ["red", "blue"] satisfies Color[];
// colors 의 타입: ("red" | "blue")[] — narrow 유지 + 검증
```

→ "검증" + "추론" 동시.

### 10.3 활용
```typescript
const config = {
    apiUrl: "https://api.example.com",
    timeout: 5000,
    retries: 3,
} satisfies Config;
// config.apiUrl 의 타입이 string 의 literal "https://api.example.com" 까지 보존.
```

---

## §11. utility types (자주 쓰는 표준)

| 타입 | 의미 |
|---|---|
| `Partial<T>` | 모든 필드 optional |
| `Required<T>` | 모든 필드 required |
| `Readonly<T>` | 모든 필드 readonly |
| `Pick<T, K>` | T 에서 K 만 |
| `Omit<T, K>` | T 에서 K 제외 |
| `Record<K, V>` | { [k: K]: V } |
| `Exclude<T, U>` | T 에서 U 제거 (union) |
| `Extract<T, U>` | T 에서 U 만 (union) |
| `NonNullable<T>` | null/undefined 제거 |
| `ReturnType<F>` | 함수 반환 type |
| `Parameters<F>` | 함수 인자 type (tuple) |
| `Awaited<T>` | Promise<X> → X |

---

## §12. VulnScope 의 type 패턴

### 12.1 OpenAPI 타입 추출
```typescript
import type { components, paths } from "@/shared/api/generated";

export type ScanView = components["schemas"]["ScanView"];
export type FindingView = components["schemas"]["FindingView"];
```

→ generated.ts 의 namespace 에서 인덱스 추출.

### 12.2 union narrowing
```typescript
// status 가 백엔드 enum 의 union
type Status = "QUEUED" | "RUNNING" | "DONE" | "FAILED" | "CANCELED";

if (s.status === "DONE") { ... }    // narrowed
```

### 12.3 as const + (typeof X)[number]
```typescript
const SEVERITIES = ["ALL", ...] as const;
type SeverityFilter = (typeof SEVERITIES)[number];
```

### 12.4 Zod schema → type
```typescript
const LoginSchema = z.object({ email: z.string(), password: z.string() });
type LoginInput = z.infer<typeof LoginSchema>;    // { email: string; password: string }
```

→ schema 가 SSOT. 수동 type 0.

---

## §13. tsconfig 의 strict

### 13.1 strict mode
```json
{ "compilerOptions": { "strict": true } }
```

켜지는 옵션:
- `strictNullChecks` — null/undefined 명시.
- `strictFunctionTypes`.
- `strictBindCallApply`.
- `strictPropertyInitialization`.
- `noImplicitAny` — implicit any 금지.
- `noImplicitThis`.
- `alwaysStrict`.

→ **TS 의 진가 = strict**. 안 켜면 의미 절반.

VulnScope 는 strict.

### 13.2 추가 권장
- `noUncheckedIndexedAccess` — `arr[0]` 의 type 이 `T | undefined` (정확).
- `exactOptionalPropertyTypes` — `{ x?: number }` 가 진짜 optional (undefined 와 missing 구분).

---

## §14. 함정 + 흔한 오해

### 14.1 any 의 위험
```typescript
const x: any = ...;
x.foo.bar.baz();    // 어떤 호출도 OK (compile time).
                    // runtime 에 깨짐.
```
→ any 만나면 의심. unknown 으로 narrowing.

### 14.2 unknown vs any
- `any`: 모든 곳에 호환. 위험.
- `unknown`: 사용 전 narrowing 강제. 안전.

### 14.3 type assertion (`as`) 의 위험
```typescript
const user = data as User;    // runtime 검증 X
```
→ 잘못된 데이터 통과. Zod 같은 runtime 검증 권장.

### 14.4 const enum 사용 X
- `const enum` 은 inline. 라이브러리 build 시 호환성 문제.
- 일반 enum 또는 union literal 권장.

---

## §15. 학습 포인트

1. **type vs interface** — type default, interface 는 declaration merging.
2. **generic = type 의 함수** — `<T>`, `extends`, default.
3. **narrowing** — typeof / in / instanceof / type predicate.
4. **discriminated union** — kind 필드로 분기.
5. **conditional + infer** — type level 에서 추출.
6. **mapped + keyof** — type 변환.
7. **template literal type** — string 조합.
8. **as const** — literal narrowing.
9. **satisfies** — 검증 + 추론 동시.
10. **utility types** — Partial/Pick/Omit/Record/ReturnType.
11. **strict mode = TS 의 진가**.
12. **any 회피, unknown + narrowing**.

### 추가 참고
- TypeScript 공식: https://www.typescriptlang.org/docs/
- Type Challenges: https://github.com/type-challenges/type-challenges
- Matt Pocock, [Total TypeScript](https://www.totaltypescript.com/).
