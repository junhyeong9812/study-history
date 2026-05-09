# Sentinel Value Validation

> **정식 패턴명**: Sentinel Value (일반 CS 용어), Null Object 변형, Special Case Object (Fowler)
> **분류**: Protocol Design Pattern, Validation Pattern, API Design Pattern
> **기원**:
>   - **C 언어의 `NULL` 포인터** — 원초 sentinel (1972~)
>   - **C 문자열 종료 `'\0'`** — 배열 끝 marker
>   - **Python `None`, Java `null`** — 부재 표현
>   - **Fowler, Martin. "Special Case" (martinfowler.com 2002)** — OOP 에서의 sentinel
>   - **HTTP `0.0.0.0`, IPv6 `::`** — "bind to all interfaces" sentinel
>   - **GraphQL `"*"` wildcard**, SQL `NULL` 3-valued logic
>   - **COBOL `HIGH-VALUES` / `LOW-VALUES`** — 경계값 sentinel
>
> **본 프로젝트 기여**: WS 프로토콜에서 field-level sentinel(`"__self__"`) 과 validation 지점의 **지식 공유 실패** 로 발생한 버그를 계기로, sentinel 도입 시 모든 validation 지점이 이를 인지해야 한다는 규칙 명시.

---

## 1. Sentinel Value 기본 정의

### 1.1 정의
> **특수한 의미를 표현하는 예약 값.** 자료형 내의 값 중 "일반 값이 아닌, 의미 표지(marker) 역할" 을 하는 것.

### 1.2 분류

| 유형 | 예시 | 특징 |
|------|------|------|
| **타입 내 예약 값** | `-1` (음수 불가 정수에서), `0` (문자열 종료 null byte) | 본래 도메인의 일부였던 값을 예약 |
| **out-of-band 값** | Python `None`, Java `null` | 타입 외부의 특수 객체 |
| **문자열 리터럴** | `"__self__"`, `"ALL"`, `"default"` | 도메인에 없는 관례적 문자열 |
| **Special Case Object** | `NullObject`, `EmptyList` | OOP 인스턴스로 구현 |

### 1.3 장점
- **프로토콜 단순화**: 필드를 optional 로 만들지 않고도 "의미상 부재" 표현
- **일관된 스키마**: 모든 메시지가 같은 형태 유지 → 파싱 단순
- **명시성**: `"__self__"` 같은 리터럴은 눈에 잘 띔

### 1.4 위험
- **Validation 지점이 sentinel 을 모르면 거부/오작동**
- **Sentinel 값이 실제 도메인 값과 충돌** (예: `"null"` 이라는 실제 username 과 sentinel `"null"` 구분 불가)
- **Nullable + sentinel 동시 도입**: 의미 중복, 혼란

---

## 2. 본 프로젝트의 Sentinel: `"__self__"`

### 2.1 프로토콜 맥락
orchestration ↔ collector WS 메시지는 모두 공통 스키마:
```json
{
  "action": "string",
  "country": "string (kr/us/eu/jp/ch)",
  "params": {...},
  "request_id": "string"
}
```

### 2.2 문제: collector 자기 자신 대상 명령
`system.redeploy_collector`, `system.journal_start` 등은 **국가 개념이 없음** (collector 는 5개국을 관찰하는 에이전트지 국가에 속하지 않음). 그러나 country 필드를 비울 수 없음 (스키마 불변성 유지).

### 2.3 해결: `"__self__"` sentinel
```json
{
  "action": "system.redeploy_collector",
  "country": "__self__",       ← sentinel
  "params": {"username": "jun", "token": "<gitlab-token>..."}
}
```

### 2.4 왜 `"__self__"` 인가?
- **이중 언더스코어**: Python 관례(`__init__`, `__main__`, `dunder`) 에서 "시스템 예약" 의미
- **도메인 값과 충돌 없음**: 국가 코드 `kr/us/eu/jp/ch` 2자리와 완전 구분
- **self-referential 의미** 명확

### 2.5 대안 평가
| 대안 | 평가 |
|------|------|
| `country: null` | nullable 타입 추가. 다른 메시지도 null 허용 판단 필요 |
| `country: ""` | 빈 문자열도 sentinel. 의미 덜 명확 |
| `country: "*"` | wildcard 관례적이나 "모든 국가" 로 오해 가능 |
| `country` 필드 제거 | 스키마 2종 관리 필요 |
| `target: "self"` vs `country: "kr"` | 별도 필드. 깔끔하지만 모든 메시지에 target 추가 필요 |

---

## 3. 버그 — Validation 지점과 Sentinel 간 지식 불일치

### 3.1 버그 코드 (2026-04-20 수정 전)
```python
async def handle_command(msg):
    action = msg.get("action", "")
    country = msg.get("country", "")

    # ⚠ country 검증 먼저
    server = _find_server(country)
    if not server:
        return _error_response(request_id, action, country,
                               f"Unknown country: {country}", start)

    base = server.migration_api

    # 여러 접두사 분기...
    if action.startswith("env."):       ...
    if action.startswith("git."):       ...
    if action.startswith("container."): ...

    # ⚠ system.* 분기가 검증 뒤에 위치
    if action.startswith("system."):
        return await _handle_system_command(action, params, request_id, start)
```

### 3.2 `_find_server` 구현
```python
def _find_server(country: str):
    for s in SERVERS:
        if s.country == country:
            return s
    return None
```

`SERVERS` 는 `[ServerConfig(country="kr"), ServerConfig(country="us"), ...]` 5개. `"__self__"` 는 일치 없음 → `None`.

### 3.3 실제 로그
```
[INFO] Relaying command: action=system.redeploy_collector country=__self__
[INFO] Command response sent: system.redeploy_collector → error
```
`_find_server` 에서 탈락 → `"Unknown country: __self__"` 에러 반환. `_handle_system_command` 에 도달조차 못 함.

### 3.4 본질
> **Validation 함수(`_find_server`) 는 일반 국가만 알고, sentinel `"__self__"` 을 인지하지 못했다.** 프로토콜 설계자는 sentinel 을 도입했지만 검증 지점을 업데이트하지 않음.

---

## 4. 수정: 3가지 선택지

### 4.1 Option A — Sentinel 경로를 Validation 이전으로 분기 (채택)
```python
async def handle_command(msg):
    action = msg.get("action", "")
    country = msg.get("country", "")

    # 1순위: sentinel 경로 (country 검증 건너뜀)
    if action.startswith("system."):
        return await _handle_system_command(action, params, request_id, start)

    # 2순위: 일반 검증
    server = _find_server(country)
    if not server:
        return _error_response(..., f"Unknown country: {country}", start)

    # 3순위: 다른 접두사 분기
    if action.startswith("env."):       ...
    if action.startswith("git."):       ...
    if action.startswith("container."): ...
```

**장점**: 명시적, 간단. sentinel 경로가 코드 읽을 때 바로 보임.
**단점**: action 문자열 기반 분기 순서에 의존. 주석 필수.

### 4.2 Option B — Validation 함수가 Sentinel 인지
```python
def _find_server(country: str):
    if country == "__self__":
        return _SELF_SENTINEL_SERVER  # 더미 객체
    for s in SERVERS:
        if s.country == country:
            return s
    return None
```

**장점**: 호출부 수정 불필요.
**단점**:
- 더미 객체(`_SELF_SENTINEL_SERVER`) 의 `migration_api` 속성 등은 의미 없음 → 에러 가능
- Sentinel 처리가 validation 함수 내에 숨어 가독성 떨어짐
- `_handle_system_command` 가 country 인자 안 받는데 위 로직은 country 를 계속 들고 감

### 4.3 Option C — 별도 스키마
```json
// 일반 메시지
{"action": "migration.start", "country": "kr", "params": {...}}

// 시스템 메시지
{"action": "system.redeploy_collector", "params": {...}}
// country 필드 없음
```

**장점**: 타입 레벨로 깔끔한 분리.
**단점**:
- 클라이언트/서버 양측 분기 로직
- 기존 스키마 호환성 깨짐 (필드 optional 필요)
- 마이그레이션 비용 큼

### 4.4 선택: Option A
최소 변경 + 명시적 의도. 5줄 이동으로 해결.

---

## 5. 함수 시그니처로 의도 드러내기

### 5.1 `_handle_system_command` 시그니처
```python
async def _handle_system_command(action, params, request_id, start_time):
    # country 파라미터 없음!
```

### 5.2 다른 핸들러와 비교
```python
async def _handle_env_command(action, country, params, request_id, start_time):
async def _handle_git_command(action, country, params, request_id, start_time):
async def _handle_docker_command(action, country, params, request_id, start_time):
async def _handle_system_command(action, params, request_id, start_time):   # ← country 없음
```

### 5.3 의미
**함수 시그니처 자체가 "이 명령은 country 에 관련 없다"** 를 선언. 주석보다 강력. IDE autocompletion 에도 노출.

### 5.4 응답 생성 시
`_handle_system_command` 는 응답 dict 의 `country` 필드에 `"__self__"` 를 하드코딩:
```python
return _success_response(
    request_id, action, "__self__", result,     # ← sentinel 하드코딩
    200 if result.get("status") == "success" else 500, elapsed,
)
```
스키마 일관성 유지.

---

## 6. Python 문법 관련

### 6.1 `str.startswith(prefix)` — 접두사 매칭
- **시그니처**: `str.startswith(prefix, start=0, end=len)` → bool
- **Tuple 전달 가능**: `action.startswith(("env.", "git.", "container."))` 로 다중 접두사 한 번에
- **여기서**: 단일 접두사 `"system."` 만 체크

### 6.2 `dict.get(key, default)` — 안전 조회
- **KeyError 회피**: 키 부재 시 default 리턴
- **`msg.get("action", "")`**: 빈 문자열 default → `startswith("system.")` 이 False 로 자연스럽게 분기 실패

### 6.3 `async def` + `await` — 코루틴
- **async 함수 정의**: `async def func(): ...` 은 호출 시 coroutine 객체 반환 (실행 안 됨)
- **실행**: `await func()` 또는 `asyncio.create_task(func())`
- **context**: 동일 이벤트 루프 내에서만 await 가능

---

## 7. 일반 원리

### 7.1 Sentinel 도입 체크리스트
Sentinel 값을 도입할 때 확인해야 할 것들:
1. [ ] Sentinel 값 정의 (무엇이 sentinel 인가)
2. [ ] 도메인 값과 충돌 없음 증명
3. [ ] 모든 validation 지점에서 sentinel 인지 확인
4. [ ] 모든 핸들러에서 sentinel 분기 구현
5. [ ] 프로토콜 문서에 sentinel 의미 명시
6. [ ] 로그/모니터링에서 sentinel 구분 가능
7. [ ] 테스트: sentinel 경로가 잘 동작하는지

### 7.2 본 프로젝트에서 빠진 것
위 체크리스트 중 **3번(validation 지점 업데이트)** 이 누락. 그 결과 프로토콜 레벨에서는 완벽한데 구현이 깨진 상태로 배포.

---

## 8. 유사 사례들

### 8.1 `0.0.0.0` — "모든 인터페이스" Sentinel
- bind 시 특정 주소 대신 모든 interface 의미
- 검증 함수가 `0.0.0.0` 을 "정상 IP" 로 취급하지 않으면 거부 가능

### 8.2 Unix UID `0` — root
- UID 0 은 특별한 권한 (super user)
- 권한 검증 코드가 "일반 사용자" 만 고려하면 root 를 어떻게 처리할지 애매

### 8.3 SQL `NULL` — 3-valued logic
- `WHERE col = NULL` 은 false (항상)
- `IS NULL` 전용 연산자 필요
- validation 프레임워크가 NULL 을 어떻게 다룰지 규칙 필요

### 8.4 HTTP `Content-Length: 0` vs missing
- 명시적 0 과 헤더 부재는 의미 다름
- 스펙에 따라 body 존재 여부 판단이 복잡

### 8.5 GraphQL `null` vs `undefined`
- `null` = 의도적 부재
- 필드 자체 부재 = 클라이언트가 요청 안 함
- 두 의미를 구분하는 서버 구현 필요

---

## 9. 안티패턴

### 9.1 Magic Number 로서의 Sentinel
```python
if count == -1:   # sentinel 의미: "무제한"
    ...
```
- 값의 의미가 코드에 없으면 이해 불가
- 수정: 상수 선언 `UNLIMITED = -1` 또는 별도 필드

### 9.2 도메인과 겹치는 Sentinel
```python
if username == "null":  # sentinel?
    ...
```
- 실제 username 이 "null" 인 사용자와 충돌
- 수정: 도메인에 없는 값 선택 (e.g., `"\0"` 또는 `None`)

### 9.3 Nullable + Sentinel 동시
```json
{"country": null}
{"country": "__self__"}
{"country": "kr"}
```
세 의미가 무엇인지 각 API 마다 해석. 혼란.

---

## 10. 관련 자료

### 10.1 참고
- **Fowler, Martin. *Patterns of Enterprise Application Architecture* (2002)** — Special Case
- **C2 Wiki — "Sentinel Value"** (http://wiki.c2.com/?SentinelValue)
- **Date, C. J. *Database in Depth* (2005)** — SQL NULL 논의
- **RFC 3986 (URI)** — `0.0.0.0`, `::` 같은 sentinel 주소

### 10.2 언어별 관례
| 언어/프로토콜 | Sentinel 관례 |
|-------------|--------------|
| Python | `None`, `...` (Ellipsis), `__dunder__` |
| Java | `null`, `Optional.empty()` |
| Rust | `None`, `()` (unit), `!` (never type) |
| Go | `nil`, zero values |
| JS | `null`, `undefined` (두 개의 sentinel!) |
| JSON | `null` (단일) |
| REST API | `_self`, `_all`, `*` |

---

## 11. 본 프로젝트 적용 위치

- [../collector-agent/command-relay.md](../collector-agent/command-relay.md) — action 분기 전체 구조
- [../collector-agent/self-redeploy.md](../collector-agent/self-redeploy.md) — `__self__` 사용처
- [../operations/incident-history.md](../operations/incident-history.md#2026-04-20--systemredeploy_collector-항상-실패) — 이번 버그 상세
