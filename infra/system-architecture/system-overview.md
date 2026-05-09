# 시스템 조감 — 다국가 검색 + 마이그레이션 + 모니터링

> 6개 구성 요소가 어떻게 짜여 있는지의 한 페이지 그림.

---

## 1. 6개 구성 요소

| # | 구성요소 | 역할 |
|---|---------|------|
| 1 | search-app (5개국 × 3서비스) | 다국가 마이그레이션 API + ES 검색 인프라 |
| 2 | collector-agent (에이전트, GPU 서버) | 컨테이너 관제, 메트릭/로그 중계, 자동 복구, 자가 배포 |
| 3 | orchestration (UI, DB 서버) | 통합 관제 UI + 스케줄러 + WebSocket 브릿지 |
| 4 | monitoring (서비스 모니터링) | 검색로그 + 블랙리스트 + 통계 + 마이그레이션 이력 |
| 5 | PostgreSQL | 영구 저장 (검색로그, 에러, 블랙리스트, 마이그레이션 이력) |
| 6 | Redis Primary/Replica | 실시간 이벤트 브로커 (Streams) + 블랙리스트 TTL 캐시 |

외부 의존:
- MariaDB: 원본 트레이드마크 데이터 (국가별 DB).
- MongoDB: 트레이드마크명 사전, 발음 사전.
- LLM API: 형태소 분리, 의미 분석.

---

## 2. 물리 배치

```
┌──────────────────────────────────────────────────────┐
│  GPU 서버                                              │
│   /data/                                           │
│     ├─ kr-search-v2/ (KR)         │
│     ├─ us-search-v2/ (US)         │
│     └─ eu-search-v2/ (EU)         │
│   /data2/                                          │
│     ├─ jp-search-v2/ (JP)         │
│     ├─ ch-search-v2/ (CH)         │
│     └─ collector-agent/  (에이전트)                       │
│                                                        │
│  systemd: collector.service                        │
│  공개 포트: 10001~10035                                │
└──────────────────────────────────────────────────────┘
                    ▲
                    │ WebSocket + HTTP
                    ▼
┌──────────────────────────────────────────────────────┐
│  DB 서버                                               │
│   /path/to/orchestration/   (FastAPI :9000)  │
│   /path/to/monitoring/      (docker-compose) │
│     ├─ monitoring-app  (:8100)                        │
│     ├─ postgres        (:5432)                        │
│     ├─ redis-primary   (:6381)                        │
│     └─ redis-replica   (:6382)                        │
│                                                        │
│  systemd: orchestration.service              │
│  공개 포트: 9000 (UI), 8100 (monitering 내부)         │
└──────────────────────────────────────────────────────┘
```

### 2.1 디스크 파티션 정책
- `/data` / `/data2` 분리 → I/O 부하 분산. 5개국 ES 가 동시 인덱싱 시 디스크 병목 완화.

### 2.2 네트워크 경계
- 내부 네트워크 (예: 99.7.x.x). WS/HTTP 인증 없음 (내부망 가정).
- 외부 노출: 오케스트레이션 9000 (UI) — Nginx 프록시 권장.

---

## 3. 컨테이너 명명/포트 맵

```
{region}-{service}
예: kr-search-engine
```

| 서버 | ES HTTP | ES Transport | Kibana | migration-api |
|------|---------|-------------|--------|--------------|
| KR | 10001 | 10031 | 10011 | 10021 |
| JP | 10002 | 10032 | 10012 | 10022 |
| CH | 10003 | 10033 | 10013 | 10023 |
| US | 10004 | 10034 | 10014 | 10024 |
| EU | 10005 | 10035 | 10015 | 10025 |

→ 1004x = ES 4번 = US, 1005x = EU. 패턴 외우면 운영 편함.

---

## 4. 데이터 흐름 3종

### 4.1 마이그레이션 플로우
```
MariaDB (원본)
   │ SQL 쿼리 (출원번호/TID 추출)
   ▼
migration-api (ProcessPoolExecutor, 국가별)
   │ bulk_index
   ▼
Elasticsearch (국가별 인덱스)
```

### 4.2 관제 플로우
```
15개 컨테이너
   │ Docker API + /orch/status HTTP
   ▼
collector-agent (10초 주기 수집)
   │ HTTP POST /api/migration/metrics/ingest
   │ WebSocket 양방향 (/ws/agent)
   ▼
orchestration
   │ /ws/migration/dashboard (broadcast)
   │ /ws/metrics (5초 push)
   ▼
브라우저 UI (4개 탭)
```

### 4.3 검색 서비스 모니터링 플로우
```
검색 서비스 (외부)
   │ Redis Streams publish
   ▼
Redis Primary (Streams)
   │ monitoring-group 컨슈머
   ▼
monitoring listener (7개 Stream × Handler)
   │ ORM
   ▼
PostgreSQL
   │
   ▼
monitoring 대시보드
```

---

## 5. 제어 평면 vs 데이터 평면

### 5.1 제어 평면
- 명령 흐름: UI → orch → agent → 컨테이너.
- 프로토콜: WebSocket (`/ws/agent`) + action 기반 RPC.
- 동기성: request_id 매칭 async response.
- 타임아웃: 300초.

### 5.2 데이터 평면
- 메트릭: agent → orch → dashboard.
- 로그: agent (docker logs -f) → orch (parser) → dashboard.
- 이벤트: 검색 서비스 → Redis Streams → monitering → PG.

### 5.3 분리 효과
- 제어 평면 장애(orch 다운) 가 데이터 평면(ES/migration-api 자체 동작) 을 막지 않음.
- agent 가 orch 와 분리 실행 → orch 재배포 중에도 agent 살아있음.

---

## 6. 핵심 설계 원칙 8가지

### 6.1 국가별 격리
5개국 ES/API/Kibana 완전 독립 compose. KR 장애 → JP/US/EU/CH 영향 없음.

### 6.2 에이전트 중앙 관제
collector-agent 1대가 15개 컨테이너 메트릭/로그/명령을 단일 WS 소켓으로 집중. orch 는 agent 만 보면 전체 상태 파악 가능.

### 6.3 서킷브레이커 자동 복구
DOWN 감지 → 3회 재시도 → phase1 쿨다운 → phase2 → open. 단조 상태 전이로 교착 방지.

### 6.4 자가 배포
collector 와 orch 둘 다 UI 에서 재배포 가능. git pull + nohup detach + systemctl restart. session_id 교체 감지로 완료 확인.

### 6.5 Backward-Compatible Protocol Extension
WS 메시지에 필드 추가 시 optional. 구버전 무시해도 깨지지 않음. 점진적 배포 가능.

### 6.6 Fire-and-Forget 브로드캐스트
느린 구독자가 핵심 경로(명령 응답 등) 를 멈추지 않도록. `asyncio.create_task(...)` 비동기 격리.

### 6.7 Event-Driven Monitoring
검색 서비스는 publish 만. 저장/집계 책임은 monitering 에만. at-least-once + 컨슈머 그룹으로 scale-out 대비.

### 6.8 Sentinel + Validation
`country="__self__"` 같은 sentinel 사용 시 모든 validation 지점이 이를 이해해야 함. 함수 시그니처로 의도 드러내기 (`_handle_system_command` 는 country 인자 없음).

---

## 7. 식별자 체계

| 식별자 | 범위 | 변경 시점 | 용도 |
|--------|------|----------|------|
| `session_id` (uuid4 hex 32자) | WS 연결 수명 | 재연결마다 | 재기동 감지 |
| `pid` | 프로세스 수명 | 재기동마다 | 디버깅 병기 |
| `request_id` (uuid4[:8]) | 단일 명령 | 매 명령마다 | async response 매칭 |
| `run_id` (uuid4) | 1회 마이그레이션 실행 | trigger 마다 | 스케줄 이력 추적 |
| `trace_id` (uuid) | 1회 검색 요청 | 검색 서비스 요청마다 | 검색로그 + 에러 연관 |
| `migration_id` | = run_id | | monitering 에서도 동일 ID |

---

## 8. 시간 동기화

- 5개국 migration-api 컨테이너: `TZ=Asia/Seoul` + `/etc/localtime:ro` 마운트.
- 오케스트레이션 스케줄러: KST 기준 cron (매 분 체크).
- 로그 타임스탬프: 모두 KST (ISO 8601 + `+09:00`).
- PostgreSQL: `timestamptz` 타입으로 UTC 저장 → 조회 시 세션 timezone 으로 변환.

---

## 9. 장애 격리 경로

| 장애 | 자동 복구 | 수동 개입 조건 |
|------|----------|-----------|
| ES 컨테이너 OOM | 서킷브레이커 자동재시작 3회 → phase2 3회 | 6회 실패 시 open |
| migration-api hang | `stop` + `restart_service` | API 응답 안 할 때 |
| collector 프로세스 crash | systemd `Restart=always` | 반복 crash 시 |
| orchestration crash | systemd `Restart=always` | 반복 crash 시 |
| WS 단절 | 지수 백오프 재연결 (3→30s) | 네트워크 단절 지속 시 |
| MariaDB 단절 | migration-api 측 재시도 | DB 복구 필요 |
| PostgreSQL 단절 | redis-py 재연결 | DB 복구 |

---

## 10. 응용 포인트

- 다국가 시스템은 격리 인스턴스 + 중앙 관제 에이전트가 표준 골격.
- 컨테이너 명명/포트 패턴을 일관되게 (예: `{국가}-{서비스}-v2`, 1000+국가코드).
- 제어/데이터 평면 분리는 장애 격리의 기본.
- 디자인 원칙은 8가지 정도로 압축해서 운영자 인수인계.
- 식별자는 명확한 범위와 변경 시점 정의.
