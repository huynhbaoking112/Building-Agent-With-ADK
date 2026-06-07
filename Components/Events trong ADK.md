---
title: Events trong ADK
tags:
  - adk
  - components
  - sessions
  - events
  - state
created: 2026-06-07
up:
  - "[[Sessions trong ADK]]"
source:
  - https://adk.dev/events/
---

# Events trong ADK

Liên kết: [[Sessions trong ADK]]

## Events là gì?

`Event` trong ADK là đơn vị thông tin chính chảy qua toàn bộ hệ thống agent.

Nói ngắn:

```text
Event = một bản ghi bất biến mô tả một chuyện vừa xảy ra trong quá trình agent chạy.
```

Event có thể đại diện cho:

- user gửi message,
- agent trả lời,
- LLM yêu cầu gọi tool,
- tool trả kết quả,
- state được cập nhật,
- artifact được lưu,
- agent chuyển quyền cho agent khác,
- loop bị escalate/dừng,
- lỗi từ model/tool/framework.

ADK vận hành bằng event stream.

```text
Runner chạy agent
  -> agent/model/tool sinh Events
  -> SessionService lưu Events
  -> app/UI đọc Events
```

## Vì sao Events quan trọng?

Events là trung tâm của ADK vì chúng đảm nhiệm nhiều việc cùng lúc.

| Vai trò | Ý nghĩa |
|---|---|
| Communication | Định dạng message chuẩn giữa UI, Runner, agent, LLM và tools |
| State & artifact changes | Mang `state_delta` và `artifact_delta` để `SessionService` apply |
| Control flow | Mang tín hiệu như transfer agent, escalate, skip summarization |
| History & observability | Tạo timeline trong `session.events` để debug/audit |

Nói cách khác:

```text
Muốn hiểu agent đã làm gì -> đọc events.
Muốn state update đúng -> state change phải đi qua event actions.
Muốn UI hiển thị đúng -> lọc final response từ event stream.
```

## Event nằm ở đâu?

Trong một `Session`, history được lưu ở:

```text
session.events
```

Nếu `Session` là conversation thread, thì `Events` là timeline của thread đó.

Ví dụ:

```text
Session
  events:
    1. User hỏi đặt vé
    2. Agent gọi tool find_flights
    3. Tool trả kết quả
    4. Agent hỏi user chọn chuyến nào
    5. State update booking_step = "select_flight"
```

## Event object có gì?

Một event có các field quan trọng:

| Field | Ý nghĩa |
|---|---|
| `author` | Ai tạo event: `user` hoặc tên agent |
| `content` | Nội dung chính: text, function call, function response |
| `actions` | Side effects/control signals: state delta, artifact delta, transfer, escalate |
| `invocation_id` | ID của toàn bộ lượt xử lý user request hiện tại |
| `id` | ID riêng của event đó |
| `timestamp` | thời điểm tạo |
| `partial` | có phải streaming chunk chưa hoàn chỉnh không |
| `branch` | path phân cấp trong multi-agent/workflow |

Hình dung đơn giản:

```python
class Event:
    content = ...
    partial = ...

    author = ...
    invocation_id = ...
    id = ...
    timestamp = ...
    actions = ...
    branch = ...
```

Trong TypeScript, event là interface `Event`.

Trong Go, event là struct trong package session.

Trong Java/Kotlin, event là class tương ứng của ADK SDK.

## author

`event.author` cho biết event đến từ đâu.

Ví dụ:

```text
author = "user"
  -> input trực tiếp từ end-user

author = "WeatherAgent"
  -> output/action từ WeatherAgent

author = "SummarizerAgent"
  -> output/action từ SummarizerAgent
```

Khi viết custom agent và tự yield event, nên set `author` đúng tên agent.

## invocation_id

`event.invocation_id` là ID của toàn bộ lượt xử lý từ user request đến final output.

Một invocation có thể sinh nhiều events.

Ví dụ:

```text
invocation_id = inv_123
  Event 1: user message
  Event 2: tool call
  Event 3: tool result
  Event 4: final response
```

Dùng `invocation_id` cho:

- logging,
- tracing,
- debug một lượt xử lý,
- gom các event thuộc cùng một user request.

## event.id

`event.id` là ID riêng của từng event.

Khác với `invocation_id`:

```text
invocation_id -> chung cho cả lượt xử lý.
event.id -> riêng cho một event cụ thể.
```

Dùng `event.id` khi cần reference một occurrence cụ thể trong event stream.

## content

`event.content` là payload chính.

Nó có thể chứa:

- text response,
- function call,
- function response,
- code result,
- content dạng khác.

Khi đọc text, luôn kiểm tra tồn tại trước:

```python
if event.content and event.content.parts:
    text = event.content.parts[0].text
```

Đừng giả định mọi event đều có text.

Tool call, tool result, state-only update, control signal có thể không có text user-facing.

## Tool call event

Khi LLM muốn gọi tool, event có function call.

Trong Python, kiểm tra:

```python
calls = event.get_function_calls()
if calls:
    for call in calls:
        tool_name = call.name
        args = call.args
```

Ý nghĩa:

```text
LLM đang yêu cầu framework/app chạy tool có tên này với args này.
```

Ví dụ:

```text
function_call:
  name = "find_airports"
  args = {"city": "London"}
```

Tool call event thường không phải final response cho user.

## Tool result event

Khi tool chạy xong, ADK tạo event chứa function response.

Trong Python, kiểm tra:

```python
responses = event.get_function_responses()
if responses:
    for response in responses:
        tool_name = response.name
        result = response.response
```

Ví dụ:

```text
function_response:
  name = "find_airports"
  response = {"result": ["LHR", "LGW", "STN"]}
```

Lưu ý trong doc:

```text
Tool result event thường có author là agent đã request tool.
Role trong content có thể là "user" để cấu trúc history cho LLM.
```

Đây là chi tiết dễ nhầm:

```text
author nói event thuộc agent nào.
role trong content phục vụ format history cho model.
```

## partial và streaming

`event.partial` cho biết event có phải streaming chunk chưa hoàn chỉnh không.

```text
partial = True
  -> còn text tiếp theo

partial = False hoặc None
  -> content chunk này đã hoàn chỉnh
```

UI streaming thường gom text từ nhiều partial events.

Ví dụ:

```python
full_response = ""

async for event in runner.run_async(...):
    if event.partial and event.content and event.content.parts:
        full_response += event.content.parts[0].text
```

Không nên coi mọi partial event là final answer.

## actions

`event.actions` chứa side effects và control signals.

Các action quan trọng:

| Action | Ý nghĩa |
|---|---|
| `state_delta` | các key state được thay đổi |
| `artifact_delta` | artifact nào được lưu/cập nhật |
| `transfer_to_agent` | chuyển quyền xử lý sang agent khác |
| `escalate` | báo hiệu dừng/thoát loop |
| `skip_summarization` | không cần LLM summarize tool result |

Nói ngắn:

```text
content = message/data chính.
actions = side effect hoặc điều khiển flow.
```

## state_delta

`event.actions.state_delta` là các thay đổi state được tạo trong bước đó.

Ví dụ:

```python
delta = event.actions.state_delta
```

Có thể là:

```python
{
    "booking_step": "confirm_payment",
    "selected_flight_id": "VN123",
}
```

Khi `SessionService.append_event()` xử lý event, nó apply delta này vào `session.state`.

Điểm quan trọng:

```text
State update đúng trong ADK là đi qua event.actions.state_delta.
```

## artifact_delta

`event.actions.artifact_delta` cho biết artifact nào đã được lưu/cập nhật.

Ví dụ:

```python
artifact_changes = event.actions.artifact_delta
```

Có thể là:

```python
{
    "verification_doc.pdf": 2
}
```

Ý nghĩa:

```text
verification_doc.pdf được save/update tới version 2.
```

UI có thể dùng event này để refresh artifact list.

Lưu ý:

```text
Việc save artifact thật thường xảy ra trước đó khi gọi context.save_artifact.
artifact_delta ghi lại metadata/update signal trong event.
```

## Control flow signals

Events có thể mang tín hiệu điều khiển flow.

### transfer_to_agent

```python
event.actions.transfer_to_agent
```

Ý nghĩa:

```text
Chuyển control sang agent được chỉ định.
```

Ví dụ:

```text
OrchestratorAgent
  -> transfer_to_agent = "BillingAgent"
```

### escalate

```python
event.actions.escalate
```

Ý nghĩa:

```text
Loop hoặc workflow nên terminate/escalate.
```

Ví dụ:

```text
CheckerAgent đã retry tối đa
  -> escalate = true
```

### skip_summarization

```python
event.actions.skip_summarization
```

Ý nghĩa:

```text
Tool result không cần được LLM summarize.
```

Khi `skip_summarization=True`, tool result có thể được coi là final response tùy trường hợp.

## Final response

Không phải event nào cũng nên hiển thị cho user.

Các event trung gian gồm:

- tool call,
- tool result,
- state update,
- artifact update,
- control signal,
- partial streaming chunk.

Doc khuyên dùng helper:

```python
event.is_final_response()
```

để biết event nào là output user-facing.

Ví dụ:

```python
async for event in runner.run_async(...):
    if event.is_final_response():
        if event.content and event.content.parts:
            print(event.content.parts[0].text)
```

`is_final_response()` có thể true khi:

- event là tool result và `skip_summarization=True`,
- event là long-running tool call,
- hoặc event là complete message không có function call/response, không phải partial, không cần xử lý thêm.

## Event flow

Events được tạo ở nhiều điểm:

| Nguồn | Event |
|---|---|
| User input | Runner wrap user message thành event `author="user"` |
| Agent logic | Agent yield/emit event |
| LLM response | Model integration chuyển raw model output thành event |
| Tool result | Framework tạo event chứa function response |

Flow xử lý:

```text
1. Event được tạo/yield/emit
2. Runner nhận event
3. Runner gửi event tới SessionService
4. SessionService apply state_delta/artifact_delta
5. SessionService set id/timestamp nếu cần
6. SessionService append event vào session.events
7. Runner yield event ra app/UI
```

Điểm quan trọng:

```text
SessionService là nơi event được persist và delta được apply.
```

## Vì sao state update gắn với event?

Nếu state update không gắn với event, bạn sẽ khó trả lời:

```text
State này đổi khi nào?
Ai đổi?
Invocation nào đổi?
Tool nào gây ra đổi?
Trước đó và sau đó flow ra sao?
```

Khi state update đi qua event:

```text
event.actions.state_delta
  -> session.events có record
  -> SessionService apply vào state
  -> debug/audit dễ hơn
```

Đây là lý do doc cảnh báo không sửa trực tiếp `session.state` lấy từ `SessionService`.

## Các pattern event thường gặp

### User input

```text
author = "user"
content = text user gửi
actions thường empty
```

### Agent final text response

```text
author = "TravelAgent"
content = text agent trả lời
partial = false
turn_complete = true
is_final_response() = true
```

### Agent streaming text response

```text
author = "SummaryAgent"
content = text chunk
partial = true
turn_complete = false
is_final_response() = false
```

### Tool call request

```text
author = "TravelAgent"
content.parts chứa function_call
is_final_response() = false
```

### Tool result

```text
author = agent đã request tool
content.parts chứa function_response
is_final_response() phụ thuộc skip_summarization
```

### State/artifact update only

```text
content = null
actions.state_delta = {...}
actions.artifact_delta = {...}
is_final_response() = false
```

### Agent transfer signal

```text
actions.transfer_to_agent = "BillingAgent"
is_final_response() = false
```

### Loop escalation signal

```text
actions.escalate = true
is_final_response() = false
```

## ToolContext.function_call_id

Khi LLM request một tool, function call đó có ID.

Tool nhận `ToolContext.function_call_id`.

Ý nghĩa:

```text
Liên kết action của tool với đúng tool request.
```

Quan trọng khi:

- một turn có nhiều tool calls,
- tool cần auth,
- framework cần biết credential/action thuộc tool call nào.

## Error events

Event cũng có thể đại diện cho lỗi.

Kiểm tra:

```python
event.error_code
event.error_message
```

Lỗi có thể đến từ:

- LLM,
- safety filter,
- resource limits,
- tool failure nghiêm trọng,
- framework packaging error.

Tool-specific error thông thường có thể nằm trong `FunctionResponse` content.

## Best practices

### Set author rõ ràng khi tự tạo event

Khi viết custom agent và tự yield event:

```python
yield Event(author=self.name, ...)
```

Authorship đúng giúp debug history dễ hơn.

### Tách content và actions

Dùng:

```text
event.content -> message/data chính
event.actions -> side effects/control signals
```

Không trộn state update vào text nếu mục tiêu là update state.

### Dùng is_final_response cho UI

UI không nên tự đoán event nào là final.

Nên:

```python
if event.is_final_response():
    display(...)
```

### Debug bằng session.events

Khi agent chạy sai, đọc sequence:

```text
author
content
function calls
function responses
actions
invocation_id
```

Đây là cách nhanh nhất để biết agent sai ở đâu.

### Dùng invocation_id để trace

Mọi event trong cùng một lượt user request có chung `invocation_id`.

Dùng nó để gom log:

```text
invocation_id = inv_123
  -> tất cả events của request đó
```

### Cẩn thận khi re-process events

Vì event actions có thể chứa state/artifact changes, app cần hiểu `SessionService` là nơi apply delta.

Nếu tự re-process events bên ngoài, cần tránh tạo side effect trùng.

## Ghi nhớ

Trang này trả lời câu hỏi:

> ADK ghi lại và điều phối mọi thứ xảy ra trong agent bằng gì?

Câu trả lời:

> Bằng `Event`.

Quy tắc nhanh:

```text
Session = conversation thread.
Events = timeline trong thread.
State = dữ liệu nhỏ hiện tại.
EventActions = state/artifact/control changes.
Runner = sinh/yield events.
SessionService = lưu event và apply deltas.
```

Khi làm app/UI:

```text
Đọc event stream từ Runner.
Dùng is_final_response() để hiển thị output cuối.
Gom partial events nếu streaming.
Đọc actions để biết state/artifact/control changes.
Debug bằng session.events.
```

## Source

- [Events](https://adk.dev/events/)
