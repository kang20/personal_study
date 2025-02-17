
## 스프링 클라우드 게이트웨이 소개
스프링 클라우드 게이트웨이는 스프링 프레임워크 , 프로젝트 리액터 , 스프링 부트를 기반으로 한 API 게이트웨이 구현체이다.

스프링 클라우드 게이트웨이는 다음 기능들을 제공한다.

- **애플리케이션의 모든 서비스 경로를 단일 URL에 매핑한다**
	스프링 클라우드 게이트웨이는 하나의 URL에 제한되지 않고 실제로 여러 경로의 진입점을 정의하고 경로 매핑을 세분화할 수 있다.
- **게이트웨이로 유입되는 요청과 응답을 검사하고 조치를 취할 수 있는 필터를 작성한다**
	이 필터를 사용하면 코드에 정책 시행 지점을 삽입해서 모든 서비스 호출에 다양한 작업을 수행할 수 있다.
- **요청을 실행하거나 처리하기 전에 주어진 조건을 충족하는지 확인할 수 있는 서술자(predicates)를 만든다**
	스프링 클라우드 게이트웨이는 자체 Route Predicate Factories 세트가 포함되어 있다.

### 스프링 클라우드 게이트웨이 세팅하기

#### 의존성

```gradle
// gateway  
implementation 'org.springframework.cloud:spring-cloud-starter-gateway'

```

#### config 구성
```yaml
# config 서버 구성
spring:  
  application:  
    name: gateway  
  config:  
    import: configserver:http://localhost:8071

# Eureka client 구성
eureka:
  instance:
    prefer-ip-address: true # IP 등록 사용
  client:
    register-with-eureka: true # 반드시 true
    fetch-registry: true
    service-url:
      defaultZone: http://localhost:8070/eureka/ 

```


### 스프링 게이트웨이 라우팅 구성
#### 개요
스프링 클라우드 게이트웨이는 **리버스 프록시(reverse proxy)** 역할을 한다.
리버스 프록시는 자원에 도달하려는 클라이언트와 자원 사이에 위치한 **중개 서버**다.

**리버스 프록시** 는 클라이언트 요청을 캡처한 후 클라이언트를 대신하여 원격 자원을 호출한다
클라이언트는 어떤 서비스와 통신하는지 알지 못한다. 클라이언트는 오직 **프록시와 통신한다.**

**게이트웨이**(리버스 프록시 이제는 게이트웨이라 부르겠다)는 상위 서비스에 클라이언트의 요청을 전달하는데, *요청을 전달하기 위해서는 유입된 호출과 상위 경로와의 매핑 방법(라우팅 방법) 을 알아야한다.*

스프링 클라우드 게이트에서는 이를 수행할 수 있는 몇 가지 메커니즘을 제공한다
- 서비스 디스커버리를 이용한 자동 경로 매핑
- 서비스 디스커버리를 이용한 수동 경로 매핑

#### 서비스 디스커버리를 이용한 자동 경로 매핑
```yaml
spring:
  cloud:
    gateway:
      discovery.locator:
        enabled: true
        lowerCaseServiceId: true
```


![[Pasted image 20250216030401.png]]

>자동 경로 매핑 기능을 활성화하면 클라이언트의 URL 을 분석해서 유레카에 등록된 서비스와 일치하는 서비스에 요청을 보낸다.


#### 수동 경로 매핑
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: service-a
          uri: lb://service-a
          predicates:
            - Path=/a/**
          filters:
            - RewritePath=/a/(?<remaining>.*), /$\{remaining}

        - id: service-b
          uri: lb://service-b
          predicates:
            - Path=/b/**
          filters:
            - RewritePath=/b/(?<remaining>.*), /$\{remaining}

```

discovery.locator 설정을 지우고 경로를 매핑 해주었다.

#### 라우팅 정보 보기

```yaml
management:
  endpoint:
    gateway:
      enabled: true # default value
  endpoints:
    web:
      exposure:
        include: gateway

```
위 처럼 actuator 노출 정보를 활성화 해주면

```
/actuator/gateway/routes
```
해당 url 로 라우팅 정보를 볼 수 있다.


#### 수동 경로 매핑 vs 자동 경로 매핑 
![[Pasted image 20250216180700.png|600]]
우선 위는 수동 경로 매핑의 라우팅 정보이다.
![[Pasted image 20250216181311.png]]

> 수동 경로 매핑 vs 자동 경로 매핑 어떤걸 사용해야할까?
- 좀 더 경험을 쌓아봐야겠지만 자동 경로 매핑은 유레카에 등록된 서비스 기준으로 url 을 자동으로 만들어주는 거 같다. 
- 서비스가 많이 자주 추가 된다면 자동 경로 매핑이 좋을 수 있지만 원하지 않는 서비스 url 이 노출 될 수 있다는 걸 생각해야 한다
	- 정확히는 모르지만 자동 노출되는 정보를 수동으로 제거할 수 있을지 모르지만 그렇게 사용
		할 바에는 수동 경로 매핑이 좋지 않을까?


## 스프링 클라우드 Gateway 뜯어보기

### Spring Webflux 의 구조

#### Gateway 이야기 하다가 웬 Webflux?
**Spring Cloud Gateway** 는 *Spring Webflux* 와 *project Reactor* 위에서 개발되었다. 따라서 Spring Webflux 의 구조를 어느 정도만 이해해보자

#### Spring Webflux 는 무엇일까
Spring Webflux 는 **완전한 논블록킹(Non-Blocking)** 아키텍처 프레임워크이다.
우리가 자주 쓰는 Spring MVC (Spring Framwork)는 블록킹 아키텍처를 기본으로 한다.

#### 블록킹? 논블록킹?
우리 **애플리케이션은** CPU  작업만을 하지 않는다. 파일 IO, JDBC API , DB API , 외부 API ...., 등 외부와의 통신을 위해 *I/O 작업도 많이 한다*

##### 블록킹 I/O 라면
I/O 작업을 할 때 쓰레드는 I/O 작업이 끝날 때까지 *대기한다*.
대기하는 쓰레드는 다른 작업을 하지 않는다.

##### 논블록킹 I/O 라면
I/O작업이 끝날 때까지 대기하지 않고 해당 쓰레드는 다른 작업을 맡아서 할 수 있어 블록킹 되지 않는다.

##### 뭐가 좋은가?
뭐가 좋은지는 딱잘라 말할 수 없다. **CPU Intensive** 한 애플리케이션이라면 논블록킹 아키텍처는 *성능에 이점을 보지 못하고*, ***불필요한 개발 난이도***만 올라갈 것이다.

하지만, 

**I/O Intensive한** I/O작업이 많거나 오래 걸리는 애플리케이션이라면 논블록킹 아키텍처는 높*은 처리율을 만족할 수 있다.*

#### 그래서 Spring MVC , Spring Webflux 많이 다른가? (구조상 차이)

**Spring MVC** 는 1 Request per Thread 를 원칙으로 한다.  따라서 블록킹 아키텍처인 것을 알 수 있다.

우리가 Spring 을 평소에 개발할 때 Request per Thread 의 구조를 많이 이용했다.
(HttpServletRequest,HttpServletResponse,ContextHolder, @Transactional....)

위 기능들은 하나의 요청에서 어떤 계층에서 사용하든 공유된다. 

이유는 ThreadLoacal을 사용하여 구현했기 때문이다.
>ThreadLocal은 쓰레드에 데이터를 저장하는 것이라고 생각하면된다.

##### Webflux는?
**Spring Webflux는** 하나의 요청에 1개의 쓰레드만 처리하지 않는다.
따라서 위에서 ThreadLocal로 캐싱하여 요청의 상태(Context)를 저장하는 것은 의미가 없다.


#### Spring Webflux 얕게 뜯어보기

![[Pasted image 20250217001128.png]]



#### 요청의 시작

```java
ApplicationContext context = ...;
HttpHandler handler = WebHttpHandlerBuilder.applicationContext(context).build();
```

WebHttpHandlerBuilder 에 의해 **HttpHandler** 는 빌드되고 HttpHandler는 Client의 요청을 ServerWebExchange로 변환하고 *DispatcherHandler 를 호출한다*

#### DispatcherHandler! Spring Webflux에서의 DispatcherServlet
DispatcherServlet는 Spring MVC에서 Client 요청을 처리할 수 있는 Handler 에게 라우팅해주는 역할을 맡았다.
그리고 처리된 응답을 client 에게 반환하는 역할을 맡기도 했다. 

**DispatcherHandler** 는 Webflux에서 위와 같은 동작을 맡고 있다.
<br>

**DispatcherHandler** 는  프로퍼티로 HandlerMapping  리스트 와 HandlerAdapter 리스트를 가지고 있다

```java
class DispatcherHandler{
	@Override  
	public Mono<Void> handle(ServerWebExchange exchange) {  
	    if (this.handlerMappings == null) {  
	       return createNotFoundError();  
	    }  
	    if (CorsUtils.isPreFlightRequest(exchange.getRequest())) {  
	       return handlePreFlight(exchange);  
	    }  
	    return Flux.fromIterable(this.handlerMappings)  
	          .concatMap(mapping -> mapping.getHandler(exchange))  
	          .next()  
	          .switchIfEmpty(createNotFoundError())  
	          .onErrorResume(ex -> handleResultMono(exchange, Mono.error(ex)))  
	          .flatMap(handler -> handleRequestWith(exchange, handler));  
	}
}


```


HandlerMapping 클래스에서 구현된 getHandler메서드로 해당 요청을 처리할 수 있는지 확인한다.
그리고 처음으로 true 인 HandlerMapping 을 처리한다.

만약 HandlerMapping 을 찾는 과정에서 오류가 나거나 없다면 다음 코드로 처리한다
```java
	  .switchIfEmpty(createNotFoundError())  
	  .onErrorResume(ex -> handleResultMono(exchange, Mono.error(ex)))  
```

Adapter 리스트를 다시 순회하여 .supports로 처리 가능한 Adapter를 찾는다. 찾으면 apapter의 *handler를 호출하고* 결과 값을 처리하는 **handleResultMono** 메서드를 실행한다

```java
private Mono<Void> handleRequestWith(ServerWebExchange exchange, Object handler) {  
    if (ObjectUtils.nullSafeEquals(exchange.getResponse().getStatusCode(), HttpStatus.FORBIDDEN)) {  
       return Mono.empty();  // CORS rejection  
    }  
    if (this.handlerAdapters != null) {  
       for (HandlerAdapter adapter : this.handlerAdapters) {  
          if (adapter.supports(handler)) {  
             Mono<HandlerResult> resultMono = adapter.handle(exchange, handler);  
             return handleResultMono(exchange, resultMono);  
          }  
       }  
    }  
    return Mono.error(new IllegalStateException("No HandlerAdapter: " + handler));  
}
```

#### [부록] Mono? Flux? 이게 뭐야

##### **1. 개요**

Spring WebFlux에서 `Mono`와 `Flux`는 **Reactive Streams**의 Publisher 역할을 수행하는 핵심 컴포넌트이다.

- **`Mono<T>`**: 0 또는 1개의 데이터만 처리하는 Publisher
- **`Flux<T>`**: 0개 이상의 데이터를 스트림 형태로 처리하는 Publisher

이들은 비동기적으로 데이터를 생성하고, Subscriber가 구독하면 데이터를 전달하는 **Pub-Sub 구조**를 따른다.

---

##### **2. Pub-Sub 구조**

###### **2.1 Publisher (`Mono` / `Flux`)**

- 데이터를 제공하는 역할
- `subscribe()`가 호출되기 전까지 실제 데이터 처리는 수행되지 않음

###### **2.2 Subscriber**

- 데이터를 요청하고(`request(n)`) 이를 소비
- `onNext()`, `onError()`, `onComplete()` 메서드를 통해 데이터 처리

###### **2.3 Subscription**

- Publisher와 Subscriber 간의 데이터 흐름을 조정
- `request(n)`을 호출하면 Publisher는 n개의 데이터를 전달

```java
Flux<String> flux = Flux.just("A", "B", "C"); 
flux.subscribe(System.out::println); // 구독이 발생해야 데이터가 전달됨
```
---

##### **3. 주요 동작 방식**

###### **3.1 동기 vs 비동기 실행**

`Mono`와 `Flux`는 기본적으로 동기 실행되지만, `publishOn()`, `subscribeOn()`을 사용하면 다른 스레드에서 비동기적으로 실행 가능하다.

```java
Flux.range(1, 5)
    .publishOn(Schedulers.boundedElastic()) // 새로운 스레드에서 실행
    .doOnNext(i -> System.out.println("Thread: " + Thread.currentThread().getId()))
    .subscribe();

```

###### **3.2 데이터 요청 흐름**

1. `doOnRequest()`를 사용하면 요청이 발생할 때 로그를 확인할 수 있다.
2. `publishOn()`은 Publisher 이후의 작업을 다른 스레드에서 실행하도록 함.

```java
Flux.range(1, 5)
    .doOnRequest(r -> System.out.println("Requested: " + r))
    .publishOn(Schedulers.parallel()) // 병렬 스레드에서 실행
    .subscribe(System.out::println);

```




### Spring Cloud Gateway 구조

#### 📌 Spring Cloud Gateway 기본 개념

1. **Route (경로)**
    
    - API Gateway의 기본 단위
    - 특정 조건(Predicate)이 만족되면, 지정된 목적지(URI)로 요청을 전달
    - 여러 개의 필터(Filter)를 거쳐 요청과 응답을 수정 가능
2. **Predicate (조건)**
    
    - HTTP 요청을 특정 기준(예: 경로, 헤더, 파라미터 등)으로 검사
    - 조건을 만족하는 요청만 해당 Route를 따라 이동
3. **Filter (필터)**
    
    - 요청 또는 응답을 수정하는 기능
    - ex) 인증, 로깅, CORS 처리, 요청 수정, 응답 캐싱 등
    - **Global Filter vs Gateway Filter**
        - **Global Filter**: 모든 요청에 적용됨
        - **Gateway Filter**: 특정 Route에서만 적용됨

#### Gateway의 HandlerMapping, HandlerAdapter, Handler

spring cloud gateway를 뜯어보기 앞서서 webflux 위에서 구현됐다면 Gateway 의 핵심 로직과 사용자의 요청을 라우팅 하기 위해서는 *HandlerMapping, HandlerAdapter, Handler를 구현해야한다.*

**Spring Cloud Gateway** 가 HandlerMapping, HandlerAdapter, Handler를 커스텀한 코드를 뜯어보자

##### 라우팅 구성하기 
위에서 Spring Cloud Gateway로 라우팅되는 매핑 정보를 커스텀한적이 있다.
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: service-a
          uri: lb://service-a
          predicates:
            - Path=/a/**
          filters:
            - RewritePath=/a/(?<remaining>.*), /$\{remaining}

```

여기서 라우트 매핑 정보를 뜯어보면 다음과 같다
- **id** : 라우트의 key 이자 이름
- **uri**:  목적 서비스의 url 현재 gateway 구성은 eureka를 적용해서 서비스 이름으로 적용 (lb는 로드밸런서 사용 유무)
- **predicate** : 해당 라우트를 적용할 조건 
- **filters** : gateway가 클라이언트의 요청을 백엔드 서비스에 넘기기 전에 혹은 백엔드로부터 응답을 받고 가공하는 공식? 같은것

> 위에 구성을 해석하면 service-a 라우트는 {gateway 주소}/a/{} 로 들어오는 요청을 //service-a/{} 로 요청을 가공해서 보낸다는 의미이다.

이제 Spring Cloud Gateway 가 어떻게 사용할지 감이 잡힐 것이다. 이제 Spring Cloud Gateway 가 어떻게 이를 구현했는지 보자

##### 클라이언트의 요청 흐름

![[Pasted image 20250217200215.png]]
>클라이언트의 요청을 webflux에서 설명했듯이 Netty 웹서버에서 시작하여 DispatcherHandler로 요청을 흘러갈 것이다.
>gateway 의 기능을 수행하기 위해서 Spring Cloud Gateway 가 구현한 Handler가  클라이언트에 Mapping 되어 Handler가 수행하고 백엔드 서비스로 요청을 보내야한다. 다음은 그 과정이다.


4. **DispatcherHandler** HandlerMapping 리스트를 순회하면서 *getHandler를* 호출하여 클라이언트의 요청을 처리할 수 있는 HandlerMapping을 찾는다.
5. 이때 webflux 의 HandlerMapping 추상 클래스인 **AbstractHandlerMapping** 를  gateway에서 구현한  **RoutePredicateHandlerMapping** 의  *getHandlerInternal* 를 호출한다.
6.  **RoutePredicateHandlerMapping** 는 lookupRoute를 호출하여 Route를 탐색한다.
7. **RoutePredicateHandlerMapping** 는 우리가 구성한 **Route** 구성을 조회할 수 있는 **RouteLocator** 를 주입 받는다.(Bean 초기화때)
8. **RoutePredicateHandlerMapping**는 **ReouteLocator** 로 Route List(우리가 구성파일로 설정한)를 순회하면서 getPredicate().apply(exchange)를 호출한다.
9. **Route마다** 정의된 **Predicate** 로 해당 요청이 Predicate 조건이 맞는지 확인한다. 조건에 해당하는 Route를 반환한다.
10. **RoutePredicateHandlerMapping** 은 요청에 해당하는 Route가 있으면 **DispatcherHandler**에게 *WebHandler의 gateway 구현체(FilteringWebHandler)를 반환한다* 이때 찾은 Route는 exchange에 캐싱하여  FilteringWebHandler가 해당 요청에 맞는 Route를 찾도록 한다.
11. **DispatcherHandler**  가 WebHandler의 *handler*를 호출한다.
12. **FilteringWebHandler** 가 Route(요청이 predicate 조건에 해당하는)에서 Filter List들을 가져오도록 getCombinedFilters(route) 호출한다.
13. **FilterChain.filter**를 호출한다.
14. 사전 필터 적용 - >Server 로 요청 -> Server의 응답 -> 사후 필터 적용 -> 클라이언트에게 응답


> 위 과정을 요약한 공식문서의 그림이다

![[Pasted image 20250217205248.png]]

  **Spring Cloud Gateway 다이어그램**  
클라이언트가 Spring Cloud Gateway로 요청을 보내면, Gateway Handler Mapping은 해당 요청이 특정 라우트와 매칭되는지 확인한다.  
매칭되는 경우, 요청을 Gateway Web Handler로 전달하며, 이 핸들러는 요청에 대해 특정한 필터 체인을 실행한다.

점선으로 구분된 이유는 필터가 프록시 요청 전후로 실행되기 때문이다.  
먼저 "사전(Pre)" 필터 로직이 실행되며, 이후 프록시 요청이 전송된다.  
프록시 요청이 완료된 후에는 "사후(Post)" 필터 로직이 실행된다.


### Spring Cloud Gateway 잘 사용하기 

#### Global Filter 커스텀하기

`GlobalFilter` 인터페이스는 `GatewayFilter`와 동일한 시그니처를 가지며, 모든 경로에 조건적으로 적용되는 특별한 필터이다.

`GlobalFilter`는 모든 경로에 공통적으로 적용되며, **filter chain**에 포함되어 처리된다.

`GlobalFilter`와 `GatewayFilter`의 순서는 `org.springframework.core.Ordered` 인터페이스에 의해 정해지며, `getOrder()` 메서드를 통해 순서를 설정할 수있다

Spring Cloud Gateway는 필터의 실행을 **pre-phase**와 **post-phase**로 구분하고, 높은 우선순위를 가진 필터가 먼저 실행된다.

```java
@Bean
public GlobalFilter customFilter() {
    return new CustomGlobalFilter();
}

public class CustomGlobalFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        log.info("custom global filter");
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return -1;
    }
}


```


#### Gateway Metrics Filter
gateway 는 actuator 의존성을 추가했을때 몇가지 정보를 노출시켜준다.


## 참고 자료
[How to Include Spring Cloud Gateway :: Spring Cloud Gateway](https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway/starter.html)

[DispatcherHandler :: Spring Framework](https://docs.spring.io/spring-framework/reference/web/webflux/dispatcher-handler.html)


