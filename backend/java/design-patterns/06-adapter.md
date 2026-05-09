# 06 — Adapter

> **분류**: 구조 패턴 (Structural)
> **한 줄**: 호환되지 않는 두 인터페이스를 *변환* 하여 함께 동작하게 한다.

---

## 1. 의도 (Intent)

> "Convert the interface of a class into another interface clients expect. Adapter lets classes work together that couldn't otherwise because of incompatible interfaces."

기존 클래스를 *그대로* 사용하면서 새로운 인터페이스에 맞추고 싶을 때.

## 2. 문제 상황 (Motivation)

서드파티 라이브러리의 결제 API 가 `processCreditCard(cardNum, amount)` 인데, 우리 코드는 `Payment.execute()` 인터페이스를 기대한다.

```java
// 우리 코드 (변경 불가)
interface Payment {
    void execute(double amount);
}

// 서드파티 (변경 불가)
class StripeApi {
    public void chargeCreditCard(String cardNumber, double amount) { ... }
}

// ❌ 직접 호출 시 인터페이스 불일치
Payment p = new StripeApi();    // 컴파일 에러
```

해결 : *어댑터* 가 둘 사이를 잇는다.

## 3. 구조 (Structure)

```
Client       Target (interface)        Adaptee (기존)
  │           ┌──────────┐            ┌──────────────┐
  └─uses ───► │ request()│            │ specificRequest()
              └────┬─────┘            └──────┬───────┘
                   │                          ▲
              ┌────┴─────┐                    │
              │ Adapter  │ ───── delegates ───┘
              │ request()│ → adaptee.specificRequest()
              └──────────┘
```

## 4. 참여자 (Participants)

- **Target**: client 가 기대하는 인터페이스.
- **Adaptee**: 변환 대상 (기존 클래스).
- **Adapter**: Target 구현 + Adaptee 보유 + 호출 변환.
- **Client**: Target 만 알고 있음.

## 5. Java 예제

### 5.1 Object Adapter (위임 — 권장)

```java
// Target — 우리 코드의 인터페이스
interface Payment {
    void execute(double amount);
}

// Adaptee — 서드파티
class StripeApi {
    public void chargeCreditCard(String cardNumber, double amount) {
        System.out.printf("Stripe: charging $%.2f to card %s%n", amount, cardNumber);
    }
}

// Adapter
class StripePaymentAdapter implements Payment {
    private final StripeApi stripe;
    private final String cardNumber;

    public StripePaymentAdapter(StripeApi stripe, String cardNumber) {
        this.stripe = stripe;
        this.cardNumber = cardNumber;
    }

    @Override
    public void execute(double amount) {
        // 인터페이스 변환 + 추가 데이터 보충
        stripe.chargeCreditCard(cardNumber, amount);
    }
}

// 사용
public class Main {
    public static void main(String[] args) {
        Payment payment = new StripePaymentAdapter(new StripeApi(), "4242424242424242");
        payment.execute(99.99);
    }
}
```

### 5.2 Class Adapter (다중 상속 — Java 에선 제한적)

```java
// Java 는 다중 클래스 상속 X. interface 다중 구현은 가능
class StripeClassAdapter extends StripeApi implements Payment {
    @Override
    public void execute(double amount) {
        chargeCreditCard("default-card", amount);
    }
}
```

부모 클래스의 protected 멤버 접근 가능. 단, 단일 상속이라 *Adaptee 가 인터페이스* 일 때만 깔끔.

### 5.3 Two-way Adapter

```java
class TwoWayAdapter implements Target, OtherTarget {
    public void targetRequest() { ... }
    public void otherRequest() { ... }
}
```

두 인터페이스 동시에 구현. 양방향 변환.

## 6. 변형 (Variants)

- **Default Method (Java 8+)**: 인터페이스에 default 메서드로 어댑터 역할 수행 가능.
- **Lambda 어댑터**: `Runnable r = () -> something.differentApi();` — 함수형 인터페이스를 어댑터로.
- **Stream Collectors**: `stream.collect(Collectors.toList())` 가 일종의 어댑터.

## 7. 함정 / 흔한 오해

### 7.1 Adapter 와 Decorator (09) 의 차이
- **Adapter**: 인터페이스 *변환*. 입력과 출력 인터페이스가 *다름*.
- **Decorator**: 같은 인터페이스에 *기능 추가*. 입출력 동일.

### 7.2 Adapter 와 Facade (10) 의 차이
- **Adapter**: 한 객체를 다른 인터페이스로 감싸기. 1:1.
- **Facade**: 여러 객체를 *단순한 새 인터페이스* 로 통합. N:1.

### 7.3 Adapter 와 Proxy (12) 의 차이
- **Adapter**: 인터페이스 *변환*.
- **Proxy**: 같은 인터페이스 *유지*, 접근 제어/지연/캐시 등 추가.

### 7.4 어댑터 남발
모든 라이브러리 호출을 어댑터로 감싸면 *어댑터 지옥*. 인터페이스 차이가 작거나 자주 바뀌지 않으면 직접 호출이 단순.

## 8. 관련 패턴

- **Bridge (07)**: 처음부터 *분리해서 설계*. Adapter 는 *나중에 호환*.
- **Decorator (09)**: 인터페이스 유지 + 기능 추가.
- **Proxy (12)**: 인터페이스 유지 + 접근 제어.
- **Facade (10)**: 여러 클래스 → 단순 인터페이스.

## 9. 실무 사례

- `java.util.Arrays.asList(T[])` — 배열을 List 로 어댑팅
- `java.io.InputStreamReader` — InputStream → Reader (byte → char)
- `java.io.OutputStreamWriter`
- Spring `HandlerAdapter` — 다양한 컨트롤러를 통일 인터페이스로
- SLF4J — 다양한 로깅 프레임워크 (Log4j, Logback, JUL) 의 어댑터
- JDBC drivers — DB 별 native API 를 JDBC interface 로 어댑팅

### Java I/O 예

```java
InputStream stdin = System.in;            // byte stream
Reader reader = new InputStreamReader(stdin);    // ← Adapter: byte → char
BufferedReader br = new BufferedReader(reader);  // ← Decorator: 버퍼링
```

`InputStreamReader` 가 어댑터 (byte 와 char 인터페이스 변환), `BufferedReader` 가 데코레이터 (같은 Reader 인터페이스 + 버퍼).
