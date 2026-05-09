# 01 — Abstract Factory

> **분류**: 생성 패턴 (Creational)
> **한 줄**: 관련된 객체들의 *family* 를 인터페이스로 생성한다. 구체 클래스를 명시하지 않는다.

---

## 1. 의도 (Intent)

서로 관련 있는 여러 종류의 객체를 *함께 생성* 해야 할 때, 클라이언트가 구체 클래스를 모른 채 *family 단위로* 객체를 얻을 수 있게 한다.

> "Provide an interface for creating families of related or dependent objects without specifying their concrete classes."

## 2. 문제 상황 (Motivation)

GUI 프레임워크가 두 가지 룩앤필을 지원한다고 하자 — `Light`, `Dark`. 각 테마는 자기만의 `Button`, `Checkbox`, `TextField` 가 있어서 *섞이면 안 된다*.

```java
// ❌ 구체 클래스 직접 사용
Button btn;
if (theme.equals("light")) {
    btn = new LightButton();
} else {
    btn = new DarkButton();
}
// 새 위젯 (Slider) 추가 시 같은 분기를 또 작성해야 함
```

분기가 코드 곳곳에 흩어진다. 새 테마 (예 : `HighContrast`) 추가 시 모든 분기를 다 찾아 고쳐야 함.

## 3. 구조 (Structure)

```
       AbstractFactory                    ProductA (interface)
      ┌──────────────┐                   ┌──────────┐
      │ createA()    │ ─── creates ────► │          │
      │ createB()    │                   └────┬─────┘
      └──────┬───────┘                        │
             │                          ┌─────┴─────┐
     ┌───────┴───────┐                  │           │
     │               │                Product   Product
ConcreteFactory1  ConcreteFactory2     A1         A2
     │ createA() → A1
     │ createB() → B1
```

## 4. 참여자 (Participants)

- **AbstractFactory**: 제품 family 의 생성 메서드 선언 (`createButton()`, `createCheckbox()`).
- **ConcreteFactory**: 특정 family 의 구체 객체 생성.
- **AbstractProduct**: 각 제품 종류의 인터페이스.
- **ConcreteProduct**: 특정 family 의 구체 제품.
- **Client**: AbstractFactory 와 AbstractProduct 만 사용.

## 5. Java 예제

```java
// 제품 인터페이스
interface Button {
    void render();
}

interface Checkbox {
    void toggle();
}

// Light family
class LightButton implements Button {
    @Override public void render() { System.out.println("Light Button rendered"); }
}

class LightCheckbox implements Checkbox {
    @Override public void toggle() { System.out.println("Light Checkbox toggled"); }
}

// Dark family
class DarkButton implements Button {
    @Override public void render() { System.out.println("Dark Button rendered"); }
}

class DarkCheckbox implements Checkbox {
    @Override public void toggle() { System.out.println("Dark Checkbox toggled"); }
}

// Abstract Factory
interface UIFactory {
    Button createButton();
    Checkbox createCheckbox();
}

// Concrete Factories
class LightUIFactory implements UIFactory {
    @Override public Button createButton() { return new LightButton(); }
    @Override public Checkbox createCheckbox() { return new LightCheckbox(); }
}

class DarkUIFactory implements UIFactory {
    @Override public Button createButton() { return new DarkButton(); }
    @Override public Checkbox createCheckbox() { return new DarkCheckbox(); }
}

// Client
class Application {
    private final Button button;
    private final Checkbox checkbox;

    Application(UIFactory factory) {
        this.button = factory.createButton();
        this.checkbox = factory.createCheckbox();
    }

    void render() {
        button.render();
        checkbox.toggle();
    }
}

// 사용
public class Main {
    public static void main(String[] args) {
        UIFactory factory = new DarkUIFactory();    // 한 줄로 family 결정
        Application app = new Application(factory);
        app.render();
    }
}
```

`Application` 은 `LightButton` / `DarkButton` 같은 구체 클래스를 모른다. `UIFactory` 라는 *family 생성기* 만 알면 됨.

## 6. 변형 (Variants)

- **Provider 주입**: Factory 를 DI 컨테이너로 주입 (Spring `@Bean`, Dishka `Provider`).
- **Static factory**: `UIFactory.light()` 같은 정적 메서드 반환.
- **Functional**: Java 8+ 에서는 `Supplier<Button>` 두 개로 대체 가능.

## 7. 함정 / 흔한 오해

- **Factory Method 와 혼동**: Factory Method 는 *한 종류 객체*, Abstract Factory 는 *family*.
- **Family 가 정해지면 새 제품 추가가 어렵다** — `UIFactory` 에 `createSlider()` 추가 시 모든 ConcreteFactory 수정 필요.
- **남용**: 제품이 1 종류면 그냥 Factory Method.

## 8. 관련 패턴

- **Factory Method (03)**: AbstractFactory 의 각 메서드는 본질적으로 Factory Method.
- **Singleton (05)**: ConcreteFactory 가 Singleton 인 경우 흔함.
- **Prototype (04)**: family 가 동적이면 prototype 등록 + clone.

## 9. 실무 사례

- `javax.xml.parsers.DocumentBuilderFactory`
- `javax.xml.transform.TransformerFactory`
- Hibernate 의 `SessionFactory` (다른 DB 방언별 family)
- Swing 의 LookAndFeel API
