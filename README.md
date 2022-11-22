![image](https://user-images.githubusercontent.com/487999/79708354-29074a80-82fa-11ea-80df-0db3962fb453.png)

# 예제 - 음식배달

## 최종 결과모델
![EDM](https://user-images.githubusercontent.com/118698671/203226941-76c97b4a-4222-4015-a6ed-32f2b490a82e.jpg)
 - 고객이 메뉴를 선택하여 주문한다. (ok)
 - 고객이 선택한 메뉴에 대해 결제한다. (ok)
 - 주문이 되면 주문 내역이 입점상점주인에게 주문정보가 전달된다.  (ok)
 - 상점주는 주문을 수락하거나 거절할 수 있다.  (ok)
 - 상점주는 요리시작때와 완료 시점에 시스템에 상태를 입력한다.  (ok)
 - 고객은 아직 요리가 시작되지 않은 주문은 취소할 수 있다.  (ok)
 - 요리가 완료되면 고객의 지역 인근의 라이더들에 의해 배송건 조회가 가능하다.  (ok)
 - 라이더가 해당 요리를 pick 한후, pick했다고 앱을 통해 통보한다. (ok)
 - 고객이 주문상태를 중간중간 조회한다. (ok)
 - 주문상태가 바뀔 때 마다 카톡으로 알림을 보낸다. (ok)
 - 고객이 요리를 배달 받으면 배송확인 버튼을 탭하여, 모든 거래가 완료된다. (ok)
 - 고객이 주문 후기를 등록한다. (ok)
 - 점주는 주문 후기를 조회한다. (ok)


## 구현 체크포인트
1. Saga (Pub / Sub)
```
@PostPersist
public void onPostPersist() {
    Paid paid = new Paid(this);
    payRepository().save(paid);
    paid.publishAfterCommit();
}

@StreamListener(
    value = KafkaProcessor.INPUT,
    condition = "headers['type']=='Paid'"
)
public void wheneverPaid_SetOrder(@Payload Paid paid) {
    Paid event = paid;
    System.out.println("\n\n##### listener SetOrder : " + paid + "\n\n");
    OrderManagement.setOrder(event);
}
```
2. CQRS
```
@StreamListener(KafkaProcessor.INPUT)
public void whenOrderPlaced_then_CREATE_1(
    @Payload OrderPlaced orderPlaced
) {
    try {
        if (!orderPlaced.validate()) return;

        // view 객체 생성
        OrderList orderList = new OrderList();
        // view 객체에 이벤트의 Value 를 set 함
        orderList.setOrderId(orderPlaced.getOrderId());
        // view 레파지 토리에 save
        orderListRepository.save(orderList);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```
3. Compensation / Correlation
```
@PreRemove
public void onPreRemove() {
    OrderCanceled orderCanceled = new OrderCanceled(this);
    orderRepository().delete(orderCanceled);
    orderCanceled.publishAfterCommit()
}
```
4. Request / Response
```
@FeignClient(name = "Pay", url = "${api.url.Pay}")
public interface PayService {
    @RequestMapping(method = RequestMethod.POST, path = "/pays")
    public void pay(@RequestBody Pay pay);
}

@PostPersist
public void onPostPersist(){
    Paid paid = new Paid(this);
    paid.publishAfterCommit();
}
```
5. Circuit Breaker
```
feign:
  hystrix:
    enabled: true

hystrix:
  command:
    default:
      execution.isolation.thread.timeoutInMilliseconds: 100
```
6. Gateway
```
spring:
  cloud:
    gateway:
      routes:
        - id: order
        uri: http://localhost:8080
        predicates:
          - Path=/order/** 
        - id: ordermanagement
        uri: http://localhost:8081
        predicates:
          - Path=/ordermanagement/** 
        - id: delivery
        uri: http://localhost:8082
        predicates:
          - Path=/delivery/** 
        - id: pay
        uri: http://localhost:8083
        predicates:
          - Path=/pay/** 
        - id: messagemanagement
        uri: http://localhost:8084
        predicates:
          - Path=/messagemanagement/**
```


