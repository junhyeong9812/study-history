# study-history

> 프로젝트들을 진행하며 공부한 내용들을 정리한 리포입니다.

각 프로젝트에서 마주친 패턴, 개념, 도구들을 그때그때 문서로 남겨 둔 학습 아카이브입니다.
주제별/프로젝트별로 폴더를 나누어 정리하며, 새로운 내용이 생길 때마다 계속 추가됩니다.

---

## 구조

```
.
├── backend/
│   ├── java/         # Java/Spring 기반 프로젝트 학습 노트
│   └── phython/      # Python/FastAPI 기반 프로젝트 학습 노트
├── frontend/
│   ├── concepts/     # React, JS, TS, 브라우저 동작 원리 등 공통 개념
│   └── next/         # Next.js 프로젝트별 패턴 정리
├── infra/            # 인프라/운영/아키텍처 학습 노트
└── cs/               # CS 기초 이론 (cse-fundamentals 21개 챕터 포함)
```

## 백엔드 (`backend/`)

### Java (`backend/java/`)
- **Hybrid-Event-Driven-Architecture** — 이벤트 드리븐 아키텍처 분석, Kafka, Flyway, Testcontainers 등
- **VulnScope** — 동시성, DDD, 헥사고날/Modulith, 인프라 추상화 등 패턴 textbook
- **design-patterns** — GoF 23개 디자인 패턴 정리 (Java 예제)
- **spring-architecture(shoptracker)** — Spring Boot, Modulith, CQRS, OpenTelemetry, GraalVM 등

### Python (`backend/phython/`)
- **concepts** — CPython 실행 모델, GIL, 메모리, 객체 모델 등 언어 핵심
- **fastapi-async** — FastAPI, Pydantic, lifespan, DI 패턴
- **phython-architecture(shoptracker)** — 헥사고날, DI(dishka), 이벤트 버스, CQRS, Saga
- **concurrency-patterns / database-toolkit / llm-integration / redis-streams-consumer / text-search-engine / web-scraping-defense / websocket-streaming / deployment-tooling**

## 프론트엔드 (`frontend/`)
- **concepts** — React 라이프사이클/훅/Reconciliation, JS/TS 코어, 브라우저 이벤트 루프/렌더링/스토리지/Web Vitals
- **next/VulnScope, next/front** — Next.js 프로젝트에서 정리한 React/스타일링/폼/서버 상태/라우팅/스트리밍/테스팅 패턴

## 인프라 (`infra/`)
- agent-based-orchestration, container-resilience, design-patterns-applied,
  event-driven-monitoring, multi-region-data-migration, operations-runbook,
  system-architecture, web-security-defense

## CS (`cs/`)
- **cse-fundamentals** — CSE 기초 종합 교과서 (미적분/선형대수/확률통계/이산수학/자료구조/알고리즘/OS/네트워크/보안/DB/PLT/병렬컴퓨팅/NLP/CV 등 21개 챕터)
- 그 외: CAS, CSRF/XSS 공격과 방어, 동시성 인메모리 스토어 가이드 등 단편 노트

---

## 커밋 규칙

문서 1개 = 커밋 1개. 형식은 다음과 같습니다.

```
feat(<직속 부모 폴더>): <파일명(확장자 제외)>
```

예) `feat(design-patterns): 21-strategy`, `feat(fastapi-async): pydantic-models`
