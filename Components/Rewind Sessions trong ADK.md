---
title: Rewind Sessions trong ADK
tags:
  - adk
  - components
  - sessions
  - rewind
  - state
created: 2026-06-07
up:
  - "[[Session Object trong ADK]]"
source:
  - https://adk.dev/sessions/session/rewind/
---

# Rewind Sessions trong ADK

Liên kết: [[Session Object trong ADK]]

## Rewind sessions là gì?

`Rewind sessions` trong ADK là cơ chế quay một session về trạng thái trước một request/invocation cụ thể.

Nói ngắn:

```text
Rewind = undo một invocation và tất cả invocation sau nó trong session.
```

Dùng khi:

- agent đi sai nhánh,
- tool cập nhật state sai,
- muốn quay lại điểm tốt trước đó,
- muốn thử hướng xử lý khác,
- muốn restart workflow từ một mốc đã biết là đúng.

Doc ghi feature này hỗ trợ trong ADK Python v1.17.0.

## Cách hiểu đơn giản

Giả sử session có 3 request:

```text
A: set state color = red
B: update state color = blue
C: update state size = large
```

Nếu muốn quay về trạng thái sau A, bạn chỉ định invocation của B:

```text
rewind_before_invocation_id = invocation_id của B
```

Kết quả:

```text
A còn hiệu lực.
B bị undo.
C cũng bị undo.
Session quay về trạng thái trước B.
```

Vì vậy tên tham số là:

```text
rewind_before_invocation_id
```

Tức là rewind về trước invocation được chỉ định.

## Cách gọi rewind

Rewind được gọi trên `Runner`.

Ví dụ theo doc:

```python
await runner.rewind_async(
    user_id=USER_ID,
    session_id=session.id,
    rewind_before_invocation_id=rewind_invocation_id,
)
```

Các thông tin cần có:

| Tham số | Ý nghĩa |
|---|---|
| `user_id` | user sở hữu session |
| `session_id` | session cần rewind |
| `rewind_before_invocation_id` | invocation mốc; rewind về trước invocation này |

## Lấy invocation id

Sau khi gọi agent, events trả về có `invocation_id`.

Ví dụ:

```python
events_list = await call_agent_async(
    runner,
    USER_ID,
    session.id,
    "update state color to blue",
)

rewind_invocation_id = events_list[1].invocation_id
```

Sau đó dùng id này:

```python
await runner.rewind_async(
    user_id=USER_ID,
    session_id=session.id,
    rewind_before_invocation_id=rewind_invocation_id,
)
```

Nếu invocation đó là lần đổi `color` sang `blue`, sau rewind state có thể quay lại `red`.

## Rewind khôi phục gì?

Khi rewind, ADK khôi phục các tài nguyên ở cấp session do ADK quản lý.

Bao gồm:

- session-level state,
- session-level artifacts,
- event context được dùng để chuẩn bị request tiếp theo cho model.

Ví dụ:

```text
Trước B:
  state["color"] = "red"

B:
  state["color"] = "blue"

Rewind trước B:
  state["color"] = "red"
```

## Rewind không xóa log

Điểm quan trọng:

```text
Rewind không xóa các request cũ khỏi log.
```

ADK tạo một rewind request đặc biệt.

Các request đã bị rewind vẫn được giữ lại để:

- debug,
- analysis,
- audit,
- hiểu lịch sử thật đã xảy ra.

Nhưng khi chuẩn bị request tiếp theo cho AI model, ADK sẽ bỏ qua phần đã rewind.

Nói cách khác:

```text
Human/debugger vẫn thấy lịch sử đầy đủ.
Model thì coi như phần bị rewind chưa xảy ra.
```

## Rewind khác delete session

| Hành động | Ý nghĩa |
|---|---|
| `delete_session` | Xóa cả session |
| `rewind_async` | Giữ session, nhưng quay về trước một invocation |

Dùng `delete_session` khi không cần conversation nữa.

Dùng `rewind` khi muốn tiếp tục từ một mốc cũ.

## Rewind khác sửa state thủ công

Sửa state thủ công chỉ đổi một vài key.

Rewind xử lý theo timeline invocation.

Nó undo:

```text
invocation được chỉ định
và tất cả invocation sau nó
```

Đồng thời khôi phục session-level state/artifacts về mốc trước invocation.

Vì vậy rewind phù hợp hơn khi workflow đã đi sai nhiều bước.

## Limitation: app-level và user-level không được rewind

Doc nhấn mạnh:

```text
Rewind chỉ khôi phục session-level resources.
```

Nó không khôi phục:

- app-level state,
- user-level state,
- app-level artifacts,
- user-level artifacts.

Ví dụ:

```text
state["color"] -> session-level, có thể rewind.
state["user:preferred_color"] -> user-level, không được rewind.
state["app:default_color"] -> app-level, không được rewind.
```

Tương tự với artifacts:

```text
contract.pdf -> session-level artifact, có thể được restore.
user:profile.pdf -> user-level artifact, không được restore bởi rewind.
```

## Limitation: external systems không được rollback

Nếu tool gọi hệ thống bên ngoài, ADK không tự rollback hệ thống đó.

Ví dụ tool đã:

- tạo order trong database ngoài,
- gửi email,
- gọi payment API,
- cập nhật CRM,
- tạo ticket thật,
- ghi log vào service ngoài.

Sau đó bạn rewind session:

```text
ADK session có thể quay lại.
External system vẫn giữ side effect đã xảy ra.
```

Ví dụ nguy hiểm:

```text
Tool charge tiền user.
Rewind session.
```

Tiền không tự refund.

Bạn phải tự thiết kế rollback/compensation logic nếu tool có side effect ngoài ADK.

## Limitation: không atomic toàn bộ

Doc lưu ý:

```text
State updates, artifact updates, event persistence không nằm trong một atomic transaction duy nhất.
```

Điều này nghĩa là nên tránh:

- rewind session đang active,
- rewind trong lúc tool/session khác đang thao tác artifact,
- concurrent manipulation trên cùng session khi rewind.

Nếu không, có thể sinh inconsistency.

## Khi nào nên dùng rewind?

Nên dùng khi:

- workflow thử nghiệm bị sai,
- agent update state sai,
- user muốn quay về bước trước,
- bạn muốn test nhiều nhánh xử lý,
- cần khôi phục session về mốc an toàn.

Ví dụ:

```text
User đang cấu hình workflow.
Agent chọn sai option.
Bạn rewind về trước invocation chọn sai.
Sau đó user chọn option khác.
```

## Khi nào cần cẩn thận?

Cần cẩn thận nếu:

- tool có side effect ngoài ADK,
- session đang được nhiều process thao tác,
- có user-level/app-level state bị ảnh hưởng,
- artifact đang được update concurrently,
- workflow cần tính nhất quán mạnh.

Nếu side effect ngoài ADK quan trọng, nên thiết kế:

- idempotency key,
- rollback API,
- compensation action,
- audit log riêng,
- confirmation trước khi gọi action irreversible.

## Best practices

### Lưu mốc invocation quan trọng

Nếu app có UI cho user quay lại bước trước, nên lưu hoặc expose invocation id tương ứng với các checkpoint.

Ví dụ:

```text
checkpoint: before_payment
invocation_id: inv_abc
```

### Không rewind active session

Tránh rewind khi session đang có request/tool đang chạy.

Nên để session idle hoặc lock session trước khi rewind.

### Phân biệt state scope

Chỉ session-level state được restore.

Nếu dùng `user:` hoặc `app:`, đừng kỳ vọng rewind khôi phục.

### External side effect cần bù trừ

Nếu tool đã thay đổi hệ thống ngoài, rewind session không đủ.

Thiết kế thêm:

```text
rollback
compensation
manual review
```

### Dùng rewind cho workflow thử nhánh

Rewind hữu ích khi muốn:

```text
Quay lại mốc trước đó
  -> thử decision khác
  -> giữ audit log đầy đủ
```

## Ghi nhớ

Trang này trả lời câu hỏi:

> Làm sao quay một session về trạng thái trước đó mà vẫn giữ log để debug/audit?

Câu trả lời:

> Dùng `runner.rewind_async(...)` với `rewind_before_invocation_id`.

Quy tắc nhanh:

```text
Rewind trước invocation được chỉ định.
Undo invocation đó và mọi invocation sau nó.
Chỉ restore session-level state/artifacts.
Không restore app/user-level resources.
Không rollback external systems.
Không nên chạy concurrent rewind trên active session.
```

## Source

- [Rewind sessions](https://adk.dev/sessions/session/rewind/)
