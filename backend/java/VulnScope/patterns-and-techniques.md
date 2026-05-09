# VulnScope 패턴/기법 학습서

> 본 문서는 **카테고리별로 분리**되었습니다. 자세한 내용은 `patterns/` 디렉토리 참조.

## 빠른 링크

| | 카테고리 | 파일 |
|---|---|---|
| 📖 | **인덱스 + 학습 순서** | [patterns/00-index.md](patterns/00-index.md) |
| A | 동시성 (Concurrency) | [patterns/01-concurrency.md](patterns/01-concurrency.md) |
| B | 데이터 구조 + 인덱싱 | [patterns/02-data-structures.md](patterns/02-data-structures.md) |
| C | DDD 도메인 모델링 | [patterns/03-ddd.md](patterns/03-ddd.md) |
| D | 헥사고날 + Modulith | [patterns/04-hexagonal-modulith.md](patterns/04-hexagonal-modulith.md) |
| E | 외부 인프라 추상화 | [patterns/05-infra-abstractions.md](patterns/05-infra-abstractions.md) |
| F | 인프라 구현 패턴 | [patterns/06-infra-implementation.md](patterns/06-infra-implementation.md) |
| G | HTTP / REST API | [patterns/07-http-rest.md](patterns/07-http-rest.md) |
| H | 보안 | [patterns/08-security.md](patterns/08-security.md) |
| I | 테스트 | [patterns/09-testing.md](patterns/09-testing.md) |
| J | 구조적 / 메타 | [patterns/10-structural-meta.md](patterns/10-structural-meta.md) |

---

## 작성 의도

기존엔 한 문서에 80여 개 패턴 요약 → 학습 자료로 부적합 (요약본).

**재구성 후**:
- 카테고리별 분리 → 관심 영역만 골라 읽기 가능.
- 각 패턴 textbook 스타일로 심화:
  - **무엇인가** (개념)
  - **왜 필요한가** (해결하는 문제)
  - **핵심 아이디어** (어떤 통찰에서 나왔나)
  - **전문 용어 사전** (CAS, happens-before, idempotency 등 등장 시 설명)
  - **VulnScope 코드** (실제 사용처 + 줄별 설명)
  - **대안과 비교** (다른 방법은 왜 안 되나)
  - **함정/실수** (처음 쓸 때 잘못 쓰는 패턴)
- 처음 개념을 접하는 사람도 흐름 따라갈 수 있게.

---

## 학습 순서 추천

### 처음 읽는 사람 (junior)
**A → C → I → H → 나머지**

### 시니어 / 아키텍처 관심
**D → E → C → A → 나머지**

### 백엔드 인터뷰 준비
**A → C → D → H → I**

상세는 [patterns/00-index.md](patterns/00-index.md) 참조.

---

## 다른 study 문서 참조

- [comparator.md](comparator.md) — F 카테고리의 Comparator 상세
- [modulith-cross-module-dependency.md](modulith-cross-module-dependency.md) — D 카테고리의 cross-module 상세
- [domain-events-and-outbox.md](domain-events-and-outbox.md) — A/C/E 카테고리의 도메인 이벤트 + outbox 패턴 상세
- [sse-libraries.md](sse-libraries.md) — F 카테고리의 SSE 라이브러리 종합
