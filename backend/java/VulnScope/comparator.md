# Comparator 정리 (VulnScope 코드 기반)

> 작성일: 2026-05-01
> 범위: `java.util.Comparator` 와 `Stream.sorted()` 의 결합. 이 프로젝트의 실제 사용처를 기준으로 설명.

---

## 1. 한 줄 요약

`Comparator<T>` 는 **"T 두 개를 받아서 어느 쪽이 앞인지 결정하는 규칙"** 이다.
정렬할 때 컬렉션이나 스트림에 이 규칙을 넘겨주면, 그 규칙대로 줄 세운다.

```java
Comparator<Scan> byCreatedDesc =
    Comparator.comparing(Scan::createdAt, Comparator.reverseOrder());

list.stream().sorted(byCreatedDesc).toList();
```

`compare(a, b)` 의 반환값 규약:
- 음수 → `a` 가 앞
- 0   → 동등
- 양수 → `b` 가 앞

직접 `(a, b) -> ...` 람다로 짤 수도 있지만, **필드를 뽑아서 그 필드끼리 비교**하는 패턴이 압도적으로 많아서 `Comparator.comparing(...)` 헬퍼가 표준이다.

---

## 2. 핵심 시그니처 3개

```java
// (1) 키만 뽑으면 끝 — 키의 자연 순서(natural order) 사용
static <T, U extends Comparable<? super U>> Comparator<T> comparing(
    Function<? super T, ? extends U> keyExtractor
);

// (2) 키를 뽑고, 그 키를 비교할 방법까지 직접 지정
static <T, U> Comparator<T> comparing(
    Function<? super T, ? extends U> keyExtractor,
    Comparator<? super U> keyComparator
);

// (3) primitive 전용 (박싱 비용 없음)
static <T> Comparator<T> comparingInt(ToIntFunction<? super T> keyExtractor);
static <T> Comparator<T> comparingLong(ToLongFunction<? super T> keyExtractor);
static <T> Comparator<T> comparingDouble(ToDoubleFunction<? super T> keyExtractor);
```

자주 쓰는 헬퍼:

| 헬퍼 | 의미 |
|---|---|
| `Comparator.naturalOrder()` | 타입의 `compareTo` 그대로 사용 (오름차순) |
| `Comparator.reverseOrder()` | 자연 순서를 뒤집음 (내림차순) |
| `comparator.reversed()` | 인스턴스 메서드 — 만든 비교기를 뒤집음 |
| `Comparator.nullsFirst(c)` | null 을 가장 앞으로 |
| `Comparator.nullsLast(c)` | null 을 가장 뒤로 |
| `c1.thenComparing(c2)` | 1차 키가 같을 때 2차 키로 정렬 |

---

## 3. 이 프로젝트의 실제 사용 5가지 패턴

### 패턴 A — 단일 필드 내림차순 (가장 흔함)

`scan/infrastructure/persistence/InMemoryScanRepository.java:38-44`

```java
@Override
public List<Scan> findByTarget(TargetId targetId) {
    return byId.values().stream()
        .filter(s -> s.targetId().equals(targetId))
        .sorted(Comparator
                .comparing(Scan::createdAt, Comparator.reverseOrder()))
        .toList();
}
```

**해석**:
- `Scan::createdAt` — 각 `Scan` 에서 `Instant createdAt` 을 뽑아낸다.
- `Comparator.reverseOrder()` — 그 `Instant` 들을 자연 순서의 **반대**(과거→미래의 반대 = 미래→과거)로 비교한다.
- 결과: **최신 스캔이 앞**으로 오는 List.

직접 람다로 풀면:
```java
.sorted((a, b) -> b.createdAt().compareTo(a.createdAt()))
```
하지만 가독성과 타입 안전성 때문에 `Comparator.comparing` 형태가 표준이다.

> 같은 패턴이 `findByOrg`, `findAllByOrg` 에도 동일하게 등장 (`Repository:46-62`).

---

### 패턴 B — 단일 필드 오름차순 (자연 순서)

`evidence/infrastructure/persistence/InMemoryEvidenceRepository.java:39`

```java
.sorted(Comparator.comparing(Evidence::createdAt))
```

**해석**: 두 번째 인자(키 비교기)를 생략하면 키의 **자연 순서**가 적용된다. `Instant` 의 자연 순서는 과거→미래이므로 → **오래된 evidence 먼저** = 첨부 시점 순서.

> 키 타입이 `Comparable` 을 구현해야 한다. `String`, `Integer`, `Instant`, `LocalDateTime`, `enum` 모두 OK.

---

### 패턴 C — primitive long 정렬 (박싱 회피)

`finding/infrastructure/persistence/InMemoryFindingRepository.java:42, 50`

```java
.sorted(Comparator.comparingLong(Finding::seq))
```

**해석**:
- `Finding::seq` 의 반환 타입은 `long` (primitive).
- `Comparator.comparing(Finding::seq)` 로 쓰면 매번 `Long` 으로 박싱 → GC 부담.
- `comparingLong` 은 `ToLongFunction` 을 받아 박싱 없이 비교한다.

**언제 쓰나**: 키가 `int/long/double` 같은 primitive 일 때. 항상 `comparingInt/Long/Double` 우선 고려.

> `Finding.seq` 는 한 스캔 내에서 단조 증가하는 시퀀스. 자연 오름차순 = 발견된 순서.

---

### 패턴 D — null 처리 + 다단계 정렬 (가장 복잡)

`target/infrastructure/persistence/InMemoryTargetRepository.java:46-53`

```java
@Override
public List<Target> findRecentByOrg(OrgId orgId, int limit) {
    return byId.values().stream()
        .filter(t -> t.orgId().equals(orgId))
        .sorted(Comparator.comparing(
                    Target::lastUsedAt,
                    Comparator.nullsLast(Comparator.reverseOrder()))
            .thenComparing(Target::createdAt, Comparator.reverseOrder()))
        .limit(limit)
        .toList();
}
```

**해석 (안쪽부터)**:

1. `Target::lastUsedAt` — 마지막 사용 시각 (`Instant`, **null 가능** — 한 번도 안 쓴 타겟).
2. `Comparator.reverseOrder()` — 자연 순서의 반대. 단, **null 을 만나면 NPE 터짐**.
3. `Comparator.nullsLast(Comparator.reverseOrder())` — null 을 **항상 맨 뒤**로 보내고, null 아닌 것들 사이에서는 reverseOrder 적용. → **"최근에 쓴 것 → 옛날에 쓴 것 → 한 번도 안 쓴 것"** 순서.
4. `.thenComparing(Target::createdAt, Comparator.reverseOrder())` — 1차 키(`lastUsedAt`)가 같을 때(예: 둘 다 null), `createdAt` 내림차순으로 보조 정렬.

**왜 nullsLast 가 필요한가**: `Instant.compareTo(null)` → NPE. `Comparator.reverseOrder()` 도 내부적으로 `compareTo` 를 호출하므로 마찬가지. `nullsFirst`/`nullsLast` 로 감싸야 안전.

---

### 패턴 E — 람다 + thenComparing

`profile/infrastructure/persistence/InMemoryProfileRepository.java:32-40`

```java
@Override
public List<Profile> findVisibleTo(OrgId orgId) {
    return byId.values().stream()
        .filter(p -> p.isVisibleTo(orgId))
        .sorted(Comparator
            .comparing((Profile p) -> p.kind().ordinal())
            .thenComparing(Profile::name))
        .toList();
}
```

**해석**:
1. `(Profile p) -> p.kind().ordinal()` — enum 의 `ordinal()` 로 정렬 키를 만든다. `ProfileKind.SYSTEM` 이 먼저 오게 하려고 enum 선언 순서를 정렬 키로 활용.
2. `.thenComparing(Profile::name)` — 같은 kind 안에서는 이름 알파벳 오름차순.

**왜 `(Profile p) ->` 라고 타입을 명시했나**: 메서드 참조 `Profile::ordinal` 같은 게 없고, 람다 파라미터 타입을 컴파일러가 추론할 단서가 부족하기 때문. 첫 `comparing` 의 제네릭이 잡혀야 다음 `thenComparing` 에서 `Profile::name` 같은 메서드 참조가 해석된다.

> `comparing(Function)` 단독 호출 시 타입 추론이 종종 막혀서 **첫 람다에 명시적 타입**을 넣는 게 흔한 트릭.

---

## 4. 자주 헷갈리는 포인트

### (a) `reversed()` vs `Comparator.reverseOrder()`

```java
Comparator.comparing(Scan::createdAt).reversed()
// vs
Comparator.comparing(Scan::createdAt, Comparator.reverseOrder())
```

둘 다 결과는 같다. 차이는:
- `.reversed()` — 만들어진 Comparator 전체를 뒤집음.
- 두 번째 인자에 `reverseOrder()` — **키 비교 방식**만 뒤집음.

`thenComparing` 과 결합할 때 차이가 생긴다:

```java
// 1차 키만 내림차순, 2차 키는 오름차순
Comparator.comparing(Target::lastUsedAt, Comparator.reverseOrder())
    .thenComparing(Target::name)

// 1차+2차 둘 다 내림차순 (전체를 뒤집음)
Comparator.comparing(Target::lastUsedAt)
    .thenComparing(Target::name)
    .reversed()
```

→ 패턴 D 가 두 번째 인자 형태를 쓴 이유: **1차만 뒤집고 싶고**, 2차(`createdAt`)는 따로 또 reverseOrder 를 줘서 명시적으로 제어하려 함.

### (b) `compareTo` 의 NPE 위험

`Instant`, `String`, `LocalDateTime` 등은 모두 자연 비교 시 null 이면 NPE. **null 이 가능한 필드**는 항상 `nullsFirst`/`nullsLast` 로 감싸자.

### (c) primitive 박싱

키가 `int/long/double` 이면 `comparingInt/Long/Double` 사용. 의식하지 않으면 매 비교마다 박싱이 일어난다.

### (d) `sorted()` 는 안정 정렬(stable)

같은 키 값이면 원래 순서를 유지한다. `thenComparing` 이 없어도 입력 순서가 보존된다.

### (e) 무한 스트림에서 `sorted` 금지

`sorted` 는 모든 원소를 모아야 한다. `Stream.generate(...).sorted()` 는 영원히 끝나지 않는다.

---

## 5. 체크리스트 (Comparator 쓸 때)

- [ ] 키 타입이 `Comparable` 인가? → 아니면 두 번째 인자에 비교기 명시.
- [ ] 키가 null 가능한가? → `nullsFirst`/`nullsLast` 로 감싸기.
- [ ] 키가 primitive 인가? → `comparingInt/Long/Double` 사용.
- [ ] 1차 키 동률 처리가 필요한가? → `thenComparing` 추가.
- [ ] 오름/내림 어느 쪽? → `reverseOrder()` 또는 `.reversed()`.
- [ ] 컬렉션이 크고 정렬이 빈번한가? → 인메모리 스트림 정렬 대신 인덱스/DB 정렬 고려.

---

## 6. 참고 — 이 프로젝트의 정렬 결정 요약

| 위치 | 키 | 방향 | null 처리 | 보조 키 |
|---|---|---|---|---|
| `InMemoryScanRepository.findByTarget/findByOrg/findAllByOrg` | `Scan.createdAt` (Instant) | 내림차순 | 없음 | 없음 |
| `InMemoryEvidenceRepository` | `Evidence.createdAt` (Instant) | 오름차순 (자연) | 없음 | 없음 |
| `InMemoryFindingRepository` | `Finding.seq` (long) | 오름차순 (자연, 박싱X) | 없음 | 없음 |
| `InMemoryTargetRepository.findRecentByOrg` | `Target.lastUsedAt` (Instant?) | 내림차순 | nullsLast | `createdAt` 내림차순 |
| `InMemoryProfileRepository.findVisibleTo` | `Profile.kind().ordinal()` (int via 람다) | 오름차순 | 없음 | `Profile.name` 오름차순 |

읽고 나면, 다섯 패턴 모두 **"키 추출 + 비교 방식"** 이라는 같은 골격임이 보일 것.
