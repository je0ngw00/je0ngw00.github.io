---
title: "Querydsl에서 DTO 조회하기: Projections vs @QueryProjection"
date: 2023-05-30 10:00:00 +0900
categories: [Development, JPA]
tags: [querydsl, jpa, dto, projection, java]
---

## 개요

조회 시 Entity를 직접 조회하면 여러 문제가 발생할 수 있습니다:

- 영속성 컨텍스트의 1차 캐시에 불필요하게 적재
- 모든 컬럼 조회로 인한 성능 저하
- 지연 로딩 연관관계의 의도치 않은 로딩

따라서 조회 전용 로직에서는 Entity 대신 **DTO를 직접 조회**하는 것이 좋습니다.

## DTO 조회의 장점

### 필요한 데이터만 선택

```sql
-- Entity 조회: 모든 컬럼
SELECT id, name, price, description, created_at, updated_at, ... FROM product

-- DTO 조회: 필요한 것만
SELECT id, name, price FROM product
```

### 레이어 간 의존성 분리

Entity 변경이 API 스펙에 영향을 주지 않습니다.

### 보안

Entity의 민감한 정보(비밀번호 해시, 내부 ID 등)를 노출하지 않습니다.

## Querydsl DTO 조회 방법

### 1. Projections.constructor()

생성자를 호출하여 DTO를 생성합니다.

```java
List<ProductDto> products = queryFactory
    .select(Projections.constructor(ProductDto.class,
        product.id,
        product.name,
        product.price
    ))
    .from(product)
    .fetch();
```

| 장점 | 단점 |
|-----|-----|
| Querydsl 의존성 없음 | 필드 순서, 타입이 정확히 일치해야 함 |
| 코드가 간결함 | **런타임에야 오류 발견** |

### 2. Projections.fields()

리플렉션으로 필드에 직접 값을 주입합니다.

```java
List<ProductDto> products = queryFactory
    .select(Projections.fields(ProductDto.class,
        product.id,
        product.name,
        product.price
    ))
    .from(product)
    .fetch();
```

| 장점 | 단점 |
|-----|-----|
| 필드 순서 상관없음 | 필드명이 일치해야 함 |
| setter 불필요 | 기본 생성자 필요, **런타임 오류** |

필드명이 다를 경우 `as()`로 별칭을 지정할 수 있습니다:

```java
Projections.fields(ProductDto.class,
    product.id,
    product.name.as("productName"),  // DTO 필드명과 맞춤
    product.price
)
```

### 3. Projections.bean()

Setter 메서드를 통해 값을 주입합니다.

```java
List<ProductDto> products = queryFactory
    .select(Projections.bean(ProductDto.class,
        product.id,
        product.name,
        product.price
    ))
    .from(product)
    .fetch();
```

| 장점 | 단점 |
|-----|-----|
| 명시적인 setter 사용 | setter 메서드 필요 |
| 필드 순서 상관없음 | **런타임 오류** |

### Projections의 공통 문제점

**모두 컴파일 시점에 오류를 잡을 수 없습니다.**

```java
// 타입이 안 맞아도 컴파일 OK, 런타임에 터짐!
Projections.constructor(ProductDto.class,
    product.id,
    product.price,  // String 자리에 BigDecimal?
    product.name
)
```

## @QueryProjection (권장)

### 사용법

DTO 생성자에 `@QueryProjection` 어노테이션을 추가합니다.

```java
@Getter
public class ProductDto {
    private final Long id;
    private final String name;
    private final BigDecimal price;

    @QueryProjection
    public ProductDto(Long id, String name, BigDecimal price) {
        this.id = id;
        this.name = name;
        this.price = price;
    }
}
```

컴파일하면 `QProductDto`가 생성됩니다:

```java
List<ProductDto> products = queryFactory
    .select(new QProductDto(product.id, product.name, product.price))
    .from(product)
    .fetch();
```

### 장점

**1. 컴파일 시점에 오류 확인**

```java
// 타입이 안 맞으면 컴파일 에러!
new QProductDto(
    product.id,       // Long ✅
    product.price,    // name 자리에 BigDecimal? ❌ 컴파일 에러!
    product.name
)
```

**2. IDE 지원**

자동완성, 리팩토링이 완벽하게 지원됩니다.

**3. 가독성**

어떤 필드를 어떤 순서로 조회하는지 타입으로 명확합니다.

### 단점

**1. Querydsl 의존성**

DTO가 Querydsl에 의존하게 됩니다. 멀티 모듈 프로젝트에서 DTO를 공유하면 의존성이 전파됩니다.

**2. Q클래스 생성 필요**

APT 플러그인 설정과 빌드 과정이 필요합니다.

### 의존성 문제 해결 방법

DTO를 두 레이어로 분리할 수 있습니다:

```java
// 1. 조회 전용 DTO (infrastructure layer) - Querydsl 의존
@Getter
public class ProductQueryDto {
    private final Long id;
    private final String name;
    private final BigDecimal price;

    @QueryProjection
    public ProductQueryDto(Long id, String name, BigDecimal price) {
        this.id = id;
        this.name = name;
        this.price = price;
    }
}

// 2. API 응답 DTO (presentation layer) - Querydsl 의존 없음
@Getter
public class ProductResponse {
    private final Long id;
    private final String name;
    private final BigDecimal price;

    public static ProductResponse from(ProductQueryDto dto) {
        return new ProductResponse(dto.getId(), dto.getName(), dto.getPrice());
    }
}
```

## 비교 정리

| 방식 | 컴파일 체크 | IDE 지원 | Querydsl 의존 |
|------|:----------:|:-------:|:------------:|
| Projections.constructor() | ❌ | ❌ | ❌ |
| Projections.fields() | ❌ | ❌ | ❌ |
| Projections.bean() | ❌ | ❌ | ❌ |
| **@QueryProjection** | ✅ | ✅ | ✅ |

## 결론

가장 좋은 에러는 **컴파일 시점에 발견되는 에러**입니다.

런타임 에러는 QA나 운영 중에 발견될 수 있어 리스크가 큽니다. 의존성 분리가 중요하다면 조회용 DTO와 응답용 DTO를 분리하는 방법으로 해결할 수 있습니다.

저는 실무에서 `@QueryProjection`을 선호합니다.
