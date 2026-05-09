# OpenTelemetry / Micrometer

> 분산 추적(traces) + 메트릭(metrics) + 로그(logs) 표준화 프레임워크.

## 개념

OpenTelemetry(OTel)는 **벤더 중립** 관찰성 표준. 코드에서는 OTel API로 계측, 백엔드는 자유(Jaeger/Tempo/Datadog/NewRelic).

이 프로젝트는 `spring-boot-starter-opentelemetry` (Boot 4의 단일 의존성)을 사용 → 자동 계측 + Micrometer 통합.

## 3대 시그널

| 시그널 | 무엇을 보나 | 단위 | 예시 |
|--------|------------|------|------|
| **Traces** | 한 요청이 거치는 모든 단계 | span 트리 | "주문 생성 요청 → 결제 처리 → 배송 생성" 흐름 |
| **Metrics** | 시간에 따른 집계 수치 | counter, gauge, histogram | 초당 요청 수, 에러율, p99 응답시간 |
| **Logs** | 이벤트 텍스트 | structured logs | "Payment approved for order X" |

이 셋이 **상관관계(correlation)** 로 연결되는 게 핵심. 트레이스 ID로 로그를 필터링하고, 메트릭의 spike에서 트레이스로 drill-down.

## Span / Trace

```
Trace (한 요청)
└─ Span: HTTP POST /orders (root)
    ├─ Span: OrderCommandService.createOrder
    │   ├─ Span: orderRepository.save
    │   │   └─ Span: SELECT/INSERT orders
    │   └─ Span: eventPublisher.publishEvent
    └─ Span: PaymentEventHandler.on (async, child via context)
        ├─ Span: payment.process (★ @Observed)
        ├─ Span: paymentGateway.process
        └─ Span: paymentRepository.save
```

각 span은:
- 시작/종료 시각, 부모 span ID, trace ID
- 속성(attributes): key-value
- 이벤트, 상태(OK/ERROR)

## 자동 계측 (Auto-instrumentation)

`spring-boot-starter-opentelemetry` 만 추가하면:
- HTTP 서버: 요청별 root span 생성
- HTTP 클라이언트(`RestTemplate`/`WebClient`): outbound span
- JDBC: `SELECT/INSERT/UPDATE` span (datasource-proxy 또는 OTel agent 필요)
- Spring Data Repository: 호출 span
- `@Async` / `ApplicationEvent`: context 전파

코드 변경 없이 trace 가 만들어짐.

## 수동 계측 — `@Observed`

```java
// payments/application/service/PaymentCommandService.java
@Observed(name = "payment.process",
          contextualName = "process-payment",
          lowCardinalityKeyValues = {"module", "payments"})
public UUID processPayment(ProcessPaymentCommand command) { ... }
```

- 메서드 호출이 자동으로 span + timer + counter 생성.
- `lowCardinalityKeyValues`: tag로 등록되며 메트릭에 차원 추가 (cardinality 폭발 주의).
- `highCardinalityKeyValues`: trace span 속성에만 (메트릭 차원 X).

내부적으로 Micrometer의 `Observation` API를 사용:
```java
Observation observation = Observation.createNotStarted("payment.detailed", registry)
    .lowCardinalityKeyValue("method", command.method())
    .highCardinalityKeyValue("orderId", command.orderId().toString());
observation.observe(() -> { ... });
```

## Sampling (샘플링)

모든 trace를 기록하면 비싸므로 일부만 기록.

```yaml
management:
  tracing:
    sampling:
      probability: 0.1   # 10%
```

샘플링 결정은 root span에서 한 번 → trace 전체가 통째로 keep/drop.

자세히는 `observability-concepts.md` 참고.

## OTLP (OpenTelemetry Protocol)

표준 export 프로토콜.
```yaml
management:
  otlp:
    tracing:
      endpoint: http://localhost:4318/v1/traces
    metrics:
      endpoint: http://localhost:4318/v1/metrics
```

수신 측은 보통 **OpenTelemetry Collector** 가 담당:
- 받은 데이터를 Jaeger(traces), Prometheus(metrics), Loki(logs) 등으로 분기
- Tail sampling, attribute redaction, batching 등 처리

## Prometheus + Grafana

메트릭은 Prometheus 형식으로도 노출:
```bash
curl http://localhost:8080/actuator/prometheus
```

Spring Boot Actuator + Micrometer Prometheus registry → 자동으로 `/actuator/prometheus` 엔드포인트.

## 컨텍스트 전파 (Context Propagation)

분산 환경에서 trace를 이어붙이려면 trace ID를 다음 hop에 전달해야 함.

표준 헤더:
- `traceparent: 00-<trace-id>-<span-id>-<flags>` (W3C Trace Context)
- `tracestate: ...`
- `baggage: key=value;...`

Spring + OTel은 RestTemplate/WebClient/Kafka 등에서 자동 주입/추출.

## Baggage

Trace 전체에 따라다니는 메타데이터(예: tenant_id, user_id). 모든 span에서 접근 가능.
- 주의: 모든 hop으로 전파되어 네트워크 비용 발생. 작게 유지.

## Exemplars

메트릭의 특정 데이터포인트에 trace ID를 붙이는 기능. "이 p99 spike가 정확히 어떤 trace에서 발생했나?"를 1클릭으로 추적.

## 이 프로젝트에서

- `application.yml`: dev에서 `probability: 1.0`
- `application-prod.yml`: 운영에서 `probability: 0.1`
- `PaymentCommandService.processPayment` 에 `@Observed` 적용 → 결제 처리 시간/호출 횟수 메트릭 자동 생성
- `application` 태그(키-값)를 모든 메트릭에 자동 부착 (`management.observations.key-values`)

## FastAPI/structlog 대응

| FastAPI 환경 | Spring + OTel |
|--------------|---------------|
| `structlog` (logs only) | OTel logs + Micrometer metrics + traces |
| `prometheus_client` 직접 | `@Observed` + Actuator |
| `opentelemetry-instrumentation-fastapi` | 자동 계측 (Boot 기본) |
| trace ID 로그 주입 | MDC 자동 주입 |
