---
title: Context trong ADK
tags:
  - adk
  - components
  - context
  - state
  - memory
created: 2026-06-07
up:
  - "[[Components]]"
source:
  - https://adk.dev/context/
---

# Context trong ADK

Liên kết: [[Components]]

## Context là gì?

`Context` trong ADK là gói thông tin mà framework đưa cho agent, callback và tool trong lúc xử lý một lượt yêu cầu.

Nói ngắn:

```text
Context = thông tin nền + tài nguyên runtime + state của lượt xử lý hiện tại.
```

Agent thường không chỉ cần message mới nhất của user. Nó còn cần biết:

- session hiện tại là gì,
- state đang có những key nào,
- user vừa gửi input gì,
- agent nào đang chạy,
- invocation hiện tại có id gì,
- có artifact service, memory service, session service không,
- tool có cần auth không,
- file/artifact nào có thể load,
- có nên dừng toàn bộ lượt xử lý không.

ADK gom các thông tin này vào context để agent và tool làm việc đúng với trạng thái hiện tại.

## Invocation là gì?

`Invocation` là một vòng xử lý từ lúc user gửi input đến lúc agent trả kết quả cuối.

Ví dụ:

```text
User hỏi
  -> Runner nhận message
  -> ADK tạo InvocationContext
  -> agent chạy
  -> model callback/tool callback/tool có thể chạy
  -> events được sinh ra
  -> response cuối trả về user
```

Mỗi invocation có `invocation_id` riêng để logging, tracing và debug.

## InvocationContext

`InvocationContext` là context đầy đủ nhất.

Nó chứa:

- `session`,
- `session.state`,
- `session.events`,
- agent hiện tại,
- `invocation_id`,
- `user_content` ban đầu,
- `artifact_service`,
- `memory_service`,
- `session_service`,
- thông tin live/streaming nếu có,
- cờ điều khiển như `end_invocation`.

Bạn thường không tự tạo `InvocationContext`.

Framework tạo nó khi gọi runner, ví dụ:

```python
async for event in runner.run_async(
    user_id="user123",
    session_id="session456",
    new_message=user_message,
):
    print(event.stringify_content())
```

Trong nội bộ, ADK tạo context cho run này rồi truyền vào agent, callback và tool theo dạng phù hợp.

## Các loại Context

ADK không bắt bạn dùng full `InvocationContext` ở mọi nơi. Thay vào đó có nhiều loại context hẹp hơn.

| Context | Dùng ở đâu | Dùng để làm gì |
|---|---|---|
| `InvocationContext` | core implementation của custom agent như `_run_async_impl`, `_run_live_impl` | Truy cập toàn bộ invocation, session, services, điều khiển flow |
| `ReadonlyContext` | nơi chỉ cần đọc, ví dụ instruction provider | Đọc `state`, `agent_name`, `invocation_id`, không ghi state |
| `CallbackContext` | agent/model callbacks như `before_agent_callback`, `before_model_callback` | Đọc/ghi state, load/save artifact, đọc user input |
| `ToolContext` | function tool và tool callbacks | Đọc/ghi state, auth, memory search, list artifact, biết `function_call_id` |

Trong TypeScript, doc nói `CallbackContext` và `ToolContext` được gom vào một kiểu `Context`.

## ReadonlyContext

`ReadonlyContext` dùng khi code chỉ cần đọc thông tin.

Ví dụ instruction provider đọc tier của user:

```python
from google.adk.agents.readonly_context import ReadonlyContext

def my_instruction_provider(context: ReadonlyContext) -> str:
    user_tier = context.state.get("user_tier", "standard")
    return f"Process the request for a {user_tier} user."
```

Điểm quan trọng:

```text
ReadonlyContext không cho sửa state.
```

Dùng nó khi không muốn function phụ thay đổi trạng thái session.

## CallbackContext

`CallbackContext` dùng trong callback của agent hoặc model.

Ví dụ:

- `before_agent_callback`,
- `after_agent_callback`,
- `before_model_callback`,
- `after_model_callback`.

Callback có thể đọc và ghi state:

```python
from google.adk.agents.context import Context
from google.adk.models import LlmRequest
from google.genai import types
from typing import Optional

def my_before_model_cb(context: Context, request: LlmRequest) -> Optional[types.Content]:
    call_count = context.state.get("model_calls", 0)
    context.state["model_calls"] = call_count + 1
    return None
```

Khi ghi:

```python
context.state["model_calls"] = call_count + 1
```

ADK tracking thay đổi đó và gắn nó với event tương ứng.

Callback cũng có thể:

- đọc `user_content`,
- load artifact,
- save artifact,
- đọc `invocation_id`,
- đọc `agent_name`.

## ToolContext

`ToolContext` dùng trong function tool và tool callbacks.

Nó có mọi thứ quan trọng của callback context, cộng thêm các năng lực riêng cho tool:

- `request_credential(...)`,
- `get_auth_response(...)`,
- `search_memory(query)`,
- `list_artifacts()`,
- `function_call_id`,
- `actions`.

Ví dụ tool đọc API key từ state:

```python
from google.adk.tools import ToolContext
from typing import Any

def search_external_api(query: str, tool_context: ToolContext) -> dict[str, Any]:
    api_key = tool_context.state.get("api_key")

    if not api_key:
        return {"status": "Auth Required"}

    return {"result": f"Data for {query} fetched."}
```

`function_call_id` quan trọng vì một model response có thể có nhiều tool call. ADK cần biết credential hoặc action thuộc tool call nào.

## State trong Context

`context.state` là cơ chế chính để chia sẻ dữ liệu nhỏ giữa các bước.

Ví dụ:

```python
tool_context.state["order_id"] = "ORD-123"
tool_context.state["temp:last_api_result"] = result
```

Dùng state cho:

- user preference,
- workflow step,
- id của entity đang xử lý,
- kết quả nhỏ từ tool trước,
- flag điều khiển flow,
- config nhỏ.

Không dùng state cho:

- file lớn,
- raw PDF bytes,
- nội dung document dài,
- ảnh/audio/video,
- data có thể phình rất lớn.

## Prefix state

Một số prefix thường dùng:

| Prefix | Ý nghĩa | Ví dụ |
|---|---|---|
| không prefix | state trong session hiện tại | `order_id` |
| `temp:` | dữ liệu tạm cho invocation hiện tại | `temp:last_api_result` |
| `user:` | dữ liệu gắn với user | `user:timezone` |
| `app:` | dữ liệu cấp app | `app:api_endpoint` |

Cách chọn:

```text
Dữ liệu chỉ cần trong lượt xử lý hiện tại -> temp:
Dữ liệu thuộc user và cần dùng lại -> user:
Dữ liệu cấu hình app -> app:
Dữ liệu của session hiện tại -> không prefix
```

## Artifacts trong Context

Context có thể làm việc với artifacts nếu runner được cấu hình `artifact_service`.

Dùng artifact cho file hoặc blob:

- PDF,
- image,
- audio,
- CSV,
- generated report,
- user upload.

Ví dụ:

```python
import google.genai.types as types

pdf_part = types.Part.from_bytes(
    data=pdf_bytes,
    mime_type="application/pdf",
)

version = await context.save_artifact(
    filename="contract.pdf",
    artifact=pdf_part,
)
```

Khi cần đọc:

```python
contract = await context.load_artifact("contract.pdf")
```

Cách nhớ:

```text
State giữ dữ liệu nhỏ.
Artifact giữ file/blob.
```

## Memory trong Context

`ToolContext` có thể gọi `search_memory(query)` nếu app có `memory_service`.

Dùng memory khi cần tìm lại tri thức hoặc thông tin từ quá khứ.

Ví dụ:

```python
relevant = tool_context.search_memory("user preferences for report format")
```

Memory khác state:

| Cơ chế | Vai trò |
|---|---|
| `state` | dữ liệu vận hành hiện tại, nhỏ, rõ key |
| `memory` | tri thức dài hạn hoặc thông tin cần retrieve theo ngữ nghĩa |
| `artifact` | file/blob |

## Auth trong ToolContext

Một số tool cần credential để gọi API ngoài.

Luồng thường gặp:

```text
Tool cần credential
  -> kiểm tra state hoặc auth response
  -> nếu chưa có thì request credential
  -> ADK tạo auth request action
  -> sau khi user/system cấp credential, tool chạy tiếp
```

Trong tool, auth thường liên quan đến:

- `request_credential(auth_config)`,
- `get_auth_response(auth_config)`,
- `tool_context.actions`,
- `tool_context.function_call_id`.

Điểm cần nhớ:

```text
Auth request phải gắn đúng function_call_id để ADK biết credential đó trả lời cho tool call nào.
```

## Đọc thông tin hiện tại

Các context thường cho đọc:

- `agent_name`,
- `invocation_id`,
- `state`,
- `user_content`,
- với tool thì có thêm `function_call_id`.

Ví dụ logging:

```python
from google.adk.tools import ToolContext

def log_tool_usage(tool_context: ToolContext, **kwargs):
    agent_name = tool_context.agent_name
    inv_id = tool_context.invocation_id
    func_call_id = getattr(tool_context, "function_call_id", "N/A")

    print(
        f"Invocation={inv_id}, Agent={agent_name}, "
        f"FunctionCallID={func_call_id}"
    )
```

## Dừng invocation

Trong một số trường hợp, agent cần dừng toàn bộ request-response cycle.

Ví dụ:

- phát hiện lỗi nghiêm trọng,
- user không có quyền,
- policy chặn,
- workflow đã đủ điều kiện kết thúc.

Khi viết custom agent core, có thể set:

```python
ctx.end_invocation = True
```

Ý nghĩa:

```text
ADK dừng xử lý invocation hiện tại một cách có kiểm soát.
```

Chỉ dùng khi thật sự cần dừng toàn bộ flow, không phải chỉ bỏ qua một tool nhỏ.

## Chọn context nào?

Quy tắc nhanh:

```text
Đang viết custom agent core
  -> InvocationContext

Đang viết instruction provider chỉ cần đọc
  -> ReadonlyContext

Đang viết agent/model callback
  -> CallbackContext

Đang viết function tool hoặc tool callback
  -> ToolContext
```

Không nên dùng context rộng hơn nhu cầu.

Ví dụ:

- Tool cần auth hoặc memory search thì dùng `ToolContext`.
- Callback chỉ đếm số lần gọi model thì dùng `CallbackContext`.
- Instruction chỉ đọc user tier thì dùng `ReadonlyContext`.
- Agent core cần kiểm tra service và dừng invocation thì dùng `InvocationContext`.

## Ví dụ flow thực tế

User hỏi:

```text
Tóm tắt contract.pdf và ghi nhớ tôi thích report dạng bullet.
```

Flow có thể là:

```text
Runner tạo InvocationContext
  -> before_model_callback đọc context.state
  -> agent quyết định cần đọc file
  -> tool gọi list_artifacts/load_artifact
  -> tool đọc contract.pdf
  -> agent tạo summary
  -> callback/tool ghi user preference vào state hoặc memory
  -> SessionService lưu state delta và events
```

Nếu file là PDF:

```text
contract.pdf nằm trong ArtifactService.
State chỉ nên giữ filename/version/summary ngắn.
```

Nếu preference cần dùng lâu dài:

```text
user:report_format = "bullet"
```

hoặc ghi vào memory nếu app thiết kế memory dài hạn.

## Sai lầm thường gặp

### Nhét file lớn vào state

Không nên:

```python
context.state["contract_pdf"] = pdf_bytes
```

Nên:

```python
await context.save_artifact("contract.pdf", pdf_part)
context.state["current_contract"] = "contract.pdf"
```

### Dùng InvocationContext ở mọi nơi

Không nên kéo full invocation vào function chỉ cần đọc state.

Nên dùng context hẹp:

```text
ReadonlyContext cho read-only.
CallbackContext cho callback.
ToolContext cho tool.
```

### Không prefix state rõ ràng

Không nên:

```python
context.state["result"] = result
```

Nên:

```python
context.state["temp:last_api_result"] = result
context.state["user:preferred_language"] = "vi"
context.state["app:api_endpoint"] = "https://api.example.com"
```

### Quên cấu hình service

Nếu muốn dùng artifact:

```text
Runner cần artifact_service.
```

Nếu muốn dùng memory:

```text
Runner/app cần memory_service.
```

Nếu service không có, các thao tác liên quan có thể lỗi hoặc không khả dụng.

## Best practices

- Dùng context hẹp nhất đủ việc.
- Dùng `context.state` cho dữ liệu nhỏ và điều khiển flow.
- Dùng prefix `temp:`, `user:`, `app:` có chủ đích.
- Dùng artifact cho file/blob, không nhét file vào state.
- Dùng memory cho tri thức dài hạn hoặc retrieve theo ngữ nghĩa.
- Ghi log bằng `invocation_id`, `agent_name`, `function_call_id` khi debug tool.
- Chỉ dùng `ctx.end_invocation = True` khi cần dừng toàn bộ lượt xử lý.
- Bắt đầu đơn giản với state và artifact, sau đó mới thêm auth/memory khi cần.

## Ghi nhớ

Trang này trả lời câu hỏi:

> Agent và tool lấy trạng thái, file, memory, auth và thông tin runtime từ đâu?

Câu trả lời:

> Từ context mà ADK truyền vào từng vị trí phù hợp.

Mô hình đúng:

```text
InvocationContext
  -> context đầy đủ cho một lượt xử lý

ReadonlyContext
  -> chỉ đọc

CallbackContext
  -> callback đọc/ghi state và artifact

ToolContext
  -> tool đọc/ghi state, artifact, auth, memory
```

Cách chọn lưu trữ:

```text
Dữ liệu nhỏ, điều khiển flow -> state
File/blob -> artifact
Tri thức dài hạn -> memory
Credential/API auth -> auth flow trong ToolContext
```

## Source

- [Context](https://adk.dev/context/)
