---
title: "Claude Code에 로컬 RAG를 붙여 정책과 코드의 갭을 찾은 경험"
date: 2026-04-18 00:00:00 +0900
categories: [AI, Claude Code]
tags: [claude-code, mcp, rag, chromadb, fastmcp, obsidian]
---

프로젝트가 커지면서 CLAUDE.md에는 핵심 패키지 구조와 설계 원칙만 남게 됩니다. 세부 비즈니스 규칙까지 담으면 매 턴마다 토큰이 새어 나가고, 그렇다고 빼면 "재신청할 때 이전 이력은 어떻게 처리하지?" 같은 질문에 Claude가 코드만 뒤지게 됩니다. 팀 Confluence에 정책 문서가 수십 페이지 있지만, 매번 브라우저를 열어 찾기는 번거로웠습니다.

그래서 로컬에 RAG(Retrieval-Augmented Generation) 환경을 직접 만들어 보기로 했습니다. Obsidian vault를 문서 저장소로 두고, ChromaDB에 임베딩을 쌓고, FastMCP로 Claude Code와 연결하는 구조입니다. 전체 구조는 빠르게 붙었지만 그사이 `transport="stdio"` 누락, 영어 임베딩 모델의 한국어 한계, 청킹 실험까지 작은 삽질이 세 번 있었고, 그 과정을 지나고 나서야 실제로 의미 있는 사용처가 보이기 시작했습니다. 같은 MCP지만 [개발 워크플로우용 커스텀 MCP](/posts/custom-mcp-server-for-developer-productivity/)와는 용도가 다른, "도메인 지식 검색" 전용 서버를 만들어 본 기록입니다.

## 기술 스택을 이렇게 골랐습니다

OpenAI embedding API처럼 외부 서비스를 쓰는 방법도 있었지만, 정책 문서 자체가 사내 자산이라 외부로 내보내기가 부담스러웠습니다. LlamaIndex나 Haystack 같은 프레임워크도 후보였는데, vault 규모가 수십 페이지 수준이라 프레임워크 의존성 없이 `sentence-transformers`와 `chromadb`만 직접 붙이는 편이 이해하기 쉬웠습니다. 그 외 구성 요소는 "로컬에서 돌고, 설치가 단순하며, 한국어 문서를 다룰 수 있을 것"을 기준으로 골랐습니다.

| 구성 요소 | 선택 | 이유 |
|-----------|------|------|
| 문서 저장소 | Obsidian Vault (`~/obsidian-vault/`) | Markdown 기반. 구조가 단순하고, 나중에 Obsidian GUI로도 열 수 있음 |
| 벡터 DB | ChromaDB | 로컬 파일 기반, 설치가 간단하고 Python 네이티브 |
| 임베딩 모델 | `BM-K/KoSimCSE-roberta-multitask` | 한국어 특화 모델. 처음엔 영어 모델로 시작했다가 갈아 끼웠습니다 |
| MCP 서버 | Python FastMCP | Claude Code와 stdio 통신. 도구 8개 노출 |
| 문서 소스 | Confluence + 수동 작성 | Atlassian MCP로 읽어서 vault에 저장하는 구조 |

vault 디렉터리는 다음과 같이 잡았습니다.

```
~/obsidian-vault/
├── confluence/        # Confluence에서 동기화한 정책 문서
│   ├── api/           # API 표준
│   └── service/       # 서비스 정책
├── policies/          # 직접 작성한 정책
├── sessions/          # 세션 자동 기록 (Hook)
├── mcp_server.py      # MCP 서버
├── .venv/             # Python 가상환경
└── .chroma/           # 벡터 DB
```

## MCP 서버 기본 구조

FastMCP는 FastAPI와 비슷한 데코레이터 방식으로 도구를 등록합니다. 기본 구조는 아래와 같습니다.

```python
from sentence_transformers import SentenceTransformer
from mcp.server.fastmcp import FastMCP
import chromadb

mcp = FastMCP("knowledge-vault")

model = SentenceTransformer("BM-K/KoSimCSE-roberta-multitask")

client = chromadb.PersistentClient(path="~/obsidian-vault/.chroma")
collection = client.get_or_create_collection(
    name="knowledge",
    metadata={"hnsw:space": "cosine"},
)

@mcp.tool()
def search(query: str, top_k: int = 5) -> str:
    """자연어로 vault 문서를 검색합니다."""
    embedding = model.encode(query).tolist()
    results = collection.query(
        query_embeddings=[embedding],
        n_results=top_k,
        include=["documents", "metadatas", "distances"],
    )
    # distances는 cosine distance (1 - similarity)
    ...

@mcp.tool()
def index_vault() -> str:
    """vault의 모든 Markdown 파일을 청킹해서 벡터 DB에 인덱싱합니다."""
    ...

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

ChromaDB의 `distances`는 유사도 점수가 아니라 cosine distance입니다. 실제 유사도는 `1 - distance`로 계산해야 합니다. 최근 Chroma 문서는 `configuration={"hnsw": {"space": "cosine"}}` 형태를 권장하지만, `metadata` 방식도 아직 동작합니다.

최종적으로 만든 도구는 8개입니다.

| 도구 | 기능 |
|------|------|
| `index_vault` | 문서를 청킹해서 벡터 인덱싱 |
| `search` | 시맨틱 검색 |
| `read_document` | 특정 문서 전문 조회 |
| `list_documents` | 문서 목록 |
| `confluence_add_space` | 동기화 스페이스 등록 |
| `confluence_list_spaces` | 등록된 스페이스 목록 |
| `confluence_sync` | 동기화 안내 |
| `confluence_save_page` | Confluence 페이지를 vault에 저장 |

## 연결이 안 돼서 배운 두 가지

MCP 서버를 처음 등록했을 때 Claude Code에서 도구가 하나도 보이지 않았습니다. `claude mcp list`가 연결 실패를 뱉길래 기존에 잘 동작하던 다른 MCP 서버(dev-mcp) 코드와 비교해 봤습니다.

첫 번째는 transport 지정이었습니다.

```python
# dev-mcp (동작)
mcp.run(transport="stdio")

# 내 코드 (동작 안 함)
mcp.run()
```

당시 쓰던 FastMCP 버전은 `transport`를 생략하면 HTTP 서버 모드로 떴고, Claude Code는 stdio를 기대하고 있어서 프로세스에 붙지 못했습니다. 최신 FastMCP는 `mcp.run()`만 써도 stdio가 기본값이지만, 저는 전송 방식이 코드에서 바로 읽히는 편을 선호해 명시적으로 쓰는 쪽을 유지하고 있습니다.

두 번째는 등록 방법이었습니다. `transport`를 고친 뒤에도 목록에 뜨지 않아서 `~/.claude/.mcp.json`을 직접 편집했는데 반영이 안 됐습니다. Claude Code의 MCP 설정 파일은 세션 타입(user / project / local)마다 경로가 달라서 엉뚱한 파일에 쓰면 내부 인덱스와 어긋납니다. 수동 편집도 지원되긴 하지만 권장 방식은 CLI 명령입니다.

```bash
claude mcp add knowledge-vault -s user -- \
  /Users/me/obsidian-vault/.venv/bin/python3 \
  /Users/me/obsidian-vault/mcp_server.py
```

`-s user`는 사용자 범위(모든 프로젝트)로 등록하라는 뜻이고, 프로젝트 전용이면 `-s local`(기본값)이나 `-s project`입니다.

```
$ claude mcp list
dev-mcp: ✓ Connected
knowledge-vault: ✓ Connected
```

동작하는 레퍼런스 코드가 옆에 있을 때는 눈으로 비교하는 게 가장 빨랐습니다.

## 영어 임베딩 모델의 한국어 한계

처음에는 `all-MiniLM-L6-v2`(영어 모델)로 시작했습니다. 한국어 문서를 넣고 검색하니 유사도가 낮게 나왔고, 관련 문서보다 엉뚱한 문서가 Top1에 올라오는 경우가 있었습니다.

```
쿼리: "API 응답 형식이 어떻게 되나요?"
결과: API-표준 문서 (유사도: 0.469)
```

관련은 있는데 확신이 서지 않는 점수였습니다. 한국어 임베딩 모델 `BM-K/KoSimCSE-roberta-multitask`로 교체하고 같은 테스트를 돌려 봤습니다.

```python
model = SentenceTransformer("BM-K/KoSimCSE-roberta-multitask")

q = model.encode("API 응답 형식")
d1 = model.encode("API 응답은 REST JSON UTF-8로 통일하고...")  # 관련 문서
d2 = model.encode("비즈니스 규칙 체크 순서는...")                # 무관 문서

# 관련 문서 유사도: 0.744
# 무관 문서 유사도: 0.281
```

관련/무관이 명확히 갈렸습니다. 실제 운영에서 쓰는 쿼리 7개로 정량 비교를 해 보면 결과는 다음과 같았습니다.

| 쿼리 | 영어 모델 | 한국어 모델 | 차이 |
|------|----------|-----------|------|
| 비즈니스 규칙 체크 순서 | 0.726 | **0.784** | +0.058 |
| 재신청 시 이력 처리 | 0.530 | **0.561** | +0.031 |
| API 응답 형식 | 0.469 | **0.674** | **+0.205** |
| 신청 버튼 비활성화 조건 | **0.693** | 0.452 | -0.241 |
| 개인별 금액 제한 동작 | **0.714** | 0.705 | -0.010 |
| 전체 처리 흐름 | 0.716 | **0.768** | +0.051 |
| 동일 건 중복 차단 | **0.867** | 0.794 | -0.073 |

평균 유사도 자체는 0.674 → 0.677로 비슷했습니다. 하지만 정확 매칭(관련 문서가 Top1에 오르는 경우)은 3/7에서 4/7로 올랐고, 한국어 의미를 정확히 잡아야 하는 쿼리에서는 큰 차이가 났습니다. 가장 개선 폭이 큰 건 "API 응답 형식"으로 +0.205였습니다.

모든 쿼리에서 한국어 모델이 이긴 건 아니라는 점도 중요했습니다. 짧은 영어·숫자 조합이 많은 쿼리에서는 오히려 영어 모델이 앞섰습니다. 그래도 한국어 정책 문서가 주력인 환경에서는 한국어 모델이 맞는 선택이었습니다. 지금 다시 고른다면 장문맥에 강한 `KURE-v1`이나 dense+sparse 하이브리드를 지원하는 `bge-m3`도 후보에 넣어 볼 만하지만, 당시 기준으로는 KoSimCSE로도 충분히 실용 범위에 들어왔습니다.

## 문서 청킹으로 정확도를 올렸습니다

초반에는 각 Markdown 파일 전체를 하나의 벡터로 저장했습니다. 문제는 긴 정책 문서일수록 "적용 범위", "체크 순서", "취소 처리", "예외 케이스"가 한 벡터에 섞여 평균화된다는 점이었습니다. "체크 순서"라는 세부 질문을 해도 문서 전체가 혼합된 벡터와 매칭되니, 정답 근처의 섹션을 정확히 찍어주지 못했습니다.

해결책은 복잡할 필요가 없었습니다. `##` 헤더를 기준으로 섹션별 청크로 쪼개고, 섹션 제목을 메타데이터에 남기기만 했습니다.

```python
def _chunk_document(doc):
    path = doc["path"]
    body = doc["body"]

    # 짧은 문서(500자 이하)는 통째로
    if len(body) < 500:
        return [{"id": path, "content": body, "section": ""}]

    # ## 헤더로 분할
    sections = re.split(r'\n(?=## )', body)
    chunks = []
    for i, section in enumerate(sections):
        section_title = section.split("\n", 1)[0].lstrip("#").strip()
        chunks.append({
            "id": f"{path}#chunk{i}",
            "content": section,
            "section": section_title,
        })
    return chunks
```

16개 파일이 52개 청크로 늘었고, 검색 결과에 "어느 문서의 어느 섹션"인지 바로 표시되기 시작했습니다.

| 쿼리 | 청킹 전 | 청킹 후 |
|------|---------|---------|
| 비즈니스 규칙 체크 순서 | 버튼-정책 (0.78) | 버튼-정책 › 가능 여부 결정 요소 (0.76) |
| 재신청 시 이력 처리 | 전체흐름 (0.56) | 재신청-정책 › 처리 절차 (0.63) |
| 상태가 대기일 때 처리 상태 | (매칭 실패) | 상태-정의 › 처리 상태 (0.63) |
| API 오류 응답 형식 | API-표준 (0.67) | API-표준 › HTTP 상태 코드 (0.62) |

정확 매칭은 4/7에서 5~6/7로 올라갔습니다. 가장 인상적이었던 건 "재신청 시 이력 처리" 쿼리입니다. 청킹 전에는 일반적인 "전체 흐름" 문서가 Top1으로 올라왔지만, 청킹 후에는 "재신청-정책" 문서의 "처리 절차" 섹션을 정확히 찍어 왔습니다.

헤더 기반 청킹은 지금도 통하는 실용적인 baseline입니다. 2026년 들어 Semantic Chunking이나 Late Chunking 같은 기법이 정확도를 한 단계 더 끌어올린다는 사례가 보이지만, 먼저 `##` 헤더로 기본기를 잡고 필요해지면 다음 단계로 넘어가는 쪽이 나아 보였습니다.

## Hooks로 자동화했습니다

수동으로 `index_vault`를 호출하는 방식은 첫 주만 버텼습니다. 새 문서를 쓰고 반영을 깜빡하면 이전 상태로 검색 결과가 돌아왔습니다. 그래서 Claude Code의 Hook 시스템으로 자동화했습니다.

먼저 세션 시작 시 자동으로 인덱싱하도록 `SessionStart` hook을 만들었습니다.

```bash
#!/bin/bash
# ~/.claude/hooks/vault-auto-index.sh
VENV="$HOME/obsidian-vault/.venv/bin/python3"
"$VENV" -c "
import sys; sys.path.insert(0, '$HOME/obsidian-vault')
from mcp_server import index_vault
print(index_vault())
"
```

`~/.claude/settings.json`에서는 현재 스키마에 맞춰 `hooks` 객체 아래 `SessionStart` 배열을 두고, 각 엔트리에 `matcher`를 지정합니다. 새 세션을 여는 경우만 잡으면 되므로 `startup`으로 맞췄습니다.

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/vault-auto-index.sh"
          }
        ]
      }
    ]
  }
}
```

세션 종료 시 작업 요약을 vault에 남기는 것도 만들었습니다. 여기서 한 번 헷갈렸던 게 `Stop`과 `SessionEnd`의 구분입니다. `Stop`은 Claude가 한 턴을 끝낼 때마다 매번 호출되는 이벤트고, 세션당 여러 번 발생합니다. 세션이 실제로 종료되는 시점에만 한 번 움직여야 하니 `SessionEnd`를 쓰는 게 맞았습니다.

`SessionEnd`의 `matcher`는 종료 사유에 따라 여러 값을 가집니다. `logout`, `prompt_input_exit`, `clear`, `resume`, `bypass_permissions_disabled`, `other` 등으로 나뉘고, 잡고 싶은 사유마다 엔트리를 추가하는 구조입니다. 저는 일단 기타 경로로 세션이 끝나는 경우를 기록하려고 `other`로 시작했습니다.

```json
{
  "hooks": {
    "SessionEnd": [
      {
        "matcher": "other",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/vault-session-record.sh"
          }
        ]
      }
    ]
  }
}
```

`/exit`나 로그아웃처럼 다른 사유로 끝낸 세션도 남기려면 같은 구조로 엔트리를 복제해 `matcher`만 `prompt_input_exit`, `logout` 등으로 바꿔 주면 됩니다.

마지막으로, 자연어 질문이 들어왔을 때 vault 검색을 자동으로 시도하도록 rules 파일 하나를 추가했습니다.

```markdown
# ~/.claude/rules/knowledge-vault-auto-search.md
정책, 비즈니스 규칙, 신청 제한 같은 키워드가 포함된 질문이 들어오면
knowledge-vault MCP의 search를 먼저 실행해 관련 문서를 확인합니다.
```

이 rule이 없을 때는 "비즈니스 규칙 체크 순서가 뭐야?"라고 물으면 Claude가 코드만 뒤지고 끝났습니다. rule을 넣은 뒤에는 vault를 먼저 검색하고, 그다음 코드를 분석하는 흐름이 됐습니다. [하네스 엔지니어링](/posts/harness-engineering-for-ai-coding-agents/) 관점에서 보면 hook과 rule은 "에이전트가 어떤 환경에서 동작하느냐"를 강제하는 장치입니다. MCP 도구는 만들어둔 것만으로는 쓰이지 않고, 언제 어떤 조건에서 쓰여야 하는지를 규칙으로 같이 설계해야 실제로 돌아갑니다.

## 정책과 코드 사이의 갭을 찾은 순간

이 환경을 만들어둔 뒤, 한 세션에서 이렇게 물었습니다. **"비즈니스 규칙 체크 순서가 뭐야?"**

vault에서 가져온 정책 문서 기준 답변은 이렇게 나왔습니다.

```
1. 중복 체크 (동일 건 존재 여부)
2. 미완료 체크
3. 개별 제한 체크
4. 수량/금액 제한 체크
```

같이 분석한 코드에서는 이렇게 나왔습니다.

```
실제 구현된 Checker (order 값 순):
1. DuplicateOrderChecker          ✅
2. PeriodChecker                  ✅
3. CapacityChecker                ✅
4. SeriesRestrictionChecker       ✅
5~8. QuantitativeChecker (4개)    ✅
```

두 목록을 맞춰 보니 다음이 드러났습니다.

```
정책에는 있지만 코드에 없는 것:
- 미완료 체크 Checker       미구현
- 개별 제한 Checker         미구현
- 중복 체크 (정책 기준)      코드의 것과 개념이 다름
```

vault가 없었다면 코드만 보고 "8개 체커가 있다"로 끝냈을 겁니다. 정책 문서와 교차 비교를 할 수 있게 되자 구현 갭 세 건이 올라왔고, 그중 두 건은 실제 다음 스프린트에 태워야 할 항목이었습니다. 코드만 보면 "뭐가 있는지"를 알지만, 정책 문서와 비교하면 "뭐가 빠져 있는지"가 보입니다.

## 남은 과제

지금 구조에서 아직 정리하지 못한 지점들이 있습니다.

| 과제 | 현재 상태 | 생각하고 있는 다음 단계 |
|------|----------|-----------------------|
| 검색 정확도 | 5~6/7 (71~86%) | 벡터 + BM25 키워드 결합한 하이브리드 검색 |
| 문서 수 | 16개 | 50개 이상 축적 후 재측정 |
| Confluence 동기화 | 반자동 (수동 `sync`) | 변경 감지 + 자동 동기화 |
| 자동 기록 요약 | 원문 저장 수준 | LLM 기반 요약 생성 |
| 자동 검색 트리거 | 키워드 수동 관리 | rule 기반 조건 고도화, Late Chunking 실험 |

하이브리드 검색은 이번 주 안에 손대 볼 예정이고, 임베딩 모델은 문서 수가 어느 정도 쌓인 뒤에 `KURE-v1`이나 `bge-m3`와 한 번 비교해 볼 생각입니다.

## 같은 환경을 만든다면

팀 정책 문서가 Claude에게 보이지 않는 상황이라면 이 구조를 한번 깔아볼 만합니다. Obsidian vault + ChromaDB + FastMCP 조합에 한국어 임베딩 모델과 `##` 헤더 청킹까지 맞춰 두면, 첫 주 안에 "정책 vs 코드" 교차 질문을 던지는 단계로 넘어갈 수 있습니다. 저는 이 지점에 닿고 나서야 이 환경을 계속 유지할 이유가 생겼습니다.
