---
title: 이펙티브 자바 3판 - 2. 객체 생성과 파괴 - 1
author: sungsu park
date: 2020-04-25 00:34:00 +0800
categories: [DevLog, Java]
tags: [Effective Java]
---

# 이펙티브 자바 3판 - 2. 객체 생성과 파괴 - 1
* [Item1. 생성자 대신 정적 팩터리 메서드를 고려하자.](#item1)
* [Item2. 생성자에 매개변수가 많다면 빌더를 고려하라](#item2)
* [Item3. private 생성자나 열거 타입으로 싱글턴임을 보증하라.](#item3)
* [Item4. 인스턴스화를 막으려거든 private 생성자를 사용하라.](#item4)



> 객체를 만들어야 할 때와 만들지 말아야 할 때를 구분하는 법
> 올바른 객체 생성 방법과 불필요한 생성을 피하는 방법
> 제때 파괴됨을 보장하고 파괴 전에 수행해야 할 정리 작업

## <a id="item1">Item1. 생성자 대신 정적 팩터리 메서드를 고려하자.</a>
* 클래스는 생성자와 별도로 정적 팩터리 메서드(static factory method)를 제공할 수 있다.
``` java
public static Boolean valueOf(boolean b) {
	return b ? Boolean.TRUE : BOolean.FALSE;
}
```
###  정적 팩터리 메서드가 생성자보다 좋은 장점 5가지
* 1 이름을 가질 수 있다.
  * BigInteger(int, int, Random)과 정적 팩터리 메서드인 BigInteger.probablePrime 중 '어느 쪽이 값이 소수인  BigInteger를 반환한다'는 의미를 더 잘 설명할 것 같은지?
  * 각 목적에 맞는 생성자가 여러개 있다고 했을때, 각각의 생성자가 어떤 역할을 하는지 정확히 기억하기 어려워 엉뚱한 것을 호출하는 실수를 할 수 있다.
* 2 호출될 때마다 인스턴스를 새로 생성하지는 않아도 된다.
  * 불변 클래스(immutable class)는 인스턴스를 미리 만들어 놓거나 새로 생성한 인스턴스를 캐싱하여 재활용하는 식으로 불필요한 객체 생성을 피할 수 있다.
  * Boolean.valueOf(boolean) 메서드는 객체를 아예 생성하지 않음.(성능 향상에 기여)
  * 반복되는 요청에 같은 객체를 반환하는 식으로 인스턴스를 철저히 통제할 수 있음 - 통제(instance-controlled) 클래스
  * 이렇게 통제하는 이유
    * 싱글턴(singleton)으로 만들수도, 인스턴스화 불가(noninstantiable)로 만들 수 있음
    * 동치인 인스턴스가 단 하나뿐임을 보장 가능(열거타입과 같이)
    * 플라이웨이트 패턴의 근간( 나중에 보는걸로..)
* 3 반환 타입의 하위 타입 객체를 반환할 수 있는 능력이 있다.
  * 추상화된 구조인 경우 인터페이스를 정적 팩터리 메서드의 반환 타입으로 해서 구현 클래스를 공개하지 않고 객체를 반환할 수 있음.
    * API를 작게 유지할 수 있다.
    * Collection 프레임워크는 45개 클래스를 공개하지 않기 때문에 API 외견을 훨씬 작게 만들수 있었음.
  * 자바8부터는 인터페이스가 정적 메서드를 가질 수 없다는 제한이 풀려서 인스턴스화 불가 동반 클래스를 둘 이유가 별로 없다.
  * 자바9에서는 private 정적 메서드까지 허락하지만, 정적 필드와 정적 멤버 클래스는 여전히 public이어야 한다.
* 4 입력 매개변수에 따라 매번 다른 클래스의 객체를 반환할 수 있음.
  * EnumSet 클래스는 public 생성자 없이 오직 정적 팩터리만 제공.
    * 원소가 64개 이하면 ㅣong변수 하나로 관리 -> RegularEnumSet
    * 원소가 65개 이상이면 long 배열로 관리하는 JumboEnumSt
    * 클라이언트는 위와 같은 사실을 몰라도 됨
* 5 정적 팩터리 메서드를 작성하는 시점에는 반환할 객체의 클래스가 존재하지 않아도 된다.
  * 이러한 유연함은 Service Provider 프레임워크의 근간이 된다. 대표적인 예로 JDBC 있다.
    * DriverManager.registerDriver() 메서드로 각 DBMS별 Driver를 설정한다. (제공자 등록 API)
    * DriverManager.getConnection() 메서드로 DB 커넥션 객체를 받는다. (service access API)
    * Connection Interface는 DBMS 별로 동작을 구현하여 사용할 수 있다. (service Interface)
  * 위와 같이 동작하게 된다면 차후에 다른 DBMS가 나오더라도 같은 Interface를 사용하여 기존과 동일하게 사용이 가능하다.

###  정적 팩터리 메서드 단점
* 1 상속을 하려면 public이나 protected 생성자가 필요하니 정적 팩터리 메서드만 제공하면 하위 클래스를 만들 수 없다.
  * 상속보단 컴포지션을 사용하도록 유도하고 불변 타입으로 만들려면 이 제약을을 지켜야 하는건 오히려 장점

* 2 정적 팩터리 메서드는 프로그래머가 찾기 어렵다.
  * 생성자처럼 잘 드러나지 않음.
  * API 문서를 잘 써놓고 메서드 이름도 널리 알려진 규약을 따라 지어야 함.

| 명명 규칙                | 설명                                                                             | 예시                                                               |
|--------------------------|----------------------------------------------------------------------------------|--------------------------------------------------------------------|
| from                     | 매개변수를 하나를 받아서 생성                                                    | ``` Date d = Date.from(instant) ```                                |
| of                       | 여러 매개변수를 받아서 생성                                                      | ``` Set<Rank> faceCards = EnumSet.of(JACK, QUEEN, KING); ```       |
| valueOf                  | from과 of의 더 자세한 버전                                                       | ``` BigInteger prime = BigInteger.valueOf(Integer.MAX_VALUE); ```  |
| instace 혹은 getInstance | 매개변수로 명시한 인스턴스를 반환, 같은 인스턴스 보장(x)                         | ``` StackWalker luke = StackWalker.getInstnace(options); ```       |
| create 혹은 newInstance  | 매번 새로운 인스턴스를 생성해 반환                                               | ``` Object newArray = Array.newInstace(classObject, arrayLen); ``` |
| get"Tpye"                | 생성할 클래스가 아닌 다른 클래스에 팩터리 메서드를 정의할 때                     | ``` FileStore fs = Files.getFileStore(path); ```                   |
| new"Type"                | 매번 새로운 인스턴스를 생성해 반환하지만 다른 클래스에 팩터리 메서드를 정의할 때 | ``` BufferedReader br = Files.newBufferedReader(path); ```         |
| "type"                   | getType과 newType의 간결한 버전                                                  | ``` List<Complaint> litany = Collections.list(legacyLitany); ```       |

### 핵심 정리
* 정적 팩터리 메서드와 public 생성자는 각자의 쓰임새가 있으니 상대적인 장단점을 이해하고 사용하는 것이 좋다.

## <a id="item2">Item2. 생성자에 매개변수가 많다면 빌더를 고려하라</a>
### 점층적 생성자 패턴
* 과거에 주로 사용했던 방식
``` java
public NutritionFacts(int servingSize, int servings) {
        this(servingSize, servings, 0);
    }

    public NutritionFacts(int servingSize, int servings,
                          int calories) {
        this(servingSize, servings, calories, 0);
    }

    public NutritionFacts(int servingSize, int servings,
                          int calories, int fat) {
        this(servingSize, servings, calories, fat, 0);
    }

    public NutritionFacts(int servingSize, int servings,
                          int calories, int fat, int sodium) {
        this(servingSize, servings, calories, fat, sodium, 0);
    }
    public NutritionFacts(int servingSize, int servings,
                          int calories, int fat, int sodium, int carbohydrate) {
        this.servingSize  = servingSize;
        this.servings     = servings;
        this.calories     = calories;
        this.fat          = fat;
        this.sodium       = sodium;
        this.carbohydrate = carbohydrate;
    }
```
* 점층적 생성자 패턴을 쓸수도 있지만, 매개변수 개수가 많아지면 클라이언트 코드를 작성하거나 읽기 어렵다.
* 타입이 같은 매개변수가 연달아 늘어서 있으면 찾기 어려운 버그로 이어질 수 있다.

``` java
NutritionFacts cocaCola = new NutritionFacts(240, 8, 100, 0, 35, 27); // 각각의 값이 어떤걸 뜻하는지 한눈에 파악할 수 없음.
```

### 자바빈즈 패턴(JavaBeans pattern)
* 매개변수가 없는 생성자로 객체를 만든 후, 세터(setter) 메서드들을 호출해주는 방식
``` java
NutritionFacts cocaCola = new NutritionFacts();
cocaCola.setServingSize(240);
cocaCola.setServings(8);
cocaCola.setCalories(100);
cocaCola.setSodium(35);
cocaCola.setCarbohydrate(27);
```
* 자바빈즈 패턴에서는 객체 하나를 만들려면  메서드를 여러개 호출해야 하고, 객체가 완전히 생성되기 전까지는 일관성(consistency)이 무너진 상태에 놓이게 된다.
* 클래스를 불변으로 만들 수 없는 것도 문제
* 이러한 단점을 완화하고자 생성이 끝난 객체를 수동으로 freezing해서 변경할 수 없도록 하기도 한다.(그래도 별로)

### 빌더 패턴(Builder pattern)
* 실제로 실무에서 많이 사용됨
* 빌더 패턴을 고려한다면 lombok을 사용하자 [lombok](https://projectlombok.org/)

``` java
@Entity
@Getter
@Builder
public class Category {
    @Id
    @GeneratedValue
    private int id;

    @Column
    private String name;

    @Column
    private String icon;

    private Level level;


    // @OneToMany(mappedBy = "id", cascade = CascadeType.ALL)
    private List<Category> subCagetoryList;

}

 @Test
    public void builderTest() {
        Category.builder()
                .id(1)
                .level(Level.FIRST)
                .name("팬션의류/잡화")
                .icon("패션의류 아이콘.png")
                .subCagetoryList(new ArrayList<>())
                .build();
    }
```
* 빌더패턴은 파이썬과 스칼라에 있는 명명된 선택적 매개변수(named optional parameters)를 흉내낸 것.
* 잘못된 매개변수를 최대한 일찍 발견하려면 빌더의 생성자와 메서드에서 입력 매개변수를 검사하고 build 메소드가 호출하는 생성자에서 여러 매개변에 걸친 불변식(invariant)을 검사하자.
  * lombok을 사용하더라도 bulderMethod를 지정한다던지 해서 검증 가능.
* 불변(immutable 혹은 immutability)과 불변식(invariant)
  * 불변(immutable) - 어떠한 변경도 허용하지 않겠다는 뜻으로 가변(mutable) 객체와 구분하는 용도로 쓰인다. 대표적으로 String 객체는 한번 생성되면 절대 바꿀 수 없는 불변 객체
  * 불변식(invariant) - 프로그램이 실행되는 동안, 혹은 정해진 기간동안 반드시 만족해야 하는 조건(리스트의 크기는 반드시 0 이상)
  * 가변(mutable) 객체에서도 불변식(invariant)가 존재할 수 있음.
* 계층적으로 설계된 클래스와 함께 쓰기에도 좋음.
  * Builder도 상속해서 처리하는데 lombok을 사용하는 것이 더 좋은 방식이라고 생각됨.

``` java
@NoArgsConstructor
@Getter
@Builder
public class BaseFashionItem implements FashionItem {
    private String brand;
    private Category category;
    private String name;
    private long price;
}

public class Shirt extends BaseFashionItem {
    private ColorType colorType;
    private SizeType sizeType;

    @Builder
    public Shirt(String brand, Category category, String name, long price, ColorType colorType, SizeType sizeType) {
        super(brand, category, name, price);
        this.colorType = colorType;
        this.sizeType = sizeType;
    }
}

 @Test
    public void builderTest() {
        Shirt shirt = Shirt.builder()
                .category(Category.builder().build())
                .brand("brand")
                .name("name")
                .price(15000)
                .colorType(ColorType.BLACK)
                .sizeType(SizeType.M)
                .build();

    }
```

* 빌더 패턴의 단점
  * 객체를 만들려면 그에 앞서 빌더부터 만들어야 하는 점.(성능에 민감한 상황에서는 문제가 될 수도 있다.)

* API는 시간이 지날수록 매개변ㄴ수가 많아지기 때문에 애초에 빌더를 고려하는 편이 나을 때가 많다.

## <a id="item3">Item3. private 생성자나 열거 타입으로 싱글턴임을 보증하라.</a>
### 싱글턴이란?
* 인스턴스를 오직 하나만 생성할 수 있는 클래스
* 싱글턴의 전형적인 예로는 함수와 같은 무상태(stateless) 개체나 설계상 유일해야 하는 시스템 컴포넌트를 들수 있다.
  * spring bean 객체를 보통 singleton scope를 이용해서 사용함.
* 클래스를 싱글턴으로 만들면 이를 사용하는 클라이언트를 테스트하기가 어려워 짐.
  * 싱글턴 인스턴스를 mock 구현으로 대체할 수 없음.
  * 인터페이스를 구현해서 만든 싱글턴이면 가능
* private 생성자를 만들면 클라이언트에서는 객체 생성 할수 있는 방법이 존재하지 않아 인스턴스가 전체에서 하나뿐임을 보장할 수 있음.
* 리플렉션 API를 이용해서 호출 가능한데, 이를 방어하기 위해서 생성자에서 두번째 객체가 생성되려 할떄 예외를 던지게 하면 됨.

### 싱글턴을 만드는 방법
* 1 public static final 필드 방식
  * 싱글턴임이 API에 명백히 드러나고, 간결하다는 장점이 있음.
``` java
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();

    private Elvis() { }

    public void leaveTheBuilding() {
        System.out.println("Whoa baby, I'm outta here!");
    }

    // 이 메서드는 보통 클래스 바깥(다른 클래스)에 작성해야 한다!
    public static void main(String[] args) {
        Elvis elvis = Elvis.INSTANCE;
        elvis.leaveTheBuilding();
    }
}
```
* 2 정적 팩터리 메서드 방식
  * API를 바꾸지 않고도 싱글턴이 아니게 변경할 수 있음.(getInstnace 메소드를 변경하면 됨)
  * 정적 팩터리를 제네릭 싱글턴 팩터리로 변경할 수 있음.
  * 메서드 참조를 공급자로 사용할 수 있음.
``` java
public class Elvis {
    private static final Elvis INSTANCE = new Elvis();
    private Elvis() { }
    public static Elvis getInstance() { return INSTANCE; }

    public void leaveTheBuilding() {
        System.out.println("Whoa baby, I'm outta here!");
    }

    // 이 메서드는 보통 클래스 바깥(다른 클래스)에 작성해야 한다!
    public static void main(String[] args) {
        Elvis elvis = Elvis.getInstance();
        elvis.leaveTheBuilding();
    }
}
```
* 3 Enum
  * 더 간결하고, 추가 노력 없이 직렬화 할 수 있음.
  * 복잡한 직렬화 상황, 리플렉션 공격에서도 완벽히 방어
  * 대부분 상황에서는 원소가 하나뿐인 Enum이 싱글턴을 만드는 가장 좋은 방법
    * Enum에서 Enum을 참조하는 경우 순환 참조에 주의

### 실글턴 클래스를 직렬화하는 방법
* 모든 인스턴스 필드를 일시적(transient)이라고 선언하고 readResolve 메소드를 제공(Item 89)
``` java
private Object readResolve() {
return INSTNACE;
}
```

## <a id="item4">Item4. 인스턴스화를 막으려거든 private 생성자를 사용하라.</a>
* 정적 메서드와 정적 필드만을 담은 클래스를 만들고 싶을때가 있다. 객체지향적이지는 않지만 나름의 쓰임새가 있음.(ex java.lang.Math, Utils 시리즈)
* 추상클래스를 만들어서 인스턴스화를 금지하는 것은 안됨(이렇게 해본적도 없음.)
* private 생성자를 추가해서 클래스의 인스턴스화를 막자.
  * lombok을 이용하면 편함.
``` java
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public class ContentLanguageConfig {
	public static final List<Language> SERVICE_LIST;
	public static final List<Language> EMAIL_PUSH_SUPPORT_LIST;
	public static final List<Language> EMAIL_JOIN_SUPPORT_LIST;

	static {
		SERVICE_LIST = Arrays.asList(Language.ENGLISH, Language.SIMPLIFIED_CHINESE, Language.TRADITIONAL_CHINESE, Language.THAI, Language.INDONESIAN, Language.JAPANESE);
		EMAIL_PUSH_SUPPORT_LIST = Arrays.asList(Language.ENGLISH, Language.SIMPLIFIED_CHINESE, Language.TRADITIONAL_CHINESE, Language.JAPANESE);
		EMAIL_JOIN_SUPPORT_LIST = Arrays.asList(Language.ENGLISH, Language.SIMPLIFIED_CHINESE, Language.TRADITIONAL_CHINESE, Language.JAPANESE);
	}

	/**
	 * 서비스중인 언어인지 확인
	 * @param language
	 * @return
	 */
	public static boolean isService(Language language) {
		return SERVICE_LIST.contains(language);
	}
}
```
* 생성자 내에서 Exception을 던지도록 한다면 주석을 달아두자
* 이 방식을 적용했을때 상속을  불가능하게 하는 효과도 가져옴.

## Comment
- 위에 있는 내용들은 어쩌보면 자바의 기본적인 내용임에도 불구하고 실제로는 놓치는 경우가 많을수 있어서 해당 작업을 직접 해보는 것이 좋다.
- sonarqube 정적 분석을 하는 경우에 Util Class에서 Instance화를 하지 않도록 가이드 해주기도 하므로 Sonarqube 같은 정적 분석툴을 도입하고 분석결과를 잘 적용하는것도 코드 퀄리티를 향상시키는데 도움이 된다.
