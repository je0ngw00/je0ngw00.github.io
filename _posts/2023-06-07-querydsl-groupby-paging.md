---
title: "Querydsl Group By 쿼리에서 페이징 처리하기"
date: 2023-06-07 11:00:00 +0900
categories: [Development, JPA]
tags: [querydsl, jpa, paging, groupby, java]
---

## 개요

Querydsl에서 Group By가 포함된 쿼리에 페이징을 적용하면 `fetchResults()`나 `fetchCount()`를 사용할 수 없습니다. Querydsl 5.0.0부터 이 메서드들이 deprecated되었고, Group By가 있는 경우 정확한 count를 반환하지 못하기 때문입니다.

이 문제를 어떻게 해결해야 하는지 정리합니다.

> **📌 2025-01 업데이트**: Querydsl 5.x 기준으로 deprecated API 대응 방법을 정리했습니다.

## fetchResults() deprecated 이유

```java
// Querydsl 5.0.0 이전 방식 - 이제 deprecated
QueryResults<EntityDto> results = queryFactory
    .select(new QEntityDto(...))
    .from(entity)
    .groupBy(entity.field1)
    .fetchResults();  // ❌ deprecated

long total = results.getTotal();
List<EntityDto> content = results.getResults();
```

**deprecated 이유:**
1. Group By가 있으면 count 쿼리가 부정확한 결과 반환
2. Having 절이 있는 경우에도 마찬가지
3. 내부적으로 서브쿼리로 감싸서 count하는데, 복잡한 쿼리에서 제대로 동작하지 않음

## 해결 방법

### 방법 1: 별도의 Count 쿼리 (권장)

content 조회와 count 조회를 분리합니다.

```java
public Page<EntityDto> getEntitiesWithGroupBy(Pageable pageable) {

    // 1. content 조회
    List<EntityDto> content = queryFactory
        .select(new QEntityDto(
            entity.field1,
            entity.field2,
            entity.field3.sum()
        ))
        .from(entity)
        .groupBy(entity.field1, entity.field2)
        .orderBy(entity.field1.asc())
        .offset(pageable.getOffset())
        .limit(pageable.getPageSize())
        .fetch();

    // 2. count 조회 (별도 쿼리)
    Long total = queryFactory
        .select(entity.field1.countDistinct())  // Group By 대상의 distinct count
        .from(entity)
        .fetchOne();

    return new PageImpl<>(content, pageable, total != null ? total : 0L);
}
```

**핵심 포인트:**
- content는 `fetch()`로 조회
- count는 Group By 대상 컬럼의 `countDistinct()`로 계산

### 방법 2: 서브쿼리 활용 (복잡한 경우)

Group By 조건이 복잡한 경우 서브쿼리로 count합니다.

```java
public Page<OrderSummaryDto> getOrderSummary(Pageable pageable) {

    // 1. content 조회
    List<OrderSummaryDto> content = queryFactory
        .select(new QOrderSummaryDto(
            order.userId,
            order.status,
            order.amount.sum(),
            order.id.count()
        ))
        .from(order)
        .groupBy(order.userId, order.status)
        .having(order.amount.sum().gt(10000))
        .offset(pageable.getOffset())
        .limit(pageable.getPageSize())
        .fetch();

    // 2. count: 같은 조건의 그룹 수 계산
    // JPAExpressions를 사용한 서브쿼리는 제한이 있어서 Native Query 사용
    Long total = countGroupedResults();

    return new PageImpl<>(content, pageable, total);
}

private Long countGroupedResults() {
    // Native Query로 서브쿼리 count
    return (Long) entityManager.createNativeQuery(
        "SELECT COUNT(*) FROM (" +
        "  SELECT user_id, status " +
        "  FROM orders " +
        "  GROUP BY user_id, status " +
        "  HAVING SUM(amount) > 10000" +
        ") AS grouped"
    ).getSingleResult();
}
```

### 방법 3: 전체 조회 후 메모리 페이징 (소량 데이터)

데이터가 많지 않다면 전체를 조회 후 메모리에서 페이징할 수 있습니다.

```java
public Page<EntityDto> getEntitiesWithGroupBy(Pageable pageable) {

    // 전체 조회
    List<EntityDto> allResults = queryFactory
        .select(new QEntityDto(...))
        .from(entity)
        .groupBy(entity.field1, entity.field2)
        .orderBy(entity.field1.asc())
        .fetch();

    // 메모리에서 페이징
    int start = (int) pageable.getOffset();
    int end = Math.min(start + pageable.getPageSize(), allResults.size());

    List<EntityDto> pageContent = start < allResults.size()
        ? allResults.subList(start, end)
        : Collections.emptyList();

    return new PageImpl<>(pageContent, pageable, allResults.size());
}
```

**주의**: 데이터가 많으면 OOM 위험이 있으므로 소량 데이터에만 사용합니다.

## Spring Data JPA의 Slice 활용

전체 count가 필요 없다면 `Slice`를 사용하는 것도 방법입니다.

```java
public Slice<EntityDto> getEntitiesSlice(Pageable pageable) {

    List<EntityDto> content = queryFactory
        .select(new QEntityDto(...))
        .from(entity)
        .groupBy(entity.field1)
        .offset(pageable.getOffset())
        .limit(pageable.getPageSize() + 1)  // 다음 페이지 존재 여부 확인용
        .fetch();

    boolean hasNext = content.size() > pageable.getPageSize();
    if (hasNext) {
        content.remove(content.size() - 1);  // 초과 항목 제거
    }

    return new SliceImpl<>(content, pageable, hasNext);
}
```

**장점:**
- count 쿼리가 필요 없어 성능 향상
- 무한 스크롤 UI에 적합

## 권장 방식 선택 기준

| 상황 | 권장 방법 |
|------|----------|
| 단순 Group By | 방법 1 (countDistinct) |
| Having 절 포함 | 방법 2 (서브쿼리/Native) |
| 데이터 < 1000건 | 방법 3 (메모리 페이징) |
| 전체 count 불필요 | Slice 활용 |

## 근본적인 고민

Querydsl에서 Group By 페이징이 복잡한 것은 어쩌면 의도된 것일 수 있습니다. Group By 결과를 페이징하는 것 자체가 비즈니스적으로 적절한지 고민해볼 필요가 있습니다.

- 집계 결과가 많다면 → 집계 단위를 조정 (일별 → 월별)
- 리포트 성격이라면 → 비동기 처리 + 파일 다운로드
- 대시보드라면 → 상위 N건만 조회 (Top N)

쿼리를 복잡하게 만들기 전에 요구사항을 재검토하는 것이 좋습니다.
