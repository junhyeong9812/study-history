# 자가 재배포 — 원격에서 자기 자신을 업데이트하기

> 운영 중인 에이전트가 자신을 git pull + systemctl restart 하는 패턴.
> 안전한 detach + 재기동 후 검증 (session_id) 까지.

---

## 1. 시나리오

```
브라우저 UI: "에이전트 재배포" 버튼 클릭
   ↓
관제 서버: WS로 system.redeploy_collector 명령 발송
   ↓
에이전트: 즉시 응답 ("redeploying"), 백그라운드 작업 trigger
   ↓
백그라운드 스크립트: git pull + sudo systemctl restart collector-agent
   ↓
systemd: 에이전트 프로세스 kill → 새 프로세스 시작
   ↓
새 에이전트: WS 재연결 + 새 session_id로 agent_hello
   ↓
브라우저 UI: session_id 변경 감지 → "✓ 배포 완료"
```

---

## 2. 핵심 패턴 4가지

| 패턴 | 역할 |
|------|------|
| **Sentinel Value** (`country="__self__"`) | "이 명령은 컨테이너가 아닌 에이전트 자신에게" |
| **Detach + Restart** | nohup + setsid 로 부모 죽어도 자식 살아남기 |
| **Generation ID** (session_id) | 재기동 후 **새 프로세스** 임을 증명 |
| **Fire-and-Forget** | API 응답은 즉시, 실제 재기동은 비동기 |

---

## 3. 명령 발행 측 (UI/orch)

### 3.1 UI 측 코드
```js
async function deployAgentRedeploy() {
    const username = document.getElementById('deploy-agent-user').value.trim();
    const token    = document.getElementById('deploy-agent-token').value.trim();

    // 재배포 직전 session_id 캡처 (검증용)
    const beforeSess = await fetchAgentSession();
    const beforeSession = beforeSess?.session_id || null;

    const r = await fetch('/api/migration/command', {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({
            action: 'system.redeploy_collector',
            country: '__self__',
            params: {username, token},
        }),
    });
    const d = await r.json();

    if (d.data?.status === 'success') {
        // 90초 동안 polling: session_id가 바뀌면 완료
        waitForHealth('agent', statusEl, 90, beforeSession);
    }
}

async function fetchAgentSession() {
    const r = await fetch('/api/migration/agent-status', {cache: 'no-store'});
    return r.ok ? r.json() : null;
}
```

### 3.2 polling 루프 (검증)
```js
async function waitForHealth(target, statusEl, maxSec, beforeSession) {
    const start = Date.now();
    while ((Date.now() - start) / 1000 < maxSec) {
        await sleep(2000);
        const s = await fetchAgentSession();
        if (s?.session_id && s.session_id !== beforeSession) {
            statusEl.textContent = '✓ 배포 완료';
            return;
        }
    }
    statusEl.textContent = '⚠ 완료 확인 실패';
}
```

핵심:
- `beforeSession` 을 **명령 발행 전** 에 캡처.
- 폴링 결과의 session_id 가 `beforeSession` 과 **다르면** → 새 프로세스가 떴다는 증거.
- 90초 cap → 무한 대기 방지.

---

## 4. 에이전트 측 — 명령 수신

```python
# agent: command_relay.py
async def handle_command(msg):
    action = msg.get("action", "")
    if action.startswith("system."):
        return await _handle_system_command(action, msg.get("params", {}), msg.get("request_id"), start)
    # ... country 검증 후 다른 분기
```

`system.*` 는 country 검증을 SKIP. 자기 자신 명령이니까.

```python
async def _handle_system_command(action, params, request_id, start):
    if action == "system.redeploy_collector":
        return await redeploy_collector(params, request_id, start)
    elif action == "system.health":
        return await get_self_health(...)
    # ...
```

---

## 5. 에이전트 측 — 재배포 실행

```python
# agent: system_control.py (단순화)
import asyncio
import subprocess

async def redeploy_collector(params, request_id, start):
    user = params.get("username")
    token = params.get("token")

    # 1. git pull (현재 프로세스에서)
    auth_url = f"https://{user}:{token}@gitlab.com/trademark_platform/search-app/collector-agent.git"
    proc = await asyncio.create_subprocess_exec(
        "git", "pull", auth_url, "main",
        cwd=COLLECTOR_DIR,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
    )
    stdout, stderr = await proc.communicate()
    if proc.returncode != 0:
        return _error_response(request_id, ..., f"git pull failed: {stderr.decode()}", start)

    # 2. 백그라운드로 재시작 스크립트 detach
    subprocess.Popen(
        ["nohup", "bash", "deploy/pull-and-restart.sh"],
        cwd=COLLECTOR_DIR,
        stdout=open("/tmp/redeploy.log", "w"),
        stderr=subprocess.STDOUT,
        start_new_session=True,
    )

    # 3. 즉시 응답 — 실제 재기동은 비동기
    return {
        "status": "success",
        "message": "재배포 예약됨. ~30초 후 새 session_id로 재연결됩니다.",
        "request_id": request_id,
    }
```

### 핵심 두 가지

#### `start_new_session=True`
- 자식 프로세스가 부모와 다른 session group.
- 부모(에이전트) 가 죽어도 자식은 살아남음.
- nohup + setsid 효과 (Python 측에서).

#### 즉시 응답
- API 응답을 보낸 후에야 systemd restart 가 발생해야 함.
- 만약 응답 전에 자기가 죽으면 클라이언트는 "응답 없음" 으로 인식.
- → 응답을 먼저 만들어 보내고, 그 다음 detach.

---

## 6. 백그라운드 스크립트

```bash
#!/usr/bin/env bash
# deploy/pull-and-restart.sh
set -eu

LOG_FILE="/tmp/collector-redeploy.log"

echo "[$(date -Iseconds)] redeploy start" >>"$LOG_FILE"

# API 응답이 클라이언트에 도달하도록 잠깐 대기
sleep 2

# requirements 변동 시 의존성 설치
if [ -f .venv/bin/pip ]; then
    .venv/bin/pip install -r requirements.txt >>"$LOG_FILE" 2>&1
fi

# systemctl restart (자기 자신을 죽임)
sudo -n systemctl restart collector-agent >>"$LOG_FILE" 2>&1

# 이 라인까지 도달한다는 건 systemctl 이 돌아온 뒤라는 의미.
echo "[$(date -Iseconds)] restart issued" >>"$LOG_FILE"
```

### `sleep 2` 의 가치
- 부모가 응답 보내고 클라이언트가 받기까지 마진.
- 짧으면 클라이언트가 "끊김" 으로 인식할 수도.
- 길면 UI 가 polling 시작 후 빈 응답 받음 (단 polling 은 무관함).

### `sudo -n`
- 비대화형. 패스워드 입력 못 받으면 즉시 실패.
- sudoers 에 NOPASSWD 등록 필수: `deploy-user ALL=(ALL) NOPASSWD: /bin/systemctl restart collector-agent`.

---

## 7. 새 프로세스 — session_id 발급

```python
# agent ws_client.py
async def ws_listen(command_handler):
    while True:
        try:
            async with websockets.connect(CONTROL_SERVER_WS) as ws:
                global _session_id
                _session_id = uuid.uuid4().hex  # ← 매 연결마다 새로 발급
                await ws.send(json.dumps({
                    "type": "agent_hello",
                    "agent": "collector",
                    "session_id": _session_id,
                    "pid": os.getpid(),
                }))
                # ...
```

→ systemd 에 의해 새 프로세스가 시작되면 새 session_id.
→ orch 가 새 session_id 로 `_agent_session_id` 갱신.
→ UI polling 이 변경 감지.

---

## 8. 흔한 함정

### 8.1 응답 전에 죽음
**증상**: UI 가 "재배포 요청" 후 응답 없음 → 사용자는 실패로 오해.

**원인**: nohup detach 가 너무 빠르게 systemctl restart 호출.

**해결**: 스크립트 안에 sleep 2 마진. Python 핸들러는 응답 만든 직후 즉시 return.

### 8.2 git pull 실패 시 재기동
**증상**: 새 코드 fetch 실패했는데 재기동 진행 → 동일 코드로 그냥 재시작.

**해결**:
- git pull 을 핸들러 안에서 **먼저 실행**, 실패 시 재기동 트리거 안 함.
- 위 코드의 `if proc.returncode != 0: return _error_response(...)`.

### 8.3 단순 connected 플래그로 검증
```js
// 잘못된 검증
const status = await fetchAgentSession();
if (status?.connected) { return "재배포 완료"; }  // ← 단순 재연결도 connected 가 됨
```

→ session_id 변화로 검증해야 새 프로세스임을 알 수 있음.

### 8.4 환경변수 / venv 누락
**증상**: 새 프로세스가 떴는데 의존성이 안 맞아 에러.

**해결**:
- requirements.txt 변경 시 자동 설치 (스크립트 안에 pip install).
- venv 의 python 바이너리를 systemd ExecStart 에 명시.

### 8.5 sudo password 노출
- `Environment=SUDO_PASSWORD=xxx` 하면 /proc/<pid>/environ 으로 노출.
- 차라리 NOPASSWD 권장. 구체 명령만 허용.

---

## 9. 쉽지만 위험한 변형들

### 9.1 docker-compose 로 자가 재배포
같은 패턴, systemctl 대신 `docker-compose up -d --build`. 에이전트 자체가 컨테이너에 있을 때.

### 9.2 Kubernetes 환경
ConfigMap/Image 업데이트 후 `kubectl rollout restart deployment/...`. 에이전트가 이를 트리거.

### 9.3 보안 강화
- token 을 매번 입력 받지 말고 GitOps + 별도 webhook.
- secret 회전 자동화.

---

## 10. 응용 포인트

- 자가 재배포 4단 패턴: 명령 수신 → git pull → nohup detach → 즉시 응답.
- **응답을 먼저, 죽기는 나중**. 클라이언트가 응답 받은 뒤에 자기를 죽임.
- `start_new_session=True` (Python) 또는 nohup + setsid (shell) — 부모 사망과 자식 분리.
- 검증은 단순 connected 가 아닌 session_id 변경.
- `sudo -n` + sudoers NOPASSWD 정밀 명령 — 패스워드 환경변수 회피.
- git pull 실패 시 재기동 트리거 안 함 — 동일 코드로 재시작 무의미.
