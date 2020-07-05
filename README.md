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
   ![비기능요구사항1](https://user-images.githubusercontent.com/63759255/86529261-48217b80-beea-11ea-8e3b-bfdd3b31d7ef.png)
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

### 헥사고날 다이어그램
![헥사고날 다이어그램](https://user-images.githubusercontent.com/63759255/86528326-e5c47d00-bee1-11ea-95cc-2e56b8df17c3.png)

  
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


### 적용후 Test(Sample)

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
* 택시 배차요청 수신 확인
```
http://a41dcd284d4994f898ece7716a77ab39-1598962157.ap-northeast-1.elb.amazonaws.com/taxi
{
  "_embedded" : {
    "taxiDispatches" : [ {
      "bookId" : 1,
      "taxiInfo" : null,
      "dispatchStatus" : "배차중",
      "price" : 0,
      "lastModifyTime" : "2020-07-05T17:14:40.831",
      "_links" : {
        "self" : {
          "href" : "http://a41dcd284d4994f898ece7716a77ab39-1598962157.ap-northeast-1.elb.amazonaws.com/taxi/1"
        },
        "taxiDispatch" : {
          "href" : "a41dcd284d4994f898ece7716a77ab39-1598962157.ap-northeast-1.elb.amazonaws.com/taxi/1"
        }
      }
    } ]
  },
  "_links" : {
    "self" : {
      "href" : "http://a41dcd284d4994f898ece7716a77ab39-1598962157.ap-northeast-1.elb.amazonaws.com/taxi"
    },
    "profile" : {
      "href" : "http://a41dcd284d4994f898ece7716a77ab39-1598962157.ap-northeast-1.elb.amazonaws.com/taxi"
    },
    "search" : {
      "href" : "http://a41dcd284d4994f898ece7716a77ab39-1598962157.ap-northeast-1.elb.amazonaws.com/taxi"
    }
  },
  "page" : {
    "size" : 20,
    "totalElements" : 1,
    "totalPages" : 1,
    "number" : 0
  }
}
```
* 배차 완료
```
http PATCH http://a41dcd284d4994f898ece7716a77ab39-1598962157.ap-northeast-1.elb.amazonaws.com/taxi/1 taxiInfo="30우8281/010-0000-0000" dispatchStatus="배차됨"

HTTP/1.1 200
Connection: keep-alive
Content-Type: application/json
Date: Sun, 05 Jul 2020 08:26:07 GMT
Keep-Alive: timeout=60
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_links": {
        "self": {
            "href": "http://http://a41dcd284d4994f898ece7716a77ab39-1598962157.ap-northeast-1.elb.amazonaws.com/taxi/1"
        },
        "taxiDispatch": {
            "href": "http://a41dcd284d4994f898ece7716a77ab39-1598962157.ap-northeast-1.elb.amazonaws.com/taxi/1"
        }
    },
    "bookId": 1,
    "dispatchStatus": "배차됨",
    "lastModifyTime": "2020-07-05T17:26:07.538",
    "price": 0,
    "taxiInfo": "30우8281/010-0000-0000"
}
```
* 운행 시작
```
http PATCH http://a41dcd284d4994f898ece7716a77ab39-1598962157.ap-northeast-1.elb.amazonaws.com/taxi/1 dispatchStatus="운행시작됨"

HTTP/1.1 200
Connection: keep-alive
Content-Type: application/json
Date: Sun, 05 Jul 2020 08:30:40 GMT
Keep-Alive: timeout=60
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_links": {
        "self": {
            "href": "http://a41dcd284d4994f898ece7716a77ab39-1598962157.ap-northeast-1.elb.amazonaws.com/taxi/1"
        },
        "taxiDispatch": {
            "href": "http://a41dcd284d4994f898ece7716a77ab39-1598962157.ap-northeast-1.elb.amazonaws.com/taxi/1"
        }
    },
    "bookId": 1,
    "dispatchStatus": "운행시작됨",
    "lastModifyTime": "2020-07-05T17:30:40.027",
    "price": 0,
    "taxiInfo": "30우8281/010-0000-0000"
}
```
* 운행 종료
```
http PATCH http://a41dcd284d4994f898ece7716a77ab39-1598962157.ap-northeast-1.elb.amazonaws.com/taxi/1 dispatchStatus="운행종료됨"

HTTP/1.1 200
Connection: keep-alive
Content-Type: application/json
Date: Sun, 05 Jul 2020 08:33:53 GMT
Keep-Alive: timeout=60
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_links": {
        "self": {
            "href": "http://a41dcd284d4994f898ece7716a77ab39-1598962157.ap-northeast-1.elb.amazonaws.com/taxi/1"
        },
        "taxiDispatch": {
            "href": "http://a41dcd284d4994f898ece7716a77ab39-1598962157.ap-northeast-1.elb.amazonaws.com/taxi/1"
        }
    },
    "bookId": 1,
    "dispatchStatus": "운행종료됨",
    "lastModifyTime": "2020-07-05T17:33:53.939",
    "price": 5000,
    "taxiInfo": "30우8281/010-0000-0000"
}
```
* 결제확인
```
http http://a41dcd284d4994f898ece7716a77ab39-1598962157.ap-northeast-1.elb.amazonaws.com/pay
HTTP/1.1 200
Connection: keep-alive
Content-Type: application/hal+json
Date: Sun, 05 Jul 2020 08:45:16 GMT
Keep-Alive: timeout=60
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_embedded": {
        "pays": [
            {
                "_links": {
                    "pay": {
                        "href": "http://a41dcd284d4994f898ece7716a77ab39-1598962157.ap-northeast-1.elb.amazonaws.com/pay/2"
                    },
                    "self": {
                        "href": "http://a41dcd284d4994f898ece7716a77ab39-1598962157.ap-northeast-1.elb.amazonaws.com/pay/2"
                    }
                },
                "bookId": 1,
                "dispatchId": 1,
                "lastModifyTime": "2020-07-05T17:42:30.845",
                "price": 5000
            }
        ]
    },
    "_links": {
        "profile": {
            "href": "http://a41dcd284d4994f898ece7716a77ab39-1598962157.ap-northeast-1.elb.amazonaws.com/pay"
        },
        "self": {
            "href": "http://a41dcd284d4994f898ece7716a77ab39-1598962157.ap-northeast-1.elb.amazonaws.com/pay"
        }
    },
    "page": {
        "number": 0,
        "size": 20,
        "totalElements": 1,
        "totalPages": 1
    }
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



 
