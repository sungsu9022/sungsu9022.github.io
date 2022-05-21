---
title: Spring Boot JSP 자동 reload 되지 않는 문제 해결 방법
author: sungsu park
date: 2020-05-03 16:34:00 +0800
categories: [DevLog, Spring]
tags: [Java, Spring, Back-end]
---

# Spring Boot JSP 자동 reload 되지 않는 문제 해결 방법
## 내용
> legacy Spring Project에서 Spring boot로 전환되었을때 즉시 발견할수 있는  문제중 하나는 IDE에서 jsp파일을 수정해도 auto reload되지 않는 문제입니다.
> 이 경우 JSP에 수정사항을 반영하기 위해서는 Application을 재시작해야 하는데요. 매번 이런 번거로움을 감수하면서 개발하는 것은 말도 안되죠.
> 이를 해결할수 있는 방안이 이미 마련 되어 있습니다.

## 적용 방법
### 1. Dependency 추가
 - maven 을 사용하는 경우

``` xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
    </dependency>
</dependencies>
```

 - gradle을 사용하는 경우

```
dependencies {
    compile("org.springframework.boot:spring-boot-devtools")
}
```

### 2. spring application.yml 또는 application.properties에 아래 설정을 추가

```
spring.devtools.livereload.enabled=true
```


``` yml
spring:
  devtools:
    livereload:
      enabled: true
```


## Reference
 - [https://docs.spring.io/spring-boot/docs/current/reference/html/using-spring-boot.html#using-boot-devtools-livereload](https://docs.spring.io/spring-boot/docs/current/reference/html/using-spring-boot.html#using-boot-devtools-livereload)
