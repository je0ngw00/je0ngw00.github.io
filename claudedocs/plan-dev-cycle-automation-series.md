# 작성 계획: Claude Code로 개발 사이클 자동화 (2편 시리즈)

> 상태: Part 1 완료, Part 2 대기 중
> 작성일: 2026-02-28
> 초안 위치: ~/Downloads/blog-draft-pr-review-ralph-loop.md, ~/Downloads/blog-draft-part2-pr-review-ralph-loop.md

---

## 시리즈 개요

| | Part 1 | Part 2 |
|---|---|---|
| 주제 | Jira 이슈 → 워크트리 병렬 구현 → PR | PR 리뷰 → 자동 반영 → Ralph Loop |
| 핵심 경험 | jira-start, jira-sprint 실무 사용 중 | pr-review-apply 실행 (PR #36) |
| 초안 완성도 | 80% (v0.4) | 60% (v0.1) |
| 목표 분량 | 3000자 내외 | 3500자 내외 |
| 발행 예정 | 2026-02-28 | 2026-03-07 |

**시리즈 관통 테마**: "반복 노동에서 의사결정으로 — 개발 사이클 전체를 스킬로 연결하기"

---

## Part 1 작성 계획

### 파일명 및 Front Matter

```yaml
파일명: 2026-02-28-claude-code-dev-cycle-part1.md
title: "Jira 티켓에서 PR까지, Claude Code 스킬로 자동화하기 — jira-start와 jira-sprint 구축기"
date: 2026-02-28 00:00:00 +0900
categories: [AI, Claude Code]
tags: [claude-code, custom-skill, jira-automation, git-worktree, developer-experience]
```

### 섹션 구조 (5개, 초안 7개에서 정리)

초안의 7개 섹션에서 구조를 정리합니다. 섹션 2(배경)와 섹션 5(스킬 체이닝)의 중복을 제거하고 핵심 경험에 집중합니다.

| # | 섹션 | 목표 분량 | 핵심 내용 |
|---|---|---|---|
| 1 | 리드 — "Jira 열고 브랜치 따고"의 반복 | 350자 | PR 83시간 통계 + 시작 단계의 마찰 |
| 2 | Custom Skill이란 + references/ 복리 효과 | 500자 | .claude/commands/ 구조, Bitbucket→Jira 재사용 |
| 3 | jira-start: 이슈 선택부터 브랜치까지 | 700자 | 7단계 파이프라인, API v3 디버깅, UX 개선 |
| 4 | jira-sprint: 워크트리로 동시에 여러 이슈 처리 | 700자 | 워크트리 병렬 구현 실무 사용 경험, Max 5 설계 이유 |
| 5 | 마무리 + Part 2 예고 | 350자 | 스킬 체이닝 맵 요약, 다음 편 연결 |

**총 목표: 2600자 본문 + 코드 블록**

### 핵심 결정사항

**포함 O**
- Jira API v3 디버깅 3종 (v2 410 deprecated, ADF 파싱, Transition ID 동적 탐색) — 실무 경험의 핵심
- UX 개선 스토리: Summary 45자 → full summary + 상태 우선 정렬
- references/ 복리 효과: Bitbucket shell-env-rules.md를 Jira 스킬에서 그대로 재사용
- jira-sprint 실무 사용 경험 — 현재 여러 이슈를 워크트리로 병렬 처리 중
- RESEARCH-1 통계: PR 83시간, AI PR 98% 증가 + 리뷰 91% 증가 역설

**포함 X**
- `/my:pr-review-apply`, Ralph Loop 상세 내용 → Part 2로
- 스킬 체이닝 전체 다이어그램 (`jira-start → commit → finalize → PR → review → Ralph Loop`) → Part 2에서 완성
- Part 1 마무리의 체이닝은 `jira-start → commit → finalize → PR 생성`까지만
- AI 코드 리뷰 시장 수치(CodeRabbit, Copilot 점유율) → Part 2
- Closed-Loop Development, Replit Agent 3 등 업계 트렌드 → Part 2

**톤 주의사항**
- jira-sprint는 실제 실무 사용 경험으로 서술 ("지금 이 글을 쓰는 동안에도 워크트리에서 여러 이슈를 병렬 처리 중")
- "스킬이 제품처럼 진화한다"는 관점 유지 (v1 → v2 개선 스토리)

---

## Part 2 작성 계획

### 파일명 및 Front Matter

```yaml
파일명: 2026-03-07-claude-code-dev-cycle-part2.md
title: "PR 리뷰를 AI가 읽고 고친다 — pr-review-apply와 Ralph Loop으로 무인 사이클 설계하기"
date: 2026-03-07 00:00:00 +0900
categories: [AI, Claude Code]
tags: [claude-code, pr-review, coderabbit, ralph-loop, closed-loop-development]
```

### 섹션 구조 (6개, 초안 8개에서 정리)

| # | 섹션 | 목표 분량 | 핵심 내용 |
|---|---|---|---|
| 1 | 리드 — PR 올린 후가 진짜 병목 | 350자 | Part 1 1문장 요약 + 첫 리뷰 대기 15시간 |
| 2 | CodeRabbit이 던진 공 | 500자 | AI 리뷰 생태계, "Prompt for AI Agents" 블록 강조 |
| 3 | pr-review-apply: 리뷰 읽고 수정하고 빌드까지 | 800자 | 파이프라인, PR #36 실제 결과, Auth 디버깅 3세션 |
| 4 | Ralph Loop: 에이전트가 잠들지 않는 밤 | 500자 | 개념, bash while-loop, git이 기억을 담당 |
| 5 | 두 스킬의 결합 — 설계와 안전장치 | 500자 | 결합 다이어그램, 비용/신뢰 트레이드오프 |
| 6 | 마무리 — 자동화가 아니라 가속화 | 350자 | 역할 변화, 실용적 시작점 권장 |

**총 목표: 3000자 본문 + 코드 블록**

### 핵심 결정사항

**포함 O**
- CodeRabbit의 "Prompt for AI Agents" 코드 블록 — AI가 AI를 위해 프롬프트를 생성하는 구조
- PR #36 실제 실행 결과: 5개 코멘트, Critical 1개(Int 오버플로), "0 of 5 applied" 판정
- HtmlTextSize 구체적 이슈: `value.length.toLong() > max.toLong() * MAX_HTML_RATIO` 수정
- Auth 디버깅 3세션: Bearer → Basic Auth, 자동감지 → 명시적 입력 → PR URL
- 비용 현실: 50 iteration = $50-100+
- 96% 개발자 AI 코드 신뢰 안 함 → "자동화가 아니라 가속화" 결론으로 연결

**포함 X (또는 경계 명확히)**
- Ralph Loop × pr-review-apply 결합은 **미완성 설계**로 솔직하게 서술
  - "이렇게 연결하면 됩니다"가 아니라 "이렇게 연결하려면 새 리뷰 감지(폴링) 문제가 남아 있습니다"
  - 기술적 도전을 숨기지 않는 것이 5~6년차 포스팅의 차별점
- Spotify 1500+ 에이전트, Replit Agent 3 등 업계 사례는 1-2줄로 축소 (주객 전도 방지)
- Closed-Loop Development를 별도 섹션으로 분리하지 않고 섹션 5에 자연스럽게 녹임

**섹션 5 서술 방향 (중요)**

Ralph Loop과 pr-review-apply를 연결하려면 "새 리뷰 코멘트가 생겼는지 감지"하는 폴링 레이어가 필요합니다. 이 부분이 아직 해결되지 않았고, 그 이유와 고민을 솔직하게 쓰는 것이 핵심입니다.

```
현재 구현 상태:
  CodeRabbit이 리뷰 생성 → /my:pr-review-apply 수동 실행 → 수정 → push

미완성 단계:
  push 후 새 리뷰 자동 감지 → Ralph Loop 재트리거
  └ 문제: Bitbucket webhook or polling 레이어 미구현
         얼마나 기다려야 하는지(리뷰 생성 시간) 불확실
         "새 코멘트 없음" 종료 조건 정의 필요
```

이 미완성 구간을 "다음 숙제"로 제시하면 독자에게 오히려 신뢰를 줍니다.


---

## 연결 전략

### 두 편의 자연스러운 연결

**Part 1 → Part 2 넘어가는 지점**
- Part 1 마무리: `finalize → PR 생성`까지 보여주고 "PR을 올린 다음이 또 다른 반복 노동의 시작입니다"로 끊기
- Part 2 시작: "지난 편에서 PR까지 자동화했습니다. 그런데 PR을 올리고 나면 리뷰가 기다립니다." (1문장 요약)

**시리즈 공통 메시지**
- Part 1의 결론: "시작과 끝을 자동화하면 개발자는 구현 그 자체에 집중할 수 있습니다"
- Part 2의 결론: "자동화가 아니라 가속화 — 판단은 여전히 사람의 몫입니다"

### 기존 포스트와 중복 방지

| 기존 포스트 | 중복 위험 요소 | 처리 방안 |
|---|---|---|
| 2026-01-17 ralph-loop | Ralph Loop 개념 설명 | Part 2 섹션 4에서 1문단 요약 + "자세한 내용은 이전 포스트 참고" 링크 |
| 2026-01-31 git-worktree | 워크트리 개념 | Part 1 섹션 4에서 개념은 생략, 바로 "스킬에서 어떻게 쓰이는지"로 |

---

## 글쓰기 원칙

- `~합니다`, `~입니다` 문체, `~요` 금지
- 주어를 "저는", "써보니", "확인해보니"로 — AI 작성 느낌 제거
- 수치는 반드시 출처 URL 각주 처리
- 코드 블록은 실제 동작하는 것만, 의사코드 금지
- 섹션당 독립 가치 — SNS 발췌 가능한 수준

---

## 작업 체크리스트

### Part 1
- [x] 계획 검토 및 확정
- [x] `_posts/2026-02-28-claude-code-dev-cycle-part1.md` 작성
- [x] 분량 확인 (3000자 내외 충족)
- [x] 기존 git-worktree 포스트 중복 확인 (이전 포스트 링크로 처리)
- [ ] PR 생성 (`add-post-dev-cycle-part1` 브랜치)

### Part 2
- [ ] Part 1 발행 확인 후 착수
- [ ] 섹션 5 "미완성 단계" 기술적 도전(폴링 레이어) 서술 확정
- [ ] `_posts/2026-03-07-claude-code-dev-cycle-part2.md` 작성
- [ ] 기존 ralph-loop 포스트 링크 삽입
- [ ] PR 생성 (`add-post-dev-cycle-part2` 브랜치)
