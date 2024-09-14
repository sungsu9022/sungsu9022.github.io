---
title: "[Spring] Spring6 Http Interface Client - 1"
author: sungsu park
date: 2024-09-13 20:47:00 +0800
categories: [DevLog, Spring]
tags: [Java, Kotlin, Spring]
---


## 1. 어떤 HttpClient을 사용할것인가?
 - Spring을 이용한 Application을 개발할 때, 다른 서버와의 HTTP 통신이 필요할 경우가 많다.
 - 이때 어떤 HttpClient를 사용할지 결정하는 것은 중요한 선택입니다. 다양한 HttpClient 옵션이 있지만, 각 방식마다 장단점이 있다.
 - 여기서는 대표적인 HttpClient들을 비교하여 어떤 선택이 가장 적합할지 살펴보자.

### 1.1 순수 HttpClient 라이브러리를 사용하는 방법
 - 순수 HttpClient 라이브러리는 Java 11부터 제공되는 java.net.http.HttpClient를 사용하는 방법이다.
 - 이 라이브러리는 Java 표준 라이브러리로, 외부 라이브러리 의존성 없이 HTTP 요청을 보낼 수 있다.
 - HttpClient는 동기 및 비동기 방식 모두를 지원하며, 간단한 HTTP 통신부터 복잡한 요청까지 다양한 기능을 제공한다.

#### 장점
- 외부 라이브러리 없이 사용 가능
- Java 표준 라이브러리의 일환으로 간편한 유지보수
- 비동기 및 동기 요청 모두 지원

#### 단점
- Spring과의 통합이 부족
- 코드가 상대적으로 장황할 수 있음

``` kotlin
import java.net.URI
import java.net.http.HttpClient
import java.net.http.HttpRequest
import java.net.http.HttpResponse

fun httpClientTest() {
    val client = HttpClient.newBuilder().build()

    val request = HttpRequest.newBuilder()
        .uri(URI.create("https://www.naver.com"))
        .GET()
        .build()

    val response: HttpResponse<String> = client.send(request, HttpResponse.BodyHandlers.ofString())
    // 추가적으로 Response Model에 대한 Deserialize 필요
    logger.info { "response : ${response.body()}" }
}
```

### 1.2 RestTemplate
 - RestTemplate는 Spring 3.0에서 처음 도입된 전통적인 HttpClient입니다. 많은 개발자들이 익숙하게 사용해왔으며, 간단한 HTTP 요청을 쉽게 처리할 수 있다.
 - 그러나 RestTemplate는 Spring 5부터는 더 이상 새로운 기능이 추가되지 않고, 유지보수 모드로 전환되었다. 이는 Spring에서 비동기 처리와 반응형 프로그래밍의 중요성이 커지면서, 이를 더 잘 지원하는 클라이언트로의 전환을 권장하고 있기 때문이다.

#### 장점
 - 간편한 사용법과 널리 사용된 예제
 - Spring과의 깊은 통합
 - 동기식 요청에 적합

#### 단점
 - 비동기 처리 지원 부족
 - 향후 기능 업데이트 없음

``` kotlin
fun restTemplateTest() {
    val restTemplate = RestTemplate()
    val url = "https://www.naver.com"
    val response = restTemplate.getForEntity(url, String::class.java)

    if (response.statusCode === HttpStatus.OK) {
        val responseBody = response.body
        logger.info { "response : $responseBody" }
    }
}
```

### 1.3 Spring WebClient
 - WebClient는 Spring 5에서 등장한 비동기 및 반응형 HTTP 클라이언트입니다. RestTemplate의 대체로 제안되며, 더 나은 성능과 확장성을 제공한다.
 - WebClient는 비동기 및 블로킹 둘 다 지원하며, 반응형 스트림을 사용해 대용량 데이터를 처리하는데 유리하다.

#### 장점
 - 비동기 및 반응형 프로그래밍 지원
 - 다양한 커스터마이징 옵션

#### 단점
 - RestTemplate에 비해 다소 복잡한 설정과 사용법
 - 다른 방식 대비 러닝커브가 존재함.

``` kotlin
fun webClientTest() {
    val webClient = WebClient.create("https://www.naver.com")

    val response = webClient.get()
        .uri("/data")
        .retrieve()
        .bodyToMono(String::class.java)

    response.subscribe { logger.info { it } }
}
```

### 1.4 Spring Cloud OpenFeign
 - Spring Cloud OpenFeign은 Spring Cloud 프로젝트에서 제공하는 Feign의 확장판이다.
 - Feign은 인터페이스 기반의 선언적 HTTP 클라이언트로, HTTP API 클라이언트를 쉽게 정의할 수 있다.

#### 장점:
- 간결하고 선언적인 HTTP 클라이언트 정의
- Spring Cloud와의 통합으로 강력한 기능 제공
- Srping MVC의 annotation을 활용 가능

#### 단점:
- Spring Cloud 의존성으로 인해 독립적인 사용은 제한적이다.
- Spring Cloud OpenFeign은 유지 관리 모드에 있으며 더 이상 적극적으로 개발되지 않으므로 사용을 지양하는 편이 좋다.

```kotlin
import org.springframework.cloud.openfeign.FeignClient
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RestController
import org.springframework.beans.factory.annotation.Autowired

@FeignClient(name = "exampleClient", url = "https://api.example.com")
interface ExampleClient {
  @GetMapping("/data")
  fun getData(): String
}

@RestController
class ExampleController(
  private val exampleClient: ExampleClient,
) {
  @GetMapping("/fetch")
  fun fetchData(): String {
    return exampleClient.getData()
  }
}
```


### 1.5 Openfeign
 - OpenFeign은 Netflix에서 처음 개발한 인터페이스 기반의 선언적 HTTP 클라이언트 라이브러리이다.
 - OpenFeign은 Java의 인터페이스에 메서드 정의만으로 HTTP 요청을 간결하게 수행할 수 있다.
 - Openfeign은 Spring과는 완벽히 독립적인 라이브러리이다.
 - 과거에는 Non-Blocking 방식을 지원하지 않았는데, 현재는 Reactive Streams Wrapper을 통해 지원한다.

#### 장점:
- 간결하고 선언적인 API 클라이언트 작성
- Spring Cloud와의 결합 없이 독립적으로 사용 가능
- 플러그인 시스템을 통해 다양한 기능 추가 가능

#### 단점
- Spring Cloud의 부가 기능이 필요할 경우 직접 구현해야 함
- Spring Cloud와의 통합 기능은 사용 불가


``` kotlin
import feign.Feign
import feign.RequestLine

interface ExampleClient {
    @RequestLine("GET /data")
    fun getData(): String
}

val exampleClient = Feign.builder()
    .target(ExampleClient::class.java, "https://api.example.com")

val data = exampleClient.getData()
println(data)
```

### 1.6 요약
- Spring 기반 Server App에서 다른 서버와 통신하기 위한 다양한 방법을 알아보았다.
- 위 방법 중 사실 개인적으로 가장 맘에 드는 라이브러리는  `1.4 Spring Cloud OpenFeign`이다.
- 하지만 Spring Cloud OpenFeign에는 치명적인 단점이 있다.
   - 첫째, Non-blocking 방식을 지원하지 않는다.
   - 둘쨰, 유지보수 모드에 돌입하여 더이상 유의미한 기능 수정이 이루어지지 않는다는 점이다.
- 이로 인해 혼동이 가중되었으나, Spring 6가 나오면서 이 부분에 대해 고민이 줄게 되었다. 이제 본격적으로 Spring6 Http Interface Client에 대해 알아보자.

## 2. Spring6 Http Interface Client
 - Spring 6에서 도입된 HTTP Interface는 REST API와의 상호작용을 간소화하기 위한 새로운 기능이다.
 - 이 기능은 REST API를 호출하기 위한 인터페이스 기반의 선언적 방법을 제공한다.
 - 이 기능을 통해 기존의 RestTemplate이나 WebClient보다 간편하게 API 호출을 할 수 있다.

### 2.1 왜 Spring6 Http Interface Client인가?
 - Feign과 같은 선언적 인터페이스 방식으로 관리할 수 있음.
 - ReactorHttpExchangeAdapter 이용해서 구현하는 경우 reactive variants를 지원한다.
 - Spring Framework에서 공식지원 하는 방식이므로, 사용하지 않을 이유가 없다.
 - 아직 초기 단계이므로 그럼에도 미흡한 부분이 있고, 이는 장기적으로 업데이트되거나 Customized를 통해 커버할수 있다.
 - 그럼 본격적으로 사용방법을 알아보도록 하자.


### 2.2 Spring6 Http Interface Client 내부 구조
#### HttpExchangeAdapter
 - `spring-web:6.1.11` 기준으로 다음과 같은 Adaptor을 제공함.
 - 기존에 사용하던 RestClient, RestTemplate, WebClient와 같은 설정을 토대로 사용이 가능하다.

<img width="693" alt="" src="https://github.com/user-attachments/assets/641b00ba-d501-469d-bd0c-ba3b0bbdf6cf">


#### Proxy 객체 생성 방식
 - 개발자는 Spring6 Http Interface의 선언형 방식으로 코드를 정의
 - 내부적으로는 HttpServiceProxyFactory > ProxyFactory > DefaultAopProxyFactory 를 통해  JdkDynamicAopProxy 객체를 생성하여 처리한다.
 - 위 내용을 볼떄 현재는 인터페이스로 정의된 방식만 지원 가능한것으로 보인다.
    - 추후 interface뿐만 아니라 class로써, 특정 API들은 본문을 직접 작성하는 방식을 지원한다거나 하면 그에 맞게 Proxy 방식도 ObjenesisCglibAopProxy 같은 방식을 통해 처리하는것도 가능하지 않을까 예상된다.


## 3 Spring 6 Http Interface Client 개발하기
- Spring6부터는 @HttpExchange 메서드를 사용하여 HTTP 서비스를 인터페이스로 정의할 수 있다.
- 인퍼테이스를 HttpServiceProxyFactory로 전달하여 RestClient나 WebClient와 같은 HTTP 클라이언트를 통해 요청을 수행하는 프록시를 생성한다.

### 3.1 Https Client Request/Response
 - spring-web에서 제공하는 Annotation을 조합해서 간결하게 ApiClient를 정의할수 있다.

##### Exchange Annotations
 - Exchange 애노테이션은 대체적으로 spring-web과 1:1 맵핑되는 Exchange Annotation을 제공한다.

| SpringWeb       | HttpInterface   |
|-----------------|-----------------|
| @RequestMapping | @HttpExchange   |
| @GetMapping     | @GetExchange    |
| @PostMapping    | @PostExchange   |
| @PutMapping     | @PutExchange    |
| @PatchMapping   | @PatchExchange  |
| @DeleteMapping  | @DeleteExchange |

#####  Method Parameters
 - 대부분의 spring-web에서 지원하는 annotation을 지원한다.
 - 하나의 Spring Web Controller와 완전히 동일한 방식으로 HttpInterface를 정의할수는 없다.
   - 하지만, 이런부분을 직접 구현함으로써 어느정도 문제를 해결할수는 있다.

| SpringWeb         | HttpInterface     | Description                    |
|-------------------|-------------------|--------------------------------|
| @RequestParam     | @RequestParam     | 동일하게 지원                        |
| @RequestHeader    | @RequestHeader    | 동일하게 지원                        |
| @PathVariable     | @PathVariable     | 동일하게 지원                        |
| @RequestBody      | @RequestBody      | 동일하게 지원                        |
| @CookieValue      | @CookieValue      | 동일하게 지원                        |
| @RequestAttribute | @RequestAttribute | webClient httpService에서만 지원    |
| @RequestPart      | @RequestPart      | 동일하게 지원                        |
| MultipartFile     | MultipartFile     | @RequestPart로 전달하여 동일하게 사용 가능. |
| @ModelAttribute   | -                 | 지원하지 않음.                       |

#####  Return Values
 - 일반적인 Blocking I/O에 대한 응답을 지원한다.
 - HttpHeader를  받는다거나 HttpStatus 및 Header를 포함한 ResponseEntity로 반환받는것을 지원한다.
 - webClient 기반인 경우 Webflux의 Mono나 Flux 도 함께 지원한다.

| ReturnType                      |
|---------------------------------|
| `Void / Unit`                   |
| `HttpHeaders`                   |
| `<T>`                           |
| `ResponseEntity<Unit>`          |
| `ResponseEntity<T>`             |
| `ResponseEntity<T>`             |
| `Mono<Unit>`                    |
| `Mono<HttpHeaders>`             |
| `Mono<T>`                       |
| `Flux<T>`                       |
| `Mono<ResponseEntity<Unit>>`    |
| `Mono<ResponseEntity<T>>`       |
| `Mono<ResponseEntity<Flux<T>>`  |

### 3.2 Http Interface Client 정의
- HttpInterfaceClient의 Request/Response를 어떻게 정의할지 알았으니 이제 샘플 코드를 살펴보자.
- User(사용자) Resource 에 대한 REST API가 있다고 하고, 이를 호출하기 위한 HttpClient 코드를 정의하면 다음과 같이 정의할수 있다.

``` kotlin
package com.starter.core.clients.internal.admin.api

import com.starter.core.models.user.UserCreateRequest
import com.starter.core.models.user.UserPatchRequest
import com.starter.core.models.user.UserResponse
import com.starter.core.models.user.UserSearchRequest
import org.springframework.web.bind.annotation.ModelAttribute
import org.springframework.web.bind.annotation.PathVariable
import org.springframework.web.bind.annotation.RequestBody
import org.springframework.web.service.annotation.DeleteExchange
import org.springframework.web.service.annotation.GetExchange
import org.springframework.web.service.annotation.HttpExchange
import org.springframework.web.service.annotation.PatchExchange
import org.springframework.web.service.annotation.PostExchange

@HttpExchange("/api/v1/users")
interface UserApiClient : UserApi {

    @PostExchange
    override fun create(@RequestBody request: UserCreateRequest): UserResponse

    @GetExchange
    override fun getUsers(@ModelAttribute request: UserSearchRequest) : List<UserResponse>

    @GetExchange("/{uuid}")
    override fun getUser(@PathVariable uuid: String): UserResponse

    @PatchExchange("/{uuid}")
    override fun patchUser(
        @PathVariable uuid: String,
        @RequestBody request: UserPatchRequest
    )

    @DeleteExchange("/{uuid}")
    override fun deleteUser(@PathVariable uuid: String)
}
```

 - 구현코드를 보면 feign과 유사한 형태로 간결하게 작성이 가능한것을 알 수 있다.
 - 다만, 위에서 말한것처럼 webClient 기반으로 정의할 경우 webflux 스펙도 모두 지원하므로 훨씬 더 좋다고 볼수 있다.
   - 사실 글을 작성하는 현재 시점에는 Spring Cloud OpenFeign 말고, 순수 Openfeign을 사용할 경우 reactiveClient로 정의할수 있는 feature가 추가된것으로 보이기는 하나, 나라면 spring 공식지원되는 spring6HttpInterface를 쓸것 같다.

### 3.3 Http Interface Client Configuration
#### HttpServiceProxyFactory
 - 2.3.2에서 HttpInterface를 정의했다면 httpServiceProxyFactory를 통해 Proxy 객체를 만들어 주어야 한다.
 - 공통적으로 사용할수 있도록 Util method를 정의하면 아래와 같이 만들수 있다.
 - HttpServiceProxyFactory Builder에는 더 많은 옵션이 있지만, 내가 필요했던건 customArgumentResolver를 등록하는것뿐이라 아래와 같이 구성했다.
   - 위 Request 정의시 ModelAttribute를 지원하지 않으므로 이를 처리할수 있도록 직접 구현한것으로 보면 된다.

```kotlin
import com.starter.core.clients.common.resolver.ModelAttributeArgumentResolver
import org.springframework.web.reactive.function.client.WebClient
import org.springframework.web.reactive.function.client.support.WebClientAdapter
import org.springframework.web.service.invoker.HttpServiceProxyFactory

object HttpInterfaceProxyFactory {

    inline fun <reified T> create(
        webClient: WebClient,
        resolvers: List<ModelAttributeArgumentResolver> = listOf(ModelAttributeArgumentResolver.DEFAULT_INSTANCE),
    ): T {
        val builder = HttpServiceProxyFactory
            .builderFor(WebClientAdapter.create(webClient))

        resolvers.forEach { builder.customArgumentResolver(it) }

        val httpServiceProxyFactory = builder.build()
        return httpServiceProxyFactory.createClient(T::class.java)
    }

}
```

#### Bean Configuration
 - Config Class를 별도로 정의하고, 이를 필요한 곳에서 Import해서 쓸수 있는 방식으로 정의해보았다.
 - HttpInterfaceProxyFactory Util 메소드를 통해 Instance를 만들고 이를 Bean으로 등록해서 사용한다.
 - webClient는 하나의 API서버군을 바라보고 할수 있도록 정의하고, 이를 HttpClient들이 주입받아 Bean을 등록하도록 정의한다.
    - 이렇게 하면 Controller와 HttpClient를 1:1로 관리할수 있어서 코드를 깔끔하게 유지할수 있다.

``` kotlin
import com.starter.core.clients.common.HttpInterfaceProxyFactory
import com.starter.core.clients.common.WebClientFactory
import com.starter.core.clients.internal.InternalClientsProperties
import com.starter.core.clients.internal.file.api.FileDownloadApiClient
import com.starter.core.clients.internal.file.api.FileUploadApiClient
import org.springframework.beans.factory.annotation.Qualifier
import org.springframework.boot.context.properties.ConfigurationPropertiesScan
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.ComponentScan
import org.springframework.context.annotation.Configuration
import org.springframework.web.reactive.function.client.WebClient

@Configuration
@ConfigurationPropertiesScan("com.starter.core.clients.internal.file")
@ComponentScan("com.starter.core.clients.internal.file")
class FileClientConfig(
    private val internalClientsProperties: InternalClientsProperties
) {
    companion object {
        private const val FILE_WEB_CLIENT = "fileWebClient"
    }

    @Bean(FILE_WEB_CLIENT)
    fun fileWebClient(): WebClient {
        return WebClientFactory
            .createNettyClient(baseUrl = internalClientsProperties.file)
    }

    @Bean
    fun fileDownloadApiClient(@Qualifier(FILE_WEB_CLIENT) webClient: WebClient): FileDownloadApiClient {
        return HttpInterfaceProxyFactory.create<FileDownloadApiClient>(webClient)
    }

    @Bean
    fun fileUploadApiClient(@Qualifier(FILE_WEB_CLIENT) webClient: WebClient): FileUploadApiClient {
        return HttpInterfaceProxyFactory.create<FileUploadApiClient>(webClient)
    }
}
```


### 3.4 Http Interface Customize
#### Api Interface로 Controller와 1:1로 관리하기
 - 내가 만들어둔 프로젝트 아키텍쳐에서는 특정 Client의 API를 담당하는 BFF 모듈을 별도로 만드는것을 목표로 하고 있는데, 이 경우 BFF에서 내부 모듈들의 API를 호출하고, 이를 aggregation하는등의 관리가 필요하다.
 - 이때 Controller와 HttpInterface가 동일한 API임을 나타내기 위해 Api Interface를 정의하고 이를 Controller와 HttpInterface가 구현하도록 구성했다.

<img width="700" alt="" src="https://github.com/user-attachments/assets/790f67ae-a9a2-49ad-9197-b452dd320b0c">

``` kotlin
// FileUploadApi.kt
interface FileUploadApi {
    fun uploadFile(fileUploadRequest: FileUploadRequest): FileResponse
}

// FileUploadRequest.kt
data class FileUploadRequest(
    val fileName: String,
    override val file: MultipartFile,
    override val fileType: FileType,
    override val description: String?,
    override val uploadDirectoryType: UploadDirectoryType
) : FileUploadRequestInterface
```

``` kotlin
// FileUploadController.kt
package com.starter.file.app.file.controller

import com.starter.core.clients.internal.file.api.FileUploadApi
import com.starter.core.models.file.FileResponse
import com.starter.core.models.file.FileUploadRequest
import com.starter.file.app.file.facade.FileUploadFacade
import io.swagger.v3.oas.annotations.Operation
import io.swagger.v3.oas.annotations.Parameter
import org.springframework.http.MediaType
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@RestController
@RequestMapping("/api/v1/files/upload")
class FileUploadController(
    private val fileUploadFacade: FileUploadFacade,
) : FileUploadApi {

    @PostMapping(consumes = [MediaType.MULTIPART_FORM_DATA_VALUE] )
    @Operation(description = "파일 업로드 API")
    override fun uploadFile(@Parameter fileUploadRequest: FileUploadRequest): FileResponse {
        val fileResponse = fileUploadFacade.uploadFile(fileUploadRequest)
        return fileResponse
    }
}
```

``` kotlin
// FileUploadApiClient.kt
@HttpExchange("/api/v1/files/upload")
interface FileUploadApiClient : FileUploadApi {

    @PostExchange(contentType = MediaType.MULTIPART_FORM_DATA_VALUE)
    override fun uploadFile(@ModelAttribute fileUploadRequest: FileUploadRequest): FileResponse
}
```

 - 이런식으로 관리하면 code trace시에도 좀 더 편하게 코드를 추적할 수 있다.
 - 하지만, HttpInterface에서 지원하는 스펙의 한계로 별도의 customize 없이는 모든 케이스를 이렇게 관리할 수 없다.

#### ModelAttributeArgumentResolver
 - 위 코드를 보면 Controller에서는 파일 업로드를 위해 MultipartFile를 포함한 Model를 받고 있다.
   - 별도의 구현이 없다면 HttpInterface에서는 `@RequestPart` 를 통해 파일을 전달해주어야 한다.
   - `multipart/form-data` 외에도 RequestParamter가 많은 경우 `@RequestParam` 을 나열하는 경우를 방지하기 위해 하나의 Request Model Class를 선언하기도 하는데, 이부분에서도 지원하지 않는다.
 - 위의 문제 해결을 위해 Custom ArgumentResolver를 구현해보자.

```kotlin
package com.starter.core.clients.common.resolver

import com.starter.core.common.utils.JsonUtil
import com.starter.core.common.utils.ReflectionUtils
import mu.KLogging
import org.springframework.core.MethodParameter
import org.springframework.core.io.Resource
import org.springframework.http.HttpEntity
import org.springframework.http.HttpHeaders
import org.springframework.lang.Nullable
import org.springframework.web.bind.annotation.ModelAttribute
import org.springframework.web.multipart.MultipartFile
import org.springframework.web.service.invoker.HttpRequestValues
import org.springframework.web.service.invoker.HttpServiceArgumentResolver
import kotlin.reflect.KFunction1

class ModelAttributeArgumentResolver(
    val argumentToMapFunc: KFunction1<Any, Map<String, Any?>> = ReflectionUtils::objectToMap,
) : HttpServiceArgumentResolver {
    companion object : KLogging() {
        val DEFAULT_INSTANCE = ModelAttributeArgumentResolver()
        val INSTANCE_WITH_JSON = ModelAttributeArgumentResolver(
            JsonUtil::toMap
        )
    }

    override fun resolve(
        argument: Any?,
        parameter: MethodParameter,
        requestValues: HttpRequestValues.Builder
    ): Boolean {
        parameter.getParameterAnnotation(ModelAttribute::class.java)
            ?: return false

        if(argument == null) {
            return false
        }

        val argumentMap = argumentToMapFunc(argument)
        val isPartRequest = isPartRequest(argumentMap)
        logger.debug { "argumentMap: $argumentMap,  isPartRequest: $isPartRequest" }
        argumentMap
            .filter { it.value != null }
            .forEach {
                if(isPartRequest) {
                    addRequestPartValues(name = it.key, value = it.value!!, requestValues = requestValues)
                } else {
                    addRequestParamValues(name = it.key, value = it.value!!, requestValues = requestValues)
                }

            }

        return true
    }

    @Nullable
    private fun isPartRequest(argumentMap: Map<String,Any?>): Boolean {
        return argumentMap.values.any { it is MultipartFile }
    }


    private fun addRequestParamValues(name: String, value: Any, requestValues: HttpRequestValues.Builder) {
        if(value is Collection<*>) {
            requestValues.addRequestParameter(name, value.joinToString())
        } else {
            requestValues.addRequestParameter(name, value.toString())
        }
    }

    private fun addRequestPartValues(name: String, value: Any, requestValues: HttpRequestValues.Builder) {
        if(value is Collection<*>) {
            requestValues.addRequestPart(name, value.joinToString())
        } else if (value is MultipartFile) {
            val value = toMultipartFileHttpEntity(name, value)
            requestValues.addRequestPart(name, value)
        } else {
            requestValues.addRequestPart(name, value.toString())
        }
    }

    private fun toMultipartFileHttpEntity(name: String, multipartFile: MultipartFile): HttpEntity<Resource> {
        val headers = HttpHeaders()
        if (multipartFile.originalFilename != null) {
            headers.setContentDispositionFormData(name, multipartFile.originalFilename)
        }

        if (multipartFile.contentType != null) {
          headers.add(HttpHeaders.CONTENT_TYPE, multipartFile.contentType)
        }

        return HttpEntity<Resource>(multipartFile.resource, headers)
    }
}

```

 - 위 구현코드는 @ModelAttribute 애노테이션이 달린 class에 대해서 Controller와 동일하게 동작할수 있도록 직접 구현한  코드이다.
   - 현재는 이에 대한 공식적인 ArgumentResolver가 없기 떄문이 이를 등록해서 사용할수 있다.
   - 만약 공식 지원이 된다면 그때 삭제하거나 ModelAttribute에 대해 지원을 하지만, 의도와 다른 게 있다면 별도의 annotaiton으로 정의해서 사용할수 있을것이다.
   - argument을 request에 넣어주기 위해 map으로 변환한 뒤 체크를 하는데, 필요에 따라 reflection 혹은 json 방식으로 사용할 수 있다.


## 4. 결론
 - Spring 기반 웹 애플리케이션을 개발할때 필연적으로 API를 호출하기 위한 HttpClient가 필요하다.
 - 이 떄, 우리는 무엇을 어떻게 쓸지를 고민하는데 Spring 6 이후부터는 큰 고민 없이 Http Interface Client 사용을 고려할 수 있을것 같다.
 - 하지만, 위에서 Customize 한 것 이외에 부가적인 문제 해결이 필요한 부분이 아직 남아있는데, 이 부분은 이후 포스팅을 통해 공유하려고 한다.
 - 관련 코드는 [starter/1.240914](https://github.com/sungsu9022/kotlin-spring-multi-module-starter/tree/1.240914) 에서 확인할수 있다.


## Refernece
- [Spring Http Interface Client 공식문서](https://docs.spring.io/spring-framework/reference/web/webflux-http-interface-client.html)
- [Spring Cloud OpenFeign](https://github.com/spring-cloud/spring-cloud-openfeign)
- [OpenFeign](https://github.com/OpenFeign/feign)

