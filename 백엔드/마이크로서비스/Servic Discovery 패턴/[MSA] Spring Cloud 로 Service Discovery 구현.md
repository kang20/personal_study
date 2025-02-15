## Discovery Server 설정 
```java

dependencies {  

    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-server'  

}

@SpringBootApplication  
@EnableEurekaServer  
public class DiscoveryServerApplication {  
  
    public static void main(String[] args) {  
       SpringApplication.run(DiscoveryServerApplication.class, args);  
    }  
  
}
```
```yaml

server:
  port: 8070

eureka:
  instance:
    hostname: eurekaserver

  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/

```

    registerWithEureka: false -> EurekaServer 자체이기 때문에 등록 x
    fetchRegistry: false -> EurekaServer 자체이기 때문에 서비스 리스트를 Eureka로부터 안들고 옴



## Discovery Client

```java
implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'

eureka:
  instance:
    prefer-ip-address: true
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
        defaultZone: http://localhost:8070/eureka/

```

## Spring Cloud Load Balancer
### Client Side Load Balance
**Spring Cloud Discovery Client** 는 *Eureka 서버가 비 가용중이거나* 어느 정도 최적화를 위해 클라이언트 측에서 *캐싱과* 부하 분산을 위해 Client 가 요청을 보낼 때 *로드밸런싱을* 적용한다.

Spring Cloud Discovery Client는 Spring Cloud Load Balancer 를 사용하여 이를 적용한다.
그리고 Discovery Server 에게 질의를 해서 (서비스 이름으로) 물리주소를 얻는다.

##### Discover Client가 다른 서비스 요청하는 방법
1. Discovery Client
2. RestTemplate
3. FegineClient


### Discovery Client
```java
@SpringBootApplication  
@EnableDiscoveryClient  // Discovery Client를 사용하기 위한 어노테이션
public class ServiceAApplication {  
    public static void main(String[] args) {  
       SpringApplication.run(ServiceAApplication.class, args);  
    }  
}

@RestController  
@RequestMapping("/test")  
@RequiredArgsConstructor  
public class TestController {  
  
    private final DiscoveryClient discoveryClient;  
  
    @RequestMapping("/discovery-client")  
    public String test() {  
       // RestTemplate restTemplate = new RestTemplate();  
  
       List<ServiceInstance> instances = discoveryClient.getInstances("service-b");  
       ServiceInstance serviceInstance = instances.get(0);  
  
       String url = String.format("%s/hello", serviceInstance.getUri().toString());  
  
       ResponseEntity<String> response = restTemplate.exchange(url, org.springframework.http.HttpMethod.GET, null,  
             String.class);  
  
  
       return response.getBody();  
    }

```

Discovery Client 를 사용하여 서비스 이름으로 instance 들을 조회를 한다.
Instance 의 url 로 요청을 보내는 작업을 수행한다.

##### 이 방식의 단점
- 로드밸런싱을 지원하지는 않는다.  (개발자의 몫)
- 너무 많은 코드가 필요
- 캐싱이 안됨 (개발자의 몫)

### RestTemplate
```java
@SpringBootApplication  
@EnableFeignClients  
public class ServiceAApplication {  
  
    public static void main(String[] args) {  
       SpringApplication.run(ServiceAApplication.class, args);  
    }  
  
  
    @Bean  
    @LoadBalanced    public RestTemplate restTemplate() {  
       return new RestTemplate();  
    }  
  
}


/// 위와 동일한 controller
private final RestTemplate restTemplate;
@RequestMapping("/rest-template")  
public String test2() {  
  
    String url = "http://service-b/hello";  
  
    ResponseEntity<String> response = this.restTemplate.exchange(url, org.springframework.http.HttpMethod.GET, null,  
       String.class);  
    return response.getBody();  
}

```

**@LoadBalanced 을 사용한 Bean으로 RestTemplate을 적용하면 서비스 이름으로 호출 가능하다.**

로드밸런싱을 지원한다.


### FegineClient

```java

@SpringBootApplication  
@EnableFeignClients  
public class ServiceAApplication {  
  
    public static void main(String[] args) {  
       SpringApplication.run(ServiceAApplication.class, args);  
    }  
}

@FeignClient(name = "service-b")  
public interface TestFegineClient {  
    @RequestMapping(  
       value = "/hello",  
       method = RequestMethod.GET,  
       consumes = "application/json"  
    )  
    String sayHello();  
  
}

  
@RequestMapping("/feign-client")  
public String test3() {  
    return testFegineClient.sayHello();  
}

```
Netflix 의 FeginClient 를 사용하면 인터페이스로 구현 가능하다.

#### FeginClient 와 RestTemplate 무슨 차이인가?
**FeginClient** 는 Netflix 의 OpenFegin을 Spring 이 호환해준다. Eureka Client 와 자동 호환하고 **Spring Cloud LoadBalancer** 와 호환된다.


RestTemplate (최신 버전은 비동기 방식을 지원하는 WebClient를 사용한다. )
RestTemplate은 Spring Cloud LoadBalancer를 위해 설정이 필요하다.

> 중요!!!
- RestTemplate의 오류는 해당 ResponseEntity 의 getStatus 메서드로 얻을 수 있다.
- FeginClient 는 FeginException 에 4xx - 5xx 에러가 존재 한다. 따라서 에러 디코더 클래스를 사용해야 한다.


## 참고
[스프링 마이크로서비스 코딩 공작소 - 예스24](https://www.yes24.com/Product/Goods/110243944)


