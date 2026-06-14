---
title: "Redis 응답 지연이 커넥션 풀 고갈과 파드 재시작으로 이어진 이유: 트랜잭션 경계와 connect timeout"
date: 2026-06-14 00:00:00 +0900
categories: [Development, Spring]
tags: [redis, lettuce, hikaricp, transaction, resilience]
---

QA 환경에 배포한 뒤 API 서버가 간헐적으로 503을 뱉고 파드가 주기적으로 재시작됐습니다. 로그를 처음 펼쳤을 때 가장 먼저 눈에 띈 것은 외부 식별 헤더 하나가 비어 있다는 메시지였고, 자연스럽게 "그 헤더가 없어서 요청이 깨진다"는 가설로 출발했습니다. 그런데 추적의 끝에서 마주한 진짜 원인은 헤더가 아니라 응답하지 않는 Redis였고, 더 흥미로운 건 그 캐시 클래스에 failover 코드가 이미 있었고 정확하게 동작했다는 점이었습니다. 문제는 failover가 없는 게 아니라, 거기까지 도달하는 데 10초가 걸렸다는 것이었습니다.

## 503과 파드 재시작이 반복됐다

한두 번이면 일시적 부하로 넘겼겠지만 패턴이 있었습니다.

- API 응답이 503으로 반복해서 떨어집니다.
- 파드가 주기적으로 재시작됩니다(liveness probe 실패).
- 재시작 직후 잠깐 정상으로 돌아왔다가, 얼마 못 가 같은 증상이 다시 나타납니다.

재시작 직후 잠깐 살아난다는 점이 신경 쓰였습니다. 프로세스 자체의 결함이라기보다, 무언가가 시간이 지나며 고갈되고 그게 재시작으로 초기화되는 그림에 가까웠습니다.

## 첫 가설은 헤더 누락이었다

로그에서 가장 먼저 보인 건 외부 인증/회원 시스템이 내려주는 식별 헤더(이하 `extId`)가 비어 있다는 메시지였습니다. 이 값으로 외부→내부 ID 매핑을 조회하는데, 값이 없으니 조회가 실패하는 흐름이 보였습니다.

> 첫 가설: `extId`가 없어서 외부→내부 ID 매핑 조회가 실패하고, 그게 요청 전체를 깨뜨리는 것 아닐까?

그럴듯했습니다. 헤더가 비면 조회가 깨지는 건 사실이니까요. 하지만 이건 함정의 입구였습니다.

## 증상과 원인을 가른 예외 빈도 분포

가설을 검증하려면 "그 예외가 정말 장애를 주도할 만큼 자주 발생하는가"를 봐야 했습니다. 로그를 통째로 읽는 대신 예외 토큰의 빈도 분포부터 뽑았습니다.

```bash
grep -oE 'Exception|Error' app.log | sort | uniq -c | sort -rn
```

여기서 그림이 바뀌었습니다. `extId` 관련 예외는 분명 있었지만 소수였고, 압도적 다수를 차지한 신호는 다른 것이었습니다.

```
Connection is not available, request timed out after 30000ms
```

그리고 같은 시간대에 커넥션 풀 상태가 찍혀 있었습니다.

```
HikariPool-1 - Pool stats (total=10, active=10, idle=0, waiting=8)
HikariPool-1 - Pool stats (total=10, active=10, idle=0, waiting=11)
```

`total=10, active=10, idle=0`. 풀의 커넥션 10개가 전부 점유 상태였고, 거기에 8~11개의 요청이 줄을 서서 30초를 기다리다 타임아웃으로 죽고 있었습니다. `extId` 누락은 증상이었지, 풀을 고갈시키는 원인이 아니었습니다. 결정적인 한 줄이 더 있었습니다. Redis 엔드포인트가 unresolved 상태라는 로그였습니다. QA 환경의 Redis가 그 시점에 응답하지 않고 있었던 것입니다.

이때 첫 가설을 그대로 신뢰하지 않고 관측된 신호를 분류했습니다.

| 신호 | 판정 | 근거 |
|---|---|---|
| `extId` 누락 예외 | 증상 | 발생하지만 소수, 풀 고갈을 설명 못 함 |
| 풀 고갈 + 30초 타임아웃 | 원인 후보 | 503·재시작 시점과 정확히 일치 |
| Redis 엔드포인트 unresolved | 트리거 | 풀 고갈의 선행 사건 |

증상을 쫓았다면 헤더 검증 로직만 고치고 장애는 그대로 재발했을 것입니다.

## Redis 장애가 DB 풀로 번진 경로

남은 질문은 "Redis가 죽은 것과 DB 커넥션 풀 고갈이 어떻게 연결되는가"였습니다. JSON 로그의 클래스/메서드, 요청 URL 필드로 호출 경로를 역추적하자 사슬이 드러났습니다.

핵심은 외부→내부 ID 매핑을 캐시에서 읽는 조회 서비스였습니다. 이 클래스에는 클래스 레벨로 `@Transactional(readOnly = true)`가 붙어 있었습니다. 캐시 조회 메서드가 호출되는 순간 DB 트랜잭션이 열리고, 그 트랜잭션 안에서 Redis 연결을 시도하는 구조였습니다.

Redis가 응답하지 않으니 Redis 연결 시도가 블로킹됐고, 그동안 이미 할당받은 DB 커넥션을 반납하지 못한 채 스레드가 묶였습니다. 요청이 몰리면 이 패턴이 누적되어 풀 10개가 순식간에 바닥났습니다.

여기에 헬스체크가 기름을 부었습니다. liveness 프로브가 호출하는 헬스 경로 역시 사용자 컨텍스트를 파싱하는 필터를 통과했고, 헤더가 있으면 같은 외부→내부 ID 매핑 경로(Redis/DB)를 탔습니다. 풀이 고갈된 상태에서 헬스 프로브도 30초를 기다리다 실패 → liveness 실패 → 파드 재시작으로 이어졌습니다.

```
Redis 엔드포인트 unresolved (응답 없음)
        │
        ▼
매핑 조회 서비스 = 클래스 레벨 @Transactional(readOnly=true)
   → DB 트랜잭션이 열린 상태에서 Redis 연결 시도
        │
        ▼
Redis 연결 블로킹 (Lettuce 기본 ~10s)
   → DB 커넥션을 쥔 채 스레드 대기
        │
        ▼
요청 누적 → HikariPool(10) 고갈 (active=10, waiting=8~11)
        │
        ├───────────────────────────┐
        ▼                            ▼
업무 요청: 30s 타임아웃 → 503     헬스 프로브 → 같은 매핑 경로 → 30s 타임아웃
                                  → liveness 실패 → 파드 재시작 → (다시 고갈)
```

하나의 외부 의존성 장애가 트랜잭션 경계 설계의 약점을 타고 DB 풀, 헬스체크, 파드 생명주기까지 번진 전형적인 연쇄 장애였습니다. 사실 외부 호출이 트랜잭션에 묶여 커넥션 반납이 늦어지는 패턴은 예전에 PG 결제 호출에서 한 번 겪고 정리해 둔 적이 있는데([트랜잭션 내 외부 호출 분리로 커넥션 풀 병목 해결하기]({% post_url 2023-05-22-transaction-external-call %})), 같은 함정을 이번엔 Redis에서 다시 만난 셈이었습니다.

## failover는 있었지만 도달이 느렸다

여기서 가장 흥미로운 지점이 나왔습니다. 이 매핑 조회 서비스에는 이미 failover 코드가 있었습니다. Redis 조회가 실패하면 `DataAccessException`을 잡아 DB 조회로 우회하는 로직이고, 그 코드는 정확하게 동작했습니다. catch 블록에 도달하기만 하면 DB로 떨어져 결과를 돌려줬습니다.

그렇다면 failover가 있는데 왜 장애가 났을까요. 문제는 "failover가 없다"가 아니라 그 catch 블록에 도달하기까지가 느렸다는 것이었습니다.

Lettuce는 connect timeout을 명시하지 않으면 기본 약 10초를 블로킹합니다. Redis가 응답하지 않을 때 호출 스레드는 약 10초 동안 연결을 시도하며 묶여 있다가, 그제서야 예외가 터지고 catch로 진입합니다. 그 10초 동안 DB 커넥션은 점유된 채입니다.

함정은 한 겹 더 있었습니다. 우리는 `spring.data.redis.timeout`을 2000ms로 설정해 두고 있었습니다. "타임아웃 걸어놨는데?"라고 안심하기 쉽습니다. 하지만 이 값은 command timeout, 즉 명령을 보낸 뒤 응답을 기다리는 시간입니다. 연결을 맺는 단계(connect)에는 적용되지 않습니다. 연결 자체가 안 되는 상황에서는 이 2초 설정이 개입할 여지가 없었습니다.

정리하면 failover 로직은 정확했고, failover에 도달하는 속도가 느렸으며, `spring.data.redis.timeout=2000`은 command timeout이라 connect 단계를 막지 못했습니다. "failover가 있다"와 "failover가 빠르다"는 완전히 다른 명제였습니다.

## 세 곳에 친 방어선

원인이 한 겹이 아니었으므로 방어선을 세 군데에 쳤습니다. 가장 구조적으로 깨끗한 것부터 적었습니다.

### 캐시 조회를 트랜잭션 밖으로

가장 근본적인 수정입니다. 매핑 조회 서비스의 클래스 레벨 `@Transactional(readOnly = true)`를 제거했습니다. 풀을 인질로 잡은 직접 원인은 이 어노테이션이 메서드 전체, 즉 Redis 호출까지 하나의 DB 트랜잭션으로 감쌌다는 데 있었습니다. 이걸 떼어내면 Redis 연결이 블로킹돼도 그 시간 동안 DB 커넥션을 붙잡고 있지 않게 됩니다. 풀 고갈의 연결 고리 자체가 끊깁니다.

```java
// 제거됨:
// @Transactional(readOnly = true)  ← 클래스 레벨 어노테이션 삭제
@Slf4j
@Service
@RequiredArgsConstructor
public class ExternalIdMappingService {
    // failover: catch (DataAccessException) → DB fallback (기존 코드, 그대로 유지)
}
```

failover 로직은 손대지 않았습니다. 원래 정확했으니까요.

덧붙이면 이 프로젝트는 Spring Data JPA를 `enableDefaultTransactions = false`로 쓰고 있어, 명시적 트랜잭션이 없으면 리포지토리 단건 조회가 별도의 긴 트랜잭션을 열지 않습니다. 다만 이건 보조적인 디테일입니다. 설령 기본값(`true`)이었더라도, 리포지토리 쿼리가 여는 트랜잭션은 그 메서드 호출 안으로 한정되어 반환 시 커넥션을 반납하므로, 풀을 붙잡고 있던 진짜 범인은 어디까지나 Redis 호출을 함께 감싸던 클래스 레벨 트랜잭션이었습니다.

### connect/command timeout을 코드로 고정

근본 수정이 풀 고갈을 막더라도, Redis가 죽었을 때 호출 스레드가 10초씩 블로킹되는 것 자체는 바람직하지 않습니다. 빠르게 실패해 fallback으로 진입하게 만들어야 합니다.

`spring.data.redis.connect-timeout`이라는 프로퍼티가 있기는 합니다. 그런데 연결이 끊긴 동안 큐에 쌓이는 명령을 만료시키는 `TimeoutOptions`는 yml로 지정할 수 없고, connect/command/큐잉 정책을 한곳에서 함께 보장하고 싶었습니다. 그래서 `LettuceClientConfigurationBuilderCustomizer`로 코드에서 묶었습니다.

```java
private static final Duration CONNECT_TIMEOUT = Duration.ofMillis(300);
private static final Duration COMMAND_TIMEOUT = Duration.ofMillis(1000);

@Bean
public LettuceClientConfigurationBuilderCustomizer lettuceResilienceCustomizer() {
    return builder -> builder
            .commandTimeout(COMMAND_TIMEOUT)
            .clientOptions(ClientOptions.builder()
                    .socketOptions(SocketOptions.builder()
                            .connectTimeout(CONNECT_TIMEOUT)
                            .build())
                    // 연결이 끊긴 상태에서 명령을 무한 큐잉하지 않고,
                    // command timeout이 지나면 만료시켜 호출 스레드가 fallback으로 진입하게 한다.
                    .timeoutOptions(TimeoutOptions.enabled(COMMAND_TIMEOUT))
                    .build());
}
```

핵심은 세 가지입니다.

- `connectTimeout(300ms)`: Redis가 응답하지 않으면 300ms 만에 연결을 포기하고 예외를 던집니다. 약 10초가 0.3초가 됩니다.
- `timeoutOptions(TimeoutOptions.enabled(...))`: Lettuce는 기본적으로 연결이 끊긴 동안 명령을 큐에 쌓아두고 command timeout을 적용하지 않습니다. 이 옵션을 켜야 큐잉된 명령도 command timeout이 지나면 만료됩니다. 즉시 실패가 아니라 설정한 timeout이 지난 뒤 실패한다는 점이 중요합니다. 이게 빠진 채 commandTimeout만 걸면 재연결을 기다리며 명령이 무한히 쌓이는 경로가 남습니다.
- `ConnectionFactory`를 직접 생성하지 않습니다. 빌더에 레이어링만 합니다. 직접 팩토리를 만들면 `@ConditionalOnMissingBean`을 건드려 standalone/cluster/sentinel 자동 감지가 통째로 우회되는데, 빌더 커스터마이저는 Spring Boot의 자동 구성 위에 timeout만 얹으므로 상위 환경의 cluster 모드 설정을 보존합니다. 잘못 만지면 운영 환경에서 cluster 연결이 깨질 수 있어, 의외로 중요한 설계 선택이었습니다.

### 헬스체크 경로 격리

헬스 프로브가 Redis/DB 의존성 경로를 타는 것 자체가 설계 결함이었습니다. 의존성이 죽었을 때 프로브가 막혀 파드가 재시작되면, 정작 복구할 시간을 빼앗깁니다. 그래서 헬스 경로를 필터와 인터셉터 양쪽에서 제외했습니다.

```java
@Override
protected boolean shouldNotFilter(HttpServletRequest request) {
    String uri = request.getRequestURI();
    return uri.equals("/v1/api/health")
            || uri.equals("/api/health")
            || uri.startsWith("/swagger-ui")
            || uri.startsWith("/v3/api-docs")
            || uri.startsWith("/actuator");
}
```

사용자 컨텍스트를 파싱하는 필터는 헤더가 있으면 외부→내부 ID 매핑 조회(Redis/DB)를 수행합니다. 헬스 프로브가 이 경로를 타면 의존성 장애 시 프로브가 막혀 파드가 재시작됩니다. `shouldNotFilter`로 헬스 경로를 아예 건너뛰게 했고, 인터셉터 등록에서도 동일하게 제외했습니다.

```java
registry.addInterceptor(userContextInterceptor)
        .addPathPatterns("/**")
        .excludePathPatterns(
                "/swagger-ui/**", "/v3/api-docs/**", "/actuator/**",
                "/v1/api/health", "/api/health");
```

이제 헬스 프로브는 Redis/DB가 죽어도 순수하게 프로세스가 살아 있는지만 확인하고 통과합니다. 의존성 장애와 파드 생명주기가 격리됐습니다.

## 수정하며 마주친 함정들

수정 과정에서 따로 적어둘 만한 함정이 몇 가지 있었습니다.

command timeout이 코드(1000ms)와 yml(2000ms)에 동시에 존재하게 됐습니다. 커스터마이저는 Boot 자동 구성 뒤에 적용되므로 코드값(1000ms)이 yml값(2000ms)을 덮어씁니다. 더 공격적인 timeout이 이기는 게 의도한 동작이지만, "yml에 2초로 적어놨는데 왜 1초로 동작하나?"라는 혼란을 부를 수 있습니다. 적용 순서를 모르면 디버깅할 때 헤맬 지점이라 인지하고 남겨뒀습니다.

connect timeout과 command timeout이 서로 다른 단계라는 점도 한 번 더 짚어둘 만합니다. `spring.data.redis.timeout`은 command timeout이라 연결 단계를 막지 못합니다. connect timeout은 `spring.data.redis.connect-timeout` 프로퍼티가 있지만, 큐잉 명령을 만료시키는 `TimeoutOptions`까지 함께 얹으려면 결국 코드로 묶는 편이 깔끔했습니다. 이번 장애의 핵심 갭이 정확히 이 "connect와 command는 다른 레이어"라는 지점이었습니다.

헬스 경로가 `/v1/api/health`인 것도 함정이었습니다. 공통 모듈이 모든 경로에 `/v1` prefix를 자동으로 붙이기 때문에 실제 URI는 `/v1/api/health`입니다. 다만 prefix가 적용되지 않는 경로로 프로브가 들어올 가능성을 막기 위해 `/api/health`도 함께 제외했습니다. prefix 동작을 모르고 `/api/health`만 제외했다면 실제 프로브 경로를 못 막을 뻔했습니다.

## 무거운 컨텍스트 없이 테스트한 이유

세 수정 모두 단위 수준에서 검증했고, 핵심은 무거운 Spring 컨텍스트를 띄우지 않고 동작만 직접 확인한 것입니다.

커스터마이저 검증은 `ApplicationContextRunner`로 컨텍스트를 통째로 띄우는 대신, 빌더에 커스터마이저를 적용한 뒤 결과 설정에서 connect/command timeout을 직접 꺼내 단언했습니다. "빌더에 timeout이 반영되는가"는 컨텍스트 전체와 무관한 순수한 구성 검증이므로, 그 수준에서 끝내는 게 피드백 루프상 옳습니다. 필터 검증은 헬스 경로에 대해 `shouldNotFilter`가 `true`를, 일반 업무 경로에 대해 `false`를 반환하는지 확인했고, 매핑 조회는 캐시 조회와 Redis 실패 시 DB fallback 경로를 확인했습니다.

## 돌아보며

이번 장애를 추적하면서 몇 가지를 다시 확인했습니다. 가장 크게 남은 건 회복탄력성을 가르는 게 "failover가 있느냐"가 아니라 "얼마나 빨리 거기 도달하느냐"라는 점입니다. catch 블록이 아무리 정확해도 도달에 10초가 걸리면 그 10초 동안 자원이 묶이고, 그 사이 풀이 마르면 정확한 fallback은 손도 써보지 못합니다. 그리고 그 속도는 command timeout이 아니라 connect timeout이라는, 한 단계 아래 레이어에서 결정됩니다.

증상과 원인을 분리하는 데 예외 빈도 분포가 결정적이었던 것도 기록해 둘 만합니다. 가장 먼저 눈에 띈 예외가 범인이 아니었고, 로그를 통독하기 전에 무엇이 압도적 다수인지부터 본 덕에 증상을 원인으로 오인하지 않았습니다. 외부 I/O를 트랜잭션 안에 두지 말 것, 그리고 헬스체크는 업무 의존성 경로를 타지 않게 할 것. 둘 다 알고는 있던 원칙인데, 같은 함정을 자원만 바꿔 다시 만나고 나서야 몸에 새겨졌습니다.
