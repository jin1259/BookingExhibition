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
 ![image](https://user-images.githubusercontent.com/87048633/131067412-9206e2c5-5c40-4b73-9257-2edff25e2195.png)
 (O) 1. 관람객은 방문하려는 날짜와 회차(시간) 선택, 연락처를 입력하고 예약한다. </br>
 (O) 2. 예약 등록시 관람료를 결제해야 한다. </br>
 (O) 3. 관람료가 결제되지 않으면 최종 관람 예약이 성립되지 않는다. </br>
 (O) 4. 예약이 완료되면 수용 가능한 관람객의 수를 감소시킨다. </br>
 (O) 5. 예약을 취소할 수 있고, 예약이 취소되면 결제를 취소한다.  </br>
 (O) 6. 결제가 취소되면 수용 가능한 관람객의 수를 증가시킨다. </br>
 (O) 7. 관람객은 본인의 예약 내역을 조회할 수 있다. </br>
 
#### 2-2. 1차 완성본에 대한 비기능적 요구사항에 대한 검증
 ![image](https://user-images.githubusercontent.com/87048633/131067367-ef31ab2a-28df-44d0-b2e2-e63ec44e5306.png)
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
- gateway의 application.yml의 일부</br>
![image](https://user-images.githubusercontent.com/87048633/130979608-fbc398b2-935d-4bba-a544-23a603f252d3.png)
</br>

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
- 적용 후 REST API 의 테스트
  --> 시나리오별 수행 결과 화면 캡처 <--

### 비동기식 호출과 Eventual Consistency
- RentalPlaced -> PaymentApproved -> DeliveryStarted -> ProductDecreased 순서로 Event가 처리됨 (전시회 예약 프로세스에 맞춰 수정 필요!)
--> kafka consumer 모니터링 내역 화면 캡처 <-- 

### CQRS
- MyPage에서 예약 신청 여부 / 결제성공여부 확인 (CQRS)
  : MyPage 조회시, 예약 신청 이벤트까지만 수신 내역 확인
   --> http 조회 내역 화면 캡처 <--
  : MyPage 조회시, 모든 이벤트 수신내역 확인
   --> http 조회 내역 화면 캡처 <--
   
### Correlation
--> 좀 더 공부해 볼 것 <-- 
   
### 동기식 호출
분석단계에서의 조건 중 하나로 렌탈(rental) -> 결제(Payment)간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다.
호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다.
rental서비스 내부의 payment 서비스
- 비기능 요구사항 중 하나인 예약(Booking) -> 결제(Payment)간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다.
- 호출 프로토콜은 앞서 Rest Repository에 의해 노출되어 있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다.
  ![image](https://user-images.githubusercontent.com/87048633/130982925-7e929b83-b1b3-4b94-94dd-339d72bc4a94.png) </br>
- Payment 마이크로 서비스를 연동하기위한 Booking 마이크로 서비스의 application.yml 파일</br>
  ![image](https://user-images.githubusercontent.com/87048633/130983236-711288ab-3d85-4667-bcaf-258ed8de9e78.png) </br>
- 동기식 호출 후 payment 서비스 처리결과</br>
  ![image](https://user-images.githubusercontent.com/87048633/131062057-fadb35f5-4584-4077-8d29-ec0fa4c76152.png) </br>
  
### Gateway
  - Gateway 서비스 확인 </br>
  	- 서비스 직접 조회</br>
      	![image](https://user-images.githubusercontent.com/87048633/131061668-68d5379f-ce86-4c05-83c3-9cc033315a08.png) </br>
	- Gateway로 조회</br>
      	![image](https://user-images.githubusercontent.com/87048633/131061736-9b6697cc-a16f-49eb-964c-7328bbd95632.png) </br>
      
### 서킷 브레이킹
--> 좀 더 공부해 볼 것 <--
  - 서킷 브레이킹 프레임워크의 선택: FeignClient + hystrix

### Polyglot Persistent/Programming
- Polyglot Persistent 조건을 만족하기 위해 기존 h2 DB를 hsqldb로 변경하여 동작시킨다.
```
		<!--
   		<dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
		</dependency>
    		-->

		<!-- polyglot start -->
		<dependency>
			<groupId>org.hsqldb</groupId>
			<artifactId>hsqldb</artifactId>
			<scope>runtime</scope>
		</dependency>
		<!-- polyglot end -->
```
- pom.yml 파일 내 DB 정보 변경 및 재기동 후 전시회 예약 처리</br>
  --> 처리 결과 화면 캡쳐 <--

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

