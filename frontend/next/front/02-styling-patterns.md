# 02. CSS / 스타일링 패턴

> 이 문서가 다루는 것: VulnScope 가 사용한 디자인 토큰, utility-first, component layer (`wf-*`), font composition, animation, CSS variable 패턴 — **아이디어와 의도**.

---

## §0. 스타일링 의 4가지 학파

CSS 스타일링은 역사적으로 4단계:

1. **글로벌 CSS** (90s~) — `style.css` 에 모두. 클래스 충돌, naming 어려움.
2. **CSS Methodology** (2010~) — BEM, OOCSS, SMACSS. naming convention.
3. **CSS-in-JS** (2015~) — styled-components, emotion. JS 에서 컴포넌트 단위. runtime cost.
4. **Utility-first** (2017~) — Tailwind. atomic class. 빌드 타임 추출.

VulnScope 는 **Tailwind v4 (utility-first) + custom component layer (`wf-*`)** 결합.

이게 왜 좋은지부터.

---

## §1. Utility-first 의 본질적 아이디어

### 1.1 전통적 CSS 의 문제
```html
<button class="login-button">Sign in</button>
```
```css
/* style.css */
.login-button {
    height: 32px;
    padding: 6px 12px;
    border: 1.2px solid #1a1a1a;
    background: white;
    /* ... */
}
```

**문제**:
- `.login-button` 의 의미가 점점 모호해짐 — 다른 곳에서도 쓰면 비슷한 거 또 만들거나 modifier (`.login-button--small`) 추가.
- CSS 파일이 무한히 커짐.
- naming 매번 고민 (BEM 등 method 등장 이유).

### 1.2 Utility-first 의 통찰
```html
<button class="h-8 px-3 py-1.5 border border-line bg-bg text-ink">
    Sign in
</button>
```

**아이디어**:
- 모든 CSS property 가 atomic class (`h-8` = `height: 2rem`).
- HTML 만 보면 스타일 다 보임.
- 새 CSS 파일 안 늘어남 (atomic class 재사용).
- naming 안 함.

**비판 + 반박**:
- "HTML 이 더러워짐" → 컴포넌트로 wrap (Btn, Input 등). HTML 길이는 같지만 의도 명확.
- "CSS 이름의 의미 손실" → `.login-button` 같은 의미는 컴포넌트 이름 (`<LoginButton>`) 으로.

### 1.3 Tailwind 의 atomic class 생성 원리
빌드 타임에 source 파일 (`*.tsx`) 스캔 → 사용한 class 만 CSS 출력.

```typescript
<div className="h-8 px-3" />
```
→ 빌드 결과에 `.h-8 { height: 2rem; }`, `.px-3 { padding-left: 0.75rem; padding-right: 0.75rem; }` 만 포함. 안 쓴 class 는 출력 X.

→ **CSS 번들 사이즈 = 실제 사용한 class 만**. 일반 CSS 는 모든 class 항상 포함.

### 1.4 직접 구현한다면
빌드 타임 utility 생성:
1. CSS template (`{prop}-{value}` 패턴 정의).
2. source 파일 스캔 (regex `class(?:Name)?="([^"]+)"`).
3. 사용된 class 추출 → 해당 CSS 만 생성.

PostCSS plugin 으로 가능. Tailwind 가 정밀 + 풍부한 plugin (variants, dark mode 등).

---

## §2. Tailwind v4 의 `@theme` + 디자인 토큰

### 2.1 디자인 토큰 이란?
**design token = 디자인 시스템의 atomic 변수**.
- color: `--color-primary`, `--color-secondary`.
- spacing: `--space-sm`, `--space-md`.
- font: `--font-sans`, `--font-mono`.
- border: `--border-width`, `--border-radius`.

→ "primary 색상" 같은 의미가 한 곳에 정의 → 변경 시 일괄 반영.

### 2.2 VulnScope 의 토큰
```css
/* globals.css */
@import "tailwindcss";

@theme {
  --color-ink-1: #1a1a1a;             /* primary text */
  --color-ink-2: #444444;             /* secondary */
  --color-ink-3: #888888;             /* tertiary / dim */
  --color-line-1: #1a1a1a;
  --color-line-2: #bbbbbb;
  --color-bg: #fdfdfa;
  --color-fill: #f4f1ea;

  /* severity palette */
  --color-severity-critical: #e8624a;
  --color-severity-high: #d97847;
  --color-severity-medium: #f0a830;
  --color-severity-low: #5a8fbf;
  --color-severity-info: #888888;
  --color-severity-fixed: #4a8a5c;

  --color-link: #5a8fbf;
  --color-highlight: #fef4a8;
  --color-log-bg: #f8f7f3;

  --font-sans: var(--font-sans), "Inter", system-ui, sans-serif;
  --font-handwritten: var(--font-handwritten), "Bradley Hand", "Marker Felt", cursive;
  --font-mono: var(--font-mono), ui-monospace, "Courier New", monospace;
}
```

### 2.3 Tailwind v4 의 `@theme` 마법
**v3 (JS 설정)**:
```javascript
// tailwind.config.js
module.exports = {
    theme: {
        colors: { 'ink-1': '#1a1a1a', ... },
    }
};
```
→ JS object 로 정의. 빌드 시 utility class 생성.

**v4 (CSS @theme)**:
```css
@theme {
    --color-ink-1: #1a1a1a;
}
```
→ CSS 안에 정의. **자동으로 utility 생성** (`bg-ink-1`, `text-ink-1`, `border-ink-1`).

→ 토큰 정의 = utility 생성. JS 설정 불필요. CSS-first.

### 2.4 CSS variable 의 의미
`--color-ink-1` 은 그냥 CSS variable. 런타임에:
```typescript
<div style={{ color: "var(--color-ink-1)" }}>...</div>
```
→ 직접 사용 가능. 동적 값 (예: dark mode, theme 변경) 가능.

### 2.5 핵심 통찰 — semantic token
naive token:
```css
--color-blue-500: #3b82f6;        /* 색상 이름 */
```
**문제**: "blue 가 primary 인지 secondary 인지" 코드만 봐서 모름.

**semantic token** (VulnScope):
```css
--color-ink-1: #1a1a1a;            /* 의미 = primary text */
--color-severity-critical: #e8624a; /* 의미 = critical 위험 */
```
**효과**:
- 사용처에서 의미 읽힘 (`text-ink-1` = "primary text 색").
- 색상 변경 시 의미 그대로.

VulnScope 의 `ink-{1,2,3}` = primary/secondary/tertiary. `severity-*` = 도메인 의미.

### 2.6 직접 구현한다면
1. CSS variable 로 토큰 정의.
2. utility class 자동 생성 — PostCSS plugin 또는 build script.
3. 또는 단순히 CSS variable 만 + 일반 CSS 작성.

**가장 단순한 방식**:
```css
:root {
    --color-primary: #1a1a1a;
}
.button-primary { background: var(--color-primary); }
```
Tailwind 안 써도 가능. 단, atomic 의 편의성이 없음.

---

## §3. Component layer (`wf-*` namespace)

### 3.1 utility-first 의 한계
유틸리티만 사용하면:
```html
<button class="inline-flex items-center justify-center gap-2 px-3 py-1.5 border border-line bg-bg text-ink-1 text-xs font-medium rounded-sm">
```
→ 길고 반복적. 매 button 마다 같은 class 줄.

### 3.2 해결: component layer
```css
@layer components {
    .wf-btn {
        display: inline-flex;
        align-items: center;
        justify-content: center;
        gap: 6px;
        padding: 6px 12px;
        border: 1.2px solid var(--color-line-1);
        background: var(--color-bg);
        font-size: 11px;
        font-weight: 500;
        color: var(--color-ink-1);
        border-radius: 2px;
    }
    .wf-btn-primary { background: var(--color-ink-1); color: #fff; }
    .wf-btn-sm { padding: 3px 8px; font-size: 10px; }
}
```

```html
<button class="wf-btn wf-btn-primary">Sign in</button>
```

### 3.3 Tailwind 의 `@layer components` 의 의미
- Tailwind 3-layer 시스템: `@base` (reset) / `@components` (의미 단위) / `@utilities` (atomic).
- specificity 자동 처리 — utility 가 component 보다 강함 (override 가능).

### 3.4 VulnScope 의 컴포넌트 레이어 ~20개
`wf-topbar`, `wf-sidebar`, `wf-nav-item`, `wf-btn`, `wf-input`, `wf-card`, `wf-stat`, `wf-table`, `wf-chip`, `wf-progress`, `wf-log`, `wf-note`, `wf-scribble`, `wf-blink`, `wf-sev`, `wf-sev-dot`, ...

→ 디자인 시스템의 의미 단위. React 컴포넌트 (Btn, Input 등) 에 1:1 매핑.

### 3.5 핵심 통찰 — utility + component 결합
- atomic utility = 자유 + 일회성 (`mt-2`, `flex`, `gap-3`).
- component class = 재사용 + 의미 (`wf-btn`).

**둘 다 사용**:
```html
<button class="wf-btn wf-btn-primary !h-11 !px-5 mt-2">
  ↑ 의미                ↑ utility (override + 추가)
```

`!` prefix = `!important`. component layer override.

### 3.6 BEM-ish naming
```
.wf-btn          ← block
.wf-btn-primary  ← block + modifier
.wf-btn-sm       ← block + modifier
```

BEM (`.btn__icon--small`) 보다 단순. underscore X. 직관적.

**왜 prefix `wf-`?**: "wireframe" 의 약자. 디자인 시스템 namespace. 다른 css 와 충돌 회피.

### 3.7 직접 구현한다면
1. design token 정의 (CSS variable).
2. component CSS class 작성 (`@layer components` 또는 일반 CSS).
3. React 컴포넌트 wrap (`Btn.tsx` 가 `wf-btn` 적용).
4. utility class 추가 (Tailwind 또는 직접 atomic CSS).

핵심: **token → component class → React component** 3단계.

---

## §4. Font composition

### 4.1 VulnScope 의 3 폰트
```typescript
// app/layout.tsx
const inter = Inter({
    subsets: ["latin"],
    variable: "--font-sans",
    display: "swap",
});

const caveat = Caveat({
    subsets: ["latin"],
    variable: "--font-handwritten",
    display: "swap",
    weight: ["400", "500", "600", "700"],
});

const jetbrains = JetBrains_Mono({
    subsets: ["latin"],
    variable: "--font-mono",
    display: "swap",
});

export default function RootLayout({ children }) {
    return (
        <html className={`${inter.variable} ${caveat.variable} ${jetbrains.variable}`}>
            <body className="wf">
                {children}
            </body>
        </html>
    );
}
```

### 4.2 next/font 의 통찰
**전통적 방식**:
```html
<link href="https://fonts.googleapis.com/css?family=Inter" rel="stylesheet">
```
→ Google CDN 호출. privacy + CLS (Cumulative Layout Shift) 위험.

**next/font**:
- 빌드 타임 폰트 다운로드 → self-host.
- privacy: Google CDN 호출 0.
- CLS: 폰트 metric 미리 계산 → fallback 폰트 size 자동 매칭.

### 4.3 CSS variable 로 폰트 노출
`Inter({ variable: "--font-sans" })` → `<html style="--font-sans: 'Inter'">`.

→ 자식 컴포넌트 어디서든 `font-family: var(--font-sans)` 사용 가능.

### 4.4 3 폰트의 의미
| 폰트 | 용도 | 의미 |
|---|---|---|
| Inter (sans) | body, label | 기본 가독성 |
| Caveat (handwritten) | h1, hero, dev hint | 친근, 인간적 |
| JetBrains Mono | 코드, URL, ID | 도구/터미널 vibe |

→ **폰트 mix 가 디자인 정체성** 의 일부. 도메인 의미와 폰트 매핑.

### 4.5 직접 구현한다면
1. 폰트 파일 (`woff2`) 직접 호스팅.
2. `@font-face` CSS.
3. CSS variable 로 노출.
4. fallback 폰트 명시.

next/font 가 자동화 + CLS 방지 + privacy.

---

## §6. CSS animation

### 6.1 wf-blink (terminal cursor)
```css
.wf-blink {
    animation: wf-blink 1s steps(2) infinite;
}
@keyframes wf-blink {
    0%, 50% { opacity: 1; }
    51%, 100% { opacity: 0; }
}
```

VulnScope 의 ScanLiveDashboard 가 `<span className="wf-blink">_</span>` 로 cursor 시뮬.

### 6.2 wf-stripe (progress bar)
```css
.wf-progress.striped .wf-progress-bar {
    background-image: linear-gradient(45deg, ...);
    animation: wf-stripe 1s linear infinite;
}
@keyframes wf-stripe {
    from { background-position: 0 0; }
    to   { background-position: 16px 0; }
}
```

### 6.3 핵심 통찰
- **CSS animation = JS 없이 60fps**. GPU compositing.
- 단순 효과는 CSS 가 항상 우월.
- 복잡한 interaction (drag 등) 만 JS.

### 6.4 직접 구현한다면
- `@keyframes` 정의.
- `animation: name duration timing-function iteration-count;` 적용.
- transform/opacity 만 사용 → GPU 가속 (layout 안 일어남).

---

## §6. Tailwind 의 modifier (responsive, hover, etc)

### 6.1 사용 예
```html
<div class="grid grid-cols-1 md:grid-cols-2 gap-3">
  ↑ default 1열, md (≥768px) 부터 2열
```

```html
<button class="hover:underline focus:ring-2">
  ↑ hover/focus 상태에 적용
```

### 6.2 핵심 통찰
**modifier prefix** (`md:`, `hover:`) 가 utility class 의 적용 조건.

빌드 타임에 적절한 CSS 생성:
```css
@media (min-width: 768px) {
    .md\:grid-cols-2 { grid-template-columns: repeat(2, ...); }
}
.hover\:underline:hover { text-decoration: underline; }
```

### 6.3 직접 구현한다면
- responsive: `@media (...)` 로 wrap.
- pseudo-class: `:hover`, `:focus` 등.
- modifier 시스템 = 변형 가능한 class 생성기.

---

## §7. Inline style vs class

### 7.1 VulnScope 의 사용
```typescript
// SeverityStat — 동적 색상은 inline style
<Chip
  style={
    currentType === t
      ? { borderColor: "var(--color-ink-1)", color: "var(--color-ink-1)" }
      : undefined
  }
>
```

```typescript
// 고정 스타일은 class
<button className="wf-btn wf-btn-primary">
```

### 7.2 핵심 통찰
- **inline style** = 동적 값 (런타임 계산). props 의존.
- **class** = 정적 + 재사용.
- 둘 다 사용 OK. 의도 분리.

### 7.3 함정
- inline style 너무 많으면 CSS class 의 cache 효과 잃음.
- conditional class 는 utility 권장: `clsx` 같은 헬퍼.

VulnScope 는 직접 배열 + filter (lightweight):
```typescript
const cls = ["wf-btn", variant === "primary" ? "wf-btn-primary" : "", className]
    .filter(Boolean)
    .join(" ");
```

---

## §8. 패턴 매트릭스

| 문제 | 패턴 |
|---|---|
| naming + reuse 폭증 | utility-first |
| 의미 한 곳 + 변경 일괄 | design token (CSS var) |
| 토큰 → utility 자동 | Tailwind v4 @theme |
| utility 반복 → 의미 묶음 | component layer (wf-*) |
| 폰트 mix + privacy | next/font + CSS variable |
| 빠른 effect | CSS animation |
| breakpoint/state | Tailwind modifier |
| 동적 vs 정적 스타일 | inline style + class |

---

## §9. 학습 포인트

1. **utility-first 의 통찰** — atomic class + 빌드 타임 추출.
2. **design token = 의미 한 곳** — semantic naming (ink-1 vs blue-500).
3. **Tailwind v4 @theme** — CSS-first config. 토큰 정의 = utility 생성.
4. **component layer 가 utility 의 짝** — 자유도 + 재사용.
5. **wf-* namespace** — 디자인 시스템 + React 컴포넌트 1:1 매핑.
6. **next/font self-host** — privacy + CLS.
7. **3 폰트 mix = 의미 분리** — body/heading/code.
8. **CSS animation > JS animation** — GPU 가속.
9. **inline style = 동적, class = 정적**.
10. **clsx-like helper** — class 합성.

### 추가 참고
- Tailwind v4 docs: https://tailwindcss.com/
- Adam Wathan, [CSS Utility Classes and "Separation of Concerns"](https://adamwathan.me/css-utility-classes-and-separation-of-concerns/)
- next/font docs: https://nextjs.org/docs/app/api-reference/components/font
- Design Tokens W3C: https://design-tokens.github.io/community-group/format/
