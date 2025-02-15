---
sticker: emoji//1f600
---
## 구성(그리고 복잡성) 관리

### 마이크로서비스에서 왜 구성을 외부에서 관리할까
**마이크로 서비스 인스턴스는** 베포 되고 시작할 때 사람의 개입을 최소화 해야한다. 하지만 구성을 같이 배포하거나 직접 배포할 때 수동으로 관리하게 된다면, *애플리케이션 사이에 구성 불일치나 예기치 않은 장애, 확장 문제 대응을 위한 지연 시간 등이 발생할 수 있다*.

### 마이크로서비스 구성 관리에 대한 논의
- **분리** : 
	서비스의 물리적 배포에서 서비스 구성 정보를 완전히 분리 해야한다.
	*애플리케이션이 시작중일 때 전달되거나 서비스가 시작할 때 중앙 저장소에서 읽어 들여야 한다.*
- **추상화** : 구성 데이터를 직접 읽어오는 코드를 작성하기 보다 REST API 같이 서비스 인터페이스를 통해 읽어 와야한다.
- **중앙 집중화** : 수백개의 서비스가 존재할 수 있기 때문에 구성 데이터를 보관하는데 사용하는 여 저장소 수를 최소화하는 것이 중요하다.
- **견고화** : 중앙 집중화 되므로 *가용성이* 높아야 한다.

> 구성 정보를 실제 코드 외부로 분리할 때 외부 의존성이 생겨서 이를 관리하고 버전 제어한다는 것이다.



### 구성 관리 아키텍처

![[Pasted image 20250129185841.png]]

1. 마이크로서비스 인스턴스가 시작되면 서비스 엔드포인트를 호출하여 동작 중인 환경별 구성 정보를 읽어 온다.  *구성관리 서비스에 대한 접속 정보는 마이크로서비스가 시작할 때 전달된다* 
2. 실제 구성 정보는 저장소에 보관된다.  예를 들어 소스 제어되는 파일, 관계형 데이터베이스, 키-값 데이터베이스
3. 애플리케이션 구성 데이터의 실제 관리는 응용 프로그램이 배포되는 방식과는 독립적으로 한다. **구성 관리에 대한 변경 사항은 일반적으로 빌드 및 배포 파이프라인으로 처리되며 여기에 수정 사항에 대한 버전 정보는 태그를 달아 여러 환경에 배포할 수 있다.**
4. 관리하는 구성 정보가 변경되면 애플리케이션 구성 데이터를 사용하는 서비스는 변경사항을 통지받고 애플리케이션 데이터 복제본을 갱신해야 한다.


## 프로젝트에 외부 구성을 사용한 이유?
현재 프로젝트는 모놀로식 구조로 다수의 서비스가 하나의 코드베이스 위에서 하나의 서버로 배포되는 구조이다. **현재 프로젝트는 안정화 되어서 구성을 변경할 일이 적다.** 
하지만 프로젝트에 추가적인 서비스를 기획하고 있고 추가되는 서버스가 많은 자원을 소비할 예정이다. 그런데, 이러한 서비스들이 추가되었을 때 모놀로식 구조의 **현재 프로젝트는 SPOF를 유발하고 이는 장애 전파로 유발된다.**
>  SPOF는 "Single Point of Failure"의 약자로, 시스템에서 하나의 구성 요소가 실패하면 전체 시스템이 중단되는 취약점을 가리킨다


</br>


### SPOF를 유발하는 이유가 장애 뿐인가?
서비스의 장애만의 이유로 SPOF를 유발하지 않는다. A라는 서비스의 버전 업데이트로 인한 배포에도 B,C,D,,, 서비스와 같이 배포 되어야 한다.
*심지어 코드 수정이 아닌 애플리케이션의 구성 변경에도 서버의 shut down 은 전파된다.*
그래서 현재 큰 모놀로식을 단순히 분리하기 보다 가장 먼저 **구성부터 분리를 목적으로** 두었다.
또한, 추가되는 서비스에 의한 구성을 중앙에서 관리할 수 있게되어 **프로젝트 확장성 측면에서도** 도움이 될거라고 판단하였다.
### 외부 구성이 SPOF를 완전히 해결하는건가?
외부 구성이 신의 은탄환이라면 모놀로식이든 MSA든 외부 구성을 사용할 것이다. 물론, 단점도 존재한다는 의미이다.
#### >단점
- **복잡도** : 외부 구성 관리는 프로젝트의 복잡도가 낮아지는 것이 아니다. 구성 관리의 복잡도는 낮아질 수 있어도 전반적인 인프라의 복잡도는 올라갈 수 있다. 작은 프로젝트라면 오히려 복잡할 가능성이 높다.
- **여전한 SPOF** : SPOF를 해결하지 못한다. 구성을 한 지점에서 관리하고 모든 서비스는 Config Server 를 의존하기 때문에 Config Server는 고가용성을 보장 하거나, Client 들이 Config Server가 동작하지 않아도 문제가 없도록 조치를 취해야 한다..



## Spring Config 사용

### 어떻게 Spring 은 구성 관리를 지원할까?
우선 spring 에서 지원하는 Spring Config 기능을 알기 전에 구성 관리 기능을 지원하기 위해 무슨 기능이 필요할까
*다음 키워드에 포커스를 맞추어 보았다*
**견고성** , **엔드포인트** , **영속화** , **장애 대응** 

### Spring Config 구성의 이해

#### >Spring Config 는 어떻게 구성되어 있을까?

![[Pasted image 20250201171023.png]]

Spring cloud Config 는 Client 와 Server 로 구성되어 있다.

Instance는 실행되는 순간 Config Server에 API 호출을 통해 자신의 **service 명 , profile 명** 을 기준으로 구성 정보를 반환 받는다. 
*Config Server 의 구성이 변환되면 해당 구성을 사용하는 Instance 에게 이를 알린다.*
이런 과정을 Spring Cloud Config 는 어떻게 지원할까

#### >Spring Cloude Config Server 시작하기


##### 의존성 추가하기
```gradle
ext {  
    set('springCloudVersion', "2024.0.0")  
}

dependencies {  
    // config server  
    implementation 'org.springframework.cloud:spring-cloud-config-server'  
}

dependencyManagement {  
    imports {  
       mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"  
    }  
}
```


##### Spring Cloud Config Server 구성
![[Pasted image 20250201171803.png]]
Config Server 는 구성 정보를 저장할 저장소가 필요하다. 

저장소는 여러가지가 후보에 있을 수 있는데 Vault , 파일 시스템, git , RDB, key-value DB 등이 있다.

##### Spring Cloud Config Server 파일 시스템으로 시작하기

```yaml
spring:  
  application:  
    name: config-server  
  profiles:  
    active: native  
  
  cloud:  
    config:  
      server:  
        native:  
          search-locations: classpath:/config
```


위처럼 구성 파일들의 위치를 지정해야한다.

```java
@SpringBootApplication  
@EnableConfigServer  
public class ConfigserverApplication {  
  
    public static void main(String[] args) {  
       SpringApplication.run(ConfigserverApplication.class, args);  
    }  
  
}
```

@EnableConfigServer 어노테이션을 사용하여 Config Server 인 것을 명시하자


##### **Spring Cloud Config Server 의 Endpoint**
Config Server는 애플리케이션 *serivce 명과 profile* 명으로 엔드포인트를 제공하고 2개의 값 기준으로 파일을 찾는다.

```css
/{application}/{profile}[/{label}]
/{application}-{profile}.yml
/{label}/{application}-{profile}.yml
/{application}-{profile}.properties
/{label}/{application}-{profile}.properties

```

라벨은 git 브랜치 명을 의미한다.

```json
curl http://localhost:8071/test/dev

{
  "name": "test",
  "profiles": [
    "dev"
  ],
  "label": null,
  "version": null,
  "state": null,
  "propertySources": [
    {
      "name": "classpath:/config/test-dev.yml",
      "source": {
        "test.property": "i am test-dev"
      }
    },
    {
      "name": "classpath:/config/test.yml",
      "source": {
        "test.property": "i am default"
      }
    }
  ]
}

```


##### **Client 측 설정**
1. 똑같이 Config 의존성을 추가한다.

```
ext {  
    set('springCloudVersion', "2024.0.0")  
}

dependencies {  
    // config server  
    implementation 'org.springframework.cloud:spring-cloud-config-server'  
}

dependencyManagement {  
    imports {  
       mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"  
    }  
}
```


2. 자신의 Application name 과 profile 설정을 한다
```yaml
spring:  
  application:  
    name:  
      test  
  profiles:  
    active: dev  
```

3. Config Server  uri 를 추가한다.
```yaml
spring:  
  config:  
    import: optional:configserver:http://localhost:8071
```


###### 결과 
```json
   {
      "name": "configserver:classpath:/config/test-dev.yml",
      "properties": {
        "test.property": {
          "value": "******",
          "origin": "Config Server classpath:/config/test-dev.yml:2:13"
        },
        "test.db": {
          "value": "******",
          "origin": "Config Server classpath:/config/test-dev.yml:4:7"
        },
        "test.redis": {
          "value": "******",
          "origin": "Config Server classpath:/config/test-dev.yml:6:11"
        },
        "test.key": {
          "value": "******",
          "origin": "Config Server classpath:/config/test-dev.yml:8:9"
        },
        "test.application.test": {
          "value": "******",
          "origin": "Config Server classpath:/config/test-dev.yml:11:11"
        }
      }
    },
    {
      "name": "configserver:classpath:/config/test.yml",
      "properties": {
        "test.property": {
          "value": "******",
          "origin": "Config Server classpath:/config/test.yml:2:13"
        }
      }
    },

```


#### > Spring Cloud Bus 를 통한 구성 변경 감지

GitHub, GitLab, Bitbucket과 같은 소스 코드 저장소 제공자는 **웹훅(Webhook)** 기능을 제공한다.  
웹훅을 설정하면 저장소 변경 사항이 감지될 때 특정 URL로 이벤트가 전송된다.

##### 웹훅과 Config Server 연동

1. `spring-cloud-config-monitor` 라이브러리를 추가한다.
2. Spring Cloud Bus를 활성화한다.
3. Config Server에서 `/monitor` 엔드포인트가 활성화된다.

> 예를 들어, GitHub는 변경 사항이 발생하면 `POST` 요청을 보내고,  
> JSON 본문에는 변경된 커밋 목록이 포함되며,  
> `X-Github-Event: push` 헤더가 설정된다.

Config Server는 이 이벤트를 감지한 후 **RefreshRemoteApplicationEvent** 이벤트를 발생시킨다.  
이벤트는 변경된 애플리케이션을 대상으로 전송되며, 기본적으로 **애플리케이션 이름과 일치하는 설정 파일이 변경되었는지 검사**한다.

##### 기본 동작 방식

- `foo.properties` 파일이 변경되면 `foo` 애플리케이션이 갱신됨
- `application.properties` 파일이 변경되면 모든 애플리케이션이 갱신됨

##### 알림 동작 방식 변경

기본 동작을 변경하려면 `PropertyPathNotificationExtractor` 를 활용할 수 있다.  
이 클래스는 웹훅 요청의 헤더 및 본문을 받아서 변경된 파일 목록을 반환한다.

##### 직접 `/monitor` 호출하여 변경 사항 전파

GitHub, GitLab 등의 JSON 기반 웹훅 외에도 직접 변경 사항을 알릴 수도 있다.  
예를 들어, 다음과 같이 `POST` 요청을 보내면 특정 애플리케이션에 변경 사항을 알릴 수 있다.


`curl -X POST http://localhost:8888/monitor -d "path=foo"`

- `foo` 애플리케이션과 일치하는 설정을 가진 모든 인스턴스에 변경 사항을 전파한다.
- 패턴에 **와일드카드**를 사용할 수도 있다.

##### Git 로컬 저장소 감지

- Config Server가 **로컬 Git 저장소를 사용**하는 경우, 웹훅이 필요 없다.
- 설정 파일이 변경되면 자동으로 변경 사항이 감지되며, `RefreshRemoteApplicationEvent`가 전송된다.

### Spring Cloud Config Client 더 자세히 알기
Spring Boot 2.4에서는 `spring.config.import` 속성을 통해 설정 데이터를 가져오는 *새로운 방식을 도입*

```
spring.config.import=optional:configserver:
```
- 이렇게 하면 기본 위치(`http://localhost:8888`)의 Config Server에 연결
- <font color="#9bbb59"> `optional:` 접두사를 제거하면, Config Server에 연결할 수 없는 경우 Config Client가 실패</font> 만약 접속 실패시 오류를 출력하고 싶다면 optional을 제거하면 된다.

Config Server의 위치를 변경하려면 `spring.config.import` 에 직접 URL 을 추가할 수 있다.
```
spring.config.import=optional:configserver:http://myhost:8888
```

#### >Spring Boot Config Data 로딩 방식

Spring Boot Config Data는 두 단계로 설정을 로드:

1. **먼저 기본 프로필**(`default`)을 사용하여 모든 설정을 로드.
2. 이후 활성화된 프로필을 확인한 후, 해당 프로필에 대한 **추가 설정을 로드**.

이로 인해 Spring Cloud Config Server에 여러 번 요청이 발생할 수 있으며, 이는 정상적인 동작이다. 이전 버전에서는 한 번의 요청만 발생했지만, 이 방식에서는 Config Server에서 활성 프로필을 결정할 수 있다.

📌 **bootstrap 파일이 필요하지 않음**  
기존의 `bootstrap.properties` 또는 `bootstrap.yml`이 필요하지 않다.

> bootstrap 이란? 
> 만약 외부 구성 서버에서 설정을 가져와야 한다면 초기화하기 전에 최소한의 설정이 필요했다.
> 따라서 application.yml 보다 먼저 초기화하여 설정하는 bootstrap 을 사용했었다.
> 하지만 Spring Config Data 의 로딩 방식이 변경되면서 bootstrap은 사용하지 않게 되었다 (spring boot 2.4 이후)


| 방식                                                             | 설명                                                                                                                                                     |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1. Spring Boot Config Data 방식 (spring.config.import 사용, 최신 방식) | Spring Boot 2.4부터 도입된 방식으로, `spring.config.import` 속성을 활용하여 설정을 가져온다. `bootstrap.yml`이 필요하지 않다.                                                        |
| 1. Bootstrap 방식 (spring.cloud.bootstrap.enabled 사용, 레거시 방식)    | Spring Boot 초기 버전부터 사용되던 방식으로, `bootstrap.yml` 파일을 통해 설정을 로드한다. Spring Boot 2.4 이후에는 기본적으로 비활성화되어 있으며, `spring.cloud.bootstrap.enabled=true` 설정이 필요하다. |



#### > Config First vs Discoverey First (Config Server 위치 찾기)
Config Server 의 위치를 찾기 위해서 위처럼 URL을 전부 마이크로서비스에 삽입한다면 Config Server 의 위치가 변경한다면 일일히 모든 서비스의 URL도 변경해야하지 않을까?

##### Config First 방식
위와 동일하게 Config 물리 서버 url 을 입력한다.

##### Discovery First 방식
> Config Server의 위치를 **서비스 디스커버리(Eureka)를 통해 찾는 방식**

### **📌 특징**

- Config Server의 URL을 직접 설정하지 않고, **Eureka 서버를 통해 동적으로 찾는다**.
- `spring.cloud.config.discovery.enabled=true` 설정을 통해 **Eureka에서 Config Server를 검색**한다.
- Config Server가 수평 확장되거나 IP가 변경되어도 Eureka가 알아서 감지할 수 있다.
- **Config Server가 Eureka에 등록되어 있어야 한다.**

**✅ Discovery First 방식 설정 예시 (`bootstrap.yml`)**

```yaml
spring:
  cloud:
    config:
      discovery:
        enabled: true
        service-id: config-server

```
- `enabled: true` → Eureka에서 Config Server를 찾도록 설정
- `service-id: config-server` → Eureka에 등록된 Config Server의 서비스 ID



## 참고 문헌
[Spring Cloud Config 공식문서](https://docs.spring.io/spring-cloud-config/docs/current/reference/html/#_quick_start)





