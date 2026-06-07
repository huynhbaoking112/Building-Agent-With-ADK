---
title: Sessions trong ADK
tags:
  - adk
  - components
  - sessions
  - state
  - memory
created: 2026-06-07
up:
  - "[[Components]]"
source:
  - https://adk.dev/sessions/
---

# Sessions trong ADK

Liên kết: [[Components]]

## Sessions là gì?

`Session` trong ADK là một cuộc hội thoại hoặc một conversation thread hiện tại giữa user và agent system.

Nói ngắn:

```text
Session = một cuộc trò chuyện đang diễn ra.
```

Một agent muốn nói chuyện nhiều lượt thì không thể chỉ nhìn message mới nhất. Nó cần biết:

- user đã nói gì trước đó,
- agent đã trả lời gì,
- tool nào đã được gọi,
- tool trả về kết quả gì,
- workflow đang ở bước nào,
- dữ liệu tạm nào đang liên quan đến cuộc trò chuyện này.

ADK gom những thứ đó vào `Session`, `State`, `Events` và `Memory`.

## Vì sao cần Session?

Nếu không có session, mỗi lần user gửi message mới, agent sẽ xử lý như một request rời rạc.

Ví dụ user nói:

```text
Tôi muốn đặt vé đi Hà Nội.
```

Sau đó user nói tiếp:

```text
Chọn chuyến thứ hai.
```

Nếu agent không có session, nó không biết "chuyến thứ hai" là chuyến nào.

Với session, agent có thể nhìn lại history:

```text
User muốn đi Hà Nội.
Tool đã trả về 3 chuyến bay.
User đang chọn chuyến thứ hai.
```

Nhờ vậy agent giữ được mạch hội thoại.

## Ba khái niệm lõi

Doc này giới thiệu 3 khái niệm chính:

| Khái niệm | Vai trò |
|---|---|
| `Session` | Cuộc hội thoại hiện tại |
| `State` | Dữ liệu nhỏ nằm trong session hiện tại |
| `Memory` | Kho thông tin dài hạn, có thể search qua nhiều session hoặc nguồn ngoài |

Ngoài ra, session còn có `Events`.

```text
Session
  -> Events
  -> State

Memory
  -> thông tin dài hạn/searchable, không chỉ thuộc một session
```

## Session

`Session` đại diện cho một interaction đang diễn ra giữa user và agent system.

Nó giống một thread chat riêng.

Ví dụ:

```text
Session A:
  User: Tôi muốn mua laptop.
  Agent: Bạn cần laptop cho công việc gì?
  User: Lập trình và chạy local model.
  Tool: search product catalog
  Agent: Có 3 lựa chọn phù hợp.
```

Một user có thể có nhiều session khác nhau.

Ví dụ:

```text
Session 1: đặt vé máy bay
Session 2: phân tích file PDF
Session 3: support ticket
```

Mỗi session có history và state riêng.

## Events

`Events` là chuỗi các sự kiện xảy ra trong session.

Ví dụ event có thể là:

- user message,
- agent response,
- model call,
- tool call,
- tool result,
- state update,
- action trong workflow.

Hình dung:

```text
Session
  events:
    1. User hỏi
    2. Agent trả lời
    3. Agent gọi tool
    4. Tool trả kết quả
    5. Agent tổng hợp
```

Events giúp ADK biết lịch sử đã diễn ra theo thứ tự nào.

Đây là phần quan trọng để agent giữ continuity trong hội thoại.

## State

`State` là dữ liệu key-value nằm trong một session.

Nó dùng để giữ dữ liệu nhỏ, cụ thể, phục vụ workflow hiện tại.

Ví dụ:

```python
session.state["destination"] = "Hà Nội"
session.state["selected_flight_id"] = "VN123"
session.state["checkout_step"] = "payment"
```

Dùng state cho:

- workflow step,
- selected item,
- order id,
- ticket id,
- user preference vừa nói trong session,
- flag đã xác thực,
- kết quả nhỏ từ tool.

Không dùng state cho:

- file PDF lớn,
- ảnh,
- audio,
- video,
- raw document dài,
- toàn bộ tool response rất lớn.

State nên là dữ liệu nhỏ và có cấu trúc rõ.

## Memory

`Memory` là kho thông tin dài hạn hoặc thông tin từ ngoài session hiện tại.

Memory có thể chứa:

- thông tin từ các session cũ,
- thông tin đã được ingest sau một cuộc trò chuyện,
- knowledge base bên ngoài,
- facts về user hoặc domain,
- dữ liệu cần search lại theo query.

Ví dụ:

```text
User thích report dạng bullet.
User thường muốn câu trả lời bằng tiếng Việt.
Policy hoàn tiền của công ty.
Thông tin sản phẩm trong knowledge base.
```

Memory khác state ở chỗ:

```text
State -> dữ liệu trong session hiện tại.
Memory -> thông tin có thể vượt qua nhiều session.
```

Memory hoạt động giống một kho tri thức có thể search.

## Session và Memory khác nhau thế nào?

| Cơ chế | Phạm vi | Dùng cho |
|---|---|---|
| `Session` | Một cuộc hội thoại hiện tại | History, events, state của thread đó |
| `State` | Bên trong session | Dữ liệu nhỏ để điều khiển flow |
| `Memory` | Qua nhiều session hoặc nguồn ngoài | Tri thức dài hạn/searchable |

Ví dụ:

```text
User đang đặt vé trong chat hiện tại
  -> lưu destination, selected_flight_id vào state.

User luôn thích report dạng bullet
  -> lưu hoặc ingest vào memory.
```

## Services quản lý context

ADK dùng service để quản lý session và memory.

Doc nhắc hai service chính:

| Service | Quản lý |
|---|---|
| `SessionService` | Session, Events, State |
| `MemoryService` | Memory, ingest, search |

## SessionService

`SessionService` quản lý lifecycle của session.

Nó chịu trách nhiệm:

- tạo session,
- lấy session,
- cập nhật session,
- append events,
- sửa state,
- xóa session.

Nói cách khác:

```text
SessionService = storage và lifecycle manager cho conversation thread.
```

Khi agent chạy, ADK cần biết session nào đang active.

Ví dụ logic:

```text
Runner nhận user_id + session_id
  -> SessionService load session
  -> agent xử lý message mới
  -> events mới được append
  -> state delta được lưu
```

## MemoryService

`MemoryService` quản lý long-term knowledge store.

Nó chịu trách nhiệm:

- ingest thông tin vào memory,
- lưu thông tin từ completed sessions nếu app thiết kế như vậy,
- search memory theo query.

Nói cách khác:

```text
MemoryService = kho tìm kiếm thông tin dài hạn.
```

Agent hoặc tool có thể search memory để tìm lại thông tin liên quan.

Ví dụ:

```text
User hỏi: Lần trước tôi chọn format report nào?
  -> agent search memory
  -> memory trả về: user thích bullet report
```

## In-memory service

ADK có in-memory implementation cho `SessionService` và `MemoryService`.

Dùng tốt cho:

- local development,
- demo,
- unit test,
- prototype nhanh.

Nhược điểm:

```text
App restart là mất dữ liệu.
```

Vì dữ liệu nằm trong memory của process, nó không phù hợp cho production nếu cần persistence.

## Persistent backend

Production nên dùng backend persistent hoặc cloud/database service.

Lý do:

- session không mất khi restart,
- có thể scale nhiều instance,
- lưu được state/events ổn định,
- user quay lại vẫn tiếp tục session,
- memory dài hạn không bị mất.

Cách chọn:

```text
Dev/test -> in-memory service.
Production -> persistent service/database/cloud backend.
```

## Quan hệ với Context

`Context` là object mà agent, callback hoặc tool nhận trong lúc chạy.

`Session`, `State`, `Events`, `Memory` là dữ liệu và service mà context có thể cho truy cập.

Hình dung:

```text
Context
  -> session hiện tại
  -> state hiện tại
  -> invocation id
  -> services như SessionService, MemoryService
```

Nói ngắn:

```text
Session/State/Memory = dữ liệu cần quản lý.
Context = cửa truy cập dữ liệu đó trong lúc agent chạy.
```

## Ví dụ flow

User bắt đầu chat:

```text
Tôi muốn đặt vé đi Đà Nẵng.
```

Flow:

```text
App tạo hoặc load Session
  -> append user message vào Events
  -> agent hỏi thêm ngày bay
  -> append agent message vào Events
```

User trả lời:

```text
Ngày mai.
```

Agent có thể dùng session history để hiểu:

```text
"Ngày mai" là ngày bay cho chuyến đi Đà Nẵng.
```

Sau khi tool tìm chuyến:

```python
session.state["destination"] = "Đà Nẵng"
session.state["travel_date"] = "tomorrow"
session.state["available_flights"] = ["VN123", "VJ456", "QH789"]
```

User nói:

```text
Chọn chuyến thứ hai.
```

Agent đọc state:

```text
available_flights[1] = VJ456
```

Vậy session + state giúp agent hiểu được câu nói ngắn của user.

## Khi nào dùng State, khi nào dùng Memory?

Dùng `State` nếu dữ liệu thuộc cuộc hội thoại hiện tại.

Ví dụ:

```text
checkout_step = "payment"
selected_flight_id = "VJ456"
current_ticket_id = "TCK-123"
```

Dùng `Memory` nếu dữ liệu cần sống qua nhiều session hoặc cần search lại.

Ví dụ:

```text
user thích tiếng Việt
user thích report dạng bullet
khách hàng thuộc phân khúc enterprise
domain knowledge của công ty
```

Quy tắc:

```text
Tạm thời trong thread hiện tại -> State.
Dài hạn hoặc cross-session -> Memory.
```

## Khi nào dùng Session?

Luôn cần session nếu agent có multi-turn conversation.

Session đặc biệt quan trọng khi:

- user nói tiếp dựa trên câu trước,
- agent cần nhớ lựa chọn trước đó,
- workflow có nhiều bước,
- có tool call và tool result,
- có trạng thái như form/checkout/ticket,
- user có thể quay lại thread cũ.

Nếu app chỉ xử lý single-turn request độc lập, session vẫn có thể tồn tại nhưng ít quan trọng hơn.

## Best practices

### Đặt session scope rõ ràng

Một session nên đại diện cho một conversation thread rõ.

Ví dụ:

```text
Một ticket support -> một session.
Một chat đặt vé -> một session.
Một workflow phân tích tài liệu -> một session.
```

Không nên trộn quá nhiều mục tiêu không liên quan vào cùng một session.

### State chỉ giữ dữ liệu nhỏ

Nên:

```python
session.state["ticket_id"] = "TCK-123"
session.state["step"] = "waiting_for_confirmation"
```

Không nên:

```python
session.state["full_pdf_text"] = "..."
session.state["image_bytes"] = b"..."
```

### Memory không thay thế Session

Memory dùng để search thông tin dài hạn.

Nó không thay thế session history đang diễn ra.

```text
Session giữ mạch hội thoại hiện tại.
Memory giữ tri thức ngoài hoặc quá khứ.
```

### Production không dùng in-memory nếu cần giữ dữ liệu

In-memory service mất dữ liệu khi restart.

Nếu user cần quay lại session cũ, phải dùng persistent backend.

### Tách source of truth

Không nên để summary hoặc memory là nơi duy nhất giữ dữ liệu quan trọng.

Nếu có dữ liệu quan trọng:

- ID nhỏ lưu vào state,
- file/blob lưu bằng artifact service,
- history lưu trong session events,
- tri thức dài hạn mới đưa vào memory.

## Ghi nhớ

Trang này trả lời câu hỏi:

> ADK quản lý ngữ cảnh hội thoại nhiều lượt bằng những khái niệm nào?

Câu trả lời:

```text
Session -> cuộc hội thoại hiện tại.
Events -> lịch sử trong session.
State -> dữ liệu nhỏ của session.
Memory -> thông tin dài hạn/searchable.
SessionService -> quản lý session/state/events.
MemoryService -> quản lý memory.
```

Cách chọn nhanh:

```text
Cần nhớ trong cuộc chat hiện tại -> Session + State.
Cần xem lại những gì đã xảy ra -> Events.
Cần nhớ qua nhiều cuộc chat -> Memory.
Cần lưu bền vững -> persistent service, không dùng in-memory.
```

## Source

- [Introduction to Conversational Context: Session, State, and Memory](https://adk.dev/sessions/)
