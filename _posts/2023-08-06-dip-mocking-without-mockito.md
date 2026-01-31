---
title: "Mockito 없이 테스트하기: DIP를 활용한 Fake 객체 패턴"
date: 2023-08-06 10:00:00 +0900
categories: [Development, Testing]
tags: [java, testing, dip, mockito, fake-object, clean-architecture]
---

## 개요

Repository 테스트를 위해 H2 DB + `@SpringBootTest`를 사용했는데, 테스트가 느리고 자주 깨졌습니다. Mockito로 전환했지만, 라이브러리 의존도가 높아지고 "가짜 통과" 문제가 생겼습니다. DIP(의존 역전 원칙)를 적용해 순수 Java로 테스트하는 방식으로 바꾼 경험을 공유합니다.

## 테스트 진화 과정

### 1단계: H2 DB + SpringBootTest

```java
@SpringBootTest
@ActiveProfiles("test")
public class CouponServiceTest {

    @Autowired
    private DataSource dataSource;

    @Autowired
    private CouponService couponService;

    @BeforeAll
    void beforeAll() throws Exception {
        try (Connection conn = dataSource.getConnection()) {
            ScriptUtils.executeSqlScript(conn,
                new ClassPathResource("/db/h2/init_data.sql"));
        }
    }

    @AfterEach
    void afterEach() {
        couponRepository.deleteAllInBatch();
    }

    @Test
    void 생일_쿠폰이_정상_생성된다() {
        // given
        CouponCreateRequest request = CouponCreateRequest.builder()
            .type(CouponType.BIRTHDAY)
            .build();

        // when
        CouponCreateResponse result = couponService.create(request);

        // then
        assertThat(result.getStatus()).isEqualTo(CouponStatus.PUBLISHED);
    }
}
```

**문제점:**
- 테스트 속도가 느림 (Spring Context 로딩, DB 연결)
- Entity 변경 시 SQL 스크립트도 수정 필요
- H2와 운영 DB(MySQL/PostgreSQL) 차이로 인한 오류
- 테스트 간 데이터 충돌

### 2단계: Mockito 적용

```java
@ExtendWith(MockitoExtension.class)
public class CouponServiceTest {

    @InjectMocks
    private CouponService couponService;

    @Mock
    private CouponRepository couponRepository;

    @Test
    void 생일_쿠폰이_정상_생성된다() {
        // given
        CouponEntity savedEntity = CouponEntity.builder()
            .id(1L)
            .status(CouponStatus.PUBLISHED)
            .build();
        given(couponRepository.save(any())).willReturn(savedEntity);

        CouponCreateRequest request = CouponCreateRequest.builder()
            .type(CouponType.BIRTHDAY)
            .build();

        // when
        CouponCreateResponse result = couponService.create(request);

        // then
        assertThat(result.getStatus()).isEqualTo(CouponStatus.PUBLISHED);
    }
}
```

**개선된 점:**
- 테스트 속도 향상
- DB 의존성 제거

**새로운 문제점:**
- Mockito 문법 학습 필요
- `given(...).willReturn(...)` 설정이 복잡해짐
- **실제 저장 로직을 테스트하지 않음** (가짜 통과)
- Repository 변경 시 Mock 설정도 변경 필요

### 3단계: DIP + Fake 객체

## DIP 적용하기

### Repository 인터페이스 분리

```java
// 인터페이스 (도메인 계층)
public interface CouponRepository {
    CouponEntity save(CouponEntity coupon);
    Optional<CouponEntity> findById(Long id);
    List<CouponEntity> findByUserId(Long userId);
}
```

```java
// 실제 구현체 (인프라 계층)
@Repository
@RequiredArgsConstructor
public class CouponRepositoryImpl implements CouponRepository {

    private final CouponJpaRepository jpaRepository;  // Spring Data JPA

    @Override
    public CouponEntity save(CouponEntity coupon) {
        return jpaRepository.save(coupon);
    }

    @Override
    public Optional<CouponEntity> findById(Long id) {
        return jpaRepository.findById(id);
    }

    @Override
    public List<CouponEntity> findByUserId(Long userId) {
        return jpaRepository.findByUserId(userId);
    }
}
```

### 테스트용 Fake 구현체

```java
// 테스트용 Fake 구현체
public class FakeCouponRepository implements CouponRepository {

    private final Map<Long, CouponEntity> storage = new HashMap<>();
    private final AtomicLong idGenerator = new AtomicLong(1);

    @Override
    public CouponEntity save(CouponEntity coupon) {
        if (coupon.getId() == null) {
            // ID 자동 생성 (실제 DB처럼 동작)
            ReflectionTestUtils.setField(coupon, "id", idGenerator.getAndIncrement());
        }
        storage.put(coupon.getId(), coupon);
        return coupon;
    }

    @Override
    public Optional<CouponEntity> findById(Long id) {
        return Optional.ofNullable(storage.get(id));
    }

    @Override
    public List<CouponEntity> findByUserId(Long userId) {
        return storage.values().stream()
            .filter(c -> c.getUserId().equals(userId))
            .collect(Collectors.toList());
    }

    // 테스트 헬퍼 메서드
    public void clear() {
        storage.clear();
        idGenerator.set(1);
    }

    public int size() {
        return storage.size();
    }
}
```

### 테스트 코드

```java
public class CouponServiceTest {

    private FakeCouponRepository couponRepository;
    private CouponService couponService;

    @BeforeEach
    void setUp() {
        couponRepository = new FakeCouponRepository();
        couponService = new CouponService(couponRepository);
    }

    @Test
    void 생일_쿠폰이_정상_생성된다() {
        // given
        CouponCreateRequest request = CouponCreateRequest.builder()
            .type(CouponType.BIRTHDAY)
            .userId(1L)
            .build();

        // when
        CouponCreateResponse result = couponService.create(request);

        // then
        assertThat(result.getStatus()).isEqualTo(CouponStatus.PUBLISHED);
        assertThat(couponRepository.size()).isEqualTo(1);  // 실제 저장 확인
    }

    @Test
    void 동일_사용자의_쿠폰_목록을_조회한다() {
        // given
        couponRepository.save(CouponEntity.builder().userId(1L).build());
        couponRepository.save(CouponEntity.builder().userId(1L).build());
        couponRepository.save(CouponEntity.builder().userId(2L).build());

        // when
        List<CouponEntity> result = couponService.findByUserId(1L);

        // then
        assertThat(result).hasSize(2);
    }
}
```

## Mock vs Fake vs Stub

| 용어 | 설명 | 예시 |
|------|------|------|
| **Mock** | 호출 여부/횟수/인자 검증 | `verify(repo).save(any())` |
| **Stub** | 미리 정해진 값 반환 | `given(...).willReturn(...)` |
| **Fake** | 실제와 유사하게 동작하는 구현체 | In-memory Repository |

Mockito는 주로 Mock + Stub 역할을 합니다. Fake는 실제 동작을 흉내내므로 더 현실적인 테스트가 가능합니다.

## 장점

### 1. 테스트 격리

외부 의존성 없이 순수 Java로 테스트합니다. 실패 원인을 명확히 파악할 수 있습니다.

### 2. 테스트 속도

Spring Context 로딩이 없어 밀리초 단위로 실행됩니다.

```
Before (SpringBootTest): 15초
After (순수 Java): 0.1초
```

### 3. 유연한 설계

DIP를 적용하면 구현체 교체가 쉬워집니다.

```java
// JPA → Elasticsearch로 변경해도 테스트 코드는 동일
public class ElasticsearchCouponRepository implements CouponRepository {
    // Elasticsearch 구현
}
```

### 4. 실제 로직 테스트

Mock은 "호출되었는가"만 확인하지만, Fake는 "제대로 동작하는가"를 확인합니다.

```java
// Mock: save()가 호출되었는지만 확인
verify(couponRepository).save(any());

// Fake: 실제로 저장되었는지 확인
assertThat(couponRepository.findById(1L)).isPresent();
```

### 5. 라이브러리 독립성

Mockito 버전 업그레이드나 API 변경에 영향받지 않습니다.

## 헥사고날 아키텍처와의 관계

이 패턴은 헥사고날(포트와 어댑터) 아키텍처의 핵심 원칙과 일치합니다.

```
┌─────────────────────────────────────┐
│           Application               │
│  ┌─────────────────────────────┐   │
│  │        Domain               │   │
│  │  ┌───────────────────────┐  │   │
│  │  │    CouponService      │  │   │
│  │  └───────────────────────┘  │   │
│  │            ↓                │   │
│  │  ┌───────────────────────┐  │   │
│  │  │  CouponRepository     │  │   │  ← Port (인터페이스)
│  │  │    (Interface)        │  │   │
│  │  └───────────────────────┘  │   │
│  └─────────────────────────────┘   │
│              ↓                     │
│  ┌─────────────────────────────┐   │
│  │  CouponRepositoryImpl      │   │  ← Adapter (구현체)
│  │  (JPA / Elasticsearch)     │   │
│  └─────────────────────────────┘   │
│              ↓                     │
│  ┌─────────────────────────────┐   │
│  │  FakeCouponRepository      │   │  ← Test Adapter
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
```

## 적용 범위

DB뿐만 아니라 외부 연동에도 동일하게 적용 가능합니다.

```java
// 외부 API 클라이언트 인터페이스
public interface PaymentClient {
    PaymentResult pay(PaymentRequest request);
}

// 실제 구현
public class TossPaymentClient implements PaymentClient { ... }

// 테스트용 Fake
public class FakePaymentClient implements PaymentClient {
    private boolean shouldFail = false;

    @Override
    public PaymentResult pay(PaymentRequest request) {
        if (shouldFail) {
            throw new PaymentFailedException("테스트 실패");
        }
        return PaymentResult.success(request.getAmount());
    }

    public void setShouldFail(boolean fail) {
        this.shouldFail = fail;
    }
}
```

## 정리

| 방식 | 속도 | 현실성 | 유지보수 |
|------|------|--------|----------|
| H2 + SpringBootTest | 느림 | 중간 | 어려움 |
| Mockito | 빠름 | 낮음 | 중간 |
| DIP + Fake | 빠름 | 높음 | 쉬움 |

Mockito가 나쁜 것은 아닙니다. 상황에 따라 적절히 선택하면 됩니다. 다만, **테스트가 보내는 신호(설계 문제)를 Mockito로 덮어버리지 않도록** 주의해야 합니다.

---

**다음으로 읽으면 좋은 글:**
- [LocalStack으로 AWS SQS/SNS 로컬 테스트 환경 구축하기](/posts/localstack-sqs-sns-part1/) - 외부 서비스 의존성을 로컬에서 테스트하기
- [상속보다 컴포지션: 결제 시스템 리팩토링 사례](/posts/composition-over-inheritance/) - DIP 적용을 통한 유연한 설계
