# 16 — Iterator

> **분류**: 행위 패턴 (Behavioral)
> **한 줄**: 컬렉션의 *내부 구조를 노출하지 않고* 원소들을 순차 접근.

---

## 1. 의도 (Intent)

> "Provide a way to access the elements of an aggregate object sequentially without exposing its underlying representation."

컬렉션이 array 든 linked list 든 tree 든 — *같은 방식* 으로 순회.

## 2. 문제 상황 (Motivation)

```java
// ❌ 컬렉션 종류에 따라 다른 순회 코드
int[] arr = ...;
for (int i = 0; i < arr.length; i++) { ... arr[i] ... }

LinkedList<Integer> list = ...;
Node n = list.head;
while (n != null) { ... n.value ...; n = n.next; }

Tree tree = ...;
// 재귀? 스택?
```

컬렉션 사용자가 *내부 구조* 를 알아야 함. 새 컬렉션 추가 시 새 순회 방식.

해결 : 공통 인터페이스 `Iterator`.

```java
for (T item : collection) { ... }    // 무엇이든 순회 같음
```

## 3. 구조 (Structure)

```
Iterable                  Iterator
   ┌──────────┐           ┌──────────┐
   │iterator()│ ─returns─ ►│hasNext() │
   └────┬─────┘            │next()    │
        │                  └─────┬────┘
        │                        │
   ConcreteAggregate     ConcreteIterator
   (List, Tree, Set)     - aggregate
                         - position
```

## 4. 참여자 (Participants)

- **Iterator**: 순회 인터페이스 (`hasNext`, `next`).
- **ConcreteIterator**: 컬렉션별 순회 구현. 위치 추적.
- **Aggregate / Iterable**: `iterator()` 반환.
- **ConcreteAggregate**: 실제 컬렉션.

## 5. Java 예제

### 5.1 직접 구현

```java
// Iterator interface (Java 표준)
// public interface Iterator<E> { boolean hasNext(); E next(); }

class IntArray implements Iterable<Integer> {
    private final int[] data;
    public IntArray(int[] data) { this.data = data; }

    @Override
    public Iterator<Integer> iterator() {
        return new Iterator<>() {
            private int index = 0;

            @Override public boolean hasNext() {
                return index < data.length;
            }

            @Override public Integer next() {
                if (!hasNext()) throw new NoSuchElementException();
                return data[index++];
            }
        };
    }
}

// 사용 — for-each 자동 활용
public class Main {
    public static void main(String[] args) {
        IntArray arr = new IntArray(new int[]{1, 2, 3, 4, 5});

        for (int n : arr) {
            System.out.println(n);
        }

        // 또는 명시적
        Iterator<Integer> it = arr.iterator();
        while (it.hasNext()) System.out.println(it.next());
    }
}
```

### 5.2 Tree iterator (in-order)

```java
class BinaryTree<T> implements Iterable<T> {
    private final Node<T> root;

    static class Node<T> {
        T value;
        Node<T> left, right;
        Node(T v) { this.value = v; }
    }

    @Override
    public Iterator<T> iterator() {
        return new Iterator<>() {
            private final Stack<Node<T>> stack = new Stack<>();
            private Node<T> current = root;

            @Override public boolean hasNext() {
                return current != null || !stack.isEmpty();
            }

            @Override public T next() {
                while (current != null) {
                    stack.push(current);
                    current = current.left;
                }
                Node<T> node = stack.pop();
                T value = node.value;
                current = node.right;
                return value;
            }
        };
    }
}
```

→ 트리 순회의 복잡한 상태 (스택 / 현재 노드) 가 iterator 안에 *캡슐화*. 클라이언트는 for-each 만.

### 5.3 함수형 (Stream API)

```java
List<Integer> list = List.of(1, 2, 3, 4, 5);
list.stream()
    .filter(n -> n > 2)
    .map(n -> n * 10)
    .forEach(System.out::println);
// 30 40 50
```

`Stream` 은 iterator 의 진화 — 연산 체이닝 + lazy.

## 6. 변형 (Variants)

### 6.1 External vs Internal Iterator

- **External**: 클라이언트가 `next()` 호출 (Java `Iterator`).
- **Internal**: 컬렉션이 forEach 받기 (Stream, Smalltalk `do:`).

```java
// External
Iterator<T> it = list.iterator();
while (it.hasNext()) handle(it.next());

// Internal
list.forEach(this::handle);
```

### 6.2 Robust Iterator
순회 중 컬렉션 변경 시 동작 :
- **Fail-fast**: ConcurrentModificationException (Java `ArrayList`).
- **Snapshot**: 시작 시 스냅샷 (CopyOnWriteArrayList).
- **Weakly consistent**: 변경 일부 반영 (ConcurrentHashMap).

### 6.3 Generator (yield)
```java
// Java 에 직접 yield 는 없지만 — Stream / Spliterator 로 효과 가능
Stream.iterate(0, i -> i + 1).limit(10).forEach(...);
```

Python 의 `yield`, JS 의 `function*`, C# 의 `yield return` 이 generator 형태.

## 7. 함정 / 흔한 오해

### 7.1 ConcurrentModification

```java
List<Integer> list = new ArrayList<>(List.of(1,2,3));
for (int n : list) {
    if (n == 2) list.remove(Integer.valueOf(2));   // ❌ Exception
}
```

순회 중 컬렉션 수정 → Java 의 fail-fast iterator 가 던짐. `iterator.remove()` 또는 `list.removeIf(...)` 사용.

### 7.2 Iterator 재사용 불가
대부분의 Iterator 는 *한 번만* 순회. 다시 처음부터 보려면 `iterator()` 다시 호출.

### 7.3 next() 가 hasNext() 보다 먼저
```java
it.next();    // hasNext 안 부르면 NoSuchElementException 위험
```

### 7.4 무한 iterator
`Stream.iterate(0, i -> i + 1)` 는 무한. `limit()` 없으면 무한 루프.

## 8. 관련 패턴

- **Composite (08)**: 트리 순회.
- **Factory Method (03)**: `iterator()` 가 factory method.
- **Visitor (23)**: 순회 + 노드별 처리.
- **Memento (18)**: iterator state 저장.

## 9. 실무 사례

- `java.util.Iterator` / `Iterable` — 모든 컬렉션
- `java.util.stream.Stream`
- `java.sql.ResultSet` — DB 결과 순회
- `Scanner` / `BufferedReader.lines()`
- Python `iter()` / `__iter__` / `__next__`
- JS `Symbol.iterator` / `for...of`
- DOM `NodeList.forEach`
- 데이터베이스 cursor (page 단위 가져오기)

```java
// Java 14+ pattern
List.of(1, 2, 3, 4).stream()
    .map(n -> n * n)
    .filter(n -> n > 5)
    .forEach(System.out::println);
```

→ Iterator 는 이제 *전제* 가 되었고, 모든 모던 컬렉션 API 가 이 위에 서있다.
