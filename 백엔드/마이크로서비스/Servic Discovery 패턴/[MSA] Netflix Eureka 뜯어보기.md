---
aliases: []
---

## Netflix Eureka란

### Eureka ? Spring Cloud ? 뭔 상관인데? 
**Netflix Eureka** 는 Netflix 가 MSA 기반 아키텍처를 위해 개발한 오픈소스인 Netflix OSS 의 종류중 하나이다.

그리고 **Spring Cloud Discovery**는 Eureka 에 대한 *AutoConfiguration 와 함께 아주 간단하게 Netflix Eureka 를 이용한 Cloud 환경의 Servic Discovery를 구성할 수 있게 하나.*


### Eureka의 기능 구성
1. Instance 저장소 (메모리)
2. Instance 정보 등록, 검색, 삭제 , 갱신
3. 인스턴스 헬스 체크
4. 고가용성을 위한 클러스터링


핵심 구현체
```java
@Singleton  
public class PeerAwareInstanceRegistryImpl extends AbstractInstanceRegistry implements PeerAwareInstanceRegistry {
```

#### 1. Instance 저장소
 

**AbstractInstanceRegistry ( 실제 세부 구현)** 에 registry 가 구현되어 있음
```java
public abstract class AbstractInstanceRegistry implements InstanceRegistry {
	private final ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry = new ConcurrentHashMap();

....

```

`Lease` 는 임대 eureka는 registry에 작은 기간 동안 보관하는데 얼마동안 보관됐는지를 나타내는 클래스라고 생각하면 됨


```java
public class Lease<T> {  
    public static final int DEFAULT_DURATION_IN_SECS = 90;  
    private T holder;  
    private long evictionTimestamp;  
    private long registrationTimestamp;  
    private long serviceUpTimestamp;  
    private volatile long lastUpdateTimestamp;  
    private long duration;
```

`InstanceInfo` 는 service 의 메타 데이터이며 registry 에 등록될 핵심 데이터

key는 appName 으로 한다

대충 요약하면 다음 **데이터가** 있다
- 서비스 인스턴스 기본 정보
- 서비스 URL 정보
- 네트워크 & 포트 정보
- 상태 및 메타데이터 정보
- 등록 및 변경 시간 정보

> 요약하면 애플리케이션 메모리에 key(appName) : value(instanceInfo) 형태로 저장한다.


#### 2. Instance 정보 등록, 검색, 삭제

**등록**
Service Instance 가 실행되면 Eureka 서버에게 Instance 에 대한 정보와 service name(app name) 을 보내서 등록을 요구한다.

**검색**
client들은 Eureka 서버를 통해서 service의 물리 주소를 얻을 수 있다.

**삭제**
*정상적인 종료는* Client Instance 가 삭제 요청을 보낸다.

클래스: `com.netflix.discovery.DiscoveryClient`
```java
public synchronized void shutdown() {
    if (isShutdown.compareAndSet(false, true)) {
        logger.info("Shutting down DiscoveryClient ...");

        // 1. 헬스체크 스레드 중지
        cancelScheduledTasks();

        // 2. Eureka 서버에 등록 해제 요청 (HTTP DELETE 요청)
        unregister();

        // 3. 기타 리소스 정리
        eurekaTransport.shutdown();
    }
}

```

*비정상적인 종료는* Server 가 이를 감지한다.
관련 클래스: `com.netflix.eureka.registry.AbstractInstanceRegistry`
```java
protected void evict() {
    long additionalLeaseMs = serverConfig.getAdditionalLeaseExpirationTimeMs();
    long currentTime = System.currentTimeMillis();

    List<Lease<InstanceInfo>> expiredLeases = new ArrayList<>();

    for (Map.Entry<String, Lease<InstanceInfo>> entry : registry.entrySet()) {
        Lease<InstanceInfo> lease = entry.getValue();
        if (lease.isExpired(additionalLeaseMs, currentTime)) {
            expiredLeases.add(lease);
        }
    }

    for (Lease<InstanceInfo> lease : expiredLeases) {
        logger.warn("Evicting instance {}", lease.getHolder().getId());
        internalCancel(lease.getHolder().getAppName(), lease.getHolder().getId());
    }
}

// Lease 클래스의 메서드
public boolean isExpired(long additionalLeaseMs, long currentTimeMs) {
    return (currentTimeMs > (lastUpdateTimestamp + duration + additionalLeaseMs));
}


```

>비정상 종료에 대한 감지 시간의 기본값은 90초이다. 
>Eureka Server 는 90초 동안 Instance의 하트비트를 받지 못한다면 registry 에서 제거 한다.


#### 3. 인스턴스 헬스 체크

**Eureka Client 는** 정해진 시간마다 Eureka Server에게 하트비트 요청을 보내서 자신이 현재 활성화된 상태인지 알린다.

```java
// DiscoveryClient 의 메서드

boolean renew() {  
    try {  
	    // 하트 비트 요청
        EurekaHttpResponse<InstanceInfo> httpResponse = this.eurekaTransport.registrationClient.sendHeartBeat(this.instanceInfo.getAppName(), this.instanceInfo.getId(), this.instanceInfo, (InstanceInfo.InstanceStatus)null);  

		// 에러 뜨면 등록 요청
        if (httpResponse.getStatusCode() == Status.NOT_FOUND.getStatusCode()) {  
            this.REREGISTER_COUNTER.increment();  
          
            long timestamp = this.instanceInfo.setIsDirtyWithTime();  
            boolean success = this.register();  
            if (success) {  
                this.instanceInfo.unsetIsDirty(timestamp);  
            }  
  
            return success;  
        } else {  
            return httpResponse.getStatusCode() == Status.OK.getStatusCode();  
        }  
    } catch (Throwable var5) {  
        Throwable e = var5;  
        return false;  
    }  
}

```

> 기본 값 30초 마다 요청을 하트비트를 보낸다.

#### 4. 고가용성을 위한 클러스터링

![[Pasted image 20250209062321.png]]

**Eureka Server는** 고가용성을 위한 클러스터링을 지원한다.

Service 가 Eureka Node 1 에 등록을 요청하면 Eureka Node 1은 2,3,4 Node 들에게 똑같은 등록 요청을 보낸다.

```java

// PeerAwareInstanceRegistryImpl 클래스
  
private void replicateToPeers(Action action, String appName, String id, InstanceInfo info, InstanceInfo.InstanceStatus newStatus, boolean isReplication) {  
    long time = SpectatorUtil.time(action.getTimer());  
  
    try {  
        if (isReplication) {  
            this.numberOfReplicationsLastMin.increment();  
        }  
  
        if (this.peerEurekaNodes != Collections.EMPTY_LIST && !isReplication) {  
            Iterator var9 = this.peerEurekaNodes.getPeerEurekaNodes().iterator();  
  
            while(var9.hasNext()) {  
                PeerEurekaNode node = (PeerEurekaNode)var9.next();  
                if (!this.peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {  
                    this.replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);  
                }  
            }  
  
            return;  
        }  
    } finally {  
        SpectatorUtil.record(action.getTimer(), time);  
    }  
  
}
```
클러스터 Node를 순회하면서 자기 자신을 제외하고 register 요청을 보내는 것을 볼 수 있다.

##### 4-알파)  FailOver

*장애복구 기능 또한 제공한다.*

- Eureka 서버는 다운되거나 재부팅하게 된다면 peer 에게 데이터를 동기화하는 SyncUp 메서드를 동작한다 (단일 노드는 해당 안됨)
- 단일 노드라도 하트비트를 클라이언트가 계속 보내고 있다면 재등록된다.





## 참고 
[Netflix/eureka: AWS Service registry for resilient mid-tier load balancing and failover.](https://github.com/Netflix/eureka)
