# 13 — Chain of Responsibility

> **분류**: 행위 패턴 (Behavioral)
> **한 줄**: 요청을 처리할 객체가 결정될 때까지 *핸들러 체인을 따라 전달*.

---

## 1. 의도 (Intent)

> "Avoid coupling the sender of a request to its receiver by giving more than one object a chance to handle the request."

요청을 보내는 쪽이 *누가 처리할지* 모르게 하고, 핸들러들 중 적합한 곳이 처리.

## 2. 문제 상황 (Motivation)

HTTP 요청 처리 — 인증, 로깅, 권한, 캐시, 본 처리 ...

```java
// ❌ 한 메서드에 다 들어있으면
class RequestHandler {
    public Response handle(Request req) {
        // 인증
        if (!authenticate(req)) return error(401);
        // 로깅
        log(req);
        // 권한
        if (!authorize(req)) return error(403);
        // 캐시
        Response cached = cache.get(req);
        if (cached != null) return cached;
        // 본 처리
        return process(req);
    }
}
```

새 단계 추가 / 일부만 사용하고 싶을 때 변경 부담. 단계마다 *조기 반환* 도 흩어짐.

해결 : 각 단계를 *핸들러* 로 분리. 체인으로 연결. 처리하거나 다음으로 넘김.

## 3. 구조 (Structure)

```
Client          Handler (interface)
  │             ┌──────────────┐
  └──► Handler1.handle(req)
            │
            ▼ next
       Handler2.handle(req)
            │
            ▼ next
       Handler3.handle(req)
            │
            ▼
         (처리 또는 종료)
```

## 4. 참여자 (Participants)

- **Handler**: 다음 핸들러 참조 + handle 메서드.
- **ConcreteHandler**: 자기 책임 처리. 못하면 next 로 위임.
- **Client**: 첫 핸들러에 요청.

## 5. Java 예제

### 5.1 정통 — 추상 클래스

```java
abstract class RequestHandler {
    protected RequestHandler next;

    public RequestHandler setNext(RequestHandler next) {
        this.next = next;
        return next;        // 체이닝 가능
    }

    public abstract Response handle(Request req);

    protected Response checkNext(Request req) {
        if (next == null) return Response.ok();
        return next.handle(req);
    }
}

class AuthHandler extends RequestHandler {
    @Override public Response handle(Request req) {
        if (req.token == null) return Response.error(401, "Auth required");
        return checkNext(req);
    }
}

class LoggingHandler extends RequestHandler {
    @Override public Response handle(Request req) {
        System.out.println("Request: " + req.path);
        return checkNext(req);
    }
}

class AuthorizationHandler extends RequestHandler {
    @Override public Response handle(Request req) {
        if (!req.user.canAccess(req.path)) return Response.error(403, "Forbidden");
        return checkNext(req);
    }
}

class CacheHandler extends RequestHandler {
    private final Map<String, Response> cache = new HashMap<>();
    @Override public Response handle(Request req) {
        Response c = cache.get(req.path);
        if (c != null) return c;
        Response result = checkNext(req);
        cache.put(req.path, result);
        return result;
    }
}

// 사용 — 체인 조립
public class Main {
    public static void main(String[] args) {
        RequestHandler chain = new AuthHandler();
        chain.setNext(new LoggingHandler())
             .setNext(new AuthorizationHandler())
             .setNext(new CacheHandler());

        Response r = chain.handle(new Request("/api/users", token, user));
    }
}
```

### 5.2 함수형 (Java 8+)

```java
import java.util.function.Function;

Function<Request, Response> auth = req -> {
    if (req.token == null) return Response.error(401, "Auth required");
    return null;    // 다음으로
};

Function<Request, Response> logging = req -> {
    System.out.println(req.path);
    return null;
};

// 체인 합성
Function<Request, Response> chain = req -> {
    Response r = auth.apply(req); if (r != null) return r;
    r = logging.apply(req); if (r != null) return r;
    return process(req);
};
```

함수형 스타일 — 클래스 적게, 람다 많이.

## 6. 변형 (Variants)

### 6.1 모든 핸들러 실행 (filter chain)
요청을 *처리* 가 아니라 *수정* 하면서 통과. Servlet Filter 가 이 방식.

### 6.2 Tree-shaped chain
하나가 여러 다음으로 분기. 흔한 GoF 변형은 아니지만 가능.

### 6.3 Command + Chain 결합
Command (14) 객체들이 chain 을 이룸.

## 7. 함정 / 흔한 오해

### 7.1 끝까지 못 처리됨
모든 핸들러가 next 만 호출하고 아무도 처리 X → null 반환. 마지막에 *기본 핸들러* 또는 명시적 에러.

### 7.2 순서 의존
인증 → 권한 → 본처리 순서가 중요. 잘못 조립 시 보안 구멍.

### 7.3 chain 이 너무 길다
디버깅 어려움. 트레이스를 찍을 수단 (handler name 로깅).

### 7.4 Decorator (09) 와의 차이
- **Decorator**: 모든 wrapping 이 *추가 행동* 수행.
- **CoR**: 한 명만 처리. 나머지는 통과.
- 코드 비슷하지만 *의도* 다름.

## 8. 관련 패턴

- **Composite (08)**: 트리에서 위로 올라가며 처리하면 CoR.
- **Decorator (09)**: 형태 비슷. 처리 분기 vs 누적.
- **Command (14)**: 핸들러가 Command 처리.

## 9. 실무 사례

- Servlet Filter chain
- Spring Security `FilterChain`
- Java exception handling (try/catch chain — 못 잡으면 위로)
- Logging frameworks 의 Appender chain
- Express.js (Node) middleware
- Logback `Filter` (deny/accept/neutral)
- AWS API Gateway authorizers
- Netty `ChannelPipeline`

```java
// Servlet Filter — Chain of Responsibility 의 산업 표준
public class AuthFilter implements Filter {
    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain) {
        if (!isAuthenticated(req)) {
            ((HttpServletResponse) resp).sendError(401);
            return;
        }
        chain.doFilter(req, resp);    // 다음 필터로
    }
}
```
