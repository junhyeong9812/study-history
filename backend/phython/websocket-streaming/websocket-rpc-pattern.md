# WebSocket 위에 RPC — request_id 기반 응답 매칭

> 양방향 WS 위에서 "명령 보내고 결과 기다리기" 를 동기 함수 호출처럼 쓰는 패턴.
> Future + 타임아웃 + 정리 로직이 핵심.

---

## 0. 분석 대상 코드

서버 측: `send_agent_command` 가 명령을 보내고 응답이 올 때까지 대기.

```python
# orchestration/app/api/migration_ws.py
import asyncio
import uuid
from collections import deque
from typing import Optional, Dict
from fastapi import APIRouter, WebSocket, WebSocketDisconnect

router = APIRouter()

_active_ws: Optional[WebSocket] = None
_command_buffer: deque = deque(maxlen=100)
_pending_responses: Dict[str, asyncio.Future] = {}
_connected = False


async def send_agent_command(action: str, country: str, params: dict = None) -> Optional[dict]:
    if not _active_ws or not _connected:
        logger.warning(f"Agent not connected, cannot send: {action} {country}")
        return None

    request_id = str(uuid.uuid4())[:8]
    command = {
        "action": action,
        "country": country,
        "params": params or {},
        "request_id": request_id,
    }

    loop = asyncio.get_event_loop()
    fut = loop.create_future()
    _pending_responses[request_id] = fut
    _command_buffer.append(command)

    try:
        await _active_ws.send_json(command)
        logger.info(f"Command sent: {action} {country} (id={request_id})")
        result = await asyncio.wait_for(fut, timeout=300.0)
        return result
    except asyncio.TimeoutError:
        _pending_responses.pop(request_id, None)
        logger.warning(f"Command timeout: {action} {country} (id={request_id})")
        return {"status": "timeout", "action": action, "country": country}
    except Exception as e:
        _pending_responses.pop(request_id, None)
        logger.error(f"Command send failed: {e}")
        return {"status": "error", "error": str(e)}


@router.websocket("/ws/agent")
async def agent_websocket(ws: WebSocket):
    global _active_ws, _connected
    await ws.accept()
    _active_ws = ws
    _connected = True

    try:
        while True:
            raw = await ws.receive_text()
            msg = json.loads(raw)

            # ... 다른 메시지 처리 (생략)

            # 명령 응답 처리
            request_id = msg.get("request_id", "")
            if request_id and request_id in _pending_responses:
                fut = _pending_responses.pop(request_id)
                if not fut.done():
                    fut.set_result(msg)
                continue

    except WebSocketDisconnect:
        pass
    finally:
        _active_ws = None
        _connected = False
```

클라이언트(에이전트) 측: 명령을 받아 처리하고 응답.

```python
# collector-agent/transport/ws_client.py
import asyncio, json, uuid, os
import websockets
from websockets.exceptions import ConnectionClosed

_active_ws = None
_session_id = None


async def _handle_command(ws, command_handler, msg):
    action = msg.get("action", "unknown")
    try:
        response = await command_handler(msg)
        await ws.send(json.dumps(response))
        logger.info(f"Command response sent: {action} → {response.get('status', 'unknown')}")
    except Exception as e:
        logger.error(f"Command handling error ({action}): {e}")
        try:
            await ws.send(json.dumps({"status": "error", "error": str(e), "action": action}))
        except Exception:
            pass


async def ws_listen(command_handler):
    backoff = 3
    max_backoff = 30

    while True:
        try:
            async with websockets.connect(CONTROL_SERVER_WS, ping_interval=None, ping_timeout=None) as ws:
                global _active_ws, _session_id
                _active_ws = ws
                _session_id = uuid.uuid4().hex
                pid = os.getpid()
                backoff = 3

                await ws.send(json.dumps({
                    "type": "agent_hello",
                    "agent": "collector",
                    "session_id": _session_id,
                    "pid": pid,
                }))

                async for raw in ws:
                    try:
                        msg = json.loads(raw)
                        # 별도 Task로 실행 — WS 수신 루프를 블로킹하지 않음
                        asyncio.create_task(_handle_command(ws, command_handler, msg))
                    except json.JSONDecodeError:
                        logger.warning(f"Invalid JSON: {raw[:200]}")

        except ConnectionClosed as e:
            logger.warning(f"WebSocket closed: {e}")
        except Exception as e:
            logger.error(f"WebSocket error: {e}")

        logger.info(f"Reconnecting in {backoff}s...")
        await asyncio.sleep(backoff)
        backoff = min(backoff * 2, max_backoff)
```

---

## 1. RPC 위에서 양방향이 필요한 이유

기존 HTTP 만으로 안 됐던 점:
1. **에이전트가 NAT/방화벽 뒤** → 서버가 직접 호출 불가. 에이전트가 outbound 만 열고 들어와야 함.
2. **서버 → 에이전트 명령** + **에이전트 → 서버 로그 스트리밍** 둘 다 필요 → 단일 채널이 효율.
3. 응답 대기 동안 **다른 명령/로그를 차단하면 안 됨** → 비동기 매칭 필요.

WebSocket 한 채널에 여러 종류의 메시지를 다중화하면서, "명령" 이라는 sub-protocol 만 동기 RPC 처럼 쓰는 것이 이 패턴의 본질.

---

## 2. request_id 매칭의 핵심 — Future 사전등록

```python
fut = loop.create_future()
_pending_responses[request_id] = fut

await _active_ws.send_json(command)
result = await asyncio.wait_for(fut, timeout=300.0)
```

**왜 send 전에 Future 를 먼저 등록하나?**
- send 후 응답이 매우 빨리 와서 **수신 루프가 응답을 매칭할 때** 이미 dict 에 등록되어 있어야 함.
- 등록을 send 후에 하면 race condition 가능 (응답이 먼저 도착 → dict에 키 없음 → 무시됨).

**Future 의 역할**:
- 송신 측이 `await fut` 으로 대기.
- 수신 측이 다른 코루틴에서 `fut.set_result(msg)` 로 깨움.
- asyncio 의 결과 전달 메커니즘.

---

## 3. 수신 루프의 매칭 로직

```python
@router.websocket("/ws/agent")
async def agent_websocket(ws):
    while True:
        raw = await ws.receive_text()
        msg = json.loads(raw)

        request_id = msg.get("request_id", "")
        if request_id and request_id in _pending_responses:
            fut = _pending_responses.pop(request_id)
            if not fut.done():
                fut.set_result(msg)
            continue
```

핵심 두 줄:
- `_pending_responses.pop(request_id)` — 매칭하면서 동시에 dict 에서 제거 (메모리 누수 방지).
- `if not fut.done()` — 송신 측이 이미 타임아웃으로 cancel 했으면 set_result 가 InvalidStateError 발생. 방어.

---

## 4. 타임아웃과 정리 — race condition 방지

```python
try:
    result = await asyncio.wait_for(fut, timeout=300.0)
    return result
except asyncio.TimeoutError:
    _pending_responses.pop(request_id, None)  # ← 두 번째 인자 None 중요
    return {"status": "timeout", ...}
```

**타임아웃 시나리오 분석**:

```
T0    송신 측: pending[id] = fut, send → ws
T100  타임아웃 발생 → pop(id) → dict 비움
T101  응답 도착 → 수신 루프: id in pending? False → 무시 (정상)
```

```
T0    송신 측: pending[id] = fut, send → ws
T299  응답 도착 → pending.pop → fut.set_result
T300  타임아웃 발생 → 그러나 wait_for 가 이미 result 받고 빠져나옴
```

**race**: T299 와 T300 이 매우 가깝다면?
- `pending.pop(id, None)` 의 두 번째 인자 **None** 덕분에 dict 에 없어도 KeyError 안 남.
- `if not fut.done()` 덕분에 이미 set_result 된 fut 에 다시 set 안 함.

---

## 5. 클라이언트 측의 task 격리

```python
async for raw in ws:
    msg = json.loads(raw)
    asyncio.create_task(_handle_command(ws, command_handler, msg))
```

**왜 await 가 아니라 create_task?**
- `command_handler` 가 prepare/migration 같은 분 단위 작업이면, await 하면 그동안 **다음 명령 수신 못 함**.
- create_task 로 분리 → 수신 루프 즉시 다음 명령 받기 가능.

**부작용 가능성**:
- 동시 명령 처리 → 같은 자원 동시 접근 (예: 동시 prepare 두 번) → 멱등성/락 필요.
- task 가 폭주할 수 있음 → 큐로 백프레셔 줄지 고려.

---

## 6. send 동시성 — 단일 송신 채널의 문제

```python
await _active_ws.send_json(command)
```

여러 코루틴이 동시에 같은 WS 에 send 하면 **WebSocket 프레임이 인터리빙** 될 수 있음. starlette/websockets 라이브러리가 일부 보호하지만, 안전하려면 send 큐 + 단일 송신 task.

```python
_send_queue = asyncio.Queue()

async def _send_loop():
    while True:
        msg = await _send_queue.get()
        await _active_ws.send_json(msg)

# 다른 코루틴들은 큐로
await _send_queue.put(command)
```

위 코드는 단일 송신자(스케줄러) + 동시성 낮음 가정으로 큐 없이 직접 send.

---

## 7. agent_hello — 세션 ID 핸드셰이크

```python
# 클라이언트
_session_id = uuid.uuid4().hex
await ws.send(json.dumps({
    "type": "agent_hello",
    "agent": "collector",
    "session_id": _session_id,
    "pid": pid,
}))

# 서버
if msg_type == "agent_hello":
    _agent_session_id = msg.get("session_id")
    _agent_pid = msg.get("pid")
    await ws.send_json({"type": "welcome", "status": "connected"})
```

**왜 session_id?**
- 재배포/재기동 시 새 session_id 발급.
- UI 가 "이전 session=abc → 폴링으로 새 session=xyz 감지 → 재배포 완료"로 판단.
- HTTP keepalive 가 끊겨도 같은 프로세스면 session 유지(단, 이 코드는 매 연결마다 새로 발급해서 단순).

**pid 병기**:
- 디버깅용. 프로세스 식별.

---

## 8. 모듈 전역 상태의 트레이드오프

```python
_active_ws: Optional[WebSocket] = None
_pending_responses: Dict[str, asyncio.Future] = {}
```

**왜 모듈 전역?**
- WS 는 여러 요청을 가로지르는 장기 자원.
- FastAPI 의 의존성은 요청 단위 → 적합하지 않음.
- 단순함: 한 클래스로 감싸지 않고 함수 + 전역으로.

**한계**:
- 단일 에이전트 가정 (`_active_ws` 가 단일 참조).
- 여러 에이전트 지원하려면 dict[agent_id, WS] 로 변경 + send 함수 시그니처에 agent_id 필요.
- 테스트 격리 어려움 (전역 상태 초기화 fixture 필요).

→ 운영 단순함과 추상화 유연성의 트레이드오프. 1대 가정이면 OK.

---

## 9. 응용 포인트

- 양방향 RPC 가 필요하면: WS + request_id + Future 사전등록 패턴.
- 항상 `asyncio.wait_for` 로 타임아웃, 타임아웃 시 dict 정리 필수.
- 수신 루프에서 응답 매칭만 하지 말고, **`if not fut.done()` 체크** 필수 (race 방어).
- 클라이언트는 명령 처리를 `create_task` 로 분리 — 수신 루프 보호.
- 송신 동시성이 의심되면 send 큐 + 단일 송신 task.
- session_id 발급은 재배포 감지의 표준 트릭.
