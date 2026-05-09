# 10 — Facade

> **분류**: 구조 패턴 (Structural)
> **한 줄**: 복잡한 서브시스템의 *단순한 인터페이스* 를 외부에 제공.

---

## 1. 의도 (Intent)

> "Provide a unified interface to a set of interfaces in a subsystem. Facade defines a higher-level interface that makes the subsystem easier to use."

서브시스템의 *복잡한 디테일* 을 숨기고, 흔한 사용 패턴에 대한 진입점만 노출.

## 2. 문제 상황 (Motivation)

비디오 변환 라이브러리 — 클라이언트가 모든 디테일을 알아야 함 :

```java
// ❌ Client 가 5 개 클래스를 직접 다룸
VideoFile file = new VideoFile("vid.mp4");
Codec codec = CodecFactory.extract(file);
if (codec instanceof MPEG4Codec) {
    Encoder encoder = new MPEG4Encoder();
    BitrateReader reader = BitrateReader.read(file, codec);
    BitrateReader result = BitrateReader.convert(reader, encoder);
    AudioMixer mixer = new AudioMixer();
    File converted = mixer.fix(result);
    saveToDisk(converted);
}
```

라이브러리가 강력하지만 사용이 너무 복잡.

해결 : Facade 가 흔한 use case 한 줄로.

```java
VideoConverter converter = new VideoConverter();
File ogg = converter.convert("vid.mp4", "ogg");
```

## 3. 구조 (Structure)

```
Client          Facade                  Subsystem (여러 클래스)
  │             ┌──────────┐          ┌────────┐ ┌────────┐
  └───uses───►  │ simple() │ ───uses──► │ ClassA │ │ ClassB │
                └──────────┘            └────────┘ └────────┘
                                       ┌────────┐ ┌────────┐
                                       │ ClassC │ │ ClassD │
                                       └────────┘ └────────┘
```

## 4. 참여자 (Participants)

- **Facade**: 단순 인터페이스 제공. 내부에서 서브시스템 조립.
- **Subsystem classes**: 실제 일.
- **Client**: Facade 만 사용 (또는 필요시 서브시스템 직접 — 옵션).

## 5. Java 예제

### 5.1 비디오 변환 Facade

```java
// Subsystem — 복잡한 클래스들
class VideoFile {
    private final String name;
    public VideoFile(String name) { this.name = name; }
    public String getName() { return name; }
}

interface Codec {}
class MPEG4Codec implements Codec {}
class OggCompressionCodec implements Codec {}

class CodecFactory {
    public static Codec extract(VideoFile file) {
        if (file.getName().endsWith(".mp4")) return new MPEG4Codec();
        return new OggCompressionCodec();
    }
}

class BitrateReader {
    public static VideoFile read(VideoFile file, Codec codec) {
        System.out.println("Reading bitrate...");
        return file;
    }
    public static VideoFile convert(VideoFile buf, Codec codec) {
        System.out.println("Converting bitrate...");
        return buf;
    }
}

class AudioMixer {
    public VideoFile fix(VideoFile result) {
        System.out.println("Fixing audio...");
        return result;
    }
}

// Facade
class VideoConverter {
    public VideoFile convert(String filename, String format) {
        System.out.println("VideoConverter: starting...");

        VideoFile file = new VideoFile(filename);
        Codec sourceCodec = CodecFactory.extract(file);
        Codec destinationCodec = format.equals("mp4")
            ? new MPEG4Codec()
            : new OggCompressionCodec();

        VideoFile buffer = BitrateReader.read(file, sourceCodec);
        VideoFile result = BitrateReader.convert(buffer, destinationCodec);
        result = (new AudioMixer()).fix(result);

        System.out.println("VideoConverter: done");
        return result;
    }
}

// Client
public class Main {
    public static void main(String[] args) {
        VideoConverter c = new VideoConverter();
        VideoFile ogg = c.convert("vid.mp4", "ogg");
        // 한 줄로 끝
    }
}
```

### 5.2 도메인 서비스의 Facade (실무 흔함)

```java
// 여러 application service 를 묶은 facade
@Service
public class OrderFacade {
    private final OrderService orders;
    private final PaymentService payments;
    private final ShippingService shipping;
    private final NotificationService notify;

    public OrderId placeOrder(PlaceOrderCommand cmd) {
        OrderId id = orders.create(cmd);
        payments.charge(id, cmd.amount);
        shipping.prepare(id, cmd.address);
        notify.sendConfirmation(cmd.userId, id);
        return id;
    }
}
```

→ 흔한 use case ("주문 시작") 를 한 메서드로. Facade 안에 *순서 + 트랜잭션* 응집.

## 6. 변형 (Variants)

- **Multi-facade**: 한 서브시스템에 *역할별* 여러 facade (예 : `AdminFacade`, `UserFacade`).
- **Singleton facade**: facade 가 stateless 면 singleton.
- **Layered facade**: facade 위에 또 facade (예 : application facade → use case facade).

## 7. 함정 / 흔한 오해

### 7.1 God Object 화
모든 메서드를 facade 하나에 넣으면 *god object*. 책임 분리 (이건 Order facade, 저건 Inventory facade).

### 7.2 Adapter (06) 와의 차이
- **Adapter**: 한 객체의 *인터페이스 변환*. 1:1.
- **Facade**: 여러 객체의 *통합 인터페이스*. N:1.

### 7.3 Mediator (17) 와의 차이
- **Facade**: 클라이언트가 서브시스템에 *접근 단순화*. 단방향.
- **Mediator**: 객체들 *서로의* 통신 중재. 양방향.

### 7.4 강제하지 않는다
GoF 원칙 : Facade 가 *대안* 일 뿐. 필요하면 서브시스템 직접 접근 가능. 강제로 막으면 유연성 손실.

## 8. 관련 패턴

- **Abstract Factory (01)**: facade 가 family 생성을 단순화.
- **Mediator (17)**: 비슷하지만 의도 다름.
- **Singleton (05)**: facade 가 singleton 인 경우.

## 9. 실무 사례

- `java.net.URL` — `openConnection()`, `openStream()` 으로 HTTP 디테일 숨김
- `javax.faces.context.FacesContext`
- Spring `JdbcTemplate` — JDBC 의 verbosity 를 단순화
- SLF4J `LoggerFactory` — 로깅 백엔드 선택 / 초기화 숨김
- Hibernate `Session` — 트랜잭션 / 캐시 / SQL 생성을 통합
- 외부 API 클라이언트 (`StripeClient.createCharge()`) — 내부에 HTTP / auth / retry 다 들어있음
