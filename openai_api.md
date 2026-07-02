# OpenAI API
## 1. JSON 모드
```python
response_format={"type": "json_object"}
```
AI 응답이 JSON 형식으로는 보장되지만 `필드명`, `타입`, `필드 존재 여부`는 보장되지 않음

## 2. Structured Output
AI 응답을 **미리 정의한 JSON 구조로 강제**하는 기능  

```python
from pydantic import BaseModel
from openai import OpenAI

client = OpenAI()


class Person(BaseModel):
    name: str
    age: int

response = client.responses.parse(
    model="gpt-5.4-nano",
    input="김철수는 25살입니다.",
    text_format=User # Structured Output
)

user = response.output_parsed  # 이 부분이 강력함

print(user.name)
print(user.age)
```

## 3. Function Calling
AI가 **어떤 함수를 호출해야 하는지 판단**하고, **함수에 전달할 인자를 생성**하는 기능
- 함수 실행 자체는 코드가 담당
- Function Calling을 잘 설계하는 것은 프롬프트 엔지니어링 성격이 있음 (Tool 설계 및 연결)
    - 회사 규정 문서를 검색한다. → 사내 정책, 인사 규정, 휴가 규정을 검색한다.
- 실무에서는 Function Calling으로 데이터를 가져오고, Structured Outputs로 결과를 정형화해서 반환하는 경우가 많음

```python
from openai import OpenAI
import json

client = OpenAI()


def search_documents(query: str):
    return [
        "연차는 최소 3일 전에 신청해야 합니다."
    ]


tools = [ # 여러개 정의 가능
    {
        "type": "function",
        "name": "search_documents",
        "description": "사내 문서를 검색한다.",
        "parameters": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string"
                }
            },
            "required": ["query"]
        }
    }
]

response = client.responses.create(
    model="gpt-5",
    input="연차 규정을 알려줘",
    tools=tools # Function Calling
)
```

## 4. Agents SDK
여러 단계의 작업을 스스로 **계획**하고, **도구를 사용**하며, 필요하면 **다른 Agent에게 작업을 넘겨서** 최종 결과를 만들어내는 AI 애플리케이션  

단순히 질문하고 답변받는 수준이 아니라
- 여러 Agent를 만들고
- Agent가 Tool을 사용하고
- 필요하면 다른 Agent에게 일을 넘기고(Handoff)
- 입력/출력을 검증하고(Guardrails)

복잡한 업무를 자동화할 수 있게 함

### 핵심 구성요소
#### 1) Agent
- 역할
- 시스템 프롬프트
- 사용할 Tool

을 가짐

#### 2) Tool
Agent가 사용할 함수

#### 3) Handoff
Agent가 **다른 Agent에게 작업을 넘기는 기능**
```
사용자
  ↓
Manager Agent
  ↓
 ├─ 여행 관련 → Travel Agent
 ├─ 코딩 관련 → Coding Agent
 └─ 금융 관련 → Finance Agent
 ```

#### 4) Guardrails
**입력**에 계좌 비밀번호처럼 민감한 정보가 들어있진 않는지  
**출력**이 지정 형식과 일치하는지 **검증**하는 장치

#### 예제 코드
```python
import asyncio
from agents import Agent, Runner, function_tool


@function_tool
def get_weather(city: str) -> str:
    """도시의 날씨를 반환합니다."""
    return f"{city}의 현재 날씨는 맑음입니다."


weather_agent = Agent(
    name="Weather Agent",
    instructions="날씨 질문에 답변하세요. 필요한 경우 get_weather 도구를 사용하세요.",
    tools=[get_weather],
)

general_agent = Agent(
    name="General Agent",
    instructions="일반적인 질문에 답변하세요.",
)

manager_agent = Agent(
    name="Manager Agent",
    instructions="""
    사용자의 질문을 보고 적절한 Agent에게 넘기세요.
    날씨 관련 질문이면 Weather Agent에게 넘기세요.
    일반 질문이면 직접 답변하거나 General Agent에게 넘기세요.
    """,
    handoffs=[weather_agent, general_agent],
)

# 비동기 방식
result = await Runner.run(
    agent,
    "서울 날씨 알려줘."
)

# 동기 방식
result = Runner.run_sync(
    agent,
    "서울 날씨 알려줘."
)

print(result.final_output)
```
