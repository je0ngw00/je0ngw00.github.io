---
title: "cmux로 AI 에이전트 3개를 동시에 굴리기 — Claude Code, Codex, Gemini interactive 협업"
date: 2026-03-22 00:00:00 +0900
categories: [AI, Claude Code]
tags: [cmux, multi-agent, claude-code, codex, gemini-cli, interactive-mode, developer-experience]
---

Claude Code로 작업하다 보면 혼자서는 부족한 순간이 옵니다. 코드 정확성은 Codex가 더 꼼꼼하고, 최신 트렌드 파악은 Gemini가 더 빠릅니다. 그래서 여러 AI 에이전트를 동시에 돌리는 "agentmaxxing"이 2026년 개발자 사이에서 자연스러운 패턴이 되고 있습니다.

문제는 여러 에이전트를 동시에 돌리면 각각이 뭘 하고 있는지 보이지 않는다는 점입니다. 명령을 보내고 결과만 받으면, 에이전트가 잘못된 방향으로 가도 끝날 때까지 모릅니다. cmux를 쓰면 이 문제가 달라집니다. 각 에이전트의 사고 과정을 실시간으로 보면서 필요할 때 개입할 수 있습니다.

이 글에서는 cmux를 소개하고, Claude Code가 Codex와 Gemini에게 일을 시키고 결과를 종합하는 interactive 협업 스킬을 실제로 만들어본 경험을 공유합니다.

## cmux: AI 에이전트 전용 터미널

cmux는 AI 코딩 에이전트를 여러 개 동시에 실행하는 개발자를 위한 macOS 네이티브 터미널 앱입니다. Ghostty의 libghostty를 기반으로 GPU 가속 렌더링을 지원하고, Swift + AppKit으로 작성되어 Electron 없이 가볍게 동작합니다. MIT 라이선스로 오픈소스이고, 2026년 3월 기준 GitHub에서 8,600개 이상의 star를 받았습니다[^1].

터미널 앱이 많은데 cmux가 다른 점은 **AI 에이전트 워크플로우에 특화**되어 있다는 것입니다.

**소켓 API와 CLI**가 핵심입니다. `cmux send`로 특정 pane에 텍스트를 보내고, `cmux read-screen`으로 해당 pane의 출력을 읽을 수 있습니다. 이것이 에이전트 간 메시지 전달의 기반이 됩니다. 단순히 화면을 분할하는 게 아니라, 프로그래밍 방식으로 pane을 제어할 수 있습니다.

**알림 링**은 에이전트가 입력을 기다리고 있을 때 해당 탭에 파란색 링을 표시합니다. 에이전트 5개를 동시에 돌리고 있어도 어느 쪽에 주의를 줘야 하는지 바로 보입니다.

**세로 탭 사이드바**에는 각 워크스페이스의 git 브랜치, 작업 디렉토리, 포트, 최근 알림이 한눈에 표시됩니다.

tmux와 비교하면 차이가 명확합니다.

| | tmux | cmux |
|---|---|---|
| pane 분리 | 가능 | 가능 |
| 입력 전송 | `tmux send-keys` | `cmux send --surface <ref>` |
| 화면 읽기 | `tmux capture-pane` | `cmux read-screen --surface <ref>` |
| 알림 시스템 | 제한적 | OSC 시퀀스 + CLI 알림 + 파란 링 |
| AI 에이전트 특화 | 없음 | 사이드바 메타데이터, 내장 브라우저 |

tmux도 `send-keys`와 `capture-pane`으로 비슷한 자동화가 가능합니다. 하지만 cmux는 에이전트 상태 추적(알림 링, 사이드바)과 구조화된 CLI(`--surface` 지정, JSON 소켓 API)를 네이티브로 제공합니다. 커스터마이징의 자유도는 tmux가 더 높고, AI 에이전트 워크플로우에 맞춘 out-of-the-box 경험은 cmux가 더 좋습니다.

## 배경: non-interactive의 한계

여러 AI 에이전트를 활용하는 가장 간단한 방법은 non-interactive 모드입니다.

```bash
codex --full-auto "analyze this codebase for security issues"
gemini -p "compare Redis vs Memcached for session caching"
```

Codex의 `--full-auto`는 자율 실행 모드이고, Gemini CLI의 `-p`는 non-interactive 스크립팅 모드입니다. 명령을 보내면 에이전트가 작업하고 결과를 돌려줍니다.

이 방식으로 충분할 때가 많습니다. 단순한 질문이나 잘 정의된 작업이라면 결과만 받으면 됩니다.

문제는 복잡한 작업에서 나타납니다. 에이전트가 잘못된 가정 위에서 작업을 이어가고 있어도, 결과가 나올 때까지 알 수 없습니다. 블로그 리서치를 위해 Codex에게 "멀티 에이전트 협업 패턴을 분석해달라"고 보냈는데, 엉뚱한 방향으로 30분을 탐색한 후에야 결과를 받은 적이 있습니다. 중간에 방향을 잡아줄 수 있었다면 시간과 토큰 모두 절약했을 겁니다.

2026년 Sonar의 개발자 설문에 따르면 개발자의 96%가 AI가 생성한 코드를 완전히 신뢰하지 않는다고 합니다[^2]. 결과만 받아서는 신뢰 판단 자체가 어렵습니다. 에이전트가 **어떤 근거로** 그 결론에 도달했는지 보여야 신뢰할 수 있습니다.

## interactive로 바꾸니 달라진 것

cmux를 쓰면 에이전트의 작업 과정을 실시간으로 볼 수 있습니다.

```bash
# pane 생성
cmux new-split right                    # Codex용 pane → surface:2
cmux new-split down --surface surface:2 # Gemini용 pane → surface:3

# 에이전트에 프롬프트 전송
cmux send --surface surface:2 "codex --full-auto 'analyze security patterns'\n"
cmux send --surface surface:3 "gemini -p 'compare caching strategies'\n"

# 진행 상황 확인
cmux read-screen --surface surface:2 --scrollback --lines 50

# 필요시 개입
cmux send --surface surface:2 "focus on SQL injection, not XSS\n"
```

이 방식이 non-interactive와 근본적으로 다른 점은 세 가지입니다.

**오류 수정 루프가 짧아집니다.** 에이전트가 잘못된 가정 위에서 작업을 시작하면, 결과물로 굳기 전에 잡을 수 있습니다. 위 예시에서 보안 분석 에이전트가 XSS에 집중하고 있는 걸 보고 SQL injection 쪽으로 방향을 바꿔줄 수 있습니다.

**신뢰 보정이 가능합니다.** 에이전트가 자신 있게 말하는 내용이 실제로 근거가 있는지 실시간으로 판단할 수 있습니다. "이 설정이 자동생성인지 모르겠다", "두 가지 구현 방식이 있다" 같은 불확실성의 순간이 보이면, 그 부분을 보강할 수 있습니다.

**동적 작업 분해가 됩니다.** 진행 상황을 보면서 태스크를 재분할하거나, 중복 작업을 발견하면 하나를 중단시키거나, 새로운 하위 작업을 추가할 수 있습니다. non-interactive에서는 처음에 정한 작업 분할이 끝까지 그대로 갑니다.

## 실전: 블로그 리서치 스킬에 적용

이 글을 쓰기 위한 리서치 자체를 cmux 멀티 에이전트로 진행했습니다. Claude Code가 오케스트레이터 역할을 하고, Codex와 Gemini에게 각각 다른 관점의 분석을 요청하는 구조입니다.

```
Claude Code (오케스트레이터)
├── cmux new-split right → Codex pane 생성
├── cmux new-split down  → Gemini pane 생성
├── cmux send            → 각 에이전트에 프롬프트 전송
├── cmux read-screen     → 진행 상황 폴링 (15초 간격)
├── 결과 종합             → cross-validation
└── cmux close-surface   → pane 정리
```

Codex에게는 기술적 정확성과 코드 패턴 분석을, Gemini에게는 업계 트렌드와 팩트 체크를 요청했습니다. 같은 주제를 다른 관점에서 분석하게 한 이유는 cross-validation 때문입니다. 두 에이전트가 동일한 결론에 도달하면 신뢰도가 높고, 다른 결론이 나오면 추가 검증이 필요한 부분을 빠르게 식별할 수 있습니다.

실제로 이번 리서치에서 유용했던 사례가 있습니다.

**라이선스 교정**: 리서치 문서 초안에 cmux 라이선스를 AGPL-3.0으로 적어뒀는데, Codex가 검증 단계에서 "GitHub 저장소에는 MIT 라이선스로 표시되어 있다"고 교정해줬습니다. 블로그에 잘못된 라이선스 정보를 올릴 뻔한 걸 잡아낸 겁니다.

**CLI 플래그 오류 즉시 발견**: 처음에 Codex에게 프롬프트를 보낼 때 `codex -a full-auto -q '...'` 형태로 보냈습니다. Codex pane에서 "unexpected argument '-q'" 에러가 바로 보였습니다. non-interactive였다면 실패한 결과만 받고 원인을 추적하느라 시간을 썼을 겁니다. pane을 보고 있었기 때문에 `codex --full-auto '...'`로 즉시 수정할 수 있었습니다.

**교차 확인**: Codex는 "cmux의 `read-screen` 명령이 공식 문서에서 확인되지 않으니 주의하라"고 지적했고, Gemini는 "cmux가 8,600 star를 넘겼다"는 최신 수치를 확인해줬습니다. 각 에이전트의 강점이 다르기 때문에 겹치게 분석시키되 가중치를 달리하면 더 정확한 결과를 얻을 수 있었습니다.

이 패턴을 Claude Code의 커스텀 스킬로 정의해두면 `/blog-research`나 `/blog-post` 호출 시 자동으로 cmux pane을 생성하고 에이전트에게 검증을 요청합니다. 공통 패턴(pane 생성/폴링/종료)은 `.claude/rules/cmux-multi-agent.md`로 분리해서 여러 스킬에서 재사용하고 있습니다.

## Trade-offs

멀티 에이전트 interactive 협업이 항상 좋은 건 아닙니다.

| 축 | 장점 | 단점 |
|---|---|---|
| **토큰 비용** | 각 에이전트가 좁은 컨텍스트로 집중 추론 | 에이전트 수만큼 프레이밍 비용 증가 |
| **컨텍스트 격리** | 워커가 서로 간섭 없이 집중 | 에이전트 간 의존성을 놓칠 수 있음 |
| **관찰 가능성** | 실시간 사고 과정 → 빠른 개입 | 인지 부하 증가, 여러 pane을 동시에 봐야 함 |
| **지연 시간** | 병렬 실행으로 wall-clock 시간 단축 | 조율 오버헤드 (pane 셋업, 결과 대기) |

10개 에이전트를 동시에 돌리면 한 세션에 $50-$100의 토큰 비용이 발생할 수 있습니다[^3]. 에이전트를 많이 돌린다고 무조건 좋은 게 아니라, 작업의 복잡도와 검증 필요성에 맞춰 에이전트 수를 조절해야 합니다.

**non-interactive가 여전히 나은 경우**도 있습니다. 잘 정의된 단순 작업, 검증이 필요 없는 일회성 질문, 또는 에이전트의 중간 과정을 볼 필요가 없는 배치 작업이라면 `-p` 플래그로 결과만 받는 게 효율적입니다. interactive는 복잡한 분석, 여러 관점의 교차 검증, 또는 에이전트가 잘못된 방향으로 갈 위험이 있는 작업에서 가치가 있습니다.

컨텍스트 격리에 대해서도 오해하기 쉬운 부분이 있습니다. cmux가 에이전트 간 모델 수준의 공유 메모리나 협업 의미론을 제공하는 건 아닙니다. 격리는 에이전트가 별도 프로세스에서 실행되기 때문에 자연스럽게 생기는 것이고, cmux는 이를 **관찰하고 조직하는 데** 도움을 줍니다.

## 정리

멀티 에이전트 협업에서 가장 큰 변화는 기술적인 것이 아니라 **관찰 가능성**입니다. 에이전트의 사고 과정이 보이면 개입 타이밍이 달라지고, 개입 타이밍이 달라지면 결과 품질이 달라집니다.

non-interactive에서 interactive로의 전환은 "결과를 나중에 평가하는 것"에서 "과정을 실시간으로 감독하는 것"으로의 변화입니다. 이건 속도만의 문제가 아니라 판단, 신뢰, 결과물의 품질에 영향을 줍니다.

cmux가 이 전환을 쉽게 만들어줍니다. `send`로 보내고 `read-screen`으로 읽고 `close-surface`로 정리하는 세 가지 명령이면 충분합니다. 복잡한 설정 없이, AI 에이전트를 터미널 pane에서 관찰 가능한 협업 파트너로 만들 수 있습니다.

[^1]: [cmux GitHub](https://github.com/manaflow-ai/cmux)
[^2]: [Sonar 2026 State of Code Developer Survey](https://www.sonarsource.com/resources/state-of-code-developer-survey-2026/)
[^3]: Gemini CLI 리서치 결과 기반 추정치. 에이전트 수, 프롬프트 복잡도, 모델에 따라 크게 달라질 수 있습니다.
