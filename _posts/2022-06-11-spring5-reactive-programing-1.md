---
title: "[Hands-On Reactive Programming in Spring 5] 1. 왜 리액티브 스프링인가?"
author: sungsu park
date: 2022-06-11 10:35:00 +0800
categories: [DevLog, Spring]
tags: [Java, Spring, Hands-On Reactive Programming in Spring 5, Reactive]

---

# 1. 왜 리액티브 스프링인가?
## 1.1 왜 리액티브인가?
- 서버진영에서 reactive(반응형)이라는 말이 2019년쯤부터 굉장히 빈번하게 들려왔다.

### 시스템의 가용량 예시
> tomcat worker thread 500으로 설정
> API 응답시간이 250ms

- 이 기준으로 계산해보면 시스템의 최대 TPS 2,000임을 알수 있다.

### 특수한 상황이 생기면?
- 시스템의 thread pool 정보와 api 응답시간 등을 기준으로 서버에서 가용량을 계산해서 HA 구성을 해놓았으면 평상시에는 예상되는 트래픽을 잘 커버할수 있을 것이다.
- 다만 블랙 프라이데이와 같이 특수한 경우에는 이런 예상치를 훨씬 상회하는 트래픽이 유입될수 있음.
- 이런 케이스에서는 가용한 모든 worker thread들이 사용될것이고, 그 이후에 들어오는 request들을 처리하기 위해 worker thead pool에 thread 할당을 요청하지만 가용하지 않아 대기하게 되고 응답시간이 증가하다가 결국 서비스가 불가능한 상태에 이를수 있다.
  -  tomcat worker thread 외에도 db connection pool, API 서비스 로직에서 호출하는 api client의 thread pool 가용량도 초과하는 경우 동일한 문제가 발생할 수 있음.

### 어떻게 대응할것인가?
- 탄력성(elasticity)을 통한 것이다. 트래픽에 따라서 또는 server metric에 따라 탄력적으로 시스템 처리량이 자동 증가/감소할 수 있다면 위와 같은 문제를 해결할 수 있을 것이다.
- thread 부족으로 인한 지연시간이 발생하지 않도록 시스템 확장
- 암달의 법칙(Amdahl's Law)과 건터(Gunther)의 보편적 확장성 모델(Universal Scalability Model)로 설명

### 응답성의 나빠지는것이란?
- 쇼핑 시스템에서 외부 결제 서비스가 중단되고 모든 사용자의 상품 구매 결제가 실패하는상황, 일반적으로 발생해서는 안되는 상황이다.
- 고품질 사용자 경험을 제공하기 위해서는 응답성에 관심을 기울여야 한다.
  - 응답시간이 오래걸리는 API가 있을떄 트래픽이 몰리면 위의 문제처럼 가용한 thread들이 모두 블로킹된 상태에서 계속 요청이 들어오게 되어 결국 서버 전체적으로 문제가 생김.

### 응답성이 나빠지는것을 방지하려면?
- 시스템 실패에도 반응성을 유지할 수 있는 능력, 즉 시스템 복원력을 갖추어야 한다.
- 시스템의 기능 요소를 격리해 모든 내부 장애를 격리하고 서비스 독립성을 확보하여 달성할 수 있다.
  -  MSA에서 Circuit Breakers 도입하여 장애 격리


## 1.2 메시지 기반 통신

``` java
@RequestMapping("/resource")
public Object processRequest() {
    RestTemplate template = new RestTemplate();
    ExamplesCollection result = template.getForObject(
       "http://example.com/api/resource2",
       ExamplesCollection.class
    );
    ...
    processResultFurther(result);
}
```

- 일반적으로 spring mvc 기반으로 requestMapping을 작성하는 경우 위와 같이 개발을 할것이다.
- request per thread model을 사용하는것으로 서비스 기술스택/아키텍쳐로 정했다면 사실 문제가 있는것은 아니다.
- 여기서 말하고 싶은 내용은 i/o 효율 관점에서의 이야기라고 보면 좋을것 같다.

<img width="760" alt="스크린샷 2022-06-11 오후 9 14 00" src="https://user-images.githubusercontent.com/6982740/173187480-22a5421b-dade-43b8-88de-a4ae9eaefa3e.png">

- 위와 같은 request per thread model에서는 i/o를 처리하는 동안 thread가 blocking되어 대기하게 된다. thread A는 blocking된 동안 다른 요청을 처리할 수 없다.
- Java에서는 병렬 처리를 위해  thread pool을 이용해 추가 스레드를 할당하는  방법이 있지만, 부하가 높은 상태에서는 이러한 기법이 새로운 I/O 작업을 동시에 처리하는데에는 매우 비효율적일 수 있다.

### 비동기 논블로킹 모델(asynchronous and non-blocking model)

<img width="459" alt="스크린샷 2022-06-11 오후 9 23 24" src="https://user-images.githubusercontent.com/6982740/173187799-0973becf-52f5-40fe-a4da-62f3fec1f2ee.png">

- 위 그림은  사람이 SMS을 처리하는 방식에 대한 내용이다.
- 이 방식이 바로 대표적인 non-blocking 통신 방식이라고 이해할  수 있다.

### message-driven 통신
- 자원을 효율적으로 사용하기 위해서는 message-driven 통신원칙을 따라야 한다.
- 이를 수행하는 방법의 하나는 메시지 브로커(message broker)를 사용하는 것이다.
- 메시지 대기열을 관리하여 시스템 부하 관리, 탄력성을 제어할 수 있다.
  - 메시지 브로커로 kafka를 사용한다고 했을떄 consumer와 partition수를 니즈에 따라 적절히 셋팅하여 처리량을 조절할 수 있다.

### 리액티브 선언문

<img width="719" alt="7" src="https://user-images.githubusercontent.com/6982740/173187907-1c409647-fae9-4311-821c-9704757703db.png">

- 모든 비즈니스의 핵심 가치는 응답성이다.
- 응답성을 확보한다는 것은 탄력성 및 복원력 같은 기법을 따른다는 의미이다.
- 탄력성 및 복원력을 확보하는 기본적인 방법의 하나는 메시지 기반 통신을 사용하는 것이다.

## 1.3. 반응성에 대한 유스케이스

<img width="628" alt="" src="https://user-images.githubusercontent.com/6982740/173187943-dcdf6e39-a82e-4272-9d2b-cd6116a6aa24.png">

- 위 그림은 modern micro service pattern을 적용한 웹 스토어 아키텍쳐이다.
- 위치 투명성을 달성하기 위해 api gateway pattern을 사용한다.
- 서비스 요소 일부에 복제본을 구성해 높은 시스템 응답성을 얻을 수 있다.
  - 복제본 하나가 중단된 경우에 복원력을 유지할 수 있다.
- kafka을 이용해 구성한 메시지 기반 통신과 독립적인 결제 서비스 fail-over 처리도 하고 빠른 응답성을 제공한다.
  - 실제 주문 처리를 하지 않고 바로 응답할 수 있으므로 thread가 blocking 되는 시간이 줄어든다.
  - 다만 transaction이 무조건 보장되어야 하는 케이스라면 위와 같은 비동기 처리방식이 적절하지 않을수 있다. 예를 들면 은행의 계좌이체를 생각했을때 돈이 오고가는데 결과적으로 데이터 일관성이 맞춰지겠으나 시점에 따라 데이터 일관성이 꺠질수 있고, 즉각적으로 fail-over를 할수 없는 상황도 있을수 있다.(상대 은행의 점검시간인 경우 즉시 fail-over하더라도 처리할수 없고 점검이 끝날떄가 되어서야 처리가 이루어질수 있다.)


### 리액티브가 적절한 시스템 분야
- 애널리틱스(analytics)분야는 엄청난 양의 데이터를 다루면서 런타임에 처리하고 사용자에게 실시간으로 통계를 제공해야 할수도 있는데 이런 케이스에서 리액티브가 효과적일 수 있다.
- 스트리밍(streaming)이라는 효율적인 아키텍쳐를 사용할수 있다.

<img width="781" alt="" src="https://user-images.githubusercontent.com/6982740/173188183-b2354ecc-d02d-433f-94ec-a76fdc6f59cb.png">

- 가용성이 높은 시스템을 구축하려면 리액티브 선언문에서 언급한 기본 원칙을 지켜야한다.
  -   복원력 확보를 위해 배압 지원 활성화 등

## 1.4 서비스 레벨에서의 반응성
> 큰 시스템은 더 작은 규모의 시스템들로 구성되기 때문에 구성 요소의 리액티브 특성에 의존한다. 즉 리액티브 시스템은 설게 원칙을 저굥ㅇ하고 이 특성을 모든 규모에 저굥ㅇ해 그 구성요소들을 합성할 수 있게 하는것을 의미힌다. (리액티브 선언문 중)

- 따라서 구성 요소 수준에서도 리액티브 설계 및 구현을 제공하는 것이 중요하다.
- 설게 원칙이란 컴포넌트 사이의 관계, 예를 들면 각 기본 요소를 조합하는 데 사용되는 프로그래밍 기법

### 일반적인 자바의 코드 작성 : 명령형 프로그래밍(imperative programing)

<img width="539" alt="스크린샷 2022-06-11 오후 9 48 40" src="https://user-images.githubusercontent.com/6982740/173188610-e11850eb-4a18-4aec-8f5e-af9afebc830d.png">

``` java
interface ShoppingCardService {
      Output calculate(Input value);
}

class OrdersService {
    private final ShoppingCardService scService;
    void process() {
        Input input = ...;
        Output output = scService.calculate(input);
        ...
    }
}
```

- `scService.calculate()`에서 I/O 작업을 수행한다고 가정했을떄 해당 메소드를 수행하는동안 스레드는 blocking된다. orderService에서 별도의 독립적인 처리를 실행하려면 추가 스레드 할당이 필요하다.(하지만 이러하 방식은 낭비이고 리액티브 관점에 본다면 그렇게 하지 말아야 한다.

### Callback 활용 방식

``` java
interface ShoppingCardService {
      void calculate(Input value, Consumer<Output> c);
}

public class OrdersService {
    private final ShoppingCardService shoppingCardService;

    void process() {
        Input input = new Input();
        shoppingCardService.calculate(input, output -> {
            ...
        });
    }
```

- 실제 자바 코드 관점에서 thread blocking이 발생하기는 하지만 ShoppingCardService로부터 결과를 반환받지않고, OrderService가 작업을 완료 후에 반응할 콜백 함수를 미리 전달하므로 ShoppingCardService로부터 분리(decoupled)됐다고 볼수 있다.

### Thread 사용 방식

 ``` java
public class AsyncShoppingCardService implements ShoppingCardService {

    @Override
    public void calculate(Input value, Consumer<Output> c) {
        // blocking operation is presented, better to provide answer asynchronously
        new Thread(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            c.accept(new Output());
        }).start();
    }
}
```

- ShoppingCardService에서 데이터를 처리하고 파라미터로 전달된 Callback function에 그 결과를 전달하여 처리하는 방식
- 이러하 방식의 장점은 컴포넌트가 콜배 함수에 의해 분리된다는것이다.
- 다점이라면 공유 데이터 변경 등 롤백 지옥을 피하기 위헤ㅐ 개발자가 멀티 스레딩을 잘 이해해야 한다.

### Future 사용 방식

``` java
public interface ShoppingCardService {
    Future<Output> calculate(Input value);
}

public class OrdersService {
    private final ShoppingCardService shoppingCardService;

    public OrdersService(ShoppingCardService shoppingCardService) {
        this.shoppingCardService = shoppingCardService;
    }

    void process() {
        Input input = new Input();
        Future<Output> result = shoppingCardService.calculate(input);

        System.out.println(shoppingCardService.getClass().getSimpleName() + " execution completed");

        try {
            result.get();
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```

- Future을 삳용하여 결과값 반환을 지연시킬 수 있다.
- Future을 이용하면 멀티 스레드의 복잡성을 숨길 수 있지만 어쩃든 필요한 결과를 얻으려면 현재 스레드를 블로킹시키는 방식으로 처리해야만 한다. 그래서 이는 확장성을 현저히 저하시킬 수 있다.


#### CompletableFuture 사용 방식

``` java
public interface ShoppingCardService {
    CompletionStage<Output> calculate(Input value);
}

public class OrdersService {
    private final ShoppingCardService shoppingCardService;

    void process() {
        Input input = new Input();

        shoppingCardService.calculate(input)
                           .thenAccept(v -> System.out.println(shoppingCardService.getClass().getSimpleName() + " execution completed"));
       ...
    }
```

- Future와 비슷하지만 반환된 결과를 기능적 선언 방식으로 처리할 수 있다.
- 이를 통해함수형 스타일 또는 선언형 스타일로 코드를 작성할수 있고 결과를 비동기적으로 처리할 수 있게 되었다.

### CompletableFuture 등의 문제점
- 위와 같은 방식은 비동기 처리를 가능하게 하지만 Spring 자체에서 i/o와 같은 블로킹 네트워크 호출을 모두 별도의 스레드로 래핑해서 처리하기 떄문에 비효율적인 방식이다.

> Spring5에서 큰변화가 있었던 이유는 기존 Spring에서는 서블릿 3 API에 포함된 대부분의 비동기 논블로킹 기능이 잘 통합되어있었으나 Spring MVC 자체가 비동기 논블로킹 클라이언트를 제공하지 않음으로써 개선된 서블릿 API의 이점을 무효로 만들었기 떄문에 많은 부분에서 변화가 생겼다.

- 자바의 기본적인 멀티 스레딩 모델은 몇몇 스레드가 그들의 작업을 동시에 실행하기 위해 하나의 CPU를 공유할수도 있다고 가정했다.
- 이말은 즉 CPU 시간이 여러 스레드간에 공유되는것이고 이러한 처리를 위해 컨텍스트 스위칭(context switching)이 필연적으로 발생한다.
  - 스레드 컨택스트를 변경할때 레지스터, 메모리 맵 및 기타 관련요소를 저장하거나 불러오는 행위가 필요하다.
- 스레드 개수의 제한적인 요소는 일반적인 스레드 사이즈인 1MB를 기준으로 생각해볼때 request per thread model에서 64,000개의 동시성을 제공하려면 64GB의 메모리가 필요하다는 사실이다.

### 리액티브 파이프라인

<img width="787" alt="스크린샷 2022-06-11 오후 10 21 29" src="https://user-images.githubusercontent.com/6982740/173189750-e5a13bc9-298c-46bf-a44a-5b881a00ce90.png">

- 비동기 처리는 일반적인 요청-응답 패턴에만 국한되지 않는다.
- 떄로는 데이터의 연속적인 스트림으로 처리해야 할수도 있고, 정렬된 변환 흐름으로 처리해야 하는 ㄴ경우도 있을수 있다.

### 마치며
- 책의 내용을 읽고 정리하는 과정에서 느낀 부분은 주로 기존 Spring MVC의 request per thread model의 한계점에 대해 이해하고, webflux 이전 비동기 처리를 어떻게 제공했는지 이 방식에서의 한계점에 대해 간략히 다루는 내용이었습니다.
- 데이터 소스 레벨까지 리액티브가 가능하다면 이를 사용해서 스레드를 효율적으로 사용할수 있다면 베스트일것이라 생각합니다. 다만 아직 r2dbc와 같은 DB 리애틱브 드라이버가 완벽히 성숙단계에 이른것은 아니므로 RDB로 서비스하는 경우라면 기술 선정에 좀 신중을 가할 필요가 있을것이라 생각됩니다.
- 반대로 redis, mongodb 등 reactive를 잘 지원할수 있는 스토리지를 사용한다면 reactive를 도입해서 서버의 자원 효율을 증대시키는것도 좋은 방법이라 생각됩니다!

## Reference
  - [Hands-On Reactive Programming in Spring 5](https://www.packtpub.com/product/hands-on-reactive-programming-in-spring-5/9781787284951)
  - [https://github.com/PacktPublishing/Hands-On-Reactive-Programming-in-Spring-5](https://github.com/PacktPublishing/Hands-On-Reactive-Programming-in-Spring-5)
