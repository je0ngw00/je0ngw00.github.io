---
title: "AWS SNS + SQS로 이벤트 기반 아키텍처 구축하기"
date: 2023-06-07 10:00:00 +0900
categories: [Development, AWS]
tags: [aws, sqs, sns, event-driven, spring, java]
---

## 개요

도메인 간 API 직접 호출로 인한 강한 결합 문제를 해결하기 위해 SNS + SQS 기반의 이벤트 기반 아키텍처(Event Driven Architecture)를 도입했습니다. Kafka도 고려했지만, 러닝커브와 비용 문제로 AWS 관리형 서비스를 선택했습니다.

> **시리즈**: [SQS+SNS vs Kafka 비교](/posts/sqs-sns-vs-kafka/) · [LocalStack으로 로컬 테스트 환경 구축하기](/posts/localstack-sqs-sns-part1/)
>
> **📌 2025-01 업데이트**: Spring Cloud AWS 3.x 사용법은 [LocalStack 2편](/posts/localstack-sqs-sns-part2/)을 참고하세요. 아래 코드는 2.x 버전 기준입니다.

## SNS와 SQS

### SNS (Simple Notification Service)

Pub/Sub 모델의 메시지 발행 서비스입니다. 하나의 토픽에 여러 구독자를 연결할 수 있습니다.

```
Publisher → SNS Topic → SQS Queue A (서비스 A)
                     → SQS Queue B (서비스 B)
                     → Lambda (처리 로직)
```

**주요 특징:**
- **Fan-out**: 하나의 메시지를 여러 구독자에게 동시 전달
- **필터링**: 구독자별로 원하는 메시지만 수신 가능
- **FIFO 토픽**: 메시지 순서 보장이 필요한 경우 사용

### SQS (Simple Queue Service)

메시지를 안전하게 저장하고 처리하는 큐 서비스입니다.

**주요 특징:**
- **메시지 보존**: 최대 14일간 메시지 보관
- **DLQ(Dead Letter Queue)**: 처리 실패 메시지 별도 관리
- **가시성 타임아웃**: 중복 처리 방지
- **FIFO 큐**: 순서 보장과 정확히 한 번 처리

## 왜 SNS + SQS 조합인가?

SNS만 사용하면 몇 가지 문제가 있습니다:

| 문제 | SNS 단독 | SNS + SQS |
|------|---------|-----------|
| 구독자 장애 시 | 메시지 유실 가능 | SQS에 보관됨 |
| 재처리 | 불가능 | 언제든 재처리 가능 |
| 처리 속도 조절 | 불가능 | 컨슈머가 자체 속도로 처리 |
| 배압(Backpressure) | 없음 | 큐가 버퍼 역할 |

## Spring Boot에서 구현하기

### 의존성 추가

```xml
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-messaging</artifactId>
    <version>2.4.4</version>
</dependency>
```

### 설정

```yaml
# application.yml
cloud:
  aws:
    region:
      static: ap-northeast-2
    credentials:
      access-key: ${AWS_ACCESS_KEY}
      secret-key: ${AWS_SECRET_KEY}

app:
  aws:
    sns:
      order-topic-arn: arn:aws:sns:ap-northeast-2:123456789:order-events
    sqs:
      order-queue-url: https://sqs.ap-northeast-2.amazonaws.com/123456789/order-queue
```

### 메시지 발행 (Publisher)

```java
@Service
@RequiredArgsConstructor
public class OrderEventPublisher {

    private final AmazonSNS amazonSNS;

    @Value("${app.aws.sns.order-topic-arn}")
    private String topicArn;

    public void publishOrderCreated(Order order) {
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(),
            order.getUserId(),
            order.getTotalAmount(),
            LocalDateTime.now()
        );

        PublishRequest request = new PublishRequest()
            .withTopicArn(topicArn)
            .withMessage(toJson(event))
            .withMessageAttributes(Map.of(
                "eventType", new MessageAttributeValue()
                    .withDataType("String")
                    .withStringValue("ORDER_CREATED")
            ));

        amazonSNS.publish(request);
    }

    private String toJson(Object obj) {
        try {
            return new ObjectMapper().writeValueAsString(obj);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("JSON 변환 실패", e);
        }
    }
}
```

### 메시지 수신 (Consumer)

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderEventListener {

    private final InventoryService inventoryService;
    private final ObjectMapper objectMapper;

    @SqsListener("${app.aws.sqs.order-queue-url}")
    public void handleOrderCreated(String message) {
        try {
            // SNS에서 온 메시지는 한 번 더 감싸져 있음
            SnsNotification notification = objectMapper.readValue(message, SnsNotification.class);
            OrderCreatedEvent event = objectMapper.readValue(
                notification.getMessage(),
                OrderCreatedEvent.class
            );

            log.info("주문 이벤트 수신: orderId={}", event.getOrderId());
            inventoryService.decreaseStock(event);

        } catch (Exception e) {
            log.error("주문 이벤트 처리 실패", e);
            throw e;  // 실패 시 DLQ로 이동
        }
    }
}

@Data
public class SnsNotification {
    @JsonProperty("Message")
    private String message;

    @JsonProperty("MessageId")
    private String messageId;

    @JsonProperty("Timestamp")
    private String timestamp;
}
```

### 메시지 필터링

특정 이벤트 타입만 수신하려면 SNS 구독에 필터 정책을 설정합니다.

```json
{
  "eventType": ["ORDER_CREATED", "ORDER_CANCELLED"]
}
```

## DLQ 설정

처리 실패한 메시지를 별도 큐로 이동시켜 나중에 재처리할 수 있습니다.

```java
// DLQ 메시지 재처리
@Scheduled(cron = "0 0 6 * * *")  // 매일 새벽 6시
public void redriveDeadLetterMessages() {
    StartMessageMoveTaskRequest request = new StartMessageMoveTaskRequest()
        .withSourceArn("arn:aws:sqs:ap-northeast-2:123456789:order-queue-dlq");

    amazonSQS.startMessageMoveTask(request);
}
```

## 운영 시 주의사항

### 멱등성 처리

네트워크 문제로 같은 메시지가 여러 번 전달될 수 있습니다.

```java
@Service
@RequiredArgsConstructor
public class IdempotentOrderHandler {

    private final RedisTemplate<String, String> redisTemplate;

    public void handle(OrderCreatedEvent event) {
        String key = "processed:order:" + event.getOrderId();

        // 이미 처리된 메시지인지 확인
        Boolean isNew = redisTemplate.opsForValue()
            .setIfAbsent(key, "1", Duration.ofHours(24));

        if (Boolean.FALSE.equals(isNew)) {
            log.info("이미 처리된 주문: {}", event.getOrderId());
            return;
        }

        // 실제 처리 로직
        processOrder(event);
    }
}
```

### 가시성 타임아웃 설정

메시지 처리 시간보다 가시성 타임아웃을 길게 설정해야 합니다. 그렇지 않으면 처리 중인 메시지가 다른 컨슈머에게 다시 전달됩니다.

```java
// 처리 시간이 긴 경우 타임아웃 연장
@SqsListener(value = "order-queue", deletionPolicy = SqsMessageDeletionPolicy.NEVER)
public void handleLongRunningTask(String message, Acknowledgment ack) {
    // 처리...
    ack.acknowledge();  // 수동으로 메시지 삭제
}
```

## 정리

| 구성 요소 | 역할 |
|----------|------|
| SNS | 이벤트 발행, Fan-out |
| SQS | 메시지 버퍼링, 안정적 전달 |
| DLQ | 실패 메시지 보관 및 재처리 |

SNS + SQS 조합은 AWS 생태계에서 이벤트 기반 아키텍처를 구축할 때 가장 실용적인 선택입니다. 관리 부담이 적고, 다른 AWS 서비스와의 연동도 쉽습니다.
