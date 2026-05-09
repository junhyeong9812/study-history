# BackgroundTasks 와 Fire-and-Forget — 응답 후 작업 vs 진짜 비동기

> "사용자 요청은 빨리 응답하고, 무거운 작업은 뒤에서 돌린다"는 흔한 요구를 풀 때 자주 혼동되는 두 도구를 한번에 정리.

---

## 0. 분석 대상 코드

### 0.1 `BackgroundTasks` — 응답 직후 실행

```python
# example-app/routers/migration_router.py
from fastapi import APIRouter, BackgroundTasks, HTTPException

router = APIRouter()

@router.post("/start", response_model=Dict[str, Any])
async def start_migration(
    request: MigrationStartRequest,
    background_tasks: BackgroundTasks
):
    if migration_status["is_running"]:
        raise HTTPException(status_code=409, detail="Migration is already running")

    migration_status.update({"is_running": True, "start_time": datetime.now(), ...})

    background_tasks.add_task(
        run_migration_background,
        request.start_index,
        request.end_index,
        request.batch_size,
        request.use_parallel,
        request.resume
    )

    return {"status": "started", "message": "Migration started in background", ...}
```

### 0.2 `asyncio.create_task` — 진짜 fire-and-forget

```python
# orchestration/app/api/migration_ws.py
@router.websocket("/ws/agent")
async def agent_websocket(ws: WebSocket):
    while True:
        msg = json.loads(await ws.receive_text())
        if msg.get("type") == "log_stream":
            asyncio.create_task(broadcast_to_dashboard(msg))   # ← 응답을 기다리지 않음
            ...
            continue
```

---

## 1. 두 도구의 본질적 차이

| 항목 | `BackgroundTasks` | `asyncio.create_task` |
|------|-------------------|----------------------|
| 실행 시점 | **응답을 보낸 직후** | 즉시 (호출하는 순간) |
| 응답과의 관계 | 응답이 끝날 때까지 대기 (그 후 실행) | 응답과 무관, 동시 진행 |
| 적합한 용도 | "응답 후 알림 발송, 로그 기록" | "응답 도중에도 다른 클라이언트로 브로드캐스트" |
| 취소 가능성 | FastAPI가 관리 | 직접 추적 (`task.cancel()`) |
| 예외 전파 | 응답에 영향 없음 (서버 로그만) | task 가 silent fail 할 수 있음 (보호 필수) |
| 라이프사이클 | 요청 단위 | 앱 수명 전체 (장기 task 가능) |

---

## 2. `BackgroundTasks` 깊이 보기

### 2.1 동작 원리

```python
async def start_migration(req, background_tasks: BackgroundTasks):
    background_tasks.add_task(run_migration_background, ...)
    return {"status": "started"}
```

내부적으로:
1. `add_task` 는 함수와 인자를 큐에 등록만 함.
2. 핸들러 함수가 응답을 만들어 반환.
3. **응답이 클라이언트에 전송된 직후** ASGI 미들웨어가 등록된 task 를 순차 실행.

→ 즉, **"오래 걸리지만 결과를 응답에 담을 필요 없는 작업"** 에 적합.

### 2.2 주의할 점

**(a) 응답이 너무 빨리 끝나는 게 함정이 아니다 — 그 후가 문제**:
- 백그라운드 작업 도중 서버가 SIGTERM 받으면 그 작업은 도중에 끊김.
- 중요한 작업이면 외부 큐(Celery, Redis)에 위임.

**(b) 동기 함수도 add_task 가능**:
- 동기 함수면 별도 스레드에서 실행됨 (스타렛 동작).
- async 함수면 같은 이벤트 루프에서.

**(c) 트랜잭션 경계 주의**:
```python
async def create_user(req, background_tasks: BackgroundTasks, db=Depends(get_db)):
    user = await db.insert(...)
    background_tasks.add_task(send_welcome_email, user.email)
    await db.commit()
    return user
```
→ `add_task` 자체는 commit/rollback 영향 없음. 하지만 **send_welcome_email 이 실행될 때는 db 세션이 이미 닫혀있음** → email 함수 안에서 db 접근하면 안 됨.

### 2.3 위 마이그레이션 예제의 의도

마이그레이션은 수십 분~수시간 걸리는 작업. API 호출 한 번에 그걸 다 끝내려고 기다리면 클라이언트 타임아웃·재시도 폭주.

흐름:
```
POST /start
  ↓
status = "started" 즉시 반환
  ↓ (응답 후)
run_migration_background 가 ProcessPoolExecutor로 병렬 처리
  ↓
GET /status 로 폴링하여 진행률 조회
```

**진짜 작업은 ProcessPoolExecutor** 가 한다. BackgroundTasks 는 단지 그걸 띄우고 빠지는 트리거 역할.

---

## 3. `asyncio.create_task` — fire-and-forget 의 진짜 의미

### 3.1 왜 fire-and-forget?

위 WS 핸들러 코드를 다시 보자:

```python
if msg.get("type") == "log_stream":
    asyncio.create_task(broadcast_to_dashboard(msg))
    continue
```

만약 그냥 `await broadcast_to_dashboard(msg)` 했다면?
- 대시보드 클라이언트 N명에게 순차 send_json.
- 그중 한 명이 좀비(끊긴 줄 모르는 소켓)면 send_json 이 hang.
- 이 hang 이 **`/ws/agent` 의 receive_text 루프 전체를 블로킹** → 에이전트의 다른 메시지(특히 명령 응답) 가 처리 안 됨.
- → 명령 타임아웃 → 운영 화면 망가짐.

`asyncio.create_task` 로 분리하면:
- broadcast 가 별도 task 로 도네 → /ws/agent 루프는 즉시 다음 메시지 처리.
- broadcast 안에서 좀비를 감지해 제거.

### 3.2 broadcast 안의 추가 보호

```python
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

→ create_task 로 격리하더라도 **개별 send_json 에 timeout** 을 걸어야 좀비 task 가 영원히 살지 않음.

### 3.3 강한 참조 보유 — task GC 함정

`asyncio.create_task` 가 만든 task 객체에 **강한 참조가 없으면** 가비지 컬렉터가 도중에 수거할 수 있다 (Python 3.11+ 에서도 약한 참조만 보유).

```python
# 잘못된 예
asyncio.create_task(do_work())  # 어디에도 안 담음

# 권장
_background_tasks: set[asyncio.Task] = set()
def fire(coro):
    t = asyncio.create_task(coro)
    _background_tasks.add(t)
    t.add_done_callback(_background_tasks.discard)

fire(do_work())
```

위 WS 핸들러 코드는 broadcast 가 자주 호출되어 사실상 GC 대상이 되기 어렵다는 사실에 의존하고 있다 — 위험. 신규 코드라면 `_background_tasks` 셋 패턴 권장.

### 3.4 예외가 묻혀 있는 문제

`asyncio.create_task(coro)` 안의 예외는 **task 가 await 되거나 done_callback 에서 처리하지 않으면** 인터프리터 종료 시 경고로만 남는다.

```python
def _log_exc(task):
    if task.cancelled():
        return
    exc = task.exception()
    if exc:
        logger.error(f"task failed: {exc}", exc_info=(type(exc), exc, exc.__traceback__))

t = asyncio.create_task(broadcast_to_dashboard(msg))
t.add_done_callback(_log_exc)
```

---

## 4. 두 도구를 같이 쓰는 케이스

### 4.1 BackgroundTasks 안에서 create_task

```python
async def heavy_job(items):
    tasks = [process_one(it) for it in items]
    await asyncio.gather(*tasks)

@router.post("/jobs")
async def create_job(req, background_tasks: BackgroundTasks):
    background_tasks.add_task(heavy_job, req.items)
    return {"status": "started"}
```

→ BackgroundTasks 로 띄우고, 안에서 `gather` 로 병렬 실행. 깔끔.

### 4.2 BackgroundTasks 가 어울리지 않는 경우

- **장기 실행 (>몇 분)**: 외부 큐(Celery/RQ/Redis Streams) 권장.
- **재시도/내구성 필요**: 외부 큐 권장 (BackgroundTasks 는 프로세스 죽으면 사라짐).
- **여러 워커 인스턴스에 분산**: 외부 큐 필수.

---

## 5. 응용 포인트

- "응답 후 가벼운 후속 작업" → `BackgroundTasks` (이메일 발송, 감사 로그 등).
- "장기 작업 트리거" → `BackgroundTasks` 로 워커 프로세스 띄우는 정도까지만. 진짜 작업은 별도 풀.
- "WS 수신 루프에서 다른 곳으로 forward" → 반드시 `asyncio.create_task` (수신 루프 보호).
- create_task 사용 시 (a) 강한 참조 (b) 예외 로깅 (c) 내부 timeout 3종을 항상 같이 챙긴다.
