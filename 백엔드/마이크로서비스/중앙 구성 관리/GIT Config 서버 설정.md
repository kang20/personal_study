---
sticker: emoji//1f600
---
## Git 을 이용한 Spring Config 세팅

### Config Server 세팅 

#### Config Server 로서 선언
```java
@SpringBootApplication  
@EnableConfigServer  
public class ConfigserverApplication {  
  
    public static void main(String[] args) {  
       SpringApplication.run(ConfigserverApplication.class, args);  
    }  
  
}
```

#### ssh 접속을 위한 publicKey 와 privateKey 생성
```bash
$ ssh-keygen -m PEM -t ecdsa -b 256
```
- private 한 깃허브 레포에 ssh 로 접속하기 위한 private key 를 생성해야함
- RSA 기반 알고리즘 key 는 깃허브가 더이상 지원하지 않음 따라서 ecdsa 기반을 사용
![[Pasted image 20250204033128.png]]
config server 에서 key를 만들면 경로를 알 수 있음

> Enter passphrase (empty for no passphrase): -> 보안을 위해 입력할 수 있음 비밀번호 같은것임



![[Pasted image 20250204033247.png]]


**.pub 이 공개키이고 없는것이 개인키임**

#### 깃허브에 공개키 등록
![[Pasted image 20250204033359.png]]
config 레포에 위에 DeployKeys 에 저장함

#### config server yaml 설정

```bash
$ cat /{위치}/.ssh/id_ecdsa
```

```yaml
spring:  
  application:  
    name: config-server  
  profiles:  
    active: git  
  cloud:  
    config:  
      server:  
        git:  
#          uri: https://github.com/kang20/test_config_repo.git  
          uri: git@github.com:kang20/test_config_repo.git  
          host-key: private host key
          host-key-algorithm: ecdsa-sha2-nistp256  
          default-label: main  
          ignore-local-ssh-settings: true  
          private-key: |  
            -----BEGIN EC PRIVATE KEY-----  
			개인키 입력
            -----END EC PRIVATE KEY-----  
          passphrase: 입력
```


#### host key 
```bash
$ ssh-keyscan -t ecdsa github.com
```

호스트키로 출력하면 다음과 같은 형식이 반환됨
```
# ~~
{github.com} {알고리즘} {base64 기반 hostkey}
```
여기서 hostkey 부분만 추출하여 복붙하면 됨