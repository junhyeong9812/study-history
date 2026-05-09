# ShopTracker 학습서 — Python FastAPI 모듈러 모놀리스

> 작성일: 2026-05-03
> 대상: ShopTracker 코드를 보면서 "이건 무슨 패턴이지?", "Python 의 X 가 어떻게 동작하지?" 묻는 사람.
> **VulnScope 학습서와 동일 구조** — 각 doc 가 독립, 텍스트북 스타일.

---

## 카테고리 (총 22 doc)

### Architecture Patterns (9)
| # | 파일 | 주제 | 키워드 |
|---|---|---|---|
| 01 | [01-hexagonal-architecture.md](01-hexagonal-architecture.md) | Hexagonal 4-layer | domain ← application ← infrastructure ← presentation, port/adapter |
| 02 | [02-dip-protocols.md](02-dip-protocols.md) | DIP + Python Protocol | structural typing, duck typing 의 type-safe 버전 |
| 03 | [03-di-and-dishka.md](03-di-and-dishka.md) | DI + 정책 주입 | Dishka container, Scope (APP/REQUEST), Provider 분리 |
| 04 | [04-event-bus.md](04-event-bus.md) | 내부 이벤트 버스 | InMemory + Protocol, fire-and-forget, fault isolation |
| 05 | [05-cqrs.md](05-cqrs.md) | CQRS (간소화) | Command vs Query handler, 두 Protocol 충족 |
| 06 | [06-saga-cross-module.md](06-saga-cross-module.md) | Saga + cross-module flow | Orders → Payments → Shipping → Tracking 이벤트 chain |
| 07 | [07-cross-cutting-via-shared-dto.md](07-cross-cutting-via-shared-dto.md) | 횡단 관심사 = shared DTO | SubscriptionContext, 모듈 간 결합 회피 |
| 08 | [08-aggregate-state-machine.md](08-aggregate-state-machine.md) | Aggregate + 상태머신 | factory method, immutable update, can_transition_to |
| 20 | [20-event-handler-di-bridge.md](20-event-handler-di-bridge.md) | 이벤트 핸들러 ↔ DI 다리 | 어댑터 패턴, REQUEST 스코프 수동, register_event_handlers, 누적 등록 |

### Implementation Patterns (6)
| # | 파일 | 주제 | 키워드 |
|---|---|---|---|
| 09 | [09-value-object-money.md](09-value-object-money.md) | Money 값 객체 | 불변, __slots__, Decimal, 연산 메서드 |
| 10 | [10-mapper-orm-domain.md](10-mapper-orm-domain.md) | Mapper (ORM ↔ Domain) | entity vs model 분리, 변환 함수 |
| 11 | [11-repository-pattern.md](11-repository-pattern.md) | Repository 패턴 | async, session injection, query/command 분리 |
| 12 | [12-fastapi-router-pydantic.md](12-fastapi-router-pydantic.md) | FastAPI router + Pydantic | APIRouter, Depends, response_model |
| 13 | [13-testing-strategies.md](13-testing-strategies.md) | 테스트 전략 (피라미드) | domain unit (DB 없음), application (Fake), integration (TestClient) |
| 21 | [21-test-infrastructure-patterns.md](21-test-infrastructure-patterns.md) | 테스트 인프라 패턴 (실전) | TestProvider 누적, FakeGateway 비결정성, 폴링, fixture scope |

### Python Concepts (7)
| # | 파일 | 주제 | 키워드 |
|---|---|---|---|
| 14 | [14-python-typing.md](14-python-typing.md) | Type hints + Protocol | TypeVar, Generic, Literal, Optional, Union |
| 15 | [15-python-dataclass.md](15-python-dataclass.md) | dataclass | @dataclass(frozen=True), default, __slots__ |
| 16 | [16-python-async-await.md](16-python-async-await.md) | async / await | asyncio, event loop, async generator |
| 17 | [17-python-decorators.md](17-python-decorators.md) | Decorators | @property, @classmethod, @staticmethod, @provide |
| 18 | [18-python-context-managers.md](18-python-context-managers.md) | Context Managers | with, async with, contextmanager, exit handling |
| 19 | [19-python-imports.md](19-python-imports.md) | Import system | package, __init__.py, relative vs absolute, lazy import |
| 22 | [22-python-match-case.md](22-python-match-case.md) | match-case (PEP 634) | 7종 패턴, capture vs literal 함정, ShopTracker 정책 분기 |

---

## 학습 순서 추천

### Python 처음 / 백엔드 입문
**14 → 15 → 16 → 17 → 11 → 09 → 01**
- Python 기본 → 도메인 표현 (dataclass) → async → 데코레이터 → 데이터 접근 → 값 객체 → 아키텍처.

### Java/Spring 백엔드 → Python 전환
**01 → 02 → 03 → 11 → 12 → 04 → 05 → 06**
- 익숙한 헥사고날부터 → Protocol 의 차이 → DI 다른 점 → FastAPI/Repository → 이벤트 → CQRS → Saga.

### 아키텍처 깊이
**01 → 06 → 07 → 04 → 08 → 02 → 03 → 20**
- 큰 그림 → 모듈 간 흐름 → 횡단 관심사 → 이벤트 → Aggregate → 의존 역전 → DI → 이벤트 ↔ DI 다리.

### 정책 주입 / 이벤트 와이어링 깊이 (본 작업의 핵심 토픽)
**03 → 22 → 07 → 04 → 20 → 21**
- DI 정책 주입 → match-case 문법 → SubscriptionContext 횡단 DTO → EventBus → 이벤트 ↔ DI 어댑터 → 통합 테스트 패턴.

### 인터뷰 준비
**01, 02, 04, 05, 11, 14, 16, 20** 우선.

---

## 각 doc 의 일관 구조

```
§0. 이 패턴/개념이 무엇이고 왜 필요한가
§1. 본질적 메커니즘 (어떻게 동작하나)
§2. ShopTracker 의 코드 + 줄별 설명
§3. 직접 구현하면 (라이브러리 없이)
§4. 함정 + 흔한 오해
§5. 다른 언어/프레임워크와 비교 (Spring, NestJS 등)
§6. 학습 포인트 (한 줄 요약)
```

---

## 다른 docs 참조

- [`PROJECT-DESIGN.md`](../PROJECT-DESIGN.md) — 전체 프로젝트 설계.
- [`RESEARCH.md`](../RESEARCH.md) — 사전 조사.
- [`phase/PHASE_{1..6}.md`](../phase/) — 페이즈별 계획.
- [`implements/phase*.md`](../implements/) — 구현 가이드.

---

## ShopTracker 의 결정 한눈에

- **Hexagonal 4-layer** — domain (FastAPI/SQLAlchemy import 0) ← application ← infrastructure ← presentation.
- **Python `Protocol`** — Java interface 대신. structural typing.
- **Dishka DI 컨테이너** — provider 분리 (App/Orders/Subscriptions), Scope 기반 lifetime.
- **InMemory Event Bus** — Protocol 정의, fire-and-forget, 핸들러 fault isolation.
- **CQRS 약식** — Command handler 와 Query handler 분리 (같은 모듈 안).
- **shared DTO** (SubscriptionContext) — 횡단 관심사 의존 회피.
- **Mapper 분리** — domain entity 와 ORM model 100% 별도.
- **dataclass(frozen=True)** — 도메인 이벤트, value object 모두 불변.

---

## ShopTracker 가 안 쓴 것 (의도)

| 안 씀 | 이유 |
|---|---|
| Django/Flask | FastAPI 가 학습 대상. async + Pydantic 친화. |
| Pydantic 을 도메인에 | 도메인은 프레임워크 모름. Pydantic 은 presentation 만. |
| Repository 의 raw SQL | SQLAlchemy 가 mapping 책임. domain 은 entity 만. |
| 실제 PG 연동 | Fake Payment Gateway 로 학습 단순화. |
| JWT/OAuth | API Key 수준. 인증 학습 X (별도 주제). |
| Celery/외부 큐 | InMemory event bus. 분산 X. |
| Pytest 외부 라이브러리 | 표준 pytest + asyncio plugin 만. |
