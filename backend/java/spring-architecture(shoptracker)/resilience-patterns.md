# 복원성 패턴 (Resilience Patterns)

> 분산 시스템에서 외부 의존성의 실패/지연을 다루는 패턴들. Circuit Breaker, Retry, Timeout, Bulkhead, Rate Limiter.
>
> 이 프로젝트에는 아직 미적용이지만, `FakePaymentGateway`가 10% 실패하는 모델이라 복원성 패턴의 적용 후보. 학습용 정리.

## 왜 필요한가

분산 시스템의 8가지 오해:
- 네트워크는 신뢰할 수 있다 (X)
- 지연은 0이다 (X)
- 대역폭은 무한하다 (X)
- ...

외부 의존성(결제 PG, 알림 SMS, 외부 API)은 언제든 느려지거나 실패할 수 있다. 보호 장치 없이 단순 호출하면:
1. 외부 API 응답 지연 5초
2. 우리 서비스 스레드 100개가 모두 대기
3. 새 요청 못 받음 → 우리도 다운
→ **연쇄 장애 (cascading failure)**

## 1. Timeout

가장 기본. 외부 호출에 명시적 시간 제한.

```java
RestClient.builder()
    .requestFactory(new SimpleClientHttpRequestFactory() {{
        setConnectTimeout(2000);    // 연결 2초
        setReadTimeout(5000);       // 읽기 5초
    }})
    .build();
```

**Timeout 없는 호출은 코드 결함**. 모든 외부 호출은 명시적 timeout.

## 2. Retry

일시적 실패(네트워크 깜빡임, 일시적 503)를 자동 재시도.

### 핵심 옵션
- **maxAttempts**: 최대 시도 (예: 3)
- **backoff**: 시도 간 대기 시간
  - **Fixed**: 항상 같은 간격 (1초, 1초, 1초)
  - **Exponential**: 지수 증가 (1초, 2초, 4초)
  - **Exponential + Jitter**: 지수 + 랜덤 변동 (thundering herd 방지)

### 멱등성 주의
**같은 호출을 두 번 해도 안전해야** retry 가능. 결제처럼 부수효과가 큰 작업은:
- Idempotency Key 도입 (요청에 고유 키, 서버가 중복 처리 방지)
- 재시도 가능한 작업과 아닌 작업 구분

### Spring Retry 예시
```java
@Retryable(retryFor = IOException.class,
           maxAttempts = 3,
           backoff = @Backoff(delay = 1000, multiplier = 2))
public GatewayResult callExternalApi() { ... }
```

## 3. Circuit Breaker

전기 차단기처럼 외부 의존성이 망가지면 호출을 **중단** 하고, 일정 시간 후 회복을 살핌.

### 3-state machine

```
        ┌─────────┐  failure rate exceeds threshold
        │ CLOSED  │  ───────────────────────────────►
        │ (정상)  │                                  ┌─────────┐
        │         │                                  │  OPEN   │
        │         │  ◄────────────────────           │ (차단)  │
        └─────────┘  test call succeeds              │         │
             ▲                          │            └────┬────┘
             │                          │                 │
             │                          │                 │ wait duration
             │                          │                 │ (예: 30초)
             │                          │                 ▼
             │           test failed    │           ┌──────────┐
             └──────────────────────────┴───────────┤HALF-OPEN │
                                                    │(시험)    │
                                                    └──────────┘
```

| 상태 | 동작 |
|------|------|
| **CLOSED** (정상) | 호출 통과. 실패율 모니터링. 임계값 넘으면 OPEN. |
| **OPEN** (차단) | 호출 즉시 실패 (Fail Fast). 외부 시스템에 부하 안 줌. 일정 시간 후 HALF_OPEN. |
| **HALF_OPEN** (시험) | 일부 호출 허용. 성공하면 CLOSED, 실패하면 다시 OPEN. |

### 핵심 옵션 (Resilience4j 기준)
- `failureRateThreshold`: 실패율 % (예: 50)
- `slidingWindowSize`: 측정 윈도 (예: 최근 10건)
- `waitDurationInOpenState`: OPEN 유지 시간 (예: 30초)
- `permittedNumberOfCallsInHalfOpenState`: HALF_OPEN에서 허용할 호출 수

### Fallback
차단 중에도 사용자에게 뭔가 응답해야 함:
```java
@CircuitBreaker(name = "payment-gateway", fallbackMethod = "fallback")
public GatewayResult process(Payment p) { ... }

public GatewayResult fallback(Payment p, Exception e) {
    return new GatewayResult(false, null, "PG 일시 장애 — 잠시 후 재시도");
}
```

## 4. Bulkhead (격벽)

선박의 격벽처럼, **자원을 분리** 해 한 곳의 침수가 전체로 번지지 않게.

### Thread Pool Bulkhead
외부 의존성마다 별도 thread pool 사용.
- 결제 PG 호출용 pool: 20개 스레드
- 알림 SMS 호출용 pool: 10개 스레드
- 결제 PG가 느려져도 알림은 영향 없음

### Semaphore Bulkhead
스레드 풀 대신 동시 호출 수만 제한 (가벼움).
```yaml
bulkhead:
  payment-gateway:
    maxConcurrentCalls: 20
```

Virtual Thread 시대에는 thread pool bulkhead의 가치가 줄지만, semaphore bulkhead로 동시 호출 제한은 여전히 유효.

## 5. Rate Limiter (속도 제한)

초당 N개로 호출 제한. 외부 API의 quota 보호 + 자기 보호.

### 알고리즘
- **Token Bucket**: 일정 속도로 토큰 충전, 호출마다 토큰 소비. 빈 양동이는 거절. 버스트 허용.
- **Leaky Bucket**: 일정 속도로 토큰 소비. 정확한 속도 유지.
- **Fixed Window**: 1초당 N개. 경계에서 burst 가능.
- **Sliding Window Log**: 정확하지만 메모리 많이 씀.

```yaml
ratelimiter:
  payment-gateway:
    limitForPeriod: 100      # 100 calls
    limitRefreshPeriod: 1s   # per 1 second
    timeoutDuration: 0       # 대기 안 하고 즉시 거절
```

## 6. Cache (보조)

복원성 직접 패턴은 아니지만 외부 의존성을 줄여 간접적으로 도움. Spring Cache + Redis/Caffeine.

## 패턴 조합

```
요청 → [RateLimiter] → [Bulkhead] → [CircuitBreaker] → [Retry] → [Timeout] → 외부 호출
                                                                              ↓
                                                                        성공/실패
                                                                              ↓
                                                                        [Fallback]
```

순서가 중요. Resilience4j 기본 적용 순서:
1. Rate Limiter (호출 자체 제한)
2. Bulkhead (동시성 제한)
3. Time Limiter (timeout)
4. Circuit Breaker (외부 상태 보호)
5. Retry (재시도)
6. Fallback (실패 시 대안)

## Resilience4j (이 프로젝트에 추가한다면)

`io.github.resilience4j:resilience4j-spring-boot3` (4.x용은 곧 등장).

### 가상 적용 — 결제 게이트웨이
```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentGateway:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 30s
        permittedNumberOfCallsInHalfOpenState: 3

  retry:
    instances:
      paymentGateway:
        maxAttempts: 3
        waitDuration: 1s
        exponentialBackoffMultiplier: 2

  timelimiter:
    instances:
      paymentGateway:
        timeoutDuration: 5s
```

```java
@Service
public class FakePaymentGateway implements PaymentGateway {

    @CircuitBreaker(name = "paymentGateway", fallbackMethod = "fallback")
    @Retry(name = "paymentGateway")
    @TimeLimiter(name = "paymentGateway")
    @Override
    public GatewayResult process(Payment payment) { ... }

    private GatewayResult fallback(Payment p, Throwable t) {
        return new GatewayResult(false, null, "PG circuit open: " + t.getMessage());
    }
}
```

이러면 `FakePaymentGateway`의 10% 실패가 누적되어도 cascading failure로 번지지 않음.

## 메트릭 / 알람

복원성 컴포넌트는 자체 메트릭을 노출:
- `resilience4j_circuitbreaker_state`
- `resilience4j_retry_calls`
- `resilience4j_ratelimiter_available_permissions`

Prometheus + Grafana 대시보드 + 알람:
- Circuit Breaker가 OPEN으로 전환 → 즉시 알람
- Retry rate spike → 외부 의존성 이상 신호

## Spring Cloud Gateway / 서비스 메시

여러 서비스가 있는 환경에서는 sidecar 또는 gateway에서 일괄 적용:
- **Spring Cloud Gateway**: 게이트웨이 단에서 rate limit, circuit breaker
- **Istio/Linkerd**: 서비스 메시가 모든 호출을 가로채 적용 (코드 변경 0)

## 안티패턴

1. **모든 호출에 retry 무지성 적용**: 멱등성 안 따짐 → 결제 두 번 처리.
2. **Circuit Breaker만 두고 Fallback 안 만듦**: OPEN 상태에서 사용자에게 그냥 에러. → fallback 필수.
3. **Timeout 없이 retry**: 5초 대기 × 3회 = 15초. 서비스가 그 동안 묶임.
4. **자기 자신에게 Circuit Breaker**: 의미 없음. 외부 의존성에만.
5. **메트릭 안 봄**: Circuit이 OPEN인데 아무도 모르면 무용지물.

## 정리

| 문제 | 해결 |
|------|------|
| 호출이 영원히 대기 | **Timeout** |
| 일시적 깜빡임 | **Retry (with backoff + jitter)** |
| 외부 의존성 다운에 우리도 끌려들어감 | **Circuit Breaker** |
| 한 의존성이 모든 스레드 잡아먹음 | **Bulkhead** |
| 외부 API quota 초과 | **Rate Limiter** |
| 차단 중에도 응답 | **Fallback** |

이 프로젝트의 `FakePaymentGateway`가 학습용 적용 후보. 다음 단계에서 Resilience4j 추가하면 좋은 페이즈가 될 것.
