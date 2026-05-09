# 08 — Composite

> **분류**: 구조 패턴 (Structural)
> **한 줄**: 부분과 전체를 *동일한 인터페이스* 로 다루는 트리 구조.

---

## 1. 의도 (Intent)

> "Compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions of objects uniformly."

리프 (leaf) 와 그룹 (composite) 을 *같은 메서드로* 다룰 수 있게.

## 2. 문제 상황 (Motivation)

파일 시스템에 *파일* 과 *디렉토리* 가 있다. 둘 다 "크기" 와 "이름" 이 있고, "출력" 할 수 있다. 하지만 디렉토리는 *자식들의 합* 이 크기.

```java
// ❌ 분기 처리
void printAll(Object item, int depth) {
    if (item instanceof File f) {
        System.out.println(" ".repeat(depth) + f.name + " " + f.size);
    } else if (item instanceof Directory d) {
        System.out.println(" ".repeat(depth) + d.name);
        for (Object child : d.children) {
            printAll(child, depth + 1);    // 재귀
        }
    }
}
```

매번 instanceof 분기. 새 종류 (Symlink) 추가 시 모든 분기 수정.

해결 : 공통 인터페이스로 추상화.

## 3. 구조 (Structure)

```
Component (interface)
   ┌──────────────┐
   │ operation()  │
   │ add(Component)
   │ remove(Component)
   └──────┬───────┘
          │
   ┌──────┴───────┐
   │              │
  Leaf         Composite
operation()    operation() {
                 for c in children: c.operation()
               }
               + children: List<Component>
```

## 4. 참여자 (Participants)

- **Component**: 공통 인터페이스 (Leaf + Composite).
- **Leaf**: 자식 없음. 실제 일을 함.
- **Composite**: 자식들 보유. 자식들에게 위임.
- **Client**: Component 만 안다.

## 5. Java 예제

```java
import java.util.ArrayList;
import java.util.List;

// Component
interface FileSystemNode {
    String getName();
    long getSize();
    void print(int depth);
}

// Leaf
class FileNode implements FileSystemNode {
    private final String name;
    private final long size;

    public FileNode(String name, long size) {
        this.name = name;
        this.size = size;
    }

    @Override public String getName() { return name; }
    @Override public long getSize() { return size; }

    @Override public void print(int depth) {
        System.out.println(" ".repeat(depth) + name + " (" + size + " bytes)");
    }
}

// Composite
class DirectoryNode implements FileSystemNode {
    private final String name;
    private final List<FileSystemNode> children = new ArrayList<>();

    public DirectoryNode(String name) {
        this.name = name;
    }

    public void add(FileSystemNode node) {
        children.add(node);
    }

    public void remove(FileSystemNode node) {
        children.remove(node);
    }

    @Override public String getName() { return name; }

    @Override public long getSize() {
        // 자식들의 합 — 트리 순회
        return children.stream().mapToLong(FileSystemNode::getSize).sum();
    }

    @Override public void print(int depth) {
        System.out.println(" ".repeat(depth) + name + "/ (" + getSize() + " bytes total)");
        for (FileSystemNode child : children) {
            child.print(depth + 1);
        }
    }
}

// 사용
public class Main {
    public static void main(String[] args) {
        DirectoryNode root = new DirectoryNode("root");
        DirectoryNode docs = new DirectoryNode("docs");

        docs.add(new FileNode("hello.txt", 100));
        docs.add(new FileNode("report.pdf", 5000));

        DirectoryNode pics = new DirectoryNode("pics");
        pics.add(new FileNode("cat.jpg", 2000));

        root.add(docs);
        root.add(pics);
        root.add(new FileNode("readme.md", 50));

        root.print(0);
        // root/ (7150 bytes total)
        //  docs/ (5100 bytes total)
        //   hello.txt (100 bytes)
        //   report.pdf (5000 bytes)
        //  pics/ (2000 bytes total)
        //   cat.jpg (2000 bytes)
        //  readme.md (50 bytes)

        System.out.println("Total: " + root.getSize());    // 7150
    }
}
```

`Client` 는 `FileSystemNode` 만 안다. 트리든 리프든 똑같이 `print()` / `getSize()`.

## 6. 변형 (Variants)

### 6.1 Transparent Composite (GoF 권장)
- `add()`, `remove()` 가 `Component` 인터페이스에.
- 단점 : Leaf 도 `add` 가 노출됨 → `UnsupportedOperationException` 던질 위험.

### 6.2 Safe Composite (위 예제)
- `add()`, `remove()` 가 `Composite` 에만.
- 단점 : Leaf 와 Composite 다루기 *완전히 동일* 하지 않음 — 추가 시 캐스팅 필요.

→ Safe 가 더 일반적.

## 7. 함정 / 흔한 오해

### 7.1 부모 참조 누가 가지나
양방향 트리 (자식이 부모 참조) 가 필요한지. 단방향이면 단순. 양방향이면 add 시 부모 설정 / remove 시 해제.

### 7.2 순환 참조
A.add(B), B.add(A) → 무한 루프. 검사 필요.

### 7.3 Decorator (09) 와의 차이
- **Composite**: 부분-전체 *트리*.
- **Decorator**: 같은 인터페이스에 *기능 적층*.
- 코드는 비슷할 수 있지만 의도가 완전히 다름.

## 8. 관련 패턴

- **Decorator (09)**: 단일 자식 chain.
- **Iterator (16)**: Composite 트리 순회.
- **Visitor (23)**: Composite 트리에 다양한 연산 추가.
- **Chain of Responsibility (13)**: 부모를 따라 올라가는 요청 처리.

## 9. 실무 사례

- 파일 시스템 (디렉토리 + 파일)
- HTML/XML DOM (Element 가 children 보유)
- GUI: Container (Panel, Frame) ←→ Widget (Button, Label)
- Swing: `Component` 와 `Container`
- `java.util.AbstractMap` 의 entry view
- 조직도 / 메뉴 트리 / 카테고리 트리
- 표현식 트리 (math expression: `Plus`, `Times`, `Number`)
