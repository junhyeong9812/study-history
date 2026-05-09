# 22 — Template Method

> **분류**: 행위 패턴 (Behavioral)
> **한 줄**: 알고리즘의 *뼈대* 는 부모, *세부* 는 자식이 구현.

---

## 1. 의도 (Intent)

> "Define the skeleton of an algorithm in an operation, deferring some steps to subclasses. Template Method lets subclasses redefine certain steps of an algorithm without changing the algorithm's structure."

같은 *순서/구조* 의 알고리즘이 세부만 다를 때.

## 2. 문제 상황 (Motivation)

데이터 처리 파이프라인 — CSV 도 JSON 도 흐름은 같음 :

```
1. 파일 열기
2. 파싱
3. 검증
4. DB 저장
5. 파일 닫기
```

```java
// ❌ 두 클래스 거의 동일
class CsvImporter {
    public void importData() {
        openFile();
        var rows = parseCsv();
        validate(rows);
        save(rows);
        closeFile();
    }
    private List<Row> parseCsv() { ... }
}

class JsonImporter {
    public void importData() {
        openFile();
        var rows = parseJson();
        validate(rows);
        save(rows);
        closeFile();
    }
    private List<Row> parseJson() { ... }
}
```

`importData` 의 흐름은 *완전히 같다*. 차이는 `parseCsv` vs `parseJson` 한 단계.

해결 : 부모 클래스가 흐름 정의, 자식이 변하는 단계만 구현.

## 3. 구조 (Structure)

```
AbstractClass
   ┌──────────────────────────┐
   │ + final templateMethod() │ ← 알고리즘 뼈대 (final)
   │                          │
   │ + step1()                │ ← 공통 (또는 default)
   │ # protected step2() abstr│ ← 자식이 구현
   │ + step3()                │
   │ # protected hook()       │ ← 선택적 (default empty)
   └──────────┬───────────────┘
              │
       ConcreteClass
          step2()
          hook() (선택)
```

## 4. 참여자 (Participants)

- **AbstractClass**: 템플릿 메서드 + 추상 단계 + (선택) hook.
- **ConcreteClass**: 추상 단계 구현.

## 5. Java 예제

### 5.1 정통

```java
import java.util.List;

abstract class DataImporter {

    // 템플릿 메서드 — final 로 막아 자식이 흐름 못 바꾸게
    public final void importData() {
        openFile();
        List<String> rows = parseFile();    // ← 추상
        validate(rows);
        if (shouldDeduplicate()) {           // ← hook
            rows = deduplicate(rows);
        }
        save(rows);
        closeFile();
    }

    // 공통 단계
    private void openFile() { System.out.println("open"); }
    private void closeFile() { System.out.println("close"); }
    private void validate(List<String> rows) { System.out.println("validate"); }
    private List<String> deduplicate(List<String> rows) {
        return rows.stream().distinct().toList();
    }
    private void save(List<String> rows) { System.out.println("save " + rows.size()); }

    // 추상 단계 — 자식이 구현 필수
    protected abstract List<String> parseFile();

    // hook — 자식이 선택적으로 오버라이드. default false.
    protected boolean shouldDeduplicate() { return false; }
}

class CsvImporter extends DataImporter {
    @Override protected List<String> parseFile() {
        System.out.println("parsing CSV");
        return List.of("a,1", "b,2", "a,1");
    }
    @Override protected boolean shouldDeduplicate() { return true; }    // hook 활성
}

class JsonImporter extends DataImporter {
    @Override protected List<String> parseFile() {
        System.out.println("parsing JSON");
        return List.of("{a:1}", "{b:2}");
    }
    // shouldDeduplicate 는 default (false) 사용
}

// 사용
public class Main {
    public static void main(String[] args) {
        new CsvImporter().importData();
        // open / parsing CSV / validate / save 2 / close

        new JsonImporter().importData();
        // open / parsing JSON / validate / save 2 / close
    }
}
```

`importData()` 가 흐름의 *유일한 정의*. 자식은 `parseFile` 만 의무, `shouldDeduplicate` 는 선택.

### 5.2 인터페이스 + default method (Java 8+)

```java
interface DataImporter {
    default void importData() {
        openFile();
        List<String> rows = parseFile();
        validate(rows);
        save(rows);
        closeFile();
    }

    List<String> parseFile();    // 필수

    default void openFile() { System.out.println("open"); }
    default void closeFile() { System.out.println("close"); }
    default void validate(List<String> rows) { /* default */ }
    default void save(List<String> rows) { /* default */ }
}

class CsvImporter implements DataImporter {
    @Override public List<String> parseFile() { return ...; }
}
```

추상 클래스 대신 인터페이스 + default. 다중 구현 가능.

## 6. 변형 (Variants)

### 6.1 Hook 메서드

```java
abstract class Game {
    public final void play() {
        initialize();
        startPlay();
        if (isMultiplayer()) waitForPlayers();   // hook
        endPlay();
    }
    protected abstract void initialize();
    protected abstract void startPlay();
    protected abstract void endPlay();
    protected boolean isMultiplayer() { return false; }    // hook default
    protected void waitForPlayers() {}
}
```

자식이 *필요할 때만* 오버라이드.

### 6.2 Strategy (21) 와 결합
Template Method 의 추상 단계를 *Strategy 인터페이스* 로 받기 — 컴포지션.

```java
class DataImporter {
    private final Parser parser;       // ← Strategy
    public DataImporter(Parser p) { this.parser = p; }

    public void importData() {
        openFile();
        var rows = parser.parse();
        ...
    }
}
```

상속 대신 컴포지션. 더 유연.

## 7. 함정 / 흔한 오해

### 7.1 자식이 흐름을 바꾸려 함
templateMethod 를 자식이 오버라이드 → 흐름 깨짐. `final` 로 막음.

### 7.2 추상 단계 너무 많음
모든 단계를 abstract 로 → 자식이 5+ 메서드 구현. 부담. 공통 default 와 정말 변하는 것만 추상.

### 7.3 Strategy (21) 와의 차이
- **Template Method**: *상속* 으로 알고리즘 단계 교체. 컴파일 타임.
- **Strategy**: *컴포지션* 으로 알고리즘 객체 교체. 런타임.
- 모던 Java 는 Strategy 선호 (compose over inherit).

### 7.4 Liskov 위반
자식이 부모 계약을 깨뜨릴 가능성. `final` 사용 + 사전 / 사후 조건 명시.

### 7.5 깊은 상속 계층
Template Method × 다중 상속 = 디버깅 지옥. *얇게* 유지.

## 8. 관련 패턴

- **Strategy (21)**: 컴포지션 대안.
- **Factory Method (03)**: Template Method 의 한 단계가 흔히 Factory Method.
- **Hooks**: Template Method 의 선택 단계.

## 9. 실무 사례

- `java.io.InputStream.read()` — 일부 자식 클래스가 구현
- `java.util.AbstractList` — `get`, `size` 만 구현하면 나머지 자동
- `HttpServlet.service()` → `doGet`, `doPost` 위임
- Spring `JdbcTemplate.execute()` — RowMapper 만 자식 구현
- JUnit `TestCase.runTest()` — 자식이 테스트 메서드 구현
- AWS Lambda handler (입력 파싱 / 출력 직렬화는 framework, 비즈니스만 자식)
- Hibernate `EmptyInterceptor` — hook 메서드 가득

```java
// Spring JdbcTemplate
List<User> users = jdbcTemplate.query(
    "SELECT * FROM users WHERE active = ?",
    new Object[]{true},
    (rs, rowNum) -> new User(rs.getLong("id"), rs.getString("name"))   // ← 자식의 변하는 부분
);
// JdbcTemplate.query 가 connection / statement / resultSet / close 처리 (template)
// 람다가 row → User 매핑 (자식의 일)
```

> Hexagonal 의 application service 에서 *흐름은 부모, 도메인 디테일은 도메인 객체* 분리도 일종의 Template Method.
