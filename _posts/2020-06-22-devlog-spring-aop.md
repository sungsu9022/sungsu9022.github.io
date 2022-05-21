---
title: Spring AOP 이야기 - AOP는 언제 써야할까?
author: sungsu park
date: 2020-06-22 16:34:00 +0800
categories: [DevLog, Spring]
tags: [Java, Spring]
---


# Spring AOP
> Spring을 공부하다보면 필연적으로 AOP를 만나게 되는데, 개발자들은 이걸 언제 쓰는게 좋을지 고민하곤 하는데요. 관련해서 사용방법과 언제 써야할지를 한번 알아보겠습니다.

## 1. AOP란?
- 부가기능 모듈을 객체지향 기술에서 주로 사용하는 오브젝트와는 다르게 특별한 이름으로 부르기 시작한 것
- 그 자체로 애플리케이션의 핵심 기능을 담고 있지는 않지만, 애플리케이션을 구성하는 중요한 한 가지 요소이고, 핵심 기능에 부가되어 의미를 갖는 특별한 모듈
 - 어드바이저는 가장 단순한 형태의 애스팩트AOP(애스펙트 지향 프로그래밍, Aspect Oriented Programming)
 - OOP를 돕는 보조적인 기술일뿐, OOP를 대체하는 새로운 개념이 아니다.
 - 부가기능이 핵심기능 안으로 침투해버리면 핵심기능 설계에 객체지향 기술의 가치를 온전히 부여하기 힘들어짐. AOP는 애스펙트를 분리함으로써 핵심기능을 설계하고 구현할 때 객체지향적인 가치를 지킬 수 있도록 돕는 것.

## 2. 언제 AOP를 사용해야 할까?
### 2.1 매번 반복되는 개발 패턴들
``` java

	private final SomethingService service;
	private final SomethingRepository repository;

	@Transactional
	public void doSomething() {

		if(isCondition()) {
			// 무언가를 한다.
		}

		if(isCondition2()) {
			// 무언가를 한다.
		} else {

		}

		if(isCondition3()) {
			if(isCondition4()) {
				for(Object item : ItemList) {
					// 무언가를 한다.
				}
			} else {
				// 무언가를 한다.
			}
		}
	}
```

 - Spring에서 제공하고자 했던 그 기술의 철학과 가치에 부합되는 방식으로 개발하자.
 - 객체지향적인 고민(?)을 더하다.
 - 실제로 객체지향의 핵심인  SOLID 원칙 중 단일책임 원칙을 위배하면서 개발하는 경우는 상당수
    - 사실 이렇게 개발하게 된 이유는 현실적으로 이것 하나 떄문에 이렇게까지 해야 하나에 대한 고민도 있을것이고, if문 하나 추가하면 오히려 가독성  측면이나 유지보수 측면에서 더 좋기 떄문이다.

### 2.2 그러면 언제 AOP를 써야하는가?
 - 반복적으로 동일한 코드를 여기저기에 개발 적용해야 하는 경우(최우선)
 - 자신의 설계에서 A클래스에 부여한 책임을 넘어서는 수준의 일을 처리해야 하는 경우
    - 이 부분은 반복적으로 동일한 코드가 계속 들어가야 하는 경우의 전제조건이 있어야 한다. 만약 하나의 영역에서만 사용되는 코드를 굳이 AOP로 만드는것은 다른사람의 코드 추적을 어렵게 만들지도 모른다.
 - 해당 처리 로직이 클래스에서 처리해야 하는 핵심로직이냐 아니면 부가 기능이냐를 기준으로 잘 판단해서 부가기능인 경우
    - 이 판단이 사실 명확한 근거는 없다. 개인과 팀이 상황에 따라 적절히 판단하는게 좋을것 같다.

### 2.3 Sevlet Filter vs Interceptor vs AOP
 - AOP에 대해서 자세히 알아보기 전에 먼저 Servlet Filter, Intercpetor, AOP에 대해서 알아볼 필요가 있다.
 - Spring 웹 애플리케이션을 개발하면서 Filter, Interceptor는 심심찮게 접했을 것이다.
 - 동작 방식은 사실 이 3가지 모두 비슷한 점이 있다.
 - 차이가 있다면 처리되는 시점의 차이가 있다.
     - Servlet Filter는 request 최앞단, 최말단에서 동작하고,
     - Interceptor는 Dispatcher Servlet 다음 단계에서 동작하고
     - AOP는 개발자가 직접 처리시점을 정할수 있다.

![filter interceptor](https://user-images.githubusercontent.com/6982740/85278169-86b54000-b4bf-11ea-8590-993ea1951008.png)

## 3. AOP 사용하기
### 3.1 Spring boot Project 설정
 - aop 라이브러리 dependency를 추가한다.

``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>

```

### 3.2 AOP Configration
 - Configuration클래스 혹은 Application Class에 @EnableAspectJAutoProxy 을 추가해준다.
 - 나는 별도의 Configration도 각 목적에 따라 나누어 처리하는것을 선호해서 별도의 RootConfig.class를 두었고, 그쪽에 선언하였다.

``` java
@Configuration
@EnableAspectJAutoProxy
public class RootConfig {
}

@SpringBootApplication(scanBasePackageClasses = {RootConfig.class})
public class Application extends SpringBootServletInitializer {
	public static void main(String[] args) {
		SpringApplication app = new SpringApplication(Application.class);
		app.addListeners(new ApplicationPidFileWriter());
		app.run(args);
	}
```

### 3.3 @EnableAspectJAutoProxy
 - @EnableAspectJAutoProxy 는 다른 Enable류와 마찬가지로 auto configration annotation이다.
 - AOP와 관련된 @Aspect와 같은 annotation을 사용할수 있도록 해준다.
 - xml에서는 ```<aop:aspectj-autoproxy>```와 동일한 효과라고 생각하면 된다.
 - @EnableAspectJAutoProxy의 proxyTargetClass attribute가 있는데 이 부분에 대해서도 간략히 알고 있으면 좋다.

### 3.4 EnableAspectJAutoProxy.proxyTargetClass
 - 이 부분은 Spring의 역사 이야기를 조금 해야 하는데 proxy 방식에는 JDK Dynamic Proxy방식과 CGLIB 방식이 있다는 것을 알아야 한다.
 - JDK Dynamic Proxy은 인터페이스를 구현해서 구현된 메소드에만 AOP를 적용할수 있는 방법이었어서 이 방식으로 proxy 처리를 하면 인터페이스에 정의된 메소드에만 proxy를 적용할 수 있다.
 - CDLIB는 대상 클래스를 상속받아서 프록시 객체를 구현하는 방식으로 클래스가 final만 아니라면 proxy를 적용할수 있다.
 - proxyTargetClass 이 옵션은 저 방식을 무엇으로 할건지에 대한 것이다.
 -  proxyTargetClass의 기본값은 false인데, 그러면 기본이 JDK Dynamic Proxy인것을 알수 있다.
 - 하지만 가장 최근 문서를 읽어보니면 프록시 대상 클래스가 인터페이스를 구현하지 않으면 기본적으로 CGLIB 프록시가 적용된다고 합니다.(spring 5.2 공식 문서 기준)

### 3.5 AOP Aspect 클래스 정의
- Aspect를 정의하고 어느시점에 처리할지에 대한 부분은 @Around, @Before, @After가 있다.
    - @Around는 메소드의 실행 전 / 후의 로직을 컨트롤할수 있다.
    - @Before, @After은 말 그대로  메소드 실행 전 / 후이다.
- 서비스 비즈니스 로직 개발하는 입장에서는 @Around을 이용해야 하는 케이스가 제일 많았다.

> 로컬환경에서만 동작해야 하는 스펙이 있다고 가정하고 AOP를 만들어보자.
> 동작하는 스펙은 편의상 String을 return하는 메소드 시그니쳐에 DummyString을 return 한다고 가정한다.

``` java
@Aspect
@Component
@Slf4j
@RequiredArgsConstructor
public class MainAspect {
	private final EnvConfig envConfig;

	@Around("execution(String com.sungsu.boilerplate.app.main.MainService.*(..)) && @annotation(com.sungsu.boilerplate.app.main.aspect.ReturnDummy)")
	public Object returnDummy(ProceedingJoinPoint joinPoint) throws Throwable {
		if (!isLocalEnvironment()) {
			return joinPoint.proceed();
		}

		// local 환경인 경우 본  메소드를 수행하지 않고 dummy value를 return
		log.info("returnDummy");
		return "returnDummy";
	}

	/**
	 * 로컬 환경인지 확인
	 * @return
	 */
	private boolean isLocalEnvironment() {
		return envConfig.isLocal();
	}
}
```

### 3.6 Aspect 동작 시점 정의
 - 어느 시점에 Aspect를 동작시킬지에 대한 방법은 다양한데, 대표적으로 포인트컷 표현식이 있다.
 - 하지만 위 3.5 예제에서는 보다시피 포인트컷 표현식(@execution) 과 @annotation을 and 조건으로 사용하고 있다.
 - 왜 Annotation을 이용해서 AOP를 적용했을까?
- 포인트컷 표현식을 이용한 AOP를 적용했을때의 문제

``` java
@Aspect
public class TestAspect {
	@Pointcut("execution(* transfer(..))")// 포인트컷 표현식
	private void anyOldTransfer() {}// 포인트컷 시그니처
}
```

- AOP 적용을 위해서 포인트컷 표현식만을 이용한다고 했을 때 AOP가 적용된 메소드에서는 이 메소드에 AOP를 걸었는지 아닌지를 한눈에 파악하기가 어렵다.
- 물론 IDE에서는 네비게이션을 제공해주기는 하지만 github을 이용해서 코드리뷰를 한다던지 했을때 이를 쉽게 파악할 수 없다.
- 대표적인 AOP 예인 @Transactional와 같은 경험을 얻기 위해서 위와 같이 처리한 것이다.
- 자세한 내용은 여기서는 다루지 않고 참조 링크에서 확인 바란다. ( [https://blog.outsider.ne.kr/843](https://blog.outsider.ne.kr/843) )

### 3.7 annotation 정의하기

- Meta Annotations

|             |                                                                                                   |
|-------------|---------------------------------------------------------------------------------------------------|
| @Retention  | 어노테이션이 적용되는 범위(?), 어떤 시점까지 어노테이션이 영향을 미치는지 결정(코드, 클래스로 컴파일, 런타임에 반영 등) |
| @Documented | 문서에도 어노테이션의 정보가 표현됩니다.                                                          |
| @Target     | 어노테이션이 적용할 위치를 결정합니다.                                                            |
| @Inherited  | 이 어노테이션을 선언하면 자식클래스가 어노테이션을 상속 받을 수 있습니다.        |
| @Repeatable | 반복적으로 어노테이션을 선언할 수 있게 합니다.                                                    |

``` java
import java.lang.annotation.*;

@Inherited
@Documented
@Retention(RetentionPolicy.RUNTIME) // 컴파일 이후에도 JVM에 의해서 참조가 가능합니다.
//@Retention(RetentionPolicy.CLASS) // 컴파일러가 클래스를 참조할 때까지 유효합니다.
//@Retention(RetentionPolicy.SOURCE) // 어노테이션 정보는 컴파일 이후 없어집니다.
@Target({
        ElementType.PACKAGE, // 패키지 선언시
        ElementType.TYPE, // 타입 선언시
        ElementType.CONSTRUCTOR, // 생성자 선언시
        ElementType.FIELD, // 멤버 변수 선언시
        ElementType.METHOD, // 메소드 선언시
        ElementType.ANNOTATION_TYPE, // 어노테이션 타입 선언시
        ElementType.LOCAL_VARIABLE, // 지역 변수 선언시
        ElementType.PARAMETER, // 매개 변수 선언시
        ElementType.TYPE_PARAMETER, // 매개 변수 타입 선언시
        ElementType.TYPE_USE // 타입 사용시
})
```

- 추가적인 이해를 위해서 @Transactional 을 인용하자면

``` java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {
    @AliasFor("transactionManager")
    String value() default "";

    @AliasFor("value")
    String transactionManager() default "";

    Propagation propagation() default Propagation.REQUIRED;

    Isolation isolation() default Isolation.DEFAULT;

    int timeout() default -1;

    boolean readOnly() default false;

    Class<? extends Throwable>[] rollbackFor() default {};

    String[] rollbackForClassName() default {};

    Class<? extends Throwable>[] noRollbackFor() default {};

    String[] noRollbackForClassName() default {};
}
```

 - 예제코드에 사용한 annotation은 간단하다. 이정도만으로도 원하는 형태로 충분히 사용 가능하다.

``` java
@Target({ElementType.METHOD, ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface ReturnDummy {
}
```

 - annotation을 통해 특정 값을 전달할수도 있다.

``` java
@Target({ElementType.METHOD, ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface ReturnDummy {
	String data() default "";
}


	@ReturnDummy(data = "test")
	public String method1() {
		log.info("method1");
		return "method1";
	}
```

 - 혹시 필요하다면 el expression으로 전달할수도 있으니 참고만 하도록 하자.


### 3.8 AOP 메소드에 적용하기
 - 애노테이션과 Aspect를 정의했다면 이제 Service Layer 같은곳에 적용할 차례이다.

``` java
// MainController.java
@Controller
@RequiredArgsConstructor
public class MainController {
	private final MainService mainService;

	@GetMapping(value = {"/", "/main"})
	public String main() {
		mainService.method1();
		mainService.method2();
		mainService.method3();
		return "index";
	}
}

// MainService.java
@Service
@RequiredArgsConstructor
@Slf4j
public class MainService {

	@ReturnDummy(data = "test")
	public String method1() {
		log.info("method1");
		return "method1";
	}

	@ReturnDummy(data = "test")
	public String method2() {
		log.info("method2");
		return "method2";
	}

	@ReturnDummy(data = "test")
	public String method3() {
		log.info("method3");
		return "method3";
	}
}
```

 - 이렇게 적용하면 로컬환경에서는 각 method1,2,3을 호출해도 해당 메소드는 호출되지 않고 AOP에 정의한 "dummyReturn" 값만을 return 하게 될 것이다.
- annotation을 정의할떄 @Target을  METHOD, TYPE 2가지를 보통 정의했는데 위 예제는 메소드 각각에 정의했지만 Class에 바로 적용도 가능한다.

``` java
@Service
@RequiredArgsConstructor
@Slf4j
@ReturnDummy(data = "test")
public class MainService {


	public String method1() {
		log.info("method1");
		return "method1";
	}

	public String method2() {
		log.info("method2");
		return "method2";
	}

	public String method3() {
		log.info("method3");
		return "method3";
	}
}
```

 - 다만 클래스에 적용하는것보단 메소드단위로 적용하는것이 개발할때 더 편리하다.(IntelliJ IDEA 기준으로 method에 정의한 annotation에서는 trace가 가능한 마킹이 생기는것을 알수 있다.)

<img width="509" alt="스크린샷 2020-06-22 오후 8 23 46" src="https://user-images.githubusercontent.com/6982740/85282236-5c1ab580-b4c6-11ea-965a-bc32945841ef.png">

<img width="487" alt="스크린샷 2020-06-22 오후 8 24 00" src="https://user-images.githubusercontent.com/6982740/85282241-5de47900-b4c6-11ea-89d1-4c80301320a8.png">


### 3.9 AOP 테스트 코드 작성
 - AspectJProxyFactory를 이용해서 aspect를 주입해주고 Proxy 객체로 덮어 씌워주면 된다.
 - 예제는 아래를 참고하면 된다.

``` java
@RunWith(MockitoJUnitRunner.class)
public class MainServiceTest {

	@InjectMocks
	private MainService mainService;
	@InjectMocks
	private MainAspect aspect;
	@Mock
	private EnvConfig envConfig;


	@Before
	public void setUp() {
		AspectJProxyFactory factory = new AspectJProxyFactory(mainService);
		factory.addAspect(aspect);
		factory.setProxyTargetClass(true);
		mainService = factory.getProxy();
	}

	@Test
	public void method1() {
		Mockito.when(envConfig.isLocal()).thenReturn(true);
		assertTrue(MainAspect.DUMMY_VALUE.equals(mainService.method1()));
	}

}
```


## 4. AOP에 대한 개인 생각
 - 나는 실제 서비스 개발을 하면서 AOP를 꽤 자주 사용하는 편이다.
 - AOP를 써야할지 말지를 판단할떄는 위에 언급한것처럼 각각의 조건을 충족할떄 하는것이 좋다.
 - AOP가 만능은 아니며, 오히려 유지보수성을 떨어뜨릴지도 모른다. 적절한 상황을 잘 판단하여 사용하도록 하자.
 - 내가 AOP를 실무에 적용한 케이스
    - A/B테스트 적용을 위해 특정 유저군들에만 한시적으로 기능을 오픈하기 위한 방식으로 사용
    - 성능 테스트시에 다른 서비스(서버 단위)에 영향을 주지 않기 위해 더미 처리를 하기 위해 사용
    - 특정 환경에서만 노출되어야 할 부가정보를 추가해주기 위해
    - 모델의 Deprecated된 필드가 있는데(DB에서 제거) 구버전 앱 지원을 위해 상위 레벨에서는 해당 데이터가 필요한 상황이 있어서 이 값을 바인딩해주는 역할로써 AOP 사용
    - Redis 기반 Lock 적용시 AOP 사용
    - 이 외 임시로 특정기간동안만 추가되는 로직이 있을때 한시적으로 AOP를 적용해서 처리하고, 그 특정기간이 종료되면 Aspect와 Annotation을 제거하여 쉽게 제거하기 위한 목적 등

## 5. 예제 코드
 - [https://github.com/sungsu9022/spring-boot-boilerplate/commit/bcc91cc94b7f8fe5c233bbf9efe154fb7a00b6e7](https://github.com/sungsu9022/spring-boot-boilerplate/commit/bcc91cc94b7f8fe5c233bbf9efe154fb7a00b6e7)
 - [https://github.com/sungsu9022/spring-boot-boilerplate](https://github.com/sungsu9022/spring-boot-boilerplate)

## Reference
 - [https://docs.spring.io/spring/docs/5.2.0.M1/spring-framework-reference/core.html#aop](https://docs.spring.io/spring/docs/5.2.0.M1/spring-framework-reference/core.html#aop)
 - [https://blog.outsider.ne.kr/843](https://blog.outsider.ne.kr/843)
