# 09. Browser Rendering — Critical Rendering Path

> 이 문서가 다루는 것: HTML/CSS/JS 가 화면이 되기까지. parse, layout, paint, composite. reflow vs repaint. GPU 가속.

---

## §0. 왜 알아야?

- 성능 튜닝의 기반.
- 왜 width 변경이 layout 트리거하는지.
- 왜 transform/opacity 가 빠른지.
- React render 와 browser 의 paint 가 다른 phase.

---

## §1. Critical Rendering Path

### 1.1 5 단계
```
1. HTML → DOM tree
2. CSS → CSSOM tree
3. DOM + CSSOM → Render tree
4. Layout (Reflow) — 위치/크기 계산
5. Paint — 픽셀로 그림
6. Composite — layer 합성 → 화면
```

### 1.2 시각화
```
HTML → DOM ─┐
            ├─→ Render Tree → Layout → Paint → Composite → 🖼️
CSS → CSSOM ┘
```

### 1.3 JS 의 위치
JS 가 중간에 끼면:
- `<script>` 가 HTML parsing 차단.
- `<script async>` — 다운로드는 병렬, 실행은 차단.
- `<script defer>` — 다운로드 병렬, DOMContentLoaded 직전 실행.

→ `<script defer>` 가 권장.

---

## §2. DOM tree

### 2.1 HTML → DOM
```html
<div>
    <h1>Hello</h1>
    <p>World</p>
</div>
```

→
```
div
├── h1
│   └── "Hello" (text node)
└── p
    └── "World"
```

### 2.2 동적 변경
JS 가 `document.createElement`, `appendChild` 등으로 DOM 변경 → reflow/repaint.

React 는 Virtual DOM 으로 변경 최소화.

---

## §3. CSSOM tree

### 3.1 CSS 도 tree
```css
body { font-size: 14px; }
body p { color: red; }
```

→ 각 selector 가 어떤 노드에 적용되는지 트리.

### 3.2 cascading
selector 우선순위:
1. inline style.
2. ID > class > tag.
3. specificity 같으면 나중 선언.

### 3.3 inheritance
text-related (color, font) 는 inherited. layout-related (width, padding) 는 X.

### 3.4 CSS 의 비용
- selector 매칭 = bottom-up (오른쪽부터). 깊은 selector 비쌈.
- `* { ... }` universal selector 비쌈 (모든 노드).

---

## §4. Render tree

### 4.1 정의
DOM + CSSOM 합성 → 실제 화면에 그릴 노드만.

- `display: none` 은 render tree 에 X.
- `visibility: hidden` 은 있음 (공간만 차지).

### 4.2 의미
- 보이는 것만 layout/paint.
- React 의 conditional render 가 display:none 보다 가벼울 수 있음 (DOM 자체 X).

---

## §5. Layout (Reflow)

### 5.1 정의
각 노드의 정확한 위치 + 크기 계산.

`width: 50%` → 부모 width 기반 계산. cascading 의 정확한 numeric value.

### 5.2 trigger
- DOM 추가/제거.
- CSS 변경 중 layout-related (width, height, padding, margin, display, position 등).
- 윈도우 resize.
- font 변경.

### 5.3 비용
- 변경된 노드 + 영향 노드 (자식 + 형제) 다시 계산.
- 깊은 tree = 비싼 layout.
- "layout thrashing" — 빈번 layout = 성능 폭망.

### 5.4 layout-trigger CSS
무거운 (피하면 좋음):
- width / height
- top / left / right / bottom
- padding / margin
- border-width
- font-size
- display
- float, position

가벼운 (composite layer 만):
- transform
- opacity
- filter (일부)

---

## §6. Paint

### 6.1 정의
각 노드를 픽셀로 그림. background, color, border, shadow 등.

### 6.2 trigger
- color, background-color 변경.
- border 변경.
- box-shadow.
- visibility.

### 6.3 비용
- 영역 크면 비쌈.
- 복잡 효과 (blur, shadow) 비쌈.

### 6.4 layer 분리
일부 element 는 별도 layer (composite layer). transform/opacity 변경 시 layer 만 다시 composite. paint 안 일어남.

---

## §7. Composite

### 7.1 정의
여러 layer 를 GPU 에서 합성 → 최종 화면.

### 7.2 layer 생성 조건
- `transform: translateZ(0)` 또는 `will-change: transform`.
- `position: fixed`.
- `opacity` < 1.
- video, canvas, iframe.
- CSS animation 일부.

### 7.3 GPU 가속의 핵심
GPU 에서 처리 가능:
- transform (translate, rotate, scale).
- opacity.
- filter (일부).

→ 60fps 부드러움.

### 7.4 함정
- 너무 많은 layer = GPU 메모리 폭증 + 합성 비용.
- "will-change: *" 남용 X.

---

## §8. Reflow vs Repaint

### 8.1 reflow (= layout)
- 위치/크기 재계산.
- 비싸 (자식 영향).

### 8.2 repaint
- 픽셀만 다시 그림.
- 위치/크기 그대로.

### 8.3 composite only
- layer 합성만.
- 가장 가볍.

### 8.4 어느 변경이 어느 단계?
| 변경 | reflow | repaint | composite |
|---|---|---|---|
| `width` | ✅ | ✅ | ✅ |
| `color` | ❌ | ✅ | ✅ |
| `transform` | ❌ | ❌ | ✅ |
| `opacity` | ❌ | ❌ | ✅ |
| `display: none → block` | ✅ | ✅ | ✅ |

→ animation 은 transform/opacity 권장.

### 8.5 예
```css
/* ❌ reflow + repaint */
.move {
    transition: left 0.3s;
}
.move.active {
    left: 100px;
}

/* ✅ composite only */
.move {
    transition: transform 0.3s;
}
.move.active {
    transform: translateX(100px);
}
```

→ 둘 다 100px 이동. 후자가 60fps.

---

## §9. layout thrashing

### 9.1 패턴
```javascript
for (let i = 0; i < 100; i++) {
    const el = elements[i];
    el.style.width = `${el.offsetWidth + 1}px`;    // ❌
}
```

`offsetWidth` (read) → layout 강제.
`style.width = ...` (write) → layout 무효화.
→ 매 iteration 마다 layout = 100번.

### 9.2 해결 — read 다음 write
```javascript
const widths = [];
for (let i = 0; i < 100; i++) {
    widths.push(elements[i].offsetWidth);    // 모두 read 먼저
}
for (let i = 0; i < 100; i++) {
    elements[i].style.width = `${widths[i] + 1}px`;    // 모두 write
}
```

→ layout 1번.

### 9.3 React 와 layout
React 는 batch 으로 DOM 업데이트 → layout thrashing 회피.

직접 DOM 조작 시 주의.

---

## §10. requestAnimationFrame 의 timing

`07-browser-event-loop.md` §4 참조.

핵심: rAF 는 **layout/paint 직전 callback**. 다음 frame 에 반영. animation 표준.

---

## §11. CSS containment

### 11.1 contain
```css
.card {
    contain: layout paint;
}
```

→ "이 element 의 layout/paint 가 외부에 영향 X" 보장.

브라우저가 isolated 처리. 성능 ↑.

### 11.2 종류
- `contain: layout` — layout 영향 격리.
- `contain: paint` — paint 영향 격리.
- `contain: size` — size 영향 격리.
- `contain: strict` — 모두.

### 11.3 활용
무거운 컴포넌트 (스크롤 list, 큰 카드) 에 적용.

---

## §12. content-visibility

### 12.1 정의
```css
.below-fold {
    content-visibility: auto;
}
```

→ 화면 밖이면 자동으로 render skip. scroll 가까워지면 render.

### 12.2 효과
- 큰 페이지 (1000개 element) 의 초기 paint 빠름.
- 안 보이는 영역 lazy.

### 12.3 함정
- 정확한 height 모름 → scroll bar jump 가능. `contain-intrinsic-size` 로 hint.

---

## §13. will-change

### 13.1 사용
```css
.animated {
    will-change: transform;
}
```

→ "이 element 의 transform 곧 변경" 알림. 브라우저가 미리 layer 만듦.

### 13.2 함정
- 남용 X — 매 element 면 GPU 메모리 폭증.
- animation 끝나면 제거.

---

## §14. 성능 측정

### 14.1 Performance API
```javascript
performance.mark("start");
// ... heavy work
performance.mark("end");
performance.measure("work", "start", "end");
const measures = performance.getEntriesByType("measure");
```

### 14.2 PerformanceObserver
```javascript
new PerformanceObserver((list) => {
    list.getEntries().forEach(entry => {
        console.log(entry.name, entry.duration);
    });
}).observe({ entryTypes: ["measure", "longtask", "paint"] });
```

### 14.3 Chrome DevTools
- Performance tab — flame chart.
- Layers tab — layer 시각화.
- Rendering tab — paint flash, layout shift.

---

## §15. 학습 포인트

1. **CRP** = Parse → DOM/CSSOM → Render tree → Layout → Paint → Composite.
2. **JS `<script defer>`** 가 권장 (parsing 비차단).
3. **layout-trigger CSS 비쌈** (width, padding 등).
4. **transform/opacity = composite only** — 가장 빠름, 60fps.
5. **layer 분리 = GPU 가속** but 남용 X.
6. **layout thrashing** = read/write 인터리브. 해결: 먼저 read 다 → 그 후 write.
7. **CSS containment** = 격리로 성능 ↑.
8. **content-visibility: auto** = 화면 밖 lazy.
9. **will-change** = layer 미리 — 남용 X.
10. **rAF = paint 직전** — animation 표준.

### 추가 참고
- web.dev, [Critical Rendering Path](https://web.dev/articles/critical-rendering-path)
- Paul Lewis, [High Performance Animations](https://web.dev/articles/animations)
- Chrome DevTools Rendering panel
