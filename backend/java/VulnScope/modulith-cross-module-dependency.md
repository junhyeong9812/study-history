# 모듈 간 의존, Modulith 식 vs 헥사고날 정통

> 작성일: 2026-05-01
> 범위: VulnScope 가 모듈 간 호출에서 어떤 결합 모델을 채택했는지, 왜 그게 헥사고날과 다르게 보이지만 정당한지.
> 진입 코드: `back/src/main/java/com/vulnscope/scan/application/service/TriggerScanService.java`

---

## 1. 문제 — 한 모듈이 다른 모듈의 정보가 필요할 때

`TriggerScanService.trigger(...)` 는 스캔을 큐에 넣기 전에 두 가지 사전 검증이 필요하다:

1. **타겟이 이 조직 소유인가** — `target` 모듈이 답할 수 있음
2. **프로파일이 존재하고 이 조직에 보이는가** — `profile` 모듈이 답할 수 있음

이걸 어떻게 받아오느냐 — 이게 **모듈 간 의존을 어떻게 그릴 것인가** 라는 아키텍처 결정이다.

---

## 2. 현재 구현 — 다른 모듈의 `api` 인터페이스 직접 import

`scan/application/service/TriggerScanService.java:1-19`:

```java
package com.vulnscope.scan.application.service;

import com.vulnscope.profile.application.api.ProfileQuery;   // ← 다른 모듈
import com.vulnscope.target.application.api.TargetQuery;     // ← 다른 모듈
// ... 나머지 import 는 자기 모듈 또는 shared/kernel ...

@Service
public class TriggerScanService {

    private final ScanRepository repository;
    private final TargetQuery targetQuery;     // ← 주입받음
    private final ProfileQuery profileQuery;   // ← 주입받음
    // ...

    public Scan trigger(OrgId orgId, TargetId targetId, ProfileId profileId) {
        if (!targetQuery.isOwnedBy(targetId, orgId)) {
            throw new TargetNotAccessible(targetId);
        }
        if (!profileQuery.existsAndVisibleTo(profileId, orgId)) {
            throw new InvalidProfile(profileId);
        }
        // ...
    }
}
```

**이게 헥사고날 정통과 다른 점**: 보통 헥사고날이라면 `scan` 모듈이 자기 안에 `TargetAccessPort` 같은 **자기 어휘로 된 포트** 를 정의하고, infrastructure 어댑터에서 그 포트를 `TargetQuery` 로 변환했을 것이다. 지금은 그 변환 계층 없이 **다른 모듈이 공개한 인터페이스를 그대로 사용**한다.

---

## 3. "그래도 되는" 두 개의 보호 장치

### 보호 장치 1 — `@NamedInterface("api")`

`target/application/api/package-info.java`:
```java
@org.springframework.modulith.NamedInterface("api")
package com.vulnscope.target.application.api;
```

`profile/application/api/package-info.java`:
```java
@org.springframework.modulith.NamedInterface("api")
package com.vulnscope.profile.application.api;
```

**의미**: 이 패키지를 **다른 모듈의 공식 출입구로 선언**. Spring Modulith 의 `verify()` 는:
- ✅ `scan → target.application.api` (공식 출입구) — 통과
- ❌ `scan → target.domain.Target` (내부) — 차단
- ❌ `scan → target.infrastructure.InMemoryTargetRepository` (내부) — 차단

→ 결합이 일어나도 **공식 표면 한 점에서만** 일어나도록 컴파일/검증 단계에서 강제.

### 보호 장치 2 — ArchUnit 도메인 격리 규칙

`back/src/test/java/com/vulnscope/architecture/ArchitectureTest.java`:
```java
@ArchTest
static final ArchRule domain_must_not_depend_on_outside =
    noClasses().that().resideInAPackage("..domain..")
        .should().dependOnClassesThat().resideInAnyPackage(
            "..application..", "..presentation..", "..infrastructure..", ...
        );
```

→ 도메인은 절대 application 의 인터페이스에도 의존 못 함. 모듈 간 결합은 **application 이 다른 application 의 api** 까지만 허용.

---

## 4. 두 모델의 비교

|  | 헥사고날 정통 | Modulith 식 (현재 채택) |
|---|---|---|
| **인터페이스 소유자** | 컨슈머 (scan 이 정의) | 프로듀서 (target 이 공개) |
| **인터페이스 어휘** | 컨슈머 도메인어 (`canScan`) | 프로듀서 도메인어 (`isOwnedBy`) |
| **어댑터** | scan/infrastructure 에 1개 필수 | 없음 |
| **컴파일 의존 그래프** | scan → 자기 포트만 | scan → target.application.api |
| **언어 결합** | 끊김 (어댑터가 흡수) | 존재 (`isOwnedBy` 변경되면 scan 도 영향) |
| **모듈 수 N 일 때 추가 코드** | N²/2 쌍 (포트+어댑터) | 0 |
| **테스트 격리** | ✅ 인터페이스라 mock 가능 | ✅ 인터페이스라 mock 가능 |
| **마이크로서비스 추출** | 매끄러움 | 인터페이스를 RPC schema 로 변환 필요 |

→ **테스트 독립성은 동일**. 두 모델 모두 인터페이스 기반이라 mock 으로 끊긴다. 차이는 **언어 결합과 변환 계층의 존재** 뿐.

---

## 5. 변경 영향 분석 — 이 결정이 무엇을 보장하고 무엇을 양보하는가

target 모듈 안에서 무언가가 바뀔 때 scan 이 받는 영향:

| 변경 위치 | scan 영향 | 이유 |
|---|---|---|
| `target.domain.Target` 필드 추가/수정 | ❌ 없음 | api 표면 밖, scan 은 못 봄 |
| `target.domain.RegisterTarget` 로직 변경 | ❌ 없음 | 내부 |
| `target.infrastructure.InMemoryTargetRepository` → JPA 로 교체 | ❌ 없음 | 인프라 교체, 인터페이스 동일 |
| `target.application.service.TargetQueryService` 내부 구현 변경 | ❌ 없음 | 인터페이스 구현 디테일 |
| **`TargetQuery.isOwnedBy(...)` 시그니처 변경** | ✅ **있음** | 공식 계약 변경 |
| **`TargetQuery` 메서드 추가** | ❌ 없음 | 기존 컨슈머는 무영향 |

→ scan 이 신경 쓰는 건 **공식 표면(`TargetQuery` 인터페이스) 단 하나**. 나머지는 전부 자유.

이게 마이크로서비스의 "OpenAPI 스펙은 안 깨고 내부는 마음껏 리팩토링" 과 동일한 모델인데, 같은 JVM 안에서 Java interface 가 그 스펙 역할을 한다.

---

## 6. 사상 (Why) — 왜 이 트레이드오프를 선택했는가

### (a) 모듈 수에 비례하는 어댑터 폭증을 피한다

11개 모듈 × 평균 N개 의존 시, 헥사고날 정통은 **포트+어댑터 쌍이 N×N 증가**. 단일 deployable 단일 팀에선 가성비가 안 맞는다.

### (b) Modulith 가 이미 "공식 출입구" 강제력을 제공한다

헥사고날의 핵심 가치 중 "**내부 구현이 새지 않게 한다**" 부분은 `@NamedInterface` + `verify()` 가 이미 보장. 여기서 헥사고날을 한 번 더 얹는 것은 **이중 방어**.

### (c) 학습 비용과 가독성

`scan.application` 코드를 처음 보는 사람이 "isOwnedBy 가 어디서 오나" 추적할 때:
- 헥사고날: scan 의 포트 → 어댑터 → target API → target service → ... (4단계)
- Modulith: scan → target API → target service → ... (3단계)

한 단계 적은 게 별것 아닌 듯해도, 모듈마다 적용되면 누적 효과가 크다.

### (d) 마이크로서비스 추출은 "그때 가서" 도 늦지 않다

진짜로 target 을 떼어내야 할 때, 그 시점에 `TargetQuery` 를 RPC 클라이언트로 감싸는 어댑터를 추가하면 된다. **선반영(over-engineering) 보다 후반영(refactor when needed) 이 cheaper**.

---

## 7. 사상의 두 층 — 모듈 내부와 모듈 간을 다르게 다룬다

이 프로젝트는 결합 정책을 **두 층으로 분리**해서 적용한다:

```
┌─────────────────────────────────────────────────────┐
│ 모듈 내부 (예: scan/)                                │
│   엄격한 헥사고날                                     │
│   domain ← application ← infrastructure              │
│   ArchUnit 으로 강제                                  │
├─────────────────────────────────────────────────────┤
│ 모듈 간 (scan ↔ target/profile/...)                  │
│   Modulith 관용                                      │
│   다른 모듈의 application/api 를 직접 의존            │
│   @NamedInterface 와 verify() 로 출입구만 강제       │
└─────────────────────────────────────────────────────┘
```

**규칙 한 줄**: "내부는 단단히, 경계는 가볍게."

---

## 8. 언제 이 결정을 뒤집어야 하나 (재평가 트리거)

다음 중 하나라도 발생하면 헥사고날 정통으로 전환하는 게 맞다:

1. **target API 의 어휘가 자주 바뀌어 scan 컴파일이 자주 깨진다** → 어댑터로 흡수
2. **target 을 별도 프로세스/서비스로 추출하기로 결정했다** → 포트가 통합 경계
3. **scan-only 정책 (캐싱, 추가 권한 검사) 을 가운데 끼우고 싶다** → 어댑터 위치
4. **target 팀과 scan 팀이 분리되어 변경 협의가 비싸졌다** → 계약을 더 두껍게

지금은 셋 다 해당 안 됨 → 현 결정 유지.

---

## 9. 코드 인용 모음 (학습용)

다른 모듈의 api 를 import 하는 위치들:

```bash
# 직접 검색해보기
grep -rn "import com.vulnscope.target.application.api" back/src/main
grep -rn "import com.vulnscope.profile.application.api" back/src/main
grep -rn "import com.vulnscope.upload.application.api" back/src/main  # evidence → upload
```

이 import 들이 전부 **공식 표면(`*.application.api`)** 으로만 가는지 확인. `*.domain` 또는 `*.infrastructure` 로 가는 import 가 있으면 즉시 위반 → ArchitectureTest 가 잡아야 한다.

---

## 10. 한 줄 요약 (꼭 기억할 것)

> **"테스트 격리는 인터페이스가 책임지고, 모듈 격리는 NamedInterface 가 책임지고, 의미론적 결합 비용은 수용한다."**

이게 VulnScope 의 모듈 간 의존 정책. 헥사고날과 다르게 보여도, 같은 가치(독립성, 테스트성, 변경 격리) 를 **다른 메커니즘으로** 달성한다.
