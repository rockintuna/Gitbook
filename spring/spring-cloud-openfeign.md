---
description: Spring Cloud OpenFeign 를 사용해본 내용
---

# Spring Cloud OpenFeign

## Spring Cloud OpenFeign

### 개요

MSA에서는 단위 기능을 하는 서비스들이 여러개 동작하며 그로인해 서비스 사이의 통신이 이루어져야 합니다.

마이크로 서비스 사이의 통신은 동기/비동기 모두 가능합니다.

기존에는 서비스 간의 동기 통신을 위해서 resttemplate으로 http client를 생성하여 통신을 했었습니다.

Spring Cloud 관련 글을 읽던 중 OpenFeign이라는 스팩이 있어 공유하고자 합니다.

### Spring Cloud OpenFeign

[Spring Cloud OpenFeign](https://spring.io/projects/spring-cloud-openfeign)

공식문서에서는 OpenFeign를 선언적 REST 클라이언트라고 설명합니다.

RestTemplate의 경우 아래와 같이 클래스 코드를 작성하게 됩니다.

```java
@Service
@RequiredArgsConstructor
public class DataManagerAccessor {
​
  private final RestTemplate restTemplate;
​
  @Value("${ndxpro.gateway.url}")
  private String gatewayUrl;
​
  public DataModelInfoResponseDto getDataModelInfo(String dataModelName, String token) {
    HttpHeaders headers = new HttpHeaders();
    headers.set("Authorization", token);
​
    HttpEntity request = new HttpEntity(headers);
    UriComponents uriComponents = UriComponentsBuilder.fromUriString(gatewayUrl)
      .path("/manager/data-models")
      .path("/" + dataModelName)
      .build();
​
    ResponseEntity<DataModelInfoResponseDto> responseEntity = restTemplate.exchange(uriComponents.toUri(), HttpMethod.GET, request, DataModelInfoResponseDto.class);
    return responseEntity.getBody();
  }
}
```

코드에서 RestTemplate을 사용하여 HTTP 요청 및 응답을 직접 생성하고 컨트롤 합니다.

아래는 OpenFeign을 사용한 클라이언트 인터페이스입니다.

```java
@FeignClient(url = "${data-manager.url}")
public interface DataManagerFeignClient {
​
    @GetMapping("/ndxpro/v1/manager/data-models/{dataModelName}")
    DataModelInfoResponseDto getResource(@PathVariable String dataModelName);
}
```

어노테이션 기반으로 인터페이스의 메서드만 작성하면 바로 사용이 가능합니다.

Eureka 같은 Service Discovery를 사용하고 있다면 서비스 이름으로 Client를 생성할수 도 있습니다.

```java
@FeignClient(name = "DATAMANAGER-SERVICE")
public interface DataManagerFeignClient {
​
    @GetMapping("/ndxpro/v1/manager/data-models/{dataModelName}")
    DataModelInfoResponseDto getResource(@PathVariable String dataModelName);
}
```

OpenFeign는 위 예시처럼 어노테이션 기반의 인터페이스를 작성하여 쉽게 HTTP Client 구현을 자동으로 생성합니다.

이를 선언적이라고 표현한 듯 합니다.

### 사용 방법

build.gradle

```gradle
//spring cloud openfeign
implementation 'org.springframework.cloud:spring-cloud-starter-openfeign:4.1.2'
```

config (@EnableFeignClients)

```java
@Configuration
@EnableFeignClients
public class FeignClientConfig {
​
}
```

property

```java
spring:
    cloud:
        openfeign:
            httpclient:
                enabled: true
                connection-timeout: 5000
                ok-http:
                    read-timeout: 5000
```

client 인터페이스 작성

```java
@FeignClient(name = "DATAMANAGER-SERVICE")
public interface DataManagerFeignClient {
​
    @GetMapping("/ndxpro/v1/manager/data-models/{dataModelName}")
    DataModelInfoResponseDto getResource(@PathVariable String dataModelName);
}
```

call

```java
@Autowired
DataManagerFeignClient dataManagerFeignClient;
​
public void test() {
  DataModelInfoResponseDto dataModelInfo = dataManagerFeignClient.getDataModel(model.getType());
}
```
