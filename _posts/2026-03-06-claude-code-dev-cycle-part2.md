---
title: "PR 리뷰를 AI가 읽고 고친다 — pr-review-apply와 Ralph Loop으로 무인 사이클 설계하기"
date: 2026-03-06 00:00:00 +0900
categories: [AI, Claude Code]
tags: [claude-code, pr-review, coderabbit, ralph-loop, closed-loop-development]
---

[지난 편](/posts/claude-code-dev-cycle-part1/)에서는 Jira 이슈를 선택하고, 브랜치를 만들고, 워크트리로 병렬 구현해서 PR까지 올리는 과정을 스킬로 자동화했습니다. 그런데 PR을 올리고 나면 이번엔 리뷰가 기다립니다.

PR 사이클 타임 중간값은 83시간입니다. 이 중 26%는 첫 리뷰 대기 구간에서 나옵니다.[^1] CodeRabbit 같은 AI 리뷰 봇이 있으면 즉시 코멘트가 달리니 이 구간이 줄어들 것 같지만, 실제로는 다른 병목이 생깁니다. AI 도입 후 개발자들이 PR을 98% 더 많이 생성하는데 리뷰 시간은 91% 늘었다는 분석이 있습니다.[^2] 리뷰 봇이 즉시 코멘트를 달아주더라도, 그 코멘트를 하나씩 열고 코드와 대조하고 수정하고 빌드 확인하고 다시 push하는 과정은 여전히 사람 몫입니다. 이 반복도 스킬로 만들 수 있지 않을까 생각했습니다.

[^1]: [Atlassian Engineering Blog - How we cut PR cycle time with AI code reviews](https://www.atlassian.com/blog/announcements/how-we-cut-pr-cycle-time-with-ai-code-reviews)
[^2]: [byteiota / SonarSource - AI code review bottleneck](https://byteiota.com/ai-code-review-bottleneck-kills-40-of-productivity/)

## CodeRabbit이 던진 공

CodeRabbit은 PR이 생성되면 자동으로 코드를 분석해서 inline 코멘트를 달아줍니다. severity 태그(Critical, Major, Minor)로 우선순위를 표시하고, diff 바깥의 관련 파일까지 참조해서 멀티라인 분석을 합니다. 그리고 한 가지 흥미로운 기능이 있습니다. 각 코멘트에 "Prompt for AI Agents" 코드 블록을 함께 제공합니다. AI 에이전트가 이 코멘트를 처리할 때 참고할 맥락을 AI 리뷰 봇이 직접 작성해주는 구조입니다.

CodeRabbit이 1,300만 건 이상의 PR을 리뷰한 통계에 따르면, AI 생성 코드의 이슈 밀도는 PR당 10.83건으로 사람이 작성한 코드(6.45건)의 약 1.7배입니다. AI가 코드를 더 빠르게 생성할수록 리뷰 대상도 늘어나는 구조입니다. 리뷰 봇이 던진 공을 매번 사람이 받아야 한다면, 병목은 리뷰 봇 이후 구간으로 옮겨갈 뿐입니다.

## /my:pr-review-apply: 리뷰 읽고 수정하고 빌드까지

`/my:pr-review-apply` 스킬의 파이프라인은 10단계입니다.

```
Auth → Fetch → Classify → Confirm → Branch → Apply → Build → Push → Merge → Summary
```

PR URL 하나로 호출하면 Bitbucket API를 통해 리뷰 코멘트 전체를 가져옵니다. 코멘트를 severity(Critical/Major/Minor/Info)와 domain(security/quality/code-style)으로 분류하고, 적용할 항목을 확인합니다. 선택 후에는 security-engineer, quality-engineer, code-simplifier가 각 도메인을 담당해 병렬로 수정을 적용합니다. 수정 후에는 반드시 빌드를 확인하고, 실패 시 자동으로 수정을 시도합니다.

실제로 써본 건 Kotlin/Spring Boot 프로젝트의 PR #36입니다. 긴글 업로드 관련 기능을 구현한 PR이었고, CodeRabbit이 5개 코멘트를 달았습니다. Critical 1개는 `HtmlTextSize.kt:37`의 `max * MAX_HTML_RATIO` 연산에 Int 오버플로 가능성이 있다는 것이었고, Major 4개는 HTML-only 입력(`<p><br/></p>`)이 `@NotBlank`를 통과하는 문제와 순수 텍스트 기준 검증 관련이었습니다.

스킬을 실행하니 코드를 읽고 "5개 항목 모두 이미 반영됨" 판정이 났습니다. 이전 커밋에서 `value.length.toLong() > max.toLong() * MAX_HTML_RATIO`로 수정되어 있었고, `@HtmlTextSize(min = 1, max = 5000)` 적용도 완료된 상태였습니다. "0 of 5 applied" 결과였지만, 이것도 의미 있는 실행이었습니다. 사람이 코멘트 5개를 하나씩 열어 코드와 대조하는 시간을 AI가 대신한 것입니다.

스킬을 구축하면서 겪은 시행착오도 있었습니다. Bitbucket API 인증에서 Bearer 토큰이 작동하지 않았습니다. Atlassian API 토큰 형식(`ATATT3x...`)은 Basic Auth만 지원합니다. 이 문제로 3 세션을 소비했고, 이전 세션에서도 같은 문제를 만났는데 references 파일에 기록이 없어 또 디버깅을 반복했습니다. 이 경험 이후 Bitbucket API 특성을 `references/bitbucket-api.md`에 정리해뒀고, 이제 같은 문제로 세션을 낭비하지 않습니다. 자동감지(v1) → 명시적 입력(v2) → PR URL 직접 입력(v3)으로 스킬이 진화한 것도 사용하면서 발견한 개선들이었습니다.

## Ralph Loop — 에이전트가 잠들지 않는 밤

Ralph Loop은 Geoffrey Huntley가 정립한 패턴입니다. 심슨 캐릭터 Ralph Wiggum에서 이름을 땄습니다. 핵심은 bash while-loop으로 Claude Code를 반복 호출하는 구조로, [이전 포스트](/posts/claude-code-ralph-loop/)에서 개념과 구현 방식을 자세히 다뤘습니다.

```bash
while true; do
  claude --prompt "프로젝트를 계속 진행하세요" --max-turns 50
  # Claude가 코드 작성 → 테스트 → 커밋 → 컨텍스트 소진 시 종료
  # 루프 재시작 → git log + 파일 상태에서 진행 상황 파악 → 이어서 작업
done
```

핵심 인사이트는 "진행 상태를 LLM 컨텍스트가 아닌 파일시스템과 git에 저장한다"는 것입니다. Claude의 컨텍스트 윈도우는 유한하지만, git 커밋 히스토리는 그렇지 않습니다. 매 iteration이 새로운 컨텍스트로 시작하더라도, 코드베이스와 커밋 로그가 이전 작업의 기억을 담당합니다. Huntley가 표현한 대로, "brute force meets persistence"입니다.

Anthropic 엔지니어링 블로그에서 에이전트 설계의 핵심을 이렇게 설명합니다. "Without structure, the model tries to do too much at once, loses track of state, and lacks feedback to self-correct." 외부 신호(컴파일러, 테스트, 린터)로 정확성을 판단하고, 진행 상태는 파일시스템에 저장하는 것이 안정적인 에이전트 구조입니다. `/my:finalize`와 `/my:pr-review-apply`가 빌드 검증을 필수 단계로 두는 것도 같은 이유입니다.

## 두 스킬의 결합 — 설계와 아직 남은 숙제

`/my:pr-review-apply`와 Ralph Loop을 연결하면 이런 흐름을 그릴 수 있습니다.

```
CodeRabbit 리뷰 생성
    ↓
/my:pr-review-apply → 리뷰 읽기 → 수정 → 빌드 → push
    ↓
새 코멘트 발생? → Ralph Loop으로 재트리거
    ↓
더 이상 새 코멘트 없음 → 사이클 종료
```

`pr-review-apply`를 Ralph Loop의 단일 iteration으로 쓰는 구조입니다. 각 iteration의 git 커밋이 이전 상태를 보존하고, 종료 조건은 "새 리뷰 코멘트 없음" 또는 "max iteration 도달"이 됩니다.

솔직히 말하면 이 결합은 아직 미완성 단계입니다. 현재 상태는 CodeRabbit 리뷰 생성 후 `/my:pr-review-apply`를 수동으로 실행하는 구조입니다. push 이후 새 리뷰를 자동으로 감지하고 Ralph Loop을 재트리거하는 폴링 레이어가 구현되어 있지 않습니다. 폴링 레이어 자체를 만드는 것은 어렵지 않지만, "언제 다시 확인할 것인가"와 "얼마나 기다려야 하는가"를 정의하는 부분이 문제입니다. CodeRabbit이 push 이후 재리뷰를 생성하는 데 걸리는 시간이 일정하지 않고, 너무 일찍 확인하면 빈 결과를 받고 사이클이 종료됩니다. 이 부분은 실험이 더 필요합니다.

비용도 현실적으로 고려해야 합니다. Ralph Loop 50회 iteration은 코드베이스 크기에 따라 $50-100 이상이 될 수 있습니다. Hard iteration cap, token budget ceiling, wall-clock timeout이 없으면 무인 사이클이 예상보다 훨씬 많은 비용을 소비합니다. 그리고 개발자의 96%가 AI 생성 코드의 기능 정확성을 완전히 신뢰하지 않는다는 조사 결과도 있습니다.[^3] 무인 사이클을 설계할 수 있지만, 최종 승인은 사람이 해야 합니다. "자동화"가 아니라 "가속화"입니다.

[^3]: [Sonar 2026 Developer Survey](https://events.sonarsource.com/2026-state-of-code-developer-survey/)

## 개발자의 역할이 달라지고 있다

2편에 걸쳐서 Jira 이슈 선택부터 PR 리뷰 반영까지 스킬로 연결하는 과정을 다뤘습니다. 각 구간에 스킬이 하나씩 붙어서, 개발 사이클의 반복적인 부분들을 처리합니다.

써보면서 달라진 건 코드 타이핑에 쓰는 시간보다, 인지 에너지의 쓰임이 바뀌었다는 점입니다. 브랜치명을 어떻게 할지, Jira 상태를 바꿨는지, 커밋 메시지를 어떻게 쓸지 같은 것들을 스킬이 담당하고, 이슈 description을 읽고 구현 방향을 결정하거나 리뷰 코멘트가 맥락에 맞는 판단인지 검토하는 데 집중하게 됐습니다. 개발자가 판단해야 할 것과 도구가 처리해야 할 것이 점점 명확해지는 느낌입니다.

시작점으로 가장 반복되는 워크플로우 하나를 스킬로 만들어보는 것을 권합니다. `references/` 디렉터리에 공통 문제 해결책을 기록해두는 것부터 시작하면, 다음 스킬을 만들 때 같은 문제를 다시 디버깅하지 않아도 됩니다. Ralph Loop은 안전장치(iteration cap, token budget, timeout)를 설정한 이후에 시도하는 것이 현실적입니다. 한 번 해결한 인프라 문제의 반복 비용이 0이 되는 구조가 조금씩 쌓이는 방식입니다.
