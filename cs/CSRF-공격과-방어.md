# CSRF (Cross-Site Request Forgery) 공격과 방어 가이드

## 1. CSRF란?

**사용자가 의도하지 않은 요청을, 사용자의 인증 정보를 도용해 서버로 보내게 만드는 공격**입니다.

핵심은 **브라우저가 쿠키를 자동으로 함께 전송한다**는 점입니다. 사용자가 A 사이트에 로그인되어 있는 상태에서 공격자의 B 사이트를 방문하면, B 사이트가 A 사이트로 요청을 보낼 때 브라우저가 A의 쿠키를 자동으로 첨부합니다. 서버는 정상 사용자의 요청으로 인식합니다.

> **XSS와의 차이**
> - XSS: 공격자의 코드가 **피해자 브라우저에서 실행**됨 (스크립트 주입)
> - CSRF: 공격자가 **피해자 권한으로 요청을 위조**함 (요청 위조)

---

## 2. 공격 시나리오

### 2.1 GET 요청 공격

사용자가 은행 사이트에 로그인된 상태에서 공격자의 페이지를 방문합니다.

```html
<!-- attacker.com 페이지 -->
<img src="https://bank.com/transfer?to=attacker&amount=1000000" />
```

브라우저가 이미지를 로드하려고 GET 요청을 보내면서 bank.com의 쿠키가 자동 첨부됩니다. 서버는 정상 요청으로 처리합니다.

### 2.2 POST 요청 공격

```html
<!-- attacker.com 페이지 -->
<form action="https://bank.com/transfer" method="POST" id="f">
  <input name="to" value="attacker" />
  <input name="amount" value="1000000" />
</form>
<script>document.getElementById('f').submit()</script>
```

페이지 로드 시 자동으로 폼이 전송되고, 쿠키도 함께 전송됩니다.

### 2.3 JSON API 공격

요즘은 대부분 JSON API를 쓰지만, fetch로 `Content-Type: application/json` 요청을 보낼 때는 CORS preflight가 작동하므로 일정 부분 막힙니다. 다만 `text/plain`으로 우회하거나, 인증이 쿠키 기반이면 여전히 위험합니다.

---

## 3. 공격이 성립하는 조건

다음 조건이 **모두** 충족돼야 CSRF가 성립합니다.

1. **쿠키 기반 인증 사용** (서버가 쿠키만 보고 사용자를 식별)
2. **사용자가 대상 사이트에 로그인된 상태**
3. **상태 변경 요청에 추가 검증이 없음** (CSRF 토큰 등)
4. **사용자가 공격 페이지를 방문**

조건 중 하나만 깨도 공격은 차단됩니다. 그래서 방어 전략은 **여러 조건을 동시에 깨는 다층 방어**가 됩니다.

---

## 4. 일반적인 방어 기법

### 4.1 CSRF 토큰 (가장 전통적)

서버가 세션마다 예측 불가능한 토큰을 발급하고, 상태 변경 요청에 그 토큰을 함께 보내도록 요구합니다.

```html
<form method="POST" action="/transfer">
  <input type="hidden" name="_csrf" value="a1b2c3d4..." />
  <input name="amount" />
</form>
```

서버는 요청의 토큰과 세션의 토큰을 비교해 일치할 때만 처리합니다. 공격자는 다른 출처에서 이 토큰을 읽을 수 없으므로(Same-Origin Policy) 위조할 수 없습니다.

### 4.2 SameSite 쿠키 (현대적 기본 방어)

쿠키에 `SameSite` 속성을 붙이면 다른 사이트에서 시작된 요청에는 쿠키가 첨부되지 않습니다.

```
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict
```

| 값 | 동작 |
|----|------|
| `Strict` | 다른 사이트에서 온 모든 요청에 쿠키 미첨부 (가장 엄격) |
| `Lax` | GET 같은 안전한 top-level navigation은 허용 (현재 브라우저 기본값) |
| `None` | 모든 요청에 쿠키 첨부 (`Secure` 필수, CSRF 위험 ↑) |

대부분의 모던 브라우저는 `Lax`를 기본값으로 적용해서, **SameSite만으로도 상당수의 CSRF가 자동으로 막힙니다.**

### 4.3 Double Submit Cookie

쿠키와 요청 헤더(또는 바디)에 같은 토큰을 보내고, 서버는 둘이 일치하는지만 확인합니다. 세션 저장소가 필요 없어 stateless API에서 유용합니다.

```
Cookie: csrf-token=xyz789
X-CSRF-Token: xyz789
```

공격자는 다른 출처에서 쿠키를 읽거나 임의의 헤더를 설정할 수 없으므로 두 값을 일치시킬 수 없습니다.

### 4.4 Origin / Referer 헤더 검증

서버가 요청의 `Origin` 또는 `Referer` 헤더를 확인해 자기 도메인에서 온 요청만 허용합니다.

```js
const allowedOrigins = ['https://myapp.com']
if (!allowedOrigins.includes(req.headers.origin)) {
  return res.status(403).send('Forbidden')
}
```

브라우저가 자동으로 첨부하는 헤더이며 JavaScript로 위조할 수 없어 신뢰할 수 있습니다.

### 4.5 Custom Header 요구

`X-Requested-With: XMLHttpRequest` 같은 커스텀 헤더를 요구합니다. 단순 폼 전송으로는 커스텀 헤더를 붙일 수 없고, fetch/XHR로 커스텀 헤더를 보내려면 CORS preflight가 발생하므로 차단됩니다.

```js
fetch('/api/transfer', {
  method: 'POST',
  headers: { 'X-Requested-With': 'fetch' },
  body: JSON.stringify(data)
})
```

### 4.6 인증 방식 변경 (토큰 기반)

쿠키 대신 **Authorization 헤더에 토큰**을 실어 보내면 CSRF는 원천 차단됩니다. 브라우저가 자동으로 첨부하지 않기 때문입니다.

```js
fetch('/api/transfer', {
  headers: { 'Authorization': `Bearer ${token}` }
})
```

단, 토큰을 어디에 저장할지가 새 문제가 됩니다 (localStorage는 XSS에 취약).

---

## 5. 프론트엔드 기술별 방어 방식

### 5.1 React / Next.js (쿠키 기반 세션)

**SameSite 쿠키 + CSRF 토큰**의 조합이 일반적입니다.

```jsx
// 서버에서 토큰 발급 후 클라이언트가 헤더로 전송
async function transfer(data) {
  const csrfToken = await fetch('/api/csrf-token').then(r => r.text())
  
  return fetch('/api/transfer', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-CSRF-Token': csrfToken
    },
    credentials: 'include',
    body: JSON.stringify(data)
  })
}
```

**Next.js Server Actions의 내장 방어**

Next.js 14+의 Server Actions는 자동으로 Origin 헤더와 Host 헤더를 비교해 CSRF를 차단합니다. 별도 토큰 작업이 거의 필요 없습니다.

```jsx
// app/actions.js
'use server'
export async function transfer(formData) {
  // Next.js가 자동으로 Origin 검증
  // 다른 출처에서 호출 시 차단됨
}
```

`next.config.js`에서 허용 Origin을 명시할 수도 있습니다.

```js
module.exports = {
  experimental: {
    serverActions: {
      allowedOrigins: ['myapp.com', '*.myapp.com']
    }
  }
}
```

### 5.2 Express / Node.js 백엔드

`csurf`는 deprecated되었으므로 `csrf-csrf`(Double Submit) 또는 직접 구현을 권장합니다.

```js
import { doubleCsrf } from 'csrf-csrf'

const { doubleCsrfProtection, generateToken } = doubleCsrf({
  getSecret: () => process.env.CSRF_SECRET,
  cookieName: '__Host-csrf',
  cookieOptions: { httpOnly: true, secure: true, sameSite: 'strict' }
})

app.get('/csrf-token', (req, res) => {
  res.json({ token: generateToken(req, res) })
})

app.post('/transfer', doubleCsrfProtection, (req, res) => {
  // 보호됨
})
```

### 5.3 SPA + REST API (분리형 아키텍처)

프론트와 API 도메인이 분리된 경우 다음 조합을 권장합니다.

1. **Authorization 헤더에 JWT** (쿠키 인증을 안 쓰면 CSRF 자체가 무력화)
2. **CORS를 정확히 설정** (`Access-Control-Allow-Origin`을 와일드카드가 아닌 특정 도메인으로)
3. **토큰은 메모리 또는 HttpOnly 쿠키에 저장** (localStorage는 XSS 노출)

```js
// axios 인터셉터 예시
axios.interceptors.request.use(config => {
  const token = getTokenFromMemory()
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})
```

### 5.4 Vue / Nuxt

Nuxt 3은 `nuxt-csurf` 모듈로 간편하게 적용할 수 있습니다.

```js
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['nuxt-csurf'],
  csurf: {
    cookieKey: 'csrf',
    methodsToProtect: ['POST', 'PUT', 'DELETE']
  }
})
```

```vue
<script setup>
const { $csrfFetch } = useNuxtApp()
await $csrfFetch('/api/transfer', { method: 'POST', body: data })
</script>
```

### 5.5 Angular

Angular의 `HttpClient`는 **CSRF 보호가 내장**되어 있습니다.

```ts
import { provideHttpClient, withXsrfConfiguration } from '@angular/common/http'

bootstrapApplication(AppComponent, {
  providers: [
    provideHttpClient(
      withXsrfConfiguration({
        cookieName: 'XSRF-TOKEN',
        headerName: 'X-XSRF-TOKEN'
      })
    )
  ]
})
```

서버가 `XSRF-TOKEN` 쿠키를 발급하면, Angular가 자동으로 그 값을 읽어서 `X-XSRF-TOKEN` 헤더로 첨부합니다. (Double Submit 패턴)

---

## 6. 실전 권장 조합

상황별로 다음 조합을 권장합니다.

| 아키텍처 | 권장 방어 |
|----------|----------|
| 전통적 SSR 웹 (쿠키 세션) | SameSite=Strict + CSRF 토큰 + Origin 검증 |
| Next.js App Router | Server Actions 활용 + SameSite=Lax |
| SPA + 자체 API | JWT (Authorization 헤더) + CORS 화이트리스트 |
| 모바일 앱 + API | JWT + 모바일은 쿠키 미사용 |

---

## 7. 실전 체크리스트

- [ ] 세션 쿠키에 `SameSite=Lax` 또는 `Strict`를 설정한다
- [ ] 세션 쿠키에 `HttpOnly`, `Secure` 플래그를 붙인다
- [ ] 상태 변경 요청(POST, PUT, DELETE)에 CSRF 토큰을 요구한다
- [ ] 서버에서 `Origin` / `Referer` 헤더를 검증한다
- [ ] CORS 설정을 와일드카드(`*`)가 아닌 명시적 도메인으로 제한한다
- [ ] GET 요청이 상태를 변경하지 않도록 설계한다 (멱등성 유지)
- [ ] 중요 작업(송금, 비밀번호 변경 등)은 재인증을 요구한다
- [ ] JWT 사용 시 토큰 저장소를 신중히 선택한다 (메모리 > HttpOnly 쿠키 > localStorage)

---

## 8. 핵심 정리

> CSRF의 본질은 **"브라우저가 쿠키를 자동 첨부한다는 사실의 악용"**이며,
> 방어의 본질은 **"이 요청이 정말 우리 사이트에서 시작됐는가"를 검증하는 것**입니다.

현대 웹에서는 **SameSite 쿠키**가 기본 방어선을 자동으로 깔아주고, **CSRF 토큰** 또는 **Origin 검증**이 추가 안전망 역할을 합니다. SPA 환경에서 **JWT를 Authorization 헤더로 보낸다면 CSRF는 구조적으로 발생하지 않습니다.**

XSS와 CSRF는 함께 다뤄야 합니다. XSS가 뚫리면 CSRF 방어(토큰 등)도 우회되기 때문에, **두 공격을 분리하지 말고 같은 보안 모델 안에서 함께 설계**하는 것이 중요합니다.
