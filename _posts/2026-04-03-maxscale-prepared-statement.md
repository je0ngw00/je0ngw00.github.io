---
title: "MaxScale에서 Prepared Statement가 죽는 이유: useServerPrepStmts만으로는 부족하다"
date: 2026-04-03 00:00:00 +0900
categories: [Development, Backend]
tags: [MariaDB, MaxScale, JDBC, PreparedStatement, HikariCP, 장애대응]
---

MaxScale 프록시 환경에서 `useServerPrepStmts=false`로 설정해도 `COM_STMT_BULK_EXECUTE` 에러가 계속 발생할 수 있습니다. `useBulkStmts` 옵션이 `useServerPrepStmts`와 **독립적으로** server-side prepared statement 프로토콜을 사용하기 때문입니다. 이 글은 금요일 오후 장애 대응에서 이 사실을 소스코드 추적으로 확인하기까지의 과정을 정리한 글입니다.

## 금요일 오후에 쏟아진 1,313건의 에러

금요일 오후 5시. 퇴근을 앞둔 시간에 모니터링 알림이 울렸습니다.

```
Unknown prepared statement handler (0) given to mysqld_stmt_execute
```

```
Unknown prepared statement handler for COM_STMT_EXECUTE given to MaxScale
```

product-api 서비스의 양쪽 Pod에서 동시에 에러가 발생하기 시작했습니다. 17:02부터 17:30까지 약 28분간 총 1,313건. 분당 최대 수십 건씩 쏟아졌습니다.

| 시간 | 상태 | 비고 |
|------|------|------|
| 17:02 | 에러 시작 | 양쪽 Pod 동시 발생 |
| 17:02~17:30 | 에러 폭주 | 총 1,313건 |
| 17:30 | 1차 조치 배포 | `useServerPrepStmts=false` |
| 17:30~18:xx | 에러 재발 | `COM_STMT_BULK_EXECUTE` |
| 18:xx | 2차 조치 배포 | `useBulkStmts=false` 추가 |
| 이후 15시간+ | 에러 0건 | 최종 해결 확인 |

이상한 점이 있었습니다. **같은 MaxScale을 사용하는 다른 서비스들은 모두 정상**이었습니다. product-api만 유독 영향을 받았습니다. 나중에 확인해보니 이유는 단순했습니다. product-api만 `useServerPrepStmts=true`와 `useBulkStmts=true`를 명시적으로 설정하고 있었고, 다른 서비스들은 기본값(client-side prepared statement)을 사용 중이었습니다.

## MaxScale이 중간에 있는 이유

```
┌─────────────┐     ┌───────────┐     ┌──────────────────┐
│ Spring Boot │────>│ MaxScale  │────>│ MariaDB Main     │
│ + HikariCP  │     │ (Proxy)   │────>│ MariaDB Replica  │
└─────────────┘     └───────────┘     └──────────────────┘
     App              L4 Proxy           Database
```

MaxScale은 MariaDB의 공식 데이터베이스 프록시입니다. Read/Write 분리, 로드밸런싱, 자동 페일오버 등을 담당합니다. 애플리케이션은 MaxScale 주소로 접속하고, MaxScale이 실제 DB 서버로 라우팅합니다.

문제는 이 **중간자**가 backend DB 커넥션을 자체적으로 관리한다는 점입니다. Server-side prepared statement는 DB 서버에 쿼리를 미리 등록(prepare)하고, 이후에는 핸들러 번호만으로 실행(execute)하는 방식입니다. MaxScale이 backend 커넥션을 재조정하면, 새 커넥션에는 이전에 prepare된 statement가 존재하지 않습니다.

## Server-Side Prepared Statement의 함정

Prepared Statement에는 두 가지 방식이 있습니다.

| 방식 | 동작 | 프로토콜 |
|------|------|----------|
| **Client-side** | 애플리케이션이 파라미터를 직접 바인딩하여 완성된 SQL 전송 | `COM_QUERY` (text) |
| **Server-side** | DB 서버에 SQL 등록 후 핸들러 ID로 실행 | `COM_STMT_PREPARE` + `COM_STMT_EXECUTE` (binary) |

MariaDB Connector/J의 `useServerPrepStmts=true` 설정은 server-side 방식을 활성화합니다. 성능상 이점이 있지만, **프록시 환경에서는 위험합니다**.

MaxScale이 backend 커넥션을 교체하면 다음과 같은 일이 벌어집니다.

1. 기존 커넥션에서 prepare된 statement handler가 사라짐
2. 클라이언트는 여전히 이전 handler ID로 `COM_STMT_EXECUTE` 요청
3. DB(또는 MaxScale)는 해당 handler를 찾지 못함
4. `Unknown prepared statement handler` 에러 발생

## 1차 조치: useServerPrepStmts=false

원인이 파악되자 즉시 조치했습니다.

```yaml
# Before
spring.datasource.hikari.data-source-properties:
  useServerPrepStmts: true

# After
spring.datasource.hikari.data-source-properties:
  useServerPrepStmts: false
```

배포 완료. 에러가 멈추길 기대했습니다.

## 재발: 다른 프로토콜에서 같은 문제

배포 후, 에러 로그가 다시 올라왔습니다. 하지만 메시지가 **미묘하게 달랐습니다**.

```
Unknown prepared statement handler for COM_STMT_BULK_EXECUTE given to MaxScale
```

`COM_STMT_EXECUTE`가 아니라 `COM_STMT_BULK_EXECUTE`. 다른 프로토콜 커맨드였습니다.

> "useServerPrepStmts를 껐는데 왜 아직도 server-side 프로토콜을 쓰지?"

이 질문이 핵심이었습니다.

## 소스코드 추적: useBulkStmts의 독립적 코드 경로

답을 찾기 위해 MariaDB Connector/J 2.x의 소스코드를 직접 분석했습니다.

### executeBatch() 분기 로직

아래는 batch 실행 시 프로토콜을 결정하는 핵심 분기를 단순화한 의사 코드입니다. 실제 소스에서는 `BasePreparedStatement`와 `ClientPreparedStatement`에 걸쳐 구현되어 있으며, 버전에 따라 클래스 구조와 조건이 다릅니다.

```java
// MariaDB Connector/J — batch 실행 분기 (핵심 로직 단순화)
public int[] executeBatch() throws SQLException {
    // ...
    if (useBulkStmts                          // <-- useBulkStmts 옵션만 확인!
        && !hasLongData
        && parameterCount > 0) {
        // COM_STMT_BULK_EXECUTE 프로토콜 사용
        executeBatchBulk();
    } else {
        // 일반 batch 실행
        executeBatchStandard();
    }
}
```

핵심 발견: **`useBulkStmts`는 `useServerPrepStmts`를 확인하지 않습니다.**

이 두 옵션은 완전히 **독립적인 코드 경로**를 탑니다.

```
                    +- useServerPrepStmts --> COM_STMT_PREPARE/EXECUTE
                    |    (개별 쿼리)          Server-side binary protocol
                    |
JDBC Connection ----+
                    |
                    +- useBulkStmts -------> COM_STMT_BULK_EXECUTE
                    |    (배치 실행)          Server-side binary protocol
                    |                        !! useServerPrepStmts와 독립!
                    |
                    +- useBatchMultiSend --> COM_QUERY (pipelining)
                         (배치 실행)          Text protocol -> MaxScale 안전
```

`useBulkStmts`가 `true`이면, `useServerPrepStmts`가 `false`여도 batch 실행 시 server-side binary 프로토콜(`COM_STMT_BULK_EXECUTE`)을 사용합니다. MaxScale 입장에서는 둘 다 동일한 문제를 일으킵니다. prepared statement handler가 필요한 프로토콜이기 때문입니다.

### 왜 이 함정에 빠지는가

`useServerPrepStmts`라는 이름이 "server-side prepared statement의 마스터 스위치"처럼 보이기 때문입니다. 하지만 실제로는 개별 쿼리 실행 경로만 제어합니다. batch 실행 경로는 `useBulkStmts`가 별도로 제어합니다.

MariaDB Connector/J 3.2.0(2023-08-25)에서 `useBulkStmts`의 기본값이 `true`에서 `false`로 변경되었습니다. [변경 사유](https://mariadb.com/kb/en/mariadb-connector-j-3-2-0-release-notes/)는 optimistic locking 환경에서 `Statement.SUCCESS_NO_INFO`가 반환되어 정확한 update count를 얻지 못하는 문제였습니다. 하지만 이 프로젝트에서는 성능 최적화를 위해 **명시적으로 `true`를 설정**해두고 있었습니다.

## 2차 조치: binary protocol 완전 차단

이 프로젝트에서는 MariaDB Connector/J **2.x**를 사용하고 있었습니다. 2.x에서는 `useBatchMultiSend` 옵션으로 text 프로토콜 기반의 파이프라이닝을 활성화할 수 있습니다.

```yaml
# Before (1차 조치 후 상태)
spring.datasource.hikari.data-source-properties:
  useServerPrepStmts: false
  useBulkStmts: true        # <-- 여전히 binary protocol 사용

# After (2차 조치)
spring.datasource.hikari.data-source-properties:
  useServerPrepStmts: false
  useBulkStmts: false        # <-- text protocol로 전환
  useBatchMultiSend: true    # <-- batch 성능 보완 (text protocol pipelining)
```

`useBatchMultiSend=true`는 text 프로토콜 기반의 파이프라이닝이므로 MaxScale 커넥션 재조정에 영향을 받지 않습니다. batch 성능 저하를 최소화하면서 안정성을 확보하는 선택이었습니다.

> 참고: `useBatchMultiSend`는 Connector/J 2.x 전용 옵션입니다. 3.x에서는 이 옵션이 제거되었고, 내부 파이프라이닝이 기본 동작으로 통합되었습니다. 3.x를 사용한다면 `useServerPrepStmts=false`와 `useBulkStmts=false`만 설정하면 됩니다.

최종 설정 비교는 다음과 같습니다.

| 옵션 | 변경 전 | 변경 후 | 프로토콜 | MaxScale 안전 |
|------|---------|---------|----------|---------------|
| `useServerPrepStmts` | `true` | **`false`** | `COM_STMT_EXECUTE` → `COM_QUERY` | O |
| `useBulkStmts` | `true` | **`false`** | `COM_STMT_BULK_EXECUTE` → 일반 batch | O |
| `useBatchMultiSend` | `true` | `true` (유지) | `COM_QUERY` pipelining | O (원래 안전) |

2차 조치 배포 후 모니터링을 지속한 결과, **15시간 이상 관련 에러가 단 한 건도 발생하지 않았습니다**.

## MaxScale 버전과 근본 원인

인프라팀 확인 결과, 운영 MaxScale 버전은 **2.5.29**였습니다.

근본 원인은 MaxScale이 backend 커넥션을 재조정(reassignment)할 때 기존 prepared statement handler를 새 커넥션으로 마이그레이션하지 못하는 것입니다. MaxScale은 prepared statement를 **세션 커맨드 히스토리**로 관리하는데, `max_sescmd_history` 설정값을 초과하면 히스토리가 pruning되면서 prepared statement가 유실될 수 있습니다.

이 문제는 MaxScale 측에서도 인지하고 있었습니다. [MXS-5302](https://jira.mariadb.org/browse/MXS-5302) 이슈로 추적되었고, **21.06.18, 22.08.15, 23.02.12, 23.08.8, 24.02.4** 버전에서 prepared statement가 `max_sescmd_history` 제한에서 제외되도록 수정되었습니다. 하지만 운영 환경의 2.5.29 버전에는 이 수정이 포함되어 있지 않았습니다.

MaxScale 업그레이드로도 해결 가능하지만, JDBC 설정 변경이 즉시 적용 가능한 안전한 조치였습니다. 프록시 버전 업그레이드는 별도의 인프라 변경 프로세스가 필요했기 때문입니다.

## 이 경험에서 배운 것

이 장애를 대응하면서 몇 가지를 다시 확인했습니다.

첫 번째는 **에러 메시지의 프로토콜 커맨드명을 정확히 읽어야 한다**는 것입니다. `COM_STMT_EXECUTE`와 `COM_STMT_BULK_EXECUTE`는 이름이 비슷하지만 완전히 다른 코드 경로에서 발생합니다. 1차 조치 후 에러가 "비슷해 보이지만 다른" 형태로 재발했을 때, 메시지를 정확히 읽은 것이 원인 추적의 시작점이었습니다.

두 번째는 **JDBC 드라이버 소스코드를 읽는 것이 가장 확실한 방법**이라는 것입니다. 공식 문서만으로는 `useServerPrepStmts`와 `useBulkStmts`가 독립적이라는 사실을 파악하기 어렵습니다. MariaDB Connector/J는 오픈소스이므로, batch 실행 관련 클래스(`BasePreparedStatement`, `ClientPreparedStatement`)의 분기 로직을 직접 확인하면 옵션 간 상호작용을 정확히 이해할 수 있습니다.

세 번째는 **"같은 인프라인데 왜 이 서비스만?"이라는 질문의 힘**입니다. 동일한 MaxScale을 사용하는 서비스들 중 product-api만 영향을 받은 이유를 추적한 것이 원인 범위를 좁히는 데 결정적이었습니다. 장애 발생 시 영향 범위를 먼저 파악하면, 서비스별 설정 차이라는 실마리를 빠르게 잡을 수 있습니다.

DB 프록시 환경에서 JDBC 설정을 잡을 때는, `useServerPrepStmts`와 `useBulkStmts`를 **모두** 확인해야 합니다. 하나만 끄면 나머지에서 여전히 server-side binary 프로토콜을 사용합니다. 프록시가 있다면 text protocol로 통일하는 것이 가장 안전한 출발점이고, 성능이 필요하다면 `useBatchMultiSend`(2.x) 또는 파이프라이닝(3.x)으로 보완할 수 있습니다.
