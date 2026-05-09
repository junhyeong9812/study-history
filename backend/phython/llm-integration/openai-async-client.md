# OpenAI AsyncClient — 형태소 분리/음역 작업

> 트레이드마크명 형태소 분리에 LLM 을 보조 도구로 사용하는 패턴.
> JSON 모드, 시스템 프롬프트, lazy 초기화.

---

## 0. 분석 대상 코드

```python
# monitoring/app/services/llm_service.py
import json
import logging
from typing import Any, Dict, List
from openai import AsyncOpenAI
from app.config import settings

_client: AsyncOpenAI | None = None

SYSTEM_PROMPT = """영어 상표명을 한글로 음역(transliteration)하세요.
각 상표명에 대해 JSON 객체를 반환하세요:
{
  "input": "원본 영어",
  "korean": "주요 한글 표기",
  "alternatives": ["대안 표기1", "대안 표기2"],
  "morphemes": ["형태소1", "형태소2"]
}
규칙:
- 한국에서 실제 사용되는 표기를 우선
- 대안이 있으면 alternatives에 포함
- 한글 형태소 단위로 분리 (예: 삼성 → ["삼", "성"])
- 여러 상표명이 주어지면 JSON 배열로 반환"""


def _get_client() -> AsyncOpenAI | None:
    global _client
    if not settings.OPENAI_API_KEY:
        return None
    if _client is None:
        _client = AsyncOpenAI(api_key=settings.OPENAI_API_KEY)
    return _client


async def analyze_trademarks(names: List[str]) -> List[Dict[str, Any]]:
    """LLM으로 상표명 형태소 분리"""
    client = _get_client()
    if client is None:
        raise RuntimeError("OPENAI_API_KEY 미설정")

    user_msg = "다음 영어 상표명들을 한글로 음역하세요:\n" + "\n".join(f"- {n}" for n in names)

    response = await client.chat.completions.create(
        model=settings.OPENAI_MODEL,
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": user_msg},
        ],
        temperature=0.3,
        response_format={"type": "json_object"},
    )

    content = response.choices[0].message.content
    parsed = json.loads(content)

    if isinstance(parsed, dict):
        if "results" in parsed:
            results = parsed["results"]
        else:
            results = [parsed]
    elif isinstance(parsed, list):
        results = parsed
    else:
        results = []

    entries = []
    for r in results:
        entries.append({
            "productNameEng": r.get("input", "").lower().strip(),
            "modelResult": r.get("korean", ""),
            "alternatives": r.get("alternatives", []),
            "morphemes": r.get("morphemes", []),
            "source": "llm",
        })

    return entries
```

---

## 1. AsyncOpenAI 의 정체

- `openai` 패키지의 비동기 클라이언트 (1.x+).
- 동기 `OpenAI` 와 동일 API + `await` 추가.
- 내부적으로 httpx 기반.

설치: `pip install openai`

---

## 2. Lazy 초기화 패턴

```python
_client: AsyncOpenAI | None = None

def _get_client() -> AsyncOpenAI | None:
    global _client
    if not settings.OPENAI_API_KEY:
        return None
    if _client is None:
        _client = AsyncOpenAI(api_key=settings.OPENAI_API_KEY)
    return _client
```

**왜 lazy?**
- API 키가 안 설정된 환경에서도 앱은 떠야 함.
- 첫 호출 시점까지 client 생성 미룸 → 시작 시 빠르고 가벼움.
- `None` 반환을 호출 측이 체크해서 graceful degrade.

---

## 3. JSON 응답 모드 — `response_format`

```python
response = await client.chat.completions.create(
    model=settings.OPENAI_MODEL,
    messages=[...],
    temperature=0.3,
    response_format={"type": "json_object"},
)
```

### `response_format={"type": "json_object"}`
- 응답이 반드시 valid JSON.
- 시스템 프롬프트에 "JSON" 단어가 있어야 동작.
- 자유 텍스트 응답에 비해 파싱 안정.

### 그래도 검증은 필요
- LLM 이 시스템 프롬프트의 JSON 스키마를 약간 벗어날 수 있음.
- 위 코드의 후처리:
  ```python
  if isinstance(parsed, dict):
      if "results" in parsed:
          results = parsed["results"]
      else:
          results = [parsed]
  elif isinstance(parsed, list):
      results = parsed
  ```
  → "결과가 dict 일 수도, list 일 수도, dict.results 일 수도" 모두 허용.

### 더 강한 보장: structured outputs (function calling / tool 모드)
```python
response = await client.chat.completions.create(
    model="gpt-4o",
    messages=[...],
    tools=[{
        "type": "function",
        "function": {
            "name": "analyze_trademark",
            "parameters": {
                "type": "object",
                "properties": {
                    "korean": {"type": "string"},
                    "alternatives": {"type": "array", "items": {"type": "string"}},
                },
                "required": ["korean"]
            }
        }
    }],
    tool_choice={"type": "function", "function": {"name": "analyze_trademark"}}
)
```

→ 스키마를 강제. 위 코드는 단순 JSON 모드 사용.

---

## 4. temperature 의 의미

```python
temperature=0.3
```

- `0.0`: 결정적. 같은 입력 → 같은 출력.
- `0.7~1.0`: 다양성/창의성.
- `0.3`: 약간의 변동성 + 안정성. 음역 같은 작업에 적합.

음역 작업의 특성:
- 정답이 어느 정도 정해져 있음 (Samsung → 삼성).
- 너무 결정적이면 자주 쓰이지 않는 표기를 놓칠 수 있음 → 살짝 열어둠.

---

## 5. 메시지 역할 — system / user / assistant

```python
messages=[
    {"role": "system", "content": SYSTEM_PROMPT},
    {"role": "user", "content": user_msg},
]
```

| 역할 | 용도 |
|------|------|
| `system` | 모델 행동 규칙. 일관된 형식 강제 |
| `user` | 실제 입력 |
| `assistant` | 이전 응답 (multi-turn 대화 시) |
| `tool` | tool call 결과 |

### 시스템 프롬프트 설계
- **출력 형식 고정**: 위 코드는 JSON 스키마 명시.
- **규칙 나열**: "한국에서 실제 사용되는 표기 우선" 등.
- **예시 포함**: "삼성 → ["삼","성"]" 처럼 1-shot.

좋은 시스템 프롬프트의 특징:
- 짧고 명확.
- 출력 형식이 모호하지 않음.
- 도메인 규칙 명시.

---

## 6. 응답 파싱

```python
content = response.choices[0].message.content
parsed = json.loads(content)
```

### `response.choices`
- 모델이 여러 응답 후보 (`n` 파라미터로 조정) 를 만들 수 있음. 보통 1.
- `.choices[0].message.content` 가 메인 텍스트.

### `usage` 속성 (토큰 사용량)
```python
print(response.usage.prompt_tokens, response.usage.completion_tokens, response.usage.total_tokens)
```

→ 비용 계산 / 모니터링.

---

## 7. 비용/속도 고려

### 모델 선택
- `gpt-4o-mini`: 빠르고 저렴. 단순 음역에 충분.
- `gpt-4o`: 정확도 ↑, 비용 ↑.
- `gpt-3.5-turbo`: 매우 저렴, 정확도 ↓.

### 배치 호출
위 코드는 한 번에 여러 트레이드마크 처리:
```python
user_msg = "다음 영어 상표명들을 한글로 음역하세요:\n" + "\n".join(f"- {n}" for n in names)
```

→ N개를 1번 호출 → 토큰 낭비 줄임 + 응답 시간 단축.
단점: 한 응답에 너무 많이 넣으면 max_tokens 초과 또는 품질 저하.

### Batch API
- OpenAI 의 `/v1/batches` 엔드포인트 — 24h 안에 처리 + 50% 할인.
- 실시간성이 필요 없으면 선택지.

---

## 8. 안전성 — API 키, 비용 폭주 방지

### API 키
- 환경변수 / 시크릿 관리자에서 로드.
- 코드/git 에 하드코딩 금지.

### 비용 폭주 방지
- OpenAI 대시보드의 monthly limit 설정.
- 앱 측에서도 횟수 limiter (`asyncio.Semaphore` 또는 token bucket).
- 큰 요청 (long context) 은 사전 토큰 추정.

### 토큰 추정
```python
import tiktoken
enc = tiktoken.encoding_for_model("gpt-4o")
tokens = enc.encode("hello world")
print(len(tokens))  # 2
```

→ 호출 전 입력 토큰 추정해서 budget 체크.

---

## 9. Anthropic Claude / 다른 공급자

위 코드는 OpenAI. Anthropic Claude 도 거의 동일한 패턴:

```python
from anthropic import AsyncAnthropic

client = AsyncAnthropic(api_key=...)
response = await client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}],
)
```

차이:
- system 은 `messages` 가 아닌 별도 `system="..."` 인자.
- `max_tokens` 가 필수.
- response.content[0].text 로 본문 접근.

Multi-provider 설계 시 thin wrapper 한 겹 두는 게 안정적.

---

## 10. 응용 포인트

- 비동기 환경은 `AsyncOpenAI`.
- API 키 미설정 시 graceful degrade 위해 lazy 초기화 + None 반환.
- JSON 형식 강제는 `response_format={"type": "json_object"}` 또는 tool_use.
- 시스템 프롬프트에 출력 스키마 + 예시 + 규칙 명시.
- 결정적 결과는 temperature=0, 약간의 다양성은 0.3 정도.
- 배치 호출로 토큰/시간 절약 (단 길이 제한 주의).
- 비용 모니터링은 `response.usage` + 외부 limit.
