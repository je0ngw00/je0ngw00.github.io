---
title: "LocalStack으로 AWS SQS/SNS 로컬 테스트 환경 구축하기 (2편): Spring Boot 통합 테스트"
date: 2023-07-16 10:00:00 +0900
categories: [Development, Testing]
tags: [localstack, aws, sqs, sns, spring-boot, testcontainers, testing]
---

## 개요

1편에서 LocalStack 환경을 구성했습니다. 이번 편에서는 Spring Cloud AWS 3.x와 Testcontainers를 사용하여 통합 테스트를 작성하는 방법을 다룹니다.

> **시리즈**: [1편: LocalStack 환경 구성](/posts/localstack-sqs-sns-part1/) · [AWS SQS+SNS 기본 개념](/posts/aws-sqs-sns-intro/)
>
> **📌 2025-01 업데이트**: Spring Cloud AWS 버전을 3.3.0으로 업데이트했습니다.

## 의존성 설정

### Spring Cloud AWS 3.x

```xml
<!-- pom.xml -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.awspring.cloud</groupId>
            <artifactId>spring-cloud-aws-dependencies</artifactId>
            <version>3.3.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- SQS -->
    <dependency>
        <groupId>io.awspring.cloud</groupId>
        <artifactId>spring-cloud-aws-starter-sqs</artifactId>
    </dependency>

    <!-- SNS -->
    <dependency>
        <groupId>io.awspring.cloud</groupId>
        <artifactId>spring-cloud-aws-starter-sns</artifactId>
    </dependency>

    <!-- Testcontainers -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>localstack</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

> **주의**: Spring Cloud AWS 3.x는 패키지명이 `io.awspring.cloud`로 변경되었습니다. (2.x는 `org.springframework.cloud.aws`)

## 환경별 설정 분리

### Config 클래스 구성

```java
@Configuration
public class AwsConfig {

    @Configuration
    @Profile("prod")
    static class ProdConfig {
        // 실제 AWS 연결 - 기본 설정 사용
    }

    @Configuration
    @Profile("local")
    @EnableConfigurationProperties(LocalStackProperties.class)
    static class LocalConfig {

        @Bean
        public SqsAsyncClient sqsAsyncClient(LocalStackProperties props) {
            return SqsAsyncClient.builder()
                .endpointOverride(URI.create(props.getEndpoint()))
                .region(Region.AP_NORTHEAST_2)
                .credentialsProvider(StaticCredentialsProvider.create(
                    AwsBasicCredentials.create("test", "test")
                ))
                .build();
        }

        @Bean
        public SnsAsyncClient snsAsyncClient(LocalStackProperties props) {
            return SnsAsyncClient.builder()
                .endpointOverride(URI.create(props.getEndpoint()))
                .region(Region.AP_NORTHEAST_2)
                .credentialsProvider(StaticCredentialsProvider.create(
                    AwsBasicCredentials.create("test", "test")
                ))
                .build();
        }
    }
}

@ConfigurationProperties(prefix = "localstack")
@Data
public class LocalStackProperties {
    private String endpoint = "http://localhost:4566";
}
```

### application.yml

```yaml
spring:
  profiles:
    active: local

---
spring:
  config:
    activate:
      on-profile: local

localstack:
  endpoint: http://localhost:4566

spring.cloud.aws:
  region:
    static: ap-northeast-2
  sqs:
    endpoint: http://localhost:4566
  sns:
    endpoint: http://localhost:4566

---
spring:
  config:
    activate:
      on-profile: prod

spring.cloud.aws:
  region:
    static: ap-northeast-2
```

## 메시지 발행/수신 구현

### SNS 발행자

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderEventPublisher {

    private final SnsTemplate snsTemplate;

    @Value("${app.sns.order-topic}")
    private String topicArn;

    public void publishOrderCreated(OrderCreatedEvent event) {
        log.info("Publishing order created event: {}", event.getOrderId());

        snsTemplate.convertAndSend(topicArn, event);
    }
}
```

### SQS 리스너

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderEventListener {

    private final OrderService orderService;

    @SqsListener("${app.sqs.order-queue}")
    public void handleOrderCreated(
            @Payload OrderCreatedEvent event,
            @Header("MessageId") String messageId,
            Acknowledgement ack) {

        log.info("Received order event: messageId={}, orderId={}",
                messageId, event.getOrderId());

        try {
            orderService.processOrder(event);
            ack.acknowledge();
        } catch (Exception e) {
            log.error("Failed to process order: {}", event.getOrderId(), e);
            // acknowledge 하지 않으면 재처리됨
            throw e;
        }
    }
}
```

## Testcontainers 통합 테스트

### 테스트 설정

```java
@Testcontainers
@SpringBootTest
@ActiveProfiles("test")
class OrderEventIntegrationTest {

    @Container
    static LocalStackContainer localStack = new LocalStackContainer(
            DockerImageName.parse("localstack/localstack:3.0")
    )
    .withServices(Service.SQS, Service.SNS)
    .withEnv("DEFAULT_REGION", "ap-northeast-2");

    @DynamicPropertySource
    static void overrideProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.cloud.aws.sqs.endpoint",
            () -> localStack.getEndpointOverride(Service.SQS).toString());
        registry.add("spring.cloud.aws.sns.endpoint",
            () -> localStack.getEndpointOverride(Service.SNS).toString());
        registry.add("spring.cloud.aws.credentials.access-key",
            () -> localStack.getAccessKey());
        registry.add("spring.cloud.aws.credentials.secret-key",
            () -> localStack.getSecretKey());
        registry.add("spring.cloud.aws.region.static",
            () -> localStack.getRegion());
    }

    @BeforeAll
    static void setupAws() throws Exception {
        // SQS 큐 생성
        localStack.execInContainer(
            "awslocal", "sqs", "create-queue",
            "--queue-name", "order-queue"
        );

        // SNS 토픽 생성
        localStack.execInContainer(
            "awslocal", "sns", "create-topic",
            "--name", "order-events"
        );

        // 구독 설정
        localStack.execInContainer(
            "awslocal", "sns", "subscribe",
            "--topic-arn", "arn:aws:sns:ap-northeast-2:000000000000:order-events",
            "--protocol", "sqs",
            "--notification-endpoint", "arn:aws:sqs:ap-northeast-2:000000000000:order-queue"
        );
    }

    @Autowired
    private OrderEventPublisher publisher;

    @Autowired
    private SqsAsyncClient sqsClient;

    @Test
    void shouldPublishAndReceiveOrderEvent() throws Exception {
        // Given
        OrderCreatedEvent event = new OrderCreatedEvent("order-123", 10000L);

        // When
        publisher.publishOrderCreated(event);

        // Then: 메시지 수신 확인
        await().atMost(Duration.ofSeconds(10))
            .untilAsserted(() -> {
                var response = sqsClient.receiveMessage(req -> req
                    .queueUrl("http://localhost:" +
                        localStack.getMappedPort(4566) +
                        "/000000000000/order-queue")
                    .maxNumberOfMessages(1)
                ).get();

                assertThat(response.messages()).isNotEmpty();
                assertThat(response.messages().get(0).body())
                    .contains("order-123");
            });
    }
}
```

### 테스트 전용 설정

```yaml
# application-test.yml
spring:
  cloud:
    aws:
      region:
        static: ap-northeast-2

app:
  sqs:
    order-queue: order-queue
  sns:
    order-topic: arn:aws:sns:ap-northeast-2:000000000000:order-events
```

## 리스너 테스트

SqsListener가 정상 동작하는지 테스트합니다.

```java
@Testcontainers
@SpringBootTest
@ActiveProfiles("test")
class OrderEventListenerTest {

    @Container
    static LocalStackContainer localStack = new LocalStackContainer(
            DockerImageName.parse("localstack/localstack:3.0"))
        .withServices(Service.SQS);

    @DynamicPropertySource
    static void overrideProperties(DynamicPropertyRegistry registry) {
        // ... 동일한 설정
    }

    @Autowired
    private SqsTemplate sqsTemplate;

    @MockBean
    private OrderService orderService;

    @Test
    void shouldProcessReceivedMessage() throws Exception {
        // Given
        OrderCreatedEvent event = new OrderCreatedEvent("order-456", 20000L);

        // When: SQS에 직접 메시지 전송
        sqsTemplate.send("order-queue", event);

        // Then: OrderService가 호출되는지 확인
        await().atMost(Duration.ofSeconds(10))
            .untilAsserted(() -> {
                verify(orderService).processOrder(argThat(e ->
                    e.getOrderId().equals("order-456")
                ));
            });
    }
}
```

## 공통 테스트 설정 추출

여러 테스트에서 LocalStack 설정을 재사용하려면 추상 클래스로 추출합니다.

```java
@Testcontainers
@SpringBootTest
@ActiveProfiles("test")
public abstract class AbstractLocalStackTest {

    @Container
    protected static LocalStackContainer localStack = new LocalStackContainer(
            DockerImageName.parse("localstack/localstack:3.0"))
        .withServices(Service.SQS, Service.SNS)
        .withEnv("DEFAULT_REGION", "ap-northeast-2")
        .withReuse(true);  // 컨테이너 재사용으로 테스트 속도 향상

    @DynamicPropertySource
    static void overrideProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.cloud.aws.sqs.endpoint",
            () -> localStack.getEndpointOverride(Service.SQS).toString());
        registry.add("spring.cloud.aws.sns.endpoint",
            () -> localStack.getEndpointOverride(Service.SNS).toString());
        registry.add("spring.cloud.aws.credentials.access-key",
            () -> localStack.getAccessKey());
        registry.add("spring.cloud.aws.credentials.secret-key",
            () -> localStack.getSecretKey());
        registry.add("spring.cloud.aws.region.static",
            () -> localStack.getRegion());
    }
}

// 사용
class MyIntegrationTest extends AbstractLocalStackTest {
    @Test
    void myTest() {
        // localStack 사용 가능
    }
}
```

## 정리

| 항목 | 내용 |
|------|------|
| LocalStack 버전 | 3.0 이상 권장 |
| Spring Cloud AWS | 3.3.0 (io.awspring.cloud) |
| 통합 테스트 | Testcontainers LocalStack 모듈 |
| 포트 | 4566 (단일 포트) |

LocalStack + Testcontainers 조합으로 AWS 의존 없이 통합 테스트를 작성할 수 있습니다. CI/CD 파이프라인에서도 동일한 테스트를 실행할 수 있어 개발 생산성이 크게 향상됩니다.
