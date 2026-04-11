---
title: "개발 생산성을 높이는 커스텀 MCP 서버 만들기"
date: 2026-04-11 00:00:00 +0900
categories: [AI, Claude Code]
tags: [claude-code, mcp, fastmcp, swagger, dev-tools]
---

외부 연동 API를 개발할 때마다 반복되는 루틴이 있었습니다. Swagger UI를 열고, 그룹을 선택하고, API를 찾고, 파라미터와 응답 스키마를 확인한 다음, 복사해서 Claude에 붙여넣고 설명하는 과정. 매번 같은 작업인데, 매번 컨텍스트가 끊깁니다.

이 반복을 없앨 방법을 찾다가 MCP 서버를 직접 만들게 되었습니다. 도구 9개, 코드 약 300줄로 Swagger 검색부터 DB 조회, API 호출까지 대화 흐름 안에서 처리할 수 있게 되었습니다.

## MCP란

**Model Context Protocol(MCP)**은 AI 모델이 외부 도구와 데이터에 접근하는 표준 프로토콜입니다. 2025년 12월 Linux Foundation 산하 [Agentic AI Foundation에 합류](https://blog.modelcontextprotocol.io/posts/2025-12-09-mcp-joins-agentic-ai-foundation/)하면서 월 9,700만 SDK 다운로드를 돌파했고, 2026년 4월 현재 [PyPI 기준 월 1억 건 이상](https://pypistats.org/packages/mcp)을 기록하고 있습니다. [mcp.so](https://mcp.so/)에 등록된 서버만 19,000개가 넘고, ChatGPT, Claude, Cursor, Gemini, VS Code 등 주요 도구들이 모두 지원합니다.

핵심 개념은 두 가지입니다. **Tool**은 AI가 호출할 수 있는 함수이고, **stdio 전송**으로 로컬 프로세스 간 통신합니다. Claude Code에서는 `.mcp.json`에 서버를 등록하면 Claude가 해당 도구들을 자동으로 인식하고 필요할 때 호출합니다. 도커도 별도 서버도 필요 없이, Claude Code가 직접 프로세스를 실행하고 세션이 끝나면 함께 종료됩니다.

이미 GitHub PR 관리, Slack 연동, DB 조회, 브라우저 자동화([Playwright MCP](https://github.com/anthropics/mcp-playwright)) 등 다양한 공개 MCP 서버가 있습니다. 하지만 공개 서버는 범용적으로 설계되어 있어서, 프로젝트 고유의 환경이나 워크플로우에는 맞지 않는 부분이 생깁니다. 그 지점에서 커스텀 MCP 서버가 필요해집니다.

## 19,000개 서버가 있는데 왜 직접 만들었나

[Swagger-MCP(Vizioz)](https://github.com/Vizioz/Swagger-MCP), [AWS Labs OpenAPI MCP](https://awslabs.github.io/mcp/servers/openapi-mcp-server), [openapi-to-mcp](https://github.com/EvilFreelancer/openapi-to-mcp) 등 Swagger/OpenAPI 전용 MCP 서버는 이미 여러 개 존재합니다. 처음에는 이것들을 써보려 했습니다.

그런데 안 맞는 부분이 있었습니다.

**여러 도메인을 한 번에 검색할 수 없었습니다.** 기존 Swagger MCP 서버는 하나의 스펙 파일을 다루는 데 초점이 맞춰져 있습니다. 프로젝트에서 연동하는 도메인이 4개였고, 이 도메인들의 엔드포인트를 도메인 경계를 넘어 통합 검색해야 했습니다. "수강신청"이라고 검색하면 어떤 도메인에 있든 관련 API가 다 나와야 하는데, 도메인마다 별도 서버를 띄우는 건 오히려 복잡했습니다.

**환경별 예외를 처리하지 못했습니다.** 4개 도메인 중 3개는 springdoc 기본 경로(`/v3/api-docs/swagger-config`)를 사용했지만, 하나는 `/api-docs/swagger-config`로 커스텀되어 있었습니다. 범용 도구는 이런 예외를 모릅니다.

**Swagger만 필요한 게 아니었습니다.** 연동 개발에서는 "스펙 확인 → DB 데이터 검증 → 실제 API 호출"이 하나의 흐름입니다. Swagger MCP, DB MCP, HTTP MCP를 각각 따로 쓰면 컨텍스트 전환이 생기고, 그 자체가 마찰이 됩니다. 하나의 서버에서 Swagger 검색, DB 조회, API 호출을 모두 처리하도록 통합했습니다.

[FastMCP 공식 문서](https://gofastmcp.com/integrations/openapi)에서도 "자동 생성된 OpenAPI MCP 서버보다 큐레이션된 MCP 서버가 유의미하게 더 좋은 성능을 낸다"고 명시하고 있습니다. 범용 도구로 80%는 되지만, 나머지 20%가 매일 반복되는 마찰이면 직접 만드는 편이 빠릅니다.

반대로, GitHub, Slack, Jira 같은 표준 SaaS 연동이나 범용 DB 조회, 단일 OpenAPI 스펙 조회 정도라면 이미 공개된 서버로 충분합니다. 기존 서버를 먼저 써보고, 안 맞는 부분이 남아 있을 때 커스텀을 고려하면 됩니다.

## 설계하면서 내린 판단들

### "하면 안 되는 것"부터 정했다

MCP 서버를 만들 때 가장 먼저 고민한 건 "무엇을 할 수 있게 할까"가 아니라 **"무엇을 하면 안 되게 할까"**였습니다.

원칙은 **부작용이 없는 조회만 허용**하는 것이었습니다.

**DB는 읽기 전용만 허용했습니다.** 애플리케이션 레벨에서 SQL 키워드를 검증하는 것과, DB 세션 자체를 읽기 전용으로 설정하는 것을 조합하여 다중으로 방어하는 구조를 적용했습니다. 결과 행 수 제한이나 LIMIT 자동 추가 같은 보조 장치도 함께 넣었습니다.

**API는 멱등성이 보장되는 GET만 허용했습니다.** POST, PUT, DELETE 같은 부작용이 있는 요청은 도구 수준에서 원천 차단했습니다. 도구 이름 자체를 `api_get`으로 만들어서, 에이전트가 다른 HTTP 메서드를 시도할 여지를 없앴습니다. dev 환경으로만 제한하고, 부작용이 있을 수 있는 도메인은 호출 대상에서 아예 제외했습니다.

### 자동 탐색과 캐싱

Swagger 스펙을 수동으로 다운로드해서 넣는 구조는 처음부터 배제했습니다. 스펙이 변경될 때마다 수동 갱신하는 건 또 다른 반복 작업이 됩니다.

MCP 서버가 시작되면 각 도메인의 swagger-config 엔드포인트를 자동으로 호출하고, 그룹별 JSON 스펙을 다운로드하여 로컬에 캐시합니다. 전체 엔드포인트를 하나의 flat index로 구축해서 키워드 검색이 가능하도록 만들었고, TTL 기반으로 만료 시 자동 재다운로드합니다. 수백 개의 엔드포인트를 도메인 경계를 넘어 횡단 검색할 수 있다는 점이, Swagger UI에서는 얻을 수 없던 경험이었습니다.

### 프로필 기반 멀티 프로젝트

프로젝트가 바뀔 때마다 MCP 서버 설정을 수정하는 건 비효율적입니다. 환경변수 하나로 프로필을 전환하는 구조를 도입했습니다.

```json
{
  "mcpServers": {
    "dev-tools": {
      "command": "python",
      "args": ["/path/to/server.py"],
      "env": {
        "MCP_PROFILE": "project-a"
      }
    }
  }
}
```

프로필마다 DB 접속 정보, 도메인 URL, 인증 정보를 분리해두면 MCP 서버 코드 하나로 여러 프로젝트를 대응할 수 있습니다. `~/.claude/.mcp.json`에 글로벌로 한 번만 등록하면 모든 프로젝트에서 사용 가능합니다.

## 만들면서 부딪힌 것들

### FastMCP가 두 개다

"FastMCP"라는 이름의 프로젝트가 두 개 존재합니다. 하나는 공식 MCP Python SDK(`mcp` 패키지)에 내장된 `mcp.server.fastmcp.FastMCP`이고, 다른 하나는 [PrefectHQ가 관리하는 독립 프로젝트](https://github.com/PrefectHQ/fastmcp)(v3.2.3)입니다. 공식 SDK 내장 버전은 가볍고 안정적이지만 기능이 제한적이고, PrefectHQ FastMCP는 OpenAPI 프로바이더, 컴포넌트 버저닝, 인증 등 더 많은 기능을 제공합니다.

공식 SDK의 FastMCP로 시작하는 건 간단합니다:

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("my-tools")

@mcp.tool()
def hello(name: str) -> str:
    """인사를 합니다."""
    return f"안녕하세요, {name}님!"

if __name__ == "__main__":
    mcp.run()
```

한 가지 주의할 점이 있습니다. 공식 SDK의 FastMCP 생성자에서 `version` 파라미터를 넘기면 TypeError가 발생합니다. 1.27.0(2026년 4월 기준 최신)에서도 지원하지 않으며, [AWS Labs MCP 서버들에서도 동일한 호환성 문제](https://github.com/awslabs/mcp/issues/996)가 보고된 바 있습니다. `instructions`, `website_url` 등 지원되는 인자는 사용할 수 있지만, `version`은 넘기지 않아야 합니다. FastMCP가 빠르게 진화하고 있어서, 공식 문서보다 실제 생성자 시그니처를 확인하는 습관이 필요합니다.

### springdoc 경로는 통일되어 있지 않다

4개 도메인 중 3개는 `/v3/api-docs/swagger-config`로 정상 동작했지만, 하나만 404를 반환했습니다. Swagger UI HTML 내 `swagger-initializer.js`를 확인해보니 `configUrl: "/api-docs/swagger-config"`로 커스텀되어 있었습니다. 기본 경로로 시도하고 실패하면 대체 경로로 재시도하는 다중 경로 시도 로직을 추가하여 해결했습니다. 프로젝트마다 설정이 다를 수 있다는 점을 전제로 설계해야 한다는 교훈이었습니다.

### SHOW TABLES에 LIMIT를 붙이면 안 된다

DB 도구에 LIMIT 자동 추가 로직을 넣었는데, `SHOW TABLES LIMIT 5`가 MariaDB에서 문법 오류를 일으켰습니다. `SHOW` 계열 명령어는 `LIMIT`을 지원하지 않기 때문입니다. SELECT 문에만 LIMIT를 추가하도록 조건 분기를 넣어 해결했지만, "모든 SQL에 동일한 규칙을 적용할 수 있다"는 가정이 틀렸다는 걸 확인한 사례였습니다.

### settings.json에 mcpServers를 넣으면 안 된다

Claude Code의 `settings.json`에 `mcpServers` 필드를 넣으면 스키마 검증 실패가 발생합니다. MCP 서버 등록은 반드시 `.mcp.json`(프로젝트별 또는 `~/.claude/.mcp.json` 글로벌)을 사용해야 합니다. 사소하지만, 처음 설정할 때 시간을 좀 썼습니다.

## 돌아보며

MCP 서버는 거창한 인프라가 아닙니다. 도커도 필요 없고, 별도 서버도 필요 없습니다. Python 파일 몇 개와 `.mcp.json` 등록 한 줄이면 됩니다.

만들면서 가장 시간을 쓴 건 도구 구현이 아니라 경계 설정이었습니다. DB는 읽기 전용, API는 멱등성이 보장되는 GET만, 환경은 dev만. 에이전트에게 도구를 쥐어주는 것은, 동시에 도구의 경계를 정하는 것이기도 합니다. 기존 MCP 서버로 해결되면 그걸 쓰면 되고, 안 되면 만들면 됩니다. 만드는 건 생각보다 쉽습니다.

Swagger UI를 마지막으로 연 게 언제인지 기억이 안 납니다.
