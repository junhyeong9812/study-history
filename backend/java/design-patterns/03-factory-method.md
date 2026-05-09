# 03 — Factory Method

> **분류**: 생성 패턴 (Creational)
> **한 줄**: 인스턴스 생성 *책임* 을 서브클래스에 위임. 부모는 인터페이스만 정의.

---

## 1. 의도 (Intent)

> "Define an interface for creating an object, but let subclasses decide which class to instantiate. Factory Method lets a class defer instantiation to subclasses."

부모 클래스는 *어떤 객체를 사용할 것인지* 알지만 *어떤 구체 클래스인지* 는 모른다. 서브클래스가 결정.

## 2. 문제 상황 (Motivation)

`Logistics` 라는 시스템이 있고 `planDelivery()` 가 어떤 운송 수단으로든 동작해야 한다.

```java
// ❌ 구체 클래스 직접
class RoadLogistics {
    void planDelivery() {
        Truck truck = new Truck();      // ← Truck 강결합
        truck.deliver();
    }
}
class SeaLogistics {
    void planDelivery() {
        Ship ship = new Ship();         // ← Ship 강결합
        ship.deliver();
    }
}
```

`planDelivery()` 자체는 두 클래스에서 *거의 같은 로직*. 차이는 *어떤 운송수단을 쓰느냐*. 이걸 서브클래스에 위임하면 :

```java
abstract class Logistics {
    void planDelivery() {
        Transport t = createTransport();    // ← 어떤 인스턴스인지 모름
        t.deliver();
    }

    protected abstract Transport createTransport();
}
```

## 3. 구조 (Structure)

```
Creator (abstract)              Product (interface)
  ┌──────────────────┐         ┌──────────┐
  │ factoryMethod()  │ ─────►  │          │
  │ operation() {    │         └────┬─────┘
  │   p = factoryMethod()             │
  │   p.use()        │         ┌─────┴─────┐
  │ }                │         │           │
  └────┬─────────────┘     ConcreteProduct
       │
   ConcreteCreator
   factoryMethod() → ConcreteProduct
```

## 4. 참여자 (Participants)

- **Product**: 생성될 객체의 인터페이스.
- **ConcreteProduct**: 실제 구현.
- **Creator**: 추상 부모. `factoryMethod()` 를 *protected abstract* 로 선언.
- **ConcreteCreator**: factoryMethod 를 오버라이드하여 ConcreteProduct 반환.

## 5. Java 예제

```java
// Product
interface Transport {
    void deliver();
}

// Concrete Products
class Truck implements Transport {
    @Override public void deliver() { System.out.println("Deliver by land"); }
}

class Ship implements Transport {
    @Override public void deliver() { System.out.println("Deliver by sea"); }
}

// Creator
abstract class Logistics {
    public void planDelivery() {
        Transport t = createTransport();    // ← 서브클래스가 결정
        t.deliver();
    }

    protected abstract Transport createTransport();
}

// Concrete Creators
class RoadLogistics extends Logistics {
    @Override protected Transport createTransport() { return new Truck(); }
}

class SeaLogistics extends Logistics {
    @Override protected Transport createTransport() { return new Ship(); }
}

// 사용
public class Main {
    public static void main(String[] args) {
        Logistics logistics = new RoadLogistics();
        logistics.planDelivery();    // "Deliver by land"
    }
}
```

`planDelivery()` 의 알고리즘은 부모에 한 번만. 변하는 부분 (`createTransport`) 만 자식이 채움 — Template Method (22) 와 결합되는 패턴.

## 6. 변형 (Variants)

### 6.1 Parameterized Factory Method

```java
abstract class Logistics {
    public Transport createTransport(String type) {
        return switch (type) {
            case "road" -> new Truck();
            case "sea" -> new Ship();
            default -> throw new IllegalArgumentException(type);
        };
    }
}
```

→ 서브클래스 대신 *매개변수* 로 분기. 단, 새 타입 추가 시 switch 수정 필요 (OCP 위반).

### 6.2 Static Factory Method (Effective Java Item 1)

```java
public class Logger {
    private Logger() {}
    public static Logger getLogger(String name) {
        // 캐시 / 싱글톤 / 다양한 결정
        return new Logger();
    }
}
```

이건 GoF 의 Factory Method 와 *다른 패턴* (이름만 비슷). 주로 Singleton (05) / Flyweight (11) 와 결합.

## 7. 함정 / 흔한 오해

- **Static Factory ≠ Factory Method**: GoF 의 Factory Method 는 *다형성* 활용 (서브클래스가 결정). Static factory 는 *생성자의 대안*.
- **항상 추상 메서드일 필요 없음**: 부모가 default 구현 두고 서브클래스가 *선택적* 으로 오버라이드 가능.
- **남용**: 새 Product 마다 새 Creator → 클래스 폭발.

## 8. 관련 패턴

- **Template Method (22)**: Factory Method 가 Template Method 의 한 단계인 경우가 흔함.
- **Abstract Factory (01)**: AbstractFactory 의 각 메서드가 Factory Method.
- **Prototype (04)**: 인스턴스 생성을 prototype.clone() 으로 대체.

## 9. 실무 사례

- `java.util.Calendar.getInstance()` — region 에 따른 GregorianCalendar / BuddhistCalendar
- `java.text.NumberFormat.getInstance()`
- Spring `BeanFactory.getBean()`
- `ThreadFactory.newThread(Runnable)`
- `Iterator iterator()` 메서드 (Collection 인터페이스의)
