# 15 — Interpreter

> **분류**: 행위 패턴 (Behavioral)
> **한 줄**: 작은 *언어* 의 문법을 클래스 트리로 표현하고, 그 트리를 해석.

---

## 1. 의도 (Intent)

> "Given a language, define a representation for its grammar along with an interpreter that uses the representation to interpret sentences in the language."

도메인 특화 언어 (DSL) 의 표현식을 *AST 트리* 로 만들고 평가.

## 2. 문제 상황 (Motivation)

문자열 표현식 `"5 + 3 * 2"` 를 계산하고 싶다. 단순 if/else 로는 우선순위 / 괄호 / 변수 처리가 복잡.

```java
// ❌ 정규식 / split 으로 풀기
String[] tokens = "5 + 3 * 2".split(" ");
// ... 우선순위는? 변수는? 함수는?
```

해결 : 문법 요소를 *클래스* 로 만들고 트리 구성.

```
       Plus
      /    \
     5     Times
           /  \
          3    2
```

각 노드가 `evaluate()` 를 가지면 트리 평가 = 식 계산.

## 3. 구조 (Structure)

```
AbstractExpression
   ┌──────────────┐
   │ interpret(ctx)│
   └──────┬───────┘
          │
   ┌──────┴───────┬──────────────┐
   │              │              │
TerminalExpr  NonterminalExpr  Variable...
(Number)      (Plus, Times)
              - left, right: Expr
              + interpret(ctx) {
                  return left.interpret(ctx) op right.interpret(ctx)
                }
```

## 4. 참여자 (Participants)

- **AbstractExpression**: 인터페이스. interpret(context) 메서드.
- **TerminalExpression**: 리프 노드 (literal, variable).
- **NonterminalExpression**: 내부 노드 (연산).
- **Context**: 변수 환경 등.

## 5. Java 예제

### 5.1 단순 산술 인터프리터

```java
import java.util.HashMap;
import java.util.Map;

// AbstractExpression
interface Expression {
    int interpret(Map<String, Integer> context);
}

// Terminal — 숫자
class Number implements Expression {
    private final int value;
    public Number(int v) { this.value = v; }
    @Override public int interpret(Map<String, Integer> ctx) { return value; }
}

// Terminal — 변수
class Variable implements Expression {
    private final String name;
    public Variable(String name) { this.name = name; }
    @Override public int interpret(Map<String, Integer> ctx) {
        return ctx.getOrDefault(name, 0);
    }
}

// Nonterminal — 덧셈
class Plus implements Expression {
    private final Expression left, right;
    public Plus(Expression l, Expression r) { left = l; right = r; }
    @Override public int interpret(Map<String, Integer> ctx) {
        return left.interpret(ctx) + right.interpret(ctx);
    }
}

class Times implements Expression {
    private final Expression left, right;
    public Times(Expression l, Expression r) { left = l; right = r; }
    @Override public int interpret(Map<String, Integer> ctx) {
        return left.interpret(ctx) * right.interpret(ctx);
    }
}

// 사용 — "5 + (x * 2)"
public class Main {
    public static void main(String[] args) {
        Expression expr = new Plus(
            new Number(5),
            new Times(new Variable("x"), new Number(2))
        );

        Map<String, Integer> ctx = new HashMap<>();
        ctx.put("x", 3);

        System.out.println(expr.interpret(ctx));   // 5 + 3*2 = 11
    }
}
```

### 5.2 Boolean 표현식 (RBAC 권한 등)

```java
interface BoolExpr {
    boolean evaluate(Map<String, Boolean> ctx);
}

class Constant implements BoolExpr {
    private final boolean v;
    public Constant(boolean v) { this.v = v; }
    @Override public boolean evaluate(Map<String, Boolean> c) { return v; }
}

class Var implements BoolExpr {
    private final String name;
    public Var(String name) { this.name = name; }
    @Override public boolean evaluate(Map<String, Boolean> c) {
        return c.getOrDefault(name, false);
    }
}

class And implements BoolExpr {
    private final BoolExpr l, r;
    public And(BoolExpr l, BoolExpr r) { this.l = l; this.r = r; }
    @Override public boolean evaluate(Map<String, Boolean> c) {
        return l.evaluate(c) && r.evaluate(c);
    }
}

class Or implements BoolExpr {
    private final BoolExpr l, r;
    public Or(BoolExpr l, BoolExpr r) { this.l = l; this.r = r; }
    @Override public boolean evaluate(Map<String, Boolean> c) {
        return l.evaluate(c) || r.evaluate(c);
    }
}

// 사용: (admin AND active) OR superuser
BoolExpr policy = new Or(
    new And(new Var("admin"), new Var("active")),
    new Var("superuser")
);
boolean allow = policy.evaluate(Map.of("admin", true, "active", true, "superuser", false));
```

## 6. 변형 (Variants)

### 6.1 Visitor (23) 와 결합
연산 (evaluate, print, optimize) 이 다양해지면 Visitor 로 분리.

### 6.2 파서 분리
Interpreter 는 *AST 평가* 만. 문자열 → AST 변환은 *파서* (parser combinator, ANTLR, JavaCC).

### 6.3 Composite (08) 와의 친척
AST 자체가 Composite. Interpreter 는 Composite 트리 + interpret 연산.

## 7. 함정 / 흔한 오해

### 7.1 문법이 복잡해지면 클래스 폭발
Plus, Minus, Times, Divide, Modulo, Power, ... 각 연산마다 클래스. 30+ 면 다른 도구 검토.

→ 본격 언어는 ANTLR / yacc / parser combinator 같은 *생성기* 사용. Interpreter 는 작은 DSL 에만.

### 7.2 성능
트리 순회는 컴파일된 코드보다 느림. 자주 평가하는 표현식은 컴파일 또는 캐시.

### 7.3 직렬화
AST 직렬화 / 역직렬화 — 각 노드 타입마다 처리.

## 8. 관련 패턴

- **Composite (08)**: AST 자체.
- **Visitor (23)**: AST 의 다양한 연산.
- **Iterator (16)**: AST 순회.
- **Flyweight (11)**: 같은 literal 공유.

## 9. 실무 사례

- 정규표현식 엔진 (Pattern → AST → match)
- SQL 파서 (예 : H2, Calcite 의 RelNode)
- Spring Expression Language (SpEL): `#{user.name}`
- JSP / Thymeleaf 표현식
- 게임의 quest condition DSL
- 검색 엔진의 query parser (Lucene QueryParser)
- Math 표현식 평가기 (`mvel`, `aviator`)
- Drools 같은 rule engine 의 LHS

> 실무에서는 직접 Interpreter 짜는 일이 적다. 대부분 ANTLR / parser library 가 토큰화 + 파싱 + AST 생성을 자동.
> *Interpreter 패턴* 이라는 말은 보통 "AST 의 각 노드가 evaluate 메서드 가짐" 을 가리킴.
