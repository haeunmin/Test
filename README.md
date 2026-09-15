# Test

### Gitbook Agent Test

MCP를 아주 쉽게 말하면, **AI가 외부 세계와 연결되는 공통 규격**이에요.

예를 들어 AI한테 이런 일을 시킨다고 해볼게요.

> “우리 회사 DB에서 지난달 매출 찾아서 요약해줘.”

AI 자체는 보통 회사 DB에 직접 접근할 수 없습니다. 그런데 DB를 연결해주는 **MCP 서버**가 있으면 이렇게 동작할 수 있어요.

**사용자 → AI → MCP → 회사 DB → MCP → AI → 사용자**

AI는 MCP를 통해 “어떤 기능을 사용할 수 있는지” 확인하고, 필요한 기능을 호출한 뒤 결과를 받아 답을 만들어냅니다.

### 1. MCP가 없으면

서비스마다 AI 연결 방식을 따로 만들어야 합니다.

```text
AI ──전용 코드── GitHub
AI ──전용 코드── Slack
AI ──전용 코드── PostgreSQL
AI ──전용 코드── Google Drive
```

연결 방식이 전부 제각각이라 개발하기 귀찮아집니다.

### 2. MCP가 있으면

각 서비스가 MCP라는 공통 규격을 따르면 됩니다.

```text
            ┌─ GitHub MCP Server
            │
AI ─ MCP ───┼─ Slack MCP Server
            │
            ├─ PostgreSQL MCP Server
            │
            └─ Google Drive MCP Server
```

그래서 제가 앞서 **“AI용 USB”**라고 표현한 거예요.

USB 규격이 있으니까 마우스, 키보드, 외장하드를 비슷한 방식으로 컴퓨터에 연결할 수 있듯이, MCP를 사용하면 여러 데이터와 도구를 AI에 비슷한 방식으로 연결할 수 있습니다.

### 3. 가장 중요한 개념: MCP Server

MCP를 이해할 때 가장 자주 보게 되는 게 **MCP Server**입니다.

예를 들어 GitHub MCP Server가 있다고 합시다. 이 서버가 AI에게 이런 기능을 제공할 수 있어요.

```text
사용 가능한 기능

- repository 목록 조회
- issue 조회
- issue 생성
- pull request 조회
- 코드 검색
```

그러면 사용자가

> “우리 프로젝트에서 열려 있는 버그 이슈 보여줘.”

라고 하면 AI가 대략 이렇게 판단합니다.

```text
1. GitHub MCP Server가 연결되어 있네.
2. issue 조회 기능이 있네.
3. 이 기능을 호출하자.
4. 결과를 받아서 사용자에게 설명하자.
```

즉 MCP 서버는 **AI가 사용할 수 있는 능력을 제공하는 어댑터**라고 생각하면 됩니다.

### 4. MCP에서 자주 나오는 3가지

MCP 서버는 대표적으로 **Tools, Resources, Prompts** 같은 것을 제공할 수 있습니다.

**Tools**는 AI가 실행할 수 있는 기능입니다.

```text
create_issue()
search_database()
send_message()
get_weather()
```

예:

> “Slack에 회의가 3시라고 알려줘.”

AI가 MCP Tool을 호출해서 메시지를 보낼 수 있습니다.

**Resources**는 AI가 읽을 수 있는 데이터입니다.

```text
회사 문서
파일
DB 데이터
로그
Git 저장소 내용
```

예:

> “우리 회사 휴가 규정 알려줘.”

AI가 MCP를 통해 회사 문서를 읽고 답할 수 있습니다.

**Prompts**는 미리 만들어놓은 작업 템플릿에 가깝습니다.

예를 들면:

```text
코드 리뷰하기
버그 분석하기
회의록 정리하기
```

같은 워크플로를 서버 쪽에서 제공할 수도 있습니다.

### 5. 실제 예시

개발자가 PostgreSQL용 MCP Server를 만들어놓았다고 해보겠습니다.

AI가 사용할 수 있는 기능이:

```text
list_tables
describe_table
query_database
```

라고 합시다.

사용자가 질문합니다.

> “지난달 가장 많이 팔린 상품 10개 알려줘.”

AI 내부에서는 개념적으로 이런 일이 벌어집니다.

```text
사용자
  ↓
AI

"DB 조회가 필요하겠군."

  ↓

MCP Client
  ↓

PostgreSQL MCP Server
  ↓

query_database(
   "SELECT ..."
)

  ↓

Database
  ↓

결과 반환
  ↓

AI

"1위는 A 상품,
 2위는 B 상품..."
```

중요한 점은 **AI가 SQL 결과를 직접 가지고 있던 게 아니라 MCP를 이용해서 외부에서 가져왔다는 것**입니다.

### 6. 개발자는 MCP Server를 이렇게 생각하면 됩니다

아주 단순화하면 MCP Server는 이런 코드와 비슷합니다.

```python
@mcp.tool()
def get_sales(month: str):
    return database.query(
        "SELECT * FROM sales WHERE month = ?",
        month
    )
```

AI 입장에서는

```text
get_sales라는 도구가 있구나.
month라는 값을 넣어야 하는구나.
```

라고 이해합니다.

사용자가

> “8월 매출 알려줘.”

라고 하면 AI가 알아서

```python
get_sales(month="2026-08")
```

같은 호출을 하는 구조입니다.

실제 MCP 구현에는 통신 방식과 스키마 등 더 많은 요소가 있지만, 핵심 아이디어는 이겁니다.

### 7. 그래서 MCP가 왜 요즘 많이 나오냐면

LLM 자체의 능력만으로는 할 수 있는 일이 제한적이기 때문입니다.

AI 혼자서는 보통:

```text
회사 DB 접근 ❌
내 파일 접근 ❌
GitHub 작업 ❌
Slack 메시지 전송 ❌
사내 API 호출 ❌
```

그런데 MCP를 연결하면:

```text
회사 DB 조회 ✅
문서 검색 ✅
GitHub 작업 ✅
Slack 연동 ✅
API 사용 ✅
```

이 가능해집니다.

그래서 **MCP의 핵심은 “AI에게 지식만 주는 게 아니라 실제 도구를 쓸 수 있게 한다”**는 데 있습니다.

마지막으로 이것만 기억하면 MCP의 80%는 이해한 겁니다.

```text
LLM = 두뇌

MCP = 연결 규격

MCP Server = AI가 사용할 수 있는 도구/데이터 제공자
```

예를 들어,

```text
Claude / ChatGPT / IDE의 AI
            ↓
           MCP
            ↓
GitHub / DB / 파일 / Slack / 사내 시스템
```

이라는 구조입니다.

원하시면 다음 단계로 **“MCP 서버를 직접 만드는 법”**을 Python 기준으로 아주 간단한 예제부터 설명해드릴 수 있어요.
