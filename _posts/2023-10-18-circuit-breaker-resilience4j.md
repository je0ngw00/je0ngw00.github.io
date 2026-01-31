---
title: "Resilience4j Circuit Breaker 적용기: 외부 API 장애 대응"
date: 2023-10-18 10:00:00 +0900
categories: [Development, Spring]
tags: [resilience4j, circuit-breaker, spring-boot, fault-tolerance, java]
---

> **📌 2026-01 업데이트**: Resilience4j 최신 버전 2.3.0 기준으로 업데이트했습니다.

## 개요

회원 도메인 장애로 쿠폰 서비스까지 영향을 받는 일이 발생했습니다. 외부 연동 장애에 유연하게 대응하고 시스템 회복력을 높이기 위해 Circuit Breaker 패턴을 도입했습니다.

## Circuit Breaker 패턴이란?

전기 회로의 차단기처럼, 장애가 발생한 서비스로의 요청을 차단하여 시스템 전체 장애를 방지하는 패턴입니다.

### 상태 전이

```
        실패율 임계치 초과
    ┌────────────────────┐
    │                    ▼
┌───────┐           ┌───────┐
│ CLOSED │           │ OPEN  │
└───────┘           └───────┘
    ▲                    │
    │    대기 시간 경과   │
    │   ┌───────────┐   │
    └───│ HALF_OPEN │◄──┘
        └───────────┘
```

| 상태 | 설명 |
|------|------|
| **CLOSED** | 정상 상태. 모든 요청 통과 |
| **OPEN** | 장애 상태. 모든 요청 즉시 실패 (Fallback 실행) |
| **HALF_OPEN** | 테스트 상태. 일부 요청만 통과시켜 복구 확인 |

## Hystrix vs Resilience4j

Hystrix는 2018년 이후 유지보수 모드로 전환되어 더 이상 새 기능이 추가되지 않습니다. Netflix 공식 GitHub에서도 Resilience4j를 권장합니다.

| 항목 | Hystrix | Resilience4j |
|------|---------|--------------|
| 상태 | 유지보수 모드 | 활발한 개발 |
| Java 버전 | Java 8 | Java 17+ 지원 |
| Spring Boot 3 | 미지원 | 지원 |
| 의존성 | 무거움 | 가벼움 |
| 기능 | Circuit Breaker 중심 | CB + Retry + RateLimiter + Bulkhead |

## 의존성 추가

```xml
<!-- Spring Boot 3.x -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.3.0</version>
</dependency>

<!-- Spring Boot 2.x -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot2</artifactId>
    <version>2.3.0</version>
</dependency>
```

## 설정

### application.yml

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        # Sliding Window 설정
        slidingWindowType: COUNT_BASED        # 또는 TIME_BASED
        slidingWindowSize: 10                 # 최근 10개 요청 기준
        minimumNumberOfCalls: 5               # 최소 5개 요청 후 계산 시작

        # 실패율 임계치
        failureRateThreshold: 50              # 50% 이상 실패 시 OPEN

        # OPEN 상태 유지 시간
        waitDurationInOpenState: 30s          # 30초 후 HALF_OPEN

        # HALF_OPEN 상태 설정
        permittedNumberOfCallsInHalfOpenState: 3  # 3개 요청으로 테스트

        # 예외 설정
        recordExceptions:
          - java.io.IOException
          - java.net.SocketTimeoutException
          - org.springframework.web.client.HttpServerErrorException
        ignoreExceptions:
          - org.springframework.web.client.HttpClientErrorException

    instances:
      memberApi:
        baseConfig: default
        waitDurationInOpenState: 60s          # 회원 API는 60초
      paymentApi:
        baseConfig: default
        failureRateThreshold: 30              # 결제는 더 민감하게
```

### 설정 값 설명

| 설정 | 설명 |
|------|------|
| `slidingWindowSize` | 실패율 계산에 사용할 요청 수 |
| `failureRateThreshold` | OPEN 전환 실패율 (%) |
| `waitDurationInOpenState` | OPEN 상태 유지 시간 |
| `recordExceptions` | 실패로 기록할 예외 |
| `ignoreExceptions` | 무시할 예외 (4xx 클라이언트 오류 등) |

## 구현

### 기본 사용법

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class MemberApiClient {

    private final RestTemplate restTemplate;

    @CircuitBreaker(name = "memberApi", fallbackMethod = "getMemberFallback")
    public MemberResponse getMember(Long memberId) {
        String url = "http://member-service/api/members/" + memberId;
        return restTemplate.getForObject(url, MemberResponse.class);
    }

    // Fallback 메서드: 원본과 동일한 파라미터 + Exception
    private MemberResponse getMemberFallback(Long memberId, Exception e) {
        log.warn("Circuit breaker fallback for memberId={}, reason={}",
                 memberId, e.getMessage());

        // 캐시된 데이터 반환 또는 기본값
        return MemberResponse.builder()
            .id(memberId)
            .name("Unknown")
            .status(MemberStatus.UNKNOWN)
            .build();
    }
}
```

### Retry와 함께 사용

```java
@Service
@RequiredArgsConstructor
public class PaymentClient {

    private final RestTemplate restTemplate;

    @Retry(name = "paymentApi", fallbackMethod = "payFallback")
    @CircuitBreaker(name = "paymentApi", fallbackMethod = "payFallback")
    public PaymentResponse pay(PaymentRequest request) {
        return restTemplate.postForObject(
            "http://payment-service/api/payments",
            request,
            PaymentResponse.class
        );
    }

    private PaymentResponse payFallback(PaymentRequest request, Exception e) {
        log.error("Payment failed: orderId={}", request.getOrderId(), e);
        throw new PaymentFailedException("결제 서비스 일시 장애", e);
    }
}
```

> **주의**: `@Retry`가 `@CircuitBreaker`보다 먼저 적용됩니다. Retry가 모두 실패해야 Circuit Breaker에 실패로 기록됩니다.

### Retry 설정

```yaml
resilience4j:
  retry:
    configs:
      default:
        maxAttempts: 3                        # 최대 3번 시도
        waitDuration: 1s                      # 재시도 간격
        retryExceptions:
          - java.io.IOException
        ignoreExceptions:
          - org.springframework.web.client.HttpClientErrorException

    instances:
      paymentApi:
        baseConfig: default
        maxAttempts: 2                        # 결제는 2번만
```

## 프로그래매틱 방식

어노테이션 대신 직접 제어할 수도 있습니다.

```java
@Service
@RequiredArgsConstructor
public class MemberService {

    private final CircuitBreakerRegistry circuitBreakerRegistry;
    private final MemberApiClient memberApiClient;

    public MemberResponse getMemberWithCircuitBreaker(Long memberId) {
        CircuitBreaker circuitBreaker = circuitBreakerRegistry
            .circuitBreaker("memberApi");

        return circuitBreaker.executeSupplier(() ->
            memberApiClient.getMember(memberId)
        );
    }

    // 상태 확인
    public CircuitBreaker.State getCircuitBreakerState() {
        return circuitBreakerRegistry
            .circuitBreaker("memberApi")
            .getState();
    }
}
```

## 모니터링

### Actuator 연동

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,circuitbreakers,circuitbreakerevents
  health:
    circuitbreakers:
      enabled: true
```

### 상태 조회

```bash
# Circuit Breaker 상태 조회
curl http://localhost:8080/actuator/circuitbreakers

# 응답 예시
{
  "circuitBreakers": {
    "memberApi": {
      "state": "CLOSED",
      "failureRate": "0.0%",
      "slowCallRate": "0.0%",
      "numberOfBufferedCalls": 10,
      "numberOfFailedCalls": 0
    }
  }
}
```

### 이벤트 리스너

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class CircuitBreakerEventListener {

    private final CircuitBreakerRegistry registry;

    @PostConstruct
    public void registerEventListeners() {
        registry.circuitBreaker("memberApi")
            .getEventPublisher()
            .onStateTransition(event -> {
                log.warn("Circuit Breaker 상태 변경: {} -> {}",
                    event.getStateTransition().getFromState(),
                    event.getStateTransition().getToState());

                // Slack 알림 등
                if (event.getStateTransition().getToState() == CircuitBreaker.State.OPEN) {
                    notifySlack("memberApi Circuit Breaker OPEN!");
                }
            })
            .onError(event -> {
                log.error("Circuit Breaker 에러: {}", event.getThrowable().getMessage());
            });
    }
}
```

## Fallback 전략

### 1. 캐시된 데이터 반환

```java
private MemberResponse getMemberFallback(Long memberId, Exception e) {
    // Redis 캐시에서 조회
    return memberCacheService.getCachedMember(memberId)
        .orElse(MemberResponse.unknown(memberId));
}
```

### 2. 기본값 반환

```java
private List<CouponResponse> getCouponsFallback(Long userId, Exception e) {
    return Collections.emptyList();  // 빈 목록 반환
}
```

### 3. 예외 전파 (필수 기능인 경우)

```java
private PaymentResponse payFallback(PaymentRequest request, Exception e) {
    throw new ServiceUnavailableException("결제 서비스 일시 장애");
}
```

## 테스트

```java
@SpringBootTest
class CircuitBreakerTest {

    @Autowired
    private CircuitBreakerRegistry registry;

    @Autowired
    private MemberApiClient memberApiClient;

    @Test
    void circuitBreaker가_실패율_초과시_OPEN_상태가_된다() {
        CircuitBreaker circuitBreaker = registry.circuitBreaker("memberApi");

        // 5번 연속 실패 시뮬레이션
        for (int i = 0; i < 5; i++) {
            try {
                circuitBreaker.executeSupplier(() -> {
                    throw new RuntimeException("테스트 실패");
                });
            } catch (Exception ignored) {}
        }

        assertThat(circuitBreaker.getState())
            .isEqualTo(CircuitBreaker.State.OPEN);
    }
}
```

## 정리

| 설정 | 권장값 | 설명 |
|------|--------|------|
| `slidingWindowSize` | 10-20 | 너무 작으면 민감, 크면 둔감 |
| `failureRateThreshold` | 50% | 서비스 중요도에 따라 조절 |
| `waitDurationInOpenState` | 30-60초 | 복구 시간 고려 |
| `minimumNumberOfCalls` | 5-10 | 초기 요청은 제외 |

Circuit Breaker는 장애 전파를 막는 첫 번째 방어선입니다. Fallback 전략을 잘 설계하면 외부 서비스 장애에도 사용자 경험을 유지할 수 있습니다.

---

**다음으로 읽으면 좋은 글:**
- [트랜잭션 내 외부 호출 분리로 커넥션 풀 병목 해결하기](/posts/transaction-external-call/) - 외부 API 호출 시 트랜잭션 관리
- [AWS SNS + SQS로 이벤트 기반 아키텍처 구축하기](/posts/aws-sqs-sns-intro/) - 비동기 메시징으로 시스템 결합도 낮추기
