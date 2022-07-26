---
title: "[Hands-On Reactive Programming in Spring 5] 9. 리액티브 애플리케이션 테스트하기"
author: sungsu park
date: 2022-07-26 12:28:00 +0800
categories: [DevLog, Spring]
tags: [Java, Spring, Hands-On Reactive Programming in Spring 5, Reactive]

---

# 9. 리액티브 애플리케이션 테스트하기
> 테스트 도구에 대한 요구사항
> StepVerifier를 사용해 Publisher를 테스트할 때의 핵심
> 고급 StepVerifier 사용 시나리오
> 웹플럭스 테스트를 위한 툴 세트

## 9.1 리액티브 스트림을 테스트하기 어려운 이유
- 대규모 시스템에 많은 수의 클래스가 포함된 수많은 컴포넌트가 있고, 테스트 피라미드(Test Pyramid) 제안을 따라야 모든 것을 재대로 검증할 수 있다.
- 리액티브 프로그래밍이 리소스 최적화에 도움을 주고, 리액터를 사용할 경우 지저분한 코드(콜백 지옥 등)에도 유용하지만 테스트하는 데에는 어려움이 있다.
- Publisher 인터페이스를 사용해 스트림을 게시하고 Subscriber 인터페이스를 구현하고 게시자의 스트림을 수집해서 그 정확성을 확인할 수 있는데, 이 방식을 테스트로 검증하는게 쉽지 안흥ㅁ.

### Test Pyramid ( https://eunjin3786.tistory.com/86 )

<img width="553" alt="스크린샷 2022-07-25 오후 4 44 17" src="https://user-images.githubusercontent.com/6982740/180724776-b2eab974-d42c-405b-8c5c-fc1d45744171.png">

- Y축 위로 올라갈 수록 실제로 돌아가는, 유저가 쓰는 영역이라 믿을 만한 영역입니다.  하지만 실행시간이 오래 걸리고 유지보수, 디버깅 하기 더욱 어려운 영역입니다.
- X축은 밑으로 내려 올 수록 더 많은 테스트 코드를 작성하게 됩니다.
- 피라미드 모델 접근법은 철저성(thoroughness), 품질(quality), 실행 속도(execution speed) 사이의 균형을 잡는 데 도움이 된다고 합니다. 그리고 이 피라미드는 세 가지 다른 테스트 유형 간의 균형을 유지하는 것도 도와준다고 합니다.


## 9.2 StepVerifier를 이용한 리액티브 스트림 테스트
- [Reactor Test](https://github.com/reactor/reactor-core) 모듈을 사용하여 리액티브 스트림을 테스트해봅시다.

### 9.2.1 StepVerifier의 핵심요소
#### `StepVerifier <T> create (Publisher<T> Soruce)`
- 원소가 올바른 순서로 생성되는지를 확인
- 빌더 기법을 사용하여 이벤트가 발생하는 순서를 정의하여 검증할 수 있다.
- 검증을 실행하려면 `.verify()` blocking 호출 메소드를 호출해야 함.

``` java
StepVerifier
	.create(Flux.just("foo","bar"))
	.expectSubscription()
	.expectNext("foo")
	.expectNext("bar") // stream 순서가 다르면 Error 발생
	.expectComplete()
	.verify();

StepVerifier
	.create(Flux.range(0,100))
	.expectSubscription()
	.expectNext(0)
	.expectNextCount(98)
	.expectNext(99)
	.expectComplete()
	.verify();
```

- StepVerifier를 사용하면 자바 Hamcrest(Matcher)와 같은 도구를 사용해 스트림 데이터 기록과 검증을 한번에 할 수 있음.

``` java
Publisher<Wallet> usersWallets = findAllUsersWallets();
StepVerifier
      .create(usersWallets)
      .expectSubscription()
      .recordWith(ArrayList::new)
      .expectNextCount(1)
      .consumeRecordedWith(wallets -> assertThat(wallets, everyItem(hasProperty("ownner", equalTo("admin")))))
      .expectComplete()
      verify();
```

- 기대값에 대해 기본 builder method를 통해서도 검증할 수 있음.

``` java
StepVerifier
	.create(Flux.just("alpha-foo", "betta-bar"))
	.expectSubscription()
	.expectNextMatches(e -> e.startsWith("alpha"))
	.expectNextMatches(e -> e.startsWith("betta"))
	.expectComplete()
	.verify();
```

- `assertNext()`, `consumeNextWith()` 등이 있음.

- Error 발생에 대한 검증
 ``` java
		StepVerifier
			.create(Flux.error(new RuntimeException("Error")))
			.expectError()
			.verify();
```


### 9.2.2 StepVerifier을 이용한 고급 테스트
- 리액티브 스트림 스펙에 따르면 무한 스트림은 `Subscriber#onComplete()` 메소드를 호출하지 않는데 이런 경우 이전에 학습한 테스트 검증을 사용할 수 없음.
- 이러한  처리를 위한 구독 취소 API를 제공합니다.

``` java
		Flux<String> infiniteStream = ... // 무한스트림
		StepVerifier
			.create(infiniteStream)
			.expectSubscription()
			.expectNext("Connected")
			.expectNext("Price: $12.00")
			.thenCancel()
			.verify();
```

#### BackPressure 검증
- `thenRequest()`를 통해 BackPressure를 검증할수 있다. 하지만 이 경우 어보플로가 발생할 수 있음.


``` java
		Flux<String> infiniteStream = Flux.just("Connected","Price: $12.00","3","4","5","6","7");

		StepVerifier
			.create(infiniteStream.onBackpressureBuffer(5), 0)
			.expectSubscription()
			.thenRequest(1)
			.expectNext("Connected")
			.thenRequest(1)
			.expectNext("Price: $12.00")
			.expectError(Exceptions.failWithOverflow().getClass())
			.verify();
```

- `then()`을 통해 특정 검증 후에 추가 작업을 실행할 수 있다.

``` java
		TestPublisher<String> idsPublisher = TestPublisher.create()

		StepVerifier
			.create(walletsRepository.findAllByid(idsPublisher))
			.expectSubscription()
			.then(() -> idsPublisher.next("1"))
			.assertNext(w -> assertThat(w, hasProperty("id", equalsTo("1"))))
// ...
// ...
			.expectComplete()
			.verify();
```

### 9.2.3 가상 시간 다루기

``` java
		Flux<String> sendWithInterval = Flux.interval(Duration.ofMinutes(1))
				.zipWith(Flux.just("a","b","c"))
				.map(Tuple2::getT2);

		StepVerifier
			.create(sendWithInterval)
			.expectSubscription()
			.expectNext("a", "b","c")
			.expectComplete()
			.verify();
```

- 위 코드를 테스트하려면 평균 테스트 시간이 3분이 넘게 걸림.
-  `Flux<String> sendWithInterval`이 1분씩 지연시키기 때문인데 `withVirtualTime()` build 메소드를 사용하여 이를 개선할 수 있다.

``` java
@Test
		StepVerifier
			.withVirtualTime(() -> Flux.interval(Duration.ofMinutes(1))
				.zipWith(Flux.just("a", "b", "c"))
				.map(Tuple2::getT2))
			.expectSubscription()
//			.then(() -> VirtualTimeScheduler.get().advanceTimeBy(Duration.ofMinutes(3)))
			.thenAwait(Duration.ofMinutes(3))
			.expectNext("a", "b","c")
			.expectComplete()
			.verify();
```

- `VirtualTimeScheduler` API와 함께 `.then()`을 사용해 특정 시간만큼 시간을 처리할 수 있다. 이는 `.thenAwait()`로 대체할 경우에도 동일한 효과를 얻을 수 있음.

### 9.2.4 리액티브 컨텍스트 검증하기
- 리액터 컨텍스트르 검증하기 위해서 다음과 같은 방식을 제공

``` java
		StepVerifier
			.create(securityService.login("admin", "admin"))
			.expectSubscription()
			.expectAccessibleContext()
			.hasKey("security")
			.then()
			.expectComplete()
			.verify();
```

### 9.2.5 웹플럭스 테스트
> 교재에 있는 예제를 동작시켜보려 했으나 잘안됩니다.ㅠ
> m1 mac에서는 embed mongo를 사용할수 없으며, local환경에서 docker-compose를 통해 mongo를 띄우고 처리하려고 해도 예제코드의 webClient api 호출을 위한 서버는 동작하지 않아서 api call 후에 reactive mongo repo에 데이터가 셋되는것을 확인이 잘 안됨.
> api를 localhost에 만들고 처리하는것도 해봤는데 뭔가 잘안되서 스킵..

### 9.2.6 WebTestClient를 이용해 컨트롤러 테스트하기
``` java
@Test
	public void verifyPaymentsWasSentAndStored() {
		Mockito.when(exchangeFunction.exchange(Mockito.any()))
		       .thenReturn(Mono.just(MockClientResponse.create(201, Mono.empty())));

		client.post()
		      .uri("/payments/")
		      .syncBody(new Payment())
		      .exchange()
		      .expectStatus().is2xxSuccessful()
		      .returnResult(String.class)
		      .getResponseBody()
		      .as(StepVerifier::create)
		      .expectNextCount(1)
		      .expectComplete()
		      .verify();

		Mockito.verify(exchangeFunction).exchange(Mockito.any());
	}
```

- WebMVC와 비슷한 방식으로 웹플럭스 애플리케이션에서 HTTP 통신을 통한 외부 호출을 모킹하는 것이 가능함.
- 이 경우 모든 HTTP 통신이 WebClient로 구현된다는 가정하에 테스트가 수행되고, http client 라이브러리가 변경되면 더이상  테스트는 유효하지 않게 된다.


### 9.2.7 웹소켓 테스트
> 교재에 있는 예제를 동작시키기까지 하려면 mock server를 만들고 작업을 해야하는데 이 부분까지는 시간이 너무 많이 들것으로 보여 스킵하고 간략한 코드 작성법만 살펴봅니다.

- 웹플럭스는 웹소켓 API 테스트를 위한 솔루션을 제공하지 않는다.
- 서버에 연결할 수 있다하더라도 StepVerifier를 사용해 수신 데이터를 확인하는 것이 어려움.

``` java
	@Test
	@WithMockUser
	public void checkThatUserIsAbleToMakeATrade() {
		URI uri = URI.create("ws://localhost:8080/stream");
		TestWebSocketClient client = TestWebSocketClient.create(uri);
		TestPublisher<String> testPublisher = TestPublisher.create();
		Flux<String> inbound = testPublisher.flux()
			.subscribeWith(ReplayProcessor.create(1))
			.transform(client::sendAndReceive)
			.map(WebSocketMessage::getPayloadAsText);

		StepVerifier
			.create(inbound)
			.expectSubscription()
			.then(() -> testPublisher.next("TRADES|BTC"))
			.expectNext("PRICE|AMOUNG|CURRENCY")
			.then(() -> testPublisher.next("TRADE: 10123|1.54|BTC"))
			.expectNext("10123|1.54|BTC")
			.then(() -> testPublisher.next("TRADE: 10090|-0.01|BTC"))
			.expectNext("10090|-0.01|BTC")
			.thenCancel()
			.verify();
	}
```

- 웹소켓 API 검증과 함께 WebSocketClient와 외부 서비스 간의 상호 작용을 모킹해야 할 수도 있습니다. 이런 상호 작용을 모킹하는것은 어려움(일반적인 `WebSocketClient.build` 가 없기 때문)
- 결과적으로 이런 케이스를 현재 하려고한다면 목 서버를 만들어서 통합테스트 코드에서 목서버를 접속하는 방법뿐입니다.

## Reference
- [Test Pyramid](https://eunjin3786.tistory.com/86)
- [Reactor Test Source Code](https://github.com/reactor/reactor-core)
- [Reactor Test Example Blog](https://www.baeldung.com/reactive-streams-step-verifier-test-publisher)
- [Reactor Test Example](https://www.tabnine.com/code/java/methods/reactor.test.StepVerifier$Step/assertNext)
- [Source Code](https://github.com/sungsu9022/study-Hands-On-Reactive-Programming-in-Spring-5)
