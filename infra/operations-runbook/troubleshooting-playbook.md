# 트러블슈팅 플레이북 — 실전 사례 모음

> 다국가 컨테이너 운영 + 에이전트 + 모니터링 시스템에서 실제로 발생한 이슈와 진단 절차.
> 운영 인수인계용.

---

## 1. 컨테이너 자동 재시작 미동작

### 증상
```
Container DOWN detected: kr-search-engine (status=unknown)
Auto-restart FAILED: ... no such service: kr-elasticsearch
```

### 진단
```bash
sudo journalctl -u collector-agent -n 100 --no-pager | \
  grep -E "(Container DOWN|Auto-restart|Circuit breaker)"
```

### 케이스 분기

#### Case A: `no such service: ...` 에러
- **원인**: docker compose 서비스 이름 변경 (예: suffix 변경) 후 자동화 코드 미업데이트.
- **해결**: 자동화 코드의 서비스 이름 매핑 테이블 갱신 + 재배포.
- **예방**: 인프라 변경 시 자동화 코드의 문자열 의존성 일제 점검.

#### Case B: `Circuit breaker OPEN (permanent)`
```
Circuit breaker OPEN: kr:elasticsearch (phase2 exhausted)
```
- **원인**: phase1 + phase2 합쳐 6회 실패 → 영구 차단.
- **해결**: 인메모리 상태 리셋 → `sudo systemctl restart collector-agent`.
- **근본 원인 조사 필수**: 왜 6회 연속 실패했나 (OOM, 설정 오류, DB 끊김).

#### Case C: `Circuit breaker phase2 blocked (remaining 음수)`
```
Circuit breaker cooldown: kr:elasticsearch (-62391s remaining)
```
- **원인**: 비단조 전이 버그 (외부 메트릭 영구 false 로 phase 전이 못 함).
- **해결**: 단조 전이로 패치된 버전 배포.

### 복구 후 확인
```bash
sudo docker ps | grep kr-search-engine
curl http://localhost:10001/_cluster/health
sudo journalctl -u collector-agent -n 20 | grep "Auto-restart SUCCESS"
```

---

## 2. UI "재배포 완료 확인 실패"

### 증상
"에이전트 재배포" 버튼 클릭 → 실제 재기동은 성공했는데 UI 는 실패 표시.

### 진단

#### 2.1 서버 응답 확인
```bash
curl -s http://localhost:9000/api/migration/agent-status | python3 -m json.tool
```

기대 응답:
```json
{"connected": true, "session_id": "abc...", "pid": 1234}
```

- `session_id` 필드 없음 → orch 구버전. session-id 패치 포함 빌드 배포.
- `session_id` null → agent 구버전. agent 재배포 필요.

#### 2.2 브라우저 캐시
가장 흔한 원인. Ctrl+Shift+R (하드 리프레시).

#### 2.3 첫 사이클 정상
새 코드 배포 직후 첫 번째 재배포 시도는 정상적으로 "실패 표시" 일 수 있음:
- 구 agent 가 session_id 안 보내고 있을 때 캡처한 `beforeSession=null`.
- 두 번째 클릭부터 정상.

### 실제 로그 체크
```bash
# agent
sudo journalctl -u collector-agent --since "2 minutes ago" | grep "WebSocket connected"
# 기대: "WebSocket connected (session=..., pid=...)"

# orch
sudo journalctl -u orchestration --since "2 minutes ago" | grep "Agent hello"
```

---

## 3. `system.*` 명령이 `Unknown country: __self__` 로 실패

### 증상
에이전트 재배포 / journalctl 스트리밍 등 `system.*` 명령이 전부 실패.

### 원인
agent command_relay 의 분기 순서 버그 — `system.*` 체크보다 country 검증이 먼저 실행되어 sentinel 값이 "Unknown country" 로 거부됨.

### 해결
- `system.*` 분기를 country 검증 **이전** 에 배치한 빌드 배포.
- 단기 회피: 영향받은 명령 사용 안 함.

자세한 패턴은 design-patterns-applied/sentinel-value-validation.md.

---

## 4. Redis Streams pending 누적

### 증상
모니터링 대시보드의 메트릭이 점점 지연됨. 새 이벤트 처리는 되지만 pending 카운트 증가.

### 진단
```bash
# Redis CLI
XINFO GROUPS stream:search:log
# pending 필드 확인

XPENDING stream:search:log monitoring-group
# 최대 idle time, consumer 분포
```

### 가능한 원인

#### 4.1 핸들러 영구 실패 (poison message)
- 특정 메시지가 항상 같은 예외 → ACK 안 함 → 영원히 pending.
- **해결**: 핸들러 로그에서 반복되는 메시지 ID 찾아 수동 XACK 또는 코드 패치.

#### 4.2 PostgreSQL 일시 장애
- 핸들러가 INSERT 시점에 DB 끊김 → 예외 → no-ACK.
- DB 복구 후 자동 재처리.

#### 4.3 컨슈머 죽음 후 메시지 인계 안 됨
- 컨슈머 hostname 이 바뀌면 이전 컨슈머의 pending 은 다른 컨슈머가 자동 인계 안 됨.
- **해결**: `XCLAIM` 으로 다른 컨슈머가 가져가게 (코드 도입 필요).

---

## 5. WebSocket 단절 반복

### 증상
agent 가 자주 disconnect/reconnect 반복. 메트릭 끊김 빈번.

### 진단
```bash
sudo journalctl -u collector-agent | grep -E "WebSocket|Reconnect"
```

### 가능한 원인

| 원인 | 신호 | 해결 |
|------|------|------|
| 네트워크 미들박스 idle drop | 일정 시간 후 단절 반복 | ping_interval 활성화 또는 application keepalive |
| orch 측 timeout | "Connection closed" + close code 1011/1012 | orch 코드의 timeout 설정 확인 |
| 메시지 폭주 → buffer 가득 | "buffer full" 류 에러 | 클라이언트 측 receive 빠르게, 또는 메시지 빈도 줄이기 |
| DNS 캐시 stale | "Name or service not known" | nscd / systemd-resolved 재시작 |

---

## 6. orch 재기동 시 in-flight 명령 손실

### 증상
orch 재배포 직후 사용자가 보낸 명령이 timeout 으로 실패.

### 원인
- 명령 발행 시 `_pending_responses[id] = fut` 가 메모리에만 있음.
- orch SIGTERM → 메모리 리셋.
- 그 사이 들어온 응답은 매칭 못 함.

### 완화
- 재배포 시간을 짧게 유지 (uvicorn 시작 5초 이내).
- UI 측 timeout 후 재시도 정책.
- 영속 큐 (Redis) 로 명령 발행을 옮기면 손실 방지 가능 — 코드 변경 필요.

---

## 7. PostgreSQL 풀 고갈

### 증상
```
sqlalchemy.exc.TimeoutError: QueuePool limit of size 10 overflow 20 reached
```

### 원인
- 핸들러가 세션 commit/close 안 하고 빠짐.
- 또는 동시성 폭증.

### 진단
```sql
SELECT count(*), state FROM pg_stat_activity GROUP BY state;
```

idle in transaction 가 많으면 → 누군가 트랜잭션 안 끝냄.

### 해결
- 코드의 `async with session.begin()` 또는 `await s.commit()` 명시 확인.
- 풀 크기 임시 증가 (`pool_size=20, max_overflow=40`).
- 근본 해결: 누수 위치 찾아서 패치.

---

## 8. 마이그레이션 hang (3시간 timeout)

### 증상
스케줄러 로그:
```
Migration timed out after 3 hours: kr (run_id=abc...)
```

### 원인 후보
- migration-api 가 실제로 hang (배치 무한 대기).
- ProcessPoolExecutor 워커 OOM → BrokenProcessPool.
- DB 측 데드락.

### 진단
```bash
# 워커 상태
sudo docker exec kr-migration-api-v2 ps aux | grep python

# 해당 컨테이너 로그
sudo docker logs --tail 200 kr-migration-api-v2
```

### 해결
- 강제 재시작: `POST /orch/stop` + `POST /orch/start` (resume=True).
- 체크포인트가 있으면 마지막 위치부터 재개.

---

## 9. Disk full

### 증상
- 컨테이너 로그 무한 증가 → 디스크 포화.
- ES 가 read-only mode (watermark).

### 진단
```bash
df -h
du -sh /var/lib/docker/containers/*  # 컨테이너 로그
```

### 해결
- `docker system prune -a` (주의: 사용 안 하는 이미지 제거).
- ES watermark 조정 또는 디스크 증설.
- 로그 rotation 설정 (`/etc/docker/daemon.json`):
  ```json
  {"log-opts": {"max-size": "100m", "max-file": "3"}}
  ```

---

## 10. 운영 체크리스트 (배포 전후)

### 배포 전
- [ ] 변경 commit 의 영향 범위 확인.
- [ ] 의존성 (requirements.txt) 변경 여부.
- [ ] DB 마이그레이션 (alembic) 필요 여부.
- [ ] 롤백 계획 (이전 commit hash, 디스크 백업).

### 배포 중
- [ ] git pull 성공 확인.
- [ ] 의존성 설치 (`pip install -r requirements.txt`).
- [ ] systemctl restart 후 service active 확인.
- [ ] journalctl 첫 30초 에러 없는지.

### 배포 후
- [ ] 헬스체크 엔드포인트 200.
- [ ] 핵심 API 1~2개 직접 호출 검증.
- [ ] WebSocket 연결 확인 (브라우저 개발자도구).
- [ ] 5분 후 메트릭 정상 흐름 확인.
- [ ] 1시간 후 로그 review.
