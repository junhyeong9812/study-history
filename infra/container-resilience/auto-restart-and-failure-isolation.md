# 자동 재시작 + 장애 격리

> 컨테이너 다운을 감지하고 자동으로 살리되, 폭주(restart loop) 와 cascading failure 를 차단하는 전체 흐름.

---

## 1. 다층 방어 (Defense in Depth)

```
Layer 1: Docker 컨테이너 격리         (국가별 compose 파일 분리)
Layer 2: 서킷브레이커                  (agent health_monitor)
Layer 3: systemd Restart=always       (agent / orch 프로세스 자체)
Layer 4: WS 재연결 + 지수 백오프      (agent ↔ orch)
Layer 5: HTTP 재시도 + 버퍼링         (agent push_metrics)
Layer 6: Redis Streams 영속 + 재처리  (monitering)
```

각 레이어는 다른 종류의 실패에 대응. 한 레이어가 부족해도 다른 레이어가 받침.

---

## 2. 컨테이너 자동 재시작 흐름

```
T=0     Container OOM kill
T=+10s  agent 10초 사이클 → DOWN 감지 (docker_collector)
T=+10.1s death_reason 조회 (docker inspect → OOMKilled=true)
T=+10.2s 서킷브레이커 _check_circuit_breaker → "allow"
T=+10.3s sudo docker compose up -d kr-elasticsearch-v2 (0.6s)
T=+11s   ES 재기동 중 → 검색 불가
T=+20~60s ES cluster status green
T=+~60s  정상 복귀
```

### 핵심 단계
1. **상태 비교**: `_prev_status` 와 비교. 같은 상태 연속이면 스킵 (중복 알림 방지).
2. **death_reason**: docker inspect 로 종료 원인 (OOM/exit code).
3. **서킷브레이커 check**: 폭주 차단.
4. **재시작 실행**: docker compose up -d (멱등 — 이미 떠 있으면 무시).
5. **history 기록**: `_record_restart` 로 슬라이딩 윈도우.

---

## 3. 단일 컨테이너 장애의 영향 격리

```
컨테이너 5개국 × 3서비스 = 15개
   │
   각자 독립된 docker-compose.yml
   │
   KR ES 다운  ─→  KR 검색만 영향
                  JP/US/EU/CH 정상
```

격리 수단:
- **국가별 compose 파일**: `kr-search-v2/docker-compose.yml`.
- **포트 분리**: KR (10001), JP (10002), ...
- **데이터 디렉토리 분리**: 각자 별도 volume.
- **Docker network**: 같은 호스트지만 nerwork 단위 격리.

→ 한 국가의 컨테이너 OOM 이 다른 국가에 영향 없음.

---

## 4. 자기 죽음의 격리 — systemd Restart=always

```ini
[Service]
Type=simple
Restart=always
RestartSec=5
```

- 어떤 이유로든 프로세스 종료 → 5초 후 재시작.
- agent crash → 자동 재기동 (단 인메모리 상태 손실).
- orch crash → 자동 재기동.

### Restart Limit
```ini
StartLimitInterval=600
StartLimitBurst=5
```

→ 600초 안에 5번 실패하면 더 이상 재시작 안 함. 무한 fail loop 차단.

운영자 알림 필요:
- `systemctl status` 가 "failed" 로 정착.
- journal 에 의존성 문제/설정 문제 등 명시.

---

## 5. WS 재연결 — 통신 단절 자동 복구

```python
async def ws_listen(command_handler):
    backoff = 3
    max_backoff = 30
    while True:
        try:
            async with websockets.connect(URL) as ws:
                backoff = 3  # 성공 시 리셋
                # ... 메시지 루프
        except (ConnectionClosed, ConnectionRefusedError, Exception) as e:
            logger.warning(f"WS error: {e}")
        await asyncio.sleep(backoff)
        backoff = min(backoff * 2, max_backoff)
```

- 단절 시 지수 백오프 (3 → 6 → 12 → 24 → 30).
- 성공 연결 시 backoff 리셋 (잠깐 깜박임 후 다시 안정 시 빠른 재시도).
- max cap → 영원히 멈추지 않음.

자세한 패턴은 [phython/websocket-streaming/reconnect-backoff.md](../../phython/websocket-streaming/reconnect-backoff.md).

---

## 6. HTTP push 실패 — 버퍼링 + 재시도

agent 측 metrics push:
```python
# transport/pusher.py (개념)
_buffer: deque = deque(maxlen=100)  # 최근 100개 push 버퍼

async def push_metrics(payload):
    try:
        await httpx_client.post(URL, json=payload, timeout=5)
        # 성공 → 버퍼의 미전송 분도 함께 시도
        while _buffer:
            buffered = _buffer.popleft()
            try:
                await httpx_client.post(URL, json=buffered, timeout=5)
            except Exception:
                _buffer.appendleft(buffered)
                break
    except Exception:
        _buffer.append(payload)  # 다음 회차에 재시도
```

→ orch 가 일시 다운돼도 agent 는 메트릭 잃지 않음.

한계:
- maxlen=100 초과하면 오래된 메트릭 유실.
- 본격적인 영속 buffer 가 필요하면 Redis/SQLite 사용.

---

## 7. Redis Streams — 모니터링 측 영속 버퍼

```
검색 서비스 → XADD → Redis (maxlen=100K)
                       │
                       └── monitering: XREADGROUP + XACK
```

- monitering 다운 동안에도 Redis 가 buffer.
- 재기동 시 pending 복구 → 손실 없음.
- maxlen 까지는 안전.

---

## 8. 복합 장애 시나리오

### 8.1 ES 다운 + collector 다운 동시
```
ES 다운 → collector 감지 전 collector 도 crash
→ systemd 가 collector 5초 후 재시작
→ 재기동 후 서킷브레이커 인메모리 리셋 (closed)
→ ES 여전히 다운 → DOWN 감지 → 재시작 시도
→ 성공 or 실패
```

**관찰**: collector 재기동 덕분에 **이전 세션의 open 상태가 자동 리셋** → 복구 기회 추가.

이건 양날의 검:
- 좋은 면: 영구 open 에서도 collector 재기동으로 한 번 더 시도.
- 나쁜 면: 진짜 문제 있는 컨테이너를 무한 재시작 시도 가능.

→ open 상태 진입 시 외부 알림 (Slack/email) 으로 운영자 개입.

### 8.2 orch 재배포 중 명령
```
T=0    UI 가 명령 발송 → orch /api/migration/command
T=+0.5 orch 가 send_agent_command, _pending_responses[id] = fut
T=+1   orch 재배포 트리거 → SIGTERM
T=+1.5 orch 종료, fut 사라짐, agent WS 끊김
T=+5   orch 재시작
T=+6   agent 재연결 + agent_hello
T=+6   원래 명령은 응답 못 받음 — UI 측 timeout
```

→ 재배포 시 in-flight 명령은 손실. UI 측에서 retry 정책.

---

## 9. 인시던트 학습 (실제 사례)

### 9.1 42시간 미동작 (2026-04-17)
- 원인: 서킷브레이커 비단조 전이 + 메트릭 영구 false → 영구 phase2_cooldown.
- 영향: KR ES 가 42시간 다운 상태 + 자동 복구 안 됨.
- 수정: 시간 기반 단조 전이로 분리.
- 학습: **전이 조건에 외부 입력 AND 금지**.

### 9.2 자동 재시작 컨테이너 이름 불일치 (2026-04-20)
- 원인: docker compose v2 의 서비스 이름 변경 (suffix `-v2` 누락).
- 영향: `docker compose up` 이 잘못된 서비스 이름으로 실패 → 자동 재시작 미동작.
- 수정: `_SERVICE_TO_CONTAINER` 매핑 + suffix 일치.
- 학습: **인프라 변경 시 자동화 코드의 문자열 의존성 일제 점검**.

---

## 10. 응용 포인트

- 컨테이너 자동 재시작 = (감지 → 원인 조회 → 서킷브레이커 → 실행 → 기록) 5단계.
- 폭주 차단은 서킷브레이커, 자기 사망은 systemd Restart, 통신 단절은 지수 백오프.
- 영속 buffer (HTTP push 실패 / Redis Streams) 는 outage 시 데이터 보호.
- 복합 장애 시 컴포넌트 재기동이 stuck 상태 자동 리셋 — 의도적 부작용.
- 모든 자동 복구는 멱등이어야 안전 (재시작은 일반적으로 멱등).
- open 상태 등 자동 복구 한계 도달 시 외부 알림 필수.
