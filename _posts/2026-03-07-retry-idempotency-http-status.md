---
title: "재시도해도 괜찮을까: HTTP 상태 코드와 멱등성으로 판단하는 Retry 전략"
date: 2026-03-07 00:00:00 +0900
categories: [Development, Backend]
tags: [retry, idempotency, http, resilience4j, spring-boot, circuit-breaker, kotlin]
---

서킷브레이커를 도입하면서 자연스럽게 Retry 설정도 붙이게 됩니다. Resilience4j를 쓰면 `@Retry` 어노테이션 하나로 재시도가 동작하지만, 막상 운영에서 문제가 생기면 "이 요청을 재시도해도 되는가"라는 질문에 확신이 없어지는 순간이 옵니다.

결제 API 호출이 타임아웃으로 실패했을 때 재시도했더니 중복 결제가 발생했던 경험이 있습니다. Circuit breaker HALF_OPEN 상태에서 재시도가 들어갔는데, 실제로는 첫 번째 요청이 이미 처리된 상태였습니다. Retry 설정이 잘못된 게 아니라 **재시도 대상을 잘못 판단한 것**이었습니다.

이 글에서는 두 가지 기준 — HTTP 상태 코드와 멱등성 — 으로 어떤 요청을 안전하게 재시도할 수 있는지 정리합니다.

## HTTP 상태 코드로 판단하기

재시도 여부를 가장 먼저 결정하는 기준은 서버가 돌려준 HTTP 상태 코드입니다.

**재시도해야 하는 코드**

| 상태 코드 | 이유 |
|-----------|------|
| `408 Request Timeout` | 서버가 요청을 처리하지 못한 일시적 상태 |
| `429 Too Many Requests` | 요청 제한. `Retry-After` 헤더 있으면 그 시간 이후 재시도 |
| `502 Bad Gateway` | 업스트림 서버가 잠깐 내려간 상태 |
| `503 Service Unavailable` | 서비스가 일시적으로 처리 불가 |
| `504 Gateway Timeout` | 업스트림 응답 지연. 단, 멱등성 확인 필수 |

**재시도하면 안 되는 코드**

| 상태 코드 | 이유 |
|-----------|------|
| `400 Bad Request` | 요청 자체가 잘못됨. 다시 보내도 400이 옵니다 |
| `401 Unauthorized` | 인증 문제. 재시도 전에 토큰 갱신이 필요 |
| `403 Forbidden` | 권한 없음. 재시도로 해결되지 않습니다 |
| `404 Not Found` | 리소스가 없음 |
| `422 Unprocessable Entity` | 비즈니스 로직 검증 실패 |

4xx 오류는 대부분 **클라이언트 쪽 문제**입니다. 재시도해봐야 같은 응답이 돌아오고, 이미 처리된 요청이라면 중복 처리 위험만 늘어납니다.

주의할 것은 `500 Internal Server Error`입니다. 서버 내부 오류인데, 재시도 여부를 상태 코드만으로 판단할 수 없습니다. 500은 요청이 처리됐을 수도, 처리 중에 터졌을 수도 있습니다. 멱등성 여부가 결정합니다.

## 멱등성으로 판단하기

멱등성(Idempotency)은 같은 요청을 여러 번 보내도 결과가 동일한 성질입니다. 수학의 `f(f(x)) = f(x)` 와 같은 개념입니다.

HTTP 메서드 기준으로 보면:

| 메서드 | 멱등성 | 안전성 | 비고 |
|--------|--------|--------|------|
| `GET` | O | O | 조회만 하므로 항상 안전 |
| `HEAD` | O | O | GET과 동일 |
| `PUT` | O | X | 덮어쓰기이므로 여러 번 보내도 결과 동일 |
| `DELETE` | O | X | 두 번째 DELETE는 404지만 상태는 동일 |
| `POST` | X | X | 호출마다 새 리소스 생성 가능 |
| `PATCH` | X (구현 의존) | X | 증분 변경이면 멱등하지 않을 수 있음 |

GET, PUT, DELETE는 멱등하므로 재시도가 안전합니다. 문제는 **POST와 PATCH**입니다.

`POST /orders`를 재시도하면 주문이 두 개 생성될 수 있습니다. `PATCH /accounts/{id}/balance` 에서 금액을 더하는 방식이면 재시도할 때마다 잔액이 바뀝니다.

## 두 기준의 교차점

상태 코드와 멱등성을 함께 보면 재시도 판단 기준이 명확해집니다.

| | 멱등한 요청 (GET, PUT, DELETE) | 멱등하지 않은 요청 (POST, PATCH) |
|---|---|---|
| **5xx, 408, 502~504** | 재시도 가능 | 멱등성 키 없으면 위험 |
| **429** | 백오프 후 재시도 | 백오프 후 재시도 (처리 안 됐을 가능성 높음) |
| **4xx (400, 401, 403...)** | 재시도 불가 | 재시도 불가 |
| **연결 자체 실패 (네트워크 오류)** | 재시도 가능 | 멱등성 키 없으면 위험 |

네트워크 오류는 상태 코드조차 없으므로 "서버가 요청을 받았는가"를 확인할 수 없습니다. 이 경우 멱등성이 없는 요청을 재시도하면 중복 처리로 이어질 수 있습니다.

## Resilience4j 설정

위 판단 기준을 Resilience4j에 적용하면 다음과 같습니다.

```yaml
resilience4j:
  retry:
    instances:
      external-api:
        max-attempts: 3
        wait-duration: 500ms
        exponential-backoff-multiplier: 2
        retry-on-result-predicate: com.example.retry.RetryOnResult
        retry-exceptions:
          - java.net.ConnectException
          - java.net.SocketTimeoutException
        ignore-exceptions:
          - com.example.exception.ClientException  # 4xx 계열 래핑
```

`RetryOnResult`에서 HTTP 상태 코드 기준을 적용합니다:

```kotlin
class RetryOnResult : Predicate<HttpResponse<*>> {
    override fun test(response: HttpResponse<*>): Boolean {
        val status = response.statusCode()
        return status in setOf(408, 429, 500, 502, 503, 504)
    }
}
```

단순히 5xx 전체를 재시도 대상에 넣지 않고, 재시도 의미가 있는 코드만 명시적으로 지정합니다. `500`을 넣을 때는 해당 API가 멱등한지 확인이 선행되어야 합니다.

서킷브레이커와 함께 쓸 때는 Retry가 Circuit Breaker 안에서 동작하도록 순서를 잡습니다:

```kotlin
@Service
class OrderService(
    private val paymentClient: PaymentClient,
) {
    @Retry(name = "payment-retry")
    @CircuitBreaker(name = "payment-cb", fallbackMethod = "paymentFallback")
    fun processPayment(request: PaymentRequest): PaymentResponse {
        return paymentClient.pay(request)
    }

    fun paymentFallback(request: PaymentRequest, ex: Exception): PaymentResponse {
        // fallback 처리
        throw PaymentUnavailableException("결제 서비스 일시 불가", ex)
    }
}
```

Resilience4j에서 `@Retry` + `@CircuitBreaker`를 함께 사용할 때 선언 순서가 적용 순서에 영향을 줍니다. `@Retry`가 먼저 선언돼 있으면 Retry가 바깥에서 감싸게 됩니다. Retry가 모두 소진된 뒤 Circuit Breaker로 실패가 기록되는 흐름입니다. `spring.cloud.circuitbreaker.resilience4j.enabled` 설정이나 `@Order`로도 제어할 수 있지만, 가장 명확한 방법은 중첩을 직접 코드로 표현하는 것입니다.

## POST를 재시도해야 할 때: 멱등성 키

비즈니스상 POST를 재시도해야 하는 경우가 있습니다. 결제, 주문 생성 같은 경우입니다. 이때는 **멱등성 키(Idempotency Key)** 를 사용합니다.

클라이언트가 요청마다 고유한 키를 생성해서 헤더에 담아 보내고, 서버는 같은 키의 요청이 들어오면 이전 결과를 그대로 반환합니다.

```kotlin
// 클라이언트: 멱등성 키 생성 및 전달
fun createOrder(request: OrderRequest): OrderResponse {
    val idempotencyKey = UUID.randomUUID().toString()
    return orderClient.createOrder(
        request = request,
        headers = mapOf("Idempotency-Key" to idempotencyKey)
    )
}
```

```kotlin
// 서버: 멱등성 키 처리
@PostMapping("/orders")
fun createOrder(
    @RequestHeader("Idempotency-Key") idempotencyKey: String,
    @RequestBody request: OrderRequest,
): OrderResponse {
    // 이미 처리된 요청이면 캐시된 결과 반환
    idempotencyStore.get(idempotencyKey)?.let { return it }

    val response = orderService.create(request)
    idempotencyStore.save(idempotencyKey, response, ttl = Duration.ofHours(24))
    return response
}
```

멱등성 키는 클라이언트가 재시도 전에 생성해두고 매번 같은 키를 보내야 합니다. 재시도할 때마다 새 키를 만들면 의미가 없습니다.

Stripe, Toss Payments 같은 결제 API가 이 방식을 표준으로 채택하고 있습니다. 재시도 로직보다 멱등성 키 설계가 선행되어야 하는 이유이기도 합니다.

## 정리하며

서킷브레이커와 Retry를 처음 적용할 때는 "실패하면 재시도"라는 단순한 그림으로 시작하기 쉽습니다. 하지만 운영에서 중복 처리 이슈를 한 번 겪고 나면, 재시도 대상을 판단하는 일이 설정보다 훨씬 중요하다는 걸 체감하게 됩니다.

재시도 전에 확인해야 할 것은 두 가지입니다. 이 상태 코드가 재시도 의미가 있는 오류인가, 그리고 이 요청이 여러 번 실행돼도 안전한가. 이 두 질문에 모두 "예"라고 답할 수 있을 때 재시도를 붙이는 게 맞습니다.
