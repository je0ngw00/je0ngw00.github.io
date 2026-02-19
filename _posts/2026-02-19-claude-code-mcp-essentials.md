---
title: "Claude Code를 10배 강하게 만드는 MCP 6가지"
date: 2026-02-19 09:00:00 +0900
categories: [AI, Claude Code]
tags: [claude-code, mcp, tavily, context7, playwright, github, model-context-protocol]
---

## MCP가 뭔가요?

Claude Code는 혼자서도 꽤 많은 걸 합니다. 코드를 읽고, 수정하고, 터미널 명령어도 실행합니다. 그런데 기본 상태에서는 한계가 있습니다. 인터넷 검색을 할 수 없고, 최신 라이브러리 문서를 모르고, 브라우저를 직접 열 수도 없습니다.

MCP(Model Context Protocol)는 이 한계를 넘기 위한 방법입니다. Anthropic이 만든 표준 프로토콜로, Claude에게 외부 도구를 연결할 수 있게 해줍니다. USB 포트처럼, 꽂는 만큼 기능이 늘어납니다.

기존에 Claude Code에서 할 수 없었던 것들이 MCP를 통해 가능해집니다.

- **웹 검색** → Tavily MCP
- **최신 공식 문서 조회** → Context7 MCP
- **프로젝트 외부 파일 접근** → Filesystem MCP
- **복잡한 문제 단계별 분석** → Sequential Thinking MCP
- **브라우저 자동화** → Playwright MCP
- **GitHub 레포 직접 조회** → GitHub MCP

[이전 포스트](/posts/claude-code-superpowers/)에서 Claude Code에 개발 워크플로우를 심는 방법을 다뤘다면, 이번 포스트는 Claude Code의 감각 기관을 확장하는 도구들을 소개합니다.

## 설치 전 확인사항

MCP를 설치하려면 **Node.js 18 이상**이 필요합니다.

```bash
node -v  # v18.0.0 이상이어야 합니다
```

MCP는 두 가지 범위로 설치할 수 있습니다.

| 옵션 | 범위 | 언제 쓰나요 |
|------|------|------------|
| `-s user` | 모든 프로젝트에서 사용 가능 | 개인 도구 (Tavily, Context7 등) |
| 기본값 | 현재 프로젝트에서만 사용 | 프로젝트별 전용 도구 |

아래에서 소개하는 6개는 모두 `-s user`로 전역 설치하는 걸 권장합니다. 어느 프로젝트에서든 쓸 수 있게요.

설치 후에는 `claude mcp list`로 등록된 MCP를 확인할 수 있습니다.

---

## 1. Tavily — 실시간 웹 검색

**한 줄 소개**: Claude Code에 인터넷을 연결합니다.

Claude Code의 가장 큰 아쉬움은 인터넷 검색을 못 한다는 점입니다. 에러 메시지를 검색하거나, 최신 라이브러리 사용법을 찾거나, Stack Overflow를 참고하는 게 불가능합니다. 결국 개발자가 직접 탭을 열어서 검색하고 결과를 복붙해야 했습니다.

Tavily MCP를 연결하면 달라집니다. Claude가 직접 검색해서 결과를 코드에 반영합니다.

**설치**

[app.tavily.com](https://app.tavily.com)에서 API 키를 발급받습니다. 회원가입 후 바로 받을 수 있습니다.

```bash
claude mcp add tavily -s user \
  -e TAVILY_API_KEY=tvly-여기에키입력 \
  -- npx -y tavily-mcp@latest
```

API 키를 나중에 수정하려면 `~/.claude/mcp.json` 파일에서 직접 편집할 수 있습니다.

**실제로 써보니**

Spring 개발 중에 에러 디버깅할 때 효과가 확실합니다. `org.hibernate.LazyInitializationException: could not initialize proxy` 같은 에러를 만났을 때, Claude가 직접 검색해서 원인과 수정안을 함께 가져옵니다. `NoSuchBeanDefinitionException`처럼 스택 트레이스가 길어서 원인 파악이 어려운 경우에도 관련 이슈나 Stack Overflow 답변을 찾아서 바로 반영합니다.

예전에는 제가 탭을 열어서 검색하고 결과를 복붙했는데, 그 과정이 사라졌습니다.

> **팁**: API 키는 월 1,000건까지 무료입니다. 일반적인 개발 작업에서는 무료 한도 안에서 충분합니다.

---

## 2. Context7 — 공식 문서 실시간 조회

**한 줄 소개**: 항상 최신 공식 문서를 참고해서 코드를 작성합니다.

Claude의 학습 데이터에는 마감 시점이 있습니다. Spring Boot 3.x가 나오면서 `javax.*`가 `jakarta.*`로 바뀌고, Spring Security 6에서 `WebSecurityConfigurerAdapter`가 deprecated됐는데, Claude가 이전 방식으로 코드를 짜는 경우가 생깁니다. "왜 이 코드가 안 되지?" 싶어서 확인해보면 이미 바뀐 API일 때가 있습니다.

Context7 MCP는 이 문제를 해결합니다. 코드를 작성할 때 해당 라이브러리의 최신 공식 문서를 실시간으로 가져와서 참고합니다.

**설치**

API 키 없이 바로 사용할 수 있습니다.

```bash
claude mcp add context7 -s user \
  -- npx -y @upstash/context7-mcp@latest
```

**실제로 써보니**

Spring Boot 2.x에서 3.x로 마이그레이션 작업을 할 때 효과를 봤습니다. Context7 없이 작업하면 구버전 방식이 섞여 나오는 경우가 있었는데, 연결하고 나서는 공식 마이그레이션 가이드 기준의 코드가 일관되게 나왔습니다. Kotlin Coroutine과 Spring WebFlux를 함께 쓰는 패턴처럼 자료가 상대적으로 적은 주제에서도 공식 문서 기준으로 잡아줍니다.

> **팁**: 특정 라이브러리 문서를 참고해야 할 때는 명시적으로 알려주는 게 좋습니다. "Spring Security 6 공식 문서 참고해서 SecurityFilterChain 설정해줘"처럼요. 그냥 작업을 요청하면 Context7을 쓰지 않는 경우도 있습니다.

---

## 3. Filesystem — 프로젝트 외부 파일 접근

**한 줄 소개**: 지정한 폴더 안의 파일을 Claude Code가 자유롭게 읽고 씁니다.

Claude Code는 기본적으로 현재 작업 디렉토리 안의 파일만 접근합니다. 다른 프로젝트 폴더를 참고하거나, 다운로드 폴더의 파일을 읽어오려면 항상 내용을 직접 붙여넣어야 했습니다.

Filesystem MCP는 접근을 허용할 폴더를 지정해서 Claude Code가 자유롭게 파일을 읽고 쓸 수 있게 해줍니다.

**설치**

```bash
claude mcp add filesystem -s user \
  -- npx -y @modelcontextprotocol/server-filesystem \
  ~/Projects ~/Documents ~/Downloads
```

뒤에 나열한 경로가 접근 허용 폴더입니다. 본인 환경에 맞게 수정하면 됩니다.

**실제로 써보니**

여러 프로젝트를 동시에 참고할 때 편합니다. "A 프로젝트의 인증 구조 참고해서 B 프로젝트에 비슷하게 만들어줘" 같은 작업이 가능합니다. 또 다운로드된 API 명세서나 텍스트 파일을 Claude Code로 분석할 때도 씁니다.

> **팁**: 보안상 꼭 필요한 폴더만 등록하는 게 좋습니다. 홈 디렉토리 전체(`~/`)를 등록하면 편하지만, 예상치 못한 파일을 수정하는 상황이 생길 수 있습니다.

---

## 4. Sequential Thinking — 단계별 사고

**한 줄 소개**: 복잡한 문제를 즉흥적으로 답하지 않고 단계적으로 분석합니다.

Claude는 복잡한 문제에서도 빠르게 답을 내놓으려는 경향이 있습니다. 아키텍처 설계나 트리키한 버그처럼 꼼꼼하게 따져봐야 하는 상황에서도 즉흥적인 답이 나올 때가 있습니다.

Sequential Thinking MCP를 연결하면 Claude가 내부적으로 단계별 사고 과정을 거칩니다. 결론으로 바로 점프하지 않고, 문제를 분해하고, 가설을 세우고, 검증하는 과정을 밟습니다.

**설치**

API 키 없이 사용할 수 있습니다.

```bash
claude mcp add sequential-thinking -s user \
  -- npx -y @modelcontextprotocol/server-sequential-thinking
```

**실제로 써보니**

시스템 설계나 버그 원인 분석에서 차이가 납니다. "이 쿼리가 왜 느린지 분석해줘"라고 하면, 즉시 인덱스 추가를 제안하는 대신 실행 계획을 먼저 확인하고, 병목 구간을 특정한 다음, 해결책을 제시하는 순서로 진행합니다. 결론은 비슷해도 근거가 훨씬 탄탄해집니다.

> **팁**: 단순한 작업에는 효과가 미미합니다. "이 변수명 바꿔줘" 같은 건 굳이 단계별 사고가 필요 없습니다. 복잡한 분석이나 설계 작업에서 "순차적으로 분석해줘"라고 명시적으로 요청하는 게 좋습니다.

---

## 5. Playwright — 브라우저 자동화

**한 줄 소개**: 실제 브라우저를 열어서 클릭하고 테스트합니다.

프론트엔드 개발을 하면서 "이 버튼이 실제로 동작하는지 확인해줘"를 Claude Code에게 부탁할 수 있게 됩니다. Playwright MCP는 Claude가 실제 브라우저를 열고, 조작하고, 결과를 확인할 수 있게 해줍니다.

**설치**

```bash
claude mcp add playwright -s user \
  -- npx @playwright/mcp@latest
```

**실제로 써보니**

E2E 테스트 시나리오를 만들 때 잘 씁니다. "로그인 → 대시보드 이동 → 데이터 입력 → 저장 버튼 클릭" 흐름을 직접 실행해보고 Playwright 테스트 코드를 작성해줍니다. 수동으로 QA를 하던 부분을 위임할 수 있습니다.

또 "이 URL 열어서 콘솔에 어떤 에러가 나오는지 알려줘"처럼 디버깅에도 씁니다. 스크린샷을 찍어서 레이아웃 문제를 찾아주기도 합니다.

> **팁**: 처음 사용할 때 "playwright mcp를 사용해서"라고 명시적으로 말해야 합니다. 그냥 요청하면 bash로 playwright를 실행하려고 할 수 있습니다.

---

## 6. GitHub — 레포 직접 조회

**한 줄 소개**: GitHub 레포의 이슈, PR, 릴리즈 노트를 Claude가 직접 읽습니다.

Spring Boot 버전을 올리거나 라이브러리를 업데이트할 때, 릴리즈 노트나 마이그레이션 가이드를 직접 찾아보는 경우가 많습니다. GitHub MCP를 연결하면 Claude가 공식 레포를 직접 조회해서 변경사항을 확인합니다.

**설치**

[GitHub Settings → Developer settings → Personal access tokens](https://github.com/settings/tokens)에서 토큰을 발급받습니다. `repo` 권한이 있으면 충분합니다.

```bash
claude mcp add github -s user \
  -e GITHUB_PERSONAL_ACCESS_TOKEN=ghp_여기에토큰입력 \
  -- npx -y @modelcontextprotocol/server-github
```

**실제로 써보니**

Spring Boot 마이그레이션 작업에서 유용합니다. "spring-projects/spring-boot 레포에서 3.3 릴리즈 노트 확인해서 변경사항 정리해줘"라고 하면 공식 릴리즈 노트를 직접 읽고 요약해줍니다. Querydsl처럼 커뮤니티 레포에서 이슈를 직접 확인하거나, 쓰고 있는 라이브러리의 특정 버그 이슈가 해결됐는지 확인하는 데도 씁니다.

> **팁**: public 레포만 조회한다면 토큰 없이도 동작하지만, 속도 제한이 있습니다. 토큰을 발급해두는 편이 안정적입니다.

---

## 한눈에 보기

| MCP | 한 줄 설명 | API 키 | 없으면 |
|-----|-----------|--------|--------|
| Tavily | 실시간 웹 검색 | ✅ 필요 (무료 플랜 있음) | 직접 검색 후 복붙 |
| Context7 | 최신 공식 문서 참조 | ❌ 불필요 | 구버전 코드가 나올 수 있음 |
| Filesystem | 외부 폴더 파일 접근 | ❌ 불필요 | 파일 내용 직접 붙여넣기 |
| Sequential Thinking | 단계별 분석 | ❌ 불필요 | 즉흥적인 답변 |
| Playwright | 브라우저 자동화 | ❌ 불필요 | 수동 테스트 |
| GitHub | 레포 이슈/릴리즈 조회 | ✅ 필요 (무료) | 직접 GitHub 탭 열기 |

## 한 번에 설치하기

```bash
# 1. Tavily (API 키 필요 — app.tavily.com에서 발급)
claude mcp add tavily -s user \
  -e TAVILY_API_KEY=tvly-여기에키입력 \
  -- npx -y tavily-mcp@latest

# 2. Context7
claude mcp add context7 -s user \
  -- npx -y @upstash/context7-mcp@latest

# 3. Filesystem (폴더 경로는 본인 환경에 맞게 수정)
claude mcp add filesystem -s user \
  -- npx -y @modelcontextprotocol/server-filesystem \
  ~/Projects ~/Documents

# 4. Sequential Thinking
claude mcp add sequential-thinking -s user \
  -- npx -y @modelcontextprotocol/server-sequential-thinking

# 5. Playwright
claude mcp add playwright -s user \
  -- npx @playwright/mcp@latest

# 6. GitHub (토큰 필요 — github.com/settings/tokens에서 발급)
claude mcp add github -s user \
  -e GITHUB_PERSONAL_ACCESS_TOKEN=ghp_여기에토큰입력 \
  -- npx -y @modelcontextprotocol/server-github
```

설치 후 Claude Code를 재시작하면 바로 적용됩니다.

```bash
claude mcp list  # 설치된 MCP 목록 확인
```

## 마무리

결국 MCP는 "Claude가 혼자 못 하는 일을 대신 해주는 연결고리"입니다. 검색, 문서 조회, 파일 접근, 브라우저 조작. 이것들이 자연스럽게 연결되면 Claude Code가 꽤 다른 도구처럼 느껴집니다.

써보면서 가장 인상적이었던 건, 이 모든 게 그냥 대화 흐름 안에서 된다는 점입니다. 도구를 따로 켜거나 탭을 전환할 필요 없이, Claude가 필요한 순간에 알아서 씁니다.

다음 포스트에서는 MCP 서버를 직접 만드는 방법을 다룰 예정입니다. 외부에서 만들어둔 걸 가져다 쓰는 게 아니라, 내 도구를 직접 Claude에게 연결하는 방법입니다.
