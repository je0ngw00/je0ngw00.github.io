---
title: "Jira 티켓에서 PR까지, Claude Code 스킬로 자동화하기 — jira-start와 jira-sprint 구축기"
date: 2026-02-28 00:00:00 +0900
categories: [AI, Claude Code]
tags: [claude-code, custom-skill, jira-automation, git-worktree, developer-experience]
---

## 반복되는 시작 의식

매일 아침 비슷한 순서로 시작합니다. Jira 열고, 오늘 할 이슈 확인하고, 브랜치명 고민하고, `git checkout -b fix/Q30-xxx`, Jira 상태를 "진행 중"으로 바꾸고. 이슈 하나에 5분 남짓이지만, 하루에 몇 번씩 반복하다 보면 은근히 쌓입니다.

컨텍스트 스위칭 비용이 실제로 상당합니다. 깊이 집중하던 작업에서 벗어나 Jira를 열고 다음 이슈를 파악하고 브랜치를 만드는 과정은, 걸리는 시간보다 집중력 전환 비용이 더 큽니다. PR 사이클 타임 중간값이 83시간이라는 통계가 있는데[^1], 이 83시간 중 26%가 첫 리뷰 대기 구간에서 나옵니다. 코드 작성 전후의 부수적인 작업들이 생각보다 많은 시간을 차지하고 있습니다.

더 흥미로운 역설도 있습니다. AI 도구 도입 이후 개발자들이 21% 더 많은 작업을 완료하고 PR을 98% 더 많이 생성하는데, 리뷰에 걸리는 시간은 오히려 91% 늘었다는 분석이 있습니다[^2]. AI가 생산성을 높이는 동시에 처리해야 할 PR 양도 폭발적으로 늘어나는 구조입니다. 시작 단계의 마찰을 줄이는 게 더 중요해진 이유입니다.

그래서 이 시작 단계를 스킬로 만들어봤습니다.

[^1]: [Atlassian Engineering Blog - How we cut PR cycle time with AI code reviews](https://www.atlassian.com/blog/announcements/how-we-cut-pr-review-cycle-time-with-ai)
[^2]: [byteiota / SonarSource - AI code review bottleneck](https://byteiota.com/ai-code-review-bottleneck-kills-40-of-productivity/)

## Custom Skill이란

Claude Code의 Custom Skill은 `~/.claude/commands/` 디렉터리에 마크다운 파일로 파이프라인을 정의하는 방식입니다. `/my:jira-start` 같은 명령어로 호출하면 Claude가 스킬 파일을 읽고 단계별로 실행합니다. 단순히 Claude에게 매번 "Jira 연동해서 브랜치 만들어줘"라고 요청하는 것과 다릅니다. 스킬 파일 안에 인증 방식, API 엔드포인트, 오류 처리, UX 출력 형식까지 모두 정의해두면, 호출할 때마다 일관되게 동작합니다.

스킬 파일 옆에는 `references/` 디렉터리를 두고 재사용 가능한 패턴을 모아둡니다. Bitbucket 스킬을 만들면서 발견한 것들 — Atlassian API 토큰은 `Bearer`가 아닌 Basic Auth만 작동한다는 것, `curl` 결과를 파이프로 넘기면 파이썬 `stdin`이 비어버리니 임시 파일을 써야 한다는 것 — 을 `shell-env-rules.md`로 정리해뒀더니, Jira 스킬을 만들 때 그대로 참조할 수 있었습니다.

처음 Bitbucket 스킬에서 인증 문제로 3 세션을 디버깅했는데, Jira 스킬에서는 동일한 문제가 발생하지 않았습니다. 한 번 해결한 인프라 문제의 반복 비용이 0이 되는 구조입니다. 스킬이 쌓일수록 다음 스킬이 빨라집니다.

사실 지금 사용하는 스킬은 `jira-start`와 `jira-sprint`만이 아닙니다. 커밋을 담당하는 `/my:commit`, 빌드 검증과 push를 담당하는 `/my:finalize`, PR 리뷰 반영을 담당하는 `/my:pr-review-apply`까지 직접 만들어 이어 쓰고 있습니다. 이번 포스팅에서는 개발 사이클의 시작 단계인 `jira-start`와 `jira-sprint`를 중심으로 다룹니다.

## jira-start: 이슈 선택부터 브랜치 생성까지

`/my:jira-start` 스킬의 파이프라인은 7단계입니다.

```
Auth → Fetch Issues → Select → Branch → Transition → Implement → Summary
```

호출하면 Jira 현재 스프린트 이슈 목록을 가져오고, 선택한 이슈로 브랜치를 만들고, 상태를 "진행 중"으로 전환합니다. `--implement` 플래그를 붙이면 이슈 description을 읽고 구현까지 위임합니다. `--plan` 플래그는 구현 전 접근 방식을 먼저 분석해서 보여줍니다.

### Jira API v3에서 만난 것들

스킬을 만들면서 Jira REST API v3의 비직관적인 부분들을 여럿 만났습니다.

**410 Deprecated**: `GET /rest/api/3/search` 엔드포인트가 이미 deprecated 상태입니다. `/rest/api/3/search/jql`을 사용해야 합니다. 문서를 안 보고 직접 호출해보니 바로 막혔습니다. 응답에 `total` 필드도 없어서 페이징을 `isLast` 기반으로 처리하도록 바꿔야 했습니다.

**ADF 파싱**: v3에서 이슈 description이 플레인 텍스트가 아니라 Atlassian Document Format(ADF)이라는 JSON 트리로 내려옵니다. 처음에는 description을 그대로 출력했더니 JSON 구조가 그대로 노출됐습니다. 재귀적으로 텍스트 노드를 추출하는 함수를 만들어서 처리했습니다.

```python
def extract_adf_text(node):
    if isinstance(node, str):
        return node
    if isinstance(node, dict):
        if node.get("type") == "text":
            return node.get("text", "")
        return " ".join(extract_adf_text(c) for c in node.get("content", []))
    return ""
```

**Transition ID 동적 탐색**: 이슈 상태를 "진행 중"으로 바꾸려면 Transition ID가 필요한데, 이 ID가 프로젝트 워크플로우마다 다릅니다. 하드코딩하면 다른 프로젝트에서 스킬이 깨집니다. `GET /rest/api/3/issue/{key}/transitions`로 전체 목록을 가져온 뒤 `to.name`에 "진행 중"이 포함된 항목을 찾아 동적으로 사용하도록 했습니다.

이런 내용들을 `references/jira-api.md`에 정리해뒀습니다. 다음 세션의 Claude가 같은 문제를 다시 겪지 않도록 하는, AI를 위한 문서화입니다.

### 이슈 목록 UX를 다듬은 과정

처음 버전은 이슈 Summary를 45자로 잘라서 보여줬습니다. 쓰다 보니 어떤 티켓인지 파악이 안 돼서 계속 Jira를 따로 열어야 했습니다. 스킬을 쓰면서 Jira를 여는 횟수가 줄어야 하는데, 오히려 확인하러 열게 되는 상황이 생긴 겁니다.

개선했습니다. Full summary를 보여주고, 정렬 기준도 바꿨습니다. "해야 할 일" 상태 이슈가 상단에 오도록 해서, 오늘 시작할 이슈를 바로 선택할 수 있게 했습니다. `--first` 플래그를 붙이면 목록 최상단 이슈를 자동 선택합니다. 아침에 스프린트 최우선 이슈로 바로 시작하는 패턴에 맞습니다.

스킬도 쓰면서 다듬는 과정이 있습니다. 처음부터 완벽할 필요는 없고, 불편함을 느낀 시점에 고치면 됩니다.

## jira-sprint: 여러 이슈를 워크트리로 동시에

`/my:jira-sprint`은 `jira-start`의 확장입니다. 단일 이슈가 아니라 스프린트에서 여러 이슈를 선택해 워크트리별로 동시에 구현합니다.

Git Worktree를 활용하는 방식입니다. 하나의 저장소에서 이슈별로 독립된 디렉터리를 만들고, 각 워크트리에서 에이전트가 병렬로 실행됩니다. Git Worktree 개념 자체는 [이전 포스트](/posts/git-worktree-claude-code/)에서 다뤘으니, 여기서는 스킬에서 어떻게 활용하는지에 집중합니다.

```
/my:jira-sprint --implement
    ↓
Q30-520 → ../project-Q30-520 → Agent A (독립 실행)
Q30-521 → ../project-Q30-521 → Agent B (독립 실행)
Q30-522 → ../project-Q30-522 → Agent C (독립 실행)
    ↓ (각 워크트리에서)
구현 완료 → /my:commit → /my:finalize → PR 생성
```

핵심 설계 원칙은 "Each worktree = independent PR"입니다. 이슈 간 의존성이 없는 작업들이라면 에이전트 여러 개를 돌려놓고 각각 독립된 PR을 만듭니다. 최대 5개 워크트리로 제한을 뒀는데, 그 이상이면 관리 비용이 구현 속도 이득을 넘어서더군요.

실제로 써보니 달라지는 점이 있습니다. 에이전트 A가 Q30-520을 구현하는 동안, 저는 Q30-521 워크트리로 가서 에이전트 B가 진행한 내용을 검토합니다. 기다리는 시간이 다른 이슈를 챙기는 시간이 됩니다. 지금 이 글을 쓰는 동안에도 워크트리 두 개에서 이슈가 돌아가고 있습니다.

한 가지 주의할 점은 이슈 description 품질입니다. description이 구체적일수록 에이전트의 구현 품질이 높아집니다. 모호한 이슈는 `--plan` 플래그로 먼저 접근 방식을 분석하게 하고, 방향이 맞는지 확인한 뒤 구현을 넘기는 흐름이 더 안전합니다.

## 스킬 체이닝

지금까지 이야기한 스킬들은 모두 개발 사이클의 각 구간을 담당하도록 직접 만든 것들입니다. 하나의 스킬이 다음 스킬의 시작점을 만들어주는 방식으로 연결됩니다.

```
/my:jira-start   → 이슈 선택 + 브랜치 생성 + Jira 상태 전환
        ↓
  구현 (직접 또는 --implement로 에이전트 위임)
        ↓
/my:commit       → 변경사항 분석 → 논리적 그룹핑 → conventional commits 자동 작성
        ↓
/my:finalize     → 코드 정리 → 품질 검토 → 빌드 검증 → push
        ↓
      PR 생성 (Bitbucket)
```

`commit` 스킬은 스테이징된 변경사항을 분석해서 논리적 단위로 나누고 conventional commit 형식으로 메시지를 작성합니다. `finalize` 스킬은 코드 품질을 검토하고 빌드가 통과하는지 확인한 뒤 push까지 합니다. 각 스킬은 하나의 책임만 갖고, 연결 지점은 Git 브랜치와 Jira 이슈 키로 자연스럽게 이어집니다.

써보면서 달라진 게 있다면, 개발 시작과 마무리의 "부수적인 작업"에 쓰던 인지 에너지가 줄었습니다. 브랜치명을 어떻게 할지, Jira 상태를 바꿨는지, 커밋 메시지를 어떻게 쓸지 같은 것들을 신경 쓰는 대신, 이슈를 읽고 구현 방향을 고민하는 데 집중할 수 있게 됐습니다. 개발자가 판단해야 할 것들에 좀 더 집중하게 된 것에 가깝습니다.

PR을 올리고 나면 이번엔 리뷰가 기다립니다. CodeRabbit 같은 AI 리뷰 봇이 즉시 코멘트를 달아주는데, 이 코멘트를 하나씩 열어 코드와 대조하고, 수정하고, 빌드 확인하고, 다시 push하는 과정도 반복 패턴입니다. 이 구간을 어떻게 자동화할 수 있는지는 다음 편에서 이어가겠습니다.
