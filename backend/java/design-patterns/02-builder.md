# 02 — Builder

> **분류**: 생성 패턴 (Creational)
> **한 줄**: 복잡한 객체의 *생성 과정* 을 단계별로 분리하여, 같은 과정으로 다른 표현을 만들 수 있게 한다.

---

## 1. 의도 (Intent)

> "Separate the construction of a complex object from its representation so that the same construction process can create different representations."

생성자 매개변수가 많고 일부는 선택적일 때 — 가독성과 안전성을 위해 *체이닝 + 단계별 설정* 으로 객체를 만든다.

## 2. 문제 상황 (Motivation)

```java
// ❌ Telescoping constructor anti-pattern
public Pizza(String size) { ... }
public Pizza(String size, boolean cheese) { ... }
public Pizza(String size, boolean cheese, boolean pepperoni) { ... }
public Pizza(String size, boolean cheese, boolean pepperoni, boolean mushroom) { ... }
// → 매개변수가 많아질수록 폭발

// 또는
new Pizza("L", true, false, true, false, true);
// → 각 boolean 이 무엇인지 호출부에서 안 보임
```

해결 :

```java
Pizza p = new Pizza.Builder("L")
    .cheese(true)
    .pepperoni(true)
    .mushroom(false)
    .build();
```

각 옵션 이름이 보이고, 순서가 자유롭고, 필수 (size) 와 선택을 구분할 수 있다.

## 3. 구조 (Structure)

```
Director (옵션)         Builder (interface)
  │                    ┌──────────────┐
  └── construct() ──►  │ buildPart1() │
                       │ buildPart2() │
                       │ getResult()  │
                       └──────┬───────┘
                              │
                       ConcreteBuilder
                              │
                              ▼
                          Product
```

## 4. 참여자 (Participants)

- **Builder**: 부분 생성 메서드 + `build()` 정의.
- **ConcreteBuilder**: 실제 생성 로직. 내부에 *partial product* 를 들고 있다가 build() 시 반환.
- **Director (선택)**: 정해진 순서로 builder 호출. 생략 가능.
- **Product**: 최종 결과.

## 5. Java 예제

### 5.1 정통 Builder (Joshua Bloch 스타일, *Effective Java*)

```java
public class Pizza {
    // 모든 필드 final — 불변
    private final String size;
    private final boolean cheese;
    private final boolean pepperoni;
    private final boolean mushroom;

    private Pizza(Builder b) {
        this.size = b.size;
        this.cheese = b.cheese;
        this.pepperoni = b.pepperoni;
        this.mushroom = b.mushroom;
    }

    public static class Builder {
        // 필수
        private final String size;
        // 선택 — 기본값
        private boolean cheese = false;
        private boolean pepperoni = false;
        private boolean mushroom = false;

        public Builder(String size) {
            this.size = size;
        }

        public Builder cheese(boolean v) { this.cheese = v; return this; }
        public Builder pepperoni(boolean v) { this.pepperoni = v; return this; }
        public Builder mushroom(boolean v) { this.mushroom = v; return this; }

        public Pizza build() {
            // 검증을 build 시점에 (선택)
            if (size == null) throw new IllegalStateException("size required");
            return new Pizza(this);
        }
    }

    @Override
    public String toString() {
        return "Pizza[%s, cheese=%s, pepp=%s, mush=%s]"
            .formatted(size, cheese, pepperoni, mushroom);
    }
}

// 사용
Pizza p = new Pizza.Builder("L")
    .cheese(true)
    .pepperoni(true)
    .build();
```

### 5.2 Director 가 있는 형태 (GoF 원전)

```java
// Director 가 *순서* 를 안다 — 도메인 지식
class HouseDirector {
    public void buildLuxury(HouseBuilder b) {
        b.foundation();
        b.walls();
        b.roof();
        b.pool();
        b.garage();
    }

    public void buildBasic(HouseBuilder b) {
        b.foundation();
        b.walls();
        b.roof();
    }
}
```

→ Director 는 *어떤 순서로 builder 의 메서드를 부를지* 안다. Pizza 같은 단순 케이스엔 불필요.

## 6. 변형 (Variants)

- **Lombok `@Builder`**: 어노테이션 한 줄로 builder 자동 생성.
- **Java 14+ `record`**: 불변 객체에 builder 가 사실상 불필요한 경우도 있음 (단, optional 매개변수가 많으면 여전히 builder).
- **Step Builder**: 컴파일 타임에 *순서* 를 강제 (필수 필드 빠뜨림 방지).

```java
// Step builder — size 후 cheese, cheese 후 build 만 가능
new PizzaBuilder().size("L").cheese(true).build();
// .cheese 호출 안 하고 .build 시도하면 컴파일 에러
```

## 7. 함정 / 흔한 오해

- **불필요한 Builder**: 매개변수 2~3 개면 일반 생성자가 더 간결.
- **Builder 결과의 mutability**: Pizza 가 final 필드여도 Builder 내부에 *컬렉션* 이 있으면 외부로 노출되어 변경될 수 있음 → `Collections.unmodifiableList`.
- **Builder 와 Fluent Interface 혼동**: 메서드 체이닝은 fluent interface, builder 는 *생성 분리*. 모든 fluent 가 builder 는 아님.

## 8. 관련 패턴

- **Abstract Factory (01)**: family 생성. Builder 는 *한 객체의 단계별 생성*.
- **Composite (08)**: Builder 가 Composite 트리를 만들 때 자연.

## 9. 실무 사례

- `StringBuilder` (단계별 문자열 조립)
- `java.lang.StringBuilder` / `StringBuffer`
- `Stream.Builder<T>`
- Lombok `@Builder`
- Spring `RestTemplateBuilder`, `UriComponentsBuilder`
- HTTP request builder: `HttpRequest.newBuilder().uri(...).GET().build()`
