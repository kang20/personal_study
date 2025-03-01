---
sticker: emoji//1f600
---
## 기존 아키텍처의 문제점
### 서비스 디스커버리란
**분산 아키텍처는** 아주 많은 호스트들로 분산되어 있어서 문리적인 호스트 이름과 머신이 위치한 IP주소를 알아야 한다. 
**이 개념은** 분산 컴퓨팅 초창기 때부터 존재했고 공식적으로 *'서비스 디스커버리'* 로 알려져 있다 
<br>

*서비스 디스커버리는* 애플리케이션이 사용하는 모든 원격 서비스의 주소가 포함된 프로퍼티 파일을 관리하는 것처럼 단순하거나 UDDI 저장소 처럼 정형화 되어 있는 것일 수 있다.

> UDDI 는 Universal Description Discovery and Integration 의 약자로 중앙에서 웹 서비스를 등록 및 검색을 위한 표준 프로토콜이다.

### 서비스 디스커버리가 마이크로서비스에 중요한 이유
1. **수평 확장**
2. **회복성**

#### 1. 수평 확장
**서비스 디스커버리를** 사용하면 애플리케이션 팀은 해당 환경에서 실행 중인 *서비스 인스턴스의 수를 빠르게 수평 확장할 수 있다는 장점이 있다*. 

기존에는 서비스 소비자는 항상 대상 서비스의 물리적 위치를 알고 있어야 했다. 하지만, 서비스 디스커버리 개념이 도입 된다면 *서비스 소비자는 추상화된 서비스의 위치만 알고 있으면 된다*.

따라서 새 서비스 인스턴스는 가용 **서비스 풀(pool)** 에 추가되거나 제거될 수 있다.

그리고 확장이 편하기 때문에 **수평적 확장**에 더욱 용이하다.

#### 2. 회복성
인스턴스가 비정상이거나 가용하지 못한 상태가 되면 대부분의 *서비스 디스커버리 엔진은 그 인스턴스를 가용 서비스 목록에서 제거한다*.

서비스 디스커버리 엔진은 사용이 불가한 서비스를 우회해서 라우팅하기 때문에 다운된 서비스로 입은 피해를 최소화한다.

### DNS 와 로드밸런서를 사용하면 더 쉽지 않나?

**서비스 디스커버리를** 사용하는 *수평 확장과 회복성은 DNS 와 로드밸런서만으로 보장할 수 있다* 그런데 왜 서비스 디스커버리일까?


## 서비스 위치 확인 (로드밸런서가 아닌 서비스 디스커버리인 이유)

### 기존 구조

![[Pasted image 20250206223842.png]]

서비스 소비자가 서비스의 위치를 찾기 위해서 DNS 또는 로드밸런서를 이용해 왔다.

이런 상황에서 애플리케이션의 **서버의 수**는 대개 **고정적이였고** **영속적이었다**.

**고가용성을** 위해 유후 상태의 보조 로드 밸런서는 *핑(ping) 신호를 보내 주 로드 밸런서가 살아 있는지 확인했다.*
-> 만약 살아있지 않다면 보조 로드밸런서는 주 로드밸런서의 IP 를 이어 받아 요청을 처리했다.


### 로드밸런서는 무엇이 잘못되었을까?

이러한 기존 모델은 고정적인 서버에서 실행되는 비교적 적은 수의 서비스에서 잘 동작하지만 클라우드 기반의 마이크로서비스 애플리케이션에서는 잘 동작하지 않는다. 이유는 다음과 같다


#### 1. SPOF
*고가용성 로드 밸런서를 만들 수 있더라도 전체 인프라스트럭처에서 보면 단일 장애 지점(SPOF)이다* 
로드 밸런서가 다운되면 의존하는 모든 애플리케이션도 다운된다.
**로드 밸런서를** *고가용성 있게 만들더라도* 애플리케이션 인프라스트럭처 안에서는 *중앙 집중식 관문이 될 가능성이 높다.*

#### 2. 로드 밸런서 수평 확장 능력의 제한 
대부분의 상용 로드밸런서는 이중화를 위해 **핫스왑 모델을** 사용하므로 로드를 처리할 서버 하나만 동작하고, ***보조 로드 밸런서**는 주 로드 밸런서가 다운된 경우 페일오버 만을 위해 존재한다.*

본질적으로 하드웨어 제약을 받는다. 또한 상용 로드 밸런서는 좀 더 가변적인 모델이 아닌 고정된 용량에 맞추어져 제한적인 라이선싱 모델을 갖는다.

> hot-swap 이란 운영 중인 시스템에 영향을 주지 않고 장치나 부품을 교체하는 것으로, 여기에서는 보조 로드 밸런서로 교체하는 것을 의미한다.

<br>

>💡ALB 같은 완전 관리형 로드밸런서는 Auto Scailing을 지원하긴  한다.

#### 3.전통적인 로드 밸런서 대부분은 고정적 관리된다.
이 로드 밸런서들은 서비스를 신속히 등록하고 취소하도록 설계되지 않았다.



#### 4. 로드밸런서 프록시 역할을 하므로 서비스 소비자 요청은 물리적 서비스에 매핑되어야 한다.
*수동으로 서비스 매칭 규칙을 정의하고 배포해야 하므로 서비스 인프라의 복잡성을 증가시킨다.*
전통적인 시나리오에서는 서비스 인스턴스가 시작되면 로드 밸런서에 등록되지 않는다. 


#### 로드 밸런서는 쓰면 안되는가?

고정적이다라는 점과 SPOF 라는 점이 단점이지만, 대부분의 애플리케이션 크기와 규모를 가진 기업 환경에서는 잘 동작한다.

프록시 적인 역할을 수행하면서 포트 제안, SSL , 최소 네트워크 접근 등 좋은 역할을 수행할 수 있다.


## 클라우드에서 서비스 디스커버리
### 서비스 디스커버리 메커니즘
#### 고가용성
*서비스 디스커버리는 핫 클러스터링 환경을 지원할 수 있어야 한다.*
서비스 디스커버리 클러스터에서 한 노드의 장애가 발생하면 클러스터 내에 다른 노드가 역할을 대신할 수 있어야 한다.
로드 밸런서와 통합된 클러스터는 서비스 중단을 방지하는 페일오버와 세션 데이터를 저장하는 세션 복제 기능을 제공할 수 있다.

#### P2P
서비스 디스커버리 클러스터의 모든 노드는 서비스 인스턴스의 상태를 상호 공유한다.

#### 부하 분산
서비스 디스커버리는 요청을 동적으로 분산시켜 관리하고 있는 모든 서비스 인스턴스에 분배해야 한다.

#### 회복성
**서비스 디스커버리 클라이언트는** 서비스 정보를 로컬에 캐싱 해야 한다.
서비스 디스커버리 서비스가 가용하지 않아도 애플리케이션은 여전히 작동할 수 있고 로컬 캐시에 저장된 정보를 기반으로 서비스를 찾을 수 있다.

#### 결함 내성
*서비스 디스커버리가 비정상 서비스 인스턴스를 탐지하면 클라이언트 요청을 처리하는 가용 서비스 목록에서 해당 인스턴스를 제거해야 한다.*


### 서비스 디스커버리 아키텍처
#### 주요 쟁점

1. **서비스 등록**
		서비스가 디스커버리 에이전트에 등록하는 방법
2. **클라이언트의 서비스 주소 검색**
		서비스 클라이언트가 서비스 정보를 검색하는 방법
3. **정보 공유**
		노드 간 서비스 정보를 공유하는 방법
4.  **상태 모니터링**
		서비스가 서비스 디스커버리에 상태를 전달하는 방법

#### 서비스 디스커버리 작동 원리
![[Pasted image 20250207024732.png]]
##### >서비스 인스턴스
- 서비스 인스턴스들은 서비스가 실행되면 자기 IP 주소를 서비스 디스커버리 에이전트에 등록한다.
- 서비스는 상태 정보를 디스커비리 에이전트에게 전달한다.
##### >클라이언트
- 클라이언트는 서비스 IP 주소를 직접 알지 못하는 대신 서비스 디스커버리 에이전트에서 주소를 얻는다.
- 디스커버리 에이전트에서 논리적 이름으로 서비스 위치를 검색한다.
##### >서비스 디스커버리 
- 서비스 디스커버리 노드는 서비스 상태 정보를 서로 공유한다.
- 서비스는 비정상 서비스를 제거한다.
- 클라이언트에게 서비스 물리 주소를 제공한다.
- 디스커버리 노드들 끼리 p2p 모델이나 gossip 같은 멀티캐스팅 프로토콜이나 전염식 프로토콜을 사용하여 클러스터에서 발생한 변경을 다른 노드가 발견할 수 있게 한다.

#### 위 방식의 단점 
위 방식을 요약하면 클라이언트는 *모든 요청마다 디스커버리에 의존적이며 디스커버리에게 요청을 한번 한 후에 ip 주소를 얻는다.* 
이러한 완전 의존은 취약한 방식이다.

#### 클라이언트 측 로드 밸런싱을 활용한 서비스 디스커버리 동작
![[Pasted image 20250207031154.png|500]]

##### 변경된 점
**클라이언트는** 디스커버리에서 얻은 *서비스의 IP 주소를 캐싱한다.* 그리고 클라이언트는 주기적으로 디스커버리와 주기적으로 소통한 후 데이터를 갱신한다.

클라이언트 캐시는 *궁극적으로 일관적이지만* 클라이언트가 ***서비스 디스커버리 인스턴스에 접속할 때 비정상 서비스 인스턴스를 호출할 위험은 항상 존재한다.***


>서비스를 호출하는 과정에서 서비스 호출이 실패하면 로컬에 있는 서비스 디스커버리 캐시가 무효화되고 서비스 디스커버리 클라이언트는 갱신을 시도한다.


## 참고
[스프링 마이크로서비스 코딩 공작소 - 예스24](https://www.yes24.com/Product/Goods/110243944)