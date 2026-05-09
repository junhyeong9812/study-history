# 23 — Visitor

> **분류**: 행위 패턴 (Behavioral)
> **한 줄**: 객체 구조를 변경하지 않고 *새 연산* 을 추가.

---

## 1. 의도 (Intent)

> "Represent an operation to be performed on the elements of an object structure. Visitor lets you define a new operation without changing the classes of the elements on which it operates."

타입 계층에 *연산이 자주 추가* 되는데 *클래스가 잘 안 바뀔* 때.

## 2. 문제 상황 (Motivation)

도형 트리 — Circle, Square, Triangle. 새 연산 (그리기 / 면적 / 직렬화 / 검증 ...) 추가가 잦다.

```java
// ❌ 클래스마다 메서드 추가
class Circle {
    void draw() { ... }
    double area() { ... }
    String toJson() { ... }
    void validate() { ... }
    // 새 연산마다 모든 도형 클래스 수정
}
class Square {
    void draw() { ... }
    double area() { ... }
    ...
}
```

OCP 위반 — 새 연산 = 모든 클래스 수정.

해결 : 연산을 *Visitor* 로 분리. 도형은 `accept(visitor)` 만.

```java
// 새 연산 = 새 Visitor 클래스. 도형 클래스 변경 X.
shapes.forEach(s -> s.accept(new AreaCalculator()));
shapes.forEach(s -> s.accept(new JsonExporter()));
shapes.forEach(s -> s.accept(new Validator()));
```

## 3. 구조 (Structure)

```
Visitor (interface)              Element (interface)
  ┌────────────────┐              ┌──────────────┐
  │ visitCircle(c) │              │ accept(v)    │
  │ visitSquare(s) │              └──────┬───────┘
  └────────┬───────┘                     │
           │                       ┌─────┴─────┐
   ConcreteVisitor1            Circle       Square
   (AreaCalc)                accept(v)→     accept(v)→
                              v.visitCircle    v.visitSquare
                              (this)            (this)
```

**더블 디스패치** 가 핵심 :
1. `shape.accept(visitor)` — shape 의 종류에 따라 dispatch (Circle.accept vs Square.accept).
2. `visitor.visitCircle(this)` — visitor 의 종류에 따라 dispatch.

→ 두 단계 dispatch 로 (Shape × Visitor) 의 모든 조합이 자동 결정.

## 4. 참여자 (Participants)

- **Visitor**: 각 ConcreteElement 별 visit 메서드.
- **ConcreteVisitor**: 특정 연산.
- **Element**: `accept(Visitor)` 메서드.
- **ConcreteElement**: `accept` 에서 `visitor.visitX(this)` 호출.
- **ObjectStructure**: Element 들의 컬렉션.

## 5. Java 예제

```java
// Element interface
interface Shape {
    <R> R accept(ShapeVisitor<R> visitor);
}

// Concrete Elements
class Circle implements Shape {
    final double radius;
    public Circle(double r) { this.radius = r; }

    @Override
    public <R> R accept(ShapeVisitor<R> v) {
        return v.visitCircle(this);
    }
}

class Square implements Shape {
    final double side;
    public Square(double s) { this.side = s; }

    @Override
    public <R> R accept(ShapeVisitor<R> v) {
        return v.visitSquare(this);
    }
}

class Triangle implements Shape {
    final double base, height;
    public Triangle(double b, double h) { this.base = b; this.height = h; }

    @Override
    public <R> R accept(ShapeVisitor<R> v) {
        return v.visitTriangle(this);
    }
}

// Visitor interface — generic 으로 반환 타입 지원
interface ShapeVisitor<R> {
    R visitCircle(Circle c);
    R visitSquare(Square s);
    R visitTriangle(Triangle t);
}

// Concrete Visitors

// 1. 면적 계산
class AreaCalculator implements ShapeVisitor<Double> {
    @Override public Double visitCircle(Circle c) { return Math.PI * c.radius * c.radius; }
    @Override public Double visitSquare(Square s) { return s.side * s.side; }
    @Override public Double visitTriangle(Triangle t) { return 0.5 * t.base * t.height; }
}

// 2. JSON 직렬화
class JsonExporter implements ShapeVisitor<String> {
    @Override public String visitCircle(Circle c) {
        return "{\"type\":\"circle\",\"radius\":" + c.radius + "}";
    }
    @Override public String visitSquare(Square s) {
        return "{\"type\":\"square\",\"side\":" + s.side + "}";
    }
    @Override public String visitTriangle(Triangle t) {
        return "{\"type\":\"triangle\",\"base\":" + t.base + ",\"height\":" + t.height + "}";
    }
}

// 사용
public class Main {
    public static void main(String[] args) {
        List<Shape> shapes = List.of(
            new Circle(5),
            new Square(3),
            new Triangle(4, 6)
        );

        AreaCalculator area = new AreaCalculator();
        for (Shape s : shapes) {
            System.out.printf("%.2f%n", s.accept(area));
        }
        // 78.54
        // 9.00
        // 12.00

        JsonExporter json = new JsonExporter();
        for (Shape s : shapes) {
            System.out.println(s.accept(json));
        }
    }
}
```

새 연산 (Validator) 추가 시 — `Validator implements ShapeVisitor<Boolean>` 만 추가. Circle, Square, Triangle 클래스는 *변경 0*.

## 6. 변형 (Variants)

### 6.1 Sealed Interface (Java 17+) + Pattern Matching

```java
sealed interface Shape permits Circle, Square, Triangle {}

double area(Shape shape) {
    return switch (shape) {
        case Circle c -> Math.PI * c.radius() * c.radius();
        case Square s -> s.side() * s.side();
        case Triangle t -> 0.5 * t.base() * t.height();
    };
}
```

→ Visitor 의 *현대 대안*. sealed 가 모든 케이스를 컴파일러가 검증. 새 도형 추가 시 모든 switch 가 컴파일 에러로 잡힘.

### 6.2 Element Hierarchy 깊을 때
계층이 깊으면 (Composite + Visitor) 트리 순회. Visitor 가 자신을 자식들에게 재귀 호출.

### 6.3 Reflective Visitor
`instanceof` 체인 한 메서드로. 정적 dispatch 의 이점 잃지만 단순.

## 7. 함정 / 흔한 오해

### 7.1 새 Element 추가 어려움
새 Shape 종류 (Hexagon) 추가 시 *모든 Visitor* 에 `visitHexagon` 추가. 클래스가 잘 안 바뀐다는 *전제* 가 깨지면 visitor 가 부담.

→ 트레이드오프 : *연산 다양 + 클래스 안정* → Visitor 유리. *클래스 다양 + 연산 안정* → 다형성 (메서드).

### 7.2 캡슐화 위반 가능
Visitor 가 Element 의 내부 데이터 접근 → public getter 노출 강제. record / 친밀한 Visitor 로 완화.

### 7.3 Double dispatch 가 어려움
처음 보면 헷갈림. 핵심은 "두 단계 dispatch 로 (E × V) 매트릭스 결정".

### 7.4 단순 케이스에 과설계
도형 3 개 × 연산 2 개면 그냥 메서드. Visitor 는 *연산 추가가 잦을 때*.

## 8. 관련 패턴

- **Composite (08)**: Visitor 로 Composite 트리 순회.
- **Iterator (16)**: 순회 + 노드별 처리.
- **Interpreter (15)**: AST 의 다양한 연산.
- **Strategy (21)**: 단일 알고리즘. Visitor 는 *타입별 분기 알고리즘*.

## 9. 실무 사례

- 컴파일러 / 인터프리터의 AST traversal (type checker, optimizer, code gen)
- ANTLR `ParseTreeVisitor`
- Jackson `JsonGenerator` (각 타입별 직렬화)
- ASM (Java bytecode 라이브러리) 의 `ClassVisitor`, `MethodVisitor`
- Eclipse JDT `ASTVisitor`
- Database query planner — 같은 AST 에 cost estimation, optimization, execution
- 정적 분석 도구 (SpotBugs, Checkstyle)

```java
// ANTLR Visitor 예
public class CalcVisitor extends CalcBaseVisitor<Integer> {
    @Override public Integer visitPlus(CalcParser.PlusContext ctx) {
        return visit(ctx.left) + visit(ctx.right);
    }
    @Override public Integer visitTimes(CalcParser.TimesContext ctx) {
        return visit(ctx.left) * visit(ctx.right);
    }
}
```

> Visitor 는 *함수형 언어의 패턴 매칭* 을 OOP 가 흉내내는 패턴. Java 21+ 의 sealed + switch 가 더 깔끔한 대안이 되어가는 중.
