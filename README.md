# Restaurant 

음식을 주문하고 요리하여 배달하는 현황을 확인 할 수 있는 CNA 입니다.

# Table of contents

- [Restaurant](#---)
  - [서비스 시나리오](#서비스-시나리오)
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
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)


# 서비스 시나리오

숙소 공유 서비스 커버하기

## 기능적 요구사항

1. 고객이 주문을 하면 주문정보를 바탕으로 요리가 시작된다.
1. 요리가 완료되면 배달이 시작된다. 
1. 고객이 주문취소를 하게 되면 요리가 취소된다.
1. 고객 주문 시 제고가 없을 경우 주문이 취소된다. 
1. 고객은 Mypage를 통해, 주문과 요리, 배달의 전체 상황을 조회할수 있다.

## 비기능적 요구사항
1. 트랜잭션
    1. 주문 취소시 요리취소가 함께 처리되도록 한다. - Sync 호출 
    1. 제고 부족시 주문이 함께 취소 되도록 해야 한다.
1. 장애격리
    1. 주문시스템이 과중되면 사용자를 잠시동안 받지 않고 잠시후에 주문하도록 유도한다 - Circuit breaker, fallback
1. 성능
    1. 고객이 mypage를 통해 주문 - 요리 - 배달의 전체 상황을 조회할수 있다. - CQRS


# 분석/설계

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과 : http://www.msaez.io/#/storming/t5Z5EXdDP0UOZDvGzeNH61hF8qG3/mine/52e31337a76ddeacc1d288ea11e24158/-MH4jm58lJNE_9tgT82F
![EventStorming_Restaurant](https://user-images.githubusercontent.com/54210936/93179266-66201d80-f770-11ea-9530-aae4dfef7e4d.png)

### 이벤트 도출
1. 주문됨
1. 주문취소됨
1. 요리재고체크됨
1. 요리시작됨
1. 배달시작됨


### 액터, 커맨드 부착하여 읽기 좋게

### 어그리게잇으로 묶기

  * 고객의 주문(Order), 식당의 요리(Cook), 배달(Delivery)은 그와 연결된 command와 event 들에 의하여 트랙잭션이 유지되어야 하는 단위로 그들끼리 묶어 줌 

### 폴리시 부착 (괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남)

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

### 기능적 요구사항 검증

  * 호스트가 본인이 등록한 숙소 목록을 조회한다. (ok)
  * 호스트가 속소를 신규 공유한다. (ok)
  * 호스트가 숙소 정보를 변경한다. (ok)
  * 호스트가 숙소 공유를 중단한다. (ok)
  * 게스트가 숙소를 선택하여 사용 예약한다. (ok)
  * 게스트가 결제한다. (ok)
  * 결제가 완료되면, 결제 & 예약 내용을 게스트에게 전달한다. (ok)
  * 예약 내역을 호스트에게 전달한다. (ok)
  * 게스트는 본인의 예약 내용 및 상태를 조회한다. (ok)
  * 게스트는 본인의 예약을 취소할 수 있다. (ok)
  * 예약이 취소되면, 결제를 취소한다. (ok)
  * 결제가 취소되면, 결제 취소 내용을 게스트에게 전달한다. (ok)
  * 게스트는 사용한 숙소의 사용 후기를 작성할 수 있다. (ok)
  * 게스트가 사용한 숙소의 사용 후기를 작성한다. (ok)
  * 숙소 사용 후기가 등록되면, 사용 금액의 1%를 마일리지로 적립한다. (ok)
  * 마일리지가 적립되면, 적립 내용을 게스트에게 전달한다. (ok)


# 구현:

분석/설계 단계에서 도출된 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트 JAVA로 구현하였다. 각 마이크로 서비스들은 Kafka와 RestApi로 연동되어 지며, H2 DB를 사용한다.


## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 주문- Order 마이크로서비스).

```
package myProject_LSP;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Order_table")
public class Order {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Integer restaurantId;
    private Integer restaurantMenuId;
    private Integer customerId;
    private Integer qty;
    private Long modifiedDate;
    private String status;

    @PrePersist
    public void onPrePersist(){

        if(!"ORDER : COOK CANCELED".equals(this.getStatus())){
            this.setStatus("ORDER : ORDERED");
        }
        this.setModifiedDate(System.currentTimeMillis());
    }

    @PostPersist
    public void onPostPersist(){
        Ordered ordered = new Ordered();
        BeanUtils.copyProperties(this, ordered);
        System.out.println(ordered.getStatus()+ "#######################33");
        if(!"ORDER : COOK CANCELED".equals(ordered.getStatus())){
            ordered.publishAfterCommit();
        }
    }

    @PreRemove
    public void onPreRemove(){
        OrderCancelled orderCancelled = new OrderCancelled();
        this.setStatus("ORDER : ORDER CANCELED");
        BeanUtils.copyProperties(this, orderCancelled);
        orderCancelled.publishAfterCommit();
        myProject_LSP.external.Cancellation cancellation = new myProject_LSP.external.Cancellation();
        cancellation.setOrderId(this.getId());
        BeanUtils.copyProperties(this, cancellation);


        OrderApplication.applicationContext.getBean(myProject_LSP.external.CancellationService.class)
            .cancel(cancellation);

    }
    
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public Integer getRestaurantId() {
        return restaurantId;
    }

    public void setRestaurantId(Integer restaurantId) {
        this.restaurantId = restaurantId;
    }
    public Integer getRestaurantMenuId() {
        return restaurantMenuId;
    }

    public void setRestaurantMenuId(Integer restaurantMenuId) {
        this.restaurantMenuId = restaurantMenuId;
    }
    public Integer getCustomerId() {
        return customerId;
    }

    public void setCustomerId(Integer customerId) {
        this.customerId = customerId;
    }
    public Integer getQty() {
        return qty;
    }

    public void setQty(Integer qty) {
        this.qty = qty;
    }
    public Long getModifiedDate() {
        return modifiedDate;
    }

    public void setModifiedDate(Long modifiedDate) {
        this.modifiedDate = modifiedDate;
    }
    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }
}


```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package myProject_LSP;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface OrderRepository extends PagingAndSortingRepository<Order, Long>{

}
```
- 적용 후 REST API 의 테스트
```
# 숙소 서비스의 등록처리
http localhost:8081/숙소s room="풀빌라"

# 예약 서비스의 예약처리
http localhost:8082/예약s roomId=1 

# 결제 서비스의 결제처리
http localhost:8083/결제s 예약ID=1

# 예약 상태 확인
http localhost:8082/예약s/1

```

## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 주문->취소 간의 호출은 트랜잭션으로 처리. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 결제 서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
@FeignClient(name="cook", url="${api.url.cook}")
public interface CancellationService {

    @RequestMapping(method= RequestMethod.GET, path="/cancellations")
    public void cancel(@RequestBody Cancellation cancellation);

}

```

- 주문을 받은 직후(@PrePersist) 제고를 체크하고, 이상이 없을 경우 (@PostPersist) 요리를 요청하도록 처리
```

  @PrePersist
  public void onPrePersist(){
      // 요리를 할 수 있는 재고가 없을 때 요리를 시작한다
      if(this.getQty() <= 0) {
         this.setStatus("COOK : QTY OVER");
         flowchk = false;
      }
  }
  @PostPersist
  public void onPostPersist(){
     if(flowchk) {   // 요리를 할 수 있는 재고가 있을 때 요리를 시작한다
        Cooked cooked = new Cooked();
        BeanUtils.copyProperties(this, cooked);
        this.setStatus("COOK : ORDER RECEIPT");
        this.qty--;
        cooked.publishAfterCommit();
      }else{
        CookQtyChecked cookQtyChecked = new CookQtyChecked();
        BeanUtils.copyProperties(this, cookQtyChecked);
        cookQtyChecked.publishAfterCommit();
     }
   }

```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 주문도 못받는다는 것을 확인:


## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트

결제가 이루어진 후에 알림 처리는 동기식이 아니라 비 동기식으로 처리하여 알림 시스템의 처리를 위하여 예약이 블로킹 되지 않아도록 처리한다.
 
- 이를 위하여 예약관리, 결제관리에 기록을 남긴 후에 곧바로 완료되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
```
package mybnb;

@Entity
@Table(name="결제관리_table")
public class 결제관리 {

 ...
    @PrePersist
    public void onPrePersist(){
        결제승인됨 결제승인됨 = new 결제승인됨();
        BeanUtils.copyProperties(this, 결제승인됨);
        결제승인됨.publish();
    }

}
```

- 알림 서비스에서는 예약완료, 결제승인 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:
```
@Service
public class PolicyHandler{

    @StreamListener(KafkaProcessor.INPUT)
    public void whenever예약됨_알림처리(@Payload 예약됨 예약됨){
        if(예약됨.isMe()){
            System.out.println("##### listener 예약됨 알림처리 : " + 예약됨.toJson());
        }
    }

    @StreamListener(KafkaProcessor.INPUT)
    public void whenever결제승인됨_알림처리(@Payload 결제승인됨 결제승인됨){
        if(결제승인됨.isMe()){
            System.out.println("##### listener 결제승인됨 알림처리 : " + 결제승인됨.toJson());
        }
    }
    
}
```

- 실제 구현을 하자면, 카톡 등으로 알림을 처리합니다.:
```
  @Autowired 알림이력Repository 알림이력Repository;
  
  @StreamListener(KafkaProcessor.INPUT)
  public void whenever결제승인됨_알림처리(@Payload 결제승인됨 결제승인됨){

      if(결제승인됨.isMe()){
          카톡전송(" Mybnb 알림 : " + 결제승인됨.toString());

          알림이력 알림이력 = new 알림이력();
          알림이력.setId(결제승인됨.get예약Id());
          알림이력Repository.save(알림이력);
      }
  }

```

알림 시스템은 예약/결제와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 알림 시스템이 유지보수로 인해 잠시 내려간 상태라도 예약을 받는데 문제가 없다:
```
# 알림 서비스를 잠시 내려놓음 (ctrl+c)

# 예약처리
http localhost:8082/예약s roomId=1  #Fail
http localhost:8082/예약s roomId=2  #Fail

# 알림이력 확인
http localhost:8082/알림이력s # 알림이력조회 불가

# 알림 서비스 기동
cd 알림
mvn spring-boot:run

# 알림이력 확인
http localhost:8082/알림이력s # 알림이력조회
```

# 운영

## CI/CD 설정

  * 각 구현체들은 github의 각각의 source repository 에 구성
  * 사용 CI/CD 플랫폼은 ? 
  
## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 예약-->결제 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml

hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

```

- 피호출 서비스(결제) 의 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게
```
# 결제관리.java (Entity)

    @PrePersist
    public void onPrePersist(){  //결제이력을 저장한 후 적당한 시간 끌기

        ...
        
        try {
            Thread.currentThread().sleep((long) (400 + Math.random() * 220));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```

* kubectl create deploy siege --image=apexacme/siege-nginx
* kubtctl exec -it pod/siege -- /bin/bash
* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 100명
- 60초 동안 실시

```
$ siege -c100 -t60S -r10 --content-type "application/json" 'http://localhost:8081/orders POST {"item": "chicken"}'

** SIEGE 4.0.5
** Preparing 100 concurrent users for battle.
The server is now under siege...

HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.73 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.75 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.77 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.97 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.81 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.87 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.12 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.16 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.17 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.26 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.25 secs:     207 bytes ==> POST http://localhost:8081/orders

* 요청이 과도하여 CB를 동작함 요청을 차단

HTTP/1.1 500     1.29 secs:     248 bytes ==> POST http://localhost:8081/orders   
HTTP/1.1 500     1.24 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.23 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.42 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     2.08 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.29 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.24 secs:     248 bytes ==> POST http://localhost:8081/orders

* 요청을 어느정도 돌려보내고나니, 기존에 밀린 일들이 처리되었고, 회로를 닫아 요청을 다시 받기 시작

HTTP/1.1 201     1.46 secs:     207 bytes ==> POST http://localhost:8081/orders  
HTTP/1.1 201     1.33 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.36 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.63 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.65 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.69 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.71 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.71 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.74 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.76 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.79 secs:     207 bytes ==> POST http://localhost:8081/orders

* 다시 요청이 쌓이기 시작하여 건당 처리시간이 610 밀리를 살짝 넘기기 시작 => 회로 열기 => 요청 실패처리

HTTP/1.1 500     1.93 secs:     248 bytes ==> POST http://localhost:8081/orders    
HTTP/1.1 500     1.92 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.93 secs:     248 bytes ==> POST http://localhost:8081/orders

* 생각보다 빨리 상태 호전됨 - (건당 (쓰레드당) 처리시간이 610 밀리 미만으로 회복) => 요청 수락

HTTP/1.1 201     2.24 secs:     207 bytes ==> POST http://localhost:8081/orders  
HTTP/1.1 201     2.32 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.16 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.19 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.19 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.19 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.21 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.29 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.30 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.38 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.59 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.61 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.62 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.64 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.01 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.27 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.33 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.45 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.52 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.57 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.69 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.70 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.69 secs:     207 bytes ==> POST http://localhost:8081/orders

* 이후 이러한 패턴이 계속 반복되면서 시스템은 도미노 현상이나 자원 소모의 폭주 없이 잘 운영됨

HTTP/1.1 500     4.76 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.23 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.76 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.74 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.82 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.82 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.84 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.66 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     5.03 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.22 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.19 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.18 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.69 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.65 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     5.13 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.84 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.25 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.25 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.80 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.87 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.33 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.86 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.96 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.34 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.04 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.50 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.95 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.54 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.65 secs:     207 bytes ==> POST http://localhost:8081/orders


:
:

Transactions:		        1025 hits
Availability:		       63.55 %
Elapsed time:		       59.78 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02
Successful transactions:        1025
Failed transactions:	         588
Longest transaction:	        9.20
Shortest transaction:	        0.00

```
- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 하지만, 63.55% 가 성공하였고, 46%가 실패했다는 것은 고객 사용성에 있어 좋지 않기 때문에 Retry 설정과 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.

- Retry 의 설정 (istio)
- Availability 가 높아진 것을 확인 (siege)

### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy pay --min=1 --max=10 --cpu-percent=15
```
- CB 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.
```
siege -c100 -t120S -r10 --content-type "application/json" 'http://localhost:8081/orders POST {"item": "chicken"}'
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy pay -w
```
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:
```
NAME    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
pay     1         1         1            1           17s
pay     1         2         1            1           45s
pay     1         4         1            1           1m
:
```
- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 
```
Transactions:		        5078 hits
Availability:		       92.45 %
Elapsed time:		       120 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02
```


## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c100 -t120S -r10 --content-type "application/json" 'http://localhost:8081/orders POST {"item": "chicken"}'

** SIEGE 4.0.5
** Preparing 100 concurrent users for battle.
The server is now under siege...

HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
:

```

- 새버전으로의 배포 시작
```
kubectl set image ...
```

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인
```
Transactions:		        3078 hits
Availability:		       70.45 %
Elapsed time:		       120 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02

```
- 배포기간중 Availability 가 평소 100%에서 70% 대로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 를 설정함:
```
# deployment.yaml 의 readiness probe 의 설정:


kubectl apply -f kubernetes/deployment.yaml
```

- 동일한 시나리오로 재배포 한 후 Availability 확인:
```
Transactions:		        3078 hits
Availability:		       100 %
Elapsed time:		       120 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02

```

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.


# 신규 개발 조직의 추가

  ![image](https://user-images.githubusercontent.com/487999/79684133-1d6c4300-826a-11ea-94a2-602e61814ebf.png)


## 마케팅팀의 추가
    - KPI: 신규 고객의 유입률 증대와 기존 고객의 충성도 향상
    - 구현계획 마이크로 서비스: 기존 customer 마이크로 서비스를 인수하며, 고객에 음식 및 맛집 추천 서비스 등을 제공할 예정

## 이벤트 스토밍 
    ![image](https://user-images.githubusercontent.com/487999/79685356-2b729180-8273-11ea-9361-a434065f2249.png)


## 헥사고날 아키텍처 변화 

![image](https://user-images.githubusercontent.com/487999/79685243-1d704100-8272-11ea-8ef6-f4869c509996.png)

## 구현  

기존의 마이크로 서비스에 수정을 발생시키지 않도록 Inbund 요청을 REST 가 아닌 Event 를 Subscribe 하는 방식으로 구현. 기존 마이크로 서비스에 대하여 아키텍처나 기존 마이크로 서비스들의 데이터베이스 구조와 관계없이 추가됨. 

## 운영과 Retirement

Request/Response 방식으로 구현하지 않았기 때문에 서비스가 더이상 불필요해져도 Deployment 에서 제거되면 기존 마이크로 서비스에 어떤 영향도 주지 않음.

* [비교] 결제 (pay) 마이크로서비스의 경우 API 변화나 Retire 시에 app(주문) 마이크로 서비스의 변경을 초래함:

예) API 변화시
```
# Order.java (Entity)

    @PostPersist
    public void onPostPersist(){

        fooddelivery.external.결제이력 pay = new fooddelivery.external.결제이력();
        pay.setOrderId(getOrderId());
        
        Application.applicationContext.getBean(fooddelivery.external.결제이력Service.class)
                .결제(pay);

                --> 

        Application.applicationContext.getBean(fooddelivery.external.결제이력Service.class)
                .결제2(pay);

    }
```

예) Retire 시
```
# Order.java (Entity)

    @PostPersist
    public void onPostPersist(){

        /**
        fooddelivery.external.결제이력 pay = new fooddelivery.external.결제이력();
        pay.setOrderId(getOrderId());
        
        Application.applicationContext.getBean(fooddelivery.external.결제이력Service.class)
                .결제(pay);

        **/
    }
```
