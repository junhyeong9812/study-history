# 07 — Bridge

> **분류**: 구조 패턴 (Structural)
> **한 줄**: 추상과 구현을 *분리* 하여 둘이 독립적으로 확장 가능하게 한다.

---

## 1. 의도 (Intent)

> "Decouple an abstraction from its implementation so that the two can vary independently."

추상 (예 : Shape) 과 구현 세부 (예 : 그리는 API) 가 둘 다 변할 수 있을 때, 그 둘을 *클래스 계층으로 곱하지 말고* 분리.

## 2. 문제 상황 (Motivation)

도형 × API 를 클래스 상속으로 표현하면 폭발 :

```
Shape
├── Circle
│   ├── CircleVulkan
│   └── CircleOpenGL
└── Square
    ├── SquareVulkan
    └── SquareOpenGL
```

도형 N 개 × API M 개 = N×M 클래스. 도형 추가 / API 추가 시 *모든 조합* 만들어야 함.

해결 : 두 차원을 *분리*.

```
Shape ─── has-a ───► Renderer
   │                     │
Circle, Square         OpenGLRenderer, VulkanRenderer
```

도형 N + API M = N+M 클래스. 조합은 런타임에.

## 3. 구조 (Structure)

```
Abstraction (Shape)              Implementor (Renderer)
   ┌────────────┐                ┌─────────────┐
   │ - impl     │ ─── uses ───► │ drawCircle()│
   │ + draw()   │                │ drawSquare()│
   └─────┬──────┘                └──────┬──────┘
         │                              │
   ┌─────┴─────┐                ┌───────┴──────┐
   │           │                │              │
RefinedAbstr1  RefinedAbstr2  ConcreteImpl1  ConcreteImpl2
(Circle)       (Square)      (OpenGLRenderer) (VulkanRenderer)
```

## 4. 참여자 (Participants)

- **Abstraction**: 추상 인터페이스. Implementor 의 참조 보유.
- **RefinedAbstraction**: Abstraction 의 구체화.
- **Implementor**: 구현 세부의 인터페이스.
- **ConcreteImplementor**: Implementor 의 구체.

## 5. Java 예제

```java
// Implementor
interface Renderer {
    void renderCircle(double x, double y, double radius);
    void renderSquare(double x, double y, double side);
}

// Concrete Implementors
class OpenGLRenderer implements Renderer {
    @Override public void renderCircle(double x, double y, double r) {
        System.out.printf("OpenGL circle at (%.0f,%.0f) r=%.0f%n", x, y, r);
    }
    @Override public void renderSquare(double x, double y, double side) {
        System.out.printf("OpenGL square at (%.0f,%.0f) side=%.0f%n", x, y, side);
    }
}

class VulkanRenderer implements Renderer {
    @Override public void renderCircle(double x, double y, double r) {
        System.out.printf("Vulkan circle at (%.0f,%.0f) r=%.0f%n", x, y, r);
    }
    @Override public void renderSquare(double x, double y, double side) {
        System.out.printf("Vulkan square at (%.0f,%.0f) side=%.0f%n", x, y, side);
    }
}

// Abstraction
abstract class Shape {
    protected final Renderer renderer;
    protected Shape(Renderer renderer) {
        this.renderer = renderer;
    }
    public abstract void draw();
}

// Refined Abstractions
class Circle extends Shape {
    private final double x, y, radius;
    public Circle(Renderer r, double x, double y, double radius) {
        super(r);
        this.x = x; this.y = y; this.radius = radius;
    }
    @Override public void draw() {
        renderer.renderCircle(x, y, radius);
    }
}

class Square extends Shape {
    private final double x, y, side;
    public Square(Renderer r, double x, double y, double side) {
        super(r);
        this.x = x; this.y = y; this.side = side;
    }
    @Override public void draw() {
        renderer.renderSquare(x, y, side);
    }
}

// 사용
public class Main {
    public static void main(String[] args) {
        Renderer gl = new OpenGLRenderer();
        Renderer vk = new VulkanRenderer();

        Shape c1 = new Circle(gl, 10, 20, 5);
        Shape c2 = new Circle(vk, 10, 20, 5);    // 같은 도형, 다른 API

        c1.draw();
        c2.draw();
    }
}
```

같은 `Circle` 이 어느 Renderer 를 받느냐에 따라 OpenGL 또는 Vulkan 으로 그려짐.

## 6. 변형 (Variants)

- **Implementor 가 Renderer + Audio + Physics 등 여러 차원**: Bridge 가 다중.
- **Refined Abstraction 추가**: `Triangle`, `Hexagon` 추가 시 Renderer 인터페이스 확장 (`renderTriangle`).

## 7. 함정 / 흔한 오해

### 7.1 Adapter (06) 와의 차이
- **Adapter**: 이미 만들어진 두 인터페이스를 *나중에* 잇기.
- **Bridge**: *처음부터* 분리해서 설계.

### 7.2 Strategy (21) 와의 차이
- **Strategy**: 알고리즘 *교체*. Strategy 는 보통 stateless.
- **Bridge**: 추상의 *구현 차원* 분리. 둘 다 자체 계층 가능.
- 코드는 비슷해 보일 수 있지만 *의도* 가 다름.

### 7.3 과설계
도형 1 종 × API 1 개면 Bridge 불필요. 변동 가능성이 *둘 다 있을 때만*.

## 8. 관련 패턴

- **Abstract Factory (01)**: Bridge 의 ConcreteImplementor 를 family 로 생성.
- **Adapter (06)**: 사후적 대안.
- **Strategy (21)**: 비슷한 코드 구조, 다른 의도.

## 9. 실무 사례

- JDBC : `Connection` (Abstraction) ←→ DB driver (Implementor)
- AWT : `Component` ←→ Toolkit (peer)
- SLF4J : Logger 인터페이스 ←→ Backend (Logback, Log4j)
- Java Collection : `AbstractList` ←→ array vs linked impl
- Spring `JdbcTemplate` : 추상 SQL ←→ DataSource 구현
