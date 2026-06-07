---
title: Session Object trong ADK
tags:
  - adk
  - components
  - sessions
  - session-service
  - state
created: 2026-06-07
up:
  - "[[Sessions trong ADK]]"
source:
  - https://adk.dev/sessions/session/
---

# Session Object trong ADK

Liên kết: [[Sessions trong ADK]]

## Session object là gì?

`Session` trong ADK là object dùng để tracking một conversation thread cụ thể giữa user và agent.

Nói ngắn:

```text
Session = container của một cuộc hội thoại riêng.
```

Khi user bắt đầu tương tác với agent, `SessionService` tạo một `Session`.

Session giữ các thông tin quan trọng của thread đó:

- session này là của app nào,
- user nào đang nói chuyện,
- lịch sử event đã xảy ra,
- state tạm của cuộc hội thoại,
- lần cuối session được cập nhật.

## Vì sao cần Session object?

Agent không nên xử lý mỗi message như một request hoàn toàn độc lập.

Ví dụ:

```text
User: Tôi muốn đặt vé đi Đà Nẵng.
Agent: Có 3 chuyến bay phù hợp.
User: Chọn chuyến thứ hai.
```

Câu "Chọn chuyến thứ hai" chỉ có nghĩa nếu agent còn nhớ:

```text
Trước đó user muốn đi Đà Nẵng.
Tool đã trả về 3 chuyến bay.
```

Session là nơi giữ mạch này.

## Các field chính của Session

Một `Session` có các property quan trọng:

| Field | Ý nghĩa |
|---|---|
| `id` | ID duy nhất của conversation thread |
| `app_name` / `appName` | app/agent application mà session thuộc về |
| `user_id` / `userId` | user sở hữu session |
| `events` | lịch sử tương tác trong session |
| `state` | dữ liệu tạm của conversation hiện tại |
| `last_update_time` / `lastUpdateTime` | thời điểm session có event mới gần nhất |

Hình dung:

```text
Session
  id: "session_abc"
  app_name: "travel_agent"
  user_id: "user_123"
  events:
    - user message
    - agent response
    - tool call
    - tool result
  state:
    destination: "Đà Nẵng"
    selected_flight_id: "VJ456"
  last_update_time: ...
```

## Identification

Session được định danh bằng các thông tin:

```text
id
app_name / appName
user_id / userId
```

`id` là ID của thread cụ thể.

`app_name` cho biết session thuộc agent app nào.

`user_id` cho biết session thuộc user nào.

Vì một service có thể quản lý nhiều session, bộ thông tin này giúp tìm đúng conversation thread.

Ví dụ:

```text
app_name = "customer_support_agent"
user_id = "u_123"
session_id = "ticket_session_456"
```

## Events

`events` là lịch sử theo thứ tự thời gian của session.

Event có thể là:

- user message,
- agent response,
- tool action,
- tool result,
- state update được gắn với event.

Ví dụ:

```text
events:
  1. User hỏi về đơn hàng
  2. Agent gọi lookup_order
  3. Tool trả về order status
  4. Agent trả lời user
```

Events là phần giúp agent tiếp tục hội thoại mà không mất mạch.

## State

`state` là scratchpad của session.

Nó chứa dữ liệu tạm liên quan đến conversation hiện tại.

Ví dụ:

```python
state = {
    "current_order_id": "ORD-123",
    "step": "waiting_for_confirmation",
    "selected_option": "refund",
}
```

Dùng state cho:

- workflow step,
- selected item,
- order id,
- ticket id,
- flag tạm,
- kết quả nhỏ từ tool.

Không dùng state cho:

- file lớn,
- raw PDF text dài,
- image bytes,
- full transcript rất lớn,
- dữ liệu cần audit nguyên văn.

Điểm quan trọng trong doc:

```text
State ban đầu có thể được truyền khi tạo session.
Các update state sau đó thường xảy ra thông qua events.
```

## Last update time

`last_update_time` cho biết lần cuối session có event mới.

Dùng được cho:

- sort session mới nhất,
- cleanup session cũ,
- resume session gần đây,
- hiển thị danh sách conversation threads.

## Tạo session đơn giản

Ví dụ Python dùng `InMemorySessionService`:

```python
from google.adk.sessions import InMemorySessionService

temp_service = InMemorySessionService()

example_session = await temp_service.create_session(
    app_name="my_app",
    user_id="example_user",
    state={"initial_key": "initial_value"},
)

print(example_session.id)
print(example_session.app_name)
print(example_session.user_id)
print(example_session.state)
print(example_session.events)
print(example_session.last_update_time)
```

Ban đầu:

```text
state có initial state.
events thường rỗng.
```

Khi agent chạy và có interaction mới, events mới được append vào session.

## SessionService là gì?

Bạn thường không tự quản lý `Session` object trực tiếp.

Thay vào đó dùng `SessionService`.

Nói ngắn:

```text
Session = dữ liệu của conversation thread.
SessionService = service quản lý lifecycle của Session.
```

`SessionService` chịu trách nhiệm:

- tạo session mới,
- resume session cũ,
- append events,
- update state thông qua events,
- list sessions,
- delete session.

## Tạo conversation mới

Khi user bắt đầu một chat mới:

```text
App gọi SessionService.create_session(...)
  -> tạo Session mới
  -> gắn app_name, user_id, session_id
  -> có thể set initial state
```

Dùng khi:

- user mở thread mới,
- workflow mới bắt đầu,
- ticket mới được tạo,
- agent cần conversation context riêng.

## Resume conversation cũ

Khi user quay lại thread cũ:

```text
App dùng session_id cũ
  -> SessionService lấy Session tương ứng
  -> Runner đưa session đó vào context
  -> agent có lại events/state cũ
```

Đây là cách agent tiếp tục nơi nó đã dừng.

Nếu dùng in-memory service và app đã restart:

```text
Session cũ sẽ mất.
```

Nếu cần resume sau restart, phải dùng persistent session service.

## Save progress

Khi agent xử lý xong một lượt:

```text
Runner tạo Event mới
SessionService append event vào session
SessionService update state dựa trên event
SessionService update last_update_time
```

Điểm quan trọng:

```text
append_event không chỉ thêm history.
Nó còn là cơ chế để state được cập nhật trong storage.
```

## List conversations

`SessionService` có thể list sessions cho một user/app.

Dùng cho UI kiểu:

```text
Danh sách cuộc trò chuyện gần đây
Ticket sessions
Workflow sessions
```

Thông tin như `last_update_time` giúp sort thread mới nhất.

## Delete session

Khi conversation kết thúc hoặc không cần nữa:

```text
sessionService.delete_session(...)
```

Dùng để cleanup:

- session demo,
- session tạm,
- conversation đã hết hạn,
- dữ liệu không còn cần giữ.

Trong production, delete session cần cân nhắc audit, compliance và retention policy.

## SessionService implementations

ADK có nhiều implementation của `SessionService`.

| Service | Persistence | Dùng khi nào |
|---|---|---|
| `InMemorySessionService` | Không | dev, test, demo |
| `VertexAiSessionService` | Có | production trên Google Cloud / Agent Runtime |
| `DatabaseSessionService` | Có | production hoặc app tự quản DB |

## InMemorySessionService

`InMemorySessionService` lưu toàn bộ session data trong memory của app process.

Ví dụ:

```python
from google.adk.sessions import InMemorySessionService

session_service = InMemorySessionService()
```

Ưu điểm:

- dễ dùng,
- không cần setup,
- nhanh,
- hợp với local dev và test.

Nhược điểm:

```text
App restart là mất toàn bộ session data.
```

Dùng cho:

- local testing,
- tutorial,
- prototype,
- demo ngắn.

Không dùng cho production nếu user cần quay lại session cũ.

## VertexAiSessionService

`VertexAiSessionService` dùng Google Cloud Agent Platform infrastructure để quản lý session.

Đặc điểm:

- persistent,
- scalable,
- tích hợp với Agent Runtime,
- hợp với app production trên Google Cloud.

Cần:

- Google Cloud project,
- Google Cloud setup/authentication,
- Agent Runtime resource,
- cấu hình liên quan đến storage/bucket theo doc.

Ví dụ Python:

```python
from google.adk.sessions import VertexAiSessionService

PROJECT_ID = "your-gcp-project-id"
LOCATION = "us-central1"

session_service = VertexAiSessionService(
    project=PROJECT_ID,
    location=LOCATION,
)
```

Dùng khi app đã chạy trên Google Cloud hoặc muốn dùng managed infrastructure.

## DatabaseSessionService

`DatabaseSessionService` lưu session vào relational database.

Hỗ trợ các backend như:

- SQLite,
- PostgreSQL,
- MySQL.

Ví dụ SQLite:

```python
from google.adk.sessions import DatabaseSessionService

db_url = "sqlite+aiosqlite:///./my_agent_data.db"
session_service = DatabaseSessionService(db_url=db_url)
```

Điểm quan trọng:

```text
DatabaseSessionService cần async database driver.
```

Với SQLite, dùng:

```text
sqlite+aiosqlite
```

Không dùng:

```text
sqlite
```

Với DB khác:

```text
PostgreSQL -> asyncpg
MySQL -> aiomysql
```

Doc cũng lưu ý schema database session thay đổi ở ADK Python v1.22.0, nên nếu nâng version cần xem hướng dẫn migration.

## Session lifecycle

Một lượt hội thoại thường đi như sau:

```text
1. Start hoặc resume
2. Runner lấy Session từ SessionService
3. Agent xử lý user query
4. Agent tạo response hoặc state update
5. Runner đóng gói thành Event
6. SessionService append_event
7. Session được lưu lại cho lượt sau
8. Khi không cần nữa thì delete session
```

Chi tiết hơn:

```text
App có user_id + session_id
  -> SessionService load Session
  -> Runner đưa session vào context
  -> Agent đọc state/events
  -> Agent xử lý query
  -> Runner tạo Event
  -> SessionService append Event
  -> state và last_update_time được cập nhật
  -> response trả về user
```

Turn sau lặp lại cycle này.

## Quan hệ với Runner

`Runner` là phần chạy agent.

Nó cần session để agent có context.

Flow:

```text
runner.run_async(user_id, session_id, new_message)
  -> lấy Session tương ứng
  -> tạo context cho invocation
  -> agent xử lý
  -> yield Events
  -> SessionService lưu Events/state
```

Bạn không cần tự nhét history vào prompt nếu dùng đúng session flow.

## Quan hệ với Context

Khi agent/tool/callback chạy, chúng nhận context.

Context cho truy cập session/state/events tùy loại context.

Hình dung:

```text
SessionService
  -> load Session

Runner
  -> tạo InvocationContext

Context
  -> cho agent/tool/callback đọc state, events, metadata
```

Session là dữ liệu.

Context là cửa truy cập dữ liệu trong lúc run.

## Ví dụ thực tế

User tạo support ticket:

```text
User: Đơn hàng ORD-123 của tôi chưa giao.
```

Session được tạo:

```text
app_name = "support_agent"
user_id = "u_123"
session_id = "s_456"
```

State:

```python
state["current_order_id"] = "ORD-123"
state["issue_type"] = "delivery_delay"
state["step"] = "checking_order_status"
```

Events:

```text
1. User report issue
2. Agent calls lookup_order
3. Tool returns status
4. Agent responds
```

User quay lại sau:

```text
Cập nhật giúp tôi ticket lúc nãy.
```

Nếu app dùng cùng `session_id`, agent biết thread này đang nói về `ORD-123`.

## Khi nào chọn service nào?

| Tình huống | Service hợp lý |
|---|---|
| Học ADK, chạy ví dụ, test nhanh | `InMemorySessionService` |
| App production trên Google Cloud / Agent Runtime | `VertexAiSessionService` |
| App production tự quản DB | `DatabaseSessionService` |
| Cần user quay lại session sau restart | Persistent service |
| Không cần giữ sau restart | In-memory đủ |

## Best practices

### Đừng tự quản lý Session nếu không cần

Nên dùng `SessionService`.

ADK thiết kế để service xử lý:

- create,
- retrieve,
- append events,
- update state,
- delete.

### Đặt session id rõ ràng

Nếu app cần resume conversation, phải lưu lại `session_id` phía app/user.

Ví dụ:

```text
user_id + session_id -> resume đúng thread.
```

### Production cần persistence

Nếu cần giữ conversation:

```text
Không dùng InMemorySessionService.
```

Dùng:

- `DatabaseSessionService`,
- `VertexAiSessionService`,
- hoặc backend persistent phù hợp.

### State chỉ giữ dữ liệu nhỏ

Nên:

```python
state["order_id"] = "ORD-123"
state["step"] = "waiting_for_user_confirmation"
```

Không nên:

```python
state["full_pdf_text"] = "..."
state["image_bytes"] = b"..."
```

### Cleanup có chủ đích

Session có thể chứa dữ liệu nhạy cảm.

Cần có policy cho:

- session hết hạn,
- xóa session,
- retention,
- audit,
- user data deletion request.

## Ghi nhớ

Trang này trả lời câu hỏi:

> Một conversation thread cụ thể trong ADK được lưu và quản lý như thế nào?

Câu trả lời:

```text
Session lưu conversation thread.
SessionService quản lý lifecycle của Session.
Events lưu lịch sử.
State lưu dữ liệu tạm.
last_update_time theo dõi hoạt động cuối.
```

Quy tắc nhanh:

```text
Dev/test -> InMemorySessionService.
Production cần persistence -> DatabaseSessionService hoặc VertexAiSessionService.
Resume conversation -> cần session_id cũ.
Update state -> thường đi qua events được append vào session.
```

## Source

- [Session: Tracking Individual Conversations](https://adk.dev/sessions/session/)
