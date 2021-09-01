# 전시회 예약 관리 시스템

## Table of contents
  - [서비스 시나리오](#서비스-시나리오)
  - [분석/설계](#분석/설계)
    - [Event Storming결과](#Event-Storming-결과)
  - [구현](#구현)
    - [DDD의 적용](#DDD의-적용)
    - [Saga](#Saga)
    - [비동기식 호출과 Eventual Consistency](#비동기식-호출과-Eventual-Consistency)
    - [CQRS](#CQRS)
    - [Correlation](#Correlation)
    - [동기식 호출](#동기식-호출)
    - [Gateway](#Gateway)
    - [서킷 브레이킹](#서킷-브레이킹)
    - [Polyglot Persistent/Programming](#Polyglot-Persistent/Programming)
  - [운영](#운영)
    - [Deploy/Pipeline 설정](#Deploy/Pipeline-설정)
    - [AutoScale Out](#AutoScale-Out)
    - [무정지 재배포](#무정지-재배포)
    - [개발/운영 환경 분리 (ConfigMap)](#개발/운영-환경-분리-(ConfigMap))
    - [Self-Healing](#Self-Healing)

## 서비스 시나리오
    기능적 요구사항
        1. 관람객은 방문하려는 날짜와 회차(시간) 선택, 연락처를 입력하고 예약한다. 
        2. 예약 등록시 관람료를 결제해야 한다. 
        3. 관람료가 결제되지 않으면 최종 관람 예약이 성립되지 않는다. 
        4. 예약이 완료되면 수용 가능한 관람객의 수를 감소시킨다.
        5. 예약을 취소할 수 있고, 예약이 취소되면 결제를 취소한다.
        6. 결제가 취소되면 수용 가능한 관람객의 수를 증가시킨다. 
        7. 관람객은 본인의 예약 내역을 조회할 수 있다. 

    비기능적 요구사항
        1. 트랜잭션 
         - 관람료가 결제되어야 최종 관람 예약이 성립된다. -> Sync 호출
        2. 장애격리
         - 전시회 관리 기능이 수행되지 않더라도 전시회 예약을 365일 24시간 받을 수 있어야 한다. -> Async(event-driven),Eventual Consistency
         - 결제시스템이 과중되면 관람 예약 신청을 잠시동안 받지 않고, 결제를 잠시 후에 하도록 유도한다. -> Circuit breaker,fallback
        3. 성능 
         - 관람객이 수시로 예약 현황을 확인할 수 있어야 한다. -> CQRS 

## 분석/설계
### Event Storming 결과
    MSAEz 로 모델링한 이벤트스토밍 결과
    http://labs.msaez.io/#/storming/fMnS97KBhKdTR73T20ep9sUs6kE2/9131f35ec41231894db13d063a05c048

#### 1. 완성된 1차 모형
 ![image](https://user-images.githubusercontent.com/87048633/131067490-a61c5806-9c65-4de7-a97a-92522f2fe8d5.png)
 
#### 2-1. 1차 완성본에 대한 기능적 요구사항에 대한 검증 
 ![image](https://user-images.githubusercontent.com/87048633/131067412-9206e2c5-5c40-4b73-9257-2edff25e2195.png) </br>
 (O) 1. 관람객은 방문하려는 날짜와 회차(시간) 선택, 연락처를 입력하고 예약한다. </br>
 (O) 2. 예약 등록시 관람료를 결제해야 한다. </br>
 (O) 3. 관람료가 결제되지 않으면 최종 관람 예약이 성립되지 않는다. </br>
 (O) 4. 예약이 완료되면 수용 가능한 관람객의 수를 감소시킨다. </br>
 (O) 5. 예약을 취소할 수 있고, 예약이 취소되면 결제를 취소한다.  </br>
 (O) 6. 결제가 취소되면 수용 가능한 관람객의 수를 증가시킨다. </br>
 (O) 7. 관람객은 본인의 예약 내역을 조회할 수 있다. </br>
 
#### 2-2. 1차 완성본에 대한 비기능적 요구사항에 대한 검증
 ![image](https://user-images.githubusercontent.com/87048633/131067367-ef31ab2a-28df-44d0-b2e2-e63ec44e5306.png) </br>
 (O) 1. 트랜잭션 </br>
       - 관람료가 결제되어야 최종 관람 예약이 성립된다. -> Sync 호출 </br>
 (O) 2. 장애격리 </br>
       - 전시회 관리 기능이 수행되지 않더라도 전시회 예약을 365일 24시간 받을 수 있어야 한다. -> Async(event-driven),Eventual Consistency </br>
       - 결제시스템이 과중되면 관람 예약 신청을 잠시동안 받지 않고, 결제를 잠시 후에 하도록 유도한다. -> Circuit breaker,fallback </br>
 (O) 3. 성능 </br>
       - 관람객이 수시로 예약 현황을 확인할 수 있어야 한다. -> CQRS </br>
       
#### 3. 최종 모델
 ![image](https://user-images.githubusercontent.com/87048633/130977591-2c1b50a4-6e47-4608-8203-aa30e2ceb9c0.png)

#### 4.Hexagonal Architecture Diagram 도출
 ![image](https://user-images.githubusercontent.com/87048633/130978000-4497cefa-f8c0-48b1-9a22-b0ba99a86734.png)
 
 
## 구현
- 분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 Spring Boot와 Java로 구현하였다.</br>
(각자의 포트넘버는 8081 ~ 808n 이다)</br>
```
cd Booking
mvn spring-boot:run

cd Exhibition
mvn spring-boot:run

cd Payment
mvn spring-boot:run

cd gateway
mvn spring-boot:run
```


- AWS 클라우드의 EKS 서비스 내에 서비스를 모두 빌드한다.
--> kubectl get all 결과 캡처 <--

### DDD의 적용
- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다. 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다.
- 예시 : Exhibition 서비스 (Exhibition.java)
```java
package bookingexhibition;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;
import java.util.Date;

@Entity
@Table(name="Exhibition_table")
public class Exhibition {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String date;
    private Integer time;
    private Integer audienceCnt;
    private String title;

    @PostUpdate
    public void onPostUpdate(){
        IncreasedAudience increasedAudience = new IncreasedAudience();
        BeanUtils.copyProperties(this, increasedAudience);
        increasedAudience.publishAfterCommit();

    }
    @PreRemove
    public void onPreRemove(){
        DecreasedAudience decreasedAudience = new DecreasedAudience();
        BeanUtils.copyProperties(this, decreasedAudience);
        decreasedAudience.publishAfterCommit();

    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getDate() {
        return date;
    }

    public void setDate(String date) {
        this.date = date;
    }
    
    public Integer getTime() {
        return time;
    }

    public void setTime(Integer time) {
        this.time = time;
    }
    
    public Integer getAudienceCnt() {
        return audienceCnt;
    }

    public void setAudienceCnt(Integer audienceCnt) {
        this.audienceCnt = audienceCnt;
    }
    
    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

}
```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다. </br>
- 예시 : Exhibition 서비스 (ExhibitionRepository.java)
```java
package bookingexhibition;

import org.springframework.data.repository.CrudRepository;

public interface ExhibitionRepository extends CrudRepository<Exhibition, Long> {

}
```

### Saga
- 전시회 정보 등록하기 </br>
	http POST http://localhost:8085/exhibitions date=2021-09-01 time=1000 audienceCnt=100 title=Arirang_2021 </br>
	http POST http://localhost:8085/exhibitions date=2021-09-01 time=1300 audienceCnt=100 title=Arirang_2021 </br>
	http POST http://localhost:8085/exhibitions date=2021-09-01 time=1600 audienceCnt=100 title=Arirang_2021 </br>
	http POST http://localhost:8085/exhibitions date=2021-09-01 time=1900 audienceCnt=100 title=Arirang_2021 </br>
	![image](https://user-images.githubusercontent.com/87048633/131498084-32e98d26-f335-477d-bf57-34c4ab5a4ee4.png) </br>

	topic 확인 </br>
	![image](https://user-images.githubusercontent.com/87048633/131497891-f83b5f2c-588e-408c-bf75-7e881fa89547.png) </br>

- 예약된 전시회가 없음을 확인하기 </br>
	booking>http http://localhost:8081/bookings  </br>  
	![image](https://user-images.githubusercontent.com/87048633/131495771-572cedee-6992-419d-bb93-34fe9815b1cc.png) </br>

	mypage>http http://localhost:8081/mypages  </br>
	![image](https://user-images.githubusercontent.com/87048633/131495794-b244b2aa-7991-4176-bc4c-cd17cd9f0795.png) </br>

	payment>http http://localhost:8084/payments  </br>
	![image](https://user-images.githubusercontent.com/87048633/131495854-170811e5-710e-410c-a26b-67908ea52c24.png) </br>

	exhibition>http http://localhost:8085/exhibitions  </br>
	![image](https://user-images.githubusercontent.com/87048633/131495823-b545dbb6-6b8e-4494-a42d-6d4a54d8d8fa.png) </br>
	
	topic 확인 </br>
	![image](https://user-images.githubusercontent.com/87048633/131496093-71437f1b-0014-49ca-9827-ae1fd6cc7d09.png) </br>

- 전시회 예약하기 </br>
	http POST http://localhost:8081/bookings date=2021-08-31 custName=Jane phoneNum=010-9899-9899 exhibitionId=1 bookingStatus=Booked time=1330 amt=10000 </br>
	![image](https://user-images.githubusercontent.com/87048633/131498650-f84aa3f4-cc55-43f4-b73b-68c4ae06975e.png) </br>
	![image](https://user-images.githubusercontent.com/87048633/131498674-7cf6ccb4-7314-4cff-ad74-d10274945dcb.png) </br>
	![image](https://user-images.githubusercontent.com/87048633/131498875-eed0c6be-cee5-4d82-9f0c-d8b7f2d73acb.png) </br>

	topic 확인 </br>
	![image](https://user-images.githubusercontent.com/87048633/131498826-8ac737de-4726-49d6-a990-b4c478f3aff7.png) </br>

- 전시회 예약 취소하기 </br>
	http DELETE http://localhost:8081/bookings/1 </br>
	![image](https://user-images.githubusercontent.com/87048633/131498964-63926e4e-20f3-448c-8d6e-2323eebfc3fc.png) </br>
	![image](https://user-images.githubusercontent.com/87048633/131498973-12ff4de3-8a63-478c-a5f6-d47ab1286657.png) </br>
	![image](https://user-images.githubusercontent.com/87048633/131499048-43142a9f-773a-4c2f-9764-707810f30a64.png) </br>

	topic 확인 </br>
	![image](https://user-images.githubusercontent.com/87048633/131499018-345fe69a-30c2-4941-a9ff-90e003e4d127.png) </br>

- 예약 정보 확인하기(CQRS) </br>
	http GET http://localhost:8081/myPages </br>
	![image](https://user-images.githubusercontent.com/87048633/131499138-3ee98005-7061-45df-9a63-73465ebcd40b.png) </br>


### 비동기식 호출과 Eventual Consistency 
- Booked -> CompletedPayment -> IncreasedAudience 순서로 Event가 처리됨 </br>
	![image](https://user-images.githubusercontent.com/87048633/131503756-78f2072f-4409-45f6-a469-ab96b2dc5366.png) </br>
 
### CQRS </br>
- 예약 정보 확인하기(CQRS) </br>
	http GET http://localhost:8081/myPages </br>
	![image](https://user-images.githubusercontent.com/87048633/131499138-3ee98005-7061-45df-9a63-73465ebcd40b.png) </br>
   
### Correlation 
- 전시회 예약 목록 확인 </br>
  ![image](https://user-images.githubusercontent.com/87048633/131502740-fd89cd72-c165-4e12-bb2c-0ea5fbbe0812.png) </br>
- 특정 예약건만 삭제 </br>
  http DELETE http://localhost:8081/bookings/1 </br>
  ![image](https://user-images.githubusercontent.com/87048633/131502990-97723ee1-551c-4341-bb3c-c6042f892f12.png) </br>
   
### 동기식 호출
분석단계에서의 조건 중 하나로 예약(Booking) -> 결제(Payment)간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다.</br>
호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다.</br>

- 비기능 요구사항 중 하나인 예약(Booking) -> 결제(Payment)간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. </br>
- 호출 프로토콜은 앞서 Rest Repository에 의해 노출되어 있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. </br>
  ![image](https://user-images.githubusercontent.com/87048633/130982925-7e929b83-b1b3-4b94-94dd-339d72bc4a94.png) </br>
- Payment서비스를 연동하기위한 Booking 서비스의 application.yml 파일</br>
  ![image](https://user-images.githubusercontent.com/87048633/130983236-711288ab-3d85-4667-bcaf-258ed8de9e78.png) </br>
- 동기식 호출 후 Payment서비스 처리결과</br>
  ![image](https://user-images.githubusercontent.com/87048633/131062057-fadb35f5-4584-4077-8d29-ec0fa4c76152.png) </br>
  
### Gateway
  - API gateway 를 통해 MSA 진입점을 통일 시킨다.
  - Gateway의 application.yml의 일부</br>
	![image](https://user-images.githubusercontent.com/87048633/130979608-fbc398b2-935d-4bba-a544-23a603f252d3.png) </br>
  - Gateway 서비스 확인 </br>
  	- 서비스 직접 조회</br>
      	![image](https://user-images.githubusercontent.com/87048633/131061668-68d5379f-ce86-4c05-83c3-9cc033315a08.png) </br>
	- Gateway로 조회</br>
      	![image](https://user-images.githubusercontent.com/87048633/131061736-9b6697cc-a16f-49eb-964c-7328bbd95632.png) </br>
      
### 서킷 브레이킹
  - 서킷 브레이킹 프레임워크의 선택 : FeignClient + hystrix </br>
  - Booking 서비스의 application.yaml </br>
  	- Hystrix 설정 : 요청처리 쓰레드에서 처리 시간이 610 밀리세컨드가 넘어서기 시작하여 어느 정도 유지되면 Circuit Breaker 회로가 닫히도록 설정 </br>
   		![image](https://user-images.githubusercontent.com/87048633/131499559-3890198b-4a8f-41fe-ad45-9228abd9d549.png)</br>
  - 피호출 서비스 (결제:Payment.java)의 임의 후바 처리 400 밀리세컨드에서 증감 220 밀리세컨드 정도 왔다갔다 하게 처리 </br>
	![image](https://user-images.githubusercontent.com/87048633/131500001-39284688-2568-46f4-a38a-07a58b34adfd.png) </br>
  - 서킷 브레이킹 처리 이전 </br>
	![image](https://user-images.githubusercontent.com/87048633/131499263-7e3cd971-5e79-41b7-9df2-5ca6ef727103.png) </br>
  - 서킷 브레이킹 처리 이후 </br>
	![image](https://user-images.githubusercontent.com/87048633/131499270-e5fa2aa2-7f0d-4c54-90ad-756746ad3972.png) </br>

### Polyglot Persistent/Programming
- Polyglot Persistent 조건을 만족하기 위해 Exhibition 서비스의 기존 h2 DB를 hsqldb로 변경하여 동작시킨다. (pom.xml) </br>
  ![image](https://user-images.githubusercontent.com/87048633/131501731-9dbdccd3-ac9b-401f-a994-d0909b27caa7.png) </br>
- 재기동 후 전시회 등록 처리</br>
  ![image](https://user-images.githubusercontent.com/87048633/131501662-38a4b5ca-9cf2-4a52-ad8f-10883e696e2b.png) </br>


## 운영
### Deploy/Pipeline 설정
- 각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 aws codebuild를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 buildspec.yml 에 포함되었다.
--> AWS 화면 캡처 <--

### AutoScale Out
  --> 좀 더 공부해 볼 것 <--
  - replica를 동적으로 늘려서 HPA를 설정한다.
  - 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다.
  - 예약 서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다.
  - 예약 서비스의 buildspec.yml에 resource 설정을 추가한다.

### 무정지 재배포
- readiness probe 를 통해 이후 서비스가 활성 상태가 되면 유입을 진행시킨다.
--> 좀 더 공부해볼 것 <--

### 개발/운영 환경 분리 (ConfigMap)
--> 좀 더 공부해볼 것 <--
- ConfigMap을 사용하여 운영과 개발 환경 분리
- kafka환경

### Self-Healing
--> 좀 더 공부해볼 것 <--

