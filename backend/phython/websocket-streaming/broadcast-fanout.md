# WebSocket 브로드캐스트와 좀비 클라이언트 차단

> "한 명의 느린 구독자가 전체를 멈추지 않게" 하는 방어 패턴.

---

## 0. 분석 대상 코드

```python
# orchestration/app/api/migration_ws.py
from fastapi import WebSocket, WebSocketDisconnect

_dashboard_clients: list[WebSocket] = []


@router.websocket("/ws/migration/dashboard")
async def dashboard_websocket(ws: WebSocket):
    await ws.accept()
    _dashboard_clients.append(ws)
    logger.info(f"Dashboard client connected (total: {len(_dashboard_clients)})")

    try:
        while True:
            raw = await ws.receive_text()
            msg = json.loads(raw)
            # 클라이언트가 보내는 요청 처리...
    except WebSocketDisconnect:
        pass
    finally:
        if ws in _dashboard_clients:
            _dashboard_clients.remove(ws)


async def broadcast_to_dashboard(data: dict):
    """
    모든 대시보드 클라이언트에 데이터 브로드캐스트.

    send_json 이 2초를 넘기면 좀비 클라이언트로 간주해 제거. 좀비의 송신 버퍼 hang 이
    호출자(특히 /ws/agent 수신 루프)를 무한 block 시켜 명령 응답 매칭이 멈추는 사태 방지.
    """
    disconnected = []
    for client in list(_dashboard_clients):
        try:
            await asyncio.wait_for(client.send_json(data), timeout=2.0)
        except Exception:
            disconnected.append(client)
    for client in disconnected:
        if client in _dashboard_clients:
            _dashboard_clients.remove(client)
```

---

## 1. 좀비 클라이언트가 뭐고 왜 위험한가

**좀비**: TCP 연결은 살아있는 것처럼 보이지만 실제로는 send 가 영원히 안 끝나는 상태.

발생 원인:
- 클라이언트 호스트 갑작스런 전원 차단 → FIN 안 옴.
- 네트워크 미들박스가 idle TCP connection 을 silent drop.
- 클라이언트가 send 버퍼는 받지만 application 측에서 안 읽음 → TCP backpressure.

연쇄 영향:
- `await ws.send_json(data)` 가 영원히 대기.
- 호출자가 동기적으로 모든 클라이언트를 await → **첫 좀비 한 명이 전체 브로드캐스트 정지**.
- 이 브로드캐스트가 `/ws/agent` 수신 루프 안에서 호출되면 → **agent 명령 응답 처리도 정지**.
- 결국 운영 화면 전체가 freeze.

---

## 2. 1차 방어: `asyncio.wait_for` 타임아웃

```python
await asyncio.wait_for(client.send_json(data), timeout=2.0)
```

- 2초 안에 send 가 끝나야 함. 그렇지 않으면 `asyncio.TimeoutError`.
- timeout 발생 시 그 task 는 cancel 됨 → send 가 도중에 멈춤.
- 단, **send 가 부분 전송된 상태**일 수 있음 → 다음 send 가 실패할 가능성 → 좀비로 분류해서 제거.

**왜 2초?**
- 정상 클라이언트 + 일반 메시지(<1KB) 면 ms 단위로 끝남.
- 2초는 **확실히 좀비** 라고 판단할 만한 마진. 1초는 잠깐 GC 같은 건 잘못 판단할 수 있음.

---

## 3. 2차 방어: Fire-and-Forget 으로 격리

위 코드는 `broadcast_to_dashboard` 자체에는 fire-and-forget 이 없지만, **호출자가** create_task 로 감싼다:

```python
# /ws/agent 수신 루프
if msg_type == "log_stream":
    asyncio.create_task(broadcast_to_dashboard(msg))
```

→ 브로드캐스트가 2초 타임아웃까지 갈 만큼 느려도, 수신 루프는 즉시 다음 메시지 처리.

**두 방어의 조합**:
- create_task → 브로드캐스트가 호출자 막지 않음.
- wait_for(timeout) → 브로드캐스트 안에서 좀비가 정상 클라이언트 막지 않음.

둘 중 하나만 있으면 위험:
- 타임아웃만: 호출자가 브로드캐스트 끝까지 기다림.
- create_task 만: 브로드캐스트 task 자체가 좀비 때문에 영원히 살아있음.

---

## 4. `for client in list(_dashboard_clients)` — 복제 순회

```python
for client in list(_dashboard_clients):
    ...
```

**왜 list() 로 복제?**
- 순회 중 `disconnected.append` + 별도 루프에서 `remove` 하지만, 다른 코루틴이 동시에 append/remove 할 수도 있음.
- 또는 같은 루프 안에서 직접 `_dashboard_clients.remove(client)` 하면 RuntimeError (size changed during iteration).
- 복제하면 순회 중 안전.

**대안**: 락(`asyncio.Lock`) 사용. 코드는 단순화 우선해서 락 없이.

---

## 5. 더 나은 병렬 브로드캐스트

위 코드는 **순차** 송신:
```python
for client in list(_dashboard_clients):
    try:
        await asyncio.wait_for(client.send_json(data), timeout=2.0)
    except Exception:
        disconnected.append(client)
```

100명 구독자면 최악의 경우 100 × 2초 = 200초.

**병렬화**:
```python
async def _safe_send(client, data):
    try:
        await asyncio.wait_for(client.send_json(data), timeout=2.0)
        return None
    except Exception:
        return client

async def broadcast_to_dashboard(data: dict):
    results = await asyncio.gather(
        *(_safe_send(c, data) for c in list(_dashboard_clients)),
        return_exceptions=True,
    )
    for client in results:
        if client and client in _dashboard_clients:
            _dashboard_clients.remove(client)
```

→ 모든 클라이언트가 동시에 send 받음. 좀비도 2초 내 cap.

이 프로젝트는 구독자 수가 적어(운영 화면 한두 명) 순차로 충분.

---

## 6. 클라이언트 등록/해제 race

```python
@router.websocket("/ws/migration/dashboard")
async def dashboard_websocket(ws):
    await ws.accept()
    _dashboard_clients.append(ws)
    try:
        while True:
            raw = await ws.receive_text()
            ...
    except WebSocketDisconnect:
        pass
    finally:
        if ws in _dashboard_clients:
            _dashboard_clients.remove(ws)
```

**핵심**:
- 등록은 accept 직후.
- 해제는 finally 로 보장 → 정상/예외 모두 정리.
- `if ws in _dashboard_clients:` — 이미 broadcast 측에서 좀비로 제거됐을 수 있음. 중복 제거 방지.

**주의**: WebSocket 객체가 hashable 한지는 별 문제 없으나, `==` 비교로 list 에서 찾음. 동일 객체 비교라 정상 동작.

---

## 7. send 동시성 문제

여러 코루틴이 같은 `ws.send_json` 을 동시에 호출하면 프레임 인터리빙 위험. 이 코드는:
- broadcast 안에서는 한 클라이언트당 한 send.
- 다른 곳에서 같은 클라이언트에 send 하지 않음.

→ 단일 송신 채널 가정. 만약 다른 경로에서도 보내면 송신 큐 + 단일 송신 task 패턴 필요.

---

## 8. 응용 포인트

- 브로드캐스트는 항상 (a) 호출자에서 fire-and-forget (b) 안에서 wait_for 타임아웃 두 겹.
- 클라이언트 리스트 순회는 `list(...)` 복제로 동시 수정 보호.
- 100+ 구독자면 `asyncio.gather` 로 병렬화.
- 등록/해제는 `try/finally` 로 짝 맞춤.
- 좀비 의심 클라이언트는 즉시 제거 (재연결은 클라이언트 책임).
