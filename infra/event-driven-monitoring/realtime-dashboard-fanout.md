# 실시간 대시보드 — push 기반 fan-out

> 백엔드의 운영 데이터를 브라우저 대시보드로 실시간 push 하는 구조.
> WebSocket broadcast + 좀비 클라이언트 차단 + 메트릭 캐시.

---

## 1. 두 push 채널

```
agent → orch → 브라우저 dashboard
```

### 채널 1: 이벤트 broadcast (`/ws/migration/dashboard`)
- 이벤트 발생 시 즉시 push (마이그레이션 진행률, 에러 등).
- 이벤트 기반 → 발생 빈도 들쭉날쭉.

### 채널 2: 메트릭 주기 push (`/ws/metrics`)
- 5초마다 정기적으로 push.
- 시스템 메트릭 (CPU/메모리/디스크).

---

## 2. broadcast 패턴

```python
# orch: 이벤트 발생 시
async def broadcast_to_dashboard(data: dict):
    """
    send_json 이 2초를 넘기면 좀비 클라이언트로 간주해 제거.
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

### 핵심 방어 3중

| 방어 | 코드 |
|------|------|
| Fire-and-forget 호출 | `asyncio.create_task(broadcast_to_dashboard(msg))` (호출자) |
| 개별 send 타임아웃 | `asyncio.wait_for(send, timeout=2.0)` |
| 좀비 즉시 제거 | `disconnected.append + remove` |

→ 한 좀비 클라이언트 때문에 전체 broadcast 가 멈추지 않음.

---

## 3. 메트릭 push — 캐시 + 주기 fan-out

```python
async def metrics_pusher():
    while True:
        snapshot = get_latest_snapshot()  # 메모리 캐시
        msg = {"type": "metrics_all", "data": snapshot}
        for ws in list(_monitor_ws_list):
            asyncio.create_task(safe_send(ws, msg))
        await asyncio.sleep(5)
```

### 캐시 활용
- agent 가 매 10초 push → orch 는 메모리에 캐시.
- pusher 는 캐시만 읽어서 broadcast (DB 안 거침).
- N 개 클라이언트 동시 접속해도 read 비용 ~0.

### `asyncio.create_task(safe_send(...))` 의 의미
- N 개 클라이언트에 동시 send.
- 좀비 한 명이 5초 cycle 을 막지 않음.

```python
async def safe_send(ws, msg):
    try:
        await ws.send_json(msg)
    except Exception:
        pass  # 조용히 무시 (좀비는 다음 사이클에서도 자연 정리)
```

---

## 4. 클라이언트 등록/해제의 race

```python
@router.websocket("/ws/migration/dashboard")
async def dashboard_websocket(ws):
    await ws.accept()
    _dashboard_clients.append(ws)
    try:
        while True:
            await ws.receive_text()  # 클라이언트가 보낼 수도 있음 (현재 프로젝트는 거의 안 씀)
    except WebSocketDisconnect:
        pass
    finally:
        if ws in _dashboard_clients:
            _dashboard_clients.remove(ws)
```

- 등록은 accept 직후.
- 해제는 finally 로 보장.
- broadcast 측이 좀비로 이미 제거했을 수 있음 → `if ws in _dashboard_clients` 체크.

순회 시 `list(_dashboard_clients)` 로 복제하여 동시 수정 위험 회피.

---

## 5. 메시지 다중화

대시보드 한 채널에 여러 이벤트 type 이 흐름:

```python
{"type": "log_stream", "country": "kr", "line": "..."}
{"type": "migration_progress", "country": "kr", "progress": {...}}
{"type": "migration_error", "error": {...}}
{"type": "schedule_triggered", "country": "kr", "run_id": "..."}
{"type": "self_metrics", "data": {...}}
```

클라이언트 측:
```js
ws.onmessage = (event) => {
    const msg = JSON.parse(event.data);
    switch (msg.type) {
        case "log_stream": appendLog(msg); break;
        case "migration_progress": updateProgress(msg); break;
        // ...
    }
};
```

### 단일 채널 vs 채널 분리
- 단일: 클라이언트 코드 단순 (한 WS).
- 분리: 채널별 독립 (예: 로그 / 메트릭 / 명령 결과 별도).

이 시스템은 단일 다중화 — 클라이언트 코드 단순함 우선.

---

## 6. push 만 vs request-response 양방향

대시보드는 server push 가 주이지만, 클라이언트가 보낼 수도 있음:

```python
# server 측 수신 처리
while True:
    raw = await ws.receive_text()
    msg = json.loads(raw)

    if msg.get("type") == "command":
        # 명령 → agent 로 forward
        asyncio.create_task(_handle_dashboard_command(ws, action, country, params))

    elif msg.get("type") == "get_overview":
        # 동기 조회
        await ws.send_json({"type": "overview", "data": metrics_store.get_overview()})

    elif msg.get("type") == "get_logs":
        await ws.send_json({"type": "saved_logs", "logs": get_command_logs(country)})
```

→ "주로 push" + "가끔 클라이언트가 명령".

명령은 `_handle_dashboard_command` 안에서 별도 task → WS 수신 루프 보호.

---

## 7. Schedule 이벤트 broadcast

```python
# scheduler 가 발화 시
async def trigger(country):
    run_id = uuid.uuid4().hex
    await broadcast_to_dashboard({
        "type": "schedule_triggered",
        "country": country,
        "run_id": run_id,
        "scheduled_time": ...,
        "start_time": now_iso(),
    })
    try:
        result = await send_agent_command("migration.start", country, ...)
        await broadcast_to_dashboard({
            "type": "schedule_completed",
            "country": country, "run_id": run_id,
            "duration_sec": (now - start).total_seconds(),
        })
    except asyncio.TimeoutError:
        await broadcast_to_dashboard({
            "type": "schedule_failed",
            "country": country, "run_id": run_id,
            "error": "timeout",
        })
```

→ UI 는 별도 polling 없이도 schedule 의 lifecycle 관찰 가능.

---

## 8. 대시보드의 페이지 새로고침 대응

브라우저 새로고침 시 WS 새로 연결 → 누락된 이벤트는 어떻게?

### 옵션 A: 메모리 buffer (이 프로젝트)
```python
_command_logs: Dict[str, deque] = {}
_MAX_COMMAND_LOGS = 20

# 클라이언트가 요청 시
elif msg_type == "get_logs":
    country = msg.get("country")
    await ws.send_json({"type": "saved_logs", "logs": get_command_logs(country)})
```

→ 국가별 최근 20개 명령 로그 in-memory.
→ 새 클라이언트가 `get_logs` 요청 → 즉시 응답.

### 옵션 B: PostgreSQL 영속 + 초기 fetch
- 대시보드 페이지 로드 시 REST API 로 최근 N개 fetch.
- 그 이후 WS 로 실시간.

옵션 A 는 단기 보관, 옵션 B 는 장기 보관. 둘 다 결합도 가능.

---

## 9. 함정 — broadcast 가 무한 backlog

브라우저 N대 + 한 명의 send 가 느리면:
- broadcast 도중 그 클라이언트에서 hang.
- 타임아웃 안 걸면 다음 broadcast 가 큐 적체.

방어:
- 위 패턴: 개별 send timeout + 즉시 좀비 제거.
- 추가 방어: broadcast 전체에 대한 큐 backlog 모니터링 (현재 미구현).

---

## 10. 응용 포인트

- 실시간 대시보드는 server push (WS broadcast) 가 표준.
- 메트릭은 캐시 + 주기 push, 이벤트는 발생 즉시 broadcast.
- 좀비 차단 3중: fire-and-forget + 개별 timeout + 즉시 제거.
- 메시지 다중화로 클라이언트 코드 단순화.
- 페이지 새로고침 대응: in-memory buffer + 초기 fetch 패턴.
- broadcast 측 hang 이 핵심 흐름 (명령 처리 등) 을 막지 않게 격리.
