# 03. 폼 + 검증 패턴

> 이 문서가 다루는 것: controlled vs uncontrolled, schema-driven validation, RHF + Zod 의 본질적 통찰.

---

## §0. 폼이 어려운 이유

폼은 web app 에서 가장 흔하지만 의외로 어려움:
1. **state 폭증** — N 필드 = N state.
2. **validation timing** — onChange / onBlur / onSubmit / 서버 검증.
3. **error display** — 어디에 어떻게 보여줄지.
4. **performance** — 매 keystroke 마다 re-render?
5. **submit 흐름** — async / loading / success / error.

VulnScope 의 LoginForm + TargetInput 이 위 5개를 RHF + Zod 로 해결.

---

## §1. Controlled vs Uncontrolled — 본질

### 1.1 Controlled (전통적 React 방식)
```typescript
function LoginForm() {
    const [email, setEmail] = useState("");
    const [password, setPassword] = useState("");

    return (
        <>
            <input value={email} onChange={(e) => setEmail(e.target.value)} />
            <input value={password} onChange={(e) => setPassword(e.target.value)} />
        </>
    );
}
```

**동작**:
- 매 keystroke 마다 `setEmail` → component re-render.
- 100자 입력 = 100번 re-render.
- React state 가 single source of truth.

**장점**: state 일관, 검증 매 keystroke 가능.
**단점**: re-render 비용. 큰 폼에서 lag.

### 1.2 Uncontrolled (DOM 이 truth)
```typescript
function LoginForm() {
    const emailRef = useRef<HTMLInputElement>(null);
    const passwordRef = useRef<HTMLInputElement>(null);

    function onSubmit(e) {
        e.preventDefault();
        const email = emailRef.current?.value;       // submit 시점에 읽기
        const password = passwordRef.current?.value;
    }

    return (
        <form onSubmit={onSubmit}>
            <input ref={emailRef} defaultValue="" />
            <input ref={passwordRef} defaultValue="" />
        </form>
    );
}
```

**동작**:
- React state 없음 → re-render 0.
- DOM 의 value 가 source of truth.
- submit 시점에만 값 읽기.

**장점**: 빠름 (re-render 0).
**단점**: 매 keystroke 검증 어려움. React 상태 바깥.

### 1.3 트레이드오프
| | Controlled | Uncontrolled |
|---|---|---|
| 매 keystroke 검증 | 쉬움 | 어려움 (이벤트 listen 추가) |
| re-render | 매번 | 0 |
| 폼 reset / programmatic 변경 | 쉬움 (setState) | DOM 직접 |
| 큰 폼 성능 | 느릴 수 있음 | 빠름 |

→ **간단한 폼 = controlled, 큰 폼 = uncontrolled**.

### 1.4 RHF 의 절묘한 선택
RHF = uncontrolled + ref subscription + 검증 시점 hook.

**아이디어**: re-render 0 (uncontrolled), but state 관리 + 검증은 RHF 가 ref event listener 로.

---

## §2. RHF 의 ref subscription 패턴

### 2.1 register 의 정체
```typescript
const { register } = useForm();
<input {...register("email")} />
```

`register("email")` 의 반환값은:
```typescript
{
    name: "email",
    onChange: (e) => { ... },     // RHF 가 내부 store 갱신 + validation
    onBlur: (e) => { ... },        // 동일
    ref: (instance) => { ... }     // ref callback — DOM 노드 capture
}
```

→ `<input>` 의 props 로 spread. RHF 가 모든 이벤트 listen.

**중요**: input 자체는 controlled value props 없음. uncontrolled. 하지만 RHF 가 ref + event 로 모든 변화 추적.

### 2.2 핵심 통찰
**ref + event 로 외부에서 state 관리** — DOM uncontrolled 지만 RHF 가 implicit controlled. re-render 는 RHF 가 필요할 때만 트리거 (예: error 발생).

### 2.3 VulnScope 의 활용
```typescript
// LoginForm.tsx
const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
} = useForm<LoginInput>({
    resolver: zodResolver(LoginSchema),
    defaultValues: { email: "admin", password: "admin" },
});

return (
    <form onSubmit={handleSubmit(onSubmit)}>
        <Input {...register("email")} />
        {errors.email && <span>{errors.email.message}</span>}
        ...
    </form>
);
```

- `register("email")` — input 에 ref + event subscribe.
- `handleSubmit(onSubmit)` — submit 시 검증 → 통과 시 onSubmit 호출.
- `errors.email` — 실패 시 해당 필드 메시지.
- `isSubmitting` — 제출 중 loading.

**re-render 횟수**:
- email 입력 100자 = 0회 re-render.
- submit 클릭 → 검증 → error 가 있으면 1회 (errors 객체 변경).
- submit 성공 → isSubmitting true → false 두 번.

→ **typed 100자에 0 re-render**. controlled 의 100회 vs RHF 의 0회.

### 2.4 직접 구현한다면
1. ref callback 으로 DOM input 모음.
2. event listener 등록 (change/blur).
3. internal state (ref-based) 에 값 누적.
4. submit 시 모든 ref 의 value 모아서 검증 + callback.

```typescript
function useMiniForm<T>() {
    const valuesRef = useRef<Partial<T>>({});

    const register = (name: keyof T) => ({
        ref: (el: HTMLInputElement | null) => { /* register */ },
        onChange: (e: ChangeEvent<HTMLInputElement>) => {
            valuesRef.current[name] = e.target.value as any;
        },
    });

    const handleSubmit = (cb: (values: T) => void) => (e: FormEvent) => {
        e.preventDefault();
        cb(valuesRef.current as T);
    };

    return { register, handleSubmit };
}
```

원리만. RHF 는 더 정밀 (validation timing, error tracking, dirty state, watch).

---

## §3. Schema-driven validation (Zod)

### 3.1 검증의 4 levels
1. **HTML attribute** (`required`, `pattern`) — 브라우저 기본. 한계 많음.
2. **JavaScript inline** (`if (email === "") return "..."`). 흩어짐.
3. **Schema-driven** (Zod, Yup) — 한 곳에 정의. 타입 추출.
4. **Server-side** — 항상 별도. client 는 UX, server 는 보안.

### 3.2 Zod schema 의 통찰
```typescript
import { z } from "zod";

export const LoginSchema = z.object({
    email: z.string().min(1, "이메일을 입력하세요"),
    password: z.string().min(1, "비밀번호를 입력하세요"),
});

export type LoginInput = z.infer<typeof LoginSchema>;   // ← 타입 자동 추출
```

**핵심 1 — schema 가 곧 타입**:
```typescript
type LoginInput = z.infer<typeof LoginSchema>;
// = { email: string; password: string }
```

→ schema + type 한 곳. SSOT. drift 0.

**핵심 2 — runtime 검증**:
```typescript
const result = LoginSchema.safeParse({ email: "", password: "..." });
// → { success: false, error: { issues: [{ path: ["email"], message: "..." }] } }
```

→ untrusted 데이터 (form, API) 를 type-safe 하게 변환.

### 3.3 superRefine — 조건부 검증
```typescript
export const TargetCreateSchema = z.object({
    type: z.enum(["WEB", "IP", "CIDR", "API"]),
    value: z.string().min(1).max(2048),
}).superRefine((v, ctx) => {
    if ((v.type === "WEB" || v.type === "API") && !/^https?:\/\//i.test(v.value)) {
        ctx.addIssue({
            code: z.ZodIssueCode.custom,
            path: ["value"],
            message: "URL은 http:// 또는 https:// 로 시작해야 합니다",
        });
    }
});
```

**아이디어**: 단일 필드 룰 (`min/max`) 외 **여러 필드 간 조건부 룰** 표현.

여기선 type 이 WEB/API 일 때만 value 가 URL 형식인지 검증.

### 3.4 RHF + Zod = `zodResolver`
```typescript
import { zodResolver } from "@hookform/resolvers/zod";

const { register, ... } = useForm<LoginInput>({
    resolver: zodResolver(LoginSchema),
});
```

**동작**:
- RHF 가 submit 시 (또는 onChange 시 — config 으로 변경) resolver 호출.
- resolver = "값 → 검증 결과 (success/errors)".
- zodResolver 가 Zod schema 로 검증 → RHF format 으로 변환.

→ **RHF 는 검증 모름. 책임은 Zod**. 분리.

### 3.5 직접 구현한다면
```typescript
type ValidationResult<T> = { success: true; data: T } | { success: false; errors: Record<string, string> };

type Validator<T> = (input: unknown) => ValidationResult<T>;

const loginValidator: Validator<LoginInput> = (input) => {
    const errors: Record<string, string> = {};
    const i = input as LoginInput;
    if (!i.email || i.email.length === 0) errors.email = "...";
    if (!i.password || i.password.length === 0) errors.password = "...";
    return Object.keys(errors).length > 0
        ? { success: false, errors }
        : { success: true, data: i };
};
```

원리. Zod 가 schema 의 declarative + 풍부한 builder + 타입 추출까지 자동.

---

## §4. submit flow 패턴

### 4.1 standard async submit
```typescript
async function onSubmit(values: LoginInput) {
    setSubmitError(null);
    try {
        await login(values);
        const from = searchParams.get("from") ?? "/";
        router.push(from);
        router.refresh();
    } catch (e) {
        setSubmitError(e instanceof Error ? e.message : "로그인 실패");
    }
}
```

**4 phases**:
1. **pre-submit**: 이전 에러 clear (`setSubmitError(null)`).
2. **call**: async API 호출.
3. **success**: navigate + refresh.
4. **error**: 메시지 표시.

### 4.2 isSubmitting 으로 UI 잠금
```typescript
const { formState: { isSubmitting } } = useForm();

<Btn type="submit" disabled={isSubmitting}>
    {isSubmitting ? "Signing in..." : "Sign in →"}
</Btn>
```

→ 제출 중 button disabled. 더블 클릭 방지.

### 4.3 핵심 통찰
**비동기 submit = 4-phase state machine**. RHF 의 `isSubmitting` + 직접 `submitError` state.

### 4.4 직접 구현한다면
```typescript
type SubmitState = "idle" | "submitting" | "success" | "error";
const [state, setState] = useState<SubmitState>("idle");
```

state machine 으로. 또는 `XState` 라이브러리. RHF 는 자동.

---

## §5. action chain (target 등록 → scan trigger → navigate)

### 5.1 VulnScope 의 흐름
```typescript
// TargetInput.tsx
async function onSubmit(values: TargetCreateInput) {
    setSubmitError(null);
    try {
        const target = await createTarget(values);                 // ① target 등록
        const scan = await triggerScan(target.id);                 // ② scan trigger
        router.push(`/scans/${scan.id}/stream`);                    // ③ navigate
    } catch (e) {
        setSubmitError(...);
    }
}
```

### 5.2 핵심 아이디어
사용자 의도 = "scan 시작". target 등록은 부수효과.

**naive (분리)**:
```typescript
// 두 button 분리
<button onClick={createTarget}>1. Register Target</button>
<button onClick={triggerScan}>2. Start Scan</button>
```
→ UX 나쁨. 사용자가 두 번 클릭.

**action chain**:
- 한 번 클릭 → 등록 + 시작 + navigate 모두.
- partial failure 시 단일 try-catch.

### 5.3 함정
- 중간 실패 시 어떻게 처리? (target 등록 OK, scan 실패).
  - VulnScope 는 unified error message. target 은 등록된 채로 남음. 다음 시도에 dedupe.
  - 엄격하면 target 삭제 (compensation transaction). 복잡.

---

## §6. dev fixture default

### 6.1 LoginForm 의 default
```typescript
useForm<LoginInput>({
    defaultValues: { email: "admin", password: "admin" },
});
```

→ dev 부팅 후 form 자동 채움. 사용자가 그냥 Sign in 클릭.

### 6.2 핵심 통찰
**dev iteration speed > 보안**. dev 환경은 fixture default 가 권장. production 빌드에 들어가지 않게 주의 (NODE_ENV 분기 또는 환경별 빌드).

### 6.3 함정
production 에 dev fixture 노출 시 사고. CI 에서 NODE_ENV=production 빌드 검증.

---

## §7. 패턴 매트릭스

| 문제 | 패턴 |
|---|---|
| re-render 비용 | uncontrolled + RHF |
| state vs DOM truth | RHF 의 ref subscription |
| 검증 한 곳 | schema-driven (Zod) |
| schema → type | z.infer |
| 조건부 검증 | superRefine |
| async submit phases | isSubmitting + try-catch |
| 사용자 의도 단일화 | action chain |
| dev iteration | defaultValues fixture |

---

## §8. 학습 포인트

1. **controlled = state, uncontrolled = ref + DOM**.
2. **RHF 의 통찰** — uncontrolled 인데 ref + event subscription 으로 implicit controlled.
3. **register spread** = `{...register("name")}`. ref + onChange + onBlur 자동.
4. **Zod schema = type 자동 추출** — SSOT.
5. **safeParse vs parse** — throw 여부.
6. **superRefine** — 여러 필드 조건부.
7. **zodResolver** — RHF + Zod 다리.
8. **isSubmitting + submitError** — submit state machine.
9. **action chain** — 사용자 의도 = 단일 click.
10. **defaultValues fixture** — dev UX (production 분리 주의).

### 추가 참고
- React Hook Form: https://react-hook-form.com/
- Zod: https://zod.dev/
- "Why I love RHF" (uncontrolled 우월성): https://react-hook-form.com/why-rhf
