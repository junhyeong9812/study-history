# 11. Web Observers

> 이 문서가 다루는 것: IntersectionObserver, MutationObserver, ResizeObserver, PerformanceObserver, MediaQueryList. DOM 변화를 효율적으로 감지하는 modern API.

---

## §0. 왜 Observer?

### 0.1 naive — scroll/resize listener
```javascript
window.addEventListener("scroll", () => {
    const rect = element.getBoundingClientRect();
    if (rect.top < window.innerHeight) {
        // visible
    }
});
```

**문제**:
- 매 scroll 마다 호출 (수천 번/초).
- `getBoundingClientRect` 강제 layout.
- 성능 폭망.

### 0.2 Observer 의 통찰
**브라우저가 효율적으로 변화 감지 → callback**.
- 브라우저 native (빠름).
- throttling 자동.
- intent 명확.

---

## §1. IntersectionObserver

### 1.1 정의
element 가 viewport (또는 ancestor) 와 교차하는지 감지. lazy loading, infinite scroll, analytics 등에.

### 1.2 사용
```javascript
const observer = new IntersectionObserver((entries) => {
    entries.forEach(entry => {
        if (entry.isIntersecting) {
            console.log("visible:", entry.target);
            observer.unobserve(entry.target);    // 한 번만
        }
    });
}, { threshold: 0.5 });    // 50% 보이면 trigger

observer.observe(document.getElementById("lazy-image"));
```

### 1.3 옵션
- `root` — 기준 element. default = viewport.
- `rootMargin` — root 의 margin (예: `"100px"` 면 100px 일찍 trigger).
- `threshold` — 교차 비율 (`0.0 ~ 1.0` 또는 array).

### 1.4 활용

#### lazy image
```javascript
const observer = new IntersectionObserver((entries) => {
    entries.forEach(entry => {
        if (entry.isIntersecting) {
            const img = entry.target;
            img.src = img.dataset.src;
            observer.unobserve(img);
        }
    });
});

document.querySelectorAll("img[data-src]").forEach(img => observer.observe(img));
```

#### infinite scroll
```javascript
const observer = new IntersectionObserver((entries) => {
    if (entries[0].isIntersecting) {
        loadMore();
    }
});
observer.observe(document.getElementById("sentinel"));    // list 끝의 sentinel
```

→ `<div id="sentinel">` 가 보이면 더 로드.

### 1.5 React 와 통합
```typescript
function useIntersection(ref: RefObject<HTMLElement>, options?: IntersectionObserverInit) {
    const [isIntersecting, setIsIntersecting] = useState(false);

    useEffect(() => {
        if (!ref.current) return;
        const observer = new IntersectionObserver(([entry]) => {
            setIsIntersecting(entry.isIntersecting);
        }, options);
        observer.observe(ref.current);
        return () => observer.disconnect();
    }, []);

    return isIntersecting;
}
```

VulnScope 미사용 (작은 list).

### 1.6 함정
- callback 이 micro/macrotask 인지 명세 모호. 즉시 실행 가정 X.
- threshold array (`[0, 0.5, 1]`) 시 매 boundary trigger.

---

## §2. MutationObserver

### 2.1 정의
DOM 변경 (추가/제거/속성) 감지.

```javascript
const observer = new MutationObserver((mutations) => {
    mutations.forEach(m => {
        console.log(m.type, m.target);
        // type: "childList" | "attributes" | "characterData"
    });
});

observer.observe(target, {
    childList: true,         // 자식 추가/제거
    attributes: true,        // 속성 변경
    characterData: true,     // text 변경
    subtree: true,           // 자손 모두
});
```

### 2.2 활용
- 외부 라이브러리 / iframe 의 DOM 변경 감지.
- React 외부에서 React DOM 추적 (legacy 통합).
- A/B test framework.

### 2.3 함정
- 매번 mutations array — 많이 모이면 overhead.
- React app 에서는 거의 안 씀 (React 가 직접 컨트롤).

---

## §3. ResizeObserver

### 3.1 정의
element 의 size 변경 감지. window resize 가 아닌 element 크기.

```javascript
const observer = new ResizeObserver((entries) => {
    entries.forEach(entry => {
        const { width, height } = entry.contentRect;
        console.log(`resized to ${width}x${height}`);
    });
});

observer.observe(element);
```

### 3.2 vs window resize event
- `window.resize` — 윈도우 크기.
- `ResizeObserver` — element 크기 (CSS 변경, content 변경 등).

### 3.3 활용
- responsive component (window 가 아닌 부모 크기 따라).
- canvas 내부 size sync.
- chart 라이브러리 (D3, Chart.js).

### 3.4 React 통합
```typescript
function useSize(ref: RefObject<HTMLElement>) {
    const [size, setSize] = useState({ width: 0, height: 0 });

    useEffect(() => {
        if (!ref.current) return;
        const observer = new ResizeObserver(([entry]) => {
            const { width, height } = entry.contentRect;
            setSize({ width, height });
        });
        observer.observe(ref.current);
        return () => observer.disconnect();
    }, []);

    return size;
}
```

### 3.5 함정
- ResizeObserver loop 위험 — callback 안에서 size 변경하면 무한 loop.
- "ResizeObserver loop limit exceeded" 경고.

---

## §4. PerformanceObserver

### 4.1 정의
performance entry (long task, paint, navigation 등) 관찰.

```javascript
const observer = new PerformanceObserver((list) => {
    list.getEntries().forEach(entry => {
        console.log(entry.name, entry.duration);
    });
});

observer.observe({ entryTypes: ["longtask", "paint", "largest-contentful-paint"] });
```

### 4.2 활용
- Web Vitals 측정 (LCP, FID, CLS).
- Long Task 모니터링.
- custom mark/measure.

### 4.3 entry types
- `longtask` — 50ms+ task.
- `paint` — first-paint, first-contentful-paint.
- `largest-contentful-paint` — LCP.
- `first-input` — FID.
- `layout-shift` — CLS.
- `navigation` — page load timing.
- `resource` — fetch / image / script load.
- `mark` / `measure` — custom.

### 4.4 web-vitals 라이브러리
```javascript
import { onLCP, onINP, onCLS } from "web-vitals";

onLCP((metric) => sendToAnalytics(metric));
onINP((metric) => sendToAnalytics(metric));
onCLS((metric) => sendToAnalytics(metric));
```

내부적으로 PerformanceObserver 사용.

상세는 12-performance-web-vitals.md.

---

## §5. MediaQueryList

### 5.1 정의
CSS media query 의 JS 버전.

```javascript
const mql = window.matchMedia("(min-width: 768px)");
console.log(mql.matches);    // true/false

mql.addEventListener("change", (e) => {
    console.log("matches:", e.matches);
});
```

### 5.2 활용
- responsive component (CSS 가 아닌 JS 분기).
- prefers-color-scheme (dark mode).
- prefers-reduced-motion (animation).

### 5.3 React 통합
```typescript
function useMediaQuery(query: string) {
    const [matches, setMatches] = useState(() => window.matchMedia(query).matches);

    useEffect(() => {
        const mql = window.matchMedia(query);
        const handler = (e: MediaQueryListEvent) => setMatches(e.matches);
        mql.addEventListener("change", handler);
        return () => mql.removeEventListener("change", handler);
    }, [query]);

    return matches;
}

// 사용
const isMobile = useMediaQuery("(max-width: 767px)");
const prefersDark = useMediaQuery("(prefers-color-scheme: dark)");
const reducedMotion = useMediaQuery("(prefers-reduced-motion: reduce)");
```

---

## §6. ReportingObserver (modern)

### 6.1 정의
deprecated API 사용, intervention (브라우저 자동 차단), CSP 위반 등 보고.

```javascript
const observer = new ReportingObserver((reports) => {
    reports.forEach(report => {
        console.log(report.type, report.body);
    });
}, { types: ["deprecation", "intervention"] });

observer.observe();
```

### 6.2 활용
- 운영 환경에서 silent error 발견.
- 분석 backend 로 보냄.

VulnScope 미사용.

---

## §7. EventSource 와 Observer 의 관계

EventSource (SSE) 도 일종의 observer 패턴 — server-sent event 관찰.

`07-streaming-patterns.md` 참조.

---

## §8. 학습 포인트

1. **Observer = 효율적 변화 감지** — naive event listener 대안.
2. **IntersectionObserver = 가시성** — lazy load, infinite scroll.
3. **MutationObserver = DOM 변경** — 외부 통합.
4. **ResizeObserver = element size** — window resize 와 다름.
5. **PerformanceObserver = perf entry** — Web Vitals.
6. **MediaQueryList = CSS query** in JS — dark mode, responsive.
7. **disconnect() 필수** — useEffect cleanup.
8. **ResizeObserver loop 주의** — callback 안 size 변경 X.
9. **threshold/rootMargin = IntersectionObserver 의 미세 제어**.
10. **prefers-reduced-motion** — accessibility 존중.

### 추가 참고
- MDN IntersectionObserver: https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API
- MDN ResizeObserver: https://developer.mozilla.org/en-US/docs/Web/API/ResizeObserver
- web-vitals: https://github.com/GoogleChrome/web-vitals
