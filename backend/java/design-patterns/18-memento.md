# 18 — Memento

> **분류**: 행위 패턴 (Behavioral)
> **한 줄**: 객체 상태를 *외부* 에 저장하여 나중에 복원 가능. 캡슐화는 깨지지 않는다.

---

## 1. 의도 (Intent)

> "Without violating encapsulation, capture and externalize an object's internal state so that the object can be restored to this state later."

undo / redo / 체크포인트 / 트랜잭션 롤백 등에 쓰임.

## 2. 문제 상황 (Motivation)

텍스트 에디터의 undo. 매 편집마다 *상태 스냅샷* 저장하고, undo 시 복원.

```java
// ❌ 단순 — 모든 필드 public 으로 노출
class Editor {
    public String text;       // ← 외부가 직접 접근
    public int cursorPos;
}
class Snapshot {
    String text;
    int cursorPos;
    Snapshot(Editor e) { text = e.text; cursorPos = e.cursorPos; }
    void restore(Editor e) { e.text = text; e.cursorPos = cursorPos; }
}
```

`Editor` 의 캡슐화 (private 필드) 를 깨뜨리지 않으면서 어떻게 상태를 외부에 저장할까?

## 3. 구조 (Structure)

```
Originator (저장 대상)
   ┌────────────────────┐
   │ - state            │
   │ + save() → Memento │ ← 자기 상태를 Memento 로
   │ + restore(Memento) │ ← Memento 에서 자기 상태 복원
   └────────────────────┘
            │ creates
            ▼
        Memento (불변)
       - state (private)

Caretaker (보관소)
   ┌──────────────┐
   │ history: List<Memento>
   └──────────────┘
```

## 4. 참여자 (Participants)

- **Originator**: 저장 대상. `save()` / `restore()`.
- **Memento**: 상태 스냅샷. *Originator 만* 내부 접근.
- **Caretaker**: Memento 보관. *내용은 모름*.

## 5. Java 예제

### 5.1 정통 Memento

```java
import java.util.Stack;

class Editor {
    private String text = "";
    private int cursorPos = 0;

    public void type(String s) {
        text += s;
        cursorPos = text.length();
    }

    public void delete(int n) {
        text = text.substring(0, Math.max(0, text.length() - n));
        cursorPos = text.length();
    }

    public String getText() { return text; }

    // Memento 생성
    public Memento save() {
        return new Memento(text, cursorPos);
    }

    // Memento 로부터 복원
    public void restore(Memento m) {
        this.text = m.getText();
        this.cursorPos = m.getCursorPos();
    }

    // ★ Memento 가 inner class 라 Editor 의 private 필드 접근 가능
    public static class Memento {
        private final String text;
        private final int cursorPos;

        // package-private 또는 private — 외부 접근 차단
        private Memento(String text, int cursorPos) {
            this.text = text;
            this.cursorPos = cursorPos;
        }

        // Editor 만 사용
        private String getText() { return text; }
        private int getCursorPos() { return cursorPos; }
    }
}

// Caretaker
class History {
    private final Stack<Editor.Memento> stack = new Stack<>();

    public void save(Editor editor) {
        stack.push(editor.save());
    }

    public void undo(Editor editor) {
        if (!stack.isEmpty()) {
            editor.restore(stack.pop());
        }
    }
}

// 사용
public class Main {
    public static void main(String[] args) {
        Editor editor = new Editor();
        History history = new History();

        editor.type("Hello");
        history.save(editor);

        editor.type(" World");
        history.save(editor);

        editor.type("!!!");
        System.out.println(editor.getText());    // "Hello World!!!"

        history.undo(editor);
        System.out.println(editor.getText());    // "Hello World"

        history.undo(editor);
        System.out.println(editor.getText());    // "Hello"
    }
}
```

`Memento` 의 필드와 메서드는 `private`. `History` 는 Memento 를 *불투명한 토큰* 으로 다룸. `Editor` 만이 Memento 의 내용에 접근.

### 5.2 Serializable Memento

```java
import java.io.Serializable;

class GameState implements Serializable {
    private static final long serialVersionUID = 1L;
    private final int level;
    private final int score;
    private final List<String> inventory;
    // ... getter/setter
}

class Game {
    private GameState state;

    public byte[] saveToFile() throws IOException {
        var baos = new ByteArrayOutputStream();
        try (var oos = new ObjectOutputStream(baos)) {
            oos.writeObject(state);
        }
        return baos.toByteArray();
    }

    public void loadFromFile(byte[] data) throws IOException, ClassNotFoundException {
        var bais = new ByteArrayInputStream(data);
        try (var ois = new ObjectInputStream(bais)) {
            this.state = (GameState) ois.readObject();
        }
    }
}
```

직렬화로 디스크 저장 — 영속화된 memento.

## 6. 변형 (Variants)

### 6.1 Incremental Memento
전체 상태가 아니라 *변경분만* 저장. 메모리 절약. 단, 복원 시 chain 으로 적용.

### 6.2 Command + Memento
Command (14) 의 undo 구현에 Memento 사용 — execute 시 memento 저장, undo 시 restore.

### 6.3 Snapshot interface
```java
record EditorSnapshot(String text, int cursorPos) {}
```

`record` 가 단순 memento 에 깔끔.

## 7. 함정 / 흔한 오해

### 7.1 Memento 의 캡슐화
공개되면 Caretaker 가 Memento 내용 조작 → 의미 깨짐. *불변* + *제한된 접근*.

### 7.2 메모리 사용
모든 변경마다 전체 상태 저장 → OOM. 제한 (마지막 N 개) 또는 incremental.

### 7.3 객체 그래프 깊이
Originator 의 필드가 다른 객체 참조면 — 그것도 함께 복원? deep copy? reference?

### 7.4 외부 자원
파일 핸들 / DB connection / 네트워크 소켓은 *복원 불가*. memento 는 데이터만.

## 8. 관련 패턴

- **Command (14)**: undo 구현.
- **Iterator (16)**: iterator state 가 일종의 memento.
- **Prototype (04)**: 다른 종류의 상태 보존.

## 9. 실무 사례

- 텍스트 에디터 / IDE 의 undo 스택
- 게임 세이브 파일
- 데이터베이스 SAVEPOINT (트랜잭션 부분 롤백)
- VM snapshot (VirtualBox, VMware)
- Git 의 commit (사실상 memento)
- Java `Serializable` (직렬화 메커니즘)
- Redux time-travel debugger (state history)
- 워드프로세서의 자동 복구 파일

```java
// Java 의 Object Serialization 이 사실상 Memento
ObjectOutputStream oos = new ObjectOutputStream(...);
oos.writeObject(state);    // memento 저장
ObjectInputStream ois = new ObjectInputStream(...);
GameState restored = (GameState) ois.readObject();    // 복원
```

> 모던 시스템의 *event sourcing* 은 memento 의 진화 — 매 변경을 event 로 저장하면 임의 시점 재구성 가능.
