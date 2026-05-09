# Observability 핵심 개념

> 트레이싱/메트릭/로그 운영에 자주 등장하는 개념들 정리. `opentelemetry.md` 의 보충판.

## 1. Trace, Span, Context

| 용어 | 정의 |
|------|------|
| **Trace** | 한 요청이 시스템을 흘러가며 만든 모든 활동의 집합. 고유한 `trace_id` 보유. |
| **Span** | trace 안의 하나의 작업 단위. `span_id`, `parent_span_id`, 시작/종료 시각, 속성. |
| **Context** | 현재 활성 trace/span 정보를 들고 다니는 객체. 스레드 로컬 또는 명시적으로 전파. |

trace = span의 트리 구조. root span은 부모가 없음.

## 2. 컨텍스트 전파 (Context Propagation)

서비스 A가 서비스 B를 HTTP로 호출할 때 trace를 이어주려면 trace ID를 헤더로 전달해야 한다.

### W3C Trace Context (표준)
```
traceparent: 00-<trace-id 32 hex>-<span-id 16 hex>-<flags>
tracestate:  key1=val1,key2=val2
baggage:     userId=42;tenant=acme
```

### B3 (Zipkin 형식, legacy)
```
X-B3-TraceId: ...
X-B3-SpanId: ...
X-B3-ParentSpanId: ...
X-B3-Sampled: 1
```

OpenTelemetry는 W3C와 B3 둘 다 지원. Spring Boot OTel은 W3C 기본.

## 3. 샘플링 전략

### Head Sampling (요청 시작 시 결정)
- **Always On / Always Off**: 0% 또는 100%
- **TraceIdRatioBased**: trace_id를 해시해 확률적으로 결정 (이 프로젝트의 `probability: 0.1`)
- **ParentBased**: 부모의 결정을 따름 (외부에서 들어온 trace는 그대로 유지)
- **RateLimiting**: 초당 N개 제한

장점: 가볍다. 단점: 에러난 trace를 못 잡을 수 있음.

### Tail Sampling (수집 후 결정)
- 일단 모든 span을 Collector가 받음
- trace 완성 후 조건으로 keep/drop 결정
  - 에러가 있으면 keep
  - p95 이상 지연이면 keep
  - 특정 user_id만 keep

장점: 흥미로운 trace를 안 놓침. 단점: Collector에 모든 데이터가 들어와 일시 보관 → 메모리/네트워크 비용.

이 프로젝트는 head sampling만. 운영에서는 OpenTelemetry Collector + tail sampling 조합이 일반적.

## 4. Baggage

trace 내내 따라다니는 key-value. 모든 span/서비스에 자동 전파.

```java
// 예시 (OTel API)
Baggage.current().toBuilder()
    .put("tenant_id", "acme")
    .build()
    .makeCurrent();
```

이후 모든 hop의 모든 span에서 `Baggage.current().getEntryValue("tenant_id")` 로 조회 가능.

주의:
- 모든 hop으로 네트워크 전송 → **작게 유지** (PII 금지)
- 메트릭에 자동 추가되지 않음 (수동 처리 필요)

## 5. Cardinality (카디널리티)

메트릭에 붙는 라벨/태그의 고유값 수.

```
http_requests_total{method="GET", path="/orders", status="200"}
                    ↑ 5개      ↑ 100개         ↑ 5개
                    → 5 × 100 × 5 = 2500 시계열
```

라벨에 `user_id`(100만 명) 같은 걸 넣으면 시계열이 폭발. Prometheus는 메모리/디스크 폭발.

규칙:
- Low cardinality (≤100 값): tag로 OK — `method`, `status`, `region`
- High cardinality: trace span 속성으로만 — `user_id`, `order_id`

`@Observed`의 `lowCardinalityKeyValues` vs `highCardinalityKeyValues`가 이 구분을 강제.

## 6. Exemplars

메트릭 데이터포인트에 trace_id를 첨부. "이 p99 spike의 정확한 trace는 무엇인가"를 한 클릭으로 추적.

```
http_server_duration_seconds_bucket{le="0.5"} 12345 # exemplar trace_id=abc123 0.487
```

Grafana에서 메트릭 그래프의 점을 클릭 → Tempo/Jaeger trace로 점프.

## 7. Span Events / Span Links

- **Event**: span 안에서 일어난 이산 사건. 시각 + 속성. (예: "cache_miss", "retry_attempt=2")
- **Link**: 다른 trace의 span과 연결. fan-out/fan-in 시 인과 관계 표현 (배치 처리 등).

## 8. RED / USE 메트릭 패턴

서비스 모니터링에서 무엇을 봐야 하는지의 모범:

### RED (Request 관점)
- **Rate**: 초당 요청 수
- **Errors**: 에러율
- **Duration**: 응답 시간 (p50/p95/p99)

### USE (Resource 관점)
- **Utilization**: 리소스 사용률 (CPU%, 메모리%)
- **Saturation**: 큐 대기/스왑 등 포화도
- **Errors**: 리소스 자체의 에러 (디스크 에러)

이 프로젝트의 `@Observed`가 자동 생성하는 timer가 Duration, counter가 Rate/Errors를 커버.

## 9. SLI / SLO / SLA

| 용어 | 정의 |
|------|------|
| **SLI** (Indicator) | 측정값. "5xx 비율", "p95 latency" |
| **SLO** (Objective) | 내부 목표. "5xx < 0.1% per month" |
| **SLA** (Agreement) | 외부 계약. "99.9% uptime, 안 지키면 환불" |

Error Budget = (1 - SLO) × 시간. 예산을 다 쓰면 새 기능 배포 중단.

## 10. 로그 - 트레이스 - 메트릭 상관관계

운영에서 흐름:
1. **메트릭 알람** (`error_rate > 5%`)
2. → 그 시간대의 **trace** 조회 (exemplar로 즉시 점프)
3. → trace의 span을 따라가며 **로그** 필터 (trace_id로 필터)
4. 원인 파악 후 수정

이걸 가능케 하는 게:
- Log line 안에 trace_id (Spring + OTel은 MDC로 자동 주입)
- 메트릭에 exemplar
- Collector에서 셋 다 같은 백엔드로 보낼 수 있음

## 11. Cardinality Bomb 사고 사례

흔한 사고:
- URL path를 그대로 라벨로 넣음 → `/orders/<uuid>` 가 모두 다른 라벨
- 사용자 입력을 라벨로
- 동적으로 생성되는 ID
→ Prometheus OOM, 백엔드 다운

방지: 라벨 정규화 (path → "/orders/{id}"), 새 라벨 도입 시 cardinality 검토.

## 12. Pull vs Push 메트릭

### Pull (Prometheus)
서버가 `/actuator/prometheus` 엔드포인트를 노출, Prometheus가 주기적으로 scrape.
- 장점: 서비스가 "살아있나"를 자연스럽게 알 수 있음
- 단점: 짧은 작업(배치, Lambda)은 안 맞음

### Push (OTLP)
서비스가 OTel Collector로 push.
- 장점: 짧은 작업/네트워크 격리 환경
- 단점: 서비스 다운을 별도로 알아야 함

이 프로젝트는 **둘 다 노출**: Prometheus pull + OTLP push.

## 13. Profile (프로파일링) — 4번째 시그널

OpenTelemetry는 최근 **continuous profiling** 도 추가 (CPU/메모리 샘플링). Pyroscope, Parca 등의 백엔드.
- 메트릭/트레이스로도 안 잡히는 "왜 CPU가 튀지?"를 stack trace 단위로 분석.

## 정리

이 프로젝트의 관찰성 스택:
- **자동 계측**: HTTP, JDBC, Spring Async (`spring-boot-starter-opentelemetry`)
- **수동 계측**: `@Observed` (PaymentCommandService)
- **Sampling**: dev 100%, prod 10%
- **Export**: Prometheus pull + OTLP push (둘 다)
- **태그**: `application=shoptracker` 자동 부착

운영에서 한 단계 더 나가려면 OpenTelemetry Collector + tail sampling + Tempo/Loki/Prometheus 조합.
