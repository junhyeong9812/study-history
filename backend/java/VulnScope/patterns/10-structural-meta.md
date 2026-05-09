# J. 구조적 / 메타

> 코드 외의 — ADR, ArchUnit, learned doc, commit 워크플로우. 5개 패턴.

---

## §0. 메타 패턴이 왜 중요한가?

코드 자체보다 코드를 둘러싼 것:
- **결정의 기록** (왜 이렇게 했는가) — 미래 자기/팀 위해.
- **자동 강제** (룰 위반 차단) — 사람 잊어도 OK.
- **학습 자료** (코드 흐름) — 새 멤버 빠르게.
- **commit history** (시간 흐름) — git log 가 학습 자료.

J 카테고리는 코드 짜는 시간만큼 중요한 5가지 메타 패턴.

---

## J1. ADR (Architecture Decision Records)

### 무엇인가
중요한 결정의 컨텍스트/대안/결과를 markdown 으로 기록.

### 어디서 쓰나
`docs/decisions/0001~0007-*.md`:
- 0001: Modulith vs Microservices
- 0002: Realtime Bus 전략
- 0003: Virtual Thread 전략
- 0004: ...
- 0007: Realtime Store and Bus

### 표준 ADR 구조
```markdown
# ADR 0001: Modulith vs Microservices

## Context
무엇이 결정 필요했는가? 배경.

## Considered Alternatives
A안, B안, C안.

## Decision
선택 + 이유.

## Consequences
결과 — 좋은 점, 나쁜 점, trade-off.
```

### 왜 필요한가

**없으면**: 6개월 후 자기 코드 보면서 "왜 이렇게 했지?" 추측. 새 멤버가 같은 질문.

**있으면**: 결정의 컨텍스트 + trade-off 명시. 유사 결정 시 참조.

### 핵심 아이디어
**결정의 archeology** — git blame 으로 누가 언제 했는지는 알지만 "왜" 는 모름. ADR 이 그 "왜" 보존.

### 전문 용어 사전
- **ADR**: Michael Nygard 가 2011 년 제안. 여러 조직 표준.
- **immutable record**: ADR 은 한 번 작성 후 immutable. 변경되면 새 ADR (이전 supersede).

### 함정
- ADR 너무 많으면 (모든 PR 마다) noise. 큰 결정만.
- ADR 작성 안 하면 점점 안 함 — culture 강제.

### 대안
- design doc (Google 스타일) — 더 큰 결정.
- RFC process — proposal 단계.
- ADR 가 가장 가벼움.

### 출처
Michael Nygard, [*Documenting Architecture Decisions*](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions) (2011).

---

## J2. ArchUnit 룰 (자동 강제)

### 무엇인가
아키텍처 컨벤션을 테스트로 자동 검증.

### 어디서 쓰나
```java
// back/src/test/java/com/vulnscope/architecture/ArchitectureTest.java
@AnalyzeClasses(packages = "com.vulnscope", importOptions = ImportOption.DoNotIncludeTests.class)
class ArchitectureTest {

    @ArchTest
    static final ArchRule domain_must_not_depend_on_outside =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "..application..", "..presentation..", "..infrastructure..",
                "org.springframework.boot..",
                "org.springframework.web..",
                "org.springframework.context..",
                "org.springframework.stereotype..",
                "com.fasterxml.jackson.."
            );

    @ArchTest
    static final ArchRule application_must_not_depend_on_presentation_or_infrastructure =
        noClasses().that().resideInAPackage("..application..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "..presentation..", "..infrastructure.."
            );

    @ArchTest
    static final ArchRule controllers_should_be_in_presentation =
        classes().that().areAnnotatedWith(RestController.class)
            .should().resideInAPackage("..presentation..");
}
```

### 왜 자동?
**컨벤션은 사람이 잊음**. PR review 마다 "이거 도메인이 인프라 의존 아니에요?" 매번 체크 X. 테스트가 자동.

### 핵심 아이디어
**규칙 → 테스트 → CI 강제**. 사람 의존 X.

### 함정
- 룰이 너무 많으면 false positive — 새 코드 작성마다 ArchUnit 우회 시도.
- 룰은 명확한 invariant 만 — "depends on" 같은 강한 룰만.

### 대안
- Spring Modulith verify — 비슷한 의미. 모듈 경계 위주.
- 코드 리뷰 — 사람 의존.

### 출처
[ArchUnit](https://www.archunit.org/) — Peter Gafert.

---

## J3. Spring Modulith `@NamedInterface` + `verify()`

### 무엇인가
모듈 경계 명시 + 자동 검증.

상세는 `04-hexagonal-modulith.md` D5/D6, `study/modulith-cross-module-dependency.md` 참조.

### 핵심
```java
// 출입구 명시
@org.springframework.modulith.NamedInterface("api")
package com.vulnscope.scan.application.api;

// 검증
@Test
void modulith_modules_have_no_violations() {
    ApplicationModules.of(VulnscopeApplication.class).verify();
}
```

### ArchUnit 과의 차이
- ArchUnit: 일반적인 아키텍처 룰.
- Modulith: 모듈 경계 + 출입구 + 의존 그래프 자동.
- 둘 다 사용 (보완).

---

## J4. learned doc 패턴 (페이즈별 풀 코드 + 줄별 주석)

### 무엇인가
각 페이즈의 **모든 파일 코드 + 줄별 주석** 형태 학습 문서.

### 어디서
- `docs/plans/2026-04-30/02-first-slice/learned/01~09g-*.md` (16개)
- `docs/plans/2026-04-30/03-design-pass/learned/10a~10p-*.md` (12개)

### 구조
```markdown
# Learned — 03 scan 모듈 (전체 파일 + 줄별 주석)

## 0. 전체 파일 트리
...

## 1. shared/kernel — 모듈 간 공유 ID/value 타입
### 1.1 ScanId.java
전체 코드:
```java
public record ScanId(UUID value) {
    public ScanId {                                                    // (1)
        Objects.requireNonNull(value);
    }
    public static ScanId generate() {                                  // (2)
        return new ScanId(UUID.randomUUID());
    }
}
```

| 줄 | 의미 |
|---|---|
| (1) | compact constructor — null 차단 |
| (2) | 정적 팩토리 — random UUID |

...

## 8. 학습 포인트 종합
1. ...
```

### 왜 이 형태?

**기존 문제**: 코드 일부만 발췌 + 일반 설명 → 학습자가 git 과 doc 왔다갔다.

**풀버전**: doc 하나 보면 모든 코드 + 의미 이해. cross-reference X.

### 핵심 아이디어
**doc 도 코드처럼 thorough** — "한 번에 다 보임". textbook 스타일.

### 대안
- 코드 + 별도 README — 분리. 위 문제 발생.
- inline 주석만 — 컨텍스트 (다른 파일과의 관계) 표현 어려움.

---

## J5. 1파일=1commit 학습 워크플로우

### 무엇인가
각 commit 이 1개 파일 + 학습 단계. git log 가 학습 흐름.

### 어디서
`docs/COMMIT_GUIDE.md` 의 phase 별 commit 순서.

예시 (Phase 04 finding):
```
1. shared/kernel/FindingId.java     → feat(finding): add finding id value type
2. shared/kernel/Severity.java      → feat(finding): add severity enum
3. shared/kernel/OwaspId.java       → feat(finding): add owasp id enum
...
29. FindingControllerTest.java      → test(finding): cover get and list-by-scan via mockmvc
```

### 왜?

**naive (큰 batch commit)**:
```
1 commit: "feat: add finding module" — 모든 파일
```
- git log 가 학습 자료 X.
- 새 멤버가 "어디부터 봐야?" 모름.

**1파일=1commit**:
- git log 가 학습 흐름.
- bisect 로 회귀 디버깅 정밀.
- 매 commit 이 의미 단위.

### Frontend 는 1페이즈=1commit

frontend 는 `package.json` 등 묶이는 게 자연 → batch.

### 핵심 아이디어
**git history = documentation**. 코드 변화의 시간 축.

### 함정
- 매 commit 빌드 안 될 수 있음 — 페이즈 끝에 test green 보장.
- commit 메시지 컨벤션 (`feat(scope): ...`, `test(scope): ...`) 일관.

### 대안
- semantic commits (Conventional Commits) — 같은 의미.
- squash merge — 학습 자료 의미 잃음.

### 출처
- TDD 의 small step 사상.
- Linus Torvalds 의 git design.

---

## §∞. 정리

| 문제 | 패턴 |
|---|---|
| 결정 기록 | J1 ADR |
| 컨벤션 자동 강제 | J2 ArchUnit |
| 모듈 경계 자동 강제 | J3 Modulith verify |
| 풀 학습 자료 | J4 learned doc 풀버전 |
| git history = 학습 | J5 1파일=1commit |

### 학습 추천
이 카테고리는 패턴 자체보다 **습관 형성** 이 핵심:
1. 큰 결정 시 ADR 한 줄이라도.
2. 새 컨벤션 도입 시 ArchUnit 룰 같이.
3. doc 작성 시 "한 번에 보임" 원칙.
4. commit 메시지 의미 단위.

이 4가지 습관이 1년 후 큰 차이.
