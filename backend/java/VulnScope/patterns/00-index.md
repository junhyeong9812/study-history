# VulnScope 패턴/기법 학습서 — 인덱스

> 작성일: 2026-05-02
> 목적: VulnScope v0.1 에서 사용된 패턴/기법을 **처음 접하는 사람도 따라갈 수 있는 textbook** 형태로 카테고리별 분리.
> 사용 가이드: 카테고리 순서대로 읽기 권장 (A → B → ... → J). 또는 관심 영역만 골라 읽기.

---

## 카테고리 목차

| 번호 | 파일 | 주제 | 핵심 키워드 |
|---|---|---|---|
| A | [01-concurrency.md](01-concurrency.md) | 동시성 | ConcurrentHashMap, AtomicLong, CAS, happens-before, ThreadLocal, 가상 스레드, @EventListener |
| B | [02-data-structures.md](02-data-structures.md) | 데이터 구조 + 인덱싱 | two-index, composite key, retention, defensive copy, path normalization |
| C | [03-ddd.md](03-ddd.md) | DDD 도메인 모델링 | Aggregate, Value Object, state machine, invariant, domain event |
| D | [04-hexagonal-modulith.md](04-hexagonal-modulith.md) | 헥사고날 + Modulith | port/adapter, CQRS, NamedInterface, ArchUnit |
| E | [05-infra-abstractions.md](05-infra-abstractions.md) | 외부 인프라 추상화 | Strategy, Facade, Store+Bus, AutoCloseable wrapper |
| F | [06-infra-implementation.md](06-infra-implementation.md) | 인프라 구현 | storage key, dedupe, ETag, base64url, SSE, Comparator |
| G | [07-http-rest.md](07-http-rest.md) | HTTP / REST API | cookie 보안, 422vs400, ProblemDetail, nested URL, envelope |
| H | [08-security.md](08-security.md) | 보안 | AuthFilter, constant-time, timing attack, path traversal, multi-tenant |
| I | [09-testing.md](09-testing.md) | 테스트 | Clock.fixed, AssertJ, MockMvc, Awaitility, MockEventSource |
| J | [10-structural-meta.md](10-structural-meta.md) | 구조적/메타 | ADR, ArchUnit, Modulith verify, learned doc, commit workflow |

---

## 카테고리별 요약 + 어떤 사람이 읽으면 좋은지

### A. 동시성 (Concurrency)
- **무엇**: 여러 스레드가 동시에 같은 데이터를 만질 때 race condition (경쟁 조건) 없이 안전하게 처리하는 기법.
- **누가 읽으면**: 멀티스레드 환경 (web server, worker) 코드를 짜는 사람. Java `java.util.concurrent` 가 처음인 사람.
- **읽고 나면**: ConcurrentHashMap 이 왜 빠른지, AtomicLong 이 lock 없이 어떻게 안전한지, ThreadLocal leak 이 왜 무서운지, 가상 스레드와 @Async 가 어떻게 만나는지 이해.

### B. 데이터 구조 + 인덱싱
- **무엇**: in-memory 저장소를 RDBMS 처럼 효율적으로 (O(1) lookup) 만드는 기법.
- **누가 읽으면**: prototype 단계의 in-memory 저장소를 만드는 사람. v0.2 DB 전환을 미리 대비.
- **읽고 나면**: two-index, composite key, retention, defensive copy 같은 기법을 자연스럽게 쓸 수 있음.

### C. DDD 도메인 모델링
- **무엇**: Eric Evans 의 Domain-Driven Design 핵심 패턴들 — Aggregate, Value Object, invariant, domain event.
- **누가 읽으면**: anemic domain model (getter/setter 만 있는 데이터 클래스) 에서 벗어나고 싶은 사람.
- **읽고 나면**: 도메인 객체가 자기 invariant 를 책임지는 의미를 이해. record + state machine 으로 표현하는 법.

### D. 헥사고날 + Modulith
- **무엇**: 도메인이 인프라/프레임워크에 의존하지 않게 하는 아키텍처 + Spring Modulith 의 모듈 경계 강제.
- **누가 읽으면**: 모놀리스인데 잘 분리하고 싶은 사람. 마이크로서비스로 가기 전 단계.
- **읽고 나면**: port/adapter 의 진짜 의미. NamedInterface 로 모듈 출입구 강제하는 법.

### E. 외부 인프라 추상화
- **무엇**: 파일 시스템/HTTP/SSE 같은 외부 시스템을 도메인이 모르게 감추는 기법.
- **누가 읽으면**: "도메인 코드에 `@Autowired RestTemplate` 적어도 되나?" 고민해본 사람.
- **읽고 나면**: Strategy/Facade 의 실전 적용. AutoCloseable wrapper 의 가치.

### F. 인프라 구현 패턴
- **무엇**: 실제로 구현할 때 마주치는 디테일 — storage key 설계, dedupe, ETag, SSE wire format, Comparator combinator.
- **누가 읽으면**: 매일 코드 짜는 모든 사람.
- **읽고 나면**: 실전에서 즉시 적용 가능한 작은 트릭 모음.

### G. HTTP / REST API
- **무엇**: REST endpoint 디자인 결정들 — cookie 보안, status code, error 표준, URL 구조, pagination.
- **누가 읽으면**: REST API 만들면서 "이거 200 인가 201 인가" 고민하는 사람.
- **읽고 나면**: RFC 7807, RFC 4918 (422 의미) 같은 표준 + Spring 의 합리적 default 활용.

### H. 보안
- **무엇**: 인증/인가/timing attack/path traversal/multi-tenant 격리 등 web app 의 기본 보안.
- **누가 읽으면**: "보안은 라이브러리가 해주는 것" 으로만 알던 사람.
- **읽고 나면**: 왜 `Arrays.equals` 가 password 비교에 위험한지, 왜 `httpOnly + sameSite=Lax` 조합인지 이해.

### I. 테스트
- **무엇**: 단위/통합/비동기/SSE 테스트 패턴들.
- **누가 읽으면**: 테스트가 flaky 하거나 작성이 두려운 사람.
- **읽고 나면**: Clock.fixed 로 시간 결정성, anonymous stub 으로 Mockito 회피, Awaitility 로 비동기 검증.

### J. 구조적 / 메타
- **무엇**: 코드 외의 — ADR, ArchUnit, learned doc, commit 워크플로우 같은 메타 패턴.
- **누가 읽으면**: 팀 (혹은 미래의 자기) 와 일하는 모든 사람.
- **읽고 나면**: 코드 자체보다 코드를 둘러싼 결정/문서가 왜 중요한지.

---

## 학습 순서 추천

### 처음 읽는 사람 (junior)
**A → C → I → H → 나머지**
- A 동시성: 즉시 도움
- C DDD: 도메인 모델링 의 새 시각
- I 테스트: 일상에 즉시 도움
- H 보안: 사고 예방
- 나머지: 관심사대로

### 시니어 / 아키텍처 관심
**D → E → C → A → 나머지**
- D 헥사고날+Modulith: 큰 그림
- E 인프라 추상화: 적용
- C DDD: 모델 정합성
- A 동시성: 구현 디테일

### 백엔드 인터뷰 준비
**A → C → D → H → I**
- A 동시성: 단골
- C DDD: 모델링 질문
- D 아키텍처: high-level
- H 보안: cookie/JWT/timing
- I 테스트: 단위 vs 통합

---

## 다른 study doc 참조

- [comparator.md](../comparator.md) — F 카테고리의 Comparator 상세
- [modulith-cross-module-dependency.md](../modulith-cross-module-dependency.md) — D 카테고리의 cross-module 상세
- [domain-events-and-outbox.md](../domain-events-and-outbox.md) — A/C/E 카테고리의 도메인 이벤트 + outbox 패턴 상세
- [sse-libraries.md](../sse-libraries.md) — F 카테고리의 SSE 라이브러리 종합
