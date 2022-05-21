---
title: 이펙티브 자바 3판 - 3. 모든 객체의 공통 메서드
author: sungsu park
date: 2020-05-17 16:34:00 +0800
categories: [DevLog, Java]
tags: [Effective Java]
---

# 이펙티브 자바 3판 - 3. 모든 객체의 공통 메서드
* [Item10. equals 는 일반 규약을 지켜 재정의하라](#item10)
* [Item11. equals를 재정의하려거든 hashCode도 재정의하라 ](#item11)
* [Item12. toString을 항상 재정의하라](#item12)
* [Item13. clone 재 정의는 주의해서 진행해라.](#item13)
* [Item14. Comparable을 구현할지 고려하라.](#item14)

## <a id="item10">Item10. equals 는 일반 규약을 지켜 재정의하라</a>
> equals 재정의하는데 있어서 곳곳에 합정이 있으며, 자칫 잘못하면 끔찍한 결과를 초래할 수 있음. 이를 회피하는 가장 쉬운 길은 아예 재정의하지 않는 것이지만 실제로 개발하다보면 재정의가 필요한 경우도 있을수 있다.(물론 lombok을 사용하면 조금 더 쉬워진다.)

### 재정의를 하지 않아야 하는 상황
- 각 인스턴스가 본질적으로 고유한 경우(값을 표현하는게 아니라 동작하는 개체를 표현하는 클래스
   - Thread 클래스 같은것, 추가로 우리가 개발하는 Spring bean용 클래스의 경우를 생각해도 좋을 것 같다.

- 인스턴스의 논리적 동치성(logical equality)을 검사할 일이 없는 경우
   - Pattern클래스의 인스턴스 2개가 같은 정규표현식을 나타내는지 검사하는것과 같은 행위(논리적 동치성 검사)
   - 설계자는 이 방식을 원하지 않거나 애초에 필요하지 않다고 판단할 수도 있음.

- 상위 클래스에서 재정의한 equals가 하위 클래스에도 딱 들어맞는 경우
   - Collection 클래스류와 같은..

- 클래스가 private이거나 package-private이고 equals 메서드를 호출할 일이 없는 경우

### equals를 재정의를 해야 하는 상황
 - 객체 식별성(object identity, 두 객체가 물리적으로 같은지?)이 아니라 논리적 동치성을 확인해야 하는데, 상위 클래스의 equals가 논리적 동치성을 비교하도록 재정의되지 않은 경우
 - 위에 정의된 말이 조금 이해가 어려운데 쉽게 풀이하면
    - 값 클래스 (Interger, String ..)
    - 값이 같은 인스턴스가 둘 이상 만들어지지 않음을 보장하는 인스턴스 통제 클래스
        - Boolean.TRUE, Boolean,FALSE, Enum류 등
    - 비즈니스 로직에서 사용할 Entity Class 적절히 PK로 활용될 수 있는 데이터셋을 기준으로 equals를 구현할수 있음

### equals 재정의할때 따라야할 일반 규약
equals 메서드는 동치 관계(equivalence relation)을 구현하며, 다음을 만족

- 반사성(reflexivity) : null이 아닌 모든 참조 값 x에 대해 x.equals(x)는 true
- 대칭성(symmetry) : null이 아닌 모든 참조 값 x,y에 대해 x.equals(y)가 true면  y.equals(x)도 true
- 추이성(transitivity) : null이 아닌 모든 참조 값 x,y,z에 대해 x.equals(y)가 true이고, y.equals(z)가 true면 x.equals(z)도 true다.
- 일관성(consistency) : null이 아닌 모든 참조 값 x,y에 대해 x.equals(y)를 반복해서 호출해도 항상 true 또는 false만을 반환
- null-아님 : null이 아닌 모든 참조값 x에 대해 x.equals(null)은 false이다.


### 대칭성(symmetry)
``` java
public final class CaseInsenstiveString{
     private final String s;
     public CaseInsensitiveString(String s){
         this.s = Objects.requireNorNull(s);
     }
 }

@Override
public boolean equals(Object o) {
    if (o instanceof CaseInsensitiveString)
        return s.equalsIgnoreCase(
                ((CaseInsensitiveString) o).s);
    if (o instanceof String)  // 한 방향으로만 작동한다!
        return s.equalsIgnoreCase((String) o);
    return false;
}
```
 - CaseInsensitiveString.equals(String) 에서는 동작하지만 String.equals(CaseInsensitiveString)에서는 동작하지 않고 한방향으로만 작동하기 때문에 대칭성을 명백히 위반하게 됨.

### 추이성(transitivity)
 - null이 아닌 모든 참조 값 x,y,z에 대해 x.equals(y)가 true이고, y.equals(z)가 true면 x.equals(z)도 true다.

``` java
public class Point {
    private final int x;
    private final int y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }

    @Override public boolean equals(Object o) {
        if (!(o instanceof Point))
            return false;
        Point p = (Point)o;
        return p.x == x && p.y == y;
    }
}

public class ColorPoint extends Point {
    private final Color color;

    public ColorPoint(int x, int y, Color color) {
        super(x, y);
        this.color = color;
}

// 대칭성 위배
@Override
public boolean equals(Object o) {
        if (!(o instanceof ColorPoint))
            return false;
        return super.equals(o) && ((ColorPoint) o).color == color;
}

// 추이성 위배
@Override
public boolean equals(Object o) {
        if (!(o instanceof Point)) {
            return false;
        }

   // o가 일반 Point면 색상을 무시하고 비교한다.
    if (!(o instanceof ColorPoint)) {
            return o.equals(this);
        }

        ColorPoint cp = (ColorPoint) o;
        return cp.point.equals(point) && cp.color.equals(color);
}
```
 - 대칭성 위배 코드를 살펴보면 Point의 equals는 색상을 무시하고, ColorPoint의 equals는 입력 매개변수의 클래스 종류가 다르다며 매번 false를 반환하게 된다.
 - 추이성 위배 코드를 살펴보면 대칭성은 지켜주지만 추이성을 깨버린다.

 - 구체 클래스를 확장해 새로운 값을 추가하면서 equals 규약을 만족시킬 방법은 존재하지 않는다...(객체 지향적 추상화의 이점을 포기하지 않는 한)

- 리스코프 치환 원칙(Liskov substitution principle)
   - 어떤 타입에 잇어 중요한 속성이라면 그 하위 타입에서도 마찬가지로 중요. 타입의 모든 메서드가 하위 타입에서도 똑같이 잘 동작해야 함.
   - Point를 기준으로 풀이하면 Point의 모든 하위 클래스는 정의상 모두 Point이므로 어디서든 Point 타입으로 활용될 수 있어야 한다는 말

``` java
// 리스코프 치환법칙 위배
public boolean equals(Object o) {
    if (o == null || o.getClass() != getClass())
        return false;
    Point p = (Point) o;
    return p.x == x && p.y == y;
}
```

- 위와 같이 equals를 구현해둔 경우에 Point의 하위 클래스를 정의하고 이를 Set과 같은 컬렉션에 넣어서 처리를 하고자하면 재대로 동작하지 않는걸 확인할 수 있다. 이 원인은 Colletion.contains 동작방식에 있음
- Colletion.contains(Object o) 동작방식
   - 대부분의 Collection interface 구현체 대부분은 contains을 판단하는데 있어서 객체의 equals 메서드를 사용하여 검증하므로 이 구현이 의도된대로 정의되어있어야 함.
 - 이 문제를 해결할 수 있는 대표적인 방법은 "상속 대신 컴포지션을 사용하는 것"이다.
    - 상위 클래스 타입의 변수를 멤버로 두고 그것을 반환하는 뷰 메소드를 public으로 작성한다.
    - 아무 값도 갖지 않는 클래스를 베이스로 두고 확장한다.(베이스 클래스를 인스턴스하지 않기 때문에 위배되지 않는다)

``` java
public class ColorPoint {
    private final Point point;
    private final Color color;

    public ColorPoint(int x, int y, Color color) {
        point = new Point(x, y);
        this.color = Objects.requireNonNull(color);
    }

    /**
     * 이 ColorPoint의 Point 뷰를 반환한다.
     */
    public Point asPoint() {
        return point;
    }

    @Override public boolean equals(Object o) {
        if (!(o instanceof ColorPoint))
            return false;
        ColorPoint cp = (ColorPoint) o;
        return cp.point.equals(point) && cp.color.equals(color);
    }
}
```

### 일관성(consistency)
 - null이 아닌 모든 참조 값 x,y에 대해 x.equals(y)를 반복해서 호출해도 항상 true 또는 false만을 반환
 - 두 객체가 같다면 앞으로도 영원히 같아야 한다는 의미
 - 클래스를 작성할때 불변 클래스로 만드는게 나을지를 심사숙고하자.
 -  equals의 판단에 신뢰할 수 없는 자원이 끼어들게 해서는안된다.
    - java.net.URL의 equals가 예다. URL과 ip의 호스트를 비교하는데 이때 네트웍을 통하게 된다.

### null-아님
 - 동치성 검사를 위해 적절한 형변환 후 값을 비교한다.
 - instanceof는 비교하는 객체가 null 인지 검사한다.(그래서 ==null 을 할 필요 없다)

``` java
// 명시적 null 검사 필요 없음.
@Override
public boolean equals(Object o) {
   if(!(o instanceof MyType)) {
        return false;
   }
  MyType type = (MyType) o;
   ...
}
```

### equals 메서드 단계별 구현 방법
 - 1) == 연산자를 사용해 입력이 자기 자신의 참조인지 확인.
 - 2) instanceof 연산자로 입력이 올바른 타입인지 확인.
 - 3) 입력을 올바른 타입으로 형변환
 - 4) 입력 객체와 자기 자신의 대응되는 핵심 필드들이 모두 일치하는지 하나씩 검사

### equals 관련 팁
- primitive type (float, double 제외) 는 "==" 연산자로 비교하고, 레퍼런스 타입 필드는 equals 메서드로 비교한다.
- float,double은 wrapper class 의 static method인  compare 메서드로 비교한다.
   - 이렇게 특별취급하는 이유는 NaN, -0.0f, 특수한 부정소수 값 등을 다뤄야 하기 때문
   - wrapper class의 equals를 사용할수도 있는데 오토박싱을 수반할 수 있어서 성능상 좋지 않을 수 있음
- 성능이 걱정된다면 cost가 적을 것 같은 필드부터 비교한다.
- 동기화용 락(lock) 필드 같은 논리적 상태와 관련 없는 필드는 연산하지 말자. (논리적 상태만을 비교하자)
- 캐쉬된 값을 저장하는 파생클래스 변수가 있을 경우 활용하자.

- 완벽한 equals 를 보여주는 코드
``` java
public final class PhoneNumber {
    private final short areaCode, prefix, lineNum;

    public PhoneNumber(int areaCode, int prefix, int lineNum) {
        this.areaCode = rangeCheck(areaCode, 999, "지역코드");
        this.prefix   = rangeCheck(prefix,   999, "프리픽스");
        this.lineNum  = rangeCheck(lineNum, 9999, "가입자 번호");
    }

    private static short rangeCheck(int val, int max, String arg) {
        if (val < 0 || val > max)
            throw new IllegalArgumentException(arg + ": " + val);
        return (short) val;
    }

    @Override public boolean equals(Object o) {
        if (o == this)
            return true;
        if (!(o instanceof PhoneNumber))
            return false;
        PhoneNumber pn = (PhoneNumber)o;
        return pn.lineNum == lineNum && pn.prefix == prefix
                && pn.areaCode == areaCode;
    }
}
```

### 추가 코멘트
 - Java Platform 내에서 equlas의 역할, 어떻게 동작하는가 이런 개념적인 내용은 Java 개발자라면 필수로 알고 있어야 하는 개념들이라고 생각합니다. 실제로 실무에서 이렇게 equlas 메서드를 직접 구현하는케이스는 상당히 드믈것이라 생각되는데요.
 - 이미 Lombok 등 다른대안들이 많이 있으니깐요. 하지만 그렇다고 이 개념들을 알 필요가 없는것은 아닙니다.




## <a id="item11">Item11. equals를 재정의하려거든 hashCode도 재정의하라</a>
> equals를 재정의한 클래스 모두에서 hashCode도 재정의해야 한다. (Lombok의 annotation이 @EqulasAndHashCode 로 묶어둔 이유기도 함.)

### hashCode 관련 규약(Object 명세)
 - 1) equals 비교에 사용되는 정보가 변경되지 않았다면 해당 객체의 hashCode는 몇 번을 호출해도 일관된 값을 반환해야 한다.
 - 2) equals(Object)가 두 객체를 같다고 판단했다면 hashCode도 같은 값을 반환해야 한다.
 - 3) equals(Object)가 두 객체를 다르다고 판단했더라도, 두 객체의 hashCode가 서로 다른 값을 반환할 필요는 없다. 단, 다른 객체에 대해서는 다른 값을 반환해야 해시 테이블의 성능이 좋아진다는 것을 알아야 한다.

### hashCode 일반 규약을 어기는 경우 HashMap, hashSet등에서 문제가 발생한다.
 - 위 2번 조항의 내용과 연관성이 있는데 같은 객체라면 같은 해시코드를 반환해야 hashMap, hashSet에 유일하게 저장될 수 있다.

``` java
HashMap<PhonNumber,String> m = new Hashmap<>();
m.put(new PhoneNumber(707, 867, 5309),"jenny");
m.get(new PhoneNumber(707, 867, 5309));
```
 - jenny가 나올것 같지만, 실제 결과는 null
- 넣을때 한번, 꺼낼때 한번 객체를 생성했다. 이들은 논리적 동치이나. 둘의 hashCode가 다르기 때문에 HashMap에서 찾을 수 없음.이 케이스로 꼭 해야 겠다면 put한 객체를 로컬변수로 저장한 뒤 해당 객체를 get()의 파라메터로 넘겨야 한다.

### 절대로 하지 말아야 할 hashCode 구현 행위
``` java
@override
public int hashcode(){ return 42;}
```
- 학창시절 HashTable의 시간복잡도를 배우면서 O(1)의 엄청난 성능을 보이는것으로 배웠을텐데, 위처럼 모든 객체가 같은 hashCode를 return하도록 한다면 O(n)의 성능을 만들어낼 수 있다. 모든 해쉬가 같기 때문에 결국 Linked List처럼 동작하게 된다.
- [hash 관련된 내용이잘 설명되어 있는 블로그](https://ratsgo.github.io/data%20structure&algorithm/2017/10/25/hash/)

### 일반적으로 hashCoed method를 만드는 방법
``` java
@Override
// PhoneNumber.java의 예
public int hashCode() {
        int result = Short.hashCode(areaCode);
        result = 31 * result + Short.hashCode(prefix);
        result = 31 * result + Short.hashCode(lineNum);
        return result;
}
```

 - 31을 곱하는 이유는 31이 홀수이면서 소수(prime)이기 떄문
 - 이렇게 복잡한 연산으로 hashCode를 만드는 이유는 hash conflict를 최대한 막아 hash 분포를 잘되게 하여 좋은 성능을 유지시키기 위함이다.
 - hash conflict을 더욱 적은 방법을 쓰려고 할떄 고려사항 구아바의 com.google.common.hash.Hashing 참조
   - 조금 살펴봤는데 충돌을 피하기 위한 여러 알고리즘을 적용된 HashFunction을 생성해주는 Util Class네요.
   - 실제로 매우 많은 데이터를 JVM 위에 올려놓고 하나의 Hash Collection을 이용하는 경우 특정 모델의 성능 향상을 위해서 저런 Util을 써볼수 있을 것 같은데 일반적인 엔터프라이즈 웹 애플리케이션 개발에서는 사용하지 않을 가능성이 높아 보이긴 합니다.

- 성능을 고려하는 목적으로 핵심필드를 빼고 hashcode를 정의하지 말자.(해시테이블 성능을 놓치게된다)

## <a id="item12">Item12. toString을 항상 재정의하라.</a>
 - default 동작은 "클래스_이름@16진수로_표시한_해시코드"가 반환됨.
 - 위 정보는 아주 쓸모가 없음..

### 항상 적합한 문자열을 반환하지 않는다.
- PhoneNumber@adbbb (클래스이름@16진수해쉬코드) 를 반환한다.
- 하위 클래스에서는 간결하고 읽기 쉬운(핵심필드 들을) 형태로 toString을 정의해주는 것이 좋다.
- 문제(디버깅)에 용이하게 만든다.
- 모든 핵심 필드들을 출력하는 것이 좋다.
```
    /**
     * 이 전화번호의 문자열 표현을 반환한다.
     * 이 문자열은 "XXX-YYY-ZZZZ" 형태의 12글자로 구성된다.
     * XXX는 지역 코드, YYY는 프리픽스, ZZZZ는 가입자 번호다.
     * 각각의 대문자는 10진수 숫자 하나를 나타낸다.
     *
     * 전화번호의 각 부분의 값이 너무 작아서 자릿수를 채울 수 없다면,
     * 앞에서부터 0으로 채워나간다. 예컨대 가입자 번호가 123이라면
     * 전화번호의 마지막 네 문자는 "0123"이 된다.
     */
@Override
public String toString() {
        return String.format("%03d-%03d-%04d",
                areaCode, prefix, lineNum);
}
```
- 하위 클래스에서 상위클래스의 적절한 toString이 있다면 말고 없다면 꼭 구현해주는것이 좋다.

### 코멘트
 - 사실 여기에 대한 생각은 log를 남기거나 하는 경우에 해당 클래스의 정보를 표시해주어야 하므로 그런 경우에만 toString을 구현해주는것이 맞는듯 하기도 하다. Lombok을 사용하는 경우에 @ToString만 붙여주면 아주 예쁘게 노출되기 떄문에 실제로 어려운일이 아니기도 하고 말이다.
 - 사실 ToString 관련해서는 책에서 다루는 개념적으로 아주 중요한 내용은 없는듯 하다.



## <a id="item13">Item13. clone 재 정의는 주의해서 진행해라.</a>
### Cloneable interface
 - 복제해도 되는 클래스임을 명시하는 용도의 믹스인 인터페이스
 - 아쉽게도 의도한 목적을 제대로 이루지 못했음..
 - 가장 큰 문제는 clone 메서드가 선언된 곳이 Cloneable이 아닌 Object라는 것(심지어 protected )
 - 이런 내용을 보면 Cloneable 인터페이스가 하는일이 없어보이지만 동작방식을 살펴보면 용도가 있긴 있다.

### Object.clone 동작 방식
 -  Object.clone은 Cloneable을 구현한 클래스의 인스턴스에서 호출하면 해당 객체의 필드들을 하나하나 복사한 객체를 반환
 - 그렇지 않은 클래스의 인스턴스에서 호출하면 CloneNotSupportedExpcetion을 반환한다.

### clone으로 파생되는 문제
 - Cloneable을 구현한 클래스는 clone 메서드를 public으로 제공하며, 사용자는 당연히 복제가 재대로 이뤄지리라 기대한다. 하지만 모순적인 매커니즘이 탄생할수 있고, 생성자를 호출하지 않고도 객체를 생성하면서 모순적인 매커니즘이 탄생한다.
 - 상속 구조에 있는 클래스에서 clone 메서드가 super.clone이 아닌 생성자를 호출해 얻은 인스턴스를 반환하더라도 컴파일 에러는 없을텐데, 이 경우 하위 클래스에서 super.clone()을 호출한 경우 잘못된 클래스의 객체가 만들어져서 결국 하위 클래스의 clone 메서드가 재대로 동작하지 않을수 있음.
 - 쓸데없는 복사를 지양한다는 관점에서는 굳이 clone 메서드를 제공하지 않는게 좋다.

### 공변 반환 타이핑(convariant return typing)
 - 재정의한 메서드의 반환 타입은 상위 클래스의 메서드의 반환하는 타입의 하위 타입일 수 있다는 의미
 - 이 주제를 벗어나기도 해서 일단은 간략하게 이야기하면 클래스 상속 구조가 아래와 같다고 할때 return type이 Animal이더라도 Dog 객체는 Animal을 구현한 하위 클래스의 객체를 반환할 수 있다고 쉽게 생각해도 좋을 것 같다.

``` java
private static class Dog extends Animal {
 ...
}

private static class Animal {
 ...
}
```

### 클래스가 가변 객체를 참조하느 경우의 clone
``` java
// Stack의 복제 가능 버전 (80-81쪽)
public class Stack implements Cloneable {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        this.elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        Object result = elements[--size];
        elements[size] = null; // 다 쓴 참조 해제
        return result;
    }

    public boolean isEmpty() {
        return size ==0;
    }

    // 원소를 위한 공간을 적어도 하나 이상 확보한다.
    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```
 - clone 메서드가 단순히 super.clone의 결과를 그대로 반환한다면 size 필드는 올바른 값을 갖겠지만, elements 필드는 원본 Stack 인스턴스와 똑같은 배열을 참조하고 있다. 원본이나 복제본 중 하나를 수정하면 다른 하나도 수정되어 불변식을 해칠수 있고, 프로그램이 이상하게 동작하거나 NPE가 발생할 수 있다.(생성자를 호출해서 만든 객체였다면 이런일은 벌어지지 않았을것이지만, clone은 사실상 생성자와 같은 효과를 냄)

 - clone은 원본 객체에 아무런 해를 끼치지 않는 동시에 복제된 객체의 불변식을 보장해야 함.

``` java
    // 코드 13-2 가변 상태를 참조하는 클래스용 clone 메서드
    @Override public Stack clone() {
        try {
            Stack result = (Stack) super.clone();
            result.elements = elements.clone();
            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
```
 - 이렇게 처리한다면 위 문제를 해결할수 있겠지만, Cloneable 아키텍처는 '가변 객체를 참조하는 필드는 final로 선언하라'는 일반 용법과 충돌이 발생함.(여기서 elemtns와 같은 가변 객체 참조 필드는 실제로 final로 선언되어있어야 하는데 그렇다면 위와 같은 clone 방식으로 처리할 수가 없는것임)

### clone 정의시 주의사항
 - 생성자에서는 재정의될 수 있는 메서드를 호출하지 않아야 함.
 - 상속용 클래스는 Cloneable을 구현해서는 안된다.
 - Cloneable을 구현한 스레드 안전 클래스를 작성할 때는 clone 메서드 역시 적절히 동기화해줘야 한다.

### clone 정의 방법
 - Clonealbe을 구현하는 모든 클래스는 clone을 재정의 해야 하고, 접근제한자는 public으로 반환 타입은 클래스 자신으로 변경한다.
 - super.clone() 호출 후 필요한 필드를 전부 적절히 수정(모든 가변 객체를 복사해야함.)

### 언제 clone을 구현해야 할까?
 - 위에 나열되있는것처럼 모든 가변 객체를 copy해주는 등 clone 구현시에는 복잡한 작업들이 뒤따른다. 그러면 꼭 이렇게까지 구현을 해야하는것일까?
 - Cloneable을 이미 구현한 클래스를확장하는 경우 어쩔 수 없이 clone을 잘 작동하도록 구현해야 한다. 그렇지 않은 상황에서는 더 나은 객체복사 방식을 사용하자(복사 생성자, 복사 팩토리와 같은)

``` java
// 복사 생성자(conversion constructor)
public Yum(Yum yum) { ... };

// 복사 팩터리(conversion factory)
public static Yum newInstance(Yum yum) { ... };
```
### 복사 생성자와 복사 팩터리가 Cloneable /clone 방식보다 나은 이유
 - 언어 모순적이고 위험천만한 객체 생성 매니즘(생성자를 쓰지 않는 방식)을 사용하지 않음.
 - 정상적인 final 필드 용법과 충돌하지 않음
 - 불필요한 검사 예외를 던지지 않고 형변환도 필요 없음
 - 해당 클래스가 구현한 인터페이스 타입의 인스턴스도 인수로 받을수 있음.

### 핵심 정리
 - 새로운 인터페이스를 만들 때는 절대 Cloneable을 확장해서는 안되며, 새로운 클래스도 이를 구현해서는 안된다. (final class라면 위험이 크지는 않음)
 - 성능 최적화 관점에서 검토한 후 별다른 문제가 없을 때만 드믈게 허용해야 한다.
 - 복제 기능은 생성자와 팩토리를 이용하는게 최고이나 배열만은 clone 메서드 방식이 가장 깔끔한, 이 규칙의 합당한 예외

## <a id="item14">Item14. Comparable을 구현할지 고려하라.</a>
> Comparable interface의 유일무이한 메서드인 compareTo
> compareTo는 Object의 메서드가 아님.
> 위 2가지를 제외하고는 Object의 equals와 같지만, compareTo는 단순 동치성 비교에 더해 순서까지 비교할 수 있으며, 제네릭한 특징을 가진다. Comparable을 구현한다는 것은 그 클래스 인스턴스간에 자연적인 순서(natural order)가 있음을 뜻한다.

``` java
pulbic interface Comparable<T> {
      int compareTo(T t);
}
```

### Comparable.compareTo 메서드 일반 규약
 - 이 객체가 주어진 객체보다 작으면 음의 정수, 같으면 0, 크면 양의 정수 반환(비교불가한 경우 ClassCastException 발생)
 - Comparable을 구현한 클래스는 모든 x,y에 대해 sgn(x.compareTo(y)) == -sgn(y.compareTo(x))여야 한다.
 - 추이성을 보장해야 한다. ( x.compareTo(y) > 0 && y.compareTo(z) > 0 이면 x.compareTo(z) > 0
 - 모든 z에 대해 x.compareTo(y) == 0이면 sgn(x.compareTo(z)) ==  sgn(y.compareTo(z))이다.
 - (x.compareTo(y) ==0) == (x.equals(y)) => 필수는 아니지만 권고사항

### 정렬된 컬렉션에서의 동치성 비교
 - 정렬된 컬렉션들은 일반적으로 equals 대신 compareTo를 사용하여 동치성 검증을 한다.
 - compareTo, equals가 일관되지 않은 BigDecimal 클래스 예제
``` java
HashSet<BigDecimal> hashSet = new HashSet();
		TreeSet<BigDecimal> treeSet = new TreeSet<>();

		BigDecimal a = new BigDecimal("1.0");
		BigDecimal b = new BigDecimal("1.00");

		hashSet.add(a);
		hashSet.add(b);

                // [1.0, 1.00]
		System.out.println(hashSet);

		treeSet.add(a);
		treeSet.add(b);

                // [1.0]
		System.out.println(treeSet);
```
 - hashSet에서는 equals 메서드로 비교하여 2개의 객체 인스턴스가 다르다고 판단한다.
 - TreeSet에서는 compareTo 메서드로 비교하기 떄문에 BigDecimal 인스턴스를 같다고 판단한다.

### compareTo 작성시 주의사항
 - Comparable은 타입을 인수로 받는 제네릭 인터페이스이므로 compareTo 메서드 인수 타입은 컴파일 타임에 정해진다.(인수 타입 확인 및 형변환 필요 없음)
 - 각 필드가 동치인지 비교하는게 아니라 순서를 비교, 객체 참조 필드를 비교하려면 compareTo 메서드를 재귀적으로 호출한다. Comparable을 구현하지 않은 필드나 표준이 아닌 순서로 비교해야 한다면 Comparator을 대신 사용

### Java8 이후의 Compare
 - Comparator 인터페이스가 일련의 비교자 생성 메서드(com-parator construction method)와 팀을 꾸려 메서드 연쇄 방식으로 비교자를 생성할 수 있게 됨.
 - d이 방식이 간결하기는 하나 약간의 성능저하가 뒤따를수 있음.

``` java
private static final Comparator<PhoneNumber> COMPARATOR =
            comparingInt((PhoneNumber pn) -> pn.areaCode)
                    .thenComparingInt(pn -> pn.prefix)
                    .thenComparingInt(pn -> pn.lineNum);

public int compareTo(PhoneNumber pn) {
        return COMPARATOR.compare(this, pn);
    }
```

 - comparingInt((PhoneNumber pn) -> pn.areaCode)에서 명시적 Casting한 부분을 주목해보면, 자바의 타입 추론 능력이 이 람다에서 타입을 알아낼만큼 강력하지 않기 때문에 프로그램이 컴파일되도록 명시적 Casting을 처리해준 것.

### hashCode 값을 기준으로 하는 비교자
``` java
 // 이 방식은 사용해서는 안됨. 정수 오버플로우를 일으키거나 부동수점 계산 방식에 따른 오류를 낼수 있음. ( 추이성 위배)
 static Comparator<Object> hashCodeOrder = new Comparator<>() {
     public int compare(Object o1, Object o2) {
          return o1.hashCode() - o2.hashCode()
     }
}

// 정적 compare 메서드를 활용한 비교자
 static Comparator<Object> hashCodeOrder = new Comparator<>() {
     public int compare(Object o1, Object o2) {
          return Interger.compare(o1.hashCode(), o2.hashCode());
      }
}

// 정적 compare 메서드를 활용한 비교자
 static Comparator<Object> hashCodeOrder = Comparator.comparingInt(o -> o.hashCode());
}
```

### 핵심정리
 - 순서를 고려해야 하는 값 클래스를 작성한다면 꼭 Comparable 인터페이스를 구현하여 그 인스턴스들을 쉽게 정렬하고, 검색하고, 비교 기능을 제공하는 컬렉션과 어우러지도록 해야 한다.
 - compareTo 메서드에서 필드의 값을 비교할때는 "<", ">" 사용하지 말아야 한다.
 - 박싱된 기본 타입 클래스에서 제공하는 정적 compare 메서드나 Comparator인터페이스가 제공하는 비교자 생성 메서드를 사용하자

## 추가 코멘트
 - equals, hashCode, Comparable 같은 경우 자바를 이용하는 개발자라면 필수적으로 개념을 확실히 알고 있어야 하지만, 간혹 놓치고 개발하다가 치명적인 버그로 이어지는 경우가 있다. 이런 케이스로 발생하는 버그는 코드를 읽으면서 쉽게 발견되지 않기 때문에 잘 알고 사용하는것이 중요하다.
 - equals, hashCode는 사실상 Lombok같은 라이브러리를 통해서 이용한다면 실수는 거의 없을 것이라고 생각된다.
 - clone은 실제로 직접 개발한 코드에서 정의해서 사용해본 적은 없다. 책에서도 정리된 것처럼 가급적이면 해당 메서드를 구현해야 할 필요성을 못느끼는데 레거시 코드에서 혹시 clone이 정의되어있고, 내가 추가로 개발한 부분이 연관성이 있다면 그떄 관련 개념을 좀 알고 있는것이 도움이 될 것 같다.

