---
title: "상속보다 컴포지션: 결제 시스템 리팩토링 사례"
date: 2023-06-04 10:00:00 +0900
categories: [Development, Design Pattern]
tags: [java, design-pattern, composition, inheritance, refactoring, effective-java]
---

## 개요

이펙티브 자바의 첫 번째 원칙 중 하나가 "상속보다는 컴포지션을 사용하라(Prefer Composition Over Inheritance)"입니다. 결제 시스템을 개발하면서 이 원칙을 직접 적용한 경험을 공유합니다.

## 왜 컴포지션인가?

### 상속의 문제점

- **강한 결합**: 부모 클래스 변경이 모든 자식 클래스에 영향
- **계층 구조의 경직성**: 복잡한 상속 체인은 수정과 확장이 어려움
- **불필요한 기능 상속**: 필요 없는 메서드도 강제로 구현해야 함

### 컴포지션의 장점

- **느슨한 결합**: 객체 간 독립적인 변경 가능
- **유연한 조합**: 필요한 기능만 조합해서 사용
- **인터페이스 분리**: 명확한 책임 정의

## 실제 사례: 결제 시스템 리팩토링

### 초기 설계 (템플릿 메서드 패턴)

결제 수단별로 비슷하지만 조금씩 다른 로직이 있어서 템플릿 메서드 패턴을 적용했습니다.

```java
@RequiredArgsConstructor
public abstract class PayTemplate {

    private final PgClient pgClient;
    private final PaymentValidator paymentValidator;
    private final PaymentCommandService paymentCommandService;

    public final void pay(PayRequest payRequest) {
        paymentValidator.validate(payRequest);
        PgResponse pgResponse = doPay(payRequest, pgClient);
        createPayment(payRequest, pgResponse);
    }

    protected abstract PgResponse doPay(PayRequest payRequest, PgClient pgClient);

    private void createPayment(PayRequest payRequest, PgResponse pgResponse) {
        paymentCommandService.createPayment(payRequest, pgResponse);
    }
}
```

```java
public class CreditCardPay extends PayTemplate {

    @Override
    protected PgResponse doPay(PayRequest payRequest, PgClient pgClient) {
        return pgClient.pay(payRequest);
    }
}
```

### 문제 발생

가상계좌 결제가 추가되면서 **현금영수증 발급** 기능이 필요해졌습니다.

```java
public abstract class PayTemplate {

    public final void pay(PayRequest payRequest) {
        paymentValidator.validate(payRequest);
        PgResponse pgResponse = doPay(payRequest, pgClient);
        CashReceipt cashReceipt = issueCashReceipt(payRequest, pgClient);  // 추가
        createPayment(payRequest, pgResponse);
    }

    protected abstract PgResponse doPay(PayRequest payRequest, PgClient pgClient);
    protected abstract CashReceipt issueCashReceipt(PayRequest payRequest, PgClient pgClient);  // 추가
}
```

신용카드에는 현금영수증이 필요 없는데, 추상 메서드라서 구현해야 합니다.

```java
public class CreditCardPay extends PayTemplate {

    @Override
    protected PgResponse doPay(PayRequest payRequest, PgClient pgClient) {
        return pgClient.pay(payRequest);
    }

    @Override
    protected CashReceipt issueCashReceipt(PayRequest payRequest, PgClient pgClient) {
        return null;  // 의미 없는 구현
    }
}
```

**null을 반환하는 빈 구현**이 생겼습니다. 결제 수단이 늘어날수록 이런 코드가 더 많아질 것입니다.

### 컴포지션으로 리팩토링

#### 1. 인터페이스 정의

```java
public interface PaymentProcessor {
    PgResponse process(PayRequest payRequest, PgClient pgClient);
}
```

#### 2. 결제 흐름 관리자

```java
@RequiredArgsConstructor
public class PaymentManager {

    private final PgClient pgClient;
    private final PaymentProcessor paymentProcessor;
    private final PaymentValidator paymentValidator;
    private final PaymentCommandService paymentCommandService;

    public void pay(PayRequest payRequest) {
        paymentValidator.validate(payRequest);
        PgResponse pgResponse = paymentProcessor.process(payRequest, pgClient);
        createPayment(payRequest, pgResponse);
    }

    private void createPayment(PayRequest payRequest, PgResponse pgResponse) {
        paymentCommandService.createPayment(payRequest, pgResponse);
    }
}
```

#### 3. 결제 수단별 구현

```java
// 신용카드: 단순 결제만
public class CreditCardProcessor implements PaymentProcessor {

    @Override
    public PgResponse process(PayRequest payRequest, PgClient pgClient) {
        return pgClient.pay(payRequest);
    }
}

// 가상계좌: 결제 + 현금영수증
@RequiredArgsConstructor
public class VirtualAccountProcessor implements PaymentProcessor {

    private final CashReceiptService cashReceiptService;

    @Override
    public PgResponse process(PayRequest payRequest, PgClient pgClient) {
        PgResponse pgResponse = pgClient.pay(payRequest);
        cashReceiptService.issue(payRequest, pgClient);
        return pgResponse;
    }
}
```

#### 4. 사용 예시

```java
@Configuration
public class PaymentConfig {

    @Bean
    public PaymentManager creditCardPaymentManager(
            PgClient pgClient,
            PaymentValidator validator,
            PaymentCommandService commandService) {
        return new PaymentManager(
            pgClient,
            new CreditCardProcessor(),
            validator,
            commandService
        );
    }

    @Bean
    public PaymentManager virtualAccountPaymentManager(
            PgClient pgClient,
            PaymentValidator validator,
            PaymentCommandService commandService,
            CashReceiptService cashReceiptService) {
        return new PaymentManager(
            pgClient,
            new VirtualAccountProcessor(cashReceiptService),
            validator,
            commandService
        );
    }
}
```

### 리팩토링 결과

| 항목 | Before (상속) | After (컴포지션) |
|------|--------------|-----------------|
| 새 결제 수단 추가 | 추상 클래스 수정 필요 | 새 Processor만 구현 |
| 불필요한 메서드 | null 반환 구현 강제 | 필요한 것만 구현 |
| 테스트 | 부모 클래스 의존 | 독립적 단위 테스트 |
| 결합도 | 높음 | 낮음 |

## 그래도 상속이 필요한 경우

상속이 적절한 경우도 있습니다:

1. **진정한 IS-A 관계**: "고양이는 동물이다"처럼 명확한 계층 구조
2. **프레임워크 확장**: Spring의 `AbstractController` 상속 등
3. **코드 재사용이 목적이 아닌 다형성이 목적인 경우**

하지만 대부분의 경우 "상속으로 해결할 수 있는 문제는 컴포지션으로도 해결 가능하고, 더 유연하다"는 점을 기억하면 좋습니다.

## 정리

컴포지션을 사용하면:
- 기존 코드 변경 없이 새 기능 추가 가능
- 객체 간 결합도가 낮아져 유지보수성 향상
- 테스트가 쉬워짐

상속이 필요한 것 같을 때, 먼저 "이걸 컴포지션으로 풀 수는 없을까?" 고민해보는 습관을 들이면 좋습니다.
