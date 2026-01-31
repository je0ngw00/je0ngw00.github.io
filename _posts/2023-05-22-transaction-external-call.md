---
title: "트랜잭션 내 외부 호출 분리로 커넥션 풀 병목 해결하기"
date: 2023-05-22 16:55:00 +0900
categories: [Development, Spring]
tags: [spring, transaction, connection-pool, performance]
---

## 문제 상황

회사의 비즈니스 특성상 월초에 높은 TPS를 요구합니다. 매월 초에 항상 30분 정도 제대로 된 서비스를 제공할 수 없는 상황이 반복되었습니다.

입사 후 제일 먼저 개선했던 사항을 회고합니다.

크게 두 가지 문제가 있었습니다:
1. 주문번호 채번에서 DB의 row level lock
2. 외부 통신이 같은 트랜잭션에 묶여 커넥션 반납이 길어지는 병목 현상

여기서는 외부 통신이 트랜잭션에 묶여 있을 때 지연이 발생하면 어떻게 되는지 기록합니다.

## 상황 재연

개선 전 주문 프로세스는 다음과 같았습니다:

1. 트랜잭션 시작
2. 요청에 대한 검증
3. PG로 결제 요청 (외부 통신)
4. 주문 생성
5. 커밋

해당 프로세스에 맞게 서비스를 작성하고, 멀티쓰레드 환경을 위해 `@Async`를 설정했습니다.

```java
@Slf4j
@Service
@EnableAsync
public class OrderService {

    private final PgClient pgClient;
    private final OrderRepository orderRepository;
    private final OrderValidator orderValidator;

    public OrderService(PgClient pgClient,
                        OrderRepository orderRepository,
                        OrderValidator orderValidator) {
        this.pgClient = pgClient;
        this.orderRepository = orderRepository;
        this.orderValidator = orderValidator;
    }

    @Async
    @Transactional
    public void create(OrderCreateRequest orderCreateRequest) throws InterruptedException {
        log.info("!! 주문 생성 호출 {} 번째 !!", orderCreateRequest.getCount());
        orderCreateRequest.valid(orderValidator);
        PgResult pgResult = pgClient.pay(orderCreateRequest);
        OrderEntity order = OrderFactory.create(orderCreateRequest, pgResult);
        log.info("## 주문 저장 전 {} 번째 ##", orderCreateRequest.getCount());
        orderRepository.save(order);
        log.info("$$ 주문 저장 후 {} 번째 $$", orderCreateRequest.getCount());
    }
}
```

외부 호출(PgClient)에서 의도적으로 connection을 잡는 설정을 추가했습니다:

```java
@Slf4j
public class PgClientTest implements PgClient{

    @Override
    public PgResult pay(OrderCreateRequest orderCreateRequest) throws InterruptedException {
        log.info("@@ 외부 결제 호출 {} 번째 @@", orderCreateRequest.getCount());
        Thread.sleep(getMillis(orderCreateRequest));
        log.info("** 외부 결제 종료 {} 번째 **", orderCreateRequest.getCount());
        return null;
    }

    private static long getMillis(OrderCreateRequest orderCreateRequest) {
        if(orderCreateRequest.getCount() < 3) {
            return 300000L;
        }
        return 0;
    }
}
```

테스트는 MySQL DB를 사용하고 `global max_connections=9`로 조절, `maximum-pool-size`는 3으로 설정했습니다.

예측으로는 3개의 connection 중 2개가 잡히고 1개만 동작할 것입니다.

호출은 150,000번을 수행했습니다:

```java
@Slf4j
@SpringBootTest
@ActiveProfiles("test")
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class OrderTransactionTest {

    @Autowired
    OrderService orderService;

    @Test
    @DisplayName("주문 생성 테스트")
    void createOrder() throws InterruptedException {
        for (int i = 1; i < 150000; i++) {
            OrderCreateRequest orderCreateRequest = new OrderCreateRequest(i);
            orderService.create(orderCreateRequest);
        }
        Thread.sleep(1000000);
    }
}
```

## 결과

```
2023-05-22 16:55:38.748  INFO [,,] 44349 --- [    Test worker] b.r.c.b.a.s.s.OrderTransactionTest       : Started OrderTransactionTest in 18.73 seconds (JVM running for 20.699)
... 중략
2023-05-22 16:58:36.968  INFO [,520342ce2c4c66a4,520342ce2c4c66a4] 44349 --- [pon-listener-22] b.r.c.b.a.s.service.order.OrderService   : $$ 주문 저장 후 149999 번째 $$
```

약 **3분** 소요되었습니다.

## 개선

외부 호출과 트랜잭션 서비스를 관리하는 Facade를 생성했습니다:

```java
@Slf4j
@Service
@EnableAsync
public class OrderFacade {

    private final PgClient pgClient;
    private final OrderService orderService;

    public OrderFacade(PgClient pgClient,
                       OrderService orderService) {
        this.pgClient = pgClient;
        this.orderService = orderService;
    }

    @Async
    public void create(OrderCreateRequest orderCreateRequest) throws InterruptedException {
        log.info("!! 주문 생성 호출 {} 번째 !!", orderCreateRequest.getCount());
        pgClient.pay(orderCreateRequest);
        log.info("## 주문 저장 전 {} 번째 ##", orderCreateRequest.getCount());
        orderService.facadeCreate(orderCreateRequest);
        log.info("$$ 주문 저장 후 {} 번째 $$", orderCreateRequest.getCount());
    }
}
```

테스트 코드:

```java
@Slf4j
@SpringBootTest
@ActiveProfiles("test")
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class OrderTransactionTest {

    @Autowired
    OrderFacade orderFacade;

    @Test
    @DisplayName("주문 생성 테스트 (트랜잭션 분리)")
    void createOrder() throws InterruptedException {
        for (int i = 1; i < 150000; i++) {
            OrderCreateRequest orderCreateRequest = new OrderCreateRequest(i);
            orderFacade.create(orderCreateRequest);
        }
        Thread.sleep(1000000);
    }
}
```

## 개선 후 결과

```
2023-05-22 17:00:43.187  INFO [coupon-backoffice-api,,] 44842 --- [    Test worker] b.r.c.b.a.s.s.OrderTransactionTest       : Started OrderTransactionTest in 19.135 seconds (JVM running for 21.056)
... 중략
2023-05-22 17:03:27.418  INFO [coupon-backoffice-api,176f65330d2c3f74,176f65330d2c3f74] 44842 --- [pon-listener-10] b.r.c.b.a.s.service.order.OrderFacade    : $$ 주문 저장 후 149999 번째 $$
```

약 **2분 20초**로 개선되었습니다.

## 정리

사실 중략된 로그나 멀티쓰레드에 대한 테스트가 이 주제와 정확히 맞는지 의아한 점이 있어 제대로 테스트가 되었다는 느낌이 없습니다.

하지만 트랜잭션 내의 불필요한 부분에 대한 고민은 계속 해봐야 할 것 같습니다.

트랜잭션은 편리한 기능이지만, 이렇게 분리를 해두면 어느 관점에서 상태를 관리해야 할지 고민도 필요합니다.
