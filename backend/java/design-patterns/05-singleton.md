# 05 — Singleton

> **분류**: 생성 패턴 (Creational)
> **한 줄**: 한 클래스의 인스턴스가 *오직 하나* 임을 보장하고, 전역 접근점을 제공한다.

---

## 1. 의도 (Intent)

> "Ensure a class has only one instance, and provide a global point of access to it."

설정, 로거, 캐시, 커넥션 풀처럼 *하나만 있어야 하는* 자원.

## 2. 문제 상황 (Motivation)

```java
// ❌ 매번 새로 만들면
class DatabasePool {
    public DatabasePool() {
        // 비싼 초기화 — 100 connections 생성
    }
}

void someService() {
    DatabasePool pool = new DatabasePool();   // ← 매번 100 connections 생성
    ...
}
```

연결 풀이 의미 없어진다. 게다가 두 객체가 같은 DB 에 *각자* 풀을 만들어 자원 낭비.

→ "한 인스턴스만 존재" 를 *언어 차원에서 강제* 하고 싶다.

## 3. 구조 (Structure)

```
Singleton
┌──────────────────────────┐
│ - instance: Singleton    │  ← static 필드
│ - Singleton()            │  ← private 생성자
│ + getInstance(): Singleton│  ← static 진입점
└──────────────────────────┘
```

## 4. 참여자 (Participants)

- **Singleton**: 자기 인스턴스를 static 으로 보유 + private 생성자.

## 5. Java 예제 — 6 가지 구현

### 5.1 Eager initialization (가장 단순)

```java
public class EagerSingleton {
    private static final EagerSingleton INSTANCE = new EagerSingleton();

    private EagerSingleton() {}

    public static EagerSingleton getInstance() {
        return INSTANCE;
    }
}
```

- 클래스 로딩 시 즉시 생성.
- 스레드 안전 (JVM 이 보장).
- 단점 : 인스턴스를 *안 쓸 때도* 생성됨.

### 5.2 Lazy initialization (스레드 안전 X)

```java
public class LazySingleton {
    private static LazySingleton instance;

    private LazySingleton() {}

    public static LazySingleton getInstance() {
        if (instance == null) {
            instance = new LazySingleton();    // ❌ race condition
        }
        return instance;
    }
}
```

여러 스레드가 동시에 `getInstance()` → 둘 다 `instance == null` 통과 → 두 개 생성됨. **동시성 환경에서 깨짐**.

### 5.3 Synchronized (스레드 안전, 느림)

```java
public class SyncSingleton {
    private static SyncSingleton instance;

    private SyncSingleton() {}

    public static synchronized SyncSingleton getInstance() {    // ← 매 호출 lock
        if (instance == null) instance = new SyncSingleton();
        return instance;
    }
}
```

매 호출마다 동기화 → 성능 손실.

### 5.4 Double-Checked Locking (DCL)

```java
public class DCLSingleton {
    private static volatile DCLSingleton instance;    // ← volatile 필수

    private DCLSingleton() {}

    public static DCLSingleton getInstance() {
        DCLSingleton local = instance;
        if (local == null) {
            synchronized (DCLSingleton.class) {
                local = instance;
                if (local == null) {
                    instance = local = new DCLSingleton();
                }
            }
        }
        return local;
    }
}
```

생성 후엔 lock 안 함. `volatile` 이 *중요* — 다른 스레드가 *부분 초기화된* 인스턴스를 보지 않게.

### 5.5 Initialization-on-demand Holder (★ 추천)

```java
public class HolderSingleton {
    private HolderSingleton() {}

    private static class Holder {
        private static final HolderSingleton INSTANCE = new HolderSingleton();
    }

    public static HolderSingleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

JVM 의 클래스 로딩 보장 활용 :
- `Holder` 는 `getInstance()` 가 처음 호출될 때 로딩 → lazy.
- 클래스 로딩은 JVM 이 동기화 보장 → thread-safe.
- volatile / synchronized 불필요.

### 5.6 Enum (★ Joshua Bloch 추천 — *Effective Java* Item 3)

```java
public enum EnumSingleton {
    INSTANCE;

    public void doSomething() {
        System.out.println("Singleton method");
    }
}

// 사용
EnumSingleton.INSTANCE.doSomething();
```

- 가장 간결.
- 직렬화 / 리플렉션 공격에도 안전.
- enum 의 본질이 *한 번만 인스턴스화*.
- 단점 : `enum` 이라는 의미가 어색할 수 있음 + 상속 불가.

## 6. 변형 (Variants)

- **Multiton**: 키별로 인스턴스 (예 : 로그 카테고리별 1 개).
- **Registry**: 여러 singleton 을 한 곳에서 관리.
- **Per-thread Singleton**: `ThreadLocal` 사용.

## 7. 함정 / 흔한 오해

### 7.1 안티패턴 논란
- 전역 상태 → 테스트 어려움.
- 의존성을 *숨김* (`Singleton.getInstance()` 가 코드 곳곳에).
- 대안 : DI 컨테이너로 *singleton scope 빈* (Spring, Dishka).

### 7.2 직렬화의 함정 (Eager / Lazy)
```java
class MySingleton implements Serializable { ... }
// 역직렬화 시 새 인스턴스 생성 → singleton 깨짐
// readResolve() 메서드 추가 필요
```

### 7.3 리플렉션 공격
```java
Constructor<MySingleton> c = MySingleton.class.getDeclaredConstructor();
c.setAccessible(true);
MySingleton hack = c.newInstance();    // ← private 무시
```

→ Enum singleton 만 안전.

### 7.4 Class loader 함정
- 다른 ClassLoader 에서 로드 → 다른 instance.
- 웹 컨테이너 (Tomcat) 에서 hot reload 시 문제.

## 8. 관련 패턴

- **Abstract Factory (01)**: ConcreteFactory 가 흔히 Singleton.
- **Facade (10)**: 전역 facade 가 Singleton 인 경우.
- **State (20)**: state 객체가 stateless 면 Singleton 으로 공유.

## 9. 실무 사례

- `Runtime.getRuntime()`
- `System` 클래스 (사실상 정적 메서드 모음 = Singleton 변형)
- Spring `@Component` (기본 singleton scope)
- 로거 (`LoggerFactory.getLogger(...)` — 로거 자체는 sub-system)
- DB 커넥션 풀 (HikariCP 인스턴스)

### Spring 에서

```java
@Component        // 기본 singleton scope
public class UserService { ... }
```

→ Spring 의 `@Component` 빈은 *기본적으로 singleton*. GoF 의 패턴이 *언어/프레임워크 차원* 으로 흡수된 예.

> **권장** : 새 코드에서 *직접 Singleton 구현보다는 DI 컨테이너의 singleton scope 빈* 사용. 테스트 / 유연성에서 우월.
