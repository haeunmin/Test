# gitbook\_agent\_build\_plan

## GitBook Agent 구축 계획서

### 1. 프로젝트 개요

#### 1.1 목적

본 프로젝트의 목표는 **사용자가 Slack에서 자연어로 질문하면, GitBook에 축적된 사내 문서에서 관련 내용을 찾아 근거 기반 답변을 제공하는 사내 Knowledge Agent를 구축하는 것**이다.

기존의 단순 키워드 검색이나 일반적인 Vector RAG에만 의존하지 않고, **자체 온톨로지 기반 Retrieval 기술**을 핵심 검색 엔진으로 사용하여 문서 간 개념·엔티티·관계를 활용한 정교한 검색을 수행한다.

기본 사용자 경험은 다음과 같다.

```
사용자
  ↓
Slack에서 @GitBook-Agent 질문
  ↓
질문 분석
  ↓
자체 Ontology Retrieval
  ↓
GitBook 문서에서 관련 근거 검색
  ↓
LLM이 검색된 근거만 바탕으로 답변 생성
  ↓
Slack Thread에 답변 + 출처 제공
```

***

### 2. 프로젝트 한 줄 정의

> **Slack에서 질문하면 GitBook 문서를 자체 온톨로지 기반 Retrieval 기술로 검색하고, 검색된 GitBook 근거만 사용해 답변하는 사내 Knowledge Agent를 구축한다.**

***

### 3. 핵심 원칙

#### 3.1 GitBook을 Knowledge Source로 사용

Agent가 답변에 사용할 수 있는 지식의 범위는 **GitBook에 존재하는 문서**로 제한한다.

포함 범위:

* GitBook에서 직접 작성한 문서
* GitBook과 GitHub Git Sync를 통해 동기화된 Markdown 문서
* GitBook 문서 안의 표, 코드 블록, 설명, 링크 및 구조 정보
* GitBook 문서로부터 추출한 Entity / Relation / Ontology Fact

제외 범위:

* Slack 대화 자체
* Web Search 결과
* GitHub Source Code
* GitHub Issue / PR
* Jira / Drive 등 별도 시스템의 정보
* LLM이 기존에 알고 있는 외부 지식

즉, **GitBook에 없는 정보는 답변하지 않는다.**

#### 3.2 Slack은 사용자 인터페이스

Slack은 지식 소스가 아니라 **Agent를 사용하는 인터페이스**로만 사용한다.

```
사용자
@agent server15 모델 serving 방식 알려줘

Agent
GitBook 문서 기준으로 server15에서는 vLLM을 사용합니다.

출처
- Model Serving > Server15
```

사용자는 별도의 사내 검색 UI를 학습할 필요 없이 기존 업무 공간인 Slack에서 바로 Agent를 사용할 수 있다.

#### 3.3 검색은 자체 Ontology Retrieval이 담당

GitBook의 기본 검색에만 의존하지 않고 다음 검색 방법을 조합한다.

```
Vector Retrieval
+
Keyword / BM25
+
Ontology Graph Retrieval
+
Reranking
```

특히 Ontology Retrieval은 다음 유형의 질문을 잘 처리하는 것을 목표로 한다.

* 용어 Alias가 다른 질문
* 여러 문서를 거쳐야 하는 Multi-hop 질문
* 특정 Entity 간 관계를 묻는 질문
* 문서 간 연결 정보가 필요한 질문
* 사용자가 정확한 문서명을 모르는 질문

***

### 4. 참고 구조

사내 Agent인 셀렉이와 유사하게 **사용자 인터페이스와 Agent 실행 코어를 분리**하는 구조를 사용한다.

셀렉이의 기본 흐름:

```
사용자 요청
  ↓
요청 분석
  ↓
Agent / Tool 선택
  ↓
정보 검색
  ↓
결과 통합
  ↓
최종 답변
```

본 프로젝트에서는 이를 GitBook Knowledge Search에 맞게 단순화한다.

```
Slack
  ↓
Slack Adapter
  ↓
Agent API
  ↓
Query Controller
  ↓
GitBook Knowledge Agent
  ↓
Ontology Retrieval
  ↓
Evidence Builder
  ↓
Answer Generator
  ↓
Slack
```

초기에는 Multi-Agent 구조를 크게 확장하지 않고 **GitBook Knowledge Agent 하나를 안정적으로 만드는 것**에 집중한다.

***

## 5. 전체 시스템 아키텍처

```
                    ┌────────────────────┐
                    │       Slack        │
                    │                    │
                    │   @agent 질문      │
                    └─────────┬──────────┘
                              │
                              ▼
                    ┌────────────────────┐
                    │   Slack Adapter    │
                    │                    │
                    │ Mention / DM       │
                    │ Thread Context     │
                    │ User Identity      │
                    └─────────┬──────────┘
                              │
                              ▼
                    ┌────────────────────┐
                    │     Agent API      │
                    │                    │
                    │ Session            │
                    │ Authentication     │
                    │ Logging            │
                    └─────────┬──────────┘
                              │
                              ▼
                    ┌────────────────────┐
                    │ Query Controller   │
                    │                    │
                    │ Query Rewrite      │
                    │ Scope Check        │
                    │ Permission Check   │
                    └─────────┬──────────┘
                              │
                              ▼
                 ┌──────────────────────────┐
                 │ GitBook Knowledge Agent  │
                 └────────────┬─────────────┘
                              │
                              ▼
                 ┌──────────────────────────┐
                 │ Ontology RAG Engine      │
                 │                          │
                 │ Entity Linking           │
                 │ Vector Retrieval         │
                 │ BM25 Retrieval           │
                 │ Graph Retrieval          │
                 │ Fusion                   │
                 │ Reranking                │
                 └────────────┬─────────────┘
                              │
                 ┌────────────┴────────────┐
                 ▼                         ▼
         ┌───────────────┐         ┌───────────────┐
         │ Ontology Graph│         │ Vector / BM25 │
         └───────┬───────┘         └───────┬───────┘
                 └────────────┬─────────────┘
                              ▼
                         GitBook Corpus
                              │
                              ▼
                    ┌────────────────────┐
                    │ Evidence Builder   │
                    └─────────┬──────────┘
                              │
                              ▼
                    ┌────────────────────┐
                    │ Answer Generator   │
                    │                    │
                    │ GitBook Evidence   │
                    │ Only               │
                    └─────────┬──────────┘
                              │
                              ▼
                            Slack
```

***

## 6. GitBook 데이터 구성

### 6.1 GitBook 직접 작성 문서

GitBook에서 직접 작성된 문서는 그대로 Knowledge Corpus에 포함한다.

### 6.2 GitHub Git Sync 문서

GitBook과 GitHub가 Git Sync로 연결되어 있는 경우, GitHub의 Markdown 문서가 GitBook 페이지로 동기화된다.

```
GitHub Repository
└── docs/
    ├── infra.md
    ├── deployment.md
    ├── serving.md
    └── policy.md
          │
          │ Git Sync
          ▼
       GitBook
```

이 경우 Agent 입장에서는 GitHub를 별도 Knowledge Source로 취급하지 않는다.

> **GitBook에 동기화된 Markdown은 모두 GitBook 문서로 취급한다.**

따라서 별도의 GitHub MCP, GitHub Search Agent, Code Search 기능은 초기 범위에 포함하지 않는다.

***

## 7. GitBook Ingestion

{% stepper %}
{% step %}
### Full Sync

초기 구축 시 GitBook 전체 문서를 수집한다.

```
GitBook
  ↓
Page List
  ↓
Page Content
  ↓
Parser
  ↓
Chunk
  ↓
Ontology / Vector / BM25 Index
```

저장 정보 예시:

```json
{
  "space_id": "space_123",
  "page_id": "page_456",
  "title": "NAS Workspace",
  "path": "/infra/nas/workspace",
  "source_url": "https://docs.company.com/infra/nas/workspace",
  "revision_id": "rev_123",
  "updated_at": "2026-09-14T12:00:00Z"
}
```
{% endstep %}

{% step %}
### Incremental Update

GitBook 문서가 변경될 때 변경된 문서만 다시 인덱싱한다.

```
GitBook 문서 변경
     ↓
Webhook / Revision Check
     ↓
변경 Page 확인
     ↓
기존 Chunk / Fact 제거
     ↓
재 Parsing
     ↓
Ontology / Vector Index Update
```

정책 문서가 수정되었을 때 Agent가 오래된 정보를 답하지 않도록 **Revision 기반 동기화**를 관리한다.
{% endstep %}
{% endstepper %}

***

## 8. Document Parsing / Chunking

단순 Token 수 기준으로 문서를 자르기보다 GitBook 문서의 구조를 유지한다.

예:

```
# NAS Workspace

## Workspace

### team_data

### team_eval

## Buffer

## Archive
```

Chunk:

```
NAS Workspace > Workspace > team_data
NAS Workspace > Workspace > team_eval
NAS Workspace > Buffer
NAS Workspace > Archive
```

Chunk Metadata 예시:

```json
{
  "chunk_id": "chunk_123",
  "page_id": "page_456",
  "heading_path": [
    "NAS Workspace",
    "Workspace",
    "team_data"
  ],
  "content": "...",
  "source_url": "..."
}
```

이 구조를 통해 답변에 정확한 Section 단위 출처를 표시할 수 있다.

***

## 9. Ontology 구축

### 9.1 목적

GitBook 문서를 단순 Text Chunk 집합으로 보는 것이 아니라, 문서 안의 주요 Entity와 Relation을 구조화한다.

예:

```
team_data
   ↓ USES
Workspace
   ↓ LOCATED_AT
/mnt/nas/workspace/team_data

Workspace
   ↓ HAS_QUOTA
5TB
```

### 9.2 초기 Entity 후보

```
Team
Person
Project
System
Service
Server
Storage
Path
Repository
Model
Dataset
Policy
Command
Environment
Document
```

### 9.3 초기 Relation 후보

```
USES
OWNS
BELONGS_TO
RUNS_ON
LOCATED_AT
DEPENDS_ON
HAS_POLICY
HAS_QUOTA
MANAGED_BY
REPLACES
RELATED_TO
DOCUMENTED_IN
```

처음부터 Ontology Schema를 크게 만들기보다 실제 GitBook 문서의 반복 패턴을 기준으로 점진적으로 확장한다.

***

## 10. Provenance

Ontology의 모든 Fact는 반드시 원본 GitBook 문서와 연결한다.

예:

```
team_data
   ↓ HAS_QUOTA
5TB
   ↓
Evidence
   ↓
GitBook
NAS Workspace > Workspace > team_data
```

Fact 저장 예시:

```json
{
  "subject": "team_data",
  "predicate": "HAS_QUOTA",
  "object": "5TB",
  "confidence": 0.98,
  "source_page_id": "page_456",
  "source_chunk_id": "chunk_123",
  "source_revision": "rev_789"
}
```

이를 통해 다음이 가능해진다.

* 답변 출처 표시
* Ontology 오류 추적
* 문서 변경 시 Fact 갱신
* Hallucination 방지
* Retrieval Debugging

***

## 11. Query Processing

사용자 질문을 그대로 Embedding Search에 넣지 않는다.

예:

```
AI팀 모델 저장 공간 어디야?
```

Query Analyzer:

```json
{
  "entities": [
    "AI Team",
    "Model"
  ],
  "intent": "find_storage_location",
  "possible_relations": [
    "USES",
    "LOCATED_AT"
  ]
}
```

주요 단계:

```
Query
  ↓
Thread Context 반영
  ↓
Standalone Query Rewrite
  ↓
Entity Detection
  ↓
Entity Linking
  ↓
Intent / Relation Analysis
```

***

## 12. Retrieval Pipeline

### 12.1 Vector Retrieval

의미적으로 유사한 문서 Chunk를 찾는다.

강점:

* 자연어 표현 차이
* Paraphrase
* 설명형 질문

### 12.2 BM25 Retrieval

정확한 고유명사, 코드명, 경로 검색에 사용한다.

예:

```
server15
team_data
K-EXAONE
/mnt/nas/archive
```

### 12.3 Ontology Graph Retrieval

본 프로젝트의 핵심 기술이다.

예:

```
질문
AI팀 모델 저장 공간 어디야?

AI Team
   ↓ USES
Workspace
   ↓ LOCATED_AT
/mnt/nas/workspace/team_data
```

단순 의미 유사도만으로 찾기 어려운 관계를 Entity와 Graph를 이용해 검색한다.

***

## 13. Retrieval Fusion / Reranking

각 Retriever의 결과를 합친다.

```
Vector Candidates
BM25 Candidates
Graph Candidates
       ↓
    Fusion
       ↓
    Reranker
       ↓
 Top-K Evidence
```

초기에는 RRF 등을 사용하고, 이후 Ontology Score를 포함한 자체 Fusion 방식을 실험할 수 있다.

예:

```
final_score =
    α × vector_score
  + β × bm25_score
  + γ × ontology_score
  + δ × reranker_score
```

***

## 14. Evidence Builder

검색 결과를 그대로 LLM에 전달하지 않는다.

Evidence Builder가 실제 답변에 사용할 근거를 구성한다.

예:

```
Evidence 1

Type
Ontology Fact

Fact
AI Team → USES → team_data workspace

Source
NAS Workspace > Team Workspace


Evidence 2

Type
GitBook Document

Content
team_data workspace의 용량은 5TB...

Source
NAS Workspace > Workspace > team_data
```

***

## 15. Answer Generation

LLM은 **검색된 GitBook Evidence를 읽고 답변을 생성하는 역할만 수행**한다.

필수 정책:

```
1. 제공된 GitBook Evidence에서 확인되는 정보만 답변한다.
2. LLM의 외부 지식으로 내용을 보완하지 않는다.
3. 모든 주요 사실은 GitBook Source와 연결한다.
4. 근거가 없으면 추측하지 않는다.
```

No-answer 예시:

```
현재 접근 가능한 GitBook 문서에서는
해당 내용을 확인할 수 없습니다.
```

***

## 16. Slack Thread Context

Follow-up 질문을 지원한다.

```
User
team_data 용량 얼마야?

Agent
5TB입니다.

User
자동 삭제돼?
```

두 번째 질문은 그대로 검색하지 않고 다음처럼 변환한다.

```
team_data workspace는 자동 삭제되는가?
관련 삭제 정책은 무엇인가?
```

검색에는 Standalone Query를 사용하고, 답변 생성에는 전체 Thread Context를 함께 활용한다.

***

## 17. Permission / Security

기본 원칙:

> **Agent가 검색할 수 있는 GitBook 문서 범위는 사용자가 접근 가능한 범위를 넘지 않는다.**

```
Slack User
   ↓
User Identity
   ↓
GitBook Permission
   ↓
Accessible Pages / Spaces
   ↓
ACL Filter
   ↓
Retrieval
```

검색 후 결과를 제거하는 방식보다 **Retrieval 이전에 권한 필터를 적용**하는 것을 원칙으로 한다.

***

## 18. Logging / Ledger

셀렉이처럼 Agent의 전체 실행 흐름을 단계별로 기록한다.

```
User
 ↓
Conversation
 ↓
Turn
 ↓
Run
 ↓
Step
 ↓
Retrieval
 ↓
Answer
 ↓
Feedback
```

저장 대상:

```
query
rewritten_query
detected_entities
detected_relations
vector_candidates
bm25_candidates
graph_candidates
fusion_scores
rerank_scores
selected_evidence
answer
sources
latency
feedback
```

이를 통해 실패 원인을 구분할 수 있다.

```
Query Rewrite 문제
Entity Linking 문제
Graph Retrieval 문제
Vector Recall 문제
Fusion 문제
Reranker 문제
Answer Generation 문제
```

***

## 19. 사용자 Feedback

Slack 답변 하단에 간단한 Feedback을 제공한다.

```
👍 도움됨
👎 도움 안됨
```

실사용 로그와 Feedback은 이후 다음 용도로 활용한다.

* 실제 질문 분포 분석
* 검색 실패 유형 분석
* Evaluation Dataset 구축
* Ontology Schema 개선
* Reranker 개선
* Agent 품질 모니터링

***

## 20. 평가 계획

### 20.1 Baseline

다음 검색 방법을 동일한 Query Set에서 비교한다.

| 방법            | 설명                             |
| ------------- | ------------------------------ |
| GitBook 기본 검색 | GitBook의 기본 검색                 |
| Vector RAG    | Embedding 기반 검색                |
| Hybrid RAG    | Vector + BM25                  |
| Ontology RAG  | Vector + BM25 + Ontology Graph |

### 20.2 Retrieval Metric

```
Recall@5
Recall@10
MRR
nDCG
Hit Rate
```

Ontology 관련 Metric:

```
Entity Linking Accuracy
Relation Accuracy
Graph Path Accuracy
Evidence Recall
```

### 20.3 Answer Metric

```
Correctness
Faithfulness
Completeness
Citation Correctness
Unsupported Claim Rate
```

***

## 21. 평가 Query 유형

### 21.1 Simple Lookup

```
team_data 용량 몇 TB야?
```

### 21.2 Alias

```
AI팀 작업 공간 어디야?
```

문서에는 `team_data`라는 이름만 있을 수 있다.

### 21.3 Multi-hop

```
AI팀이 사용하는 workspace의 삭제 정책 알려줘.
```

필요 경로:

```
AI Team
  → Workspace
  → team_data
  → deletion policy
```

### 21.4 Relation Query

```
server15와 관련된 storage 정책 알려줘.
```

### 21.5 Cross-document

```
프로젝트 진행 중에는 데이터를 어디 두고,
프로젝트가 끝나면 어디로 옮겨야 해?
```

여러 GitBook Page의 Evidence가 필요하다.

***

## 22. MVP 범위

1차 MVP에는 다음 기능만 포함한다.

```
Slack UI
+
Agent API
+
GitBook Ingestion
+
Document Parsing
+
Ontology Builder
+
Vector / BM25 / Graph Retrieval
+
Fusion / Reranking
+
Evidence Builder
+
Answer Generator
+
Source Citation
+
Thread Follow-up
+
Feedback
+
Logging
```

***

## 23. 초기 MVP에서 제외할 기능

초기에는 다음 기능을 의도적으로 제외한다.

```
GitHub Code Search
GitHub Issue / PR Search
Slack 대화 검색
Web Search
Jira Search
Google Drive Search
Scheduler
Reporter
복잡한 Multi-Agent Planning
GitBook 자동 수정
```

핵심 목표는 먼저 **GitBook Retrieval 품질을 검증하는 것**이다.

***

## 24. 개발 단계

{% stepper %}
{% step %}
### Phase 1. Retrieval Prototype

목표:

> Ontology Retrieval이 일반적인 Vector / Hybrid Retrieval보다 실제로 더 좋은지 검증

구현:

```
GitBook Ingestion
Document Parsing
Vector Index
BM25
Ontology Schema
Entity / Relation Extraction
Graph Index
Retrieval Evaluation
```
{% endstep %}

{% step %}
### Phase 2. Slack Agent MVP

```
Slack
  ↓
Agent API
  ↓
GitBook Knowledge Agent
  ↓
Ontology Retrieval
  ↓
Answer
```

기능:

```
@mention
DM
Thread Follow-up
Source Citation
No-answer
Feedback
Logging
Permission
```
{% endstep %}

{% step %}
### Phase 3. Retrieval 고도화

다음 기능을 개선한다.

```
Entity Linking
Alias Resolution
Multi-hop Retrieval
Cross-document Retrieval
Fusion
Reranking
No-answer 판단
Citation Validation
Permission-aware Retrieval
```
{% endstep %}

{% step %}
### Phase 4. 제품 확장

Agent의 Knowledge Source는 GitBook으로 유지하면서 다음 방향으로 확장할 수 있다.

#### UI 확장

```
Slack
→ 사내 Web
→ CLI
→ GitBook Embedded UI
```

#### Retrieval 확장

```
Ontology Schema 자동 개선
Graph Retrieval 최적화
Query Routing
Personalized Retrieval
Domain별 Ontology
```

#### Agent 기능 확장

```
문서 요약
문서 비교
관련 문서 추천
정책 변경 탐지
문서 간 충돌 탐지
FAQ 자동 생성
문서 품질 점검
```

#### GitBook 운영 기능 확장

```
오래된 문서 탐지
중복 문서 탐지
문서 간 모순 탐지
Ontology 기반 관련 문서 연결
누락된 문서 후보 추천
```
{% endstep %}
{% endstepper %}

***

## 25. 확장 방향

본 프로젝트는 단순한 Q\&A Bot에서 끝나지 않고 여러 방향으로 확장할 수 있다.

### 25.1 Knowledge Discovery

사용자가 질문하지 않아도 GitBook 내부의 연결 관계를 분석해 관련 문서를 추천한다.

```
현재 문서
  ↓
Ontology 관계 분석
  ↓
관련 문서 추천
```

### 25.2 문서 품질 관리

Ontology를 이용해 다음을 탐지할 수 있다.

```
동일 개념의 중복 문서
서로 충돌하는 정책
오래된 정보
끊어진 Reference
정의되지 않은 사내 용어
```

### 25.3 Knowledge Graph UI

GitBook 문서를 Graph 형태로 탐색할 수 있도록 확장할 수 있다.

```
Server15
 ├─ RUNS_ON → A100
 ├─ USES → vLLM
 ├─ MOUNTS → NAS
 └─ DOCUMENTED_IN → Model Serving
```

### 25.4 Personalized Knowledge Agent

사용자의 팀 / 직무 / 권한을 기반으로 관련도가 높은 문서를 우선 검색할 수 있다.

단, 권한 밖의 문서를 노출하지 않는 것을 전제로 한다.

### 25.5 평가 자산화

실사용 질문과 Feedback을 쌓아 다음 자산으로 활용한다.

```
Internal QA Benchmark
Retrieval Benchmark
Ontology Evaluation Dataset
Agent Evaluation Dataset
```

이를 통해 향후 다른 Agent 시스템의 평가에도 활용할 수 있다.

***

## 26. 권장 개발 일정

| 기간     | 주요 작업                            | 산출물                     |
| ------ | -------------------------------- | ----------------------- |
| Week 1 | GitBook 구조 분석 / Connector        | 전체 문서 Ingestion         |
| Week 2 | Parser / Chunk / Baseline        | Vector + BM25           |
| Week 3 | Ontology Schema / Extraction     | 초기 Knowledge Graph      |
| Week 4 | Entity Linking / Graph Retrieval | Ontology Retriever      |
| Week 5 | Fusion / Reranker / Evaluation   | Hybrid Retrieval Engine |
| Week 6 | Agent API / Slack Bot            | Slack Agent MVP         |
| Week 7 | Permission / Logging / Feedback  | 운영 기반                   |
| Week 8 | Evaluation / Failure Analysis    | MVP Release             |

***

## 27. Repository 구조 예시

```
gitbook-agent/

├── apps/
│   ├── slack-bot/
│   └── agent-api/
│
├── agent/
│   ├── query-controller/
│   ├── knowledge-agent/
│   ├── evidence-builder/
│   └── answer-generator/
│
├── retrieval/
│   ├── query-analyzer/
│   ├── entity-linker/
│   ├── vector/
│   ├── bm25/
│   ├── graph/
│   ├── fusion/
│   └── reranker/
│
├── ontology/
│   ├── schema/
│   ├── extractor/
│   └── graph-store/
│
├── connectors/
│   └── gitbook/
│
├── storage/
│   ├── vector-db/
│   ├── graph-db/
│   └── ledger/
│
├── evaluation/
│   ├── dataset/
│   ├── retrieval/
│   └── qa/
│
└── infra/
```

***

## 28. MVP 완료 기준

* [ ] Slack에서 `@agent` 질문 가능
* [ ] GitBook 전체 문서 Ingestion
* [ ] Git Sync된 GitHub Markdown도 검색 가능
* [ ] GitBook 변경사항 Incremental Update
* [ ] Structure-aware Chunking
* [ ] Vector Search
* [ ] BM25 Search
* [ ] Ontology Graph 구축
* [ ] Entity Linking
* [ ] Graph Retrieval
* [ ] Hybrid Fusion
* [ ] Reranking
* [ ] GitBook Source Citation
* [ ] Thread Follow-up
* [ ] No-answer 처리
* [ ] Permission-aware Retrieval
* [ ] Feedback 수집
* [ ] Retrieval Trace 저장
* [ ] Baseline 비교 평가

***

## 29. 핵심 성공 기준

이 프로젝트의 가장 중요한 검증 질문은 다음이다.

> **우리의 Ontology Retrieval을 사용했을 때 GitBook의 관련 문서를 기존 검색 방식보다 실제로 더 잘 찾는가?**

따라서 기능 수보다 다음 비교를 우선한다.

```
GitBook 기본 검색
      vs
Vector RAG
      vs
Vector + BM25
      vs
Ontology + Vector + BM25
```

특히 아래 유형에서 성능 향상을 검증한다.

```
Alias
Multi-hop
Relation
Cross-document
Implicit Entity
```

***

## 30. 우선순위

```
1순위
Ontology Retrieval 성능

2순위
GitBook Ingestion / Sync / Provenance

3순위
Evidence 기반 Answer 품질

4순위
Slack UX

5순위
Evaluation / Logging

6순위
확장 기능
```

***

## 31. 최종 목표

초기에는 아래 구조를 안정적으로 구축한다.

```
Slack
  ↓
GitBook Knowledge Agent
  ↓
Ontology Retrieval
  ↓
GitBook Evidence
  ↓
Answer
```

장기적으로는 이를 기반으로 **사내 GitBook 전체를 구조적으로 이해하고, 검색·질의응답·문서 관리·지식 탐색까지 지원하는 Knowledge Agent Platform**으로 확장한다.

***

### 최종 요약

> 사용자는 Slack에서 평소처럼 질문한다.\
> Agent는 GitBook에서 관련 문서를 찾는다.\
> 문서를 찾는 핵심 기술은 자체 Ontology Retrieval이다.\
> LLM은 검색된 GitBook 근거만 바탕으로 답변한다.\
> 이후 이 구조를 기반으로 문서 검색, Knowledge Graph, 문서 품질 관리, 평가, 개인화 등 다양한 방향으로 확장한다.
