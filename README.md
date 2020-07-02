Tanda(택시예약 시스템)
======================
본 시스템은 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 시스템입니다.
  
## Table of contents

 * 택시예약관리 시스템
  * 소스코드
  * 서비스 시나리오
  * 분석/설계
  * 구현:
    * DDD 의 적용
    * 폴리글랏 퍼시스턴스
    * 폴리글랏 프로그래밍
    * 동기식 호출 과 Fallback 처리
    * 비동기식 호출 과 Eventual Consistency
  * 운영
    * CI/CD 설정
    * 동기식 호출 / 서킷 브레이킹 / 장애격리
    * 오토스케일 아웃
    * 무정지 재배포
    
 ## 택시예약관리 시스템
  ![탄다](https://user-images.githubusercontent.com/63759255/86345665-cc7bc080-bc96-11ea-9b94-c3ae43cef05a.jpg)
 
 
 ## 소스코드
  https://github.com/tanda-msa
 
 ## 서비스 시나리오
 
 #### 기능적 요구사항
 
   - 고객이 택시를 요청한다
   - 기사가 배차를 승낙한다
   - 택시가 배차 된다
   - 배차가 되면 고객에게 도착 시간을 알린다
   - 가용한 차량이 없으면 고객의 요청이 취소된다
   - 차량에 탑승하면 운행중으로 상태가 바뀐다
   - 차량에서 하차하면 운행종료로 상태가 바뀐다
   - 하차시 요금이 자동 결제 된다
   - 고객은 배차 요청을 탑승전까지 취소할 수 있으며 택시가 배차상태라도 배차가 취소된다
 
#### 비기능 요구사항
 
  1. 트랜잭션
   고객 배차 취소 요청은 비동기 방식이나, Saga Pattern을 적용하여 최종 일관성을 유지한다.  
  2. 장애격리
     - 결제시스템(Sync연동)이 과중되면 Circuit breaker(FeignClient, Hystrix) 동작 및 운행종료(fallback)로 넘어가지 않는다.
     - 요금 결제가 되지 않으면 운행종료로 넘어가지 않는다 (sync호출)
  3. 성능 
     - 서포트 기능(CPQR) 이 수행되지 않더라도 차량 요청 및 배차는 365일 24시간 받을 수 있어야 한다      Async (event-driven), Eventual Consistency
     - 가용한 차량이 없으면 사용자를 잠시동안 받지 않고 잠시후에 호출하도록 유도 (circuit breaker)
     - 차량운행 상태의 변경시 알림을 준다 (Event driven)
        1. 배차(비가용)
        2. 운행시작(비가용)
        3. 운행종료(가용)
  4. 기타
     - 결제(Sync) 외 비동기
   
 ####  KPI 
  1. 예약 서비스(book)
     - 타 시스템의 장애 여부와 상관없이 택시 요청에 대한 응답을 놓치지 않는다

 2. 차량 운행 서비스 (Taxideploy)
     - 모든 요청에 대해 1분이내 배차/재요청 응답한다.

 3. CORS
     - 예약 배차 관련 모든 상태 정보를 전달한다
     
## 분석/설계
#### Event Storming 결과

* MSAEz 로 모델링한 이벤트스토밍 결과

_url 삽입 필요_

#### 이벤트 도출

_사진 삽입_

#### 액터, 커맨드, 폴리시 부착

_사진 삽입_

#### 어그리게잇으로 묶기
```
  . Core Domain : 예약 , 차량관리
  . Supporting Domain : CQRS
  . General Domain : 결제  
```
  
## 구현(github 소스 참고)
분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 
스프링부트와 H2 DB(예약 서비스),MySQL(배차,CQRS 서비스) 및 Kafka 클러스트(공통)를 통해 AWS Cloud 상에 구현하였다.
구현한 각 서비스에서 사용된 포트 정보는 아래와 같다. (각자의 포트넘버는 8081 ~ 808n 이다)
로컬 PC 에서 구현한 소스를 빌드업하고 Dockerising 하여 AWS 클라우드의 Kubernetes 클러스트에 배포
```
# Book //port number: 8081

# TaxiDeploy //port number: 8082

# Pay //port number: 8083

# CQRS //port number: 8084
```

각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity로 선언하였다


## 적용후 Test(Sample)

* 예약서비스에서 예약요청
```
http http://localhost:8081/books customerInfo="손흥민/010-1122-0015/1111-2222-3333-0015" departmentLoc="여의도" arrivalLoc="상도동" 
Connection: keep-alive
Content-Type: application/json
Date: Thu, 02 Jul 2020 10:59:25 GMT
Keep-Alive: timeout=60
Location: http://localhost:8081/books/6
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_links": {
        "book": {
            "href": "http://localhost:8081/books/6"
        },
        "self": {
            "href": "http://localhost:8081/books/6"
        }
    },
    "arrivalLoc": "상도동",
    "bookStatus": "접수됨",
    "customerInfo": "손흥민/010-1122-0005/1111-2222-3333-0005",
    "departmentLoc": "여의도",
    "lastModifyTime": "2020-07-02T19:59:25.509"
}
```

* 예약서비스에서 고객발 취소 요청
```
http patch localhost:8081/books/3 bookStatus="고객발 취소됨"
HTTP/1.1 200
Connection: keep-alive
Content-Type: application/json
Date: Thu, 02 Jul 2020 10:52:48 GMT
Keep-Alive: timeout=60
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

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







 
