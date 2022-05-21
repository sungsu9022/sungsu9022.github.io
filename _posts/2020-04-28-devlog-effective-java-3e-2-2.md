---
title: 이펙티브 자바 3판 - 2. 객체 생성과 파괴 - 2
author: sungsu park
date: 2020-04-28 00:34:00 +0800
categories: [DevLog, Java]
tags: [Effective Java]
---

# 이펙티브 자바 3판 - 2. 객체 생성과 파괴 - 2
* [Item5. 자원을 직접 명시하지 말고 의존 객체 주입을 사용하라.](#item5)
* [Item6. 불필요한 객체 생성을 피하라](#item6)
* [Item7. 다 쓴 객체 참조를 해제하라](#item7)
* [Item8. finalizer 와 cleaner 의 사용을 피하라](#item8)
* [Item9. 생성자에 매개변수가 많다면 빌더를 고려하라](#item9)


## <a id="item5">Item5. 자원을 직접 명시하지 말고 의존 객체 주입을 사용하라.</a>
* 사용하는 자원에 따라 동작이 달라지는 클래스에는 정적 유틸리티 클래스나 싱글턴 방식이 적합하지 않음.
* 인스턴스를 생성할 때 생성자에 필요한 자원을 넘겨주는 방식을 사용
### 의존 객체 주입은 유연성과 테스트에 용이하다.
``` java
public class SpellChecker {
	private final Lexicon dictionary;

	public SpellChecker(Lexicon dictionary) {
		this.dictionary = Objects.requireNonNull(dictionary);
	}

	public boolean isVlaid(String word) { ...	}

	public List<String> suggestions(String type) { ...	}
}
```
* 의존 객체 주입을 생성자, 정적 팩터리, 빌더 등 아무런 방법에 적용하면 됨.

### 팩터리 메서드 패턴
* 의존 객체 주입의 쓸만한 변형 방식
* 생성자에 자원 팩터리 객체를 넘겨주는 방식
* Supplier<T> 인터페이스를 사용하면 됨.
* 한정적 와일드카드 타입(bounded wildcard type)을 사용해 팩터리의 타입 매개변수를 제한
``` java
Mosaic create(Supplier<? extends Tile> titleFactory) { ... }
```
* 의존 객체 주입이 유연성과 테스트 용이성을 개선해주긴 하지만, 의존성이 많아지면 코드를 어렵게 만들기도 함.

### 핵심 정리
자원이 클래스 동작에 영향을 준다면 싱글턴과 정적 유틸리티 클래스는 사용하지 않는 것이 좋다.(의존 객체 주입을 통해 하자)

## <a id="item6">Item6. 불필요한 객체 생성을 피하라</a>
똑같은 기능의 객체를 매번 생성하기 보다는 객체 하나를 재사용하는 편이 나을 때가 많다.(당연한 이야기)

### String 인스턴스 관련
``` java
String s = new String("bikini"); // 따라하지 말것.

String s = "bikini";
```
* 생성자로 생성하는 케이스는 매번 새로운 String 인스턴스를 생성한다.
* 2번쨰 방식을 사용하면 하나의 String 인스턴스를 사용하고, 가상 머신 안에서 이와 똑같은 문자열 리터럴을 사용하는 모든 코드가 같은 객체를 재사용함이 보장된다.
   * 참조 : https://docs.oracle.com/javase/specs/jls/se7/html/jls-3.html#jls-3.10.5

* Boolean(String) 생성자 <<<< Boolean.valueOf(String)

### 생성비용이 비싼 객체 처리
* 비싼 객체가 반복해서 필요하다면 캐싱하여 재사용한다.
* String.matches 메소드를 쓰면 간편하지만, 성능이 중요한 상황에서 반복해서 사용하기엔 적합하지 않음.
   * 해당 메소드에서 생성하는 Pattren 객체는 한번 쓰고 버려짐.
   * Pattren 유한 상태 머신(finite sate machine)을 만들기 때문에 인스턴스 생성 비용이 높음.
* Regular Expression -> Pattren 객체를 이용
* String.matches vs Pattern.matchers 성능 비교
   * 1.1마이크로s / 0.17 마이크로s

### 오토박싱
* 오토박싱이란 기본타입과 박싱된 기본 타입을 섞어 쓸때 자동으로 상호 변환해주는 기술.
* 불필요한 객체를 만들어내는 예 중 하나이다.
* 오토 박싱이 기본 타입과 그에 대응하는 박싱된 타입의 구분을 흐려주지만, 완전히 없애주는 것은 아님.
``` java
private static long sum() {
        Long sum = 0L;
        for (long i = 0; i <= Integer.MAX_VALUE; i++)
            sum += i;
        return sum;
    }
```
* sum 변수의 long이 아닌 Long으로 선언해서 불필요한 Long 인스턴스가 생성됨.

* 박싱된 타입보다는 기본 타입을 사용하고, 의도치 않은 오토박싱이 숨어들지 않도록 주의하자.

###  객체 생성이 비싸니 피해야 한다(?) -> 객체 풀
* 아주 무거운 객체가 아닌 다음에야 단순히 객체 생성을 피하고자 객체 풀을 만들다던지는 안하는 것이 좋음.
* DB 연결같은 경우 생성비용이 비싸니 재사용하는 편이 낫지만 그렇지 않은 경우가 많음.
* 객체 풀은 코드를 헷갈리게 만들고 메모리 사용량을 늘리고, 성능을 떨어뜨린다.

### 방어적 복사 vs 불필요한 객체 생성
* **방어적 복사가 필요한 상황에서 객체를 재사용했을 때의 피해는** 필요 없는 객체를 반복 생성했을 때의 피해보다 훨씬 크다.
   * 잘 모르면 차라리 불필요한 객체 생성은 여러번 하는게 나을수도 있음.


## <a id="item7">Item7. 다 쓴 객체 참조를 해제하라</a>
### 메모리 누수
``` java
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        return elements[--size];
    }

    /**
     * 원소를 위한 공간을 적어도 하나 이상 확보한다.
     * 배열 크기를 늘려야 할 때마다 대략 두 배씩 늘린다.
     */
    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }

}
```
* 위 스택을 오래 수행하다보면 점차 가비지 컬렉션 활동과 메모리 사용량이 늘어나 성능 저하 발생
* 메모리 누수의 원인?
   * 객체들의 다 쓴 참조(obsolete reference)을 여전히 가지고 있기 때문. ( elements 배열의 활성 영역 밖 )
* 해법
  * 해당 참조를 다 썼을 때 null 처리(참조 해제) - pop 시점에 null 처리
``` java
public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
       Object result = elements[--size];
       elements[size] = null; // 다쓴 참조 해제
        return result;
    }
```

### 객체 참조를 null 처리 해야 하는 경우
*  객체 참조를 null 처리하는 일은 예외적인 경유여야 한다.
* 가장 좋은 참조 해제 방법
   * 참조를 담은 변수를 유효 범위(scope) 밖으로 밀어내는 것
* 자기 메모리를 직접 관리하는 클래스 인 경우 프로그래머가 항상 메모리 누수에 주의해야 함.
* 캐시 역시 메모리 누수를 일으키는 주범
* WeakHashMap
   * http://blog.breakingthat.com/2018/08/26/java-collection-map-weakhashmap/
   * key의 참조가 사라지면 자동으로 GC 대상이 됨
   * 정확히 이런 케이스에서만 유용
```
public class WeakHashMapTest {

    public static void main(String[] args) {
        WeakHashMap<Integer, String> map = new WeakHashMap<>();

        Integer key1 = 1000;
        Integer key2 = 2000;

        map.put(key1, "test a");
        map.put(key2, "test b");

        key1 = null;

        System.gc();  //강제 Garbage Collection

        map.entrySet().stream().forEach(el -> System.out.println(el));

    }
}
```
* LinkedHashMap
   * http://javafactory.tistory.com/735

* 리스너(Listener) 혹은 콜백(Callback)
   * 클라이언트 코드에서 콜백을 등록만 하고 명확히 해제하지 않는 경우에 발생할 수 있음.
   * 콜백을 약한 참조(weak reference)로 저장하면 즉시 수거 ( WekHashMap에 키로 저장)

### 핵심 정리
* 메모리 누수는 겉으로 잘 드러나지 않음.
* 철저한 코드리뷰 힙 프로파일러 같은 디버깅 도구를 동원해야만 발견되기도 함.
* 즉 발견하기 어렵기 때문에 예방법을 잘 익히자!


## <a id="item8">Item8. finalizer 와 cleaner 의 사용을 피하라</a>
### GC는 컨트롤 가능한가?
- 내가 원할때 소멸시키는가 / 아니다.
- finalizer의 대안 cleaner 역시 문제가 많다.
- try with resource(auto closable) vs finalize  gc 성능이 50배(12ns vs 550ns) 차이 난다.
- 그럼 언제 저것들을 쓰고 있나? / 효과있나?
    * 닫지 않은 파일/커넥션등을 아주~늦게 나마 회수해준다.(FileInputStream, ThreadPoolExecutor)
    * 네이티브피어(jni 같이 c 등 다른 언어 메소드를 연결하는 것)  객체(자바 객체가 아니니 알지 못해서) //이
때는 성능저하가 불가피 할 듯 보이고 close()를 꼭 해야할 것 같다.
- 이 대안은 그럼 무엇?
    * AutoCloseable을 구현한다.
``` java
public class Room implements AutoCloseable {
    private static final Cleaner cleaner = Cleaner.create();

    // 청소가 필요한 자원. 절대 Room을 참조해서는 안 된다!
    private static class State implements Runnable {
        int numJunkPiles; // Number of junk piles in this room

        State(int numJunkPiles) {
            this.numJunkPiles = numJunkPiles;
        }

        // close 메서드나 cleaner가 호출한다.
        @Override public void run() {
            System.out.println("Cleaning room");
            numJunkPiles = 0;
        }
    }

    // 방의 상태. cleanable과 공유한다.
    private final State state;

    // cleanable 객체. 수거 대상이 되면 방을 청소한다.
    private final Cleaner.Cleanable cleanable;

    public Room(int numJunkPiles) {
        state = new State(numJunkPiles);
        cleanable = cleaner.register(this, state);
    }

    @Override public void close() {
        cleanable.clean();
    }
}
```
- 사실 위의 코드는 정확한 흐름을 모르겠다.(수정필요)
- 그럼 힙 메모리 세팅, gc 종류 선택 등에 대해 공유(난 경험이 없다..)

## <a id="item9">Item9. try - finally 보다 try-with-resource를 사용하라</a>
 - 자원(파일, 커넥션) 을 닫는 것을 클라이언트가 놓칠 수 있다.(커넥션이 계속 열고 안닫아지면..)
 - 일반적으로 finally에 close를 많이 하는게 대다수이지만 JAVA7에서 추가된 try-with-resource를 사용하는것이 좋음.
``` java
static String firstLineOfFile(String path) throws IOException {
        BufferedReader br = new BufferedReader(new FileReader(path));
        try {
            return br.readLine();
        } finally {
            br.close();
        }
 }
```
- 만약 한번더 오픈을 한다면?
``` java
 static void copy(String src, String dst) throws IOException {
        InputStream in = new FileInputStream(src);
        try {
            OutputStream out = new FileOutputStream(dst);
            try {
                byte[] buf = new byte[BUFFER_SIZE];
                int n;
                while ((n = in.read(buf)) >= 0)
                    out.write(buf, 0, n);
            } finally {
                out.close();
            }
        } finally {
            in.close();
        }
    }
```
- 이 경우 어떤 문제에 의해서 close에서도 문제가 생기다면? 두번째(close)예외의 메시지만 준다. 그래서 문제 파악을 힘들게 만든다.
- 아래는 try with resource 로 고친 코드
``` java
 static String firstLineOfFile(String path) throws IOException {
        try (BufferedReader br = new BufferedReader(
                new FileReader(path))) {
            return br.readLine();
        }
  }

static void copy(String src, String dst) throws IOException {
        try (InputStream   in = new FileInputStream(src);
             OutputStream out = new FileOutputStream(dst)) {
            byte[] buf = new byte[BUFFER_SIZE];
            int n;
            while ((n = in.read(buf)) >= 0)
                out.write(buf, 0, n);
        }
 }
```

