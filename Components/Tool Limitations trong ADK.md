---
title: Tool Limitations trong ADK
tags:
  - adk
  - components
  - tools
  - limitations
created: 2026-06-07
up:
  - "[[Components]]"
source:
  - https://adk.dev/tools/limitations/
---

# Tool Limitations trong ADK

Liên kết: [[Components]]

## Tool limitations là gì?

`Tool limitations` là các giới hạn khi dùng tool trong ADK.

Không phải tool nào cũng có thể trộn chung trong cùng một agent.

Một số built-in tool đặc biệt có thể yêu cầu:

- chỉ dùng một mình trong một agent,
- không đặt chung với custom tool,
- không đặt bên trong `sub_agents`,
- hoặc phải tách thành agent riêng rồi gọi qua `AgentTool`.

Trang này quan trọng khi thiết kế agent có nhiều tool như:

- Google Search,
- Agent Search,
- Code Execution,
- custom function tool,
- nhiều specialist agent.

## Ý chính

Với custom tool bình thường, một agent có thể có nhiều tool.

Ví dụ:

```python
root_agent = Agent(
    name="RootAgent",
    tools=[tool_a, tool_b, tool_c],
)
```

Nhưng với một số built-in tool đặc biệt, không nên đặt chung với tool khác trong cùng một agent.

Ví dụ không supported:

```python
root_agent = Agent(
    name="RootAgent",
    model="gemini-flash-latest",
    tools=[custom_function],
    code_executor=BuiltInCodeExecutor(),
)
```

Agent trên vừa có `custom_function`, vừa có `BuiltInCodeExecutor`. Theo doc, cách này không supported trong trường hợp tool bị giới hạn.

## Các tool có giới hạn đặc biệt

Doc nhắc đến các nhóm tool sau:

- Code Execution với Gemini API.
- Google Search với Gemini API.
- Agent Search.

Các tool này có thể phải đứng một mình trong một agent object.

Nghĩa là không nên thiết kế như sau:

```text
Một agent
  tools:
    - Google Search
    - custom tool
    - Code Execution
```

Thay vào đó nên tách thành nhiều agent chuyên trách.

## Workaround chính: tách agent rồi gọi bằng AgentTool

Pattern được doc khuyến nghị là:

```text
RootAgent
  tools:
    - SearchAgent as AgentTool
    - CodeAgent as AgentTool
```

Mỗi specialist agent giữ một built-in tool đặc biệt.

Root agent không trực tiếp nhét tất cả built-in tool vào cùng một list.

Ví dụ:

```python
from google.adk.agents import Agent
from google.adk.tools.agent_tool import AgentTool
from google.adk.tools import google_search
from google.adk.code_executors import BuiltInCodeExecutor

search_agent = Agent(
    name="SearchAgent",
    model="gemini-flash-latest",
    instruction="You're a specialist in Google Search.",
    tools=[google_search],
)

coding_agent = Agent(
    name="CodeAgent",
    model="gemini-flash-latest",
    instruction="You're a specialist in Code Execution.",
    code_executor=BuiltInCodeExecutor(),
)

root_agent = Agent(
    name="RootAgent",
    model="gemini-flash-latest",
    description="Root Agent",
    tools=[
        AgentTool(agent=search_agent),
        AgentTool(agent=coding_agent),
    ],
)
```

Flow:

```text
User
  -> RootAgent
      -> gọi SearchAgent qua AgentTool nếu cần search
      -> gọi CodeAgent qua AgentTool nếu cần chạy code
      -> tổng hợp kết quả
  -> User
```

Điểm quan trọng:

- `SearchAgent` chỉ lo search.
- `CodeAgent` chỉ lo code execution.
- `RootAgent` lo điều phối và tổng hợp.
- Built-in tool đặc biệt không bị trộn chung trong một agent.

## Vì sao AgentTool hợp ở đây?

`AgentTool` biến một agent khác thành tool của root agent.

Như vậy root agent có thể dùng nhiều năng lực:

```text
search
code execution
custom logic
specialist reasoning
```

nhưng mỗi năng lực nhạy cảm vẫn nằm trong một agent riêng.

Đây là pattern tốt khi gặp giới hạn:

```text
Không trộn tool trong một agent.
Tách thành nhiều specialist agents.
Root gọi specialist bằng AgentTool.
```

## Cảnh báo về sub_agents

Doc cũng cảnh báo rằng built-in tools không nên dùng bên trong `sub_agents`, trừ một số ngoại lệ/workaround cụ thể.

Ví dụ không supported:

```python
url_context_agent = Agent(
    name="UrlContextAgent",
    model="gemini-flash-latest",
    instruction="You're a specialist in URL Context.",
    tools=[url_context],
)

coding_agent = Agent(
    name="CodeAgent",
    model="gemini-flash-latest",
    instruction="You're a specialist in Code Execution.",
    code_executor=BuiltInCodeExecutor(),
)

root_agent = Agent(
    name="RootAgent",
    model="gemini-flash-latest",
    sub_agents=[
        url_context_agent,
        coding_agent,
    ],
)
```

Vấn đề:

```text
RootAgent
  sub_agents:
    - UrlContextAgent dùng built-in tool
    - CodeAgent dùng built-in code execution
```

Theo doc, built-in tools trong subagent kiểu này không supported.

Cách an toàn hơn:

```python
root_agent = Agent(
    name="RootAgent",
    model="gemini-flash-latest",
    tools=[
        AgentTool(agent=url_context_agent),
        AgentTool(agent=coding_agent),
    ],
)
```

Tức là dùng `AgentTool` thay vì `sub_agents` cho các specialist agent có built-in tool bị giới hạn.

## Workaround bypass_multi_tools_limit

Doc có nhắc workaround `bypass_multi_tools_limit=True`.

Trong ADK Python, workaround này áp dụng cho:

- `GoogleSearchTool`,
- `VertexAiSearchTool`.

Mục đích là bypass giới hạn multi-tool cho một số search tool.

Ví dụ ý tưởng:

```python
GoogleSearchTool(
    bypass_multi_tools_limit=True,
)
```

Lưu ý:

- Đây không phải workaround chung cho mọi tool.
- Không nên áp dụng suy diễn cho Code Execution hoặc mọi built-in tool khác.
- Cần kiểm tra version ADK đang dùng.

## Lưu ý về version

Doc ghi rõ giới hạn `one tool per agent` chỉ áp dụng cho Search trong ADK Python v1.15.0 trở xuống.

Từ ADK Python v1.16.0 trở lên, ADK có built-in workaround để bỏ giới hạn này cho Search.

Nhưng vẫn cần cẩn thận với từng loại tool:

- Code Execution với Gemini API có giới hạn riêng.
- Google Search có khác biệt theo model/version.
- Agent Search có giới hạn riêng và hiện không có trong TypeScript.

Vì vậy khi dùng built-in tool đặc biệt, nên đọc doc của tool đó trước khi trộn nhiều tool trong một agent.

## TypeScript notes

Doc có một vài ghi chú riêng cho TypeScript:

- Code Execution trong TypeScript yêu cầu Gemini 2.0+ và không có limitation giống Python trong phần này.
- Google Search limitation trong TypeScript chỉ áp dụng với Gemini 1.x models.
- Agent Search hiện không available trong TypeScript.

Điều này có nghĩa là limitation có thể khác nhau theo ngôn ngữ và model.

Không nên copy kiến trúc Python sang TypeScript mà không kiểm tra lại doc/version.

## Khi nào nên tách agent?

Nên tách agent nếu:

- một agent cần dùng Code Execution và tool khác,
- một agent cần dùng Google Search và custom tool trong version bị giới hạn,
- muốn dùng nhiều built-in tool đặc biệt cùng lúc,
- muốn root agent có nhiều specialist nhưng không muốn vi phạm tool limitation,
- tool behavior bị runtime báo không supported.

Pattern:

```text
SearchAgent
  -> chỉ search

CodeAgent
  -> chỉ code execution

RootAgent
  -> AgentTool(SearchAgent)
  -> AgentTool(CodeAgent)
```

## Khi nào không cần tách?

Không cần tách nếu:

- chỉ dùng custom function tools bình thường,
- tool không nằm trong nhóm built-in tool bị giới hạn,
- version ADK đang dùng đã có workaround phù hợp,
- agent chỉ có một built-in tool và không cần tool khác.

Ví dụ bình thường:

```python
assistant = Agent(
    name="Assistant",
    tools=[get_weather, get_time, lookup_order],
)
```

Nếu `get_weather`, `get_time`, `lookup_order` là custom function tools bình thường, thiết kế này thường ổn.

## So sánh sub_agents và AgentTool trong phần này

Với built-in tool bị giới hạn, doc ưu tiên pattern `AgentTool`.

| Cách dùng | Khi nào hợp | Ghi chú |
|---|---|---|
| `tools=[AgentTool(agent=...)]` | Root gọi specialist như tool | Workaround chính trong doc |
| `sub_agents=[...]` | Agent team/delegation thông thường | Không phù hợp nếu subagent dùng built-in tool bị giới hạn |
| Nhiều custom tools trong một agent | Tool thường, không bị limitation | Thường ổn |
| Nhiều built-in tools đặc biệt trong một agent | Cần kiểm tra kỹ | Có thể không supported |

## Quy tắc nhớ nhanh

```text
Custom tools thường:
  -> có thể đặt nhiều tool trong một agent.

Built-in tools đặc biệt:
  -> kiểm tra limitation.

Nếu bị limitation:
  -> tách thành specialist agent.
  -> root gọi specialist qua AgentTool.

Không chắc:
  -> ưu tiên tách agent, thiết kế rõ hơn và ít rủi ro hơn.
```

## Ví dụ kiến trúc tốt

```text
RootAgent
  instruction:
    - nếu cần thông tin mới, gọi SearchAgent
    - nếu cần tính toán/chạy code, gọi CodeAgent
    - nếu cần tổng hợp, tự tổng hợp

  tools:
    - AgentTool(SearchAgent)
    - AgentTool(CodeAgent)

SearchAgent
  tools:
    - Google Search

CodeAgent
  code_executor:
    - BuiltInCodeExecutor
```

Ưu điểm:

- Mỗi agent có trách nhiệm rõ.
- Dễ debug.
- Dễ thay model/tool cho từng specialist.
- Tránh trộn built-in tool bị giới hạn.

## Ghi nhớ

Trang này trả lời câu hỏi:

> Tại sao không nên nhét Google Search, Code Execution và custom tools vào cùng một agent?

Câu trả lời:

> Vì một số built-in tool có limitation. Cách an toàn là tách thành specialist agents và để root agent gọi chúng bằng AgentTool.

## Source

- [Limitations for ADK tools](https://adk.dev/tools/limitations/)
