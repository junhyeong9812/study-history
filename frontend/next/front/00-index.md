# VulnScope Frontend 패턴/기법 학습서 — 인덱스

> 작성일: 2026-05-02
> **방향**: 라이브러리 카탈로그 X. **패턴/아이디어 중심** — 라이브러리가 어떤 패턴을 쓰는지, 왜 그 패턴인지, 직접 구현한다면 어떻게 할지.
> 대상: React 기본은 알지만 패턴 의도는 깊이 모르는 사람.

---

## 카테고리

| 번호 | 파일 | 주제 | 핵심 아이디어 |
|---|---|---|---|
| 01 | [01-react-patterns.md](01-react-patterns.md) | React 핵심 패턴 | hooks 의 closure, server vs client, Suspense, ref vs state |
| 02 | [02-styling-patterns.md](02-styling-patterns.md) | CSS / 스타일링 | design tokens, utility-first, component layer (wf-*), font composition |
| 03 | [03-form-patterns.md](03-form-patterns.md) | 폼 + 검증 | uncontrolled + ref subscription, schema-driven validation |
| 04 | [04-server-state-patterns.md](04-server-state-patterns.md) | server state | stale-while-revalidate, queryKey 위계, observer 구독 |
| 05 | [05-type-safe-api.md](05-type-safe-api.md) | type-safe API | OpenAPI codegen, path-based 타입, 백엔드 SSOT |
| 06 | [06-routing-patterns.md](06-routing-patterns.md) | 라우팅 | server/client 컴포넌트, route groups, middleware as edge auth |
| 07 | [07-streaming-patterns.md](07-streaming-patterns.md) | 실시간 스트리밍 | EventSource, useEffect cleanup, ref-based dedupe |
| 08 | [08-testing-patterns.md](08-testing-patterns.md) | 테스트 | RTL queries, MockEventSource, hook 격리 테스트 |

---

## 각 doc 의 구조

```
§0. 이 패턴이 해결하는 문제 (왜 등장했는가)
§1. 본질적 아이디어 (the core insight)
§2. 라이브러리는 어떻게 구현했나 (RHF/Zod/Query 등)
§3. VulnScope 에서의 활용 (실제 코드 + 줄별 설명)
§4. 직접 구현한다면 (라이브러리 없이)
§5. 함정 + 대안
§6. 학습 포인트
```

→ "라이브러리 사용법" 학습이 아니라 **그 라이브러리가 가진 통찰을 자기 것으로**.

---

## 학습 순서 추천

### React 기본은 아는 사람 (typical reader)
**01 React → 06 Routing → 03 Form → 06 Server state → 05 Type-safe → 02 Styling → 07 Streaming → 08 Test**

### Next.js 16 / App Router 처음
**06 Routing 먼저** → 01 React → 나머지 순서.

### 인터뷰 준비
**01 React (hooks 의 closure 문제) → 04 Server state (cache 무효화) → 03 Form (controlled vs uncontrolled) → 06 Routing**

---

## VulnScope frontend 의 결정 한눈에

- **Next.js 16 App Router** (Pages Router X) — server component default, RSC 의 server-side 데이터 페치.
- **TypeScript strict** — runtime safety + IDE 자동완성.
- **openapi-fetch** (axios X) — 백엔드 OpenAPI 에서 codegen → 100% 타입 안전.
- **TanStack Query** — server state 관리 only. client state 는 React 기본 (useState).
- **RHF + Zod** — 비제어 폼 + 런타임 검증.
- **Tailwind v4** + custom `wf-*` component layer — utility-first + 디자인 토큰.
- **Jest + RTL** — Next.js 공식 통합.

---

## VulnScope 가 안 쓴 것 (의도)

| 안 씀 | 이유 |
|---|---|
| Redux/Zustand/Jotai | server state = TanStack Query, client state = useState 충분 |
| axios | openapi-fetch 가 가볍고 typed |
| styled-components/emotion | Tailwind utility-first + runtime cost 0 |
| Material-UI/Ant Design | 직접 디자인 (wireframe vibe) |
| Formik | RHF 가 비제어 (re-render 적음) |
| Apollo Client | GraphQL 안 씀 |
| lodash | ES2020+ 표준으로 충분 |
| dayjs/date-fns | `Date` + 직접 helper 로 충분 |

---

## 다른 study 문서 참조

- [../patterns/00-index.md](../patterns/00-index.md) — 백엔드/공통 패턴.
- [../sse-libraries.md](../sse-libraries.md) — SSE 라이브러리 (frontend EventSource 포함).
