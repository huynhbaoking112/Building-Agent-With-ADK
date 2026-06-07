---
title: State trong ADK
tags:
  - adk
  - components
  - sessions
  - state
  - memory
created: 2026-06-07
up:
  - "[[Sessions trong ADK]]"
source:
  - https://adk.dev/sessions/state/
---

# State trong ADK

Liên kết: [[Sessions trong ADK]]

## State là gì?

`session.state` trong ADK là scratchpad của một session.

Nói ngắn:

```text
State = dữ liệu nhỏ, động, có thể thay đổi trong lúc hội thoại/workflow chạy.
```

Trong một `Session`, `events` giữ lịch sử đầy đủ đã xảy ra, còn `state` giữ các chi tiết agent cần nhớ nhanh để xử lý lượt tiếp theo.

Ví dụ:

```python
session.state["current_booking_step"] = "confirm_payment"
session.state["departure_city"] = "Hanoi"
session.state["user_is_authenticated"] = True
```

State giúp agent:

- cá nhân hóa tương tác,
- theo dõi tiến độ task,
- tích lũy thông tin nhỏ,
- lưu flag để ra quyết định,
- truyền dữ liệu nhỏ giữa agent, callback và tool.

## State khác Events thế nào?

| Cơ chế | Vai trò |
|---|---|
| `session.events` | Lịch sử tuần tự: user message, agent response, tool call, tool result |
| `session.state` | Key-value data hiện tại để agent theo dõi workflow |

Ví dụ:

```text
events:
  - user nói muốn đặt vé
  - agent hỏi ngày bay
  - user chọn ngày

state:
  destination = "Đà Nẵng"
  booking_step = "select_flight"
```

Events cho biết chuyện gì đã xảy ra.

State cho biết hiện tại workflow đang ở đâu và cần nhớ gì.

## State là key-value

State là collection dạng dictionary/map.

Key:

```text
Luôn là string.
```

Value:

```text
Nên là dữ liệu serializable.
```

Các kiểu nên dùng:

- string,
- number,
- boolean,
- list đơn giản,
- dict/map đơn giản.

Ví dụ tốt:

```python
session.state["order_id"] = "ORD-123"
session.state["retry_count"] = 2
session.state["user_is_authenticated"] = True
session.state["shopping_cart_items"] = ["book", "pen"]
```

Không nên lưu object phức tạp:

```python
session.state["db_connection"] = connection
session.state["custom_object"] = MyClass()
session.state["callback_fn"] = some_function
```

Nếu cần object phức tạp, lưu ID:

```python
session.state["order_id"] = "ORD-123"
```

Rồi lấy object thật từ database hoặc service khác.

## Persistence phụ thuộc SessionService

State có persist qua app restart hay không phụ thuộc vào `SessionService`.

| Service | State có bền không? |
|---|---|
| `InMemorySessionService` | Không, app restart là mất |
| `DatabaseSessionService` | Có |
| `VertexAiSessionService` | Có |

Vì vậy:

```text
Dev/test -> InMemorySessionService ổn.
Production cần giữ state -> DatabaseSessionService hoặc VertexAiSessionService.
```

## Prefix của state

Prefix trong key quyết định scope và persistence behavior.

| Prefix | Scope | Ví dụ | Khi dùng |
|---|---|---|---|
| không prefix | session hiện tại | `current_step` | dữ liệu riêng của conversation này |
| `user:` | user hiện tại, qua nhiều session cùng app | `user:preferred_language` | preference/profile user |
| `app:` | toàn app, mọi user/session | `app:api_endpoint` | config/shared setting |
| `temp:` | invocation hiện tại | `temp:raw_api_response` | dữ liệu trung gian trong một lượt xử lý |

Ví dụ:

```python
state["current_step"] = "payment"
state["user:preferred_language"] = "vi"
state["app:global_discount_code"] = "SAVE10"
state["temp:last_tool_result"] = {"status": "ok"}
```

## Không prefix: Session State

Không có prefix nghĩa là state thuộc session hiện tại.

Ví dụ:

```python
session.state["current_intent"] = "book_flight"
session.state["current_booking_step"] = "select_date"
session.state["needs_clarification"] = True
```

Dùng cho:

- workflow step,
- lựa chọn user trong thread hiện tại,
- dữ liệu task hiện tại,
- flag chỉ liên quan conversation này.

Scope:

```text
app_name + user_id + session_id
```

## user: User State

Prefix `user:` nghĩa là state gắn với user.

Ví dụ:

```python
session.state["user:preferred_language"] = "vi"
session.state["user:theme"] = "dark"
session.state["user:name"] = "An"
```

Dùng cho:

- user preference,
- profile details,
- dữ liệu cần dùng lại qua nhiều session của cùng user.

Scope:

```text
app_name + user_id
```

Lưu ý:

```text
user: state shared qua các session của cùng user trong cùng app.
```

Không nên dùng `user:` cho dữ liệu chỉ thuộc một conversation cụ thể.

## app: App State

Prefix `app:` nghĩa là state gắn với toàn application.

Ví dụ:

```python
session.state["app:api_endpoint"] = "https://api.example.com"
session.state["app:global_discount_code"] = "SAVE10"
```

Dùng cho:

- global settings,
- shared templates,
- config chung của app.

Scope:

```text
app_name
```

Lưu ý:

```text
app: state shared qua mọi user và mọi session trong cùng app.
```

Không nên lưu dữ liệu riêng tư của user vào `app:`.

## temp: Temporary Invocation State

Prefix `temp:` nghĩa là dữ liệu chỉ sống trong invocation hiện tại.

Invocation là toàn bộ quá trình từ lúc agent nhận input đến lúc tạo output cuối cho input đó.

Ví dụ:

```python
session.state["temp:raw_api_response"] = response
session.state["temp:validation_needed"] = True
session.state["temp:last_operation_status"] = "success"
```

Dùng cho:

- intermediate result,
- raw API response tạm,
- flag tạm giữa các tool call,
- dữ liệu chỉ cần trong một lượt xử lý.

Không dùng cho:

- user preference,
- conversation summary,
- dữ liệu cần sang turn sau,
- accumulated data.

Điểm quan trọng:

```text
temp: không persist sau invocation.
```

Khi parent agent gọi sub-agent trong cùng invocation, chain đó share cùng `invocation_id`, nên cũng share cùng `temp:` state.

## Agent thấy state như thế nào?

Agent code làm việc với một collection duy nhất:

```python
session.state
```

Nhưng phía dưới, `SessionService` xử lý scope và storage theo prefix.

Hình dung:

```text
session.state["current_step"]
  -> session-level storage

session.state["user:preferred_language"]
  -> user-level storage

session.state["app:api_endpoint"]
  -> app-level storage

session.state["temp:last_result"]
  -> invocation-only state
```

Bạn không cần tự merge nhiều storage nếu dùng đúng API của ADK.

## Inject state vào instruction

Với `LlmAgent`, có thể inject state trực tiếp vào instruction bằng cú pháp `{key}`.

Ví dụ:

```python
from google.adk.agents import LlmAgent

story_generator = LlmAgent(
    name="StoryGenerator",
    model="gemini-flash-latest",
    instruction="Write a short story focusing on the theme: {topic}."
)
```

Nếu state có:

```python
session.state["topic"] = "friendship"
```

Thì model nhận instruction:

```text
Write a short story focusing on the theme: friendship.
```

Lợi ích:

- instruction rõ ràng,
- tránh bắt model tự suy luận từ history,
- dễ thấy chỗ nào là dynamic,
- giảm lỗi khi đổi tên state key.

## Key có thể thiếu

Nếu instruction dùng:

```text
{topic}
```

nhưng state không có key `topic`, agent có thể throw error.

Nếu key có thể không tồn tại, dùng:

```text
{topic?}
```

Dấu `?` nghĩa là optional placeholder.

## Literal curly braces

Nếu instruction cần dấu `{}` thật, ví dụ JSON:

```text
Return JSON like {"city": "<name>", "population": <number>}
```

ADK có thể hiểu nhầm `{city}` hoặc phần trong braces là state placeholder.

Cách xử lý:

```text
Dùng InstructionProvider thay vì instruction string thường.
```

Ví dụ:

```python
from google.adk.agents import LlmAgent
from google.adk.agents.readonly_context import ReadonlyContext

def my_instruction_provider(context: ReadonlyContext) -> str:
    return 'Format your output as JSON: {"city": "<name>", "population": <number>}'

agent = LlmAgent(
    model="gemini-flash-latest",
    name="template_helper_agent",
    instruction=my_instruction_provider,
)
```

Khi dùng `InstructionProvider`, ADK không tự inject state placeholder trong string trả về.

## Dùng InstructionProvider nhưng vẫn inject state

Nếu muốn vừa có full control vừa inject state, dùng utility `inject_session_state`.

Ví dụ:

```python
from google.adk.agents import LlmAgent
from google.adk.agents.readonly_context import ReadonlyContext
from google.adk.utils import instructions_utils

async def my_dynamic_instruction_provider(context: ReadonlyContext) -> str:
    template = 'This is a {adjective} instruction. Use JSON like: {"key": "value"}.'
    return await instructions_utils.inject_session_state(template, context)

agent = LlmAgent(
    model="gemini-flash-latest",
    name="dynamic_template_helper_agent",
    instruction=my_dynamic_instruction_provider,
)
```

Utility này chỉ replace placeholder hợp lệ, còn JSON braces không hợp lệ làm state key sẽ được giữ nguyên.

## Cách update state đúng

Doc nhấn mạnh: update state phải đi theo event lifecycle của ADK.

Có 3 cách chính:

```text
1. output_key
2. EventActions.state_delta
3. CallbackContext.state hoặc ToolContext.state
```

Mục tiêu là để ADK capture thay đổi vào `EventActions.state_delta`, rồi `SessionService.append_event()` apply và persist.

## Cách 1: output_key

Dùng `output_key` khi muốn lưu final text response của agent vào state.

Ví dụ:

```python
from google.adk.agents import LlmAgent

greeting_agent = LlmAgent(
    name="Greeter",
    model="gemini-flash-latest",
    instruction="Generate a short, friendly greeting.",
    output_key="last_greeting",
)
```

Khi agent trả lời cuối cùng, ADK tự lưu response vào:

```python
state["last_greeting"]
```

Phía sau, `Runner` dùng `output_key` để tạo `EventActions` có `state_delta`, rồi gọi `append_event`.

Dùng khi:

```text
Muốn lưu text output cuối của agent vào một key.
```

## Cách 2: EventActions.state_delta

Dùng `EventActions.state_delta` khi cần update phức tạp:

- nhiều key,
- non-string values,
- key có prefix `user:` hoặc `app:`,
- update không gắn trực tiếp với final text response của agent.

Ví dụ:

```python
from google.adk.events import Event, EventActions

state_changes = {
    "task_status": "active",
    "user:login_count": 1,
    "user:last_login_ts": current_time,
    "temp:validation_needed": True,
}

actions_with_update = EventActions(state_delta=state_changes)

system_event = Event(
    invocation_id="inv_login_update",
    author="system",
    actions=actions_with_update,
    timestamp=current_time,
)

await session_service.append_event(session, system_event)
```

Sau khi append:

```text
task_status được cập nhật trong session state.
user:login_count được cập nhật trong user state.
user:last_login_ts được cập nhật trong user state.
temp:validation_needed không persist qua invocation.
```

## Cách 3: CallbackContext hoặc ToolContext

Trong callback hoặc tool, cách tốt nhất là sửa `context.state`.

Ví dụ:

```python
from google.adk.agents import CallbackContext

def my_callback(context: CallbackContext):
    count = context.state.get("user_action_count", 0)
    context.state["user_action_count"] = count + 1
    context.state["temp:last_operation_status"] = "success"
```

Hoặc trong tool:

```python
from google.adk.tools import ToolContext

def my_tool(tool_context: ToolContext):
    count = tool_context.state.get("user_action_count", 0)
    tool_context.state["user_action_count"] = count + 1
    tool_context.state["temp:last_operation_status"] = "success"
```

ADK tự capture thay đổi này vào `EventActions.state_delta`.

Đây là cách thường dùng nhất khi viết tool/callback.

Trong TypeScript, doc nói phần này dùng unified `Context` type.

## append_event làm gì?

`SessionService.append_event()` không chỉ thêm event vào history.

Nó còn:

- thêm `Event` vào `session.events`,
- đọc `state_delta` từ `event.actions`,
- apply state changes theo prefix,
- xử lý persistence theo service type,
- update `last_update_time`,
- đảm bảo thread-safety cho concurrent updates.

Vì vậy update state đúng cách phải đi qua event lifecycle.

## Cảnh báo: không sửa trực tiếp session.state

Không nên làm:

```python
retrieved_session = await session_service.get_session(...)
retrieved_session.state["key"] = "value"
```

Lý do:

- không ghi vào event history,
- mất auditability,
- có thể không persist với `DatabaseSessionService`,
- có thể không persist với `VertexAiSessionService`,
- không thread-safe,
- không update `last_update_time`,
- bypass logic của ADK.

Chỉ đọc trực tiếp `session.state` thì được.

Muốn ghi state, dùng:

```text
output_key
EventActions.state_delta
CallbackContext.state
ToolContext.state
```

## State không phải Artifact

State không nên giữ file/blob lớn.

Không nên:

```python
state["pdf_bytes"] = pdf_bytes
state["full_contract_text"] = very_long_text
state["image_bytes"] = image_bytes
```

Nên:

```text
File/blob -> ArtifactService.
State -> filename, version, id, summary ngắn nếu cần.
```

Ví dụ:

```python
state["current_contract_file"] = "contract.pdf"
state["current_contract_version"] = 2
```

## State không phải Memory

State là dữ liệu vận hành hiện tại.

Memory là tri thức dài hạn hoặc searchable knowledge.

Ví dụ:

```text
checkout_step = "payment"
  -> state

user thích report dạng bullet qua nhiều session
  -> memory hoặc user: state, tùy thiết kế
```

Nếu dữ liệu cần search semantic hoặc sống dài hạn qua nhiều session, cân nhắc memory.

## Best practices

### Minimalism

Chỉ lưu dữ liệu thật sự cần.

State càng gọn thì dễ debug và ít rủi ro serialization.

### Serializable values

Chỉ lưu basic types và structure đơn giản.

Không lưu object, function, connection, file handle.

### Key rõ nghĩa

Nên:

```text
current_booking_step
selected_flight_id
user:preferred_language
temp:last_api_response
```

Không nên:

```text
x
data
result
tmp
```

### Prefix đúng scope

Chọn prefix theo phạm vi dữ liệu:

```text
Thread hiện tại -> không prefix
User qua nhiều session -> user:
Toàn app -> app:
Một invocation -> temp:
```

### Tránh deep nesting

Nên giữ structure nông, dễ serialize và dễ update.

Không nên tạo state nested quá sâu.

Ví dụ tốt:

```python
state["selected_flight_id"] = "VJ456"
state["booking_step"] = "payment"
```

Hơn là một object workflow quá sâu và khó update.

### Luôn update qua standard flow

Update state qua:

- `output_key`,
- `EventActions.state_delta`,
- `context.state` trong callback/tool.

Không sửa trực tiếp `session.state` lấy từ `SessionService`.

## Ví dụ thực tế

User đặt vé:

```text
User: Tôi muốn đi Đà Nẵng.
Agent: Bạn muốn đi ngày nào?
User: Ngày mai.
Tool: tìm chuyến bay.
Agent: Có 3 chuyến.
User: Chọn chuyến thứ hai.
```

State hợp lý:

```python
state["destination"] = "Đà Nẵng"
state["travel_date"] = "2026-06-08"
state["available_flight_ids"] = ["VN123", "VJ456", "QH789"]
state["selected_flight_id"] = "VJ456"
state["booking_step"] = "confirm_payment"
```

Nếu user thích tiếng Việt:

```python
state["user:preferred_language"] = "vi"
```

Nếu tool có raw API response chỉ cần trong lượt hiện tại:

```python
state["temp:raw_flight_api_response"] = response
```

## Ghi nhớ

Trang này trả lời câu hỏi:

> Agent lưu dữ liệu nhỏ trong một session ở đâu và cập nhật thế nào cho đúng?

Câu trả lời:

> Dùng `session.state`, nhưng update qua event lifecycle của ADK.

Quy tắc nhanh:

```text
State = key-value nhỏ, serializable.
Không prefix = session.
user: = user.
app: = app.
temp: = invocation hiện tại.

Đọc session.state trực tiếp thì được.
Ghi state phải qua output_key, EventActions.state_delta, hoặc context.state.
Không sửa trực tiếp session.state lấy từ SessionService.
```

## Source

- [State: The Session's Scratchpad](https://adk.dev/sessions/state/)
