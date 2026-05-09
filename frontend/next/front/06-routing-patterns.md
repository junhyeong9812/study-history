# 06. 라우팅 패턴

> 이 문서가 다루는 것: Next.js App Router 의 server vs client component 분리, route groups, middleware as edge cross-cutting — 라우팅 패턴의 본질.

---

## §0. Next.js App Router 가 무엇이고 왜 등장했나

### 0.1 Pages Router (전통, Next.js 12 까지)
```
pages/
  index.tsx           → /
  scans/
    [scanId].tsx      → /scans/:scanId
```

- 파일 = 라우트.
- 모든 페이지가 client component (CSR/SSR/SSG 의 trade-off).
- `getServerSideProps` 로 server-side fetch (제한적).

### 0.2 App Router (Next.js 13+)
```
app/
  page.tsx            → /
  (auth)/
    login/page.tsx    → /login
  (app)/
    scans/
      [scanId]/
        page.tsx      → /scans/:scanId
        stream/
          page.tsx    → /scans/:scanId/stream
```

- **server component default** + 명시적 `"use client"`.
- 파일 컨벤션 풍부 (`page`, `layout`, `loading`, `error`, `not-found`).
- nested layouts.
- streaming + RSC (React Server Components).

App Router 가 **server-first** 전략. VulnScope 채택.

---

## §1. Server Component vs Client Component (가장 큰 변화)

§01-react-patterns.md 의 §2 도 참조. 여기는 routing 관점.

### 1.1 server component
```typescript
// app/(app)/scans/[scanId]/page.tsx
export default async function ScanResultPage({
    params,
}: {
    params: Promise<{ scanId: string }>;
}) {
    const { scanId } = await params;
    return <ScanResult scanId={scanId} />;
}
```

- async function 가능.
- DB/API 직접 호출 가능 (서버 코드).
- hooks 사용 X.
- HTML 만 응답 (JS 번들 X).

### 1.2 client component
```typescript
// features/scan/components/ScanResult.tsx
"use client";

export function ScanResult({ scanId }: { scanId: string }) {
    const { scan, ... } = useScanResult(scanId);    // hook
    // ...
}
```

- hooks 사용 가능.
- onClick, onChange 등 interactivity.
- JS 번들 포함 → hydrate.

### 1.3 boundary 의 의미
**server → client** 는 OK (server 가 client component 렌더 가능).
**client → server** 는 **불가** (보통).

```typescript
// page.tsx (server)
import { ScanResult } from "..."; // client component

export default async function Page({ params }) {
    const { scanId } = await params;
    // 여기서 server-side fetch 가능 (Promise 만들고 client 에 props 로)
    return <ScanResult scanId={scanId} />;
}
```

VulnScope 의 모든 page = thin server component (parameter 추출 + client component 호출).

### 1.4 핵심 통찰
**server 가 routing/layout/initial fetch, client 가 interactivity** 분담.

→ 초기 로드 빠름 (server HTML) + interactive 부분만 hydrate (client JS).

### 1.5 직접 구현한다면
React Server Components 의 메커니즘:
1. Server: server component → React Element tree (with client placeholders).
2. Network: tree 직렬화 (RSC payload — 특수 format).
3. Client: payload deserialize → DOM mount + client component hydrate.

매우 복잡. Next.js 14+ 의 핵심.

---

## §2. Route Groups (괄호 디렉토리)

### 2.1 사용
```
app/
  (auth)/                   ← 괄호 = URL 영향 X
    layout.tsx              ← (auth) 안 페이지 공통 layout
    login/page.tsx          → /login
  (app)/
    layout.tsx              ← (app) 안 페이지 공통 layout
    page.tsx                → /
    scans/page.tsx          → /scans
    settings/page.tsx       → /settings
```

### 2.2 핵심 통찰
**`(name)` 디렉토리 = URL 안 영향, layout 만 분리**.

- `(auth)` = login 같은 auth 페이지. Topbar 만, sidebar 없음.
- `(app)` = 메인 앱. Topbar + Sidebar.

→ URL 은 `/login`, `/`, `/scans` 그대로. **layout 만 다름**.

### 2.3 VulnScope 의 layout 분리
```typescript
// app/(auth)/layout.tsx
export default function AuthLayout({ children }) {
    return (
        <div className="min-h-screen flex flex-col">
            <Topbar />
            <main className="flex-1 flex items-center justify-center p-6">
                {children}
            </main>
        </div>
    );
}
```

```typescript
// app/(app)/layout.tsx
const navItems: SidebarItem[] = [...];

export default function AppLayout({ children }) {
    return (
        <div className="min-h-screen flex flex-col">
            <Topbar actions={<LogoutButton />} />
            <div className="flex flex-1 min-h-0">
                <Sidebar items={navItems} />
                <main className="flex-1 p-6 overflow-auto">{children}</main>
            </div>
        </div>
    );
}
```

→ `/login` (auth) 와 `/scans` (app) 가 **다른 layout** 받지만 URL 은 깔끔.

### 2.4 직접 구현한다면
일반 routing 라이브러리 (React Router 등) 도 nested route + layout 가능. App Router 의 file-system 컨벤션이 더 명시적.

---

## §3. Middleware (edge cross-cutting)

### 3.1 사용
```typescript
// middleware.ts (root)
import { NextResponse, type NextRequest } from "next/server";

const PUBLIC_PATHS = ["/login", "/api/auth"];
const SESSION_COOKIE = "vulnscope_session";

export function middleware(req: NextRequest) {
    const { pathname } = req.nextUrl;

    if (PUBLIC_PATHS.some((p) => pathname.startsWith(p))) {
        return NextResponse.next();
    }

    const session = req.cookies.get(SESSION_COOKIE);
    if (!session) {
        const url = new URL("/login", req.url);
        url.searchParams.set("from", pathname);
        return NextResponse.redirect(url);
    }

    return NextResponse.next();
}

export const config = {
    matcher: ["/((?!_next/static|_next/image|favicon.ico).*)"],
};
```

### 3.2 핵심 통찰
**모든 요청 가로챔, 페이지 도달 전. SSR 전에 동작**.

- edge runtime (Vercel 의 edge worker) 또는 Node.js 에서.
- 빠름 (가벼운 V8 isolate).
- 인증, redirect, A/B test, rate limit 등 cross-cutting.

### 3.3 vs Spring AuthFilter 비교
| | Next.js middleware | Spring AuthFilter |
|---|---|---|
| 위치 | edge / Node, request 처리 전 | servlet container, request 처리 전 |
| 책임 | redirect, cookie 검사 (가벼움) | 진짜 인증 (HMAC verify) |
| 실패 시 | redirect to /login | 401 응답 |

VulnScope 의 분담:
- middleware: cookie 존재 여부만 (가벼움). 없으면 redirect.
- 백엔드 AuthFilter: 진짜 검증 (HMAC). 위조 cookie 면 401.

→ **defense in depth**. middleware 는 UX (로그인 페이지로), 백엔드 는 보안.

### 3.4 matcher 의 의미
```typescript
matcher: ["/((?!_next/static|_next/image|favicon.ico).*)"]
```
→ `_next/static`, `_next/image`, `favicon.ico` 제외 모든 path. 정적 자원에 middleware 안 거치게 (성능).

### 3.5 direct 구현
- Express middleware 같은 패턴. Next.js 가 file-based.
- edge runtime 은 V8 isolate (빠른 cold start).

---

## §4. Async params (Next.js 15+)

### 4.1 변경
```typescript
// Next.js 14 까지
export default function Page({ params }: { params: { scanId: string } }) {
    const scanId = params.scanId;
}

// Next.js 15+
export default async function Page({ params }: { params: Promise<{ scanId: string }> }) {
    const { scanId } = await params;       // ← Promise + await
}
```

### 4.2 왜 변경?
- Next.js 의 streaming + suspense 통합.
- params 가 비동기로 전달될 수 있음 (인증 후 동적 결정 등).

### 4.3 핵심 통찰
**Next.js 15+ 는 모든 RSC 가 async function 가능**. params/searchParams/cookies 등 dynamic API 가 Promise.

### 4.4 함정
- 옛 코드 마이그레이션 시 missing await → 빌드 시 warn → silent error.
- IDE 가 Promise 타입 보여주면 await 추가.

---

## §5. Dynamic routes 와 catch-all

### 5.1 dynamic route
```
app/scans/[scanId]/page.tsx    → /scans/:scanId
app/scans/[scanId]/findings/[findingId]/page.tsx
                                → /scans/:scanId/findings/:findingId
```

`[name]` = path parameter.

### 5.2 catch-all
```
app/api/[...path]/route.ts     → /api/anything/here
```

`[...name]` = 모든 sub-path 매치. VulnScope BFF proxy 가 사용.

### 5.3 optional catch-all
```
app/[[...path]]/page.tsx       → /, /a, /a/b 모두
```

`[[...name]]` = optional. VulnScope 미사용.

### 5.4 핵심 통찰
**file-based routing + 변수**. dynamic 의 강력함.

---

## §6. BFF route handler (Backend-for-Frontend proxy)

### 6.1 사용
```typescript
// app/api/[...path]/route.ts
export const dynamic = "force-dynamic";

async function proxy(req: NextRequest, ctx: { params: Promise<{ path: string[] }> }) {
    const { path } = await ctx.params;
    const target = `${BACKEND_API_BASE_URL}/${path.join("/")}${req.nextUrl.search}`;

    const init: RequestInit = {
        method: req.method,
        headers: forwardHeaders(req),
        body: methodAllowsBody(req.method) ? req.body : undefined,
        duplex: methodAllowsBody(req.method) ? "half" : undefined,
    } as RequestInit;

    const upstream = await fetch(target, init);
    // ... response 헤더 화이트리스트 + body stream pass
}

export const GET = proxy;
export const POST = proxy;
// ... 5 methods
```

### 6.2 BFF 의 역할
- URL prefix 변환 (`/api/scans` → `<backend>/scans`).
- 헤더 화이트리스트 (host, referrer 차단).
- Set-Cookie passthrough (httpOnly 유지).
- streaming body (SSE 통과).

### 6.3 핵심 통찰
**frontend 와 backend 사이 명시적 다리**. 보안 + URL 단순화 + CORS 회피.

상세는 백엔드 patterns 와 학습 doc `09b-bff-routes.md` 참조.

### 6.4 직접 구현
일반 reverse proxy (nginx, Caddy) 와 같은 의미. Next.js 의 route handler 가 inline.

---

## §7. Loading / Error / Not Found 컨벤션

### 7.1 파일 컨벤션
```
app/scans/
  page.tsx          → 일반 페이지
  loading.tsx       → 로딩 UI (Suspense fallback)
  error.tsx         → 에러 boundary
  not-found.tsx     → 404
```

### 7.2 자동 boundary
- `loading.tsx` = page 의 Suspense fallback 자동.
- `error.tsx` = error boundary 자동.

VulnScope 는 컴포넌트 레벨 처리 (현재 컨벤션 파일 미사용).

---

## §8. 패턴 매트릭스

| 문제 | 패턴 |
|---|---|
| server-rendered HTML + client interactivity | server vs client component |
| layout 분리 + URL 영향 X | route group `(name)` |
| 모든 요청 가로챔 | middleware |
| URL 변수 | `[name]` dynamic route |
| 모든 sub-path | `[...name]` catch-all |
| frontend ↔ backend 다리 | BFF route handler |
| async dynamic API | Next.js 15+ Promise + await |

---

## §9. 학습 포인트

1. **App Router = server-first** — server component default.
2. **server/client boundary** — `"use client"` 가 client subtree 시작.
3. **route group `(name)`** — URL 영향 X, layout 분리.
4. **middleware = edge cross-cutting** — auth redirect 같은.
5. **defense in depth** — middleware (UX) + 백엔드 AuthFilter (보안).
6. **matcher** — middleware 적용 path 제어 (정적 자원 제외).
7. **async params (Next 15+)** — Promise + await.
8. **catch-all `[...path]`** — BFF proxy 의 핵심.
9. **file 컨벤션** — page/layout/loading/error/not-found.
10. **server 가 thin (param 추출), client 가 비즈니스** — VulnScope 패턴.

### 추가 참고
- Next.js Routing: https://nextjs.org/docs/app/building-your-application/routing
- Server Components: https://react.dev/reference/rsc/server-components
- Middleware: https://nextjs.org/docs/app/building-your-application/routing/middleware
