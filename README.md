![image](https://user-images.githubusercontent.com/70673885/97950284-bcf1bd00-1dd9-11eb-8c8a-b3459c710849.png)

# 예제 - 휴대폰구매앱

본 예제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 예제입니다.
이는 클라우드 네이티브 애플리케이션의 개발에 요구되는 체크포인트들을 통과하기 위한 예시 답안을 포함합니다.
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [예제 - 휴대폰구매앱](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [폴리글랏 프로그래밍](#폴리글랏-프로그래밍)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)

# 서비스 시나리오

기능적 요구사항
1. 고객이 APP에서 폰을 주문한다.
1. 고객이 결제한다.
1. 주문이 되면 주문 내역이 대리점에 전달된다.
1. 대리점에 주문 정보가 도착하면 배송한다.
1. 배송이 되면 APP에서 배송상태를 조회할 수 있다.
1. 고객이 주문을 취소할 수 있다.
1. 주문이 취소되면 결제가 취소된다.
1. 고객이 결제상태를 APP에서  조회 할 수 있다.
1. 고객이 모든 진행내역의 모니터링을 제공해야 한다.

비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 주문건은 아예 거래가 성립되지 않아야 한다.> Sync 호출
    1. 주문이 취소되면 결제가 취소되고 주문정보에 업데이트가 되어야 한다.> SAGA, 보상 트랜젝션
1. 장애격리
    1. 대리점관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다.> Async (event-driven), Eventual Consistency
    1. 결제시스템이 과중되면 주문을 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다> Circuit breaker, fallback
1. 성능
    1. 고객이 모든 진행내역을 조회 할 수 있도록 성능을 고려하여 별도의 view로 구성한다.> CQRS


# 체크포인트

- 분석 설계


  - 이벤트스토밍: 
    - 스티커 색상별 객체의 의미를 제대로 이해하여 헥사고날 아키텍처와의 연계 설계에 적절히 반영하고 있는가?
    - 각 도메인 이벤트가 의미있는 수준으로 정의되었는가?
    - 어그리게잇: Command와 Event 들을 ACID 트랜잭션 단위의 Aggregate 로 제대로 묶었는가?
    - 기능적 요구사항과 비기능적 요구사항을 누락 없이 반영하였는가?    

  - 서브 도메인, 바운디드 컨텍스트 분리
    - 팀별 KPI 와 관심사, 상이한 배포주기 등에 따른  Sub-domain 이나 Bounded Context 를 적절히 분리하였고 그 분리 기준의 합리성이 충분히 설명되는가?
      - 적어도 3개 이상 서비스 분리
    - 폴리글랏 설계: 각 마이크로 서비스들의 구현 목표와 기능 특성에 따른 각자의 기술 Stack 과 저장소 구조를 다양하게 채택하여 설계하였는가?
    - 서비스 시나리오 중 ACID 트랜잭션이 크리티컬한 Use 케이스에 대하여 무리하게 서비스가 과다하게 조밀히 분리되지 않았는가?
  - 컨텍스트 매핑 / 이벤트 드리븐 아키텍처 
    - 업무 중요성과  도메인간 서열을 구분할 수 있는가? (Core, Supporting, General Domain)
    - Request-Response 방식과 이벤트 드리븐 방식을 구분하여 설계할 수 있는가?
    - 장애격리: 서포팅 서비스를 제거 하여도 기존 서비스에 영향이 없도록 설계하였는가?
    - 신규 서비스를 추가 하였을때 기존 서비스의 데이터베이스에 영향이 없도록 설계(열려있는 아키택처)할 수 있는가?
    - 이벤트와 폴리시를 연결하기 위한 Correlation-key 연결을 제대로 설계하였는가?

  - 헥사고날 아키텍처
    - 설계 결과에 따른 헥사고날 아키텍처 다이어그램을 제대로 그렸는가?
    
- 구현
  - [DDD] 분석단계에서의 스티커별 색상과 헥사고날 아키텍처에 따라 구현체가 매핑되게 개발되었는가?
    - Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터 접근 어댑터를 개발하였는가
    - [헥사고날 아키텍처] REST Inbound adaptor 이외에 gRPC 등의 Inbound Adaptor 를 추가함에 있어서 도메인 모델의 손상을 주지 않고 새로운 프로토콜에 기존 구현체를 적응시킬 수 있는가?
    - 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가?
  - Request-Response 방식의 서비스 중심 아키텍처 구현
    - 마이크로 서비스간 Request-Response 호출에 있어 대상 서비스를 어떠한 방식으로 찾아서 호출 하였는가? (Service Discovery, REST, FeignClient)
    - 서킷브레이커를 통하여  장애를 격리시킬 수 있는가?
  - 이벤트 드리븐 아키텍처의 구현
    - 카프카를 이용하여 PubSub 으로 하나 이상의 서비스가 연동되었는가?
    - Correlation-key:  각 이벤트 건 (메시지)가 어떠한 폴리시를 처리할때 어떤 건에 연결된 처리건인지를 구별하기 위한 Correlation-key 연결을 제대로 구현 하였는가?
    - Message Consumer 마이크로서비스가 장애상황에서 수신받지 못했던 기존 이벤트들을 다시 수신받아 처리하는가?
    - Scaling-out: Message Consumer 마이크로서비스의 Replica 를 추가했을때 중복없이 이벤트를 수신할 수 있는가
    - CQRS: Materialized View 를 구현하여, 타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이) 도 내 서비스의 화면 구성과 잦은 조회가 가능한가?

  - 폴리글랏 플로그래밍
    - 각 마이크로 서비스들이 하나이상의 각자의 기술 Stack 으로 구성되었는가?
    - 각 마이크로 서비스들이 각자의 저장소 구조를 자율적으로 채택하고 각자의 저장소 유형 (RDB, NoSQL, File System 등)을 선택하여 구현하였는가?
  - API 게이트웨이
    - API GW를 통하여 마이크로 서비스들의 집입점을 통일할 수 있는가?
    - 게이트웨이와 인증서버(OAuth), JWT 토큰 인증을 통하여 마이크로서비스들을 보호할 수 있는가?
- 운영
  - SLA 준수
    - 셀프힐링: Liveness Probe 를 통하여 어떠한 서비스의 health 상태가 지속적으로 저하됨에 따라 어떠한 임계치에서 pod 가 재생되는 것을 증명할 수 있는가?
    - 서킷브레이커, 레이트리밋 등을 통한 장애격리와 성능효율을 높힐 수 있는가?
    - 오토스케일러 (HPA) 를 설정하여 확장적 운영이 가능한가?
    - 모니터링, 앨럿팅: 
  - 무정지 운영 CI/CD (10)
    - Readiness Probe 의 설정과 Rolling update을 통하여 신규 버전이 완전히 서비스를 받을 수 있는 상태일때 신규버전의 서비스로 전환됨을 siege 등으로 증명 
    - Contract Test :  자동화된 경계 테스트를 통하여 구현 오류나 API 계약위반를 미리 차단 가능한가?


# 분석/설계


## AS-IS 조직 (Horizontally-Aligned)
  ![image](https://user-images.githubusercontent.com/487999/79684144-2a893200-826a-11ea-9a01-79927d3a0107.png)

## TO-BE 조직 (Vertically-Aligned)
  ![image](https://user-images.githubusercontent.com/487999/79684159-3543c700-826a-11ea-8d5f-a3fc0c4cad87.png)


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://www.msaez.io/#/storming/G4Le38IyNmPdGV7UTxmqbVhBw8z1/share/fef3e793823083653eb1b4ef257a6bb3/-MLAUDjJIxzggM4AGxrE


### 이벤트 도출
![image](https://user-images.githubusercontent.com/70673885/97949704-dc87e600-1dd7-11eb-9525-544b2411cc51.png)
### 부적격 이벤트 탈락
![image](https://user-images.githubusercontent.com/70673885/97949767-0a6d2a80-1dd8-11eb-8c2f-fa445fa61418.png)

    - 과정중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
    - 폰종류가선택됨, 결제버튼클릭됨, 배송수량선택됨, 배송일자선택됨  :  UI 의 이벤트이지, 업무적인 의미의 이벤트가 아니라서 제외
	- 배송취소됨, 메시지발송됨  :  계획된 사업 범위 및 프로젝트에서 벗어서난다고 판단하여 제외
	- 주문정보전달됨  :  주문됨을 선택하여 제외

### 액터, 커맨드 부착하여 읽기 좋게
![image](https://user-images.githubusercontent.com/73699193/97982030-82f2dc00-1e16-11eb-821d-27351387f8ad.png)

### 어그리게잇으로 묶기
![image](https://user-images.githubusercontent.com/73699193/97982108-a158d780-1e16-11eb-9270-6e9646268fd1.png)

    - 주문, 대리점관리, 결제 어그리게잇을 생성하고 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 그들 끼리 묶어줌

### 바운디드 컨텍스트로 묶기

![image](https://user-images.githubusercontent.com/73699193/97982213-c77e7780-1e16-11eb-87ef-03dbe66a6cf2.png)

    - 도메인 서열 분리 
        - Core Domain:  app(front), store : 없어서는 안될 핵심 서비스이며, 연견 Up-time SLA 수준을 99.999% 목표, 배포주기는 app 의 경우 1주일 1회 미만, store 의 경우 1개월 1회 미만
        - Supporting Domain:  customer(고객센터view) : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.
        - General Domain:  pay : 결제서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음 

### 폴리시 부착 (괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남)

![image](https://user-images.githubusercontent.com/73699193/97982278-e3821900-1e16-11eb-97f4-fa2f59fc7ae0.png)

### 폴리시의 이동

![image](https://user-images.githubusercontent.com/73699193/97982413-19bf9880-1e17-11eb-9720-cd82cf1060ff.png)

### 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

![image](https://user-images.githubusercontent.com/73699193/97982527-45428300-1e17-11eb-8641-b658bab34fc6.png)

    - 컨텍스트 매핑하여 묶어줌.
    - 팀원 중 외국인이 투입되어 유비쿼터스 랭귀지인 영어로 변경	

### 완성된 모형

![image](https://user-images.githubusercontent.com/73699193/97982584-60ad8e00-1e17-11eb-8fb6-af87b7c6ff91.png)

    - View Model 추가

### 기능적 요구사항 검증

![image](https://user-images.githubusercontent.com/73699193/97982759-96527700-1e17-11eb-9144-f95de1e0d01e.png)

    - 고객이 APP에서 폰을 주문한다. (ok)
    - 고객이 결제한다. (ok)
    - 결제가 되면 주문 내역이 대리점에 전달된다. (ok)
    - 대리점에 주문 정보가 도착하면 배송한다. (ok)
	- 배송이 되면 APP에서 배송상태를 조회할 수 있다. (ok)

![image](https://user-images.githubusercontent.com/73699193/97982841-b2eeaf00-1e17-11eb-9f09-9b74f85a96ca.png)

	- 고객이 주문을 취소할 수 있다. (ok)
	- 주문이 취소되면 결제가 취소된다. (ok)
	- 고객이 결제상태를 APP에서  조회 할 수 있다. (ok)

![image](https://user-images.githubusercontent.com/73699193/97982928-d3b70480-1e17-11eb-957e-6a9093d2a0d7.png)
  
	- 고객이 모든 진행내역의 모니터링을 제공해야 한다. (ok)


### 비기능 요구사항 검증

![image](https://user-images.githubusercontent.com/73699193/97983019-f6e1b400-1e17-11eb-86ef-d43873ccbb7d.png)

    - 1) 결제가 되지 않은 주문건은 아예 거래가 성립되지 않아야 한다. (Req/Res)
    - 2) 대리점관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다. (Pub/sub)
    - 3) 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다. (Circuit breaker)
    - 4) 주문이 취소되면 결제가 취소되고 주문정보에 업데이트가 되어야 한다.  (SAGA, 보상트렌젝션)
    - 5) 고객이 모든 진행내역을 조회 할 수 있도록 성능을 고려하여 별도의 view로 구성한다. (CQRS, DML/SELECT 분리)


## 헥사고날 아키텍처 다이어그램 도출 (Polyglot)

![image](https://user-images.githubusercontent.com/70673885/97952091-99ca0c00-1ddf-11eb-809b-6e73618c0b17.png)

    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐
    - 대리점의 경우 Polyglot 검증을 위해 Hsql로 셜계


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트와 파이선으로 구현하였다. 
gateway를 포함한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)


cd app
mvn spring-boot:run

cd pay
mvn spring-boot:run 

cd store
mvn spring-boot:run  

cd customer
mvn spring-boot:run  


## DDD 의 적용
도메인 드리븐 디자인으로 업무영역(도메인)을 주요업무 기준으로 4개의 MSA를 도출하였다.
분석/설계 단계에서 도출된 MSA는 총 4개로 아래와 같다.
* customer 는 고객샌터에서 사용하는 모니터링뷰의 이름이다.(CQRS)

| MSA | 기능 | port | 조회 API | Gateway 사용시 |
|---|:---:|:---:|---|---|
| app | 주문 관리 | 8081 | http://localhost:8081/orders | http://app:8080/orders |
| pay | 결제 관리 | 8083 | http://localhost:8082/payments | http://pay:8080/payments |
| store | 대리점 관리 | 8085 | http://localhost:8083/storeManages | http://store:8080/storeManages |
| customer | 모니터링 | 8084 | http://localhost:8084/customers | http://customer:8080/customers |



## 폴리글랏 퍼시스턴스
대리점의 경우 H2 DB인 주문과 결제와 달리 Hsql으로 구현하여 MSA간 서로 다른 종류의 DB간에도 문제 없이 동작하여 다형성을 만족하는지 확인하였다. 


app, pay, customer의 pom.xml 설정

![image](https://user-images.githubusercontent.com/73699193/97972993-baf32280-1e08-11eb-8158-912e4d28d7ea.png)


store의 pom.xml 설정

![image](https://user-images.githubusercontent.com/73699193/97973735-e0346080-1e09-11eb-9636-605e2e870fb0.png)



## Gateway 적용

gateway > applitcation.yml 설정

![image](https://user-images.githubusercontent.com/73699193/98060621-5d54e980-1e8d-11eb-943c-692c5953c6a1.png)

생성한 gateway 서비스 run
```
cd gateway
mvn spring-boot:run  
```

httpie 실행
```
kubectl exec -it httpie bin/bash -n phone82
```
app 확인
```
http GET http://app:8080/orders/1
```
![image](https://user-images.githubusercontent.com/73699193/98065273-a01bbf00-1e97-11eb-9942-f036600ddc89.png)

pay 확인
```
http GET http://pay:8080/payments/1
```
![image](https://user-images.githubusercontent.com/73699193/98065346-c80b2280-1e97-11eb-9d79-50333308ba0c.png)

store 확인
```
http GET http://store:8080/storeManages/1
```
![image](https://user-images.githubusercontent.com/73699193/98065418-ebce6880-1e97-11eb-8bd4-3916202239ee.png)

customer 확인
```
http GET http://customer:8080/customers/1
```
![image](https://user-images.githubusercontent.com/73699193/98065232-867a7780-1e97-11eb-90ba-fe882c68fb71.png)



## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 주문(app)->결제(pay) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 
호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 결제서비스를 호출하기 위하여 FeignClient 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 
```
# (app) external > PaymentService.java

package phoneseller.external;

@FeignClient(name="pay", url="${api.pay.url}")
public interface PaymentService {

    @RequestMapping(method= RequestMethod.POST, path="/payments")
    public void pay(@RequestBody Payment payment);

}
```
![image](https://user-images.githubusercontent.com/73699193/98065833-b1190000-1e98-11eb-9e44-84d4961011ed.png)


- 주문을 받은 직후(@PostPersist) 결제를 요청하도록 처리
```
# (app) Order.java (Entity)

    @PostPersist
    public void onPostPersist(){

       phoneseller.external.Payment payment = new phoneseller.external.Payment();
        payment.setOrderId(this.getId());
        payment.setProcess("Ordered");
        
        AppApplication.applicationContext.getBean(phoneseller.external.PaymentService.class)
            .pay(payment);
    }
```
![image](https://user-images.githubusercontent.com/73699193/98066539-a6f80100-1e9a-11eb-8dd8-bf213d90e5fb.png)

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 주문도 못받는다는 것을 확인:

```
#결제(pay) 서비스를 잠시 내려놓음 (ctrl+c)

#주문하기(order)
http http://localhost:8081/orders item=note20 qty=1   #Fail
```
![image](https://user-images.githubusercontent.com/73699193/98072284-04934a00-1ea9-11eb-9fad-40d3996e109f.png)

```
#결제(pay) 서비스 재기동
cd pay
mvn spring-boot:run

#주문하기(order)
http http://localhost:8081/orders item=note21 qty=2   #Success
```
![image](https://user-images.githubusercontent.com/73699193/98074359-9f8e2300-1ead-11eb-8854-0449a65ff55c.png)

- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커는 운영단계에서 설명한다.)




## 비동기식 호출 / 시간적 디커플링 / 장애격리 


결제(pay)가 이루어진 후에 대리점(store)으로 이를 알려주는 행위는 비 동기식으로 처리하여 대리점(store)의 처리를 위하여 결제주문이 블로킹 되지 않아도록 처리한다.
 
- 결제승인이 되었다(payCompleted)는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
# (pay) Payment.java (Entity)

package phoneseller;

@Entity
@Table(name="Payment_table")
public class Payment {

 ...

        PayCompleted payCompleted = new PayCompleted();
        BeanUtils.copyProperties(this, payCompleted);
        System.out.println(payCompleted.toJson());
        payCompleted.publishAfterCommit();  
		
 ...
 
}
```
![image](https://user-images.githubusercontent.com/73699193/98075277-6f478400-1eaf-11eb-88c8-2b4a7736e56b.png)

- 대리점(store)에서는 결제승인(payCompleted) 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다.
- 주문접수(OrderReceive)는 송출된 결제승인(payCompleted) 정보를 store의 Repository에 저장한다.:
 
```
# (store) PolicyHandler.java

@Service
public class PolicyHandler{

    @Autowired
    StoreManageRepository storeManageRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPayCompleted_OrderReceive(@Payload PayCompleted payCompleted){

        if(payCompleted.isMe()){
            System.out.println("##### listener OrderReceive : " + payCompleted.toJson());
            System.out.println("store_policy_paycompleted_orderreceive");

            StoreManage storeManage = new StoreManage();
            storeManage.setOrderId(payCompleted.getOrderId());
            storeManage.setProcess(payCompleted.getProcess());
            storeManageRepository.save(storeManage);
        }
    }

}

```
![image](https://user-images.githubusercontent.com/73699193/98076059-e0d40200-1eb0-11eb-94ad-c4ea114cb3aa.png)


대리점(store)시스템은 주문(app)/결제(pay)와 완전히 분리되어있으며(sync transaction 없음), 이벤트 수신에 따라 처리되기 때문에, 대리점(store)이 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다.(시간적 디커플링):
```
# 대리점(store) 서비스를 잠시 내려놓음 (ctrl+c)

#주문하기(order)
http http://localhost:8081/orders item=note30 qty=2  #Success

#주문상태 확인
http get http://localhost:8081/orders    # 상태값이 'Shipped'이 아닌 'Payed'에서 멈춤을 확인
```
![image](https://user-images.githubusercontent.com/73699193/98078301-2b577d80-1eb5-11eb-9d89-7c03a3fa27dd.png)
```
#대리점(store) 서비스 기동
cd store
mvn spring-boot:run

#주문상태 확인
http get http://localhost:8081/orders     # 'Payed' 였던 상태값이 'Shipped'로 변경된 것을 확인
```
![image](https://user-images.githubusercontent.com/73699193/98078837-2cd57580-1eb6-11eb-8850-a8c621410d61.png)

# 운영

## Deploy / Pipeline
- 네임스페이스 만들기
```
kubectl create ns phone82
kubectl get ns
```
![image](https://user-images.githubusercontent.com/73699193/97960790-6d20ef00-1df5-11eb-998d-d5591975b5d4.png)

- 폴더 만들기, 해당폴더로 이동
```
mkdir phone82
cd phone 82
```
![image](https://user-images.githubusercontent.com/73699193/97961127-0ea84080-1df6-11eb-81b3-1d5e460d4c0f.png)

- 소스 가져오기
```
git clone https://github.com/phone82/app.git
```
![image](https://user-images.githubusercontent.com/73699193/98089346-eb4cc680-1ec5-11eb-9c23-f6987dee9308.png)

- 빌드하기
```
cd app
mvn package -Dmaven.test.skip=true
```
![image](https://user-images.githubusercontent.com/73699193/98089442-19320b00-1ec6-11eb-88b5-544cd123d62a.png)

- 도커라이징: Azure 레지스트리에 도커 이미지 푸시하기
```
az acr build --registry admin02 --image admin02.azurecr.io/app:latest .
```
![image](https://user-images.githubusercontent.com/73699193/98089685-6dd58600-1ec6-11eb-8fb9-80705c854c7b.png)

- 컨테이너라이징: 디플로이 생성 확인
```
kubectl create deploy app --image=admin02.azurecr.io/app:latest -n phone82
kubectl get all -n phone82
```
![image](https://user-images.githubusercontent.com/73699193/98090560-83977b00-1ec7-11eb-9770-9cfe1021f0b4.png)

- 컨테이너라이징: 서비스 생성 확인
```
kubectl expose deploy app --type="ClusterIP" --port=8080 -n phone82
kubectl get all -n phone82
```
![image](https://user-images.githubusercontent.com/73699193/98090693-b80b3700-1ec7-11eb-959e-fc0ce94663aa.png)

- pay, store, customer, gateway에도 동일한 작업 반복




-(별첨)deployment.yaml을 사용하여 배포 

- deployment.yaml 편집
```
namespace, image 설정
env 설정 (config Map) 
readness 설정 (무정지 배포)
liveness 설정 (self-healing)
resource 설정 (autoscaling)
```
![image](https://user-images.githubusercontent.com/73699193/98092861-8182eb80-1eca-11eb-87c5-afa22140ebad.png)

- deployment.yaml로 서비스 배포
```
cd app
kubectl apply -f kubernetes/deployment.yml
```

## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 단말앱(app)-->결제(pay) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml
feign:
  hystrix:
    enabled: true
    
hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

```
![image](https://user-images.githubusercontent.com/73699193/98093705-a166df00-1ecb-11eb-83b5-f42e554f7ffd.png)

* siege 툴 사용법:
```
 siege가 생성되어 있지 않으면:
 kubectl run siege --image=apexacme/siege-nginx -n phone82
 siege 들어가기:
 kubectl exec -it pod/siege-5c7c46b788-4rn4r -c siege -n phone82 -- /bin/bash
 siege 종료:
 Ctrl + C -> exit
```
* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 100명
- 60초 동안 실시

```
siege -c100 -t60S -r10 -v --content-type "application/json" 'http://app:8080/orders POST {"item": "abc123", "qty":3}'
```
- 부하 발생하여 CB가 발동하여 요청 실패처리됨

![image](https://user-images.githubusercontent.com/73699193/98098327-87c89600-1ed1-11eb-8b3f-c6594b94841c.png)

- 밀린 부하가 pay에서 처리되면서 다시 order를 받기 시작 

![image](https://user-images.githubusercontent.com/73699193/98098702-07eefb80-1ed2-11eb-94bf-316df4bf682b.png)

- report

![image](https://user-images.githubusercontent.com/73699193/98099047-6e741980-1ed2-11eb-9c55-6fe603e52f8b.png)

- CB 잘 적용됨을 확인


### 오토스케일 아웃

- 대리점 시스템에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:
```
# autocale out 설정
kubectl autoscale deploy store --min=1 --max=10 --cpu-percent=15 -n phone82
```
![image](https://user-images.githubusercontent.com/73699193/98100149-ce1ef480-1ed3-11eb-908e-a75b669d611d.png)


-
- CB 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.
```
kubectl exec -it pod/siege-5c7c46b788-4rn4r -c siege -n phone82 -- /bin/bash
siege -c100 -t120S -r10 -v --content-type "application/json" 'http://store:8080/storeManages POST {"orderId":"456", "process":"Payed"}'
```
![image](https://user-images.githubusercontent.com/73699193/98102543-0d9b1000-1ed7-11eb-9cb6-91d7996fc1fd.png)

- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy store -w -n phone82
```
- 어느정도 시간이 흐른 후 스케일 아웃이 벌어지는 것을 확인할 수 있다. max=10 
- 부하를 줄이니 늘어난 스케일이 점점 줄어들었다.

![image](https://user-images.githubusercontent.com/73699193/98102926-92862980-1ed7-11eb-8f19-a673d72da580.png)

- 다시 부하를 주고 확인하니 Availability가 높아진 것을 확인 할 수 있었다.

![image](https://user-images.githubusercontent.com/73699193/98103249-14765280-1ed8-11eb-8c7c-9ea1c67e03cf.png)


## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함


- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
kubectl apply -f kubernetes/deployment_readiness.yml
```
- readness 옵션이 없는 경우 배포 중 서비스 요청처리 실패

![image](https://user-images.githubusercontent.com/73699193/98105334-2a394700-1edb-11eb-9633-f5c33c5dee9f.png)


- deployment.yml에 readness 옵션을 추가 

![image](https://user-images.githubusercontent.com/73699193/98107176-75ecf000-1edd-11eb-88df-617c870b49fb.png)

- readness적용된 deployment.yml 적용

```
kubectl apply -f kubernetes/deployment.yml
```
- 새로운 버전의 이미지로 교체
```
cd acr
az acr build --registry admin02 --image admin02.azurecr.io/store:v4 .
kubectl set image deploy store store=admin02.azurecr.io/store:v4 -n phone82
```
- 기존 버전과 새 버전의 store pod 공존 중
![image](https://user-images.githubusercontent.com/73699193/98106161-65884580-1edc-11eb-9540-17a3c9bdebf3.png)

- Availability: 100.00 % 확인
![image](https://user-images.githubusercontent.com/73699193/98106524-c152ce80-1edc-11eb-8e0f-3731ca2f709d.png)



## Config Map

- apllication.yml 설정

* default쪽

![image](https://user-images.githubusercontent.com/73699193/98108335-1c85c080-1edf-11eb-9d0f-1f69e592bb1d.png)

* docker 쪽

![image](https://user-images.githubusercontent.com/73699193/98108645-ad5c9c00-1edf-11eb-8d54-487d2262e8af.png)

- Deployment.yml 설정

![image](https://user-images.githubusercontent.com/73699193/98108902-12b08d00-1ee0-11eb-8f8a-3a3ea82a635c.png)

- config map 생성 후 조회
```
kubectl create configmap apiurl --from-literal=url=http://pay:8080 --from-literal=fluentd-server-ip=10.xxx.xxx.xxx -n phone82
```
![image](https://user-images.githubusercontent.com/73699193/98107784-5bffdd00-1ede-11eb-8da6-82dbead0d64f.png)

- 설정한 url로 주문 호출
```
http POST http://app:8080/orders item=dfdf1 qty=21
```

![image](https://user-images.githubusercontent.com/73699193/98109319-b732cf00-1ee0-11eb-9e92-ad0e26e398ec.png)

- configmap 삭제 후 app 서비스 재시작
```
kubectl delete configmap apiurl -n phone82
kubectl get pod/app-56f677d458-5gqf2 -n phone82 -o yaml | kubectl replace --force -f-
```
![image](https://user-images.githubusercontent.com/73699193/98110005-cf571e00-1ee1-11eb-973f-2f4922f8833c.png)

- configmap 삭제된 상태에서 주문 호출   
```
http POST http://app:8080/orders item=dfdf2 qty=22
```
![image](https://user-images.githubusercontent.com/73699193/98110323-42f92b00-1ee2-11eb-90f3-fe8044085e9d.png)

![image](https://user-images.githubusercontent.com/73699193/98110445-720f9c80-1ee2-11eb-851e-adcd1f2f7851.png)

![image](https://user-images.githubusercontent.com/73699193/98110782-f4985c00-1ee2-11eb-97a7-1fed3c6b042c.png)
(pod 정보)


  ![image](https://user-images.githubusercontent.com/487999/79684133-1d6c4300-826a-11ea-94a2-602e61814ebf.png)


## Self-healing (Liveness Probe)

- store 서비스 정상 확인

![image](https://user-images.githubusercontent.com/27958588/98096336-fb1cd880-1ece-11eb-9b99-3d704cd55fd2.jpg)


- deployment.ymㅣ에 Liveness Probe 옵션 추가
```
cd ~/phone82/store/kubernetes
vi deployment.yml

(아래 설정 변경)
livenessProbe:
	tcpSocket:
	  port: 8081
	initialDelaySeconds: 5
	periodSeconds: 5
```
![image](https://user-images.githubusercontent.com/27958588/98096375-0839c780-1ecf-11eb-85fb-00e8252aa84a.jpg)

- 적용 확인 1

![image](https://user-images.githubusercontent.com/27958588/98096393-0a9c2180-1ecf-11eb-8ac5-f6048160961d.jpg)

- 적용 확인 2

![image](https://user-images.githubusercontent.com/27958588/98096461-20a9e200-1ecf-11eb-8b02-364162baa355.jpg)

