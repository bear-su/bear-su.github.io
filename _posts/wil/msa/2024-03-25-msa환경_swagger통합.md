---
title: MSA 프로젝트에서 Swagger 통합하기 (Feat. Spring Cloud)
excerpt: MSA 기반으로 프로젝트를 진행하면 Swagger를 통합하는 방법이 중요합니다. 이를 위해 Spring Cloud를 이용한 MSA 기반 프로젝트에서 Swagger를 통합하는 방법을 알아보겠습니다.
categories: [ msa ]
tags: [ msa , swagger, spring-cloud, springboot, openapi ]
image:
  path: /assets/img/msa/covers/msa.png
---

MSA기반으로 프로젝트를 진행하다 보면 정말 많은 서비스 모듈이 생기게 됩니다. 포스트맨과 같은 API 테스트 툴을 사용하면 상관없지만 Swagger를 사용하는 경우
각 서비스에 맞게 Swagger UI를 실행해야 하며 각각의 주소를 기억해야 하는 번거로움이 있습니다.

또 보통 마이크로서비스는 API Gateway를 통해 연결되며, gateway의 주소만 public IP로 공개하는 것이 일반적입니다. 이 경우 Swagger의 API 요청도, 실제 서비스와 동일하게
Gateway를 통해 테스트 하는 것이 좋습니다.

따라서 이번 포스팅에서는 Spring Cloud를 이용한 **MSA프로젝트에서 Swagger를 통합하는 방법**을 알아보겠습니다.

## 🚀 API Gateway에 Swagger 통합하기
### 1. SpringDoc 의존성 추가
- `SpringDoc`은 OpenAPI 스펙을 기반으로 자동으로 API 문서를 생성해주는 라이브러리이며, Swagger 라이브러리가 포함되어 있어 Swagger UI를 사용할 수 있습니다.


- Netty 사용시 
~~~java
implementation 'org.springdoc:springdoc-openapi-starter-webflux-ui:2.4.0' 
~~~

- Tomcat 사용시
~~~java
implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.4.0'
~~~

### 2. application.yml 설정
~~~java
springdoc:
  swagger-ui:                  
    path: /swagger-ui.html              # swagger UI의 엔드포인트
    urls:                               # Swagger UI에서 표시할 API 문서의 URL
      - url: /post-service/v3/api-docs  # /v3/api-docs는 OpenAPI 3.0의 기본 URL이다. 
        name: post-service
      
~~~

#### 참고
저의 spring cloud gateway 설정은 다음과 같이 되어있습니다.
~~~yml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          # {{ gateway-uri }}/user-service/** -> user-service로 라우팅합니다.
          predicates:
              - Path=/user-service/**
          filters:
            - RemoveRequestHeader=Cookie
            # 기본적으로 gateway를 통해 라우팅되면 서비스 이름도 경로에 포함되는데 이를 없애주는 작업
            - RewritePath=/user-service/(?<segment>.*), /$\{segment}
        - id: post-service
          uri: lb://post-service
          predicates:
            - Path=/post-service/**
          filters:
            - RemoveRequestHeader=Cookie
            - RewritePath=/post-service/(?<segment>.*), /$\{segment}
~~~

### 3. 마이크로서비스에 의존성 추가
API 문서화를 원하는 모든 서비스에 의존성을 추가해주겠습니다. 본 포스팅에서는 `post-service`와 `user-service`에 의존성을 추가하겠습니다.
1번과 동일하게 진행합니다.

### 4. 마이크로서비스에 Configuration 클래스 추가
~~~java
/*                            Post Service                                   */
@Configuration
@OpenAPIDefinition
public class SwaggerConfig {

    @Bean
    public OpenAPI customOpenAPI(@Value("${openapi.service.url}") String url) {
        return new OpenAPI()
                .servers(List.of(new Server().url(url)))
                .info(new Info().title("Post Service API")
                        .version("v0.0.1"));
    }
}

/*                            User Service                                   */
@Configuration
@OpenAPIDefinition
public class SwaggerConfig {

    @Bean
    public OpenAPI customOpenAPI(@Value("${openapi.service.url}") String url) {
        return new OpenAPI()
                .servers(List.of(new Server().url(url)))
                .info(new Info().title("User Service API")
                        .version("v0.0.1"));
    }
}
~~~

- openapi.service.url: gateway uri/service-name <br>
  - 예시 ) http://localhost:8000/user-service

### 5. API Gateway Route 설정
~~~java
@Configuration
@OpenAPIDefinition
public class SwaggerConfig {

    ...
    
    @Bean
    public RouteLocator routeLocator(RouteLocatorBuilder builder) {
        return builder
                .routes()
                // request.path가 다음과 같으면 uri()로 이동
                .route(r -> r.path("/post-service/swagger-ui.html").and().method(HttpMethod.GET).uri("lb://post-service"))
                .route(r -> r.path("/user-service/swagger-ui.html").and().method(HttpMethod.GET).uri("lb://user-service"))
                .build();
    }
}

~~~

## 🛠️ 트러블 슈팅
### 🔧️ CORS 에러
API Gateway를 통해 Swagger의 API 요청이 전송되기 때문에 CORS 에러가 발생할 수 있습니다. 기존에 클라이언트 주소로 사용하던 `localhost:3000` 외에 추가로 Gateway 주소를 추가해주었습니다.
~~~java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOriginPatterns("http://localhost:3000", "http://localhost:8000")
                .allowedHeaders("*")
                .allowCredentials(true)
                .allowedMethods(
                        HttpMethod.GET.name(),
                        HttpMethod.HEAD.name(),
                        HttpMethod.POST.name(),
                        HttpMethod.PUT.name(),
                        HttpMethod.PATCH.name(),
                        HttpMethod.DELETE.name()
                );
    }
}
~~~


### 🔧 Springdoc 라이브러리의 이해 부족으로 발생한 이슈
`springdoc-openapi`는 런타임에 애플리케이션을 검사하여 스프링 구성, 클래스 구조 및 다양한 어노테이션을 기반으로 API 문서 생성을 자동화해주는 라이브러리입니다.

#### ✅ springdoc의 지원
- OpenAPI 3
- Spring-boot v3 (Java 17 & Jakarta EE 9)
- JSR-303, specifically for @NotNull, @Min, @Max, and @Size.
- Swagger-ui
- OAuth 2
- GraalVM native images

#### ✅ OpenAPI
springdocs 의존성을 추가하면 ApiDocs 클래스 사용이 활성화되고 `/v3/api-docs` 주소에서 JSON 형태로 API 문서를 확인할 수 있습니다.
~~~java
public static class ApiDocs {
    private String path = "/v3/api-docs";
    private boolean enabled = true;
    private boolean resolveSchemaProperties;
    private boolean resolveExtensionsProperties;
    private Groups groups = new Groups();
    private OpenApiVersion version;
}
~~~

#### ✅ Swagger-UI
JSON 형태로 문서를 보는 것이 불편하므로 Swagger-UI를 사용하여 문서를 시각화할 수 있습니다.

#### ✅ 정리
즉, `SpringDoc`라이브러리 안에 `OpenAPI`와 `Swagger-UI`가 존재하며 각각은 JSON 형태의 문서 자동화와 시각화된 문서를 제공하는 역할을 합니다.


## 🔍 참고
- [Spring Docs Official](https://springdoc.org/)
- [Spring Cloud Gateway를 이용한 API Gateway 구축하기](https://medium.com/@pubuduc.14/swagger-openapi-specification-3-integration-with-spring-cloud-gateway-part-2-1d670d4ab69a)
