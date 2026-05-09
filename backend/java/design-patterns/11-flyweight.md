# 11 — Flyweight

> **분류**: 구조 패턴 (Structural)
> **한 줄**: 다수의 미세 객체를 *공유* 하여 메모리 절약.

---

## 1. 의도 (Intent)

> "Use sharing to support large numbers of fine-grained objects efficiently."

같은 *intrinsic state* 를 가진 객체가 많을 때, 인스턴스를 *공유* 한다.

## 2. 문제 상황 (Motivation)

게임에서 나무 1만 그루를 그려야 한다. 각 나무는 :
- 종류 (oak, pine, birch) — 같은 종류는 같은 메시 / 텍스처
- 위치 (x, y) — 그루마다 다름

```java
// ❌ 1만 개 인스턴스, 각각 메시 / 텍스처 보유
class Tree {
    Mesh mesh;            // 큰 데이터 (수 MB)
    Texture texture;      // 큰 데이터
    int x, y;             // 작은 데이터
}
// 1만 × 메시(2MB) = 20GB → OOM
```

해결 : 메시 / 텍스처를 *종류별로 하나만*, x/y 만 인스턴스마다.

```java
class TreeType {       // intrinsic — 공유
    Mesh mesh;
    Texture texture;
}

class Tree {           // extrinsic — 인스턴스마다
    TreeType type;     // ← 공유 참조
    int x, y;
}
```

1만 × (참조 + int*2) = 작음. TreeType 은 종류 수만큼 (예 : 3 개).

## 3. 구조 (Structure)

```
Flyweight (interface)
   ┌──────────────────┐
   │ operation(extr)  │
   └────────┬─────────┘
            │
   ConcreteFlyweight
     - intrinsicState

FlyweightFactory
   - cache: Map<Key, Flyweight>
   + getFlyweight(key) {
       if (!cache.has(key)) cache[key] = new ...
       return cache[key]
     }
```

## 4. 참여자 (Participants)

- **Flyweight**: 공유될 인터페이스. extrinsic state 를 매개변수로.
- **ConcreteFlyweight**: intrinsic state 보유.
- **FlyweightFactory**: 캐시. 같은 key 면 같은 인스턴스 반환.
- **Client**: extrinsic state 를 따로 보관.

## 5. Java 예제

```java
import java.util.HashMap;
import java.util.Map;

// Flyweight (intrinsic state — 공유)
class TreeType {
    private final String name;
    private final String mesh;
    private final String texture;

    public TreeType(String name, String mesh, String texture) {
        this.name = name;
        this.mesh = mesh;
        this.texture = texture;
        System.out.println("Creating TreeType: " + name);
    }

    public void draw(int x, int y) {
        System.out.printf("Drawing %s at (%d,%d) with mesh=%s tex=%s%n",
            name, x, y, mesh, texture);
    }
}

// FlyweightFactory
class TreeFactory {
    private static final Map<String, TreeType> cache = new HashMap<>();

    public static TreeType getType(String name, String mesh, String texture) {
        String key = name + "_" + mesh + "_" + texture;
        return cache.computeIfAbsent(key, k -> new TreeType(name, mesh, texture));
    }
}

// Client (extrinsic state — 인스턴스마다)
class Tree {
    private final int x, y;
    private final TreeType type;     // ← 공유

    public Tree(int x, int y, TreeType type) {
        this.x = x; this.y = y;
        this.type = type;
    }

    public void draw() {
        type.draw(x, y);
    }
}

// Forest (대량 인스턴스 보관)
class Forest {
    private final java.util.List<Tree> trees = new java.util.ArrayList<>();

    public void plantTree(int x, int y, String name, String mesh, String texture) {
        TreeType type = TreeFactory.getType(name, mesh, texture);    // 공유
        trees.add(new Tree(x, y, type));
    }

    public void draw() {
        for (Tree t : trees) t.draw();
    }
}

// 사용
public class Main {
    public static void main(String[] args) {
        Forest forest = new Forest();
        for (int i = 0; i < 1000; i++) {
            forest.plantTree(i, i * 2, "Oak", "oak.mesh", "oak.png");
            forest.plantTree(i, i * 3, "Pine", "pine.mesh", "pine.png");
        }
        // TreeType 은 2 개만 생성됨 (Oak, Pine), Tree 는 2000 개.
        // "Creating TreeType" 로그가 2 번만 찍힘.
    }
}
```

## 6. 변형 (Variants)

### 6.1 Immutable flyweight (★ 안전)

```java
record TreeType(String name, String mesh, String texture) {
    public void draw(int x, int y) { ... }
}
```

`record` 로 *불변* 보장. 공유 안전.

### 6.2 String pool (자동 flyweight)
Java `String` 은 literal 이 자동으로 풀에 캐시.
```java
String a = "hello";
String b = "hello";
a == b;   // true — 같은 인스턴스
```

### 6.3 Integer cache
```java
Integer a = 100;
Integer b = 100;
a == b;   // true (-128 ~ 127 캐시)

Integer c = 200;
Integer d = 200;
c == d;   // false (캐시 범위 벗어남)
```

## 7. 함정 / 흔한 오해

### 7.1 Mutable flyweight
공유 객체가 mutable → 한 곳에서 변경하면 다른 모든 곳 영향. *공유 객체는 immutable*.

### 7.2 Intrinsic / Extrinsic 구분 실수
어떤 상태가 공유 가능한지 명확히. 잘못 분리하면 동작이 깨짐.

### 7.3 메모리 vs 코드 복잡도 트레이드오프
대량 객체가 *없으면* flyweight 는 과설계. JVM 도 이미 String pool 등 가짐.

### 7.4 캐시 누수
flyweight factory 의 캐시가 무한 증가 → OOM. weak reference 또는 size limit.

## 8. 관련 패턴

- **Singleton (05)**: flyweight factory 가 singleton 인 경우.
- **Factory Method (03)**: factory 의 일종.
- **Composite (08)**: 트리의 leaf 가 flyweight.

## 9. 실무 사례

- `Integer.valueOf(int)` — -128~127 캐시
- `Boolean.TRUE`, `Boolean.FALSE`
- `String` 인터닝 (`String.intern()`)
- 게임 엔진의 텍스처 / 메시 풀
- HTTP 헤더 이름 (자주 쓰이는 Content-Type 등 캐시)
- 폰트 / 글리프 객체 (워드프로세서)
- 데이터베이스 connection pool — 모양이 다르지만 본질은 공유

```java
// Integer cache 예
Integer.valueOf(100) == Integer.valueOf(100);   // true
new Integer(100) == new Integer(100);           // false (캐시 우회)
```

> 모던 Java 에서는 `record` + 정적 팩토리 메서드가 flyweight 의 가장 깔끔한 구현.
