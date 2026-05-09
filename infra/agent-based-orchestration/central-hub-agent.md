# 중앙 허브 에이전트 — N개 컨테이너 단일 게이트

> 하나의 관제 서버가 분산된 컨테이너들을 관리할 때, **에이전트 1대로 N개 컨테이너 묶기** 패턴.
> 5개국 × 3서비스 = 15개 컨테이너를 단일 WebSocket 으로 집중하는 설계.

---

## 1. 문제 정의

### 1.1 N:N 직접 연결의 비용
```
관제 서버 ──┐
            ├──→ 컨테이너 1 (HTTP/SSH)
            ├──→ 컨테이너 2
            ├──→ 컨테이너 3
            ...
            └──→ 컨테이너 15
```

문제:
- 각 컨테이너에 대한 인증/네트워크 정책 N배.
- 컨테이너 추가/삭제 시 관제 서버 갱신 필요.
- 컨테이너가 NAT 뒤면 관제 서버가 직접 접근 불가.
- 컨테이너 측 작은 권한 변경도 관제 서버에 영향.

### 1.2 에이전트 허브 모델
```
관제 서버 ──── WebSocket ──── 에이전트(host) ───── 컨테이너 1
                                              ├── 컨테이너 2
                                              ├── 컨테이너 3
                                              ...
                                              └── 컨테이너 15
```

이점:
- 관제 서버는 **에이전트 1개의 연결**만 관리.
- 컨테이너 변경은 에이전트 측에서만 흡수.
- 에이전트가 outbound 연결을 열음 → NAT/방화벽 친화.
- 컨테이너에 대한 권한이 모두 에이전트 한 곳에 집중.

---

## 2. 책임 분담

| 컴포넌트 | 책임 |
|---------|------|
| **관제 서버 (orchestration)** | UI, 정책 결정, 명령 발행, 메트릭 저장 |
| **에이전트 (collector)** | 메트릭 수집 + 명령 중계 + 자동 복구 + 자가 배포 |
| **개별 컨테이너 (migration-api)** | 비즈니스 로직만. 관제 신경 안 씀 |

핵심 원칙: **관제는 결정, 에이전트는 실행, 컨테이너는 일**.

---

## 3. 통신 채널 설계

### 3.1 단일 WebSocket — `/ws/agent`

```python
# 관제 서버 측
@router.websocket("/ws/agent")
async def agent_websocket(ws: WebSocket):
    await ws.accept()
    _active_ws = ws
    while True:
        msg = json.loads(await ws.receive_text())
        # 메시지 타입에 따라 분기
```

```python
# 에이전트 측
async def ws_listen(command_handler):
    async with websockets.connect(CONTROL_SERVER_WS) as ws:
        await ws.send(json.dumps({"type": "agent_hello", "session_id": ..., "pid": ...}))
        async for raw in ws:
            asyncio.create_task(_handle_command(ws, command_handler, json.loads(raw)))
```

### 3.2 메시지 다중화

한 채널에 여러 종류 메시지가 흐른다:

| type | 방향 | 용도 |
|------|------|------|
| `agent_hello` | A→S | 핸드셰이크 (session_id, pid) |
| `welcome` | S→A | hello ack |
| 명령 (`{action, country, params, request_id}`) | S→A | 컨테이너 제어 |
| 명령 응답 (`{request_id, status, data}`) | A→S | 명령 결과 |
| `log_stream` | A→S | 컨테이너 로그 forward |
| `agent_journal_line` | A→S | 에이전트 자체 journalctl |

`type` 또는 `action`/`request_id` 필드로 분기.

### 3.3 보조 HTTP 채널

WS 외에 메트릭/로그 push 는 HTTP POST:
- `/api/migration/metrics/ingest` — 10초마다 메트릭 push.
- `/api/migration/logs/ingest` — 에러 로그 push.

WS 와 HTTP 분리 이유:
- 메트릭은 sustained 트래픽 → HTTP keep-alive 가 효율.
- WS 는 명령 중심 (간헐적, request-response).

---

## 4. 명령 중계 패턴

### 4.1 action 접두사 라우팅

에이전트의 `command_relay.handle_command(msg)`:

```python
async def handle_command(msg: Dict[str, Any]) -> Dict[str, Any]:
    action = msg.get("action", "")
    country = msg.get("country", "")
    params = msg.get("params", {})
    request_id = msg.get("request_id", "")

    # 1순위: system.* — country 검증 SKIP
    if action.startswith("system."):
        return await _handle_system_command(action, params, request_id, start)

    # 2순위: country 검증
    server = _find_server(country)
    if not server:
        return _error_response(request_id, action, country, f"Unknown country: {country}", start)

    base = server.migration_api  # 예: http://localhost:10021

    # 3순위: 접두사별 분기
    if action.startswith("env."):       return await _handle_env_command(...)
    if action.startswith("git."):       return await _handle_git_command(...)
    if action.startswith("container."): return await _handle_docker_command(...)

    # 4순위: route_map 테이블 매칭
    if action not in route_map:
        return _error_response(..., f"Unknown action: {action}", start)
    endpoint_key, method = route_map[action]
    url = f"{base}{ORCH_ENDPOINTS[endpoint_key]}"
    # HTTP GET/POST/DELETE 프록시
```

### 4.2 분기 순서의 의도

| 순위 | 분기 | country 검증 | 이유 |
|-----|------|------------|------|
| 1 | `system.*` | X | 자기 자신(에이전트) 명령. country 무관 |
| 2 | (검증 게이트) | O | 이후 모든 분기는 country 필요 |
| 3 | `env.*`/`git.*`/`container.*` | O | 호스트 측 작업 (compose 디렉토리) |
| 4 | route_map | O | 컨테이너 내부 API 프록시 |

### 4.3 HTTP 프록시 vs 직접 실행
- `migration.*`/`index.*`/`customer.*` — migration-api 컨테이너에 HTTP 프록시.
- `git.*`/`env.*`/`container.*` — 호스트에서 subprocess 실행.

→ **에이전트는 두 모드 동시 — 컨테이너 외부 작업은 직접, 컨테이너 내부 작업은 프록시**.

---

## 5. session_id — 재기동 감지

```python
# 에이전트 측
_session_id = uuid.uuid4().hex  # 매 연결마다 새로 발급
await ws.send(json.dumps({
    "type": "agent_hello",
    "agent": "collector",
    "session_id": _session_id,
    "pid": os.getpid(),
}))
```

### 활용
1. UI 가 재배포 트리거 → orch 가 에이전트에 `system.redeploy_collector` 명령.
2. UI 가 즉시 polling 시작: `GET /api/migration/agent-status` → `{session_id: "abc..."}`.
3. 에이전트 재시작 후 새 session_id 로 재연결.
4. UI 가 polling 결과의 session_id 가 바뀐 걸 감지 → "재배포 완료".

### 왜 단순 "connected" 플래그로 부족한가
- WS 단절은 짧은 네트워크 글리치로도 발생.
- "끊김 → 재연결" 이 재배포의 증거가 아님.
- session_id 가 **바뀌면** 새 프로세스. 단순 재연결과 구분.

---

## 6. 단일 에이전트 가정의 한계와 확장

### 6.1 현재 코드의 단순화
```python
_active_ws: Optional[WebSocket] = None  # 단일 참조
```

→ 두 번째 에이전트 연결되면 첫 번째가 덮어씌워짐.

### 6.2 다중 에이전트로 확장
```python
_agents: Dict[str, WebSocket] = {}  # agent_id → WS

@router.websocket("/ws/agent")
async def agent_ws(ws):
    await ws.accept()
    msg = await ws.receive_text()
    hello = json.loads(msg)
    agent_id = hello["agent_id"]  # 클라이언트가 자기 id 보냄
    _agents[agent_id] = ws

    while True:
        # ... 메시지 처리
```

명령 발송 시:
```python
async def send_agent_command(agent_id, action, ...):
    ws = _agents.get(agent_id)
    if not ws:
        return None
    await ws.send_json({"action": action, "request_id": ..., ...})
```

### 6.3 파티셔닝 — 에이전트 1대 → N대
- 컨테이너 수가 늘어나면 한 에이전트가 모두 관리하기 부담.
- "한 에이전트당 한 호스트" 또는 "한 에이전트당 N개 국가" 로 분리.
- 관제 서버는 명령 발행 시 어느 에이전트에 보낼지 라우팅.

---

## 7. 자가 배포 통합

에이전트 자기 자신을 재배포할 수 있어야 → 운영 편의.

```
UI → orch: system.redeploy_collector 명령
orch → agent: WS 로 명령 전달
agent: 즉시 응답 ("redeploying"), 백그라운드로 git pull + nohup detach restart
agent: systemd restart → 새 session_id 로 재연결
UI: session_id 폴링으로 완료 감지
```

핵심: **명령 응답이 재배포 완료를 의미하지 않음**. 재배포는 비동기 + 검증은 session_id polling.

---

## 8. 관제 평면 vs 데이터 평면 분리

### 8.1 관제 평면
- 명령 흐름: UI → orch → agent → 컨테이너.
- 프로토콜: WS RPC + request_id.
- 장애 시 영향: 명령 못 보냄. 단 컨테이너의 자체 동작은 계속.

### 8.2 데이터 평면
- 메트릭/로그 흐름: agent → orch → UI.
- 프로토콜: HTTP push + WS broadcast.
- 장애 시 영향: 모니터링 단절. 단 명령은 여전히 가능.

### 8.3 분리의 가치
- orch 가 죽어도 컨테이너의 비즈니스 로직(검색, 마이그레이션) 은 정상.
- agent 가 죽어도 orch UI 는 떠 있음 ("agent disconnected" 상태로 표시).
- 컨테이너가 죽으면 다른 컨테이너는 정상 + agent 가 자동 재시작 시도.

각 레이어가 독립적으로 fail-fast / fail-soft.

---

## 9. 응용 포인트

- N개 컨테이너 관리 = 에이전트 1대 + 단일 WS 채널.
- 명령은 `{action, country, params, request_id}` 표준 봉투.
- action 접두사로 라우팅 (`system.` / `env.` / `git.` / `container.` / 도메인.).
- 재기동 감지는 session_id (단순 connected 플래그로 부족).
- 단일 에이전트로 시작 → 부하 증가 시 파티셔닝.
- 관제/데이터 평면 분리 → 한 평면 장애가 다른 평면에 미치지 않게.
