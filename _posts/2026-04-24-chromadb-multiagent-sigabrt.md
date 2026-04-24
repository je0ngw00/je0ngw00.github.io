---
title: "멀티 에이전트 환경에서 ChromaDB PersistentClient가 SIGABRT로 종료된 이유"
date: 2026-04-24 00:00:00 +0900
categories: [AI, Claude Code]
tags: [claude-code, chromadb, mcp, multi-agent, debugging, fastmcp]
---

[이전 편](/posts/local-rag-with-claude-code/)에서 Obsidian vault와 ChromaDB, FastMCP를 엮어 Knowledge Vault MCP 서버를 구축했습니다. 그 뒤로 정책 문서 검색에 꽤 잘 쓰고 있었는데, 멀티 에이전트 워크플로우를 본격적으로 돌리기 시작하면서 상황이 달라졌습니다. 같은 코드, 같은 데이터인데 cmux claude-teams로 여러 에이전트를 동시에 띄우자 Python 프로세스가 SIGABRT로 죽었습니다. 원인은 ChromaDB도, FastMCP도 아니었고 제가 오해한 "동시 접근"의 범위였습니다.

## 평온했던 Knowledge Vault MCP

이전 편의 구성은 단순했습니다. Python FastMCP 서버가 `mcp__knowledge-vault__search` 툴을 제공하고, 내부에서 `chromadb.PersistentClient(path=...)`로 로컬 디렉터리에 벡터 인덱스를 두는 방식입니다. 검색 요청이 오면 쿼리를 임베딩하고 컬렉션에 질의를 던져 결과를 반환합니다.

```python
import chromadb

client = chromadb.PersistentClient(path=str(CHROMA_DIR))
collection = client.get_or_create_collection("knowledge")
```

단일 Claude 세션에서는 아무 문제 없이 돌아갔습니다. 하나의 채팅이 순차적으로 MCP를 호출하고, 한 요청이 끝나야 다음 요청이 들어오니 embedded 모드의 전제와 정확히 맞는 상황이었습니다. 몇 주 동안 그 상태로 쓰다가 문제를 만났습니다.

## cmux claude-teams에서 일어난 일

cmux의 claude-teams 워크플로우는 역할별로 분리된 여러 에이전트를 병렬로 돌립니다. 보안 리뷰어, 품질 리뷰어, 컨벤션 리뷰어, 테스트 리뷰어 같은 에이전트가 동시에 변경 사항을 검토하는 구조입니다. 각 에이전트는 리뷰 중에 Knowledge Vault MCP로 정책 검색 요청을 보냅니다.

에이전트들이 한참 돌아가다가 Python 프로세스가 아래와 같은 크래시 로그를 남기고 SIGABRT로 종료됐습니다.

```
Process:   Python [XXXX]
Reason:    SIGABRT

Thread 0 Crashed:
  chromadb_rust_bindings.abi3.so  ...
  malloc_vreport                  ...
  free_list_checksum_botch        ...
```

`malloc_vreport`와 `free_list_checksum_botch`는 macOS `libsystem_malloc`의 힙 무결성 검사가 실패할 때 찍히는 로그입니다. use-after-free나 버퍼 오버런처럼 네이티브 레벨에서 메모리 상태가 깨졌을 때 나타나는 패턴으로, 여러 실행 주체가 같은 네이티브 자원을 공유하는 상황과 맞아떨어집니다. 다만 이 시그니처가 "ChromaDB 다중 프로세스 접근"에서만 나타나는 대표 패턴은 아니라는 점은 기억해 두어야 했습니다. 디스크 인덱스 손상이나 특정 버전 Rust 바인딩의 버그로도 비슷한 모습이 나올 수 있기 때문입니다.

처음엔 ChromaDB 버전 문제라고 생각했습니다. 버전을 올려봤지만 여전히 같은 자리에서 터졌습니다. 그러면 MCP 서버 쪽에 동시 접근 대비가 없어서 그런가 싶어 코드를 열어봤더니 `threading.Lock`이 이미 들어가 있었습니다.

## Lock이 있는데 왜? 방향을 잘못 잡은 디버깅

```python
import threading

_lock = threading.Lock()

def search(query: str):
    with _lock:
        return collection.query(query_embeddings=[embedding], n_results=5)
```

Lock이 있으니 동시 접근은 막혀 있을 것이라고 지레짐작했습니다. 그래서 "ChromaDB 자체 버그 아닌가"라는 가설로 넘어가 Rust 바인딩 이슈 트래커와 릴리스 노트를 뒤졌습니다. 의심스러운 이슈는 꽤 있었지만, 증상과 1:1로 맞아떨어지는 것은 없었습니다. 그 방향으로 한참을 돌다가 다시 처음으로 돌아왔습니다.

돌이켜 보니 증상 발생 시점에 한 가지 변화가 있었습니다. "**cmux claude-teams를 돌리기 시작한 뒤부터**"라는 단서입니다. 그 단서 앞에서 버전 탓부터 했던 게 첫 번째 실수였습니다.

## 단서는 "언제 처음 발생했는가"였다

claude-teams 모드에서 각 에이전트는 **별도 Python 프로세스**로 실행됩니다. 보안 리뷰어도, 품질 리뷰어도, 컨벤션 리뷰어도 각자 독립적인 프로세스입니다. 그 프로세스들이 모두 같은 knowledge-vault MCP에 검색 요청을 보내면 결과적으로 여러 프로세스가 동시에 같은 ChromaDB 데이터 디렉터리에 접근하게 됩니다.

ChromaDB 공식 Cookbook은 이 상황을 분명히 선을 긋고 있습니다.

> Chroma is not process-safe for concurrent writers sharing the same local persistence path.

같은 로컬 persistence 경로를 두 개 이상의 프로세스가 공유하면서 쓰는 상황을 공식적으로 지원하지 않는다는 뜻입니다. 내부적으로 ChromaDB는 SQLite 위에 per-process 인메모리 segment 캐시와 HNSW 인덱스를 유지합니다. 파일 쓰기는 SQLite 잠금으로 어느 정도 직렬화된다고 해도, 각 프로세스가 자기 메모리에 올려 둔 인덱스 상태는 서로 동기화되지 않습니다. 문서화된 대표 증상은 "`get()`은 최신 데이터를 돌려주는데 `query()`는 stale한 결과를 돌려준다" 같은 캐시 불일치입니다. 힙 손상까지 발생한다고 공식적으로 명시된 것은 아니지만, 네이티브 바인딩이 개입하는 경로에서는 같은 전제 위반이 더 거친 실패로 나타날 수 있습니다. 이번 크래시도 그 연장선에서 해석한 것입니다.

여기서 `threading.Lock`의 한계가 드러납니다. [Python threading 문서](https://docs.python.org/3/library/threading.html#lock-objects)에 명시되어 있듯이 `threading.Lock`은 **같은 프로세스 안의 스레드끼리만** 경합을 막습니다. 프로세스 경계를 넘나드는 동시 접근에는 아무 효과가 없습니다.

```
에이전트 A (프로세스 1) ─┐
에이전트 B (프로세스 2) ─┤→ ChromaDB embedded → 힙 손상 → SIGABRT
에이전트 C (프로세스 3) ─┘
```

그림으로 그려 놓으면 단순합니다. 이 점을 늦게 알아챘다는 것이 문제였습니다.

## 해결안을 두 갈래로 놓고 봤다

근본 해결은 두 방향 중 하나였습니다.

**옵션 A: ChromaDB를 서버 모드로 분리한다.** `chroma run`으로 별도 HTTP 서버를 띄우고, MCP 서버는 `HttpClient`로 접속합니다. 여러 프로세스가 같은 ChromaDB 서버에 HTTP를 통해 접근하므로 상태를 소유하는 주체가 서버 한 곳으로 모입니다.

**옵션 B: 파일 잠금으로 embedded 모드를 지킨다.** `fcntl.flock`이나 `filelock` 같은 파일 기반 잠금으로 쓰기를 직렬화하는 방향입니다. 얼핏 간단해 보이지만 이 길은 막다른 길입니다. 파일 쓰기를 직렬화해도 각 프로세스가 자기 메모리에 캐시해 둔 인덱스 상태까지 동기화되지는 않기 때문에, stale read나 또 다른 형태의 상태 불일치가 남습니다. ChromaDB는 애초에 이 구조를 다중 프로세스에서 지원한다고 문서화한 적이 없고, 파일 잠금으로 살리라고 권한 적도 없습니다.

선택 기준은 단순했습니다. **상태를 소유하는 주체가 하나여야 합니다.** 여러 프로세스가 같은 상태를 동시에 소유하려고 애쓰는 대신, 상태를 서버 한 곳에 모으고 나머지는 클라이언트가 되는 구조로 가는 편이 자연스럽습니다. 옵션 A로 결정했습니다.

## PersistentClient에서 HttpClient로

코드 변경 자체는 거의 한 줄이었습니다.

```python
# 변경 전
client = chromadb.PersistentClient(path=str(CHROMA_DIR))

# 변경 후
client = chromadb.HttpClient(host="localhost", port=8000)
```

하지만 이 변경에는 "ChromaDB 서버가 떠 있다"는 전제가 따라붙습니다. MCP 서버를 Claude Code에 연결할 때 매번 `chroma run`을 수동으로 띄워 두기는 번거로워서, MCP 서버 초기화 시점에 ChromaDB 서버 상태를 확인하고 없으면 자동으로 띄우는 함수를 추가했습니다.

```python
import subprocess
import time
import requests

CHROMA_URL = "http://localhost:8000"

def _ensure_chroma_server():
    # 이미 실행 중이면 그냥 통과
    try:
        requests.get(f"{CHROMA_URL}/api/v2/healthcheck", timeout=1)
        return
    except requests.RequestException:
        pass

    # 없으면 백그라운드로 실행
    subprocess.Popen(
        ["chroma", "run", "--path", str(CHROMA_DIR), "--port", "8000"],
        stdout=subprocess.DEVNULL,
        stderr=subprocess.DEVNULL,
    )

    # 최대 10초까지 readiness 확인
    for _ in range(10):
        time.sleep(1)
        try:
            requests.get(f"{CHROMA_URL}/api/v2/healthcheck", timeout=1)
            return
        except requests.RequestException:
            continue

    raise RuntimeError("ChromaDB 서버 시작 실패")
```

헬스체크 경로는 공식 문서에 명시된 [`/api/v2/healthcheck`](https://docs.trychroma.com/reference/chroma-api/system/healthcheck)를 사용했습니다. 루트 경로 `/api/v2/`만 조회하는 것보다 문서화된 헬스체크 엔드포인트를 쓰는 편이 버전별 응답 차이에서 자유롭습니다.

변경 후 구조는 아래와 같습니다.

```
에이전트 A (프로세스 1) ─┐
에이전트 B (프로세스 2) ─┤→ HTTP → ChromaDB 서버 (단일 프로세스) → 정상
에이전트 C (프로세스 3) ─┘
```

여러 에이전트가 동시에 검색을 보내도 요청이 ChromaDB 서버 한 곳에 모여 처리되고, 인덱스 상태를 소유하는 프로세스가 하나로 고정됩니다. 같은 워크플로우를 반복해도 힙 손상은 더 이상 재현되지 않았습니다.

## 자동 기동 코드의 한계도 인정해야 한다

이 자동 기동 함수가 편리하긴 해도 만능은 아닙니다. 로컬 개발용으로 충분히 쓸 만하지만 몇 가지 경계는 분명히 있습니다.

첫째, 여러 MCP 인스턴스가 거의 동시에 기동될 때 `chroma run`을 두 번 띄우려 시도하는 race가 있을 수 있습니다. 한쪽은 8000 포트에 bind하지 못해 실패하고, 그 사이 이미 떠 있는 쪽을 잡아 사용하면 대부분 문제가 없지만, 타이밍에 따라서는 첫 readiness 체크가 실패하는 얇은 구간이 생깁니다.

둘째, 부모인 MCP 서버가 죽어도 `chroma run` 자식 프로세스는 살아남아 orphan이 됩니다. 장시간 개발하다 보면 떠 있는 `chroma run` 프로세스를 주기적으로 확인해 정리해야 하는 상황이 생깁니다.

진짜로 "프로덕션급"으로 운영한다면 이 자동 기동 로직 대신 `launchd`(macOS)나 `systemd --user`, 또는 Docker로 ChromaDB 서버를 별도 서비스로 떼어놓는 쪽이 깔끔합니다. 지금은 로컬 단일 사용자 환경이고 MCP 서버와 ChromaDB의 생명주기가 비슷해도 괜찮다는 판단으로 자동 기동을 유지했지만, 이 선택이 로컬 편의성을 위한 타협이라는 점은 코드 주석 없이도 기억해 두어야 했습니다.

## 같은 로컬이어도 단일 프로세스가 아닐 수 있다

멀티 에이전트 팀을 돌리기 전까지는 이 문제를 만날 일이 없었습니다. 단일 Claude 세션에서 순차적으로 MCP를 호출하는 동안에는 embedded 모드의 전제가 깨진 적이 없기 때문입니다. 규모나 사용 방식이 바뀌는 순간 전제 자체가 바뀐다는 게 이 사건의 진짜 교훈이었습니다.

구축 당시 "나중에 멀티 에이전트로 쓸 수도 있지 않을까"를 먼저 떠올렸다면 처음부터 서버 모드로 갔겠지만, 솔직히 그 생각은 못 했습니다. 다만 이번 일을 겪으면서 로컬 MCP 서버를 설계할 때 한 가지를 체크리스트에 넣게 됐습니다. "여러 에이전트, 여러 워커, 병렬 테스트 같은 상황에서 이 서버를 동시에 호출해도 괜찮은 구조인가?" 자동화와 병렬 에이전트가 점점 당연해지는 환경에서는, 로컬이라는 단어가 "단일 접근자"를 보장해 주지 않는다는 사실을 늦지 않게 반영해 두는 편이 낫습니다.

디버깅 측면에서도 한 가지를 다시 확인했습니다. 크래시가 처음 났을 때 "버전 문제 같다"는 가설로 먼저 움직였는데, 그보다 먼저 물었어야 할 질문은 "**언제부터 이 증상이 나타났는가**"였습니다. 증상 발생 시점과 그 직전에 환경에서 달라진 것을 먼저 좁혔다면, Rust 바인딩 이슈 트래커를 한참 뒤지는 대신 claude-teams 호출이라는 단서 하나로 훨씬 빨리 답에 닿았을 것입니다. 같은 실수를 다음번에는 조금 덜 하고 싶어서 이 글을 남겨 둡니다.
