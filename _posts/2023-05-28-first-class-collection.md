---
title: "일급 컬렉션으로 컬렉션을 객체답게 다루기"
date: 2023-05-28 10:00:00 +0900
categories: [Development, Design Pattern]
tags: [java, design-pattern, first-class-collection, clean-code]
---

## 개요

업무를 하다 보면 종종 컬렉션 객체를 사용할 일이 나옵니다.

```java
List<Order> orders = orderRepository.findByUserId(userId);
List<Long> orderIds = orders.stream().map(Order::getOrderId).toList();
int totalPrice = orders.stream().mapToInt(Order::getPrice).sum();
```

이런 로직이 여러 곳에 중복되고, 컬렉션을 다루는 로직이 서비스 레이어 여기저기에 흩어져 있는 경우가 많았습니다.

그럴 때면 저는 항상 해당 객체를 한 번 더 감싸 일급 컬렉션을 만들어 객체가 일을 할 수 있도록 리팩토링을 했습니다.

```java
public class Orders {
    private final List<Order> orders;

    public Orders(List<Order> orders) {
        this.orders = orders;
    }

    public List<Long> getOrderIds() {
        return this.orders.stream().map(Order::getOrderId).toList();
    }

    public int getTotalPrice() {
        return this.orders.stream().mapToInt(Order::getPrice).sum();
    }
}
```

```java
Orders orders = new Orders(orderRepository.findByUserId(userId));
List<Long> orderIds = orders.getOrderIds();
int totalPrice = orders.getTotalPrice();
```

이러한 컬렉션을 한 번 감싸고, **그 외 다른 멤버 변수가 없는 상태**를 일급 컬렉션이라 합니다.

## 일급 컬렉션이란

일급 컬렉션(First-Class Collection)은 객체 지향 프로그래밍에서 컬렉션을 일급 객체(First-Class Object)로 취급하는 개념입니다. 소트웍스 앤솔로지의 객체지향 생활체조 규칙 8번에서 유래했습니다.

핵심 규칙은 간단합니다:
- 컬렉션을 포함한 클래스는 **반드시 다른 멤버 변수가 없어야** 합니다.
- 컬렉션 하나만 가지고, 그 컬렉션에 대한 행위만 정의합니다.

## 일급 컬렉션을 왜 사용해야 할까?

### 1. 비즈니스 로직의 응집

컬렉션에 대한 로직이 흩어지지 않고 한 곳에 모입니다.

**Before**: 서비스 레이어에 로직이 분산됨
```java
// OrderService.java
public int calculateTotalPrice(List<Order> orders) {
    return orders.stream().mapToInt(Order::getPrice).sum();
}

// OrderController.java
public int getOrderTotal(List<Order> orders) {
    return orders.stream().mapToInt(Order::getPrice).sum(); // 중복!
}
```

**After**: 일급 컬렉션에 로직이 응집됨
```java
public class Orders {
    private final List<Order> orders;

    public Orders(List<Order> orders) {
        this.orders = orders;
    }

    public int getTotalPrice() {
        return this.orders.stream().mapToInt(Order::getPrice).sum();
    }

    public Orders filterByStatus(OrderStatus status) {
        List<Order> filtered = this.orders.stream()
            .filter(order -> order.getStatus() == status)
            .toList();
        return new Orders(filtered);
    }

    public boolean hasOrderOver(int amount) {
        return this.orders.stream().anyMatch(order -> order.getPrice() > amount);
    }
}
```

### 2. 유효성 검사와 제약 조건

생성 시점에 검증 로직을 넣어 항상 유효한 상태를 보장할 수 있습니다.

```java
public class Orders {
    private static final int MAX_ORDER_COUNT = 10;
    private final List<Order> orders;

    public Orders(List<Order> orders) {
        validateSize(orders);
        validateNoDuplicate(orders);
        this.orders = new ArrayList<>(orders); // 방어적 복사
    }

    private void validateSize(List<Order> orders) {
        if (orders.size() > MAX_ORDER_COUNT) {
            throw new IllegalArgumentException(
                "주문은 최대 " + MAX_ORDER_COUNT + "개까지 가능합니다."
            );
        }
    }

    private void validateNoDuplicate(List<Order> orders) {
        long distinctCount = orders.stream()
            .map(Order::getProductId)
            .distinct()
            .count();
        if (distinctCount != orders.size()) {
            throw new IllegalArgumentException("중복된 상품이 있습니다.");
        }
    }
}
```

### 3. 불변성 보장

외부에서 컬렉션을 수정할 수 없도록 보장할 수 있습니다.

```java
public class Orders {
    private final List<Order> orders;

    public Orders(List<Order> orders) {
        // 방어적 복사 후 불변 리스트로 변환
        this.orders = Collections.unmodifiableList(new ArrayList<>(orders));
    }

    // getter에서도 불변 리스트 반환
    public List<Order> getOrders() {
        return this.orders; // 이미 unmodifiable이므로 안전
    }
}
```

**주의**: `Collections.unmodifiableList(orders)`만 하면 원본 리스트 수정 시 영향을 받습니다. 반드시 `new ArrayList<>(orders)`로 복사 후 감싸야 합니다.

### 4. 의미 있는 이름 부여

`List<Order>`보다 `Orders`가, `List<Long>`보다 `LottoNumbers`가 더 명확한 의미를 전달합니다.

```java
// 무슨 List인지 파악하기 어려움
public void process(List<Long> numbers) { ... }

// 의도가 명확함
public void process(LottoNumbers numbers) { ... }
```

## 실무 활용 예시

### 장바구니 구현

```java
public class CartItems {
    private final List<CartItem> items;

    public CartItems(List<CartItem> items) {
        this.items = new ArrayList<>(items);
    }

    public int getTotalPrice() {
        return items.stream()
            .mapToInt(item -> item.getPrice() * item.getQuantity())
            .sum();
    }

    public int getTotalQuantity() {
        return items.stream()
            .mapToInt(CartItem::getQuantity)
            .sum();
    }

    public boolean isEmpty() {
        return items.isEmpty();
    }

    public CartItems merge(CartItems other) {
        List<CartItem> merged = new ArrayList<>(this.items);
        merged.addAll(other.items);
        return new CartItems(merged);
    }
}
```

## 주의사항

일급 컬렉션이 항상 정답은 아닙니다.

- **단순한 경우**: 컬렉션에 대한 로직이 거의 없다면 오버엔지니어링이 될 수 있습니다.
- **클래스 수 증가**: 모든 컬렉션을 감싸면 클래스 파일이 많아집니다.
- **적용 기준**: 컬렉션에 대한 로직이 2곳 이상에서 중복되거나, 비즈니스 규칙이 있을 때 적용을 고려하면 좋습니다.

## 정리

일급 컬렉션은 다음과 같은 장점을 제공합니다:

| 장점 | 설명 |
|------|------|
| 비즈니스 로직 응집 | 컬렉션 관련 로직이 한 곳에 모임 |
| 유효성 검사 | 생성 시점에 불변식 보장 |
| 불변성 | 외부 수정 방지 |
| 명확한 의미 | 타입 자체가 문서 역할 |

컬렉션을 단순히 데이터 저장소로 쓰지 말고, 행위를 가진 객체로 만들어 보시기 바랍니다.

---

**다음으로 읽으면 좋은 글:**
- [정적 팩토리 메서드: from, of, getInstance 네이밍의 의미](/posts/static-factory-method/) - 객체 생성을 캡슐화하는 또 다른 방법
- [상속보다 컴포지션: 결제 시스템 리팩토링 사례](/posts/composition-over-inheritance/) - 객체 합성으로 유연한 설계 만들기
