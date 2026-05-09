# Spring Kafka의 핵심 객체들 — `ProducerFactory`, `KafkaTemplate`, `ConsumerRecord`, `ProducerRecord`

> `OutboxRelay`, `KafkaProducerStub`, `KafkaTestConsumer`에 등장하는 Kafka/Spring-Kafka 타입들을 풀어 정리한 학습 노트.
> "이 객체가 누구로부터 와서 어디로 가는가"의 흐름을 한 그림으로 보이게 만든다.

---

## 0. 등장 인물 한 줄 요약

| 객체 | 정체 | 어디서 옴 |
|------|------|---------|
| `ProducerRecord<K,V>` | 발행할 메시지 한 건 (topic, key, value, headers) | kafka-clients (apache 원본) |
| `ConsumerRecord<K,V>` | 수신한 메시지 한 건 | kafka-clients |
| `ProducerFactory<K,V>` | 카프카 producer 클라이언트의 **생성 공장** | spring-kafka |
| `ConsumerFactory<K,V>` | consumer 클라이언트 생성 공장 | spring-kafka |
| `KafkaTemplate<K,V>` | producer를 감싼 고수준 API (template 패턴) | spring-kafka |
| `SendResult<K,V>` | send() 결과 (record + record metadata) | spring-kafka |
| `RecordMetadata` | 발행된 메시지의 위치(파티션, 오프셋, 시각 등) | kafka-clients |
| `KafkaConsumer<K,V>` | 저수준 컨슈머 (raw API) | kafka-clients |
| `@KafkaListener` | 컨슈머를 어노테이션으로 등록 | spring-kafka |

크게 두 진영:

- **kafka-clients (Apache Kafka 원본)**: 데이터 표현(`ProducerRecord`, `ConsumerRecord`, `RecordMetadata`)과 raw 클라이언트(`KafkaProducer`, `KafkaConsumer`).
- **spring-kafka**: 위에 Spring 친화적인 추상(`KafkaTemplate`, `ProducerFactory`, `@KafkaListener`)을 얹음.

---

## 1. 데이터 표현 — `ProducerRecord` / `ConsumerRecord`

### 1.1 `ProducerRecord<K,V>` — 발행할 메시지

```java
ProducerRecord<String,String> record = new ProducerRecord<>(
    "order-events",          // topic
    "42",                    // key (선택, 같은 키는 같은 파티션에 모임)
    "{\"orderId\":42}"       // value (payload)
);
record.headers().add("messageId", "outbox-1".getBytes());
record.headers().add("eventType", "OrderConfirmed".getBytes());
```

| 필드 | 역할 |
|------|------|
| topic | 어느 토픽에 보낼지 |
| partition | 명시적 파티션 지정 (보통 null → key 해시로 결정) |
| timestamp | 메시지 시각 (보통 자동) |
| key | 같은 key는 같은 파티션에 → 순서 보장 단위 |
| value | 본문 (보통 JSON 문자열 또는 직렬화 가능 객체) |
| headers | 메타데이터 (messageId, eventType 등) |

### 1.2 헤더 vs 페이로드

```
[페이로드(value)에 모든 정보를 박는 경우]
{
  "messageId": "outbox-1",
  "eventType": "OrderConfirmed",
  "orderId": 42,
  "amount": 1000
}

[헤더 + 페이로드 분리 (이 프로젝트의 선택)]
headers: { messageId: "outbox-1", eventType: "OrderConfirmed" }
value:   { "orderId": 42, "amount": 1000 }
```

분리하는 이유:
- **메타데이터를 파싱 안 하고 라우팅**: 컨슈머가 헤더만 보고 처리/스킵 결정 가능 (페이로드 파싱 비용 절감).
- **스키마 진화 안전**: payload 스키마가 바뀌어도 헤더 메타는 안정.
- **표준화**: messageId, eventType은 어떤 도메인이든 공통이므로 헤더가 자연스러움.

### 1.3 `ConsumerRecord<K,V>` — 수신한 메시지

```java
ConsumerRecord<String,String> r = consumer.poll(...).iterator().next();
String topic = r.topic();
int partition = r.partition();
long offset = r.offset();
String key = r.key();
String value = r.value();
Header h = r.headers().lastHeader("messageId");
String messageId = h == null ? null : new String(h.value());
```

`ProducerRecord`와 거의 대칭이지만 `partition`/`offset`/`timestamp`가 **확정 값**으로 채워져 있음 (브로커가 부여한 위치).

이 프로젝트의 `NotificationKafkaListener`가 정확히 이 객체를 받는다:

```java
@KafkaListener(topics = "order-events", groupId = "notification")
public void onMessage(ConsumerRecord<String,String> record) {
    String messageId = header(record, "messageId");
    ...
}
```

---

## 2. `ProducerFactory<K,V>` — Producer 생성 공장

### 2.1 정체

Kafka의 `KafkaProducer`(원본 클라이언트)는 **무겁고 thread-safe한 객체**다 — 한 인스턴스를 여러 스레드가 공유하는 게 정상. `ProducerFactory`는 이 인스턴스의 생성과 라이프사이클을 추상한 Spring 인터페이스.

```java
public interface ProducerFactory<K, V> {
    Producer<K, V> createProducer();           // 새 producer 인스턴스
    Producer<K, V> createProducer(String txId);  // 트랜잭셔널 producer
    Map<String, Object> getConfigurationProperties();   // bootstrap.servers 등
    Serializer<K> getKeySerializer();
    Serializer<V> getValueSerializer();
    ...
}
```

### 2.2 누가 만드는가

이 프로젝트에선 직접 안 만든다. Spring Boot의 `KafkaAutoConfiguration`이 자동 생성.

```java
// Spring Boot 내부 (단순화)
@Bean
@ConditionalOnMissingBean
public ProducerFactory<?, ?> kafkaProducerFactory(KafkaProperties properties) {
    DefaultKafkaProducerFactory<?, ?> factory =
        new DefaultKafkaProducerFactory<>(properties.buildProducerProperties());
    return factory;
}
```

`application.yml`의 `spring.kafka.producer.*` 또는 `spring.kafka.bootstrap-servers`가 이 빈의 설정을 채움.

### 2.3 왜 직접 쓰는가 (이 프로젝트)

`KafkaProducerStub`이 `KafkaTemplate`을 상속하는데, 부모 생성자가 `ProducerFactory`를 요구한다:

```java
public class KafkaProducerStub extends KafkaTemplate<String, String> {
    public KafkaProducerStub(ProducerFactory<String, String> producerFactory) {
        super(producerFactory);   // KafkaTemplate(ProducerFactory) 생성자 호출
    }
}
```

`@TestConfiguration`이 Spring 자동 구성된 `ProducerFactory`를 주입해 stub에 전달.

```java
@Bean @Primary
public KafkaProducerStub kafkaProducerStub(ProducerFactory<String, String> pf) {
    return new KafkaProducerStub(pf);
}
```

### 2.4 `ConsumerFactory<K,V>`도 같은 패턴

`@KafkaListener`를 쓸 때 Spring이 `ConsumerFactory`로 컨슈머 인스턴스를 만든다. 이 프로젝트의 `KafkaTestConsumer`는 **Spring 추상 안 쓰고 raw `KafkaConsumer`** 를 직접 쓰지만, 운영 컨슈머 경로는 `ConsumerFactory`를 거친다.

---

## 3. `KafkaTemplate<K,V>` — 고수준 Producer API

### 3.1 역할

`Producer.send()`를 직접 호출하면 매번 직렬화·예외 처리·콜백 등록을 손으로 해야 한다. `KafkaTemplate`은 그 보일러플레이트를 묶은 **template 패턴**.

```java
@Autowired KafkaTemplate<String,String> kafka;

CompletableFuture<SendResult<String,String>> future = kafka.send(record);
// .get(timeout) — 동기 대기
// .whenComplete(...) — 비동기 콜백
```

### 3.2 주요 메서드

| 메서드 | 동작 |
|-------|------|
| `send(topic, key, value)` | 가장 간단한 형태 |
| `send(ProducerRecord)` | 헤더 등 자유롭게 |
| `send(Message)` | Spring Messaging 메시지 |
| `executeInTransaction(...)` | 카프카 트랜잭션 (트랜잭셔널 producer 필요) |
| `flush()` | 버퍼 비우기 (테스트 시 유용) |

### 3.3 반환 타입 — `CompletableFuture<SendResult>`

Spring Kafka 3.0+(2022)부터 `ListenableFuture`가 deprecated되고 `CompletableFuture`로 변경.

```java
kafka.send(record)
    .thenAccept(result -> log.info("sent at offset {}", result.getRecordMetadata().offset()))
    .exceptionally(ex -> { log.error("send failed", ex); return null; });
```

이 프로젝트의 `OutboxRelay`는 동기 대기를 쓴다:

```java
kafka.send(record).get(5, TimeUnit.SECONDS);
```

`get(timeout)`이 던지는 예외를 catch해 retry_count 누적. 이유:
- relay는 백그라운드 폴링이라 응답을 기다려도 요청 처리에 영향 없음.
- 동기로 받아야 outbox 행 상태 갱신을 같은 트랜잭션에서 처리 가능.

### 3.4 `SendResult<K,V>`

```java
public class SendResult<K,V> {
    ProducerRecord<K,V> producerRecord;
    RecordMetadata recordMetadata;   // partition, offset, timestamp 등
}
```

성공 시 받은 결과. 보통 `recordMetadata.offset()`을 로깅하거나 메트릭에 기록.

---

## 4. `RecordMetadata` — 브로커가 부여한 위치 정보

```java
RecordMetadata md = sendResult.getRecordMetadata();
md.topic();         // "order-events"
md.partition();     // 0~N-1
md.offset();        // 파티션 내 순서
md.timestamp();     // 브로커가 기록한 시각
```

**같은 메시지가 두 번 발행되면 offset이 다르게 부여됨** — 멱등성은 컨슈머 측 책임 (Inbox 패턴).

---

## 5. `KafkaConsumer<K,V>` — 저수준 컨슈머

### 5.1 정체

raw API. `KafkaTestConsumer`가 직접 사용:

```java
KafkaConsumer<String,String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(List.of("order-events"));
ConsumerRecords<String,String> records = consumer.poll(Duration.ofSeconds(5));
records.forEach(r -> ...);
```

### 5.2 운영 컨슈머는 `@KafkaListener`로 추상

```java
@Component
public class NotificationKafkaListener {
    @KafkaListener(topics = "order-events", groupId = "notification")
    public void onMessage(ConsumerRecord<String,String> record) {
        ...
    }
}
```

내부적으로:
1. Spring이 `ConsumerFactory`로 `KafkaConsumer` 인스턴스 생성.
2. 별도 스레드(`KafkaMessageListenerContainer`)에서 `poll()` 루프 실행.
3. 받은 레코드를 어노테이션 메서드에 전달.
4. 메서드 정상 종료 시 자동 commit (auto-commit 또는 ack).

### 5.3 왜 테스트는 raw 컨슈머를 쓰나

- **격리**: `@KafkaListener`는 운영 코드의 일부라 같은 토픽을 구독하면 운영 컨슈머와 경쟁 (`groupId`가 같으면 한 쪽만 받음).
- **검증 시점 통제**: 테스트가 "이 시점에 메시지 1건이 와야 한다"를 직접 검증해야 하므로 동기 polling이 편함.
- **헤더 직접 검사**: 어노테이션 호출은 deserialization 후 객체로 받지만, raw는 원본 ConsumerRecord 그대로.

---

## 6. 흐름 다이어그램 — 이 프로젝트의 outbox → kafka → notification

```
[1. OrderService.confirm]
         │
         ▼
   outboxRepository.save(OutboxEvent)         ← DB INSERT (PostgreSQL)
         │
         ▼ (커밋 후)
[2. OutboxRelay.poll @Scheduled]
         │
         ▼
   List<OutboxEvent> pending
         │
         ▼
   for each: ProducerRecord 생성
         │   (topic, key=aggregateId, value=payload, headers=[messageId, eventType])
         ▼
   kafkaTemplate.send(record).get(5s)         ← KafkaTemplate
         │
         ▼ (내부)
   ProducerFactory.createProducer()
         │
         ▼
   KafkaProducer.send() → 브로커
         │
         ▼
   SendResult { RecordMetadata { partition, offset } }
         │
         ▼
   markPublished()                             ← outbox 행 상태 갱신

         ═════════════════════════════
         [Kafka 브로커: order-events 토픽]
         ═════════════════════════════
                     │
                     ▼
[3. NotificationKafkaListener]
         │   @KafkaListener (groupId="notification")
         ▼
   ConsumerRecord<String,String> 받음
         │
         ▼
   header("messageId"), header("eventType"), value
         │
         ▼
   inboxConsumer.consume(messageId, eventType, payload)
         │
         ▼
   InboxRepository.existsByMessageId()         ← 멱등성 체크
         ├─ 있으면 return
         └─ 없으면 inbox INSERT + notificationService.process()
```

각 단계의 객체 변환을 한 줄로 따라가면 outbox 패턴 전체가 한눈에 들어온다.

---

## 7. Serializer / Deserializer

### 7.1 모든 메시지는 byte[]

Kafka 내부에서 메시지는 **바이트 배열**. 자바 객체 ↔ 바이트 변환을 Serializer/Deserializer가 담당.

| 자바 타입 | (De)Serializer |
|---------|--------------|
| `String` | `StringSerializer` / `StringDeserializer` |
| `byte[]` | `ByteArraySerializer` / `ByteArrayDeserializer` |
| `Long`, `Integer` | `LongSerializer`, `IntegerSerializer` |
| 임의 객체 (JSON) | `JsonSerializer` / `JsonDeserializer` (spring-kafka 제공) |
| Avro / Protobuf | Confluent의 `KafkaAvroSerializer` 등 |

### 7.2 이 프로젝트의 선택

```java
// KafkaTestConsumer
props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
```

key·value를 그냥 **String으로** 다룬다. JSON 직렬화는 `ObjectMapper`로 미리 한 결과를 넘김 (`OutboxWriter`에서). 이유:

- **payload 스키마 명시적 관리**: `OrderConfirmedPayload` record가 진실의 원천. 자동 직렬화에 맡기면 클래스 변경이 외부에 새기 쉬움.
- **헤더 기반 라우팅 단순**: value는 그냥 문자열, 의미는 헤더의 eventType으로 결정.
- **Phase 2의 학습 목표**: 직렬화 추상보다는 outbox/inbox 흐름 자체에 집중.

운영 시스템에선 보통 Avro + Schema Registry 조합으로 가지만, 이 프로젝트 범위 밖.

---

## 8. 트랜잭션 관련 — 카프카 트랜잭션 vs 이 프로젝트의 선택

### 8.1 Kafka 트랜잭션이란

producer가 여러 메시지를 **atomic하게** 발행 (모두 성공 또는 모두 실패). consumer는 `isolation.level=read_committed`로 커밋된 트랜잭션만 읽음.

### 8.2 왜 이 프로젝트는 안 쓰는가

이미 더 강한 보장이 있다 — **Outbox 패턴**.

```
[Outbox]
DB 트랜잭션이 outbox 행을 영속화 → 어차피 단일 시스템(DB)의 트랜잭션
Relay가 비동기로 발행 → at-least-once + Inbox 멱등성 = effectively-exactly-once

[Kafka 트랜잭션]
DB 트랜잭션과 Kafka 트랜잭션을 묶을 수 없음 (이중 쓰기 문제 그대로)
```

Outbox가 이중 쓰기를 푸는 정통 방법이라 Kafka 트랜잭션은 중복.

---

## 9. 자주 만나는 함정

### 9.1 `partition` 분산이 막혀 한 파티션만 쓰임

key를 모든 메시지에 같은 값으로 넣으면 같은 파티션만 사용 → 병렬성 저하.

→ key를 적절한 카디널리티로 (이 프로젝트는 `aggregateId` = orderId라 다양함).

### 9.2 헤더 값의 인코딩

```java
record.headers().add("messageId", "abc".getBytes());   // UTF-8 기본
new String(header.value());                              // 디코딩 시도
```

명시적으로 `StandardCharsets.UTF_8`을 지정하는 게 안전:

```java
record.headers().add("messageId", "abc".getBytes(StandardCharsets.UTF_8));
new String(header.value(), StandardCharsets.UTF_8);
```

### 9.3 `KafkaTemplate.send().get()` 무한 대기

`.get()` 호출이 타임아웃 인자가 없으면 영원히 블록. 항상 `.get(timeout, unit)` 사용.

### 9.4 KafkaConsumer는 thread-safe하지 않음

여러 스레드에서 같은 인스턴스로 `poll()` 호출 금지. 한 스레드 = 한 컨슈머.

### 9.5 `@KafkaListener` 메서드 안에서 새 트랜잭션이 안 시작됨

기본 동작: `@KafkaListener`가 자동으로 트랜잭션 경계를 만들지 않음. `@Transactional`을 메서드에 붙여야 함.

이 프로젝트는 `InboxConsumer.consume`에 `@Transactional`을 명시 — 멱등성 체크 + INSERT + 비즈니스 로직을 한 트랜잭션으로 묶음.

---

## 10. 한 줄 요약

> **`ProducerRecord`/`ConsumerRecord`는 메시지 데이터, `ProducerFactory`/`ConsumerFactory`는 클라이언트 생성 공장, `KafkaTemplate`은 producer 고수준 API, `@KafkaListener`는 consumer 고수준 API.**
> 이 프로젝트는 운영 경로에 `KafkaTemplate` + `@KafkaListener`를 쓰고, 테스트에선 `KafkaTemplate` 대체(`KafkaProducerStub`)와 raw `KafkaConsumer` 헬퍼(`KafkaTestConsumer`)를 함께 둔다.
> Serializer는 `String`으로 단순화하고 JSON은 애플리케이션이 직접 다룬다 — payload 스키마(`OrderConfirmedPayload`)를 진실의 원천으로 명시.

---

## 11. 함께 보면 좋은 자료

- 공식: [Spring for Apache Kafka Reference](https://docs.spring.io/spring-kafka/reference/html/)
- `docs/study/event-driven-tools.md` — Kafka의 위치 (커밋 로그 사상)
- `docs/study/event-store-vs-simple-bus.md` — 이 프로젝트가 왜 Kafka 트랜잭션 대신 Outbox를 쓰는지
- `docs/study/spring-test-property-injection.md` — Kafka 컨테이너의 무작위 포트가 어떻게 ProducerFactory에 연결되는지
- `docs/phase-2-tdd.md` Step 0.4, 3, 5 — KafkaProducerStub / OutboxRelay / NotificationKafkaListener 구현
