# Anti-Bot 방어 기법 — 탐지 신호 (Python 분석가의 시각)

> 트레이드마크 검색 사이트(외부 검색 사이트) 의 9-레이어 방어 체계를 분석하면서 정리한 학습 노트.
> "어떻게 만들면 봇이 못 들어오는가" 의 시각으로 풀어 적음. 인프라 관점은 별도 폴더 참고.

---

## 1. 탐지 시그널 7개 카테고리

| 카테고리 | 시그널 예시 | Python 대응 가능성 |
|---------|------------|------------------|
| 디바이스 핑거프린트 | Canvas/WebGL 해시 | 매우 어려움 (실 GPU 필요) |
| 행동 패턴 | 마우스 궤적, 키 입력 타이밍 | 어려움 (자연스러운 패턴 시뮬레이션) |
| JS 실행 환경 | navigator.webdriver, 콘솔 변조 | 가능 (CDP 패치, undetected-chromedriver) |
| 이벤트 신뢰도 | event.isTrusted | 어려움 (OS 레벨 입력 필요) |
| 시간 기반 토큰 | 매 요청마다 다른 암호화 키 | 가능 (브라우저 자동화로 토큰 획득) |
| 세션 / 쿠키 | 경로별 격리, 짧은 만료 | 가능 (재취득 자동화) |
| 응답 구조 | 핵심 데이터를 별도 암호화 API 분리 | 회피 어려움 |

---

## 2. Canvas/WebGL 핑거프린트 — 가장 높은 벽

```javascript
// 사이트 측 코드 (사이트가 사용자 브라우저에서 실행)
function fingerprintCanvas() {
    var canvas = document.createElement('canvas');
    var ctx = canvas.getContext('2d');
    ctx.fillText('FingerPrint', 4, 17);
    return hash(canvas.toDataURL());
}
```

**왜 강한가**:
- 같은 텍스트 그려도 GPU/드라이버/안티앨리어싱 설정이 다르면 미세하게 다른 픽셀 → 다른 해시.
- Headless 브라우저는 SwiftShader 같은 소프트웨어 렌더러 → 알려진 해시 패턴 → 블랙리스트.

**Python 측 대응 한계**:
- Playwright/Selenium 으로 실 브라우저 띄워도 SwiftShader 신호 노출.
- xvfb (가상 디스플레이) → 도리어 감지 (xvfb 특유의 해상도/렌더러).
- 실제 GPU + 실제 디스플레이가 있어야 함 → 자동화 비용 ↑.

---

## 3. 이벤트 isTrusted — 자동화의 본질적 한계

```javascript
element.addEventListener('click', function(e) {
    if (!e.isTrusted) {
        return;  // JS 발생 이벤트 무시
    }
});
```

| 클릭 방식 | isTrusted | 결과 |
|----------|-----------|------|
| 사람이 마우스 클릭 | true | 정상 |
| `el.click()` (JS) | false | 차단 |
| `dispatchEvent(new MouseEvent())` | false | 차단 |
| Selenium ActionChains (CDP) | true | 통과 (단 행동 패턴에서 차단 가능) |
| pyautogui (OS 레벨) | true | 통과 |

**결론**: `isTrusted` 만으로는 자동화가 막히지 않음. CDP 또는 OS 레벨 입력이면 우회. 그 다음 행동 분석이 진짜 벽.

---

## 4. 행동 패턴 — 마우스/키 입력 타이밍

탐지 측 신호:
- 마우스 이동 없이 클릭만 → 봇.
- 키 입력 간격이 완벽히 일정 → 봇.
- 페이지 로드 후 1초 안에 클릭 → 봇.
- Ctrl+V 붙여넣기 → 약한 봇 의심.

Python 자동화 코드의 흔한 흔적:
```python
# 단순 click — 흔적 명확
await page.click("#search-button")

# 사람처럼 보이게 (여전히 부족)
await page.mouse.move(100, 200)
await asyncio.sleep(0.3)
await page.mouse.move(120, 220, steps=10)
await page.mouse.click(120, 220)
```

→ steps + sleep 조합도 통계적으로 detect 가능 (간격 분포가 사람과 다름).

---

## 5. navigator.webdriver

```python
# Selenium / Playwright 가 자동으로 설정
navigator.webdriver  // → true
```

자동화 도구가 이 플래그를 노출. 대응:
```python
# Playwright
await page.add_init_script("Object.defineProperty(navigator, 'webdriver', {get: () => false})")

# undetected-chromedriver — selenium의 패치 버전
```

→ 1단계 우회. 다른 신호로 또 잡힘.

---

## 6. 요청 암호화 — dynaPath 패턴

```
원본 요청:
  URL:  /api/search?method=searchTM
  Body: queryText=4020260003987

       ↓ JS 측에서 암호화 ↓

암호화된 요청:
  URL:  /api/encrypted/Z4XMLKP9bYY9QcNN3WZHBN.../4B8MBDJB3OBDM2P
  Body: xw_=3FEF3IN8834CFN7BAFEOBLB66D894C94CA8FCAD9C5JA5AN4FIC8E
```

암호화 입력:
- 원본 URL + 파라미터.
- 디바이스 핑거프린트.
- 시간 기반 키 (매 요청 변경).
- 세션 상수.

**Python 직접 호출 시도의 결과**:
- 정확한 암호화 알고리즘을 역공학해도 시간 키 + 핑거프린트가 유효해야 함.
- 핑거프린트 없이 보내면 서버 측에서 즉시 거부.

→ 결국 **실 브라우저 자동화** + 핑거프린트 보존이 거의 유일한 방법. Python 만으로는 안 됨.

---

## 7. 응답 구조 분리 — 데이터 조각화

```json
// 검색 API (공개)
{
  "GD": "화장품|향수|메이크업 화장품",
  "SC": "G1201|G1301|S120907|S128302"
}
// 어떤 상품이 어떤 코드에 매핑되는지 알 수 없음

// 상세 API (암호화 필수)
{
  "tb_KT15List": [
    ["03", null, "화장품", null, "G1201,S120907,S128302", "0", ""],
    ["03", null, "샴푸", null, "G1201,G1301", "0", ""]
  ]
}
```

**의도**:
- 공개 API 만 긁어가는 크롤러는 **불완전한 데이터** 만 가져감.
- 가치 있는 매핑 정보는 암호화 API 만 제공.
- 즉 데이터 가치 자체로 차별화.

---

## 8. Python 분석가가 알아야 할 도구들

### 8.1 Playwright/Selenium
- 실 브라우저 자동화. 단 webdriver/headless 신호 주의.
- 권장: `undetected-chromedriver` 또는 Playwright + stealth 패치.

### 8.2 mitmproxy
- HTTPS 트래픽 인터셉트 → 암호화된 요청/응답 관찰.
- "이 요청에 어떤 데이터가 담기는가" 파악.

### 8.3 Chrome DevTools (수동)
- Network 탭 → Initiator 추적 → JS 호출 체인 역추적.
- Sources 탭 → 난독화된 JS 의 break point.
- 안티 디버깅이 작동하면 무한 debugger 문 → 우회 패치 필요.

### 8.4 패킷 분석
- mitmproxy 가 안 되면 wireshark.
- HTTP/2 + TLS 1.3 환경은 분석 어려움.

---

## 9. 정당한 사용 vs 회색 영역

위 분석은 **방어를 이해하기 위한 학습** 목적. 운영 사이트의 ToS 를 위반하는 자동화는 법적 문제 가능.

정당한 케이스:
- 자기 회사 데이터 백업/마이그레이션 (외부 검색 사이트 가 아닌 자사 시스템).
- 보안 테스트 (서면 권한).
- 학술 연구 (Robots.txt 준수, 부하 최소화).

회피해야 할 케이스:
- 경쟁사 데이터 무단 수집.
- DOS 유발할 만큼의 부하.

---

## 10. 응용 포인트

- 자기 시스템의 anti-bot 설계: **단일 시그널에 의존하지 않는 다층 방어** (9 레이어).
- "행동 패턴 + 핑거프린트" 가 가장 강력. 단순 webdriver 체크는 1차 방어.
- 데이터 가치 자체를 차별화 (공개 API 는 일부, 핵심은 암호화).
- 봇 분석 자체를 자동화하려면 실 브라우저 + 실 GPU + OS 레벨 입력 모두 필요.
- Python 자동화는 **인증된 자동화**, **개인 자동화** 영역에 집중. 적대적 크롤링은 비용 ↑↑.
