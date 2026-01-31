---
title: "Git Worktree + Claude Code: 터미널 여러 개로 동시에 개발하기"
date: 2026-01-31 09:00:00 +0900
categories: [AI, Claude Code]
tags: [claude-code, git-worktree, parallel-development, productivity]
---

## 하나의 기능, 여러 모듈

멀티모듈 프로젝트에서 하나의 기능을 개발할 때 여러 모듈을 동시에 건드려야 하는 경우가 있습니다. API 모듈, 도메인 모듈, 인프라 모듈을 순차적으로 작업하다 보면 "이거 동시에 하면 빠를 텐데"라는 생각이 듭니다.

Claude Code를 쓰면서 이 생각이 더 강해졌습니다. 한 모듈 작업을 Claude에게 맡기고 기다리는 동안 다른 모듈도 작업할 수 있으면 좋겠는데, 같은 디렉토리에서 여러 CLI를 돌리면 충돌이 날 수 있습니다.

이 문제를 **Git Worktree**로 해결할 수 있습니다.

## Git Worktree란

Git Worktree는 하나의 저장소에서 여러 브랜치를 **동시에 다른 폴더로** 체크아웃하는 기능입니다. Git 2.5부터 지원합니다.

보통은 브랜치를 전환하려면 `git checkout`이나 `git switch`를 씁니다. 근데 작업 중인 게 있으면 stash하거나 커밋해야 합니다. Worktree를 쓰면 이런 번거로움 없이 여러 브랜치를 각각 다른 폴더에서 동시에 작업할 수 있습니다.

### 기본 명령어

```bash
# worktree 생성 (새 브랜치와 함께)
git worktree add -b feature/new-feature ../project-feature

# worktree 생성 (기존 브랜치)
git worktree add ../project-hotfix hotfix/urgent-fix

# 목록 확인
git worktree list

# 삭제
git worktree remove ../project-feature
```

`-b` 옵션을 쓰면 새 브랜치 생성과 worktree 추가를 동시에 할 수 있습니다.

### 저장소 복제와의 차이

"그냥 저장소를 여러 번 clone하면 되는 거 아닌가?"라고 생각할 수 있습니다. 차이점이 있습니다.

| 방식 | 디스크 사용 | Git 히스토리 |
|-----|-----------|-------------|
| 저장소 복제 | 전체 복사 (무거움) | 각각 독립 |
| Git Worktree | `.git` 공유 (가벼움) | 공유 |

Worktree는 `.git` 디렉토리를 공유하기 때문에 디스크 공간을 훨씬 적게 씁니다. 브랜치와 커밋 히스토리도 공유되어서 한쪽에서 커밋하면 다른 쪽에서도 바로 보입니다.

## Claude Code와 조합하기

Claude Code는 터미널에서 실행되는 AI 코딩 도구입니다. 여러 인스턴스를 동시에 실행할 수 있다는 게 IDE 기반 도구와 다른 점입니다.

Worktree와 조합하면 이런 워크플로우가 가능합니다.

### 워크플로우

```bash
# 1. 메인 프로젝트에서 worktree 생성
cd my-project
git worktree add -b feature/user-auth ../my-project-command
git worktree add -b feature/user-auth ../my-project-query

# 2. 각 터미널에서 Claude Code 실행
# 터미널 1
cd ../my-project-command
claude

# 터미널 2
cd ../my-project-query
claude

# 3. 각각 다른 작업 지시
```

같은 브랜치(`feature/user-auth`)를 여러 worktree에서 쓸 수는 없습니다. 하지만 위 예시처럼 같은 브랜치 이름으로 생성하면, 나중에 하나로 합치기 편합니다. 실제로는 `feature/user-auth-command`, `feature/user-auth-query`처럼 나누거나, 같은 브랜치에서 서로 다른 폴더만 수정하는 방식으로 진행합니다.

### 병렬 작업의 핵심

여기서 중요한 건 **각 Claude Code 인스턴스가 서로 다른 파일을 수정해야 한다**는 점입니다. 같은 파일을 동시에 수정하면 나중에 머지할 때 충돌이 납니다.

그래서 이 방식은 모듈이 잘 분리된 프로젝트에서 효과적입니다.

## 실제 사례: CQRS 프로젝트

제가 실제로 써본 케이스는 CQRS 구조 프로젝트였습니다. Command(쓰기)와 Query(읽기)가 완전히 분리된 구조입니다.

```
my-project/
├── command/          # 쓰기 로직
│   └── user/
├── query/            # 읽기 로직
│   └── user/
└── shared/           # 공통 코드
```

사용자 인증 기능을 개발한다고 하면, Command 모듈에서는 회원가입/로그인 처리를, Query 모듈에서는 사용자 조회 API를 만들어야 합니다. 원래는 하나씩 순차적으로 했는데, Worktree로 동시에 진행해봤습니다.

```bash
# worktree 생성
git worktree add ../project-command main
git worktree add ../project-query main

# 터미널 3개 열고 각각 Claude Code 실행
# 터미널 1: command 모듈 작업
# 터미널 2: query 모듈 작업
# 터미널 3: shared 모듈 작업 (필요시)
```

### 왜 충돌이 안 나는가

CQRS 구조의 장점이 여기서 드러납니다. Command와 Query는 애초에 서로 다른 디렉토리에 있고, 서로 참조하지 않도록 설계되어 있습니다. 같은 도메인 기능이라도 물리적으로 분리되어 있어서 동시에 작업해도 충돌 위험이 거의 없습니다.

멀티모듈 프로젝트도 마찬가지입니다. `api`, `domain`, `infra` 모듈이 분리되어 있다면 각 모듈을 다른 Claude에게 맡길 수 있습니다.

### 같은 티켓, 한 번에 개발

이 방식의 장점은 **하나의 티켓을 한 번에 끝낼 수 있다**는 점입니다. 예전에는 "Command 끝나면 Query 하고, Query 끝나면 테스트 작성하고..." 이런 식으로 순차 진행했습니다. 지금은 세 개를 동시에 돌려놓고, 중간중간 확인하면서 진행합니다.

체감상 개발 속도가 확실히 빨라졌습니다. 특히 Claude가 작업하는 동안 기다리는 시간이 줄었습니다.

## 주의사항

### 포트 충돌

각 worktree에서 로컬 서버를 띄우거나 테스트를 실행하면 포트가 충돌할 수 있습니다.

해결 방법:
- 테스트용 서버는 랜덤 포트 사용
- TestContainer 쓰면 `withRandomPort()` 옵션 활용
- 환경별로 포트를 다르게 설정

```java
// TestContainer 예시
@Container
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
    .withExposedPorts(5432);  // 랜덤 포트로 매핑됨
```

### 토큰 비용

Claude Code를 여러 개 돌리면 토큰 비용도 배로 듭니다. 3개 동시 실행하면 비용도 대략 3배입니다. 효율이 좋은 건 맞지만, 비용 대비 효과를 따져봐야 합니다.

저는 급한 기능 개발이나 마감이 빠듯할 때 주로 씁니다. 일상적인 작업에는 하나씩 순차적으로 하는 게 비용 면에서 낫습니다.

### 적합한 상황 vs 부적합한 상황

**적합한 상황:**
- 모듈이 명확히 분리된 프로젝트 (CQRS, 멀티모듈, MSA)
- 같은 기능의 여러 레이어를 동시에 개발할 때
- 시간이 촉박한 개발

**부적합한 상황:**
- 모듈 간 의존성이 강한 경우
- 같은 파일을 여러 곳에서 수정해야 하는 경우
- 여유 있게 진행해도 되는 작업

### 컨텍스트 분리

각 Claude Code 인스턴스는 서로의 작업을 모릅니다. 터미널 1에서 한 작업을 터미널 2의 Claude는 알 수 없습니다. 그래서 공통으로 수정해야 하는 부분이 있다면 한 곳에서만 작업하거나, 작업 순서를 조율해야 합니다.

## 편의 도구

Worktree 관리가 번거롭다면 도구를 활용할 수 있습니다.

- **Treekanga**: worktree 생성/삭제를 간편하게, Cursor/VSCode 연동 지원
- **Git Worktree Runner**: CodeRabbit에서 만든 래퍼, `new`, `ai`, `rm` 등 간단한 명령어 제공

```bash
# Treekanga 예시
treekanga add feature-branch  # 브랜치 존재 여부 자동 판단
treekanga delete              # 인터랙티브하게 선택 삭제

# Git Worktree Runner 예시
gw new feature-branch         # worktree 생성
gw ai                         # Claude Code 실행
```

직접 써보진 않았지만, worktree를 자주 쓴다면 검토해볼 만합니다.

## 마무리

Git Worktree + Claude Code 조합은 모듈이 잘 분리된 프로젝트에서 효과적입니다. 핵심은 **동시에 작업해도 충돌이 나지 않는 구조**를 갖추는 것입니다. CQRS나 멀티모듈처럼 물리적으로 분리된 구조라면 이 방식이 잘 맞습니다.

써보니까 "기다리는 시간"이 확실히 줄었습니다. Claude가 Command 모듈 작업하는 동안 Query 쪽 진행 상황을 확인하고, 필요하면 수정 지시를 내리고. 이런 식으로 병렬로 진행하니까 하나의 기능을 완성하는 시간이 단축됐습니다.

다만 비용은 신경 써야 합니다. 급할 때 쓰는 부스터 정도로 생각하면 될 것 같습니다.
