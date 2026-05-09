# 12 — Proxy

> **분류**: 구조 패턴 (Structural)
> **한 줄**: 다른 객체에 대한 *접근을 대리* 하는 객체. 지연 로딩, 권한, 캐시 등.

---

## 1. 의도 (Intent)

> "Provide a surrogate or placeholder for another object to control access to it."

대상 객체에 *직접 접근 대신* proxy 를 거치게 하여 부가 행위 (지연 / 권한 / 로깅 / 캐시 등) 추가.

## 2. 문제 상황 (Motivation)

큰 이미지를 처음부터 로드하면 느림. 실제 *그릴 때* 로드하고 싶다.

```java
// ❌ 매 인스턴스 즉시 로드
class Image {
    public Image(String filename) {
        loadFromDisk(filename);    // 즉시 — 5 초
    }
    public void display() { ... }
}

List<Image> gallery = new ArrayList<>();
for (int i = 0; i < 100; i++) {
    gallery.add(new Image("img" + i + ".jpg"));    // 100 × 5 초 = 500 초
}
gallery.get(0).display();    // 첫 번째만 보고 끝낼 때도 다 로드됨
```

해결 : proxy 가 *placeholder*, 실제 로드는 처음 `display()` 호출 시.

## 3. 구조 (Structure)

```
Subject (interface)              RealSubject
   ┌──────────────┐                ┌──────────────┐
   │ operation()  │                │ operation()  │ ← 무거운 일
   └──────┬───────┘                └──────┬───────┘
          │                                ▲
          ├──────────────┬─────────────────┘
          │              │
       Client          Proxy
                       - real: RealSubject (lazy)
                       + operation() {
                           if (real == null) real = new RealSubject()
                           return real.operation()
                         }
```

## 4. 참여자 (Participants)

- **Subject**: RealSubject 와 Proxy 의 공통 인터페이스.
- **RealSubject**: 진짜 객체.
- **Proxy**: Subject 구현 + RealSubject 참조 (lazy 또는 미리 보유).

## 5. Java 예제

### 5.1 Virtual Proxy (지연 로딩)

```java
interface Image {
    void display();
}

class RealImage implements Image {
    private final String filename;

    public RealImage(String filename) {
        this.filename = filename;
        loadFromDisk();
    }

    private void loadFromDisk() {
        System.out.println("Loading " + filename + " (5 sec)");
    }

    @Override public void display() {
        System.out.println("Displaying " + filename);
    }
}

class ImageProxy implements Image {
    private final String filename;
    private RealImage real;        // ← lazy

    public ImageProxy(String filename) {
        this.filename = filename;
    }

    @Override public void display() {
        if (real == null) {
            real = new RealImage(filename);     // 처음 사용 시 로드
        }
        real.display();
    }
}

// 사용
public class Main {
    public static void main(String[] args) {
        Image img = new ImageProxy("photo.jpg");    // 로드 X — 즉시
        // ... 시간 흐름 ...
        img.display();    // 여기서 로드 + 표시
        img.display();    // 두 번째는 캐시된 real 재사용
    }
}
```

### 5.2 Protection Proxy (권한)

```java
class SecureImageProxy implements Image {
    private final RealImage real;
    private final User user;

    public SecureImageProxy(String filename, User user) {
        this.real = new RealImage(filename);
        this.user = user;
    }

    @Override public void display() {
        if (!user.hasPermission("view")) {
            throw new SecurityException("No permission");
        }
        real.display();
    }
}
```

### 5.3 Remote Proxy (RPC)

```java
class RemoteServiceProxy implements Service {
    private final HttpClient http;

    @Override public Result execute(Request req) {
        // 직렬화 → HTTP → 서버에서 실행 → 역직렬화
        String body = serialize(req);
        String resp = http.post("https://api.example.com/exec", body);
        return deserialize(resp);
    }
}
```

→ 클라이언트는 *로컬 객체* 처럼 호출. proxy 가 네트워크 디테일 숨김.

### 5.4 Caching Proxy

```java
class CachingDatabaseProxy implements Database {
    private final Database real;
    private final Map<String, Object> cache = new HashMap<>();

    @Override public Object query(String sql) {
        return cache.computeIfAbsent(sql, k -> real.query(k));
    }

    @Override public void invalidate(String key) {
        cache.remove(key);
        real.invalidate(key);
    }
}
```

### 5.5 Smart Reference

```java
class SmartPtr<T> {
    private final T target;
    private int refCount;

    public T get() { refCount++; return target; }
    public void release() {
        if (--refCount == 0) cleanup(target);
    }
}
```

C++ 의 `shared_ptr` 같은 reference counting.

## 6. 변형 (Variants)

### 6.1 Dynamic Proxy (Java reflection)

```java
import java.lang.reflect.Proxy;
import java.lang.reflect.InvocationHandler;

interface Service {
    String greet(String name);
}

class RealService implements Service {
    @Override public String greet(String name) { return "Hello, " + name; }
}

// 런타임에 proxy 생성
Service real = new RealService();
Service proxy = (Service) Proxy.newProxyInstance(
    Service.class.getClassLoader(),
    new Class<?>[] { Service.class },
    (instance, method, args) -> {
        System.out.println("Before " + method.getName());
        Object result = method.invoke(real, args);
        System.out.println("After " + method.getName());
        return result;
    });

proxy.greet("World");
// Before greet
// After greet
```

→ Spring AOP 의 핵심 메커니즘. 트랜잭션 / 로깅 / 보안을 자동 wrap.

### 6.2 CGLIB / ByteBuddy
인터페이스 없는 클래스에도 proxy 생성. 바이트코드 생성.

## 7. 함정 / 흔한 오해

### 7.1 Decorator (09) / Adapter (06) 와의 차이
- **Decorator**: *기능 추가*. 같은 인터페이스.
- **Adapter**: *인터페이스 변환*. 다른 인터페이스.
- **Proxy**: *접근 제어*. 같은 인터페이스. 본질은 *대신*.

### 7.2 Proxy 가 너무 똑똑
proxy 안에 너무 많은 비즈니스 로직 → 실은 새 컴포넌트. proxy 는 *얇은 레이어* 가 본질.

### 7.3 Identity 문제
`real.equals(proxy)` 는 false. 캐시 키로 쓸 때 주의.

### 7.4 무한 재귀
proxy 의 메서드가 또 proxy 를 호출 → stack overflow. AOP 자기호출 함정.

## 8. 관련 패턴

- **Decorator (09)**: 친척. 의도 다름.
- **Adapter (06)**: 친척. 의도 다름.
- **Facade (10)**: 단순한 인터페이스 *구성*. Proxy 는 1:1 대리.

## 9. 실무 사례

- Spring `@Transactional` — proxy 가 트랜잭션 begin/commit
- Spring `@Async` — proxy 가 ExecutorService 로 비동기 실행
- Spring AOP advisor
- Hibernate lazy loading — `User.getOrders()` 가 처음 호출 시 SQL
- Java RMI — Remote Proxy
- gRPC stub — Remote Proxy
- mock 라이브러리 (Mockito) — 메서드 호출 가로채기

### Spring 의 트랜잭션 proxy

```java
@Service
@Transactional
public class OrderService {
    public void placeOrder(...) {
        // Spring 이 proxy 로 wrapping
        // 실행 전: tx.begin()
        // 실행 후: tx.commit() 또는 rollback()
    }
}
```

→ 코드는 *순수* 비즈니스 로직. 트랜잭션 디테일은 proxy 에. AOP = Proxy 패턴의 산업화.
