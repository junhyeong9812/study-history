# Session ID by Resource Lifetime

> **정식 패턴명**: Generation Counter / Epoch ID (일반화), Correlation ID Variant
> **분류**: Stateful Tracking Pattern, Distributed Systems Identity Pattern
> **도입**: 2026-04-20, 본 프로젝트의 에이전트 재배포 완료 감지
> **기원**:
>   - **Jepsen.io / Aphyr** 의 분산 시스템 리소스 교체 감지 논의 (2013~)
>   - **Raft consensus** 의 `term` 개념 (리더 교체 감지)
>   - **PostgreSQL** 의 `backend_pid` + `backend_start` 기반 세션 식별
>   - **Kubernetes** 의 `Pod.metadata.uid` 와 `ResourceVersion` (재생성 감지)
>   - **etcd lease ID** — 세션 생명주기 종속 자원

---

## 1. 문제 정의 (Problem Statement)

### 1.1 증상
분산 시스템에서 "원격 리소스가 재기동됐는지" 를 감지하고 싶을 때, **상태 스냅샷(snapshot) 기반 polling** 은 짧은 다운타임을 놓친다.

### 1.2 구체 사례
- UI 가 `/api/status` 를 2500ms 간격 polling
- 원격 agent 가 systemctl restart 로 ~1초 다운 → 재기동
- polling cycle_N: `connected=true`, cycle_{N+1}: `connected=true` (재기동 후 이미 복구)
- **다운 이벤트 완전 상실**. UI 는 아무 일도 안 일어난 것으로 인지

### 1.3 본질
**Sampling Theorem 위반**. 신호(다운타임 펄스) 가 샘플링 주기의 Nyquist 한계(2배) 보다 짧으면 원복 불가능. 간격을 줄여도 완벽 보장 안 됨.

---

## 2. 해법 — Monotonic Generation Identifier

### 2.1 핵심 원칙

> **리소스의 각 인스턴스(재생성마다) 에 고유 ID 를 발급하고, 클라이언트가 "이전 ID" 를 기억하여 현재 ID 와 비교한다. 교체 여부는 sampling rate 무관하게 영구 식별 가능.**

### 2.2 상태 모델

| 시점 | agent session_id | orchestration 저장 | UI 캡처 |
|-----|-----------------|-------------------|--------|
| t0 agent 기동 #1 | `"abc123"` | `"abc123"` | - |
| t1 UI 재배포 버튼 | `"abc123"` | `"abc123"` | `beforeSession="abc123"` |
| t2 agent 재기동 중 | - | `null` (disconnect) | - |
| t3 agent 기동 #2 | `"xyz789"` | `"xyz789"` | - |
| t4 UI polling | `"xyz789"` | `"xyz789"` | current != beforeSession → 완료 |

**t4 가 t3 직후든 1시간 후든 결과 동일**. 상태 변화가 "영속 값" 이라 관찰 시점 무관.

### 2.3 왜 "generation" 인가
Raft/ZooKeeper 용어로 "term" 또는 "generation" 은 리더가 바뀔 때마다 증가하는 monotonic counter. 구 리더의 메시지가 늦게 도착해도 term 이 과거값이면 무시. 같은 원리로 구버전 리소스의 응답이 섞여도 식별 가능.

---

## 3. 구현 — Python 측 (collector-agent)

### 3.1 전체 파일 (`transport/ws_client.py`)
```python
import asyncio
import json
import os
import uuid
from typing import Callable, Awaitable, Optional

import websockets
from websockets.exceptions import ConnectionClosed

from config import CONTROL_SERVER_WS
from logger import setup_logger

logger = setup_logger("collector.ws")

_active_ws = None
_session_id: Optional[str] = None


def get_ws():
    """현재 활성 WebSocket 객체 반환 (다른 모듈이 스트리밍 등에 사용)."""
    return _active_ws


def get_session_id() -> Optional[str]:
    """현재 WS 세션 ID 반환. 연결 전이면 None."""
    return _session_id


async def ws_listen(command_handler: Callable[[dict], Awaitable[dict]]):
    backoff = 3
    max_backoff = 30

    while True:
        try:
            async with websockets.connect(
                CONTROL_SERVER_WS,
                ping_interval=None,
                ping_timeout=None,
            ) as ws:
                global _active_ws, _session_id
                _active_ws = ws
                _session_id = uuid.uuid4().hex
                pid = os.getpid()
                logger.info(
                    f"WebSocket connected (session={_session_id[:8]}, pid={pid})"
                )
                backoff = 3

                await ws.send(json.dumps({
                    "type": "agent_hello",
                    "agent": "collector",
                    "session_id": _session_id,
                    "pid": pid,
                }))
                ...
        except ConnectionClosed:
            ...
        await asyncio.sleep(backoff)
        backoff = min(backoff * 2, max_backoff)
```

### 3.2 메서드별 동작 + 언어 문법

#### 3.2.1 `uuid.uuid4().hex` — 세션 식별자 생성
- **동작**: 128비트 random UUID (RFC 4122 variant 1, version 4) 생성 → 32자 hex 문자열로 변환 (`-` 하이픈 없음).
- **왜 uuid4?**: 시스템 시계/MAC 주소 등 외부 입력 없이 pseudo-random 만으로 생성. 같은 프로세스에서 동시 발급도 충돌 실질 불가 (2^122 sample space).
- **왜 hex 형태?**: JSON 전송 용이 + URL-safe + 비교 가능. `str(uuid4())` (하이픈 포함 36자) 도 동등하지만 관례상 hex 사용.
- **대안 비교**:
  - `secrets.token_hex(16)` — 16바이트 = 32자 hex, 동등한 효과. 암호학적 안전성은 uuid4 보다 약간 강함
  - `time.time_ns()` — 나노초 시각, 단조 증가 보장되지만 클럭 역행 시 중복 가능
  - 시퀀스 정수 — 영속 저장 필요, 프로세스 재기동 시 상태 유지 어려움
- **선택 이유**: stdlib 내장, 관례적, 명확한 "uniqueness only" 의도.

#### 3.2.2 `os.getpid()` — 보조 식별자
- **동작**: 현재 OS 프로세스 ID 정수 반환. POSIX 의 `getpid(2)` 시스템콜 래퍼.
- **용도**: 진단용. session_id 만으로 교체 감지는 충분하지만 "어느 프로세스가 연결했는지" 를 로그에서 연결하기 위해 병기.
- **특성**: 프로세스 수명 동안 불변. 재기동마다 새 PID (kernel 이 재활용할 수는 있으나 근접 시점에는 드묾).

#### 3.2.3 `global _active_ws, _session_id` — 모듈 전역 변경 선언
- **문법**: Python 에서 함수 내부에서 모듈 전역 변수에 **재대입** 하려면 `global` 선언 필수. 읽기는 선언 없이 가능하지만 대입 시 선언 안 하면 로컬 변수로 shadow 됨.
- **왜 전역?**: `get_ws()`, `get_session_id()` 가 **다른 모듈** (예: `relay/docker_control.py` 의 `_stream_lines`) 에서 호출되어 WS send 에 쓰이기 때문. 함수 인자로 전달하는 대신 전역 접근자 제공.
- **비교 — 클래스로 캡슐화했다면?**: `WsClient` 클래스의 인스턴스 속성으로 두면 더 객체지향적. 단 singleton 생성자 필요, import 순서 꼬일 수 있어 모듈 전역이 더 단순.

#### 3.2.4 `async with websockets.connect(...) as ws` — async context manager
- **문법**: `async with` = PEP 492 의 async context manager. `__aenter__` / `__aexit__` 코루틴 호출.
- **동작**: 블록 진입 시 WS handshake 수행, 종료(블록 exit 또는 예외) 시 close frame 송신 + TCP close.
- **`ping_interval=None, ping_timeout=None`**: websockets 라이브러리 기본값은 주기 ping. 본 프로젝트는 orchestration 이 명시적 close 를 보내므로 ping 기반 유지 불필요. None 으로 비활성화하여 false-positive 단절 방지.

#### 3.2.5 `f-string` 로그 (`f"... {_session_id[:8]} ..."`)
- **문법**: PEP 498 literal string interpolation. 컴파일 타임에 AST 로 분해되어 `str.__format__` 호출로 변환. `%` 포맷이나 `str.format()` 보다 빠름.
- **`_session_id[:8]`**: 32자 중 앞 8자만 로그 표시. 전체는 지저분, 앞 8자로 충분히 구분 가능 (충돌 확률 2^32 = 40억 중 1).

#### 3.2.6 `await ws.send(json.dumps({...}))` — dict → JSON 직렬화 + 송신
- **동작**: Python dict 를 JSON 문자열로 직렬화 후 WS 텍스트 프레임으로 전송.
- **dict literal `{"type": ..., "session_id": ..., "pid": ...}`**: 메시지 스키마를 코드에 직접 작성. 별도 dataclass 나 Pydantic 모델로 타입 안전성을 높일 수도 있지만 agent 는 스키마가 단순해 dict 직접 사용.
- **왜 json.dumps?**: WS 는 text/binary 프레임. JSON 은 양측이 이해하는 공통 포맷. MessagePack 등 binary 대안도 가능하나 가독성 우선.

#### 3.2.7 지수 백오프 루프
```python
backoff = 3
...
await asyncio.sleep(backoff)
backoff = min(backoff * 2, max_backoff)
```
- **패턴명**: Exponential Backoff (AWS 분산 시스템 전형 패턴)
- **`min(backoff * 2, max_backoff)`**: 상한 캡. 무한 증가 방지.
- **왜 3 → 30?**: 초기 3초로 짧은 복구 대응, 상한 30초로 idle 상태 CPU 낭비 방지. 실제 운영 관찰 기반 경험값.

---

## 4. 구현 — FastAPI 측 (orchestration)

### 4.1 전체 코드 (`app/api/migration_ws.py`)
```python
import asyncio
import json
import logging
import uuid
from collections import deque
from typing import Optional, Dict, Any

from fastapi import APIRouter, WebSocket, WebSocketDisconnect

logger = logging.getLogger("ws.migration")
router = APIRouter()

_active_ws: Optional[WebSocket] = None
_command_buffer: deque = deque(maxlen=100)
_pending_responses: Dict[str, asyncio.Future] = {}
_connected = False
_agent_session_id: Optional[str] = None
_agent_pid: Optional[int] = None


@router.websocket("/ws/agent")
async def agent_websocket(ws: WebSocket):
    global _active_ws, _connected, _agent_session_id, _agent_pid

    await ws.accept()
    _active_ws = ws
    _connected = True
    logger.info("Agent connected")

    try:
        while True:
            raw = await ws.receive_text()
            try:
                msg = json.loads(raw)
            except json.JSONDecodeError:
                logger.warning(f"Invalid JSON: {raw[:200]}")
                continue

            msg_type = msg.get("type", "")

            if msg_type == "agent_hello":
                _agent_session_id = msg.get("session_id")
                _agent_pid = msg.get("pid")
                logger.info(
                    f"Agent hello: {msg.get('agent')} "
                    f"(session={(_agent_session_id or '')[:8]}, pid={_agent_pid})"
                )
                await ws.send_json({"type": "welcome", "status": "connected"})
                continue
            ...

    except WebSocketDisconnect:
        logger.warning("Agent disconnected")
    finally:
        _active_ws = None
        _connected = False
        _agent_session_id = None
        _agent_pid = None


def get_agent_session() -> Dict[str, Any]:
    """에이전트 현재 WS 상태 + session 정보 반환.

    UI 재배포 완료 감지용. session_id 변경 = 재기동 완료.
    """
    return {
        "connected": _connected,
        "session_id": _agent_session_id,
        "pid": _agent_pid,
    }
```

### 4.2 메서드별 동작 + 언어 문법

#### 4.2.1 `@router.websocket("/ws/agent")` — FastAPI WebSocket 데코레이터
- **동작**: FastAPI(Starlette) 의 WebSocket 핸들러 등록. Starlette 내부적으로 ASGI `websocket` scope 수신 → 코루틴 호출.
- **데코레이터 문법**: PEP 318. `@router.websocket(...)` 는 `agent_websocket = router.websocket("/ws/agent")(agent_websocket)` 의 syntactic sugar.
- **비교 — HTTP 핸들러와 차이**: `@router.get(...)` 은 Response 객체 반환, `@router.websocket(...)` 은 반환값 대신 `await ws.send_*` 로 응답.

#### 4.2.2 `await ws.accept()` — WS handshake 완료
- **동작**: HTTP 101 Switching Protocols 응답 전송. 이후 양방향 프레임 교환 가능.
- **언제 호출?**: `receive_*` / `send_*` 전에 반드시 한 번. 없으면 예외.

#### 4.2.3 `msg.get("type", "")` — Dict 의 안전 조회
- **문법**: `dict.get(key, default)` — 키 부재 시 default 리턴 (KeyError 안 발생).
- **왜 빈 문자열 default?**: 이후 `msg_type == "agent_hello"` 비교가 실패하고 분기 무시. None 이면 추가 None check 필요.

#### 4.2.4 `(_agent_session_id or '')[:8]` — Short-circuit + slice
- **동작**: `_agent_session_id` 가 None 이면 falsy → `or` 가 뒤의 `''` 리턴. 이후 `[:8]` 는 빈 문자열의 0~8 슬라이스 = 빈 문자열.
- **왜?**: None 인 채로 `[:8]` 하면 `TypeError: 'NoneType' object is not subscriptable`. 방어적 처리.

#### 4.2.5 `try / except / finally` — 자원 정리 패턴
- **문법**: `finally` 는 **예외 발생 여부 무관** 항상 실행.
- **용도**: WebSocket disconnect 시 (정상 종료든 예외든) 전역 상태를 깨끗이 리셋.
- **왜 중요?**: `_connected=True` 가 남아있으면 UI 가 여전히 연결된 것으로 오인. 반드시 finally 에서 None/False 로 복귀.

#### 4.2.6 `Optional[WebSocket]`, `Dict[str, asyncio.Future]` — PEP 484 타입 힌트
- **문법**: `typing` 모듈의 제네릭. 런타임 강제 없음, IDE/mypy 정적 분석용.
- **`Optional[X]`**: `Union[X, None]` 의 약칭. Python 3.10+ 는 `X | None` 도 가능.
- **`Dict[K, V]`**: 딕셔너리 타입. Python 3.9+ 는 lowercase `dict[K, V]` 도 가능.

#### 4.2.7 `deque(maxlen=100)` — 고정 크기 양방향 큐
- **동작**: `collections.deque`. `maxlen` 초과 시 반대편에서 자동 제거 (ring buffer).
- **왜?**: 명령 이력 최대 100개만 보관. 메모리 무제한 증가 방지.
- **시간 복잡도**: append/popleft O(1), 인덱스 접근 O(n).

#### 4.2.8 `asyncio.Future` — async 응답 대기 프리미티브
- **역할**: Low-level async primitive. `create_future()` 로 생성, `set_result()` / `set_exception()` 으로 완료, `await fut` 로 대기.
- **여기서**: 명령 `request_id` 키로 pending future 저장 → agent 응답 수신 시 `set_result`. `await asyncio.wait_for(fut, timeout=300)` 로 타임아웃 포함 대기.

---

## 5. 구현 — JavaScript 측 (브라우저 UI)

### 5.1 전체 코드 (`app/static/js/deploy.js`)
```js
async function fetchAgentSession() {
    try {
        const r = await fetch('/api/migration/agent-status', {cache: 'no-store'});
        if (!r.ok) return null;
        return await r.json();
    } catch(e) { return null; }
}

async function deployAgentRedeploy() {
    ...
    const beforeSess = await fetchAgentSession();
    const beforeSession = beforeSess?.session_id || null;
    ...
    waitForHealth('agent', st, 90, beforeSession);
}

async function waitForHealth(kind, statusEl, maxSec=90, beforeSession=null) {
    const start = Date.now();
    let wentDown = false;
    const url = kind === 'orch' ? '/api/monitor/health-self' : '/api/migration/agent-status';
    while ((Date.now() - start) < maxSec * 1000) {
        await new Promise(r => setTimeout(r, 2500));
        try {
            const resp = await fetch(url, {cache: 'no-store'});
            if (!resp.ok) { wentDown = true; continue; }
            const d = await resp.json();

            if (kind === 'agent') {
                if (d.connected && beforeSession && d.session_id && d.session_id !== beforeSession) {
                    statusEl.textContent = '✓ 배포 완료 — 서비스 정상';
                    return true;
                }
                if (!beforeSession) {
                    if (!d.connected) { wentDown = true; continue; }
                    if (wentDown) { /* 완료 */ return true; }
                }
                if (!d.connected) wentDown = true;
                continue;
            }
            /* orch 는 기존 wentDown 방식 */
            ...
        } catch(e) { wentDown = true; }
    }
    statusEl.textContent = '⚠ 완료 확인 실패 — journalctl/로그 확인 권장';
    return false;
}
```

### 5.2 JS 문법 + 동작

#### 5.2.1 `beforeSess?.session_id` — Optional Chaining (ES2020)
- **문법**: `obj?.prop` — `obj` 가 `null`/`undefined` 면 전체 식이 `undefined`, 아니면 `obj.prop`.
- **대안**: `beforeSess && beforeSess.session_id`. 동등하지만 chaining 이 간결.
- **여기서**: `fetchAgentSession` 이 null 반환 가능하므로 안전 접근.

#### 5.2.2 `|| null` — 논리 OR default
- **문법**: falsy (undefined/null/0/""/NaN) 면 우측 값.
- **왜 `?? null` 이 아닌 `|| null`?**: `??` 는 null/undefined 에만 대응. 여기서는 빈 문자열도 null 취급하고 싶으므로 `||` 사용. (실질 차이는 없지만 명시적 의도)

#### 5.2.3 `await new Promise(r => setTimeout(r, 2500))` — 비동기 sleep
- **숙어**: JS 에 표준 sleep 없음. `Promise` + `setTimeout` 조합으로 구현.
- **동작**: `Promise` 를 새로 만들고, 그 resolver(`r`)를 `setTimeout` 콜백으로 2500ms 후 호출. `await` 이 resolve 까지 대기.
- **왜 util 함수 안 쓰나?**: inline 이 더 명확. 라이브러리 의존성 없음.

#### 5.2.4 `fetch(url, {cache: 'no-store'})` — 브라우저 캐시 우회
- **옵션**: `cache: 'no-store'` → 브라우저 HTTP cache 미사용 (CDN/Service Worker 캐시도 bypass).
- **왜?**: `/agent-status` 는 매 순간 최신값 필요. 브라우저가 1초 전 응답을 재사용하면 session_id 비교 무의미.

#### 5.2.5 `continue` — 조건 미달 시 다음 polling cycle
- **제어 흐름**: 현재 iteration 종료, while loop 맨 위로 돌아가 조건 재평가 → `await sleep` 부터 재시작.
- **왜 `return false` 아님?**: 아직 타임아웃 안 됐으니 계속 시도.

#### 5.2.6 `Date.now() - start` — 경과 시간 (ms)
- **API**: `Date.now()` = 1970-01-01 UTC 기준 epoch ms. 정수.
- **maxSec * 1000**: 초 → ms 변환.

---

## 6. 대안 설계 비교

### 6.1 polling 간격 축소 (2500ms → 250ms)
- **개선**: 대부분 다운타임 잡힘
- **단점**:
  - 네트워크 트래픽 10배
  - 250ms 보다 짧은 다운타임은 여전히 miss
  - 근본 해결 아님
- **결론**: 임시방편. session_id 패턴이 우월.

### 6.2 Server-side event counter
- **방식**: disconnect 때마다 `_connection_count += 1`. API 가 이 값 반환. UI 는 변화 감지.
- **비교**: session_id 와 본질 동일. 단조 증가 정수라 디버깅 쉬움 ("3회 재기동").
- **단점**: 영속화 안 하면 orch 재기동 시 count 0 으로 리셋 → UI 가 감소 감지하지 못함. session_id (UUID) 는 값 자체가 다르니 영향 없음.

### 6.3 WebSocket 직접 구독 (UI 가 `/ws/agent` 상태 브로드캐스트)
- **방식**: orch 가 agent connect/disconnect 이벤트를 `/ws/dashboard` 로 push. UI 는 수동 polling 없이 실시간 수신.
- **장점**: 가장 반응적. polling 무효.
- **단점**: 메시지 분실/순서 역전 대응 필요. polling + session_id 가 더 단순.

### 6.4 Process PID 만 사용 (uuid 없이)
- **단점**: kernel 이 PID 재활용 가능 (32768 rollover 또는 짧은 시간 내 우연 일치).
- **개선**: `pid + start_time` 조합이 안전. 하지만 이미 uuid 가 있으니 pid 는 진단용.

---

## 7. 한계와 트레이드오프

### 7.1 순단 재연결 = 재기동 오인
- **상황**: 네트워크 깜빡임으로 WS 재연결만 일어나도 session_id 바뀜.
- **영향**: 재배포 요청 없을 때 UI 가 "완료" 로 오인할 수 있지만, 재배포 버튼 context 외에는 polling 하지 않으므로 실제 문제 거의 없음.
- **엄격한 구분 필요 시**: pid 도 함께 비교. "session 바뀜 AND pid 바뀜" = 프로세스 재기동.

### 7.2 첫 사이클 실패
- **시나리오**: 구버전 agent → 새 UI. `fetchAgentSession` 이 `{connected:true}` 만 받음 (session_id 필드 부재). `beforeSession = null` → fallback 방식 (wentDown) → 타이밍 레이스.
- **완화**: 점진적 배포. 첫 배포 이후부터는 session_id 기반 정상.

### 7.3 UUID 충돌
- **확률**: 2^122 sample space. 지구 생명 전체에 걸쳐 우연 충돌 0 에 수렴.
- **실제 위협**: PRNG seed 오염. Python `uuid4` 는 `os.urandom` (CSPRNG) 사용하므로 안전.

---

## 8. 관련 패턴

| 패턴 | 유사점 | 차이점 |
|------|-------|-------|
| **Optimistic Lock (version column)** | 버전 변화로 race 감지 | 리소스 수명이 아닌 레코드 단위 |
| **Correlation ID** | 요청 tracking ID | 세션 전체가 아닌 단일 요청 |
| **ETag / If-None-Match** | 변경 감지 | HTTP 리소스 해시, lifetime 과 무관 |
| **Kubernetes Pod UID** | Pod 재생성마다 새 UID | 완전 동일 개념 (다만 Orchestrator 주입) |
| **Raft Term** | 리더 generation | 동일 개념 + monotonic ordering 포함 |
| **ZooKeeper ZXID** | transaction generation | 더 포괄적 |

---

## 9. 참고 자료

- **RFC 4122** — UUID 표준
- **Nygard, Michael. *Release It!* (2nd ed., 2018)** — Stability Patterns 전반
- **Ongaro, Diego. *In Search of an Understandable Consensus Algorithm (Raft), 2014*** — term 개념
- **Kleppmann, Martin. *Designing Data-Intensive Applications* (2017), 9장** — Distributed System Trouble
- PostgreSQL docs: `pg_stat_activity.backend_start`

---

## 10. 본 프로젝트 적용 위치

- [../collector-agent/self-redeploy.md](../collector-agent/self-redeploy.md) — 전체 시나리오
- [../collector-agent/overview.md](../collector-agent/overview.md) — agent 구조
- [../orchestration/deploy-tab.md](../orchestration/deploy-tab.md) — UI 로직
- [../orchestration/ws-bridge.md](../orchestration/ws-bridge.md) — WS 브릿지
- [../operations/incident-history.md](../operations/incident-history.md#2026-04-20--ui-재배포-완료-감지-오탐)
