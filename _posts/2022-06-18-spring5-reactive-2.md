---
title: "[Hands-On Reactive Programming in Spring 5] 2. 스프링을 이용한 리액티브 프로그래밍 기본 개념"
author: sungsu park
date: 2022-06-18 20:28:00 +0800
categories: [DevLog, Spring]
tags: [Java, Spring, Hands-On Reactive Programming in Spring 5, Reactive]

---

# 2. 스프링을 이용한 리액티브 프로그래밍 기본 개념
> 관찰자 패턴
> 스프링 서버에서 보낸 이벤트를 구현한 발행-구독(Publish-Subscribe) 구현
> RxJava 역사 및 기본 개념
> 마블(Marble) 다이어그램
> 리액티브 프로그래밍을 적용한 비즈니스 사례
> 리액티브 라이브러리의 현재 상황

## 2.1 리액티브를 위한 스프링 프레임워크의 초기 해법
- 콜백 및 CompletableFuture 는 메시지 기반 아키텍처를 구현하는데  널리 사용된다. 이러한 역할을 수행하는 주요 후보로 `리액티브 프로그래밍`을 이야이기할수 있다.
- 스프링 프레잌워크는 리액티브 애플리케이션을 구축하는데 유용한 기능들을 제공하고 이를 일부 살펴보도록 하자

### 2.1.1 관찰자(Observer) 패턴
- 관찰자 패턴이 리액티브 프로그래밍과 관련이 없는것처럼 보일수 있으나 약간만 수정하면 그것이 리액티브 프로그래밍 기초가 된다.
- 관찰자 패턴은 관찰자라고 불리는 자손의 리스트를 가지고 있는 주체(subject)를 필요로 한다.
- subject는 자신의 메소드 중 하나를 호출해 관찰자에게 상태변경을 알린다.

<img width="529" alt="스크린샷 2022-06-18 오후 5 14 17" src="https://user-images.githubusercontent.com/6982740/174429162-f8ea4c4a-d1ff-409c-b4e9-895b446322fe.png">

- 관찰자 패턴을 사요용하면 런타임에 객체 사이에 일대다 의존성을 등록할 수 있다.
- 보통 이런 유형의 통신은 단방향으로 이루어지고, 각 부분이 활발히 상호작용하게 하면서 각각의 결합도를 낮출수 있다.
- 이 시스템을 통해 효율적으로 이벤트를 배포할수 있다.

<img width="709" alt="스크린샷 2022-06-18 오후 5 15 46" src="https://user-images.githubusercontent.com/6982740/174429214-37e117ce-efdb-4f72-8133-ca48427fd903.png">

- SubJect와 Observer 2개의 인터페이스로 구성되고, Observer는 usbject에 등록되고  Subject로부터 알림을 수신한다.

``` java
public interface Observer<T> {
   void observe(T event);
}

public interface Subject<T> {
   void registerObserver(Observer<T> observer);
   void unregisterObserver(Observer<T> observer);
   void notifyObservers(T event);
}
```

- Observer와 Subject 모두 인터페이스에 기술된 것 이상은 서로에 대해 알지 못한다.
- Observer 인스턴스는 Subject의 존재를 전혀 인식하지 못할수도 있음. 이 경우 Observers을 등록해주는 역할을 담당할 세번쨰 컴포넌트가 필요할 수도 있다.(Srping DI와 같은 방식이 예시가 될수 있음.)

``` java
public class ConcreteObserverA implements Observer<String> {
   @Override
   public void observe(String event) {
      System.out.println("Observer A: " + event);
   }
}

public class ConcreteObserverB implements Observer<String> {
   @Override
   public void observe(String event) {
      System.out.println("Observer B: " + event);
   }
}

public class ConcreteSubject implements Subject<String> {
   private final Set<Observer<String>> observers = new CopyOnWriteArraySet<>();

   public void registerObserver(Observer<String> observer) {
      observers.add(observer);
   }
   public void unregisterObserver(Observer<String> observer) {
      observers.remove(observer);
   }
   public void notifyObservers(String event) {
      observers.forEach(observer -> observer.observe(event));
   }
}
```

- SUbject의 구현체 안에는 notify를 받는 데 관심이 있는 Observer Set이 있다.
-  registerObserver 및 unregisterObserver 메서드를 이용해 수정(구독 or 구독 취소)가 가능하다.


### 2.1.2 관찰자 패턴 사용 예시

``` java
@Test
   public void observersHandleEventsFromSubjectWithAssertions() {
      // given
      Subject<String> subject = new ConcreteSubject();
      Observer<String> observerA = Mockito.spy(new ConcreteObserverA());
      Observer<String> observerB = Mockito.spy(new ConcreteObserverB());

      // when
      subject.notifyObservers("No listeners");

      subject.registerObserver(observerA);
      subject.notifyObservers("Message for A");

      subject.registerObserver(observerB);
      subject.notifyObservers("Message for A & B");

      subject.unregisterObserver(observerA);
      subject.notifyObservers("Message for B");

      subject.unregisterObserver(observerB);
      subject.notifyObservers("No listeners");

      // then
      Mockito.verify(observerA, times(1))
              .observe("Message for A");
      Mockito.verify(observerA, times(1))
              .observe("Message for A & B");
      Mockito.verifyNoMoreInteractions(observerA);

      Mockito.verify(observerB, times(1))
              .observe("Message for A & B");
      Mockito.verify(observerB, times(1))
              .observe("Message for B");
      Mockito.verifyNoMoreInteractions(observerB);
   }
```

#### 구독을 취소할 필요가 없는 경우 Java 8 람다을 사용해서 구현할수 있다.

``` java
@Test
   public void subjectLeveragesLambdas() {
      Subject<String> subject = new ConcreteSubject();

      subject.registerObserver(e -> System.out.println("A: " + e));
      subject.registerObserver(e -> System.out.println("B: " + e));
      subject.notifyObservers("This message will receive A & B");
   }
```

#### CopyOnWriteArraySet
- CopyOnWriteArraySet은 크기가 일반적으로 작고 읽기 전용 작업이 변경 작업보다 훨씬 많을 때 사용하면 좋은 자료구조인데, thread safety 가 보장되기 떄문에 멀티 스레드 환경에서 사용할 수 있습니다.
- 하지만 변경 작업 같은 경우(add, set, remove) snapshot(복제본)을 이용하여 변경작업을 하기 때문에 비용이 비싸다.
- 내부적으로 object lock, synchronized 등이 사용되기 때문에 읽기 작업이 많고 변경작업이 적은 경우에 사용하는 것이 좋다.

``` java
public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```

#### thread pool을 활룡한 메시지 병렬 처리
- 대기 시간이 긴 이벤트를 처리하는 관찰자가 많은 경우 다음과 같이 병렬 처리할 수 있다.

``` java
   private final ExecutorService executorService = Executors.newCachedThreadPool();

   public void notifyObservers(String event) {
      observers.forEach(observer ->
              executorService.submit(
                      () -> observer.observe(event)
              )
      );
   }
```

- 주의할 점은 자바에서 스레드당 약 1MB를 소비하므로 일반 JVM 응용 프로그램은 몇천개의 스레드만으로도 사용 가능한 메모리를 모두 소모할 수 있다.
  - 추가적으로 thread 1개가 생성될떄 1MB 메모리를 바로 소비한다는 의미는 아니고, 64bit OS 기준으로 stack size가 1MB까지만 할당된다는 의미
  - 그리고 실제로 malloc은 단순히 가상 페이지 를 지정 하기만 할뿐 실제로 사용될 때까지 물리적 페이지 가 할당되지는 않는다. 아마도  추측하기에는 JVM애플리케이션의 `스레드 개수 * 1MB` > `Max Heap Size` 더라도 애플리케이션 구동 자체에 문제가 없는 이뉴는 이런 이유가 아닐까 싶음.

> JVM Thread stack size 확인 : java -XX : + PrintFlagFinal -version | grep ThreadStackSize


#### java.util 패키지 Observer 및 Observable
- JDK 1.0에서 릴리즈되어 제네릭 적용이 안되어 있음. 컴파일 타임에 타입 안정성을 보장하지 않기 떄문에 사용을 지양 해야 함.
- 직접 구현해서 사용할수 도있지만 오류 처리, 비동기 실행, 스레드 안정성, 성능 등 직접적인 관리 비용이 크므로 성숙한 구현체 라이브러리를 사용하도록 하자!

### 2.1.3 `@EventListener`을 사용한 발행-구독 패턴
- Spring에서는 `@EventListener` 애토네이션과 이벤트 발행을 위한 `ApplicationEventPublisher` 클래스를 제공한다.
- 이는 관찰자 패턴과 달리 게시자와 구독자는 다음  그림과 같이 서로를 알 필요가 없다.

<img width="541" alt="스크린샷 2022-06-18 오후 5 53 27" src="https://user-images.githubusercontent.com/6982740/174430439-7d1d83d1-a7ca-411d-80b6-86c7bb594c11.png">

- 발행-구독 패턴은 게시자와 구독자 간에 간접적인 계층을 제공한다.
- 이벤트 채널(메시지 브로커 또는 이벤트 버스라고도 함)은 수신 메시지를 구독자에게 배포하기 전에 필터링 작업을 할수도 있음.
- 토픽 기반 시스템(topic-based-system)의 구독자는 관심 토픽에 게시된 모든 메시지를 수신하게 된다.
- `@EventListener` 애노테이션은 토픽 기반 라우팅과 내용 기반 라우팅 모두에 사용할 수 있다.
- 조건 속성(condition attribute)은 스프링 표현 언어(SPring Expression Languages, SpEL)을 사용하는 내용 기반 라우팅 이벤트 처리를 가능하게 한다.

### 2.1.4 `@EventListener`을 활용한 애플리케이션 개발
- 서버에서 클라이언트로의 비동기 메시지를 전달할 수 있는 웹소켓(WebSocket) 및 SSE(Server-Sent Events) 프로토콜이 있다.
- 일반적으로 SSE는 브라우저에 메시지를 업데이트하거나 연속적인 데이터 스트림을 보내는 데 사용
- SSE기 리액티브 시스템의 구성 요소간에 통신 요구사항을 충족시키는 최고의 후보이다.

#### WebSocket vs SSE 간단한 차이점 비교

<img width="808" alt="스크린샷 2022-06-18 오후 6 05 42" src="https://user-images.githubusercontent.com/6982740/174430852-543dceb6-386d-4c6e-a828-7e8a86df7a64.png">

#### 스프링 웹 MVC를 이용한 비동기 HTTP 통신
- 서비릇 3.0에서 추가된 비동기 지원 기능은 HTTP 요청을 처리하는 기능을 확장했고, 컨테이너 스레드를 사용하는 방식으로 구현됨.
- `Callable<T>`는 컨테이너 스레드 외부에서 실행될수도 있지만 블로킹 호출
- `DeferredResult<T>`는 `setResult(T result)` 메서드를 호출해 컨테이너 스레드 외부에서도 비동기 응답을 생성하므로 이벤트 루프 안에서 사용할 수 있음.
- Spring 4.2부터 지원하는 `ResponseBodyEmitter`가 `DeferredResult`와 비슷하게 동작한다.
- `SseEmitter`는 `ResponseBodyEmitter`을 상속하는 구조로 되어있고, 이걸 사용하면 데이터(payload)를 비동기적으로 보낼 수 있다.
  - 이때 서블릿 스레드를 차단하지 않기 때문에 큰 파일을 스트리밍해야 하는 경우 매우 유용하다.

#### SSE 엔드포인트 노출
- 온도 센서로부터 사용자에게 랜덤한 온도를 전달하는 SSE 엔드포인트

``` java
@RestController
public class TemperatureController {
   static final long SSE_SESSION_TIMEOUT = 30 * 60 * 1000L;
   private static final Logger log = LoggerFactory.getLogger(TemperatureController.class);

   private final Set<SseEmitter> clients = new CopyOnWriteArraySet<>();

   @RequestMapping(value = "/temperature-stream", method = RequestMethod.GET)
   public SseEmitter events(HttpServletRequest request) {
      log.info("SSE stream opened for client: " + request.getRemoteAddr());
      SseEmitter emitter = new SseEmitter(SSE_SESSION_TIMEOUT);
      clients.add(emitter);

      // Remove SseEmitter from active clients on error or client disconnect
      emitter.onTimeout(() -> clients.remove(emitter));
      emitter.onCompletion(() -> clients.remove(emitter));

      return emitter;
   }

   @Async
   @EventListener
   public void handleMessage(Temperature temperature) {
      log.info(format("Temperature: %4.2f C, active subscribers: %d",
         temperature.getValue(), clients.size()));

      List<SseEmitter> deadEmitters = new ArrayList<>();
      clients.forEach(emitter -> {
         try {
            Instant start = Instant.now();
            emitter.send(temperature, MediaType.APPLICATION_JSON);
            log.info("Sent to client, took: {}", Duration.between(start, Instant.now()));
         } catch (Exception ignore) {
            deadEmitters.add(emitter);
         }
      });
      clients.removeAll(deadEmitters);
   }
}
```

- SseEmitter 클래스는 SSE 이벤트를 보내는 목적으로만 이 클래스를 사용할 수 있다.
- Controller에서 `SseEmitter`을 반환하지만 실제로는 `SseEmitter.complete()` 메소드가 호출되거나 오류 발생 또는 시간 초과 발생할 때까지 실제 요청 처리는 계속 된다.

#### 비동기 지원 설정

``` java
@EnableAsync
@SpringBootApplication
public class Application implements AsyncConfigurer {

   public static void main(String[] args) {
      SpringApplication.run(Application.class, args);
   }

   @Override
   public Executor getAsyncExecutor() {
      ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
      executor.setThreadNamePrefix("sse-");
      executor.setCorePoolSize(2);
      executor.setMaxPoolSize(100);
      executor.setQueueCapacity(5);
      executor.initialize();
      return executor;
   }

   @Override
   public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
      return new SimpleAsyncUncaughtExceptionHandler();
   }
}
```
- `@EnableAsync`을 통해 Async Auto Configration을 수행
-  Executor setQueueCapacity 설정을 통해 스레드풀 사이즈를 적절히 조절
  - 이걸 지정하지 않으면 내부적으로는 Integer.MAX_VALUE 사이즈의 LinkedBlockingQueue를 생성해서 core 사이즈만큼의 스레드에서 task를 처리할 수 없을 경우 queue에서 대기하게 됩니다. queue가 꽉 차게 되면 그때 max 사이즈만큼 스레드를 생성해서 처리하게 됩니다.
  - 이런 옵션 조절을 통해 동시성 성능을 제어한다고 이해하면 됨.

#### SSE를 지원하는 UI 작성

``` html
<body>
<ul id="events"></ul>
<script type="application/javascript">
function add(message) {
    const el = document.createElement("li");
    el.innerHTML = message;
    document.getElementById("events").appendChild(el);
}

var eventSource = new EventSource("/temperature-stream");
eventSource.onmessage = e => {
    const t = JSON.parse(e.data);
    const fixed = Number(t.value).toFixed(2);
    add('Temperature: ' + fixed + ' C');
}
eventSource.onopen = e => add('Connection opened');
eventSource.onerror = e => add('Connection closed');
</script>
</body>
```

- 웹페이지는 EventSource를 통해 클라이언트 및 서버 접속을 유지하고 이벤트를 수신한다.
- 또 네트워크 문제 발생이나 접속 시간이 초과하는 경우 자동으로 재접속한다.

<img width="270" alt="스크린샷 2022-06-18 오후 6 41 22" src="https://user-images.githubusercontent.com/6982740/174432103-33b271d8-7d91-4a7e-bf2a-6c0de28c3a26.png">

#### 솔루션에 대한 평가
- 지금까지 설명한 솔루션에는 몇가지 문제가 있다.
- Spring에서 제공하는 발행-구독 매커니즘은 애플리케이션 생명주기 이벤트를 처리하기 위해 도입되었고, 고부하 및 고성능 시나리오를 위한 것이 아니다.
- 또한 비즈니스 로직을 정의하고 구현하기 위해 스프링 내부 매커니즘을 사용해야 하므로 프레임워크의 사소한 변경으로 인해 전체 애플리케이션의 안정성을 위협할 수 있다.
- 또 많은 메서드에 `@EventListener` 애노테이션이 붙어 있고, 전체 워크플로를 설명하는 한줄의 명시적 스크립트도 없는 애플리케이션이라는 것도 단점일수 있다.
- `SseEmitter`을 사용하면 스트림의 종료와 오류 처리에 대한 구현을 추가할 수 있지만 `@EventListener`는 그렇지 않다.
- 이벤트를 비동기적으로 브로드캐스팅하기 위해 스레드풀을 사용하는데 이는 진정한 비동기적 리액티브 접근에서는 필요 없는 일.
- 이러한 문제점 해결을 위해 널리 채택된 최초의 리액티브 라이브러리인 RxJava를 알아보도록 하자.


## 2.2 리액티브 프레임워크 RxJava
- 현재는 RxJava말고도 Akka Streams와 Reactor 프로젝트도 있으나 시작은 RxJava부터 시작됨.
- Rxjava 라이브러니는 Reacative Extensions(ReactiveX라고도 함)의 자바 구현체이다.
- 일반적으로 ReactiveX는 관찰자 패턴, 반복자 패턴 및 함수형 프로그래밍의 조합으로 정의된다.

### 2.2.1 관찰자 + 반복자 = 리액티브 스트림

``` java
public interface RxObserver<T> {
   void onNext(T next);
   void onComplete();
   void onError(Exception e);
}
```

- Iterator와 매우 비슷하지만 `next()` 대신 `onNext()` 콜백에 의해 새로운 값이 통지된다.
- 그리고 `hasNext()` 대신 `onComplete()` 메소드를 통해 스트림의 끝을 알린다.
- 오류 전파를 위해 `onError(Exception e)` 콜백이 제공됨.

#### 리액티브 Observable 클래스
- 리액티브 Observable 클래스는 관찰자 패턴의 주체(Subject)와 일치
- 이벤트를 발생시킬 때 이벤트 소스 역할을 수행

#### Subscriber 추상 클래스
- Observer 인터페이스를 구현하고 이벤트를 소비한다.(실제로 기본 구현체)
- 런타임에 Observable과 Subscriber간의 관계는 메시지 구독 상태를 확인하고 필요한 경우 이를  취소할 수도 있는 구독의 의해 제어됨.

<img width="560" alt="스크린샷 2022-06-18 오후 6 51 44" src="https://user-images.githubusercontent.com/6982740/174432498-3e639cba-c407-4634-ab0d-593bb0564b9c.png">


### 2.2.2 스트림의 생산과 소비
- Observable은 구독자가 구독하는 즉시 구독자에게 이벤트를 전파하는 이벤트 생성기

``` java
public void simpleRxJavaWorkflow() {
      Observable<String> observable = Observable.create(
         new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> sub) {
               sub.onNext("Hello, reactive world!");
               sub.onCompleted();
            }
         }
      );
   }
```

- 자바 스트림 API와 달리 Observable은 재사용이 가능하며 모든 구독자는 구독하자마자 "Hello, reactive workld!" 이벤트를 받게 된다.
- 위 방식은 backpressure)을 지원하지 않아 현재는 사용되지 않는 방식

``` java
Subscriber<String> subscriber = new Subscriber<String>() {
            public void onNext(String s) {
                System.out.println(s);
            }

            public void onCompleted() {
                System.out.println(s"Done!;
            }

            public void onError(Throwable e) {
                System.out.println(e);
            }
        };
```

#### 람다 표현식 사용방식

``` java
public void simpleRxJavaWorkflowWithLambdas() {
      Observable.create(
         sub -> {
            sub.onNext("Hello, reactive world!");
            sub.onCompleted();
         }
      ).subscribe(
         System.out::println,
         System.err::println,
         () -> System.out.println("Done!")
      );
   }
```

- 위와 같이 람다를 사용해서 간략하게 작성할 수도 있으며, 이 외에도 배열, Iterable 컬렉션, Callable, Future 등을 이용해 인스턴스를 만드는 방법도 제공한다.

#### Observable.concat
- 생성과 함께 Observable 스트림을 다른 Observable 인스턴스와 결합해 생성하여 복잡한 워크플로를 쉽게 구현할 수 있다.

``` java
Observable.concat(hello, world, Observable.just("!"))
         .forEach(System.out::print);
```

### 2.2.3 비동기 시퀀스 생성
- 주기적으로 비동기 이벤트 시퀀스를 생성할 수 있다.

``` javav
public void timeBasedSequenceExample() throws InterruptedException {
      Observable.interval(1, TimeUnit.SECONDS)
         .subscribe(e -> System.out.println("Received: " + e));

      Thread.sleep(5000);
   }
```

- 이벤트가 생성되는 것과는 별개의 스레드를 통해 처리되기 때문에 메인스레드 실행을 지연시켜야만 동작한다.

#### 다른 방법

``` java
public void managingSubscription2() throws InterruptedException {
      CountDownLatch externalSignal = new CountDownLatch(3);

      Subscription subscription = Observable
              .interval(100, MILLISECONDS)
              .subscribe(System.out::println);

      externalSignal.await(450, MILLISECONDS);
      subscription.unsubscribe();
   }
```
- 위와 같이 `CountDownLatch`가 전파될때까지 이벤트를 계속 소비하는 방식으로도 처리가 가능.

### 2.2.4 스트림 변환과 마블 다이어그램
#### 마블 다이어그램(marble diagram)
- 메소드 시그니쳐만으로 이해가 어려울수 있기떄문에 발명된 스트림 변환 시각화 다이어그램

#### Map 연산자

``` java
<R> Observable<R> map(Func1<T, R> func)
```

- func 함수가 타입 `<T>`를 타입 `<R>`로 변환하고, map을 통해 `Observable<T>`를 `Observable<R>`로 변환할 수 있음을 의미

<img width="684" alt="스크린샷 2022-06-18 오후 7 03 57" src="https://user-images.githubusercontent.com/6982740/174432955-0b574126-2463-489f-bd5e-281ca18ec5da.png">

#### Filter 연산자

<img width="645" alt="스크린샷 2022-06-18 오후 7 04 18" src="https://user-images.githubusercontent.com/6982740/174432968-d5053625-bfa1-4a58-b5a0-886eb3372351.png">

#### Count 연산자
- 스트림이 무한대일 때는 완료되지 않거나 아무것도 반환할 수 없음.

<img width="649" alt="스크린샷 2022-06-18 오후 7 04 48" src="https://user-images.githubusercontent.com/6982740/174432988-0e31d52e-4100-420f-9122-531e51df3f9a.png">

#### Zip 연산자
- 두 개의 병렬 스트림 값을 결합하는  연산자

<img width="755" alt="스크린샷 2022-06-18 오후 7 05 31" src="https://user-images.githubusercontent.com/6982740/174433013-97958b4b-937d-443d-8572-037838b9505b.png">


``` java
public void zipOperatorExample() {
      Observable.zip(
              Observable.just("A", "B", "C"),
              Observable.just("1", "2", "3"),
              (x, y) -> x + y
      ).forEach(System.out::println);
   }

/**
*
* A1
* B2
* C3
*
**/
```

#### 사용자 지정 연산자 작성
- `Observable.Transformer<T, R>`에서 파생된 클래스를 구현해 사용자 지정 연산자를 작성할수도 있음.
- 이는 `Observable.compose(transformer)` 연산자를 적용해 워크플로에 포함될 수 있다.


### 2.2.5 RxJava 사용의 전제 조건 및 이점
- 서로 다른 리액티브 라이브러리는 API도 조금씩 다르고 구현 방식도 다양하다.(구독자가 관찰 가능한 스트팀에 가입한 후, 비동기적으로 이벤트를 생성해 프로세스를 시작한다는 핵심 개념은 모두 동일)
- 이런 접근방식은 매우 융통성이 있는 구조이고,  생성 및 소비되는 이벤트의 양을 제어할 수 있다. 그로 인해 데아터 작성시에만 필요하고 그 이후에는 CPU 리소스 사용량을 줄일 수 있음.

``` java
// 페이지와 관계 없이 전체 결과를 반환 받는 구조라서 문제
public interface SearchEngine {
   List<URL> search(String query, int limit);
}

// 다음 데이터 반환을 기라딜때 스레드가 블로킹되는 문제가 있음.
public interface IterableSearchEngine {
   Iterable<URL> search(String query, int limit);
}

// CompletableFuture 결과가 List라 한번에 전체를 반환하거나 아무것도 반환하지 않는 방식으로만 동작하는 문제
public interface FutureSearchEngine {
   CompletableFuture<List<URL>> search(String query, int limit);
}

```


``` java
public interface RxSearchEngine {
   Observable<URL> search(String query);
}
```

- Rx의 접근방식을 사용하면 응답성을 크게 높힐 수 있음.
- 최초 데이터 수신 시간(Time To First Byte) 또는 주요 랜더링 경로(Critical Rendering Path) 메트릭으로 성능을 평가하는데 여기서 기존 방식보다 훨씬 나은 결과를 보여준다.

``` java
public void deferSynchronousRequest() throws Exception {
      String query = "query";
      Observable.fromCallable(() -> doSlowSyncRequest(query))
         .subscribeOn(Schedulers.io())
         .subscribe(this::processResult);

      Thread.sleep(1000);
   }
```

- 위 워크플로우는 한 스레드에서 시작해 소수의 다른 스레드로 이동하고, 완전히 다른 새 스레드에서 처리가 완료될 수 있음.
- 이 과정에서 객체 변형은 리스크가 있을수 있으며, 일반적으로 불변 객체를 사용한다.
- 불변 객체는 함수형 프로그래밍 핵심 원리중 하나
  - Java8 Stream에서도 final Variable만 참조할수 있는 이유가 이러한 이유라고 보면 된다.
  - 이 간단한 규칙으로  병렬 처리에서 발생할 수 있는 대부분의 문제를 예방할수 있다.

### 2.2.6 RxJava를 이용해 애플리케이션 다시 만들기

``` java
@Component
public class TemperatureSensor {
   private static final Logger log = LoggerFactory.getLogger(TemperatureSensor.class);
   private final Random rnd = new Random();

  // private filed로 하나만 정의해서 재사용할수 있음
   private final Observable<Temperature> dataStream =
      Observable
         .range(0, Integer.MAX_VALUE)             // 사실상 무한 스트림
         .concatMap(ignore -> Observable
            .just(1)
            .delay(rnd.nextInt(5000), MILLISECONDS)
            .map(ignore2 -> this.probe()))                   // ignore2는 단일 원소 스트림을 생성하는데 필요해서 정의된 것이기에 무시해도 괜찮.
         .publish()      // 각 구독자(SSE 클라이언트)는 각 센서 판독 결과를 공유하지 않는다.
         .refCount();  // 하나 이상의 구독자가 있을때만 입력 공유 스트림에 대한 구독을 생성

   public Observable<Temperature> temperatureStream() {
     return dataStream;
   }

   private Temperature probe() {
      double actualTemp = 16 + rnd.nextGaussian() * 10;
      log.info("Asking sensor, sensor value: {}", actualTemp);
      return new Temperature(actualTemp);
   }
}
```

#### Custom Sse Emitter
``` java
static class RxSseEmitter extends SseEmitter {
      static final long SSE_SESSION_TIMEOUT = 30 * 60 * 1000L;
      private final static AtomicInteger sessionIdSequence = new AtomicInteger(0);

      private final int sessionId = sessionIdSequence.incrementAndGet();
      private final Subscriber<Temperature> subscriber;

      RxSseEmitter() {
         super(SSE_SESSION_TIMEOUT);

         this.subscriber = new Subscriber<Temperature>() {
            @Override
            public void onNext(Temperature temperature) {
               try {
                  RxSseEmitter.this.send(temperature); // onNext() 신호를 수신하면 응답으로부터 SSE 클라이언트에게 신호를 보낸다.
                  log.info("[{}] << {} ", sessionId, temperature.getValue());
               } catch (IOException e) {
                  log.warn("[{}] Can not send event to SSE, closing subscription, message: {}",
                     sessionId, e.getMessage());
                  unsubscribe();
               }
            }

            @Override
            public void onError(Throwable e) {
               log.warn("[{}] Received sensor error: {}", sessionId, e.getMessage());
            }

            @Override
            public void onCompleted() {
               log.warn("[{}] Stream completed", sessionId);
            }
         };

         onCompletion(() -> {
            log.info("[{}] SSE completed", sessionId);
            subscriber.unsubscribe();
         });
         onTimeout(() -> {
            log.info("[{}] SSE timeout", sessionId);
            subscriber.unsubscribe();
         });
      }

      Subscriber<Temperature> getSubscriber() {
         return subscriber;
      }

      int getSessionId() {
         return sessionId;
      }
   }
```

#### SSE 엔드포인트 노출

``` java
@RequestMapping(value = "/temperature-stream", method = RequestMethod.GET)
   public SseEmitter events(HttpServletRequest request) {
      RxSeeEmitter emitter = new RxSeeEmitter();
      log.info("[{}] Rx SSE stream opened for client: {}",
         emitter.getSessionId(), request.getRemoteAddr());

      temperatureSensor.temperatureStream()
         .subscribe(emitter.getSubscriber());

      return emitter;
}
```

- SSE 세션을 온도 측정 스트림을 구독한 새로운 RxSseEmitter에만 바인딩한다.
- 이 구현은 스프링의 EventBus를 사용하지 않으므로 이식성이 더 높고 스프링 컨텍스트가 없이도 테스트가 가능하다.
- 또한 `@EventListener`, `@EnableAsync` 같은 의존성이 없어서 애플리케이션 구성도 더 간단하다.
- RxJava Schedular를 구성한 세밀한 스레드 관리를 할수도 있지만, 이러한 구성이 스프링 프레임워크에 의존하지 않는다.
- 이는 리액티브 프로그래밍이 가지는 능동적 구독이라는 개념의 자연스러운 결과이다.


## 2.3 리액티브 라이브러리의 간략한 역사
- MS에서 Rx의 개념이 시작되었고 Rx.NET을 시작으로, 언제부턴가 외부로 퍼져나감.
- 깃헙의 저스틴 스파서머스와 폴 베츠는 2012년 objectiveC용 Rx를 구현했고, 넷플릭스에서 Rx Java로 이식하여 오픈소스로 공개했다.

## 2.4 리액티브의 전망
- Ratpack에서 RxJava 채택
- Android 유명한 http client인 Retrofit에서도 RxJava 지원
- Vert.x에서도 정식 리액티브 스트림 제공(ReadStream, WriteStream)

### 주의사항
- 하나의 자바 응용 프로그램에 다른 종류의 리액티브 라이브러리 또는 프레임워크를 동시에 사용하면  문제가 발생할 수 있다.
- 리액티브 라이브러리들의 동작은 일반적으로 비슷하지만 세부 구현은 조금씩 다를수 있어서 동시 사용시 발견 및 수정하기 어려운 숨겨진 에러를 유발할 수 있다.
- 이런 전체 리액티브 환경을 아우르며 호환성을 보장하기 위한 표준 API가 바로 `리액티브 스트림`이다. 다음에 이를 자세히 알아보도록 하자!


## Reference
- CopyOnWriteArraySet : https://coding-start.tistory.com/314
- JVM Thread Memory usage 관련 : https://jsravn.com/2019/05/01/jvm-thread-actual-memory-usage/
- 웹소켓과 SSE 비교 : https://surviveasdev.tistory.com/entry/%EC%9B%B9%EC%86%8C%EC%BC%93-%EA%B3%BC-SSEServer-Sent-Event-%EC%B0%A8%EC%9D%B4%EC%A0%90-%EC%95%8C%EC%95%84%EB%B3%B4%EA%B3%A0-%EC%82%AC%EC%9A%A9%ED%95%B4%EB%B3%B4%EA%B8%B0
