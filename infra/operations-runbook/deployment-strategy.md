# 배포 전략 — 자가 배포 + 수동 배포 + 안전 장치

> 운영 중 시스템을 업데이트하는 패턴. 자가 배포 UI / SSH 수동 / 롤백.

---

## 1. 두 가지 배포 모드

### 1.1 UI 자가 배포 (권장)
- 브라우저에서 클릭으로 배포.
- GitLab PAT 입력 → orch 가 agent 에 명령 → agent 가 자기 자신 git pull + restart.
- 사용자 친화 + 비기술자도 가능.
- 단점: orch 자체가 고장나면 못 씀.

### 1.2 SSH 수동 배포 (백업)
- 운영자가 직접 SSH 접속.
- git pull + systemctl restart 명시적 실행.
- 항상 가능.
- UI 자가 배포 실패 시 fallback.

---

## 2. UI 자가 배포 흐름

```
브라우저 UI: PAT 입력 + 재배포 버튼
   ↓
orch: POST /api/migration/command (action=system.redeploy_collector, params={user, token})
   ↓
agent: git pull 실행 (auth URL with token)
   ↓
agent: pip install -r requirements.txt (변경 시)
   ↓
agent: nohup detach + sudo systemctl restart
   ↓
agent: 새 프로세스 + 새 session_id
   ↓
UI: session_id polling으로 완료 감지 → "✓ 배포 완료"
```

자세한 패턴은 [agent-based-orchestration/self-redeployment.md](../agent-based-orchestration/self-redeployment.md).

---

## 3. SSH 수동 배포 절차

```bash
# 1. SSH 접속
ssh user@host
cd /path/to/project

# 2. git pull
git pull origin main
# (PAT 입력)

# 3. 의존성 변경 시
.venv/bin/pip install -r requirements.txt

# 4. 재시작
sudo systemctl restart {service-name}

# 5. 기동 확인
sudo systemctl status {service-name}
sudo journalctl -u {service-name} -n 50 --no-pager

# 6. 헬스체크
curl http://localhost:9000/health
```

---

## 4. GitLab PAT (Personal Access Token)

### 4.1 발급
1. GitLab → 우상단 아바타 → Edit Profile.
2. Access Tokens.
3. Add new token:
   - Name: 의미 있는 식별자 + 만료일.
   - Scope: `read_repository` (pull 만) 또는 `write_repository`.
4. 한 번만 표시되는 토큰 저장.

### 4.2 사용
```bash
git clone https://oauth2:<gitlab-token>@gitlab.com/group/repo.git
# 또는
git pull   # 후 username=oauth2, password=<gitlab-token>
```

### 4.3 자격증명 영속화
```bash
git config --global credential.helper store        # 평문 저장 (주의)
git config --global credential.helper cache --timeout=86400  # 24h 캐시
```

운영 권장:
- store 는 `.git-credentials` 평문 → 권한 0600 + 호스트 보안 신뢰 환경에서만.
- cache 가 더 안전 (메모리 only).

---

## 5. 롤백 전략

### 5.1 git 기반 롤백
```bash
# 직전 commit 으로
git checkout HEAD~1

# 특정 commit
git checkout abc1234

# 그 후 재시작
sudo systemctl restart {service}
```

### 5.2 의존성도 롤백 필요할 때
```bash
git checkout abc1234
pip install -r requirements.txt --force-reinstall  # 이전 버전 복원
```

### 5.3 DB 마이그레이션 후 롤백
- alembic 사용 시: `alembic downgrade -1` 또는 `alembic downgrade <revision>`.
- 문제: 데이터 손실 가능 (DROP COLUMN 등).
- 안전: 다운타임 허용 변경은 forward-compatible 으로 작성.

### 5.4 컨테이너 이미지 롤백
```bash
docker compose -f docker-compose.yml up -d --no-build
# 이전 이미지 tag 로 docker-compose.yml 변경 후
```

---

## 6. Forward-Compatible Migration 패턴

DB 스키마 변경 시 다운타임 없이:

### 패턴 1: Expand-Migrate-Contract
1. **Expand**: 새 컬럼 추가 (구 버전 코드는 무시).
2. **Migrate**: 데이터 백필.
3. **Contract**: 구 컬럼 제거 (구 버전 코드 모두 사라진 후).

### 패턴 2: 두 컬럼 동시 운영
- 한동안 old + new 두 컬럼 모두 쓰기.
- 새 코드는 new 만 읽기.
- 안정 후 old 제거.

→ 한 번의 배포로 끝나지 않음. 보통 2~3 단계 배포.

---

## 7. 배포 안전 장치

### 7.1 헬스체크 자동 검증
```bash
# 배포 후 자동 검증 스크립트
sudo systemctl restart service
sleep 5
if curl -fs http://localhost:9000/health > /dev/null; then
    echo "OK"
else
    echo "FAIL — rolling back"
    git checkout HEAD~1
    sudo systemctl restart service
    exit 1
fi
```

### 7.2 Canary 배포 (다국가 환경)
- 한 국가만 먼저 배포 → 1시간 모니터링 → 문제 없으면 나머지 국가.
- 문제 발생 시 그 국가만 롤백 → 다른 국가 영향 없음.
- 격리 인스턴스의 가치.

### 7.3 Blue-Green / Rolling
- 단일 인스턴스 환경에서는 짧은 다운타임 감수.
- 다중 인스턴스 + LB 면 rolling 가능.

---

## 8. systemd 의 재시작 안전 장치

```ini
[Service]
Restart=always
RestartSec=5
StartLimitInterval=600
StartLimitBurst=5
```

- 600초 안에 5번 실패하면 더 이상 재시작 안 함.
- fail loop 차단.
- 운영자 알림 필요 (별도 모니터링).

```bash
# 재시작 한도 도달 후
sudo systemctl reset-failed {service}  # 카운터 리셋
sudo systemctl start {service}
```

---

## 9. 시크릿 관리

### 9.1 Environment 파일 분리
```
/etc/myapp/.env  (root only, 0600)
DATABASE_URL=...
OPENAI_API_KEY=...
```

systemd:
```ini
EnvironmentFile=/etc/myapp/.env
```

→ 코드/git 에서 시크릿 분리.

### 9.2 sudoers 좁게 NOPASSWD
```
deploy-user ALL=(ALL) NOPASSWD: /bin/systemctl restart collector-agent
```

→ 정확한 명령만 패스워드 없이 허용.

### 9.3 GitLab PAT 회전
- 만료일 설정 + 사전 알림.
- 회전 시 이전 토큰 폐기 + 운영 시스템 재인증.

---

## 10. 응용 포인트

- 자가 배포 (UI) 는 편의, SSH 수동은 backup. 둘 다 준비.
- git pull 은 자기 죽기 전에 — 죽은 후엔 가져올 수 없음.
- DB 마이그레이션은 forward-compatible 단계로.
- systemd `StartLimitBurst` 로 fail loop 차단 + 외부 알림.
- 시크릿은 EnvironmentFile + 0600.
- 다국가/다인스턴스는 Canary 배포.
- 롤백 절차는 평소에 한 번 연습.
