# 09 — Decorator

> **분류**: 구조 패턴 (Structural)
> **한 줄**: 객체에 *동적으로* 책임을 추가. 상속의 대안.

---

## 1. 의도 (Intent)

> "Attach additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality."

기능을 *조합* 해서 객체를 만들고 싶을 때. 상속으로 풀면 클래스 폭발.

## 2. 문제 상황 (Motivation)

커피 주문 — 베이스 + 옵션 (우유, 설탕, 휘핑) :

```java
// ❌ 모든 조합을 클래스로
class Coffee { ... }
class CoffeeWithMilk { ... }
class CoffeeWithMilkAndSugar { ... }
class CoffeeWithSugar { ... }
class CoffeeWithMilkAndSugarAndWhip { ... }
// → 옵션 N 개면 2^N 클래스
```

해결 : 옵션을 *데코레이터* 로 감싸기.

```java
Beverage b = new Espresso();
b = new Milk(b);
b = new Sugar(b);
b = new Whip(b);
// b.cost() = espresso + milk + sugar + whip 가 자동 계산
```

## 3. 구조 (Structure)

```
Component (interface)
   ┌────────────┐
   │ operation()│
   └─────┬──────┘
         │
    ┌────┴────┐
    │         │
ConcreteComponent   Decorator (abstract)
   operation()      ┌─────────────────┐
                    │ - inner: Component
                    │ + operation() {  │
                    │    inner.operation()
                    │    + 추가 행동   │
                    │ }                │
                    └────┬─────────────┘
                         │
                    ┌────┴─────┐
                    │          │
              ConcreteDecA   ConcreteDecB
```

## 4. 참여자 (Participants)

- **Component**: 인터페이스.
- **ConcreteComponent**: 기본 구현.
- **Decorator**: Component 구현 + 같은 Component 참조 보유.
- **ConcreteDecorator**: 추가 기능.

## 5. Java 예제

```java
// Component
interface Beverage {
    String description();
    double cost();
}

// ConcreteComponent
class Espresso implements Beverage {
    @Override public String description() { return "Espresso"; }
    @Override public double cost() { return 3.0; }
}

class HouseBlend implements Beverage {
    @Override public String description() { return "House Blend"; }
    @Override public double cost() { return 2.5; }
}

// Decorator (abstract)
abstract class CondimentDecorator implements Beverage {
    protected final Beverage inner;
    protected CondimentDecorator(Beverage inner) {
        this.inner = inner;
    }
}

// Concrete Decorators
class Milk extends CondimentDecorator {
    public Milk(Beverage b) { super(b); }
    @Override public String description() { return inner.description() + ", Milk"; }
    @Override public double cost() { return inner.cost() + 0.5; }
}

class Sugar extends CondimentDecorator {
    public Sugar(Beverage b) { super(b); }
    @Override public String description() { return inner.description() + ", Sugar"; }
    @Override public double cost() { return inner.cost() + 0.2; }
}

class Whip extends CondimentDecorator {
    public Whip(Beverage b) { super(b); }
    @Override public String description() { return inner.description() + ", Whip"; }
    @Override public double cost() { return inner.cost() + 0.7; }
}

// 사용
public class Main {
    public static void main(String[] args) {
        Beverage b = new Espresso();
        b = new Milk(b);
        b = new Sugar(b);
        b = new Whip(b);

        System.out.println(b.description());    // Espresso, Milk, Sugar, Whip
        System.out.printf("$%.2f%n", b.cost()); // $4.40
    }
}
```

각 데코레이터가 *얇음*. 책임 한 가지만. 조합으로 다양한 결과.

## 6. 변형 (Variants)

### 6.1 Function composition (Java 8+)

```java
Function<Beverage, Beverage> withMilk = Milk::new;
Function<Beverage, Beverage> withSugar = Sugar::new;

Beverage b = withMilk.andThen(withSugar).apply(new Espresso());
```

함수형 스타일. 데코레이터를 *함수* 로 보는 관점.

### 6.2 Stateful Decorator
데코레이터가 자기 상태 가짐 (예 : 호출 횟수 카운트). 조심 — 같은 데코레이터를 여러 곳에서 공유하면 race condition.

## 7. 함정 / 흔한 오해

### 7.1 Adapter (06) / Proxy (12) 와의 차이
- **Adapter**: 인터페이스 *변환* (다른 인터페이스).
- **Proxy**: 같은 인터페이스, 접근 제어 / 지연 / 캐시.
- **Decorator**: 같은 인터페이스, *기능 추가*.
- 셋 다 wrapping 형태. 의도가 다름.

### 7.2 무한 wrapping
새 행동마다 데코레이터를 추가 → wrapping 깊이가 깊어져 디버깅 / 스택 트레이스 복잡.

### 7.3 순서 의존
`new Sugar(new Milk(b))` 와 `new Milk(new Sugar(b))` 가 다를 수 있음 — 비가환 연산.

## 8. 관련 패턴

- **Composite (08)**: 단일 자식 chain 으로 보면 Decorator.
- **Strategy (21)**: 알고리즘 자체를 교체. Decorator 는 *추가*.
- **Adapter (06) / Proxy (12)**: wrapping 친척.

## 9. 실무 사례

- `java.io.BufferedReader`, `BufferedInputStream` — 버퍼링 추가
- `java.io.GZIPInputStream` — 압축 해제 추가
- `Collections.synchronizedList(list)` — 동기화 추가
- `Collections.unmodifiableList(list)` — 읽기 전용 추가
- Servlet `HttpServletRequestWrapper` — request 변형
- Spring AOP 의 일부 (advice 로 메서드 wrapping)
- Java Stream: `stream.filter(...).map(...).distinct()` — 함수형 데코레이터 체인

### Java I/O 의 데코레이터 체인

```java
// 파일 → 버퍼 → 데이터 디코딩 → 문자
DataInputStream in = new DataInputStream(           // 데이터 디코딩
    new BufferedInputStream(                         // 버퍼링
        new FileInputStream("data.bin")));           // 기본 컴포넌트
```

각 InputStream 데코레이터가 한 가지 책임. 자유 조합.
