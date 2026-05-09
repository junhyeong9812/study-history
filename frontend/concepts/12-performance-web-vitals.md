# 12. Performance + Web Vitals

> 이 문서가 다루는 것: Core Web Vitals (LCP, INP, CLS), code splitting, lazy loading, React 최적화 (memo/useMemo/useCallback) — 언제 정말 효과 있는지.

---

## §0. 성능 측정의 의미

### 0.1 user-perceived performance
사용자가 느끼는 성능 = 객관적 metric:
- 로드 빠름?
- 인터랙션 즉각?
- 화면 안정적?

→ Google 의 **Core Web Vitals** 가 표준.

### 0.2 metric 우선
"감"이 아닌 측정. PerformanceObserver / Lighthouse / web-vitals 라이브러리.

---

## §1. Core Web Vitals (3개)

### 1.1 LCP (Largest Contentful Paint)
- **무엇**: viewport 안 가장 큰 element 가 그려진 시점.
- **목표**: < 2.5s (good), < 4s (needs improvement).
- **개선**: 이미지 최적화, 서버 응답 빠르게, render-blocking 제거.

### 1.2 INP (Interaction to Next Paint, FID 후속)
- **무엇**: 사용자 input → 다음 paint 까지.
- **목표**: < 200ms (good), < 500ms (needs improvement).
- **개선**: long task 줄이기, code splitting, React Concurrent.

### 1.3 CLS (Cumulative Layout Shift)
- **무엇**: 페이지 라이프사이클 동안 layout shift 누적.
- **목표**: < 0.1 (good), < 0.25 (needs improvement).
- **개선**: image / video 에 width/height 명시, font fallback 매칭.

### 1.4 측정 (web-vitals 라이브러리)
```javascript
import { onLCP, onINP, onCLS } from "web-vitals";

onLCP(metric => console.log("LCP", metric.value));
onINP(metric => console.log("INP", metric.value));
onCLS(metric => console.log("CLS", metric.value));
```

VulnScope 는 측정 미설정 — v0.2 후보.

---

## §2. 추가 Web Vitals

### 2.1 FCP (First Contentful Paint)
- 첫 콘텐츠 (텍스트, 이미지) 가 그려진 시점.
- LCP 의 lighter 버전.
- 목표 < 1.8s.

### 2.2 TTFB (Time to First Byte)
- 서버 응답 첫 byte 도착 시점.
- 백엔드 + 네트워크.

### 2.3 TTI (Time to Interactive)
- 페이지가 실제 인터랙션 가능 시점.
- main thread 가 유휴.

---

## §3. Image 최적화

### 3.1 width/height 명시 (CLS 방지)
```html
<img src="..." width="400" height="300" />
```

→ 이미지 로드 전에도 공간 확보. layout shift 0.

### 3.2 lazy loading
```html
<img src="..." loading="lazy" />
```

→ viewport 가까워야 로드. 초기 로드 빠름.

### 3.3 modern format (WebP, AVIF)
- WebP — 25-35% 작음 vs JPEG.
- AVIF — 더 작음. 브라우저 지원 확장 중.

### 3.4 srcset (responsive)
```html
<img srcset="small.jpg 480w, large.jpg 1200w" sizes="(max-width: 600px) 480px, 1200px" />
```

→ 화면 크기에 맞는 이미지.

### 3.5 Next.js Image
```typescript
import Image from "next/image";
<Image src="/photo.jpg" width={400} height={300} alt="..." />
```

→ 자동 최적화 (format, srcset, lazy, blur placeholder).

VulnScope 는 이미지 거의 없음. SVG / icon 위주.

---

## §4. Code splitting

### 4.1 의도
모든 JS 한 번에 로드 X. 필요한 페이지만.

### 4.2 React lazy + Suspense
```typescript
const Heavy = lazy(() => import("./Heavy"));

function App() {
    return (
        <Suspense fallback={<Spinner />}>
            <Heavy />
        </Suspense>
    );
}
```

→ Heavy 가 별도 chunk. 처음엔 안 로드.

### 4.3 Next.js dynamic import
```typescript
import dynamic from "next/dynamic";
const Heavy = dynamic(() => import("./Heavy"), { loading: () => <Spinner /> });
```

→ Next.js 가 chunk 분리 + ssr 옵션.

### 4.4 route 기반 자동
Next.js App Router 는 페이지마다 자동 chunk.

→ /scans 페이지는 /findings 페이지 코드 안 받음.

VulnScope 는 자연 활용 (Next.js 기본).

---

## §5. React 최적화

### 5.1 React.memo
```typescript
const MemoChild = React.memo(Child);
```

→ props shallow compare. 같으면 re-render skip.

**언제**:
- 자식이 큰 list 또는 비싼 render.
- 부모 re-render 가 자주.

**언제 안**:
- 자식이 가벼움.
- props 가 거의 매번 변경.

### 5.2 useMemo
```typescript
const result = useMemo(() => expensiveComputation(data), [data]);
```

**언제**:
- expensive computation.
- 자식 React.memo 의 props reference.

### 5.3 useCallback
```typescript
const handleClick = useCallback(() => { ... }, [deps]);
```

**언제**:
- 자식 React.memo 의 onClick props.

### 5.4 함정 — 남용
**잘못된 패턴**: 모든 컴포넌트 React.memo + 모든 함수 useCallback + 모든 값 useMemo.

→ memoization 자체 비용 (compare deps + 결과 보관) > render skip 효과.

**규칙**: 측정 후 적용. naive 가 충분.

### 5.5 React Profiler
Chrome DevTools React tab → Profiler → 가장 느린 컴포넌트 찾기.

---

## §6. Bundle size

### 6.1 측정
Next.js 기본:
```bash
npm run build
# 각 route 의 JS 크기 표시
```

### 6.2 분석 도구
- `@next/bundle-analyzer` — visual.
- `source-map-explorer` — generic.

### 6.3 줄이는 방법
- code splitting (위 §4).
- tree shaking (ESM + sideEffects: false).
- 무거운 라이브러리 회피 (moment → date-fns / dayjs / native).
- dynamic import.

### 6.4 측정 우선
- "뭐가 큰지" 모르고 줄이려 X.
- bundle analyzer 먼저.

---

## §7. Server-side rendering vs Client

### 7.1 SSR (Server-side Rendering)
- server 가 HTML 생성 → client 에 전송.
- 빠른 LCP (HTML 즉시).
- 검색엔진 친화.
- Next.js 의 server component default.

### 7.2 CSR (Client-side Rendering)
- HTML 거의 비어있음.
- JS 다운 → 실행 → 렌더.
- 느린 LCP.
- 검색엔진 어려움 (JS 실행 필요).

### 7.3 SSG (Static Site Generation)
- 빌드 시 HTML 생성.
- CDN 에서 즉시 응답.
- 가장 빠름.
- 동적 콘텐츠 어려움.

### 7.4 Next.js 의 mix
- 기본 SSR + RSC.
- `generateStaticParams` 로 SSG 도.
- 페이지마다 선택.

VulnScope = SSR (in-memory data 라 매 요청).

---

## §8. Long Task 분리

### 8.1 50ms+ task = INP 악화
- main thread 50ms 점유 → input 무반응.
- 매 long task 가 INP 점수 깎음.

### 8.2 분리 전략
- **time slicing**: 큰 작업 → setTimeout 0 으로 chunk.
- **Web Worker**: 무거운 계산을 다른 스레드에.
- **React useTransition**: non-urgent update 표시.

### 8.3 Web Worker 예
```javascript
const worker = new Worker("./heavy-worker.js");
worker.postMessage({ data: ... });
worker.onmessage = (e) => console.log(e.data);
```

→ DOM 접근 X. 순수 계산만.

VulnScope 미사용 (작은 계산).

---

## §9. Caching strategy

### 9.1 service worker (PWA)
- offline cache.
- network-first / cache-first / stale-while-revalidate.

### 9.2 HTTP cache
- 10-browser-storage-network.md §8 참조.

### 9.3 React Query (server state cache)
- 04-server-state-patterns.md 참조.

### 9.4 CDN
- 정적 자원 (image, JS) edge cache.
- TTL 설정.

VulnScope = Next.js + React Query 만. CDN/SW 미사용.

---

## §10. CSS 의 성능

09-browser-rendering.md 참조.

핵심:
- transform/opacity = composite only (60fps).
- width/height/padding = layout (느림).
- containment / content-visibility 활용.

---

## §11. 학습 포인트

1. **Core Web Vitals** = LCP / INP / CLS.
2. **LCP < 2.5s, INP < 200ms, CLS < 0.1**.
3. **이미지 width/height 명시** = CLS 방지.
4. **lazy loading** = `loading="lazy"`.
5. **code splitting** = lazy + Suspense + Next dynamic.
6. **React.memo** = shallow compare. 큰 자식만.
7. **useMemo/useCallback** = React.memo + props reference 안정.
8. **memoization 남용 X** — 측정 후.
9. **Long Task 50ms+** = INP 악화.
10. **Web Worker** = 무거운 계산 분리.
11. **SSR vs CSR vs SSG** — Next.js 의 mix.
12. **Bundle analyzer 먼저** — "뭐가 큰지" 알고 줄이기.

### 추가 참고
- web.dev Core Web Vitals: https://web.dev/articles/vitals
- React Profiler: https://react.dev/reference/react/Profiler
- Next.js Performance: https://nextjs.org/docs/app/building-your-application/optimizing
