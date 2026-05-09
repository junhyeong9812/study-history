# Python 운영 배포 — venv + systemd + uvicorn

> 파이썬 FastAPI 서비스를 systemd 데몬으로 운영하는 표준 골격.

---

## 0. 분석 대상 코드

```bash
# orchestration/install-service.sh
#!/bin/bash
set -e

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
SERVICE_NAME="orchestration"
SERVICE_FILE="/etc/systemd/system/${SERVICE_NAME}.service"
RUN_USER="${SUDO_USER:-deploy-user}"

# venv가 있으면 venv Python, 없으면 시스템 Python
if [ -f "${SCRIPT_DIR}/.venv/bin/python3" ]; then
    PYTHON_BIN="${SCRIPT_DIR}/.venv/bin/python3"
else
    PYTHON_BIN=$(which python3 2>/dev/null || echo "/usr/bin/python3")
fi

cat > "${SERVICE_FILE}" << EOF
[Unit]
Description=Search Orchestration Dashboard
After=network.target docker.service

[Service]
Type=simple
User=${RUN_USER}
WorkingDirectory=${SCRIPT_DIR}
ExecStart=${PYTHON_BIN} -m uvicorn app.main:app --host 0.0.0.0 --port 9000 --ws wsproto
Restart=always
RestartSec=5
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable ${SERVICE_NAME}
systemctl start ${SERVICE_NAME}
```

---

## 1. venv vs 시스템 Python

```bash
if [ -f "${SCRIPT_DIR}/.venv/bin/python3" ]; then
    PYTHON_BIN="${SCRIPT_DIR}/.venv/bin/python3"
else
    PYTHON_BIN=$(which python3 ...)
fi
```

### 왜 venv?
- 시스템 Python 의 패키지를 건드리지 않음 (다른 시스템 도구가 깨질 수 있음).
- 프로젝트별 의존성 격리.
- venv 의 `bin/python3` 를 직접 지정 → systemd 가 자동으로 venv 활성화 효과.

### venv 만들기
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
deactivate
```

systemd 에서는 `source activate` 안 함. **venv 의 python 바이너리를 직접 호출** 만으로 그 venv 의 site-packages 가 sys.path 에 들어감.

### `python3 -m uvicorn` vs `.venv/bin/uvicorn`
- 둘 다 가능. 위 코드는 `python3 -m uvicorn` 으로 명시 → uvicorn 콘솔 스크립트가 venv 안에 있는지 따로 안 봐도 됨.

---

## 2. systemd 유닛 파일의 핵심 옵션

```ini
[Unit]
Description=...
After=network.target docker.service
```

### `After`
- 시작 순서 의존성. network.target / docker.service 가 시작된 후 이 서비스 시작.
- "필수 의존" 이 아니라 "순서" 만 강제. docker 가 죽어도 이 서비스는 동작 시도.
- 진짜 필수면 `Requires=docker.service`.

```ini
[Service]
Type=simple
User=${RUN_USER}
WorkingDirectory=${SCRIPT_DIR}
ExecStart=...
Restart=always
RestartSec=5
Environment=PYTHONUNBUFFERED=1
```

### `Type=simple`
- ExecStart 가 즉시 foreground 로 실행됨 (fork/detach 안 함).
- uvicorn / 일반 파이썬 프로세스의 표준.
- `Type=forking` 은 데몬화하는 프로세스용 (전통 syslog 등).

### `User=${RUN_USER}`
- 비특권 사용자로 실행. 보안 기본.
- root 로 띄워도 동작은 하지만 권장 안 됨.

### `WorkingDirectory`
- 코드 경로. 상대경로 import 가 안 깨지게.

### `Restart=always` + `RestartSec=5`
- 프로세스가 어떤 이유로든 종료되면 5초 후 재시작.
- 자동 복구의 기본.
- `RestartSec=5` 너무 짧으면 fail loop 시 cpu 폭주 → cooldown 확보.

대안:
- `Restart=on-failure` — 정상 종료 (exit 0) 면 재시작 안 함. 일반 데몬 권장.
- `RestartLimitInterval=10min` + `RestartLimitBurst=5` — 5회 실패하면 더 이상 재시작 안 함.

### `Environment=PYTHONUNBUFFERED=1`
- print/logging 의 stdout 즉시 flush.
- journald 로그가 실시간으로 보임.
- 미설정 시 버퍼링되어 종료 후에야 로그 보일 수 있음.

```ini
[Install]
WantedBy=multi-user.target
```
- `enable` 시 multi-user.target.wants/ 에 심볼릭 링크 생성.
- 부팅 후 자동 시작.

---

## 3. 자가 재배포 스크립트

```bash
# orchestration/scripts/redeploy-self.sh
#!/usr/bin/env bash
set -eu

LOG_FILE="/tmp/orchestration-redeploy.log"
SERVICE="orchestration"

echo "[$(date -Iseconds)] redeploy-self.sh start" >>"$LOG_FILE"

sleep 2  # API 응답이 클라이언트에 도달하도록 잠깐 대기

if [ -n "${SUDO_PASSWORD:-}" ]; then
    echo "$SUDO_PASSWORD" | sudo -S systemctl restart "$SERVICE" >>"$LOG_FILE" 2>&1
else
    sudo -n systemctl restart "$SERVICE" >>"$LOG_FILE" 2>&1
fi
```

### 핵심 트릭

#### `set -eu`
- `-e`: 실패 시 즉시 종료.
- `-u`: 정의 안 된 변수 사용 시 에러.
- 신뢰할 수 있는 스크립트의 표준.

#### `sleep 2` 의 의미
- API `/api/monitor/self-redeploy` 핸들러가 이 스크립트를 nohup detach 로 띄움.
- 핸들러는 즉시 응답 반환.
- 2초 마진 동안 응답이 클라이언트에 도달.
- 그 후 systemctl restart → 자기 자신을 재시작.

#### `sudo -n` (NOPASSWD)
- `-n`: non-interactive. 패스워드 입력 못 받으면 즉시 실패.
- sudoers 에 `deploy-user ALL=(ALL) NOPASSWD: /bin/systemctl restart orchestration` 등 미리 등록.
- 보안: 이 한 명령만 패스워드 없이 허용. 광범위한 NOPASSWD 금지.

#### nohup detach 로 띄우기 (호출 측 패턴)
```python
# Python 핸들러
import subprocess
subprocess.Popen(
    ["nohup", "bash", "scripts/redeploy-self.sh"],
    stdout=open("/dev/null", "w"),
    stderr=subprocess.STDOUT,
    start_new_session=True,
)
return {"status": "redeploying"}
```

`start_new_session=True` 가 핵심:
- 부모 프로세스(uvicorn)가 죽어도 자식이 살아남음.
- systemctl restart 가 부모를 죽여도 스크립트는 계속 실행.

---

## 4. 자가 재배포의 안전 패턴

### 4.1 git pull 은 미리
```python
# API 핸들러
@app.post("/api/monitor/self-redeploy")
async def self_redeploy():
    # 1. git pull (현재 프로세스에서)
    result = subprocess.run(["git", "pull"], cwd=BASE_DIR, capture_output=True)
    if result.returncode != 0:
        raise HTTPException(500, "git pull failed")

    # 2. 의존성 설치
    subprocess.run([f"{VENV}/bin/pip", "install", "-r", "requirements.txt"], check=True)

    # 3. 백그라운드로 재시작 스크립트
    subprocess.Popen([..., "scripts/redeploy-self.sh"], start_new_session=True)

    # 4. 즉시 응답
    return {"status": "redeploying"}
```

→ 새 코드 가져오기는 자기 죽기 전에. 죽은 후엔 가져올 수 없음.

### 4.2 재기동 감지 (session_id)
- 클라이언트가 응답 받기 전 session_id 캡처.
- 폴링으로 서버 상태 확인 → session_id 가 바뀌면 재배포 완료.
- 자세한 패턴은 [infra/agent-based-orchestration/](../../infra/agent-based-orchestration/) 참고.

### 4.3 롤백 가능성
- git checkout <previous-tag> 로 이전 버전.
- 다시 self-redeploy → 이전 버전으로 재시작.
- 단, requirements.txt 바뀌면 pip install 도 다시 해야 함.

---

## 5. uvicorn CLI 옵션

```
${PYTHON_BIN} -m uvicorn app.main:app --host 0.0.0.0 --port 9000 --ws wsproto
```

| 옵션 | 의미 |
|------|------|
| `app.main:app` | `app/main.py` 의 `app` 인스턴스 |
| `--host 0.0.0.0` | 모든 인터페이스 바인딩 (단 보안: 외부 노출 시 NGINX 앞에 두기) |
| `--port 9000` | 리스닝 포트 |
| `--ws wsproto` | WebSocket 라이브러리. wsproto 또는 websockets. wsproto 가 가벼움 |
| `--workers N` | 워커 수. 1 권장 (ProcessPool 직접 관리) |
| `--reload` | 개발 모드 핫리로드. 운영 X |
| `--log-level info` | 로그 레벨 |

운영에서는 `--access-log` 끄고 NGINX 측 액세스 로그에 의존하기도 함.

---

## 6. 로그 — journald

systemd 가 stdout/stderr 를 journald 로 자동 캡처.

```bash
sudo journalctl -u orchestration -f       # 실시간
sudo journalctl -u orchestration --since "1 hour ago"
sudo journalctl -u orchestration -n 100   # 최근 100줄
```

### journald → 파일로 redirect
필요하면 `StandardOutput=file:/var/log/search-app.log` 등도 가능. 하지만 보통 journald 가 충분.

### `Environment=PYTHONUNBUFFERED=1` 의 가치
- 안 쓰면 print 출력이 버퍼링되어 journald 에 늦게 도착.
- 운영 디버깅 시 가장 자주 빠뜨리는 환경변수.

---

## 7. 보안 — sudoers + 권한 분리

### sudoers 항목 추가
```
# /etc/sudoers.d/search-app
deploy-user ALL=(ALL) NOPASSWD: /bin/systemctl restart orchestration
deploy-user ALL=(ALL) NOPASSWD: /bin/systemctl status orchestration
```

→ 광범위 NOPASSWD 금지. 정확히 필요한 명령만 허용.

### 환경변수로 시크릿
- `Environment=DATABASE_URL=postgres://...` 직접 적기 X (로그에 남을 수 있음).
- `EnvironmentFile=/etc/search-app/.env` 로 별도 파일 + 0600 권한.

```ini
[Service]
EnvironmentFile=/etc/search-app/.env
```

---

## 8. 응용 포인트

- venv 의 python 바이너리 직접 호출 → activate 불필요.
- systemd `Type=simple` + `Restart=always` + `RestartSec=5` 가 표준 골격.
- `Environment=PYTHONUNBUFFERED=1` 거의 필수 (실시간 로그).
- 자가 재배포는 git pull → pip install → nohup detach 스크립트 + sleep 2 + sudo restart 패턴.
- sudoers 는 정확한 명령만 NOPASSWD.
- 시크릿은 EnvironmentFile + 0600.
- journalctl -u SERVICE -f 가 운영의 lifeline.
