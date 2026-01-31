---
title: "정적 팩토리 메서드: from, of, getInstance 네이밍의 의미"
date: 2023-05-29 10:00:00 +0900
categories: [Development, Design Pattern]
tags: [java, design-pattern, static-factory-method, effective-java]
---

## 개요

업무를 하게 되면 여러 객체를 생성하기 위한 Factory라는 클래스를 만들어 정적 메서드로 처리하는 경우가 많았습니다.

- requestDto → dto
- dto → entity
- dto → clientRequest

... 등등 아무 생각 없이 작성하던 정적 팩토리 메서드(Static Factory Method)에 대해 알아보겠습니다.

## 팩토리란 무엇인가?

디자인 패턴에서 말하는 팩토리는 `객체 생성을 캡슐화`하는 것입니다.

## 정적 팩토리 메서드란?

**생성자 대신 정적 메서드로 객체를 생성하는 방식**입니다. GoF 디자인 패턴의 팩토리 메서드와는 다른 개념이며, 이펙티브 자바 아이템 1에서 소개된 기법입니다.

우리가 자주 쓰는 JDK 메서드들이 이미 정적 팩토리 메서드입니다:

```java
// Boolean
Boolean bool = Boolean.valueOf(true);  // new Boolean(true) 대신

// List
List<String> list = List.of("a", "b", "c");

// Optional
Optional<String> opt = Optional.of("value");
Optional<String> empty = Optional.empty();

// Collections
List<String> emptyList = Collections.emptyList();
```

## 생성자와 어떤 차이가 있을까?

"이펙티브 자바"에서 가장 첫 번째로 나오는 아이템이 바로 **"생성자 대신 정적 팩토리 메서드를 고려하라"**입니다.

### 1. 이름을 가질 수 있다

생성자는 클래스 이름과 동일해야 하지만, 정적 팩토리 메서드는 의미 있는 이름을 가질 수 있습니다.

```java
// 생성자: 무슨 주문인지 알기 어려움
Order order1 = new Order(request);
Order order2 = new Order(regularRequest);

// 정적 팩토리: 의도가 명확함
Order order1 = Order.from(request);
Order order2 = Order.createRegularOrder(regularRequest);
Order order3 = Order.createGiftOrder(giftRequest);
```

### 2. 호출할 때마다 새 인스턴스를 만들지 않아도 된다

불변 객체나 자주 사용되는 인스턴스를 캐싱해서 재활용할 수 있습니다.

```java
public class Connection {
    private static final Connection INSTANCE = new Connection();

    private Connection() {}

    // 매번 같은 인스턴스 반환 (싱글톤)
    public static Connection getInstance() {
        return INSTANCE;
    }
}

// JDK 예시: Boolean.valueOf()는 캐싱된 객체 반환
Boolean.valueOf(true);   // 항상 같은 Boolean.TRUE 반환
Boolean.valueOf(false);  // 항상 같은 Boolean.FALSE 반환
```

### 3. 반환 타입의 하위 타입 객체를 반환할 수 있다

인터페이스나 상위 클래스를 반환 타입으로 선언하고, 실제로는 하위 타입을 반환할 수 있습니다.

```java
public interface Discount {
    int apply(int price);

    // 조건에 따라 다른 구현체 반환
    static Discount of(DiscountType type, int value) {
        return switch (type) {
            case PERCENT -> new PercentDiscount(value);
            case FIXED -> new FixedDiscount(value);
            case NONE -> price -> price;  // 람다로 즉석 구현
        };
    }
}

// 사용: 구현체를 몰라도 됨
Discount discount = Discount.of(DiscountType.PERCENT, 10);
```

### 4. 입력 매개변수에 따라 다른 클래스의 객체를 반환할 수 있다

```java
// EnumSet: 원소 수에 따라 다른 구현체 반환
EnumSet<Color> colors = EnumSet.of(Color.RED, Color.BLUE);
// 64개 이하면 RegularEnumSet, 초과하면 JumboEnumSet 반환
// 클라이언트는 어떤 구현체인지 알 필요 없음
```

## 정적 팩토리 예시

제가 진행했던 쿠폰 시스템의 정책 생성 부분입니다.

```java
public class CouponPolicyFactory {

    private CouponPolicyFactory() {
    }

    public static CouponPolicyEntity of(CouponPolicyCreateRequest request,
                                        Supplier<String> numberSupplier) {
        return CouponPolicyEntity.createBuilder()
                .policyNumber(numberSupplier.get())
                .name(request.getName())
                .discountType(request.getDiscountType())
                .discountValue(request.getDiscountValue())
                .build();
    }
}
```

1. 생성자를 private으로 정의하여 인스턴스화를 막았습니다.
2. `of` 네이밍 컨벤션을 따랐습니다 (여러 매개변수).
3. 복잡한 생성 로직을 캡슐화했습니다.

## 단점

정적 팩토리 메서드에도 단점이 있습니다:

### 1. 상속이 어렵다

정적 팩토리 메서드만 제공하면 생성자가 private이므로 상속이 불가능합니다. 다만 이는 상속보다 컴포지션을 유도하는 장점이 될 수도 있습니다.

### 2. 찾기 어렵다

생성자는 API 문서에서 명확히 드러나지만, 정적 팩토리 메서드는 다른 정적 메서드와 구분이 어렵습니다. 따라서 **네이밍 컨벤션을 지키는 것이 중요**합니다.

## 네이밍 컨벤션

| 메서드명 | 설명 | 예시 |
|---------|------|------|
| **from** | 하나의 매개변수로 인스턴스 생성 | `Date.from(instant)` |
| **of** | 여러 매개변수로 인스턴스 생성 | `List.of("a", "b")` |
| **valueOf** | from, of의 자세한 버전 | `Boolean.valueOf(true)` |
| **getInstance / instance** | 인스턴스 반환 (동일 인스턴스 보장 X) | `Calendar.getInstance()` |
| **create / newInstance** | 매번 새로운 인스턴스 생성 보장 | `Array.newInstance(...)` |
| **get[Type]** | 다른 타입의 인스턴스 반환 | `Files.getFileStore(path)` |
| **new[Type]** | 다른 타입의 새 인스턴스 반환 | `Files.newBufferedReader(path)` |

## 정리

정적 팩토리 메서드는 단순히 생성자의 역할을 대신하는 것뿐만 아니라, 가독성 좋은 코드를 작성하고 유연한 객체 생성을 가능하게 합니다.

객체 간 형 변환이 필요하거나, 생성 로직이 복잡하거나, 캐싱이 필요한 경우라면 정적 팩토리 메서드를 고려해 보시기 바랍니다.

---

**다음으로 읽으면 좋은 글:**
- [일급 컬렉션으로 컬렉션을 객체답게 다루기](/posts/first-class-collection/) - 컬렉션을 감싸 비즈니스 로직을 응집하는 방법
- [상속보다 컴포지션: 결제 시스템 리팩토링 사례](/posts/composition-over-inheritance/) - 유연한 객체 조합 패턴
