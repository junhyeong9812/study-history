# 자바 제네릭 와일드카드 `<?>` — 짧게

> `Future<?> f1 = pool.submit(task)` 같은 코드에 등장하는 `<?>`가 무엇인가를 정리한 짧은 학습 노트.

---

## 0. 등장한 코드

```java
Runnable task = () -> inboxConsumer.consume("msg-1", "t", "{}");
ExecutorService pool = Executors.newFixedThreadPool(2);
Future<?> f1 = pool.submit(task);    // ← 여기
Future<?> f2 = pool.submit(task);
f1.get(); f2.get();
```

`Future<?>`의 `<?>`는 **"타입 매개변수가 무엇인지 신경 쓰지 않는다"** 는 표시.

---

## 1. 왜 그냥 `Future`로 안 쓰는가

```java
Future f1 = pool.submit(task);   // ❌ raw type 경고
```

자바 5+에서 제네릭 타입은 항상 타입 매개변수를 함께 적어야 한다. 안 적으면 **raw type**이 되어 컴파일러 경고와 unchecked 안전성 손실.

`<?>`는 **"타입 인자를 명시하되 구체 타입엔 무관심"** 의 표준 표기. 컴파일러가 raw type 경고를 안 띄우고 타입 안전성도 일부 보장.

```java
Future<?> f = pool.submit(task);   // ✅
```

---

## 2. `Future<?>`가 의미하는 것

`ExecutorService.submit`에는 두 가지 오버로드가 있다.

```java
<T> Future<T> submit(Callable<T> task);    // 결과 있는 작업
Future<?>     submit(Runnable task);        // 결과 없는 작업 (Runnable.run은 void)
```

`Runnable`은 결과가 없으니 반환되는 `Future`의 타입 매개변수가 의미 없다. JDK가 그래서 `Future<?>`로 시그니처를 정의 — "결과 타입은 어떤 타입이든, 의미 있는 값은 안 들어 있음".

`f.get()`은 호출 가능하지만 항상 `null` 반환 (Runnable이 끝났다는 신호로만 사용).

---

## 3. 와일드카드의 세 가지 형태

| 형태 | 의미 |
|------|------|
| `List<?>` | unbounded — 어떤 타입이든 무관 |
| `List<? extends Number>` | upper bound — Number 또는 그 하위 타입 (`Integer`, `Double` 등) |
| `List<? super Integer>` | lower bound — Integer 또는 그 상위 타입 (`Number`, `Object`) |

```java
void printAll(List<?> list) {
    for (Object o : list) System.out.println(o);    // 읽기는 OK
    list.add(new Object());                          // ❌ 쓰기는 거의 다 막힘
}
```

`<?>`는 **읽기 전용에 가깝다** — 어떤 타입이 들어 있는지 모르니 안전하게 추가할 수 없음.

---

## 4. PECS 규칙

> **P**roducer **E**xtends, **C**onsumer **S**uper

- 컬렉션이 **값을 생산**(읽기 전용)하면 `extends` (upper bound).
- 컬렉션이 **값을 소비**(쓰기 전용)하면 `super` (lower bound).

```java
// 읽기만 (numbers에서 값을 꺼내 sum)
double sum(List<? extends Number> numbers) { ... }

// 쓰기만 (dest에 값을 넣음)
void addInts(List<? super Integer> dest) { dest.add(42); }
```

`<?>`(unbounded)는 양쪽 다 거의 못 쓰니 **"타입에 무관심하게 다루기만 할 때"** 만 유용.

---

## 5. 일상에서 만나는 패턴

```java
// 이번 코드처럼 — 결과를 안 쓰고 완료만 기다릴 때
Future<?> f = pool.submit(task);
f.get();

// Class 객체를 타입 무관하게 받기
Class<?> type = obj.getClass();

// 컬렉션을 안전하게 출력만 할 때
void log(List<?> items) { items.forEach(System.out::println); }
```

이 코드의 `Future<?>`는 첫 번째 사례 — **완료 신호만 받으면 되니** 결과 타입을 표현할 필요 없음.

---

## 6. 함정 — `Future<?>`로는 결과를 못 받는다

```java
Future<?> f = pool.submit((Callable<String>) () -> "hello");
String s = f.get();   // ❌ Object를 String에 못 담음
Object o = f.get();   // ✅ Object로 받아야 함
```

결과를 쓰려면 구체 타입 필요:

```java
Future<String> f = pool.submit(() -> "hello");
String s = f.get();   // ✅
```

이번 코드는 결과가 없는 `Runnable`이라 `Future<?>`가 정확한 선택.

---

## 7. 한 줄 요약

> **`<?>`는 "타입 인자 자리에 무엇이든 들어와도 OK, 나는 신경 안 쓴다"는 표시.**
> raw type(`Future`)을 쓰면 안 되니 그 자리 채우기용. **읽기만 / 완료만 기다리기만** 같은 타입 무관심 시나리오에 쓴다. 결과 타입을 다뤄야 한다면 구체 타입을 명시(`Future<String>`).

---

## 8. 함께 보면 좋은 자료

- 공식: [Java Tutorial — Wildcards](https://docs.oracle.com/javase/tutorial/java/generics/wildcards.html)
- *Effective Java* — Item 31: Use bounded wildcards to increase API flexibility (PECS)
- `docs/study/java-functional-toolbox.md` — `Consumer<T>` 등 다른 제네릭 사용 사례
