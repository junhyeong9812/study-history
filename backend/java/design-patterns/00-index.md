# GoF 23 Design Patterns — Java 학습 자료

> Erich Gamma, Richard Helm, Ralph Johnson, John Vlissides (Gang of Four, 1994) 의 *Design Patterns: Elements of Reusable Object-Oriented Software* 에서 제시한 23 개 패턴을 Java 예제와 함께 정리.
>
> 목적: "어떤 구조의 코드를 어떤 이름으로 부르는지" 를 정리하여, 코드 리뷰 / 설계 토론 시 공통 어휘로 사용한다.

---

## 카테고리

GoF 는 패턴을 *목적* 에 따라 3 분류한다.

| 분류 | 의도 | 개수 |
|---|---|---|
| **생성 (Creational)** | 객체 생성 메커니즘. 어떻게 인스턴스를 만들 것인가. | 5 |
| **구조 (Structural)** | 클래스/객체의 합성. 어떻게 큰 구조를 조립할 것인가. | 7 |
| **행위 (Behavioral)** | 객체 간 상호작용 + 책임 분배. 어떻게 협력할 것인가. | 11 |

---

## 23 개 목록

### 생성 패턴 (Creational, 5)

| # | 패턴 | 한 줄 설명 |
|---|---|---|
| 01 | [Abstract Factory](01-abstract-factory.md) | 관련된 객체들의 *family* 를 인터페이스로 생성. 구체 클래스 미지정. |
| 02 | [Builder](02-builder.md) | 복잡한 객체의 *생성 과정* 을 단계별로 분리. 같은 과정으로 다른 표현. |
| 03 | [Factory Method](03-factory-method.md) | 인스턴스 생성을 서브클래스가 결정. 인터페이스만 부모가 정의. |
| 04 | [Prototype](04-prototype.md) | 기존 인스턴스를 복제(clone)하여 새 인스턴스 생성. |
| 05 | [Singleton](05-singleton.md) | 한 클래스의 인스턴스가 *오직 하나*. 전역 접근점 제공. |

### 구조 패턴 (Structural, 7)

| # | 패턴 | 한 줄 설명 |
|---|---|---|
| 06 | [Adapter](06-adapter.md) | 호환되지 않는 인터페이스를 변환. 기존 코드 *재사용*. |
| 07 | [Bridge](07-bridge.md) | 추상과 구현을 분리하여 *독립적으로 확장* 가능. |
| 08 | [Composite](08-composite.md) | 부분-전체 계층을 *동일하게* 다루는 트리 구조. |
| 09 | [Decorator](09-decorator.md) | 객체에 동적으로 책임을 추가. 상속의 대안. |
| 10 | [Facade](10-facade.md) | 서브시스템의 *단순한 인터페이스* 를 외부에 제공. |
| 11 | [Flyweight](11-flyweight.md) | 다수의 미세 객체를 *공유* 하여 메모리 절약. |
| 12 | [Proxy](12-proxy.md) | 다른 객체에 대한 접근을 *대리*. 지연 로딩, 권한, 캐시. |

### 행위 패턴 (Behavioral, 11)

| # | 패턴 | 한 줄 설명 |
|---|---|---|
| 13 | [Chain of Responsibility](13-chain-of-responsibility.md) | 요청을 *처리할 객체* 가 결정될 때까지 핸들러 체인을 따라 전달. |
| 14 | [Command](14-command.md) | 요청을 *객체로* 캡슐화. 큐, undo, 로그 가능. |
| 15 | [Interpreter](15-interpreter.md) | 문법을 클래스로 표현. 작은 DSL 인터프리팅. |
| 16 | [Iterator](16-iterator.md) | 컬렉션의 내부 구조를 노출하지 않고 *순차 접근*. |
| 17 | [Mediator](17-mediator.md) | 객체 간 직접 참조 대신 *중재자* 로 통신. |
| 18 | [Memento](18-memento.md) | 객체 상태를 *외부* 에 저장하여 나중에 복원 (undo). |
| 19 | [Observer](19-observer.md) | 1:N 의존. 한 객체 변경 시 의존하는 객체들에게 *통보*. |
| 20 | [State](20-state.md) | 객체의 *상태에 따라 행동이 바뀜*. 조건문 대신 클래스. |
| 21 | [Strategy](21-strategy.md) | 알고리즘을 캡슐화하여 *교체 가능*. 정책 주입. |
| 22 | [Template Method](22-template-method.md) | 알고리즘의 *뼈대* 는 부모, *세부* 는 자식이 구현. |
| 23 | [Visitor](23-visitor.md) | 객체 구조를 변경하지 않고 *새 연산* 을 추가. |

---

## 학습 순서 추천

### 처음 배우는 사람
**21 → 22 → 19 → 09 → 05 → 03 → 06 → 10**
- 가장 흔하고 직관적인 8 개. 일상에서 쓰이는 빈도 순.

### 객체 생성에 집중
**05 → 03 → 01 → 02 → 04**
- 생성 패턴 5 종을 단순 → 복잡 순으로.

### 협력 구조 이해
**21 → 19 → 14 → 17 → 13**
- 객체들이 어떻게 *역할을 나눠 일을 처리* 하는지.

### 인터뷰 준비 (자주 출제)
**05, 21, 19, 03, 09, 06** — 이 6 개는 거의 항상.

---

## 각 문서의 일관 구조

```
1. 의도 (Intent)
2. 문제 상황 (Motivation)
3. 구조 (Structure) — 텍스트 다이어그램
4. 참여자 (Participants)
5. Java 예제 — 컴파일 가능한 코드
6. 변형 (Variants)
7. 함정 / 흔한 오해
8. 관련 패턴
9. 실무 사례
```

---

## "이 코드는 어떤 패턴인가?" 빠른 식별표

| 코드 단서 | 의심 패턴 |
|---|---|
| `private static instance` + `getInstance()` | Singleton (05) |
| `interface Factory { create() }` 가 여러 구현 | Factory Method (03) / Abstract Factory (01) |
| `setX().setY().build()` 체이닝 | Builder (02) |
| 객체에 같은 인터페이스의 다른 객체를 wrapping | Decorator (09) / Proxy (12) / Adapter (06) |
| `interface X` + `class A implements X`, `class B implements X` 를 런타임에 교체 | Strategy (21) |
| `protected abstract step1(); public final algorithm() { step1(); ... }` | Template Method (22) |
| `subscribe(observer) / notify()` | Observer (19) |
| 트리 노드가 자기 자신을 children 으로 가짐 | Composite (08) |
| `next.handle(request)` 체이닝 | Chain of Responsibility (13) |
| `command.execute() / undo()` | Command (14) |
| 같은 메서드가 *상태 객체* 에 따라 동작 다름 | State (20) |
| 컬렉션 내부 노출 없이 `next() / hasNext()` | Iterator (16) |
| `accept(visitor) → visitor.visitX(this)` | Visitor (23) |

---

## 패턴 ≠ 만능

GoF 의 핵심 메시지는 *언제 어떤 패턴을 쓰는가* 와 함께 **언제 패턴을 쓰지 않는가** 를 아는 것.

흔한 함정 :
1. **패턴 적용 자체가 목적이 됨** — 문제 없는 코드를 패턴으로 *복잡하게* 만듦.
2. **구식 패턴 채택** — Java 8+ 의 `Optional`, lambda, `record` 가 일부 패턴을 단순화 (예: Strategy 가 람다 한 줄).
3. **패턴 이름의 오용** — "이거 Strategy 야" 라고 하지만 사실 단순 if/else.

→ 패턴은 *어휘* 다. 어휘를 알면 토론이 빨라진다. 그러나 모든 코드를 패턴으로 표현할 필요는 없다.

---

## 추가 참고

- Erich Gamma 외, *Design Patterns: Elements of Reusable Object-Oriented Software* (Addison-Wesley, 1994) — 원전
- *Head First Design Patterns* (Eric Freeman, 2004) — 그림과 비유로 읽기 쉬움
- Joshua Bloch, *Effective Java* — Java 관점에서의 패턴 보강 (Builder, Singleton 안전 구현 등)
- Refactoring.Guru: https://refactoring.guru/design-patterns — 시각적 설명 + 다국어
- Sourcemaking: https://sourcemaking.com/design_patterns
