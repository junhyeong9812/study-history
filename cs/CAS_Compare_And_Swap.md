# CAS (Compare-And-Swap)

## 한 줄 정의

> **"메모리의 값이 내가 예상한 값이면, 새 값으로 바꿔줘. 아니면 아무것도 하지 마."**

이걸 **하나의 CPU 명령어로 원자적으로** 실행하는 게 CAS예요.

---

## 왜 필요한가?

멀티스레드 환경에서 가장 흔한 패턴: **읽고-수정하고-쓰기 (Read-Modify-Write)**

```
1. 변수 값을 읽는다       (Read)
2. 그 값을 1 증가시킨다    (Modify)
3. 다시 변수에 쓴다        (Write)
```

문제는 이게 **3개의 분리된 동작**이라는 거예요. 그 사이에 다른 스레드가 끼어들 수 있어요.

### 끔찍한 시나리오

`counter = 5`인 상태에서 두 스레드가 동시에 +1을 하려 함:

```
시간 ──→
스레드 A: [counter 읽음: 5] ─────────[+1 = 6 계산] ─────[counter에 6 씀]
스레드 B:        [counter 읽음: 5] ─────────[+1 = 6 계산] ─────[counter에 6 씀]
```

기대 결과: `7` (5 + 1 + 1)
실제 결과: `6` ❌

이걸 **race condition**이라고 부르고, 한 번의 증가가 사라졌어요(lost update).

---

## 락(Lock)으로 푸는 전통적인 방법

```java
synchronized (this) {
    counter++;
}
```

- 한 번에 한 스레드만 들어감
- 안전하지만 **느림**:
  - 락 획득/해제 비용
  - 다른 스레드는 **대기 (blocking)**
  - 컨텍스트 스위칭 발생
  - 데드락 위험

수많은 스레드가 짧은 작업을 두고 경쟁하면 락은 병목이 됨.

---

## CAS의 동작 원리

CAS는 CPU가 직접 제공하는 **원자적 명령어**예요. 의사 코드로 보면:

```c
boolean compareAndSwap(memory, expected, newValue) {
    if (memory == expected) {
        memory = newValue;
        return true;       // 성공
    } else {
        return false;      // 실패 (아무것도 안 함)
    }
}
```

이 전체가 **CPU 명령어 한 방**으로 실행돼요. 절대 중간에 끼어들 수 없음.

### 실제 CPU 명령어
| 아키텍처 | 명령어 |
|---|---|
| x86, x64 | `CMPXCHG` (Compare and Exchange) |
| ARM | `LDREX` / `STREX` (Load/Store Exclusive) - 등가 동작 |
| RISC-V | `LR` / `SC` (Load Reserved / Store Conditional) |

---

## CAS로 카운터 증가시키기

```java
do {
    int current = counter;          // 1. 현재 값 읽기
    int next = current + 1;         // 2. 새 값 계산
} while (!CAS(counter, current, next));   // 3. 시도! 실패하면 다시 from 1
```

흐름:

1. 현재 값을 읽음 (예: 5)
2. 새 값 계산 (6)
3. CAS 시도: "현재 값이 5면 6으로 바꿔줘"
   - **성공**: 누구도 안 끼어들었다 → 끝
   - **실패**: 누가 먼저 바꿨다 → 처음부터 다시

### 시각화

```
스레드 A                          스레드 B
─────────                         ─────────
read counter = 5
                                  read counter = 5
                                  CAS(5, 6) → 성공! counter=6
CAS(5, 6) → 실패! (지금 6이라)
read counter = 6
CAS(6, 7) → 성공! counter=7
```

스레드 A는 한 번 실패했지만 **재시도**해서 결국 성공. 락 없이도 결과가 정확함 ✅

이 패턴을 **optimistic concurrency**(낙관적 동시성 제어)라고 불러요. "충돌은 드물 것"이라고 낙관적으로 가정하고, 실패하면 재시도.

---

## 락 vs CAS 비교

| 특성 | 락 (Pessimistic) | CAS (Optimistic) |
|---|---|---|
| **접근 방식** | 일단 막고 본다 | 일단 시도하고 실패하면 재시도 |
| **블로킹** | O (다른 스레드 대기) | X (재시도만) |
| **컨텍스트 스위치** | 발생 | 거의 없음 |
| **경쟁이 적을 때** | 비효율 | 매우 빠름 |
| **경쟁이 심할 때** | 안정적 | 재시도 폭증 가능 |
| **데드락** | 가능 | 불가능 |

---

## 자바에서의 CAS

`java.util.concurrent.atomic` 패키지의 `Atomic*` 클래스들이 내부적으로 CAS를 사용해요.

### `AtomicLong.incrementAndGet()` 내부 (단순화)

```java
public final long incrementAndGet() {
    long prev, next;
    do {
        prev = get();
        next = prev + 1;
    } while (!compareAndSet(prev, next));
    return next;
}
```

이게 우리가 코드에서 그냥 `nextSeq.getAndIncrement()`만 호출해도 안전한 이유예요. 내부에서 CAS 루프가 돌아가고 있음.

### `Unsafe`와 `VarHandle`
JDK 내부에서는 `sun.misc.Unsafe`나 자바 9부터 도입된 `VarHandle`을 통해 CPU의 CAS 명령어를 호출해요. JVM이 이걸 네이티브 명령어로 컴파일해서 실행.

---

## ABA 문제

CAS의 **유명한 함정**이에요.

### 시나리오

```
초기 상태: A
스레드 1: A를 읽음
스레드 2: A → B로 바꿈
스레드 2: B → A로 다시 바꿈
스레드 1: CAS(A, C) → 성공! (값이 A니까)
```

스레드 1 입장에서는 **"내가 처음 읽은 후 아무 일도 없었다"**고 착각해요. 하지만 실제로는 두 번이나 바뀌었음.

### 언제 문제가 되는가?

단순한 카운터에선 ABA가 문제 안 됨 (A→B→A로 돌아왔으면 결과는 같음).

문제는 **참조 타입**이나 **포인터** 다룰 때:
- A 노드를 가리키는 포인터를 읽음
- 다른 스레드가 A를 제거하고, 메모리를 재활용해서 같은 주소에 A'를 만듦
- 원래 스레드가 CAS 성공 → 잘못된 객체를 조작!

### 해결법: 버전 번호 (Stamped Reference)

값과 함께 **버전(또는 stamp)을 같이 비교**해요.

자바의 `AtomicStampedReference`:

```java
AtomicStampedReference<Node> ref = new AtomicStampedReference<>(node, 0);

// 값 + 스탬프 둘 다 일치해야 성공
ref.compareAndSet(expectedRef, newRef, expectedStamp, newStamp);
```

값이 A→B→A로 돌아왔어도 스탬프가 다르면 CAS는 실패. ABA 회피.

---

## CAS의 약점

### 1. Spinning (busy-wait)
실패 시 재시도하면서 CPU를 계속 씀. 경쟁이 심하면 **CPU만 태우고 진행이 안 됨**.

### 2. Livelock
스레드들이 서로 양보하면서 계속 재시도만 하는 상태. 데드락은 아니지만 진전이 없음.

### 3. Starvation (기아)
운 나쁜 스레드는 매번 CAS 실패해서 영영 못 끝낼 수 있음.

### 4. 단일 변수에만 적용
"카운터 두 개를 동시에 원자적으로 증가" 같은 건 단일 CAS로 못 함. **DCAS(Double CAS)** 같은 변형이 있지만 하드웨어 지원이 제한적.

---

## CAS가 쓰이는 곳

CAS는 **lock-free 자료구조**의 기반이에요.

### 자바 표준 라이브러리
- `AtomicInteger`, `AtomicLong`, `AtomicReference`
- `ConcurrentHashMap` (내부 버킷 업데이트)
- `ConcurrentLinkedQueue`, `ConcurrentLinkedDeque`
- `LongAdder` (CAS 기반 분산 카운터)
- `StampedLock`

### 데이터베이스
- **Optimistic Locking**: 행에 version 컬럼 두고, UPDATE 시 version 같으면 +1, 다르면 실패. 개념적으로 CAS와 동일.

```sql
UPDATE users SET name='Alice', version=version+1
WHERE id=1 AND version=5;   -- WHERE 조건이 실패하면 0 rows affected
```

### 분산 시스템
- **etcd/ZooKeeper**: 키-값 업데이트에 "이전 값이 X일 때만 Y로 바꿔라" 식의 CAS API 제공.
- **HTTP**: `If-Match` 헤더를 이용한 ETag 기반 동시성 제어도 본질적으로 CAS.

---

## 정리

| 항목 | 내용 |
|---|---|
| **본질** | "값이 X면 Y로 바꿔라"를 원자적으로 |
| **CPU 명령어** | x86의 `CMPXCHG` 등 |
| **장점** | Lock-free, 블로킹 없음, 데드락 없음 |
| **단점** | 경쟁 심하면 재시도 폭증, ABA 문제 |
| **대표 예** | 자바 `Atomic*`, DB optimistic locking |

### 핵심 한 줄

> **"락은 못 들어오게 막고, CAS는 충돌하면 다시 시도한다."**

---

## 더 깊이 들어가려면

- **Lock-free vs Wait-free**: lock-free는 "전체 시스템은 진전"하지만, wait-free는 "모든 스레드가 진전"을 보장. CAS 기반은 보통 lock-free.
- **Memory Ordering / Memory Barrier**: CAS 명령어는 보통 메모리 배리어 효과도 가짐. CPU 캐시 일관성과 연결되는 깊은 주제.
- **Hazard Pointers, RCU**: ABA 문제와 메모리 회수를 함께 다루는 더 정교한 기법들.
