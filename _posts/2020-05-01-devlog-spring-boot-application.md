---
title: Spring Boot + JPA + H2 Application 간단하게 띄우기
author: sungsu park
date: 2020-05-01 00:34:00 +0800
categories: [DevLog, Spring]
tags: [Java, Spring, Back-end]
---


# Spring Boot + JPA + H2 Application 간단하게 띄우기
* [1. 프로젝트 만들기](#item1)
* [2. 애플리케이션 기본 설정](#item2)
* [3. Web MVC Configuration](#item3)
* [4. JSP Configration](#item4)
* [5. JPA Configration](#item5)

## <a id="item1">1. 프로젝트 만들기</a>
### 1.1 spring initializr을 통해 프로젝트 뼈대 만들기
 -https://start.spring.io/ 에서 손쉽게 dependency 가 추가된 프로젝트 틀을 만들수 있다.
 - 하지만 이렇게 만든다고 끝은 아니고.. 추가로 이것저것 설정 등을 해주어야 한다.
<img width="1132" alt="스크린샷 2020-05-01 오전 12 40 20" src="https://user-images.githubusercontent.com/6982740/80730294-5d71d680-8b44-11ea-9af2-274a71df795f.png">

## <a id="item2">2. 애플리케이션 기본 설정</a>
 - 아래와 같이 @SpringBootApplication만 붙이면 바로 Application을 구동시킬수 있다.
``` java
@SpringBootApplication(scanBasePackageClasses = {RootConfig.class})
public class Application extends SpringBootServletInitializer {
	public static void main(String[] args) {
		SpringApplication app = new SpringApplication(Application.class);
		app.addListeners(new ApplicationPidFileWriter());
		app.run(args);
	}

	@Override
	protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
		return application.sources(Application.class);
	}
}
```

 - 하지만 일반적으로 만들고자 하는 웹 애플리케이션을 만들려면 추가적인 설정들이 필요한데 필자는 scanBasePackageClasses 속성을 이용해서 RootConfig Configuration class를 만들고 이를 기준으로 처리하도록 하였다.

``` java
@Configuration
@ComponentScan(basePackageClasses = ApplicationPackageRoot.class)
public class RootConfig {
}
```

- 여기서 특정 패키지 루트 하위에 있는 모든 bean 컴포넌트들을 스캔시키기 위해 최상단에 ApplicationPackageRoot 라는 inteface를 만들고 이 패키지 위치를 기준으로 @ComponentScan 처리하도록 했음.
 - 참고로 @Configuration annotation을 사용하면 우리가 일반적으로 AutoConfigration을 위해 사용하는 @Enable{XXX} 류의 설정용 Bean을 생성하여 사용할 수 있다.

- 이런식으로 프로젝트 구조를 잡고 설정을 했을떄 패키지 구조를 살펴보면 다음과 같다.
<img width="329" alt="스크린샷 2020-05-01 오전 12 48 32" src="https://user-images.githubusercontent.com/6982740/80731219-952d4e00-8b45-11ea-96d4-5343e7cee679.png">

## <a id="item3">3. Web MVC Configuration</a>
 - 먼저 dependency를 추가하자.

``` xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
```
- EnableWebMvc를 통해 기본적인 mvc 설정은 모두 커버가 된다. 자세한 내용이 알고 싶다면 DelegatingWebMvcConfiguration.class 파일을 참조하도록 하자

``` java
@EnableWebMvc
public class WebMvcConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/bundle/**")
                .addResourceLocations("/static/bundle/").setCachePeriod(3600)
                .resourceChain(true).addResolver(new PathResourceResolver());
    }
}
```

## <a id="item4">4. JSP Configration</a>
 - jsp를 사용하려면 일단 pom.xml에 dependancy를 추가해주어야 한다.

``` xml
<!-- jsp -->
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-tomcat</artifactId>
	<scope>provided</scope>
</dependency>
<dependency>
	<groupId>org.apache.tomcat.embed</groupId>
	<artifactId>tomcat-embed-jasper</artifactId>
	<scope>provided</scope>
</dependency>
<dependency>
	<groupId>javax.servlet</groupId>
	<artifactId>jstl</artifactId>
</dependency>
```

 - 위 디펜던시를 추가하지 않으면 jsp 파일을 찾을수 없다고 404 Page Not Found를 맞이할 수 있을 것이다.
 - 그리고 템플릿 엔진으로 jsp를 사용하기 위해 필자가 추가로 한 부분은 internalResourceViewResolver Bean을 추가한 것이다.
``` java
@Configuration
@EnableWebMvc
public class WebMvcConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/bundle/**")
                .addResourceLocations("/static/bundle/").setCachePeriod(3600)
                .resourceChain(true).addResolver(new PathResourceResolver());
    }

    @Bean
    public ViewResolver internalResourceViewResolver() {
        final InternalResourceViewResolver viewResolver = new InternalResourceViewResolver();
        viewResolver.setPrefix("/WEB-INF/templates/");
        viewResolver.setSuffix(".jsp");
        return viewResolver;
    }
}
```
 - 이를 통해 jsp web 페이지를 랜더링할 수 있다. 여기서 이런 방식 말고 다른 방법으로 처리할수도 있는데 그 방법은 application.yml 파일에 설정을 추가해주는 것이다.
``` yml
spring:
  mvc:
    view:
      prefix: /WEB-INF/templates/
      suffix: .jsp
```

## <a id="item5">5. JPA Configration</a>
### 5.1 DataSource
 - 실제 웹 애플리케이션을 운영하는 것이 아니라 샘플 프로젝트를 만들고 연습하기에 가장 쉬운 방법은 H2 in memory DB를 사용하는 것일 것이다. 이를 사용하기란 매우 간단하다.
 - 먼저 h2 dependency를 pom.xml에 추가한다.

``` xml
<!-- db -->
<dependency>
	<groupId>com.h2database</groupId>
	<artifactId>h2</artifactId>
	<scope>runtime</scope>
</dependency>
```

 - PersistenceConfigration에서 DataSource  Bean을 정의한다.
``` java
@Bean
public DataSource dataSource() {
	EmbeddedDatabaseBuilder builder = new EmbeddedDatabaseBuilder();
	return builder.setType(EmbeddedDatabaseType.H2).build();
}
```
 - 이제 H2를 사용할 준비는 끝났다. 이제 다음으로 넘어가서 JPA 설정을 해보자

### 5.2 JPA
 - 여기서 JPA라고 이야기했지만, 실제로는 JPA 구현체 중에서 유명한 것 중 하나인 hibernate를 사용할 것이다.
 - 이를 사용하기 위해서는 먼저 dependency를 추가해주어야 한다.

``` xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
	<groupId>org.hibernate</groupId>
	<artifactId>hibernate-ehcache</artifactId>
	<version>5.4.14.Final</version>
</dependency>
```

 - 그리고 이와 관련된 configuration 값들을 정의해준다.
``` yml
spring:
  datasource:
    driver-class-name: org.h2.Driver
    url: jdbc:h2:mem:test;DB_CLOSE_DELAY=-1
    username:
    password:
  h2:
    console:
      enabled: true
      path: /h2
  jpa:
    database-platform: org.hibernate.dialect.H2Dialect
    database: H2
    generate-ddl: false
    open-in-view: false
    show-sql: true
    hibernate:
      ddl-auto: create
      naming:
        physical-strategy: org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl
        implicit-strategy: org.hibernate.boot.model.naming.ImplicitNamingStrategyLegacyJpaImpl
    properties:
      hibernate:
        format_sql: true
        use_sql_comments: true
        cache.use_second_level_cache: true
        cache.use_query_cache: false
        generate_statistics: true
        cache.region.factory_class: org.hibernate.cache.ehcache.EhCacheRegionFactory
```

 - 이 외에도 hibernate 설정은 아주 다양하게 많이 있을텐데, 자세한 것은 공식 문서를 참조하도록 하자. ( https://hibernate.org/orm/documentation/5.4/ )
 - 사실 위에서 가장 중요한 부분은 "hibernate.ddl-auto" 설정 값인데 이 값은 만약 실제 서비스에서 사용하고자 한다면 "validate" 정도의 설정값을 사용하는게 좋다. 하지만 예제에서는 인메모리 DB를 사용하는 특성상 create를 사용한것이니 참고하자.

 - application.yml에 설정값을 넣어두었다면 이제 Java Configration 을 정의해보자
``` java
@Configuration
@EnableJpaRepositories(basePackages = "com.sungsu.boilerplate.app")
@EnableTransactionManagement
public class PersistenceConfig {

	@Bean
	public DataSource dataSource() {
		EmbeddedDatabaseBuilder builder = new EmbeddedDatabaseBuilder();
		return builder.setType(EmbeddedDatabaseType.H2).build();
	}

	@Bean
	public PlatformTransactionManager transactionManager(EntityManagerFactory entityManagerFactory) {
		JpaTransactionManager txManager = new JpaTransactionManager();
		txManager.setEntityManagerFactory(entityManagerFactory);
		return txManager;
	}
}
```

 - @EnableJpaRepositories의 basePackages 하위에 있는 @Entity 와 @Repository를 자동 등록이 된다.
 - EnableJpaRepositories를 보면 default로 "transactionManager" 를 사용하기 때문에  PlatformTransactionManager Type의 클래스로 bean 등록을 해주면 된다.
 - 이렇게 하면 H2 + JPA application이 완성되었다.

## Result
### console
<img width="1209" alt="스크린샷 2020-05-01 오전 1 32 14" src="https://user-images.githubusercontent.com/6982740/80735659-fe17c480-8b4b-11ea-8bf1-e030746ad91a.png">

### web
<img width="558" alt="스크린샷 2020-05-01 오전 1 32 20" src="https://user-images.githubusercontent.com/6982740/80735415-9ceff100-8b4b-11ea-9fc9-c830b108c71a.png">

## Comment
 - 작성된 내용 중 혹시 제가 잘못 알고 있는 내용이 있다면 언제든지 코멘트 부탁드립니다.
 - 이 포스팅을 쓰면서 만든 샘플코드는 github에 올려두었다. 혹시 신규 프로젝트를 시작하고자할떄 필요하다면 언제든지 사용하셔도 됩니다.
   - [https://github.com/sungsu9022/sprping-boot-boilerplate](https://github.com/sungsu9022/sprping-boot-boilerplate)
