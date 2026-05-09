# 08. 프론트엔드 테스트 패턴

> 이 문서가 다루는 것: Testing Library 의 user-centric 철학, MockEventSource (globalThis 교체), hook 격리 테스트, jest 의 jsdom 환경.

---

## §0. 프론트엔드 테스트의 어려움

| 어려움 | 이유 |
|---|---|
| DOM 시뮬 | 브라우저 API (window, document) 가 Node 에 없음 |
| async event | setTimeout, Promise, useEffect 가 rendering 후 |
| 외부 시스템 | fetch, EventSource, localStorage 모두 mock |
| component vs hook | hook 만 테스트도 가능해야 |

VulnScope 의 11개 테스트 패턴 (백엔드 + frontend 통합) — 이 doc 는 frontend 만.

---

## §1. Testing Library 의 철학 — user-centric

### 1.1 Enzyme (옛 방식) 의 문제
```typescript
// Enzyme — 구현 디테일 검증
const wrapper = shallow(<LoginForm />);
expect(wrapper.find('Input').prop('value')).toBe('');
expect(wrapper.state('email')).toBe('');     // ❌ internal state
```

→ 컴포넌트 구현 변경 시 (state → ref, etc) 테스트 깨짐. **brittle**.

### 1.2 Testing Library 의 통찰
**"사용자가 보는 것" 만 테스트**.
- 사용자: "Email" 라벨을 보고 input 에 입력.
- 테스트: `screen.getByLabelText("Email")` → input 찾기.
- DOM 의 attribute / text content 만 사용.
- internal state, ref, hook 직접 access X.

```typescript
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";

it("submits with email and password", async () => {
    render(<LoginForm />);
    const user = userEvent.setup();

    await user.type(screen.getByLabelText("Email"), "admin");
    await user.type(screen.getByLabelText("Password"), "admin");
    await user.click(screen.getByRole("button", { name: /sign in/i }));

    // 결과 검증 (URL change, error message 등)
});
```

### 1.3 Query 우선순위
RTL 권장 순서 (실제 사용자 접근성과 비슷한 순서):
1. **getByRole** — `<button>`, `<input>` 등 ARIA role.
2. **getByLabelText** — `<label>` 으로 연결된 input.
3. **getByPlaceholderText** — placeholder 만 있는 경우.
4. **getByText** — 일반 텍스트.
5. **getByTestId** — 위 모두 안 되면 (최후).

→ data-testid 는 마지막. 사용자가 안 보는 hint 라.

### 1.4 핵심 통찰
**"리팩토링 해도 테스트 안 깨짐" = 사용자 동작이 안 바뀌면**. internal 변경 무관.

### 1.5 직접 구현
Testing Library 가 매우 가벼움 — 직접 구현 필요 거의 없음. 단, query 함수 의 의도 이해는 중요.

---

## §2. Jest + jsdom 환경

### 2.1 jsdom 이 무엇인가
**Node.js 에서 DOM 시뮬**. 진짜 브라우저 X, but `document`, `window`, `HTMLElement` 등 API 제공.

### 2.2 Jest 설정
```javascript
// jest.config.mjs
import nextJest from "next/jest.js";
const createJestConfig = nextJest({ dir: "./" });

const customJestConfig = {
    setupFilesAfterEach: ["<rootDir>/jest.setup.ts"],
    testEnvironment: "jsdom",                                // ← 핵심
    moduleNameMapper: {
        "^@/(.*)$": "<rootDir>/src/$1",
    },
};

export default createJestConfig(customJestConfig);
```

`testEnvironment: "jsdom"` → 매 test file 에 jsdom global 주입.

### 2.3 setup file
```typescript
// jest.setup.ts
import "@testing-library/jest-dom";    // toBeInTheDocument 등 매처 추가
```

### 2.4 next/jest 의 마법
Next.js 의 build 설정 (TypeScript, JSX, CSS module 등) 을 자동 적용. 직접 babel/swc 설정 X.

### 2.5 jsdom 의 한계
- 진짜 layout/paint X. `getBoundingClientRect()` 가 0 반환.
- 일부 브라우저 API 없음 (예: `IntersectionObserver`). polyfill 또는 mock 필요.
- WebGL, canvas 등 X.

→ unit / hook 테스트엔 충분. visual / e2e 는 Playwright 등.

---

## §3. MockEventSource — globalThis 교체 패턴

### 3.1 문제
- jsdom 에 `EventSource` 자체가 없음.
- 진짜 SSE 서버 띄워서 테스트 = 느림 + 복잡.
- hook 의 동작 (dedupe, lifecycle) 만 검증하고 싶음.

### 3.2 해결: globalThis 교체
```typescript
class MockEventSource {
    static instances: MockEventSource[] = [];
    url: string;
    onopen: (() => void) | null = null;
    onerror: (() => void) | null = null;
    closed = false;
    private listeners = new Map<string, Listener[]>();

    constructor(url: string, init?: { withCredentials?: boolean }) {
        this.url = url;
        MockEventSource.instances.push(this);
    }

    addEventListener(type: string, listener: Listener) {
        const arr = this.listeners.get(type) ?? [];
        arr.push(listener);
        this.listeners.set(type, arr);
    }

    emit(type: string, data: string, lastEventId = "") {
        const event = new MessageEvent(type, { data, lastEventId });
        this.listeners.get(type)?.forEach(l => l(event));
    }

    close() { this.closed = true; }
}

beforeEach(() => {
    MockEventSource.instances = [];
    (globalThis as any).EventSource = MockEventSource;        // ← 핵심
});
```

### 3.3 핵심 통찰
**globalThis 의 API 를 mock 으로 교체**. test 동안만 적용.

→ hook 코드는 변경 0. 진짜 EventSource 받는 줄 알고 동작.

### 3.4 사용 예
```typescript
it("dedupes events with seq <= lastSeq on reconnect", async () => {
    const { result } = renderHook(() => useScanStream("s1"));
    const es = MockEventSource.instances[0];

    act(() => {
        es.emit("finding", JSON.stringify({ title: "A" }), "1");
        es.emit("finding", JSON.stringify({ title: "B" }), "2");
    });
    await waitFor(() => expect(result.current.events).toHaveLength(2));

    act(() => {
        es.emit("finding", JSON.stringify({ title: "B-dup" }), "2");    // 중복 seq
        es.emit("finding", JSON.stringify({ title: "C" }), "3");
    });
    await waitFor(() => expect(result.current.events).toHaveLength(3));    // 4 X (dedupe)

    expect(result.current.events.map(e => (e.payload as any).title))
        .toEqual(["A", "B", "C"]);    // B-dup 없음
});
```

### 3.5 직접 구현
- mock class 가 listener 등록받고 emit 으로 시뮬.
- globalThis 교체.
- afterEach 에서 원래 값 복원.

```typescript
let originalEventSource: typeof EventSource;

beforeEach(() => {
    originalEventSource = globalThis.EventSource;
    (globalThis as any).EventSource = MockEventSource;
});

afterEach(() => {
    (globalThis as any).EventSource = originalEventSource;
});
```

---

## §4. renderHook + act + waitFor — hook 격리 테스트

### 4.1 renderHook
```typescript
import { renderHook } from "@testing-library/react";

const { result } = renderHook(() => useScanStream("scan-123"));
expect(result.current.connected).toBe(false);
```

`renderHook` = 가짜 컴포넌트 안에서 hook 실행. `result.current` 가 hook 의 반환값.

### 4.2 act
```typescript
act(() => {
    MockEventSource.instances[0].onopen?.();    // state 변경 트리거
});
```

`act` 가 React 의 update batch 보장. update 후 DOM/state 가 일관 상태.

→ React 의 lifecycle 매끄럽게 진행.

### 4.3 waitFor
```typescript
await waitFor(() => expect(result.current.connected).toBe(true));
```

비동기 state 변경 (setState 가 async batch) 기다림. polling + timeout.

### 4.4 핵심 통찰
- **renderHook**: hook 격리.
- **act**: state update batch.
- **waitFor**: async 검증.

→ 셋이 React Testing Library 의 핵심 비동기 검증 도구.

---

## §5. Zod schema 단위 테스트

### 5.1 코드
```typescript
import { LoginSchema } from "./schemas";

describe("LoginSchema", () => {
    it("accepts valid email and password", () => {
        const result = LoginSchema.safeParse({ email: "admin", password: "admin" });
        expect(result.success).toBe(true);
    });

    it("rejects empty email", () => {
        const result = LoginSchema.safeParse({ email: "", password: "admin" });
        expect(result.success).toBe(false);
        if (!result.success) {
            expect(result.error.issues[0].path).toEqual(["email"]);
            expect(result.error.issues[0].message).toBe("이메일을 입력하세요");
        }
    });
});
```

### 5.2 핵심 통찰
- **schema = 검증의 SSOT** → 단위 테스트로 전체 룰 cover.
- `.safeParse` 가 throw 안 함 → 결과 객체 검증 가능.

### 5.3 효과
- 폼 컴포넌트 테스트 (RTL) 안에서 검증 룰 다 cover 안 해도 됨.
- schema 만 test → 룰 변경 시 빠른 피드백.

---

## §6. component snapshot — VulnScope 미사용

### 6.1 snapshot test 란
```typescript
it("renders correctly", () => {
    const { container } = render(<MyComponent />);
    expect(container.firstChild).toMatchSnapshot();
});
```

→ 첫 실행 시 snapshot 파일 생성. 이후 변경 시 diff.

### 6.2 VulnScope 미사용 이유
- 디자인 자주 변경 → snapshot 매번 update → 의미 없음.
- behavior test (RTL) 가 더 의미.
- snapshot 은 의도 안 한 변경 발견 용도. 디자인 패스가 끝나면 도입 가능.

---

## §7. 테스트 분류

### 7.1 VulnScope 의 frontend test
| 종류 | 파일 | 검증 |
|---|---|---|
| Schema 단위 | `auth/schemas.test.ts`, `target/schemas.test.ts` | Zod 검증 룰 |
| Component 단위 | `SeverityBadge.test.tsx`, `FindingsTable.test.tsx`, `Btn.test.tsx`, `Progress.test.tsx`, `SeverityDot.test.tsx`, `Stat.test.tsx` | 렌더링 + props |
| Hook 단위 | `useScanStream.test.tsx` | hook 동작 (dedupe, lifecycle) |

### 7.2 비교
| 종류 | 의도 | 비용 |
|---|---|---|
| schema 단위 | 검증 룰 모든 case | 빠름 |
| component 단위 | 렌더링 + props | 빠름 (jsdom) |
| hook 단위 | hook 격리 | 빠름 |
| integration | 여러 컴포넌트 통합 | 중간 |
| e2e (Playwright) | 진짜 브라우저 | 느림 |

VulnScope = 단위 위주. e2e 는 v0.2 후보.

---

## §8. 패턴 매트릭스

| 문제 | 패턴 |
|---|---|
| DOM 시뮬 | jsdom + Testing Library |
| user-centric query | getByRole / getByLabelText |
| 외부 API mock | globalThis 교체 |
| hook 격리 | renderHook |
| state batch | act |
| async 검증 | waitFor |
| Zod 룰 검증 | safeParse + issues |
| component 의도 검증 | RTL behavior test |

---

## §9. 학습 포인트

1. **user-centric** — 사용자가 보는 것만 테스트. internal X.
2. **Query 우선순위** — Role > Label > Text > TestId.
3. **jsdom 환경** — Node 에서 DOM 시뮬. visual X.
4. **globalThis 교체** — EventSource 같은 브라우저 API mock.
5. **renderHook** — hook 만 격리 테스트.
6. **act + waitFor** — state batch + async 검증.
7. **safeParse vs parse** — throw 여부.
8. **schema 단위 테스트 = 모든 룰 cover** — 폼 component 부담 줄임.
9. **snapshot 은 디자인 안정화 후 도입** — 자주 변경 시 noise.
10. **단위 + 비동기 검증 = 대부분의 frontend 가치 보장**.

### 추가 참고
- Testing Library: https://testing-library.com/docs/react-testing-library/intro/
- Jest: https://jestjs.io/
- next/jest: https://nextjs.org/docs/app/building-your-application/testing/jest
- Kent C. Dodds, [Common Testing Mistakes](https://kentcdodds.com/blog/common-mistakes-with-react-testing-library)
