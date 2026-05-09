# 관제 평면 vs 데이터 평면 분리

> 한 시스템 안에서 "명령을 흘리는 길" 과 "데이터를 흘리는 길" 을 의도적으로 분리.
> 한 평면이 죽어도 다른 평면이 살아남게 하는 설계.

---

## 1. 두 평면의 정의

### 관제 평면 (Control Plane)
- **무엇**: 시스템에 무엇을 하라고 시키는 명령 흐름.
- **예시**: "kr 마이그레이션 시작", "es 재시작", "에이전트 재배포".
- **특성**: 비교적 드물게 발생, request-response 패턴, 결과 회신 필요.

### 데이터 평면 (Data Plane)
- **무엇**: 시스템이 만들어내는 운영 데이터의 흐름.
- **예시**: 메트릭 push, 로그 stream, 진행률 update.
- **특성**: 지속적 high throughput, push 위주, 결과 회신 불필요.

---

## 2. 본 시스템의 평면 매핑

```
관제 평면:
  UI → orch (HTTP /api/migration/command)
  orch → agent (WS /ws/agent, request_id 매칭)
  agent → migration-api (HTTP /orch/start)
  agent → docker (subprocess docker compose)

데이터 평면:
  agent → orch (HTTP POST /metrics/ingest, 10초)
  agent → orch (WS log_stream)
  orch → UI (WS /ws/migration/dashboard, broadcast)
  orch → UI (WS /ws/metrics, 5초 push)

별도 데이터 평면 (분리된 모니터링):
  검색 서비스 → Redis Streams → monitering → PostgreSQL
```

---

## 3. 분리의 가치

### 3.1 한 평면 장애의 격리

| 장애 | 관제 평면 영향 | 데이터 평면 영향 | 비즈니스 영향 |
|------|--------------|---------------|------------|
| orch 다운 | 명령 불가 | 메트릭 push 실패 | 컨테이너 자체 동작은 정상 |
| agent 다운 | 컨테이너 제어 불가 | 메트릭/로그 끊김 | 컨테이너 자체 동작은 정상 |
| WS 단절 | 명령 회신 지연/실패 | log_stream 끊김 | 컨테이너 동작 정상 |
| HTTP push 실패 | 영향 없음 | 메트릭 누락 | 모니터링 누락만 |
| 컨테이너 다운 | 명령은 가지만 실패 응답 | 해당 컨테이너 메트릭 0 | 해당 기능 다운 |

핵심: **관제/데이터 평면 어느 쪽이 죽어도 컨테이너의 비즈니스 로직은 살아있다**.

### 3.2 백프레셔 분리

데이터 평면이 폭주해도 관제 평면 막히지 않게:
- 로그 broadcast → fire-and-forget + 타임아웃.
- 좀비 클라이언트의 send hang 이 명령 처리에 영향 없음.

```python
# WS 수신 루프 안
if msg_type == "log_stream":
    asyncio.create_task(broadcast_to_dashboard(msg))  # 데이터 평면, 비동기
    continue

# 명령 응답
request_id = msg.get("request_id")
if request_id and request_id in _pending_responses:
    fut.set_result(msg)  # 관제 평면, 동기 매칭
    continue
```

→ 같은 WS 채널을 쓰지만, 처리는 분리.

### 3.3 보안 경계 분리

- 관제 평면: 인증 강화 가능 (admin 권한, 토큰 등).
- 데이터 평면: 메트릭 push 는 inbound 만 보호.
- 외부 노출 시 관제 평면만 화이트리스트 IP, 데이터 평면은 더 자유롭게.

---

## 4. 평면별 프로토콜 선택

### 관제 평면
- **WebSocket + request_id**: 양방향, 응답 매칭 필요.
- **HTTP REST**: stateless 명령 (예: GET /agent-status).
- 핵심: 결과 회신 + 오류 보고.

### 데이터 평면
- **HTTP push**: 안정적, keep-alive 로 효율.
- **WebSocket broadcast**: 실시간 push (UI 업데이트).
- **Redis Streams**: 결과적 영속 + at-least-once.
- 핵심: throughput + 손실 허용성.

---

## 5. 별도 데이터 평면 (Redis Streams 모니터링)

검색 서비스의 모니터링은 본 관제 시스템과 **완전히 분리된 데이터 평면**:

```
검색 서비스 (외부)
    │ XADD stream:search:log
    ▼
Redis (Primary)
    │ XREADGROUP monitoring-group
    ▼
monitoring
    │ ORM
    ▼
PostgreSQL
```

이 평면은:
- 관제 평면 (orchestration) 과 **무관**. orch 가 죽어도 검색 모니터링은 정상.
- 검색 서비스가 발행자 → monitering 이 독립 소비자.
- monitering 이 죽어도 검색 서비스는 정상 (Redis 가 buffer 역할).

→ "관제 시스템 / 운영 데이터 / 사용자 데이터" 3개 도메인 평면이 모두 분리된 구조.

---

## 6. 흔한 안티패턴 — 평면 혼합

### 6.1 명령에 메트릭 첨부
```python
# 잘못된 예
async def send_command(action, country):
    response = await ws.send({"action": action, "metrics": current_metrics})
    # 명령 페이로드에 메트릭 같이 보냄
```

→ 메트릭 폭주 시 명령 자체가 무거워짐. 평면 분리 위배.

### 6.2 메트릭 푸시에 명령 결과 첨부
```python
# 잘못된 예
metrics_payload["last_command_result"] = "..."
```

→ 메트릭 소비자(분석/저장) 가 명령 결과 처리 로직까지 알아야 함.

### 6.3 같은 채널에 동기 send 혼합
```python
# 잘못된 예 — broadcast 와 명령 응답이 같은 await 체인에
await ws.send_json(broadcast_msg)
await ws.send_json(command_response)
```

→ broadcast 가 좀비 클라이언트로 hang 하면 명령 응답도 hang.

### 6.4 fix
- 단일 채널 다중화는 OK, 단 처리는 분리 (`asyncio.create_task` 로 broadcast 격리).
- 또는 채널 자체를 분리 (관제용 WS + 데이터용 WS).

---

## 7. 평면 분리의 한계

### 7.1 진단 시 합치고 싶을 때
- "이 명령이 실행되는 동안의 메트릭 변화" 같은 분석은 두 평면의 데이터 결합 필요.
- 해결: 공통 trace_id / run_id 를 양쪽에 심음.

### 7.2 인증/권한이 평면별로 다르면
- 관제는 admin only, 데이터는 readonly.
- 같은 사용자가 두 권한 가지면 OK. 다르면 분리 운영 부담.

### 7.3 단일 채널 비용
- WS 한 개로 다중화하면 송신/수신 동시성 안전 + 좀비 격리 등 복잡함.
- 분리하면 채널이 N배 → 클라이언트가 모두 관리.

---

## 8. 응용 포인트

- 시스템 설계 첫 그림에서 "명령" 과 "데이터" 를 색깔 다르게 그린다.
- 한 평면 장애가 다른 평면을 죽이지 않도록 **fire-and-forget + timeout** 으로 격리.
- 메트릭/로그는 push, 명령은 RPC.
- 외부 모니터링은 별도 평면 (Redis Streams 같은 영속 큐) 로 분리.
- 진단 결합이 필요하면 trace_id 를 양 평면에 심음.
- 단일 채널 다중화 vs 채널 분리 — 시스템 규모와 안전 요구에 따라 결정.
