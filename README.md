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
  https://github.com/tanda-msa
 
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
     - 결제(Sync) 외 비동기
   
 ###  KPI 
  1. 예약 서비스(book)
     - 타 시스템의 장애 여부와 상관없이 택시 요청에 대한 응답을 놓치지 않는다

 2. 차량 운행 서비스 (Taxideploy)
     - 모든 요청에 대해 1분이내 배차/재요청 응답한다.

 3. CORS
     - 예약 배차 관련 모든 상태 정보를 전달한다
     
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

### SagaPatter 적용(비기능요구사항-1,3 트랜젝션 )
   ![비기능요구사항1](https://user-images.githubusercontent.com/63759255/86529261-48217b80-beea-11ea-8e3b-bfdd3b31d7ef.png)

### Feign Client 구현(비기능요구사항-2,3 장애격리)  
#### 1. dependency 추가(pom.xml)
```xml
<dependency>
<groupId>org.springframework.cloud</groupId>
<artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
 ```    
#### 2. FeignClient Enabling(App.java)
```java
@EnableFeignClients
public class App {
```    
#### 3. FeignClient 인터페이스 생성(PayService.java) 
```java
@FeignClient(name = "pay", url = "${api.url.pay}")
public interface PayService {
@RequestMapping(method = RequestMethod.POST, path = "/pays", consumes = "application/json")
void billRelease(Pay pay);
}
```    
#### 4. @PreUpdate (결제완료처리 전) 결제모듈 실행(TaxiDispatch.java)   
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
### 적용후 Test(AWS)

* 예약서비스에서 예약요청
![택시요청](https://user-images.githubusercontent.com/63759255/86531256-a35b6a00-befa-11ea-9a91-0b4613d1cc4d.png)
![택시요청db1](https://user-images.githubusercontent.com/63759255/86531307-12d15980-befb-11ea-8eb5-2f801722fbf4.png)

* 택시 배차요청 수신 확인
![배차됨2](https://user-images.githubusercontent.com/63759255/86531267-b8d09400-befa-11ea-877d-25581c1232a2.png)
* 배차 완료 확인
![배차중2](https://user-images.githubusercontent.com/63759255/86531268-ba01c100-befa-11ea-82d8-8f432ff3dcab.png)
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

<<<<<<< HEAD
=======
{
    "_links": {
        "book": {
            "href": "http://localhost:8081/books/3"
        },
        "self": {
            "href": "http://localhost:8081/books/3"
        }
    },
    "arrivalLoc": "마포2",
    "bookStatus": "고객발 취소됨",
    "customerInfo": "홍길동/010-2233-0002/1111-2222-3333-0002",
    "departmentLoc": "김포",
    "lastModifyTime": "2020-07-02T19:52:48.307"
}
```
=======
>>>>>>> f8305ea0100b37d8f454422279242c7b1e5f3ce4
* 취소처리 확인
![취소처리1](https://user-images.githubusercontent.com/63759255/86531336-53c96e00-befb-11ea-8ea4-922cfbdec305.PNG)
* CQRS 확인
![ridehistories](https://user-images.githubusercontent.com/63759255/86531662-24683080-befe-11ea-950a-fc6e8451fca8.PNG)
* kakao 서비스 확인
![kakao](https://user-images.githubusercontent.com/63759255/86531660-23370380-befe-11ea-9069-788f0fc61d95.PNG)


>>>>>>> e03ddd56bef99d8e1dddc7155d60eefb81761f8a
* 각 서비스에서 메세지 Pub/Sub 전송 확인

![kafka test2](https://user-images.githubusercontent.com/63759255/86353866-f0450380-bca2-11ea-9bbf-da248b5f5fc0.png)

### 동기식 호출 / 장애격리 / Eventual Consistency

추

작
성

## 운영

### CI/CD 설정

**Github , AWS CodePipeline , AWS EKS 연계 구성을 아래와 같은 단계로 수행**

* CREATE IAM ROLE
* MODIFY AWS-AUTH CONFIGMAP
* FORK SAMPLE REPOSITORY
* GITHUB ACCESS TOKEN
* CODEPIPELINE SETUP
* TRIGGER NEW RELEASE
* CLEANUP


### 서킷 브레이킹 / 장애격리

내


용

### 오토스케일 아웃

내

용

### 무정지 재배포

내

용



 
