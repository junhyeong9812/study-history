# Java Record 타입
java 14에서 프리뷰로 도입되고 Java 16에서 정식 확정된 Record는 데이터를 담기 위한 불변(Immutable) 클래스를 간결하게 선언하는
방법이다.

## 기본 문법
```commandline
public record Point(int x, int y) {}
```
이 한줄이 아래와 동일한 효과를 가진다.
```commandline
public final class Point {
    private final int x;
    private final int y;
    
    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
    
    public int x() { return x; }
    public int y() { return y; }
    
    @Override public boolean equals(Object o) {...}
    @Override public int hashCode() {...}
    @Override public String toString() {...}
}
```
즉, Record를 선언하면 컴파일러가 final 필드, 생성자, 접근자(getter), equals, hashCode, toString을 자동으로 생성한다.

컴파일러가 해주는 일을 코드로 정리해보자.
```commandline
import java.lang.invoke.*;
import java.util.Objects;

/**
 * record Point(int x, int y) {}를 컴파일러가 펼치면 이렇게 된다.
*/
public final class Point extends Record {
    // --1. 컴포넌트 필드 (private final)
    private final int x;
    private final int y;
    
    // --2. 정규 생성자 (Canonical Constructor)
    // 컴팩트 생성자를 썻다면 그 본문이 this.xx = xx; 앞에 삽입된다.
    public Point(int x, int y) {
        // -- 컴팩트 생성자 본문이 여기 들어온다.
        // 예: if ( x < 0 ) throw new IllegalArgumentException();
        // 예: x = Math,abs(x); 파라미터 재할당 가능
        
        // 컴파일러가 자동 삽입하는 필드 할당
        this.x = x;
        this.y = y;
    }
    
    // --3. 접근자 (getter) -getX()가 아니라 X()이다.
    public int x() {return this.x;}
    public int y() {return this.y;}
    
    // --4.toString
    // 실제로는 invokedynamic+ ObjectMethods.bootstrap으로 동적 생성되지만
    // 동적을 풀어쓴다면
    @Override
    public String toString() {
        return "Point[x=" + x + ", y=" + y + "]";
    }
    
    // --5. equals
    // 같은 타입이고, 모든 컴포넌트가 같으면 true
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Point other)) return false;
        return this.x == other.x && this.y == other.y;
    }
    
    // --6. hashCode
    // 모든 컴포넌트로 해시 계산
    @Override
    public int hashCode() {
        return Objects.hash(x, y);
    }
```

## 주요 특징
1. 불변성 - 모든 컴포넌트 필드가 private final이므로 한번 생성하면 값을 바꿀 수 없다. setter가 존재하지 않는다.
2. 투명한 캐리어 - Record의 핵심 철학은 "데이터의 투명한 운반체"이다. 상태를 숨기는 것이 아니라 선언한 그대로 노출하는게 목적
3. 상속 불가 - Record는 암묵적으로 java.lang.Record를 상속하며 final이므로 다른 클래스를 상속하거나, 다른 클래스가 Record를 상속할 수 없다. 다만 인터페이스 구현은 가능하다.

## 커스텀 생성자
컴팩트 생성자(compact constructor)를 사용하면 유효성 검증을 깔끔하게 추가할 수 있다:
```commandline
public record Range(int lo, int hi) {
    public Range {
        if (lo > hi) throw new IllegalArgumentException("lo > hi");
        // 별도의 this.lo = lo; 같은 할당이 필요없다.
    }
}
```
일반 생성자 형태도 가능하지만 컴팩트 생성자가 더 관용적이다.

## 커스텀 메서드와 접근자
자동 생성된 메서드를 오버라이드하거나 추가 메서드를 정의할 수 있다.
```commandline
public record Person(String firstName, String lastName) {
    public String fullName() {
        return firstName + "" + lastName;
    }
    
    @Override
    public String toString() {
        return fullName();
    }
}


```

## 활용 예시
``` commandline
// DTO / 값 객체
public record UserDto(long id, String name, String email) {}

// 복합 키
public record CacheKey(String region, long userId) {}

// sealed interface와 조합 (패턴 매칭에 유리)
sealed interface Shape permits Circle, Rectangle {}
record Circle(double radius) implements Shape {}
record Rectangle(double width, double height) implements Shape {}
```
특히 마지막 예시처럼 sealed interface + record + 패턴 매칭을 조합하면 대수적 데이터 타입을 (ADT)에 가까운 표현이 가능해져,
Switch 식에서 타입별 분기를 깔끔하게 처리할 수 있다.

### 마지막 방식을 코드 구조로 본다면?
```commandline
sealed interface Shape permits Circle, Rectangle { }
record Circle(double radius) implements Shape { }
record Rectangle(double width, double height) implements Shape { }

// switch  패턴 매칭 (java 21+)
double area(Shape shape) {
    return switch (shape) {
        case Circle c    -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.width() * r.height();
        // default가 필요 없음! sealed이므로 컴파일러가 모든 케이스를 알고 있음
    }
}
```
위 코드에서 3가지가 맞물린다.
sealed interface - Shape의 하위 타입이 Circle과 Rectangle뿐이라고 컴파일러에게 알려준다.
그래서 switch에서 default가 없어도 컴파일 에러가 안난다. 
나중에 Triangle을 추가하면 이 switch를 쓰는 모든 곳에서 컴파일 에러가 발생해서 누락을 방지한다.

record - 각 타입이 어떤 데이터를 가지는지 명확하게 선언합니다. Circle은 radius, Rectangle은 width와 height, 
불편이고 equals/hashCode도 자동이니까 값 객체로 안전하게 쓸 수 있다.

패턴 매칭 - case Circle c가 타입 검사와 변수 바인딩을 동시에 한다. 예전처럼 instanceof 체크 후 캐스팅할 필요 없다.
이걸 함수형 언어와 비교하면 Haskell의 대수적 데이터 타입과 거의 같은 구조이다.
```commandline
-- Haskell 버전
data Shape = Circle Double | Rectangle Double Double

area :: Shape -> Double
area (Circle r)      = pi * r * r
area (Rectangle w h) = w * h
```

## 제약 사항
Record에는 몇가지 제약이 있다.
컴포넌트 외의 인트턴스 필드에 추가로 선언할 수 없고, 다른 클래스를 상속할 수 없으며 필드가 final이라 가변 상태를 가질 수 없다.
또한 컴포넌트에 List같은 가변 객체를 넣으면 외부에서 내용을 바꿀 수 있으므로, 방어적 복사(defensive copy)를 고려해야된다.