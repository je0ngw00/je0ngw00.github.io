---
title: "JPA @OneToOne 양방향 관계에서 Lazy Loading이 안 되는 이유와 해결책"
date: 2023-06-29 10:00:00 +0900
categories: [Development, JPA]
tags: [jpa, hibernate, onetoone, lazy-loading, n+1, performance]
---

## 개요

쿠폰 발급 배치에서 35,000건 처리에 30분이 걸리는 성능 문제가 발생했습니다. 원인을 추적해보니 `@OneToOne` 양방향 관계에서 Lazy Loading이 제대로 동작하지 않아 N+1 문제가 발생한 것이었습니다.

## 문제 상황

```java
@Entity
public class CouponEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long couponId;

    // 양방향 OneToOne: 연관관계 주인이 아닌 쪽 (mappedBy)
    @OneToOne(mappedBy = "couponEntity", fetch = FetchType.LAZY)
    private CouponUsedEntity couponUsedEntity;
}

@Entity
public class CouponUsedEntity {

    @Id
    private Long couponId;

    // 연관관계 주인 (FK 보유)
    @OneToOne
    @MapsId
    @JoinColumn(name = "coupon_id")
    private CouponEntity couponEntity;
}
```

분명히 `FetchType.LAZY`로 설정했는데, CouponEntity를 조회할 때마다 CouponUsedEntity가 함께 조회되었습니다.

```
Hibernate: select * from coupon where ...
Hibernate: select * from coupon_used where coupon_id = ?  -- 35,000번 반복!
```

## 원인: 프록시의 한계

**핵심 원인: 연관관계의 주인이 아닌 쪽에서는 Lazy Loading이 동작하지 않습니다.**

JPA의 Lazy Loading은 프록시 객체를 사용합니다. 프록시는 실제 엔티티 대신 가짜 객체를 넣어두고, 실제 접근 시 쿼리를 실행합니다.

문제는 `@OneToOne`에서 **연관관계 주인이 아닌 쪽(mappedBy가 있는 쪽)은 상대방 엔티티가 존재하는지 알 수 없다**는 점입니다.

```
CouponEntity 조회 시:
1. couponUsedEntity가 null인지, 실제 객체가 있는지 알 수 없음
2. 프록시는 null을 감쌀 수 없음
3. 결국 즉시 조회해서 확인해야 함 → Lazy Loading 불가
```

반면, 연관관계 주인 쪽(FK를 가진 쪽)은 FK 값 존재 여부로 null인지 판단할 수 있어 Lazy Loading이 가능합니다.

## 해결 방법

### 방법 1: Fetch Join (즉각적 해결)

조회 시 명시적으로 함께 가져옵니다.

```java
public List<CouponEntity> findCoupons() {
    return queryFactory
        .selectFrom(couponEntity)
        .leftJoin(couponEntity.couponUsedEntity).fetchJoin()
        .where(...)
        .fetch();
}
```

**장점**: 기존 코드 최소 변경
**단점**: 매번 join이 발생하므로 불필요한 경우에도 데이터를 가져옴

### 방법 2: @MapsId 활용 (권장)

양방향을 유지하면서 Lazy Loading을 가능하게 합니다.

```java
@Entity
public class CouponUsedEntity {

    @Id
    private Long couponId;  // PK이자 FK

    @OneToOne(fetch = FetchType.LAZY)
    @MapsId  // PK를 FK로 사용
    @JoinColumn(name = "coupon_id")
    private CouponEntity couponEntity;
}
```

`@MapsId`를 사용하면 부모의 PK 존재 여부로 자식 존재를 판단할 수 있어 Lazy Loading이 가능해집니다.

### 방법 3: 양방향 관계 제거 (근본적 해결)

양방향이 정말 필요한지 다시 생각해봅니다.

```java
@Entity
public class CouponEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long couponId;

    // 양방향 제거 - couponUsedEntity 필드 삭제
}

// 필요할 때 별도 조회
public CouponUsedEntity findByCouponId(Long couponId) {
    return couponUsedRepository.findByCouponId(couponId);
}
```

대부분의 경우 양방향 관계는 편의를 위한 것일 뿐, 필수가 아닙니다.

### 방법 4: Batch Fetch Size 설정

N+1을 완전히 해결하진 못하지만, 쿼리 수를 줄일 수 있습니다.

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 100
```

```java
// 또는 엔티티에 직접 설정
@OneToOne(mappedBy = "couponEntity", fetch = FetchType.LAZY)
@BatchSize(size = 100)
private CouponUsedEntity couponUsedEntity;
```

이렇게 하면 100개씩 IN 절로 조회합니다:

```sql
-- Before: 35,000개 쿼리
SELECT * FROM coupon_used WHERE coupon_id = ?

-- After: 350개 쿼리
SELECT * FROM coupon_used WHERE coupon_id IN (?, ?, ..., ?)  -- 100개씩
```

### 방법 5: Hibernate Bytecode Enhancement (고급)

빌드 시점에 바이트코드를 조작하여 Lazy Loading을 가능하게 합니다.

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.hibernate.orm.tooling</groupId>
    <artifactId>hibernate-enhance-maven-plugin</artifactId>
    <version>${hibernate.version}</version>
    <executions>
        <execution>
            <goals>
                <goal>enhance</goal>
            </goals>
            <configuration>
                <enableLazyInitialization>true</enableLazyInitialization>
            </configuration>
        </execution>
    </executions>
</plugin>
```

```java
@OneToOne(mappedBy = "couponEntity", fetch = FetchType.LAZY)
@LazyToOne(LazyToOneOption.NO_PROXY)  // 프록시 대신 bytecode enhancement 사용
private CouponUsedEntity couponUsedEntity;
```

**주의**: 설정이 복잡하고, 프록시와 동작 방식이 달라 기존 코드에 영향을 줄 수 있습니다.

## 해결 결과

Fetch Join을 적용한 결과:

| 항목 | Before | After |
|------|--------|-------|
| 처리 건수 | 35,000건 | 35,000건 |
| 소요 시간 | 30분 | **10초 이내** |
| 쿼리 수 | 35,001개 | 1개 |

## 정리

| 방법 | 적용 난이도 | 효과 | 권장 상황 |
|------|-----------|------|----------|
| Fetch Join | 낮음 | 높음 | 즉각적 해결 필요 |
| @MapsId | 중간 | 높음 | 새 프로젝트, 리팩토링 가능 |
| 양방향 제거 | 중간 | 높음 | 양방향이 필수 아닌 경우 |
| Batch Size | 낮음 | 중간 | 완전 해결 어려울 때 |
| Bytecode Enhancement | 높음 | 높음 | 전문적 튜닝 필요 |

`@OneToOne` 양방향 관계를 설계할 때는 이 문제를 미리 인지하고, 정말 양방향이 필요한지 고민하는 것이 좋습니다.
