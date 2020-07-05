# 작업 중 입니다 ....(일요일 완성 예정)

Tanda(택시예약 시스템)
======================
본 시스템은 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 시스템입니다.
  
## Table of contents

 * 택시예약관리 시스템
  * 소스코드
  * 서비스 시나리오
    * 기능적요구사항
    * 비기능 요구사항
    * KPI
  * 분석/설계
    * 이벤트 스토밍 결과
    * 이벤트 도출 (액터, 커맨드, 폴리시)
    * 어그리게잇으로 묶기
  * 구현:
    * DDD 의 적용
    * 적용후 Test(Sample) : 비 동기식 호출 과 Fallback 처리
    * 동기식 호출 / 장애격리 / Eventual Consistency
  * 운영
    * CI/CD 설정
    * 서킷 브레이킹 / 장애격리
    * 오토스케일 아웃
    * 무정지 재배포
    
 ## 택시예약관리 시스템
  ![탄다](https://user-images.githubusercontent.com/63759255/86345665-cc7bc080-bc96-11ea-9b94-c3ae43cef05a.jpg)
 
 
 ## 소스코드
  | No | Service Name| Github Address |
  | :--------: | :--------: | :--------: |
  | 0 | Readme | https://github.com/tanda-msa/tanda |
  | 1 | gateway | https://github.com/jamesby99/gateway.git | 
  | 2 | book | https://github.com/one7ime/book.git | 8081 | 
  | 3 | taxi | https://github.com/choijuho/taxi.git | 8082 | 
  | 4 | pay | https://github.com/choijuho/taxi.git | 8083 | 
  | 5 | cqrs | https://github.com/jamesby99/cqrs.git | 8084 | 

 
 ## 서비스 시나리오
 
 ### 기능적 요구사항
 
   - 고객이 택시를 요청한다
   - 기사가 배차를 승낙한다
   - 택시가 배차 된다
   - 배차가 되면 고객에게 도착 시간을 알린다
   - 가용한 차량이 없으면 고객의 요청이 취소된다
   - 차량에 탑승하면 운행중으로 상태가 바뀐다
   - 차량에서 하차하면 운행종료로 상태가 바뀐다
   - 하차시 요금이 자동 결제 된다
   - 고객은 배차 요청을 탑승전까지 취소할 수 있으며 택시가 배차상태라도 배차가 취소된다
 
### 비기능 요구사항
 
  1. 트랜잭션
   고객 배차 취소 요청은 비동기 방식이나, Saga Pattern을 적용하여 최종 일관성을 유지한다.  
  2. 장애격리
     - 결제시스템(Sync연동)이 과중되면 Circuit breaker(FeignClient, Hystrix) 동작 및 운행종료(fallback)로 넘어가지 않는다.
     - 요금 결제가 되지 않으면 운행종료로 넘어가지 않는다 (sync호출)
     - 구현(taxi 서비스)
  3. 성능 
     - 서포트 기능(CPQR) 이 수행되지 않더라도 차량 요청 및 배차는 365일 24시간 받을 수 있어야 한다      Async (event-driven), Eventual Consistency
     - 가용한 차량이 없으면 사용자를 잠시동안 받지 않고 잠시후에 호출하도록 유도 (circuit breaker)
     - 차량운행 상태의 변경시 알림을 준다 (Event driven)
  4. 기타
     - 결제(Sync) 외 비동기, 운영의 유연성 및 안전성 확보(ConfigMap, Secrets)
   
 ###  KPI 
  1. 예약 서비스(book)
     - 타 시스템의 장애 여부와 상관없이 택시 요청에 대한 응답을 놓치지 않는다

 2. 차량 운행 서비스 (Taxideploy)
     - 모든 요청에 대해 1분이내 배차/재요청 응답한다.

 3. CORS
     - 예약 배차 관련 모든 상태 정보를 전달한다.
  
#### 사용자 UI (예상 Sample)
  ![Taxi 호출 UI3](https://user-images.githubusercontent.com/63759255/86532850-ef60db80-bf07-11ea-9508-dfe82f9bb8b9.jpg)

## 분석/설계

### 이벤트 도출 (액터-노랑, 커맨드-파랑, 이벤트-빨강, 폴리시-초록)

![이벤트스토밍](https://user-images.githubusercontent.com/63759255/86353291-fbe3fa80-bca1-11ea-9461-5f71c24e84ae.jpg)


### Event Storming 최종 결과

* MSAEz 로 모델링한 이벤트스토밍 결과

![이벤트설계0701_1](https://user-images.githubusercontent.com/63759255/86527629-e6f2ab80-bedb-11ea-94d4-96aa13d26c25.png)


### 어그리게잇으로 묶기
```
  . Core Domain : 예약 , 차량관리
  . Supporting Domain : CQRS
  . General Domain : 결제  
```

각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다

1. 예약 서비스 (Book)
```
public class Book {
	private Long bookId; // ID : 자동생성
	private String customerInfo; // 고객정보 : "유병전/전화번호/카드번호"
	private String departmentLoc; // 출발지 : 분당KT본사
	private String arrivalLoc; // 도착지 : 강남역
	private String bookStatus; // 둘중 하나 : 접수,취소(고객),취소(시스템)
	private LocalDateTime lastModifyTime; // DB INSERT, UPDATE Time으로 @PreUpate Hook에서 셋팅
	private boolean confirmed = false; 
}
```

2. 차량 관리 서비스 (TaxiDispatch)
```
public class TaxiDispatch {
	private Long dispatchId; // ID : 자동생성
	private Long bookId; // Book Entity와 relation
	private String taxiInfo; // 배차된 택시정보 : 차번호/전화번호
	private String dispatchStatus; // 배차됨, 운행시작됨, 운행종료됨, 배차취소됨
	private int price; //운행종료후 API로 전달됨
	private LocalDateTime lastModifyTime; // DB INSERT, UPDATE Time으로 @PreUpate Hook에서 셋팅
```

3. CQRS 서비스 (CQRS)
```
public class Kakao {
private Long kakaoId;					//ID 	: 자동생성
	private Long bookId;					//Book Entity와 relation
	private String receiver;				//customerInfo or taxiInfo
	private String msg;
	private LocalDateTime lastModifyTime;	//DB INSERT, UPDATE Time으로 @PreUpate Hook에서 셋팅 
}
public class RideHistory {		
	private Long bookId;					//entity ID
	private Long dispatchId;				
	private String customerInfo;			//고객정보 : "유병전/전화번호"
	private String departmentLoc;			//출발지	: 분당KT본사
	private String arrivalLoc;				//도착지	: 강남역
	private String taxiInfo;				//배차된 택시정보 : 차번호/전화번호
	private String bookStatus;				//둘중 하나 : 접수,취소
	private String dispatchStatus;			//배차됨, 운행시작됨, 운행종료됨, 배차취소됨
	private int price;						//결제된 택시요금
	private LocalDateTime lastModifyTime;	//DB INSERT, UPDATE Time으로 @PreUpate Hook에서 셋팅 
}
```

4. 결제 서비스 (Pay)
```
public class Pay {		
	private Long payId;						//ID 	: 자동생성
	private Long bookId;					//Book Entity와 relation
	private Long dispatchId;				//Dispatch Entity와 relation
	private int	 price;						//결제된 택시요금
}
```

### 헥사고날 다이어그램
![헥사고날 다이어그램2](https://user-images.githubusercontent.com/63759255/86530692-c6cfe600-bef5-11ea-8124-c9a75d0c2889.png)

  
## 구현(github 소스 참고)

### DDD 의 적용
분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 
스프링부트와 H2 DB(예약 서비스),MySQL(배차,CQRS 서비스) 및 Kafka 클러스트(공통)를 통해 AWS Cloud 상에 구현하였다.
구현한 각 서비스에서 사용된 포트 정보는 아래와 같다. (각자의 포트넘버는 8081 ~ 808n 이다)
로컬 PC 에서 구현한 소스를 빌드업하고 Dockerising 하여 AWS 클라우드의 Kubernetes 클러스트에 배포

| No | Service Name| Github Address | Port | Describe |
| :--------: | :--------: | :--------: | :-------- | :-------- |
| 1 | gateway | https://github.com/jamesby99/gateway.git | 80 | 게이트웨이 |
| 2 | book | https://github.com/one7ime/book.git | 8081 | 예약서비스 |
| 3 | taxi | https://github.com/choijuho/taxi.git | 8082 | 차량배차 서비스 |
| 4 | pay | https://github.com/choijuho/taxi.git | 8083 | 결제서비스 |
| 5 | cqrs | https://github.com/jamesby99/cqrs.git | 8084 | CQRS(VIEW) 서비스 |


각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity로 선언하였다

### 업무 프로세스 흐름도

![업무 프로세스 흐름도 ](https://user-images.githubusercontent.com/63759255/86529658-5755f880-beed-11ea-94a0-dd9e516a10d4.png)


### 폴리글랏 프로그래밍
| No | Service Name| db | skill stack | etc |
| :--------: | :--------: | :--------: | :-------- | :-------- |
| 1 | gateway | mysql | Spring gateway |  |
| 2 | book | H2 | Spring, JPA, Lombok |  |
| 3 | taxi | mysql | Spring, JPA, Lombok, openfeign |   |
| 4 | pay | mysql | Spring, JPA, Lombok |  |
| 5 | cqrs | mysql | Spring, JPA, Lombok |  |
#### H2 DB application.properties
```yml
  datasource:
    driver-class-name: org.h2.Driver
    url: jdbc:h2:tcp://${_DATASOURCE_ADDRESS}/${_DATASOURCE_TABLESPACE}
    username: ${_DATASOURCE_USERNAME}
    password: ${_DATASOURCE_PASSWORD}  
```
#### H2 DB application.properties
```yml
  datasource:
    url: jdbc:mysql://${_DATASOURCE_ADDRESS}/${_DATASOURCE_TABLESPACE}?autoReconnect=true&serverTimezone=JST&useUnicode=true&characterEncoding=UTF-8
    username: ${_DATASOURCE_USERNAME}
    userpassword: ${_DATASOURCE_PASSWORD}
    driverClassName: com.mysql.cj.jdbc.Driver
```
#### 보안을 위한 Kubernetes secret properties (secret.yml)
```yml
apiVersion: v1
kind: Secret
metadata:
  name: cqrs-secret
data:
  db-user: dGFuZGFjcXJzCg==
  db-pw: dGFuZGEyMDIwCg==
```
#### skill set dependencies (pom.xml)
##### H2 DB
```yml
<dependency>
  <groupId>com.h2database</groupId>
  <artifactId>h2</artifactId>
  <scope>runtime</scope>
</dependency>
```
##### Mysql DB
```yml
<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
  <scope>runtime</scope>
</dependency>
```
##### Lombok
```yml
<dependency>
  <groupId>org.projectlombok</groupId>
  <artifactId>lombok</artifactId>
  <optional>true</optional>
</dependency>
```
##### openfeign
```yml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```
etc..

### 적용후 Test(AWS k8s 환경에서 시험)

* 예약서비스에서 예약요청
![택시요청](https://user-images.githubusercontent.com/63759255/86531256-a35b6a00-befa-11ea-9a91-0b4613d1cc4d.png)
![택시요청db1](https://user-images.githubusercontent.com/63759255/86531307-12d15980-befb-11ea-8eb5-2f801722fbf4.png)

* 택시 배차요청 수신 확인
![배차중2](https://user-images.githubusercontent.com/63759255/86531268-ba01c100-befa-11ea-82d8-8f432ff3dcab.png)
* 배차 완료 확인  
![배차됨2](https://user-images.githubusercontent.com/63759255/86531267-b8d09400-befa-11ea-877d-25581c1232a2.png)
* 운행 시작 확인
![운행시작2](https://user-images.githubusercontent.com/63759255/86531269-ba9a5780-befa-11ea-878d-c274c2310a2e.png)
* 운행 종료 확인
![운행완료2](https://user-images.githubusercontent.com/63759255/86531271-ba9a5780-befa-11ea-8f48-3e4793db79ed.png)
* 결제확인
![페이처리2](https://user-images.githubusercontent.com/63759255/86531272-bb32ee00-befa-11ea-9c3b-bf1c290015f4.png)
* 예약서비스에서 고객발 취소 요청
![택시취소](https://user-images.githubusercontent.com/63759255/86531266-b79f6700-befa-11ea-8a1c-01e96a16c9c3.png)
![택시요청db2](https://user-images.githubusercontent.com/63759255/86531259-ab1b0e80-befa-11ea-83b4-d493ae9547f0.png)
<<<<<<< HEAD
![택시취소DB](https://user-images.githubusercontent.com/63759255/86531821-79587680-beff-11ea-87fd-185e74690eb2.png)
* 취소처리 확인
![취소처리1](https://user-images.githubusercontent.com/63759255/86531336-53c96e00-befb-11ea-8ea4-922cfbdec305.PNG)
* CQRS 확인
![ridehistories](https://user-images.githubusercontent.com/63759255/86531662-24683080-befe-11ea-950a-fc6e8451fca8.PNG)
* kakao 서비스 확인
![kakao](https://user-images.githubusercontent.com/63759255/86531660-23370380-befe-11ea-9069-788f0fc61d95.PNG)
* 각 서비스에서 메세지 Pub/Sub 전송 확인
![kafka test2](https://user-images.githubusercontent.com/63759255/86353866-f0450380-bca2-11ea-9bbf-da248b5f5fc0.png)
* CQRS 프로젝션 예시
[](url)
![CQRS](https://user-images.githubusercontent.com/63759255/86534772-21793a00-bf16-11ea-9646-fe166cbaa8bc.png)

### 동기식 호출 / 장애격리 / Eventual Consistency
#### Saga Patter 적용(비기능요구사항-1,3 트랜젝션 )
   ![비기능요구사항1](https://user-images.githubusercontent.com/63759255/86529261-48217b80-beea-11ea-8e3b-bfdd3b31d7ef.png)

#### Feign Client 구현(비기능요구사항-2,3,4 장애격리, Request-response구현)  
##### 1. dependency 추가(pom.xml)
```xml
<dependency>
<groupId>org.springframework.cloud</groupId>
<artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
 ```    
##### 2. FeignClient Enabling(App.java)
```java
@EnableFeignClients
public class App {
```    
##### 3. FeignClient 인터페이스 생성(PayService.java) 
```java
@FeignClient(name = "pay", url = "${api.url.pay}")
public interface PayService {
@RequestMapping(method = RequestMethod.POST, path = "/pays", consumes = "application/json")
void billRelease(Pay pay);
}
```    
##### 4. @PreUpdate (결제완료처리 전) 결제모듈 실행(TaxiDispatch.java)   
```java
Pay pay = new Pay();
pay.setBookId(f.getBookId()); 
pay.setDispatchId(f.getDispatchId());
pay.setPrice(this.getPrice()); 
try {
PayService payService = App.applicationContext.getBean(PayService.class);
payService.billRelease(pay);					
} catch (Exception e) {
    throw new RuntimeException(String.format("결제실패가 실패했습니다(%s)\n%s", this, e.getMessage()));
}
```
## Gateway를 외부에서 클러스터내 마이크로 서비스 API 호출 구현
```
spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: book
          uri: http://book:8081
          predicates:
            - Path=/books/**
        - id: taxi
          uri: http://taxi:8082
          predicates:
            - Path=/taxiDispatches/**
        - id: pay
          uri: http://pay:8083
          predicates:
            - Path=/pays/**
        - id: CQRS
          uri: http://cqrs:8084
          predicates:
            - Path=/rideHistories/**, /kakaos/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true
```
## 쿠버네티스 클러스터에 구축 내용
* POD
```
NAME                           READY   STATUS    RESTARTS   AGE
pod/book-67bdcd9c85-8fjl4      1/1     Running   0          139m
pod/book-67bdcd9c85-d9zjz      1/1     Running   0          139m
pod/book-67bdcd9c85-npnfj      1/1     Running   0          138m
pod/book-67bdcd9c85-wwm92      1/1     Running   0          138m
pod/cqrs-85cd7575ff-r2kr6      1/1     Running   0          146m
pod/cqrs-85cd7575ff-vp8s4      1/1     Running   0          146m
pod/gateway-7668c7d7c8-5h5v2   1/1     Running   0          143m
pod/gateway-7668c7d7c8-cgg7s   1/1     Running   0          143m
pod/gateway-7668c7d7c8-ljkht   1/1     Running   0          143m
pod/pay-8574d58dfd-6r47s       1/1     Running   0          132m
pod/pay-8574d58dfd-947qg       1/1     Running   0          133m
pod/pay-8574d58dfd-lg6mp       1/1     Running   0          133m
pod/taxi-74559c9c9b-728fz      1/1     Running   1          76m
pod/taxi-74559c9c9b-fv7l8      1/1     Running   0          76m
pod/taxi-74559c9c9b-s72hr      1/1     Running   0          76m
pod/taxi-74559c9c9b-vjjrp      1/1     Running   0          76m
```
* Service
```
NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP                                                                    PORT(S)        AGE
service/book         ClusterIP      10.100.157.234   <none>                                                                         8081/TCP       3h5m
service/cqrs         ClusterIP      10.100.185.213   <none>                                                                         8084/TCP       12h
service/gateway      LoadBalancer   10.100.202.86    a41dcd284d4994f898ece7716a77ab39-1598962157.ap-northeast-1.elb.amazonaws.com   80:32378/TCP   2d9h
service/kubernetes   ClusterIP      10.100.0.1       <none>                                                                         443/TCP        3d10h
service/pay          ClusterIP      10.100.229.18    <none>                                                                         8083/TCP       177m
service/taxi         ClusterIP      10.100.94.200    <none>                                                                         8082/TCP       76m
```
* Deployment와 Replica
```
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/book      4/4     4            4           3h5m
deployment.apps/cqrs      2/2     2            2           146m
deployment.apps/gateway   3/3     3            3           2d9h
deployment.apps/pay       3/3     3            3           177m
deployment.apps/taxi      4/4     4            4           76m

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/book-67bdcd9c85      4         4         4       139m
replicaset.apps/cqrs-85cd7575ff      2         2         2       146m
replicaset.apps/gateway-7668c7d7c8   3         3         3       143m
replicaset.apps/pay-8574d58dfd       3         3         3       133m
replicaset.apps/taxi-74559c9c9b      4         4         4       76m
```
* Autoscaler
```
NAME                                          REFERENCE            TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/book      Deployment/book      <unknown>/50%   3         20        4          3h5m
horizontalpodautoscaler.autoscaling/cqrs      Deployment/cqrs      <unknown>/50%   1         10        2          12h
horizontalpodautoscaler.autoscaling/gateway   Deployment/gateway   <unknown>/50%   2         10        3          143m
horizontalpodautoscaler.autoscaling/pay       Deployment/pay       <unknown>/50%   2         20        3          177m
horizontalpodautoscaler.autoscaling/taxi      Deployment/taxi      <unknown>/50%   3         20        4          76m
```
## 운영

### CI/CD 설정
Github, Maven Build, Dockerlize, Kubernetes 배포의 전과정을 직접 익힐 수 있도록 Dockerfile과 Kubernetes yml 파일을 직접 작성하고 이를 자동으로 일괄 수행되도록 리눅스 쉡스크립트를 작성하여 CI/CD를 구현하였다.
* 작성한 쉡스립트 ( cicd.sh )
```
#!/bin/sh

_MSA='taxi'
_GIT_HUB='https://github.com/choijuho/taxi.git'

echo "${_MSA}의 CI/CD 작업을 시작합니다."

echo "[1단계]: 기존 소스 삭제 및 git clone..."
echo "> git주소: ${_GIT_HUB}"
rm -rf ./${_MSA}
git clone ${_GIT_HUB} > ./logs/${_MSA}-git.log 2>&1

cd ./${_MSA}

echo "[2단계]: maven package 작업 중..."
mvn clean > ../logs/${_MSA}-mvn-clean.log 2>&1
mvn package > ../logs/${_MSA}-mvn-package.log 2>&1

echo "[3단계]: docker build&push 작업 중..."
echo "> build image name: jamesby99/${_MSA}:latest"
docker build -t jamesby99/${_MSA}:latest . > ../logs/${_MSA}-docker-build.log 2>&1
docker push jamesby99/${_MSA}:latest

echo "[4단계]: 변경사항을 kubernetes에 반영합니다."
kubectl apply -f ${_MSA}.yml

echo "${_MSA}의 CI/CD 작업이 완료되었습니다."
``` 
* 수행결과
```
ubuntu@ip-172-31-15-195:~/tanda/taxi$ ./cicd.sh
taxi의 CI/CD 작업을 시작합니다.
[1단계]: 기존 소스 삭제 및 git clone...
> git주소: https://github.com/choijuho/taxi.git
[2단계]: maven package 작업 중...
[3단계]: docker build&push 작업 중...
> build image name: jamesby99/taxi:latest
The push refers to repository [docker.io/jamesby99/taxi]
6ee47f0f3a68: Pushed
ceaf9e1ebef5: Layer already exists
9b9b7f3d56a0: Layer already exists
f1b5933fe4b5: Layer already exists
latest: digest: sha256:bcd2c84fa21d06bd72d00baf03eedd20b5391026aa14bc4973715d7356014258 size: 1159
[4단계]: 변경사항을 kubernetes에 반영합니다.
deployment.apps/taxi created
service/taxi created
horizontalpodautoscaler.autoscaling/taxi created
taxi의 CI/CD 작업이 완료되었습니다.   
```

### 서킷 브레이킹 / 장애격리
서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용
* 동기 수신단에서는 처리후 아래와 같은 랜덤 지연 부하를 만들어줌
```
	@PostPersist
	protected void payAccepted() throws InterruptedException {
		PayRepository repository = App.applicationContext.getBean(PayRepository.class);

		repository.findById(payId).ifPresent(f -> {
			BillIssued bi = new BillIssued();
			bi.setPayId(f.getPayId());
			bi.setDispatchId(f.getDispatchId());
			bi.setBookId(f.getBookId());
			bi.setPrice(f.getPrice());
			bi.setLastModifyTime(f.getLastModifyTime());
			KafkaSender.pub(bi);
		});
		
        try {
            Thread.sleep((long) (400 + Math.random() * 220));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
		

	}
```
* FeignClient + Hystrix 형태로 구성된 interface가 호출될 수 있도록 API Call 구성하여 테스트
```
siege -c255 -t60S -r10 --content-type "application/json" 'a41dcd284d4994f898ece7716a77ab39-1598962157.ap-northeast-1.elb.amazonaws.com/taxiDispatches/1 PATCH {"dispatchStatus":"운행종료됨", "price":"10000"}'
```
* Failed transactions: 43건과 Longest transaction: 32.30이나 서킷 브레이크 동작으로 준수한 성공률 유지
```
[error] CONFIG conflict: selected time and repetition based testing
defaulting to time-based testing: 60 seconds
** SIEGE 4.0.4
** Preparing 255 concurrent users for battle.
The server is now under siege...
Lifting the server siege...
Transactions:                   1883 hits
Availability:                  97.77 %
Elapsed time:                  59.08 secs
Data transferred:               0.86 MB
Response time:                  7.08 secs
Transaction rate:              31.87 trans/sec
Throughput:                     0.01 MB/sec
Concurrency:                  225.55
Successful transactions:        1883
Failed transactions:              43
Longest transaction:           32.30
Shortest transaction:           0.41
```

### 오토스케일 아웃
* 부하에 오토스케일을 확인하기 위해 아래 1개 POD로부터 시작하며, 최대 20개 POD까지 증가되도록 구성
```
NAME                           READY   STATUS    RESTARTS   AGE
pod/book-67bdcd9c85-d9zjz      1/1     Running   0          4h8m

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/book      1/1     1            1           4h55m

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/book-67bdcd9c85      1         1         1       4h8m

NAME                                          REFERENCE            TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/book      Deployment/book      <unknown>/10%   1         20        1          4h55m
```
* 부하 명령어
```
siege -c255 -t240S -r10 --content-type "application/json" 'a41dcd284d4994f898ece7716a77ab39-1598962157.ap-northeast-1.elb.amazonaws.com/books POST {"customerInfo="박성홍/010-1122-0001/1111-2222-3333-0001","departmentLoc":"김포","arrivalLoc":"마포" }'     
```
* Replica 변화 측정하였으나, 부하가 부족했는지 아니면 비동기 방식의 마이크로서비스의 성능이 좋아서 그런지 오토스케일은 확인하지 못함. 동기식대비 수십배 트랙잭션 수용
```
k get deploy book -w
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
book   1/1     1            1           4h58m
```
```
Transactions:                  96386 hits
Availability:                  99.99 %
Elapsed time:                 239.92 secs
Data transferred:              54.33 MB
Response time:                  0.62 secs
Transaction rate:             401.74 trans/sec
Throughput:                     0.23 MB/sec
Concurrency:                  250.38
Successful transactions:           0
Failed transactions:              14
Longest transaction:           30.29
Shortest transaction:           0.00
```
### 무정지 재배포
* 5개 POD로 먼저 구성하고, docker image가 변경되도록 하여 재배포를 수행한다. 이때 부하를 줌으로써 서비스영향을 측정한다.
```
ubuntu@ip-172-31-15-195:~/tanda/book$ k get po
NAME                       READY   STATUS    RESTARTS   AGE
book-67bdcd9c85-2qpbw      1/1     Running   0          59s
book-67bdcd9c85-d7ljl      1/1     Running   0          59s
book-67bdcd9c85-kllqq      1/1     Running   0          59s
book-67bdcd9c85-ph7m5      1/1     Running   0          59s
book-67bdcd9c85-wcsp5      1/1     Running   0          59s
```
* 해당 서비스의 yml에는 livenessProbe와 readinessProbe가 동작되도록 반영하였다.
```
      containers:
      - name: book
        image: jamesby99/book:latest
        livenessProbe:
          httpGet:
            path: /
            port: 8081
          initialDelaySeconds: 5
          periodSeconds: 15
          timeoutSeconds: 2            
        readinessProbe:
          httpGet:
            path: /
            port: 8081
          initialDelaySeconds: 5
          timeoutSeconds: 1                      

```
* 미미한 일부 유실이 있었지만, 안정적으로 배포가 수행됨을 확인하였다. 
```
Transactions:                  97556 hits
Availability:                  99.96 %
Elapsed time:                 239.25 secs
Data transferred:              54.99 MB
Response time:                  0.57 secs
Transaction rate:             407.76 trans/sec
Throughput:                     0.23 MB/sec
Concurrency:                  233.72
Successful transactions:           0
Failed transactions:              36
Longest transaction:           31.06
Shortest transaction:           0.00
```
### Configmap과 Secret 반영
모두 마이크로서비스에는 Configmap를 구성하여 외부 주입에 의한 Pod운영의 유연성을 확보하였다.
* DB주소와 Table스페이스 ConfigMap 예
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: cqrs-cm
data:
  db-address: 3.34.223.233:3306
  db-table-sapce: db_tanda_cqrs
```
* user id와 password Secrets 예
```
apiVersion: v1
kind: Secret
metadata:
  name: cqrs-secret
data:
  db-user: dGFuZGFjcXJzCg==
  db-pw: dGFuZGEyMDIwCg==
```
* 적용 yaml (예: CQRS 일부 발췌)
```
        env:
        - name: _DATASOURCE_ADDRESS
          valueFrom:
            configMapKeyRef:
               name: cqrs-cm
               key: db-address
        - name: _DATASOURCE_TABLESPACE
          valueFrom:
            configMapKeyRef:
               name: cqrs-cm
               key: db-table-sapce
        - name: _DATASOURCE_USERNAME
          valueFrom:
            secretKeyRef:
               name: cqrs-secret
               key: db-user
        - name: _DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
               name: cqrs-secret
               key: db-pw
```
* 적용 applicaion. (예: CQRS 일부 발췌)
```
  datasource:
    url: jdbc:mysql://${_DATASOURCE_ADDRESS:localhost:3306}/${_DATASOURCE_TABLESPACE:db_tanda_cqrs}?autoReconnect=true&serverTimezone=JST
    username: ${_DATASOURCE_USERNAME}
    password: ${_DATASOURCE_PASSWORD}
    driverClassName: com.mysql.cj.jdbc.Driver   
```
