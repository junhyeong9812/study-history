# 04 — Prototype

> **분류**: 생성 패턴 (Creational)
> **한 줄**: 기존 인스턴스를 *복제(clone)* 하여 새 인스턴스를 만든다.

---

## 1. 의도 (Intent)

> "Specify the kinds of objects to create using a prototypical instance, and create new objects by copying this prototype."

객체 생성 비용이 크거나, 런타임에 *어떤 클래스인지 모른 채* 같은 종류 객체를 만들어야 할 때.

## 2. 문제 상황 (Motivation)

```java
// ❌ 직접 생성 — 다음 두 경우 어려움
// 1. 생성 비용이 큼 (DB 조회, 외부 API)
// 2. 런타임에 타입을 모름

class GameMonster {
    private final String type;
    private final int hp;
    private final List<Skill> skills;       // 무거운 초기화

    public GameMonster(String type) {
        this.type = type;
        this.skills = loadSkillsFromDB(type);   // ← DB 호출
        this.hp = computeBaseHp(type);
    }
}

// 같은 타입의 몬스터 100 마리 → DB 100 번 호출
```

해결 : 첫 1 마리만 DB 로 만들고, 나머지는 *복제*.

```java
GameMonster prototype = new GameMonster("dragon");   // DB 1 회
List<GameMonster> dragons = new ArrayList<>();
for (int i = 0; i < 100; i++) {
    dragons.add(prototype.clone());      // DB 호출 X
}
```

## 3. 구조 (Structure)

```
Prototype (interface)
   ┌──────────┐
   │ clone()  │
   └────┬─────┘
        │
   ┌────┴─────┐
   │          │
ConcretePrototype1  ConcretePrototype2
clone() → 자기 복사    clone() → 자기 복사
```

## 4. 참여자 (Participants)

- **Prototype**: `clone()` 메서드 인터페이스.
- **ConcretePrototype**: 실제 복제 로직.
- **Client**: prototype 보유 + `clone()` 호출.

## 5. Java 예제

### 5.1 단순 복제

```java
public abstract class Shape implements Cloneable {
    public String color;
    public int x, y;

    @Override
    public Shape clone() {
        try {
            return (Shape) super.clone();    // ← Object.clone() — 얕은 복사
        } catch (CloneNotSupportedException e) {
            throw new AssertionError(e);
        }
    }

    public abstract void draw();
}

public class Circle extends Shape {
    public int radius;

    @Override public void draw() {
        System.out.printf("Circle(r=%d, color=%s) at (%d,%d)%n", radius, color, x, y);
    }
}

// 사용
Circle original = new Circle();
original.color = "red";
original.radius = 10;

Circle copy = (Circle) original.clone();
copy.color = "blue";    // copy 만 변경
original.draw();        // red
copy.draw();            // blue
```

### 5.2 Deep copy (가변 컬렉션 포함 시)

```java
public class Squad implements Cloneable {
    private String name;
    private List<Soldier> soldiers;     // ← 가변

    @Override
    public Squad clone() {
        try {
            Squad copy = (Squad) super.clone();
            // ★ 필수 — 안 하면 두 Squad 가 같은 list 공유
            copy.soldiers = new ArrayList<>(this.soldiers.size());
            for (Soldier s : this.soldiers) {
                copy.soldiers.add(s.clone());      // 각 Soldier 도 복제
            }
            return copy;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError(e);
        }
    }
}
```

`Object.clone()` 은 기본적으로 *얕은 복사*. 필드가 객체 참조면 두 인스턴스가 *같은 객체를 공유*. 필요시 deep copy 직접.

### 5.3 Prototype Registry

```java
class ShapeRegistry {
    private final Map<String, Shape> prototypes = new HashMap<>();

    void register(String key, Shape prototype) {
        prototypes.put(key, prototype);
    }

    Shape get(String key) {
        return prototypes.get(key).clone();
    }
}

// 사용
registry.register("red-circle", configuredRedCircle);
Shape s1 = registry.get("red-circle");
Shape s2 = registry.get("red-circle");
// s1 != s2, 그러나 둘 다 red, 같은 radius
```

→ Factory Method 의 대안. 새 종류는 *clone 가능한 prototype 등록* 만으로 추가.

## 6. 변형 (Variants)

- **`Cloneable` 안 쓰는 방법**: 복사 생성자 (`new Shape(other)`) 또는 정적 메서드 (`Shape.copyOf(other)`).
- **직렬화 기반 deep copy**: `Serializable` + ObjectOutputStream → 모든 필드 자동 복제. 느림.
- **`record`**: Java 14+ 의 record 는 자동 `equals/hashCode` 가 있지만 clone 은 없음. `with` 메서드 (Java 21+ scheduled) 가 유사.

## 7. 함정 / 흔한 오해

- **`Cloneable` 의 디자인 결함**: Joshua Bloch 가 *Effective Java* 에서 사용 *비추* — 차라리 복사 생성자 / 정적 팩토리 추천.
- **얕은 복사의 함정**: 가변 필드를 깜빡 → 두 인스턴스가 같은 list 공유 → 한쪽 변경이 다른 쪽에 영향.
- **`clone()` 이 protected**: `Object.clone()` 은 protected → 외부 호출 불가. public 으로 오버라이드 필요.

## 8. 관련 패턴

- **Singleton (05)**: prototype registry 가 singleton 인 경우 흔함.
- **Composite (08)**: Composite 트리 복제에 prototype.
- **Memento (18)**: 상태 저장도 일종의 prototype.

## 9. 실무 사례

- `Object.clone()` (디자인 결함이 있어 권장 X)
- `Cloneable` 인터페이스
- `ArrayList.clone()` — 얕은 복사
- Spring `prototype` scope 빈 — 매 요청마다 새 인스턴스 (이름은 같지만 의미가 다름)
- 게임 엔진의 사전 설정된 적 / 무기 / 스킬 템플릿
- Photoshop / Figma 의 객체 복제

> Java 에서는 *복사 생성자* (`new Shape(original)`) 가 더 권장됨 :
> ```java
> public Circle(Circle other) {
>     this.color = other.color;
>     this.x = other.x; this.y = other.y;
>     this.radius = other.radius;
> }
> ```
