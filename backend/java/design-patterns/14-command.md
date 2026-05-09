# 14 — Command

> **분류**: 행위 패턴 (Behavioral)
> **한 줄**: 요청을 *객체* 로 캡슐화. 큐, undo, 로그가 가능해진다.

---

## 1. 의도 (Intent)

> "Encapsulate a request as an object, thereby letting you parameterize clients with different requests, queue or log requests, and support undoable operations."

행동 자체를 *값처럼* 다루기 — 저장하고, 미루고, 큐에 넣고, 되돌리기.

## 2. 문제 상황 (Motivation)

GUI 의 메뉴 / 버튼 — 각 항목마다 다른 동작. 그리고 *되돌리기* (undo) 도 지원해야 한다.

```java
// ❌ 버튼이 동작을 직접
class CopyButton {
    public void onClick() {
        Editor.copy();
    }
}
class PasteButton {
    public void onClick() {
        Editor.paste();
    }
}
// → 버튼마다 클래스. undo 추가 시 모든 버튼 수정.
```

해결 : 동작을 *Command 객체* 로 캡슐화.

```java
Button copyBtn = new Button(new CopyCommand(editor));
Button pasteBtn = new Button(new PasteCommand(editor));
// undo: 마지막 command 의 undo() 호출
```

## 3. 구조 (Structure)

```
Client     Invoker        Command (interface)        Receiver
  │           │            ┌──────────┐
  └─creates──►              │ execute()│
                            │ undo()   │
                            └────┬─────┘
                                 │
                          ConcreteCommand
                          - receiver: Receiver
                          - state
                          + execute() → receiver.action()
                          + undo()
```

## 4. 참여자 (Participants)

- **Command**: execute() 인터페이스.
- **ConcreteCommand**: Receiver 와 매개변수 보유. execute() 가 receiver 호출.
- **Receiver**: 실제 일.
- **Invoker**: Command 를 *언제* 실행할지 결정.
- **Client**: Command 인스턴스 생성.

## 5. Java 예제

### 5.1 텍스트 에디터 + Undo

```java
import java.util.Stack;

// Receiver
class TextEditor {
    private StringBuilder text = new StringBuilder();

    public void append(String s) { text.append(s); }
    public void delete(int n) { text.delete(text.length() - n, text.length()); }
    public String getText() { return text.toString(); }
}

// Command interface
interface Command {
    void execute();
    void undo();
}

// Concrete Commands
class AppendCommand implements Command {
    private final TextEditor editor;
    private final String text;

    public AppendCommand(TextEditor editor, String text) {
        this.editor = editor;
        this.text = text;
    }

    @Override public void execute() { editor.append(text); }
    @Override public void undo() { editor.delete(text.length()); }
}

class DeleteCommand implements Command {
    private final TextEditor editor;
    private final int count;
    private String backup;     // 저장하여 undo 시 복원

    public DeleteCommand(TextEditor editor, int count) {
        this.editor = editor;
        this.count = count;
    }

    @Override public void execute() {
        String t = editor.getText();
        backup = t.substring(t.length() - count);
        editor.delete(count);
    }
    @Override public void undo() {
        editor.append(backup);
    }
}

// Invoker
class CommandHistory {
    private final Stack<Command> history = new Stack<>();

    public void execute(Command cmd) {
        cmd.execute();
        history.push(cmd);
    }

    public void undo() {
        if (!history.isEmpty()) {
            history.pop().undo();
        }
    }
}

// Client
public class Main {
    public static void main(String[] args) {
        TextEditor editor = new TextEditor();
        CommandHistory history = new CommandHistory();

        history.execute(new AppendCommand(editor, "Hello "));
        history.execute(new AppendCommand(editor, "World"));
        System.out.println(editor.getText());    // "Hello World"

        history.undo();
        System.out.println(editor.getText());    // "Hello "

        history.undo();
        System.out.println(editor.getText());    // ""
    }
}
```

### 5.2 함수형 단순화 (Java 8+)

```java
interface Command {
    void execute();
}

// 람다로 즉시
Command cmd = () -> editor.append("Hello");
cmd.execute();

// Runnable 도 사실 Command
Runnable r = () -> System.out.println("hi");
r.run();
```

undo 가 없으면 람다로 충분. undo 가 있으면 클래스 형태가 자연스러움.

## 6. 변형 (Variants)

### 6.1 Macro Command (Composite Command)

```java
class MacroCommand implements Command {
    private final List<Command> commands;
    public MacroCommand(List<Command> cs) { this.commands = cs; }
    @Override public void execute() { commands.forEach(Command::execute); }
    @Override public void undo() {
        // 역순으로 undo
        for (int i = commands.size() - 1; i >= 0; i--) commands.get(i).undo();
    }
}
```

여러 Command 를 하나로 — Composite (08) 와의 결합.

### 6.2 Queued Command
```java
ExecutorService exec = Executors.newSingleThreadExecutor();
exec.submit(cmd::execute);    // 백그라운드 실행
```

### 6.3 Persistent Command
Command 를 직렬화 → DB 또는 큐에 저장 → 나중에 실행. *Event Sourcing* 의 기반.

## 7. 함정 / 흔한 오해

### 7.1 Undo 가 항상 가능한 건 아님
파일 삭제, 외부 API 호출 등은 *기술적으로 되돌릴 수 없음*. 비즈니스 의미상의 *보상* 필요 (Saga).

### 7.2 Command 객체의 비대화
한 Command 에 너무 많은 상태 — 직렬화 어려워짐, 메모리 부담. Command 는 *얇게*.

### 7.3 Strategy (21) 와의 차이
- **Strategy**: 알고리즘 *선택*. 같은 입력에 다른 알고리즘.
- **Command**: 행동 *저장*. 나중에 실행 / undo / 큐.
- 코드는 비슷할 수 있음 — 의도가 다름.

### 7.4 Receiver 누락
ConcreteCommand 가 receiver 없이 *직접* 일 → Receiver 와 Command 가 같은 클래스. 단순한 경우 OK 지만 분리의 이점 잃음.

## 8. 관련 패턴

- **Composite (08)**: Macro Command.
- **Memento (18)**: undo 구현에 필요한 스냅샷.
- **Chain of Responsibility (13)**: handler 가 Command 처리.
- **Strategy (21)**: 비슷한 코드.

## 9. 실무 사례

- `java.lang.Runnable` — 가장 단순한 Command
- `Thread(Runnable)` 의 Runnable
- `ExecutorService.submit(Callable)`
- Swing `Action` / GUI 메뉴 항목
- 데이터베이스 transaction (BEGIN ... COMMIT/ROLLBACK)
- Git commit (각 commit 이 거의 Command)
- Redux action / dispatch (JS)
- Event Sourcing 의 event 가 사실상 Command 의 영속화

### 큐와 비동기

```java
BlockingQueue<Command> queue = new LinkedBlockingQueue<>();

// Producer
queue.put(new AppendCommand(editor, "hi"));

// Consumer (다른 스레드)
while (true) {
    Command cmd = queue.take();
    cmd.execute();
}
```

→ 분산 작업 큐의 본질.
