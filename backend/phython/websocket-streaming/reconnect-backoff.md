# WebSocket 재연결과 지수 백오프

> 끊긴 WS 를 안전하게 다시 붙이고, 폭주하지 않도록 백오프하는 표준 패턴.

---

## 0. 분석 대상 코드

에이전트의 ws_listen 무한 루프.

```python
# collector-agent/transport/ws_client.py
import asyncio, json, uuid, os
import websockets
from websockets.exceptions import ConnectionClosed


async def ws_listen(command_handler):
    backoff = 3
    max_backoff = 30

    while True:
        try:
            logger.info(f"Connecting to {CONTROL_SERVER_WS}...")
            async with websockets.connect(
                CONTROL_SERVER_WS,
                ping_interval=None,
                ping_timeout=None,
            ) as ws:
                global _active_ws, _session_id
                _active_ws = ws
                _session_id = uuid.uuid4().hex
                pid = os.getpid()
                backoff = 3  # 연결 성공 시 백오프 리셋

                await ws.send(json.dumps({
                    "type": "agent_hello",
                    "agent": "collector",
                    "session_id": _session_id,
                    "pid": pid,
                }))

                async for raw in ws:
                    try:
                        msg = json.loads(raw)
                        asyncio.create_task(_handle_command(ws, command_handler, msg))
                    except json.JSONDecodeError:
                        logger.warning(f"Invalid JSON: {raw[:200]}")

        except ConnectionClosed as e:
            logger.warning(f"WebSocket closed: {e}")
        except ConnectionRefusedError:
            logger.warning(f"WebSocket connection refused")
        except Exception as e:
            logger.error(f"WebSocket error: {e}")

        logger.info(f"Reconnecting in {backoff}s...")
        await asyncio.sleep(backoff)
        backoff = min(backoff * 2, max_backoff)
```

---

## 1. 외부 while True 의 의미

```python
while True:
    try:
        async with websockets.connect(...) as ws:
            ...
    except ...:
        ...
    await asyncio.sleep(backoff)
```

- 하나의 연결이 살아있는 동안 `async with` 블록 안에서 대기.
- 연결이 끊기면 (정상/예외) 블록 빠져나오고 sleep 후 다음 반복 → 재연결.
- 외부 루프가 없으면 첫 disconnect 가 영구 종료.

---

## 2. 지수 백오프 — `min(backoff * 2, max_backoff)`

```python
backoff = 3
max_backoff = 30
...
backoff = min(backoff * 2, max_backoff)
```

진행: 3 → 6 → 12 → 24 → 30 → 30 → ...

**왜 지수?**
- 서버가 죽었을 때 모든 클라이언트가 1초마다 두드리면 복구 후 재연결 폭주(thundering herd).
- 지수 증가는 자연스럽게 부하 분산.

**왜 cap?**
- 끝없이 늘면 한 번 끊긴 후 영구 무응답 위험. 30초가 상한이면 1분 안에 두 번은 시도.

**연결 성공 시 리셋**:
```python
async with websockets.connect(...) as ws:
    backoff = 3   # ← 성공 후 리셋
```

→ 짧은 깜박임이 반복되는 환경에서, 매번 처음부터 시작.

---

## 3. jitter 가 빠진 이유와 보완

위 코드는 **jitter 없음**. 모든 에이전트가 같은 시점에 끊기면 같은 시점에 재시도.

운영상 collector 가 1대뿐이라 영향 없지만, 다중 인스턴스라면:
```python
import random
delay = backoff * (0.5 + random.random())  # 0.5x~1.5x 지터
await asyncio.sleep(delay)
```

또는 **decorrelated jitter** (AWS 권장):
```python
backoff = min(max_backoff, random.uniform(base, backoff * 3))
```

---

## 4. ping_interval=None 의 결정

```python
websockets.connect(URL, ping_interval=None, ping_timeout=None)
```

기본값:
- `ping_interval=20` (20초마다 ping 송신)
- `ping_timeout=20` (20초 안에 pong 안 오면 연결 close)

`None` 으로 끄면:
- ping/pong 자동 발생 안 함.
- 연결 끊김 감지가 느려짐 (TCP keepalive 만 의존).
- 그러나 NAT/방화벽이 ping/pong 트래픽을 idle 로 잘못 인식하지 않음.

위 코드는 **명시적으로 끔** → 운영 환경에서 ping/pong 으로 인한 false disconnect 가 빈번했을 가능성. 또는 양측이 application-level 메시지(로그 스트림 등) 가 자주 오가 keepalive 역할을 충분히 하는 상황.

---

## 5. 예외 분기 — 연결 단계별 분리

```python
except ConnectionClosed as e:
    logger.warning(f"WebSocket closed: {e}")
except ConnectionRefusedError:
    logger.warning(f"WebSocket connection refused")
except Exception as e:
    logger.error(f"WebSocket error: {e}")
```

| 예외 | 의미 | 흔한 원인 |
|------|------|----------|
| `ConnectionClosed` | 연결 후 정상/비정상 close | 서버 재시작, 클라이언트 timeout, 4xx close code |
| `ConnectionRefusedError` | 연결 자체가 거부됨 | 서버 다운, 잘못된 포트 |
| 기타 `Exception` | DNS, TLS, 모듈 버그 | 네트워크 단절, 인증서 만료 |

→ 동일하게 재시도하지만 로그 레벨/메시지가 달라 운영 시 진단 쉬워짐.

---

## 6. 재연결 시 상태 복원

```python
async with websockets.connect(...) as ws:
    _active_ws = ws
    _session_id = uuid.uuid4().hex   # ← 매 연결마다 새 ID
    await ws.send(json.dumps({"type": "agent_hello", ...}))
```

**핵심 결정**:
- `session_id` 는 **매 연결마다 새로 발급**. 이전 연결의 state 와 분리.
- `agent_hello` 핸드셰이크로 서버에 자기 자신을 다시 알림.

**서버 측의 영향**:
- 서버가 명령 응답을 기다리던 중 에이전트가 끊기면, 그 응답은 영영 안 옴.
- 송신 측 `wait_for(timeout=300)` 가 결국 timeout 으로 정리.
- 새 연결의 session_id 가 달라지므로 UI 가 "재배포됨"을 감지.

---

## 7. 자주 빠지는 함정

### 7.1 backoff 리셋 위치
연결 직후가 아니라 **연결이 충분히 안정** 됐다고 판단되는 시점에 리셋해야 안정적.
- 너무 빨리 리셋 → "연결되자마자 0.1초 후 끊김" 이 반복되면 백오프가 영원히 짧음.
- 위 코드는 connect 직후 리셋 — 단순함을 우선.

개선:
```python
async with websockets.connect(...) as ws:
    await ws.send(...)  # hello
    await ws.recv()     # welcome 받음 → 연결 검증 후
    backoff = 3
```

### 7.2 close 가 silent
WS close code 4xxx 가 의미 있는 거부(인증 실패 등) 일 수 있음:
```python
except ConnectionClosed as e:
    if e.code == 4001:
        logger.error("Auth failed, exiting")
        return  # ← 재시도 안 함
    ...
```

영구 종료 조건 분기가 없으면 잘못된 자격증명으로 무한 재시도.

### 7.3 task 누수
`asyncio.create_task(_handle_command(...))` 로 만든 task 가 연결 끊김 시 cancel 안 됨 → 영원히 살 수 있음. 보강:
```python
_inflight: set[asyncio.Task] = set()

async for raw in ws:
    msg = json.loads(raw)
    t = asyncio.create_task(_handle_command(ws, command_handler, msg))
    _inflight.add(t)
    t.add_done_callback(_inflight.discard)

# 연결 끊김 시
for t in _inflight:
    t.cancel()
```

### 7.4 잠깐 끊겼다 붙는 사이 send 시도
```python
def get_ws():
    return _active_ws

# 다른 코드
ws = get_ws()
if ws:
    await ws.send(...)  # ← 이 시점 ws 가 closed 일 수 있음
```

→ send 자체에 try/except, 또는 `_connected` 플래그 같이 체크.

---

## 8. 클라이언트 vs 서버의 차이

위 코드는 **클라이언트 측** 재연결.

서버 측은 다른 모델:
- FastAPI/Starlette 의 `@app.websocket` 핸들러는 **연결당 한 번** 호출됨.
- 끊기면 함수가 종료. 다음 연결은 새 호출.
- 서버는 "재연결" 이라는 개념이 없고, 그냥 들어오는 연결을 받음.
- 서버 측 상태(`_pending_responses`, `_active_ws`) 만 정리하면 됨.

---

## 9. 응용 포인트

- WS 클라이언트는 항상 외부 while + 지수 백오프 + cap.
- backoff 리셋은 연결 안정 확인 후가 안전 (welcome 수신 등).
- 다중 인스턴스 환경은 jitter 추가.
- ping_interval/ping_timeout 은 환경 특성에 맞춤. 모르면 기본값 유지.
- close code 분기로 영구 실패 종료 조건 추가.
- in-flight task 추적해서 연결 끊김 시 cancel.
