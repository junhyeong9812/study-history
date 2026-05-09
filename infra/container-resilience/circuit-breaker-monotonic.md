# Circuit Breaker — Monotonic State Transition

> Nygard 의 Circuit Breaker 패턴에서 **상태 전이 조건이 외부 입력(메트릭) 에 의존하면 발생하는 교착(deadlock)** 을 시간 기반 단조 전이로 해결한 변형.
> 운영에서 42시간 자동 재시작 미동작 인시던트로 발견.

---

## 1. Circuit Breaker 기본 모델 (Nygard 2007)

### 1.1 세 상태
| 상태 | 의미 | 요청 처리 |
|------|------|----------|
| **CLOSED** | 정상 | 요청 통과, 실패 카운트 |
| **OPEN** | 차단 | 요청 즉시 실패 |
| **HALF_OPEN** | 시험 | 제한된 요청 통과, 성공 시 CLOSED, 실패 시 OPEN |

### 1.2 전이
```
CLOSED ─ N회 연속 실패 ──→ OPEN
OPEN ─ 타임아웃 경과 ──→ HALF_OPEN
HALF_OPEN ─ 성공 ──→ CLOSED
HALF_OPEN ─ 실패 ──→ OPEN
```

### 1.3 목적
- cascading failure 차단.
- fast fail (불필요한 재시도 회피).
- 자동 복구 시도.

---

## 2. 본 시스템의 4단계 변형

### 2.1 상태 매핑
| 상태 | 의미 | 대응 표준 |
|------|------|----------|
| `closed` | 정상 (재시작 허용) | CLOSED |
| `phase1_tripped` | 1단계 트립 (10분 쿨다운) | OPEN |
| `phase2` | 2단계 (3회 추가 기회) | HALF_OPEN (확장) |
| `open` | 영구 차단 (수동 개입) | OPEN (terminal) |

### 2.2 phase2 의 의도
- 표준 HALF_OPEN: **단일 시도** 후 성공/실패로 결정.
- phase2: **3회 시도** 까지 허용 → 단일 샘플 오판 회피.
- 컨테이너 재시작은 멱등 → 여러 번 해도 안전.

### 2.3 파라미터
| 상수 | 값 | 의미 |
|------|---|------|
| `PHASE1_WINDOW` | 300s | 슬라이딩 윈도우 |
| `PHASE1_THRESHOLD` | 3 | 윈도우 내 실패 임계 |
| `PHASE2_COOLDOWN` | 600s | phase1 → phase2 대기 |
| `PHASE2_THRESHOLD` | 3 | phase2 내 시도 허용 |

---

## 3. 버그 — 비단조 전이로 교착

### 3.1 버그 코드
```python
if state["phase"] == "phase1_tripped":
    elapsed = now - state["phase1_trip_time"]
    if elapsed < PHASE2_COOLDOWN:
        return "phase2_cooldown"

    # ⚠ 전이 조건에 외부 입력 AND
    if _server_metrics_healthy():
        state["phase"] = "phase2"
        return "allow"
    else:
        return "phase2_cooldown"   # 전이 안 함
```

### 3.2 교착 시나리오
- phase1_tripped 진입 (t0).
- t0+600s: 쿨다운 만료, 메트릭 체크.
- ES 다운 직후라 메모리 95% 넘음 → `_server_metrics_healthy() == False`.
- 상태 전환 안 됨 → `phase2_cooldown` 반환.
- `phase1_trip_time` 은 t0 그대로.
- 다음 루프: 메트릭 여전히 나쁨 → 또 phase2_cooldown.
- **무한 반복.** `remaining` 이 음수로 누적.

### 3.3 근본 원인
**FSM 의 전이 조건(transition guard)에 외부 비단조 입력(non-monotonic predicate)을 AND 로 넣으면 그 입력이 영구 false 일 때 reachability 깨짐**.

---

## 4. 수정 — Monotonic Transition 원칙

### 4.1 규칙
> 상태 전이는 오직 **시간 경과** 또는 **카운터 증가** 같은 단조 증가 값으로만 조건 지운다.
> 외부 입력은 전이를 막지 말고, 전이 이후의 행동만 조정한다.

### 4.2 전이 vs 행동 분리
| 역할 | 사용 값 |
|------|--------|
| **전이 조건** | `elapsed >= cooldown` (시간), `attempts >= threshold` (카운터) |
| **행동 결정** | metrics healthy 여부 (allow vs defer) |

### 4.3 수정된 코드
```python
if state["phase"] == "phase1_tripped":
    elapsed = now - state["phase1_trip_time"]
    if elapsed < PHASE2_COOLDOWN:
        return "phase2_cooldown"

    # ⭕ 시간 경과 = 단조 전이 트리거. 메트릭 무관.
    state["phase"] = "phase2"
    state["phase2_attempts"] = 0
    logger.info(f"Circuit breaker phase2 entered: {key}")
    # fall through → phase2 블록 실행

if state["phase"] == "phase2":
    if state["phase2_attempts"] >= PHASE2_THRESHOLD:
        state["phase"] = "open"   # 카운터 기반 단조 전이
        return "open"

    # ⭕ 외부 입력은 "이번 턴 허용/보류" 에만 관여
    if not _server_metrics_healthy():
        return "phase2_metrics_block"  # attempts 증가 없이 다음 턴 대기

    return "allow"
```

### 4.4 `fall through` 패턴
- Python 은 switch fall-through 없지만, 연속 if 블록에서 상태 변경 후 return 안 하면 다음 if 가 실행됨.
- 의도적 fall through 는 **주석으로 명시** (리뷰어가 return 누락 버그로 오해 방지).

---

## 5. 안전성 증명 스케치

- **Liveness**: phase1 → phase2 전이가 시간 기반 → 시계 진행만 있으면 반드시 전이 → 교착 없음.
- **Safety**: phase2 내부에서 metrics_block 반복돼도 `attempts` 증가 안 함 → open 으로 premature 전이 없음.
- **Termination**: phase2 에서 3회 실패 → `attempts >= THRESHOLD` → open. 정상 회복 시 allow.

---

## 6. fail-open 메트릭 체크

```python
def _server_metrics_healthy() -> bool:
    if not _latest_system_metrics:
        return True  # 메트릭 미수집 → 낙관적 허용
    cpu = _latest_system_metrics.get("cpu_percent", 0)
    mem = _latest_system_metrics.get("memory_percent", 0)
    if cpu >= 98 or mem >= 98:
        return False
    return True
```

설계 원칙:
- **Fail-open** (모르면 허용): 메트릭 부재 시 True. 재시작은 멱등 → 모르면 시도하는 게 안전.
- **Narrow gate**: 98% 임계는 매우 보수적. 일반 고부하 (90%대) 는 통과.
- **Disk 제외**: 재시작이 디스크 부하 증가 안 시킴.

---

## 7. 비단조 안티패턴 일반화

```python
# ⚠ 안티패턴
if elapsed >= cooldown and external_check():
    state = next_state
```

`external_check()` 가 영구 false 면 영구 고착.

다른 사례:
- "DB 연결 가능할 때만 백그라운드 작업 시작" → DB 영구 단절 시 작업 영영 못 시작.
- "사용자 input 있을 때만 진행" → input 영원히 안 와도 되는 케이스 누락.
- "리소스 충분하면 다음 단계" → 리소스가 영원히 부족하면 stuck.

→ 모든 "external 조건 + AND 전이" 는 의심하라.

---

## 8. 운영 가시성

### 8.1 로그 구조
```
WARNING Container DOWN detected: kr-search-engine
INFO    Auto-restarting: kr:elasticsearch
INFO    Restart success: kr/elasticsearch in 0.6s
WARNING Circuit breaker phase1 tripped: kr:elasticsearch (3 failures in 5min)
INFO    Circuit breaker phase2 entered: kr:elasticsearch
INFO    Circuit breaker phase2_metrics_block: kr:elasticsearch (defer)
WARNING Circuit breaker open: kr:elasticsearch (manual intervention required)
```

### 8.2 메트릭 export (Prometheus 권장)
- 서비스별 현재 phase (gauge: 0/1/2/3).
- phase1 trip 빈도 (counter).
- phase2 진입/완료 (counter).
- open 영구 중단 건수.
- metrics_block 지속 시간.

---

## 9. 관련 라이브러리/이론

### 라이브러리
- **Java/JVM**: Netflix Hystrix (deprecated), resilience4j, Spring Cloud CircuitBreaker.
- **Python**: pybreaker, tenacity.
- **Go**: sony/gobreaker.

### 이론
- **Nygard, *Release It!* (2007/2018)** — 원전.
- **Mealy/Moore FSM (1955)** — 상태 기계 이론.
- **Crash-only software (Candea & Fox, 2003)** — 재시작 멱등성 논거.

---

## 10. 응용 포인트

- Circuit Breaker 의 **전이 조건은 단조 증가 값** (시간, 카운터) 만.
- 외부 입력(메트릭/연결 상태)은 **행동** 에 관여하지 전이에 관여하지 말 것.
- HALF_OPEN 의 단일 샘플이 부족하면 phase2 (N회 시도) 로 확장.
- fail-open: 모르는 상태는 허용. 단 "허용" 의 부작용이 멱등이어야 함.
- 운영 로그는 모든 phase 전이를 명확히 출력.
- Prometheus 같은 메트릭으로 phase 분포/빈도 추적.
