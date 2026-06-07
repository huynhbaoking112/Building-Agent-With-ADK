---
title: Context Compaction trong ADK
tags:
  - adk
  - components
  - context
  - compaction
  - compression
created: 2026-06-07
up:
  - "[[Context trong ADK]]"
source:
  - https://adk.dev/context/compaction/
---

# Context Compaction trong ADK

Liên kết: [[Context trong ADK]]

## Context compaction là gì?

`Context compaction` trong ADK là cơ chế nén context khi agent chạy lâu.

Trang doc cũng gọi nó là `Context compression`.

Nói ngắn:

```text
Context compaction = tóm tắt các event cũ trong session để context nhẹ hơn.
```

Khi agent chạy, session sẽ tích lũy nhiều dữ liệu:

- user instructions,
- user messages,
- retrieved data,
- tool responses,
- generated content,
- events trong workflow.

Session càng dài thì context gửi vào model càng lớn.

Hệ quả:

- model xử lý chậm hơn,
- tốn nhiều token hơn,
- response latency tăng,
- agent dễ bị quá tải context nếu workflow dài.

Context compaction giải quyết bằng cách tóm tắt các event cũ thành summary, thay vì giữ toàn bộ event cũ ở dạng raw.

## Hình dung đơn giản

Không compaction:

```text
Event 1 raw
Event 2 raw
Event 3 raw
Event 4 raw
Event 5 raw
Event 6 raw
...
```

Có compaction:

```text
Summary của events cũ
Event gần đây raw
Event gần đây raw
```

Agent vẫn có bối cảnh chính, nhưng không phải gửi toàn bộ lịch sử chi tiết vào model.

## Vì sao cần context compaction?

Agent workflow thường không chỉ là chat đơn giản.

Một workflow có thể có:

- nhiều lượt user hỏi,
- nhiều lần gọi tool,
- tool response rất dài,
- nhiều retrieved documents,
- nhiều bước reasoning,
- nhiều agent hoặc sub-workflow.

Nếu cứ giữ toàn bộ event history, mỗi model request sau phải mang theo nhiều dữ liệu hơn request trước.

```text
Turn 1 -> context nhỏ
Turn 5 -> context lớn hơn
Turn 20 -> context rất lớn
```

Compaction giúp session dài vẫn chạy được với chi phí và latency hợp lý hơn.

## Cách hoạt động

ADK dùng cách tiếp cận kiểu sliding window.

Ý tưởng:

```text
Giữ các event gần đây ở dạng raw.
Tóm tắt các event cũ thành summary.
Khi đủ điều kiện, tiếp tục compact window mới.
```

Trong Python và Java, trigger compaction dựa trên số event/invocation đã hoàn tất.

Trong TypeScript, trigger có thể dựa trên token threshold.

Sau khi cấu hình, `Runner` xử lý compaction ở background khi session đạt ngưỡng.

## Cấu hình Python

Trong Python, cấu hình ở cấp `App` bằng `events_compaction_config`.

```python
from google.adk.apps.app import App
from google.adk.apps.app import EventsCompactionConfig

app = App(
    name="my-agent",
    root_agent=root_agent,
    events_compaction_config=EventsCompactionConfig(
        compaction_interval=3,
        overlap_size=1,
    ),
)
```

Ý nghĩa:

| Setting | Nghĩa |
|---|---|
| `compaction_interval` | Sau bao nhiêu event/invocation hoàn tất thì trigger compaction |
| `overlap_size` | Giữ lại bao nhiêu event cũ làm phần chồng lấp giữa các window |
| `summarizer` | Optional, chỉ định model/cách tóm tắt riêng |

Ví dụ:

```python
EventsCompactionConfig(
    compaction_interval=3,
    overlap_size=1,
)
```

Nghĩa là:

```text
Cứ mỗi 3 event/invocation hoàn tất thì compact.
Khi compact window mới, lấy thêm 1 event từ window trước để giữ mạch ngữ cảnh.
```

## Ví dụ compaction interval và overlap

Nếu cấu hình:

```text
compaction_interval = 3
overlap_size = 1
```

Flow:

```text
Event 3 hoàn tất
  -> nén events 1-3 thành summary

Event 6 hoàn tất
  -> nén events 3-6
  -> event 3 là overlap từ window trước

Event 9 hoàn tất
  -> nén events 6-9
  -> event 6 là overlap từ window trước
```

Overlap giúp summary mới không bị mất mạch nối.

Nếu không có overlap, summary sau có thể thiếu bối cảnh ở ranh giới giữa hai window.

## Cấu hình TypeScript

Trong TypeScript, cấu hình trên `LlmAgent` bằng `contextCompactors`.

```ts
import {
  Gemini,
  LlmAgent,
  LlmSummarizer,
  TokenBasedContextCompactor,
} from "@google/adk";

const agent = new LlmAgent({
  name: "my-agent",
  model: "gemini-flash-latest",
  contextCompactors: [
    new TokenBasedContextCompactor({
      tokenThreshold: 1000,
      eventRetentionSize: 1,
      summarizer: new LlmSummarizer({
        llm: new Gemini({ model: "gemini-flash-latest" }),
      }),
    }),
  ],
});
```

Ý nghĩa:

| Setting | Nghĩa |
|---|---|
| `tokenThreshold` | Khi session vượt ngưỡng token này thì compact |
| `eventRetentionSize` | Giữ lại ít nhất bao nhiêu event raw gần nhất |
| `summarizer` | Model/cơ chế dùng để tạo summary |

Khác Python/Java:

```text
Python/Java -> thường cấu hình theo số event/invocation.
TypeScript -> ví dụ doc dùng token threshold.
```

## Summarizer là gì?

`summarizer` là model hoặc component dùng để tóm tắt event history.

Bạn có thể để mặc định hoặc tự chỉ định model summarization.

Ví dụ Python:

```python
from google.adk.apps.app import App, EventsCompactionConfig
from google.adk.apps.llm_event_summarizer import LlmEventSummarizer
from google.adk.models import Gemini

summarization_llm = Gemini(model="gemini-flash-latest")
my_summarizer = LlmEventSummarizer(llm=summarization_llm)

app = App(
    name="my-agent",
    root_agent=root_agent,
    events_compaction_config=EventsCompactionConfig(
        compaction_interval=3,
        overlap_size=1,
        summarizer=my_summarizer,
    ),
)
```

Dùng custom summarizer khi:

- muốn dùng model rẻ hơn cho summary,
- muốn dùng model nhanh hơn,
- muốn kiểm soát prompt tóm tắt,
- muốn summary giữ một số loại thông tin quan trọng.

## Compaction khác caching thế nào?

| Cơ chế | Vấn đề xử lý | Cách xử lý |
|---|---|---|
| Context caching | Context tĩnh/lặp lại bị gửi lại nhiều lần | Cache context đó ở phía model |
| Context compaction | Event history trong session dài dần | Tóm tắt event cũ thành summary |

Nói ngắn:

```text
Caching = tối ưu context lặp lại.
Compaction = giảm kích thước lịch sử session.
```

Ví dụ caching:

```text
Instruction dài 20 trang dùng cho mọi request.
```

Ví dụ compaction:

```text
Cuộc hội thoại 30 lượt với nhiều tool response dài.
```

## Compaction khác state, memory, artifact thế nào?

| Cơ chế | Vai trò |
|---|---|
| `state` | Lưu dữ liệu nhỏ để điều khiển workflow |
| `memory` | Lưu hoặc truy hồi tri thức dài hạn |
| `artifact` | Lưu file/blob như PDF, image, CSV |
| `context caching` | Cache context lớn nhưng lặp lại |
| `context compaction` | Tóm tắt event history cũ trong session |

Cách nhớ:

```text
State -> key/value nhỏ.
Memory -> tri thức dài hạn.
Artifact -> file/blob.
Caching -> không gửi lại context tĩnh.
Compaction -> nén lịch sử dài.
```

## Khi nào nên dùng?

Nên dùng context compaction khi:

- session dài,
- workflow nhiều bước,
- agent gọi nhiều tool,
- tool responses dài,
- user chat nhiều lượt trong cùng session,
- context làm model chậm,
- token usage tăng theo thời gian,
- agent cần giữ mạch tổng quan nhưng không cần nguyên văn toàn bộ lịch sử cũ.

Ví dụ phù hợp:

```text
Research agent chạy nhiều bước:
  - search nhiều lần
  - đọc nhiều tài liệu
  - gọi tool phân tích
  - tổng hợp qua nhiều lượt
```

Hoặc:

```text
Support agent nói chuyện với user trong một session dài:
  - hỏi nhiều thông tin
  - kiểm tra ticket
  - gọi API
  - cập nhật trạng thái
```

## Khi nào không nên dùng hoặc cần cẩn thận?

Cần cẩn thận nếu:

- agent cần quote chính xác nội dung cũ,
- chi tiết nhỏ trong event cũ rất quan trọng,
- tool response cũ chứa dữ kiện không nên bị tóm tắt mất,
- workflow cần audit exact history,
- summary sai có thể làm agent ra quyết định sai.

Compaction là đánh đổi:

```text
Giảm token và latency
  đổi lại
có nguy cơ mất chi tiết trong phần đã tóm tắt.
```

Nếu cần giữ dữ liệu chính xác, nên lưu riêng:

- ID quan trọng vào `state`,
- file/result lớn vào artifact,
- fact cần nhớ lâu vào memory,
- raw logs/audit trail vào storage riêng nếu cần audit.

## Best practices

### Giữ event gần nhất ở dạng raw

Đừng compact quá mạnh.

Nên giữ lại một số event gần nhất để agent thấy tình huống hiện tại ở dạng đầy đủ.

Trong Python/Java:

```text
overlap_size > 0
```

Trong TypeScript:

```text
eventRetentionSize > 0
```

### Đừng xem summary là dữ liệu chính xác tuyệt đối

Summary dùng để giữ mạch ngữ cảnh.

Nó không thay thế source of truth.

Nếu cần dữ liệu chính xác:

```text
Lưu ID/result quan trọng vào state hoặc artifact.
```

### Chọn interval theo độ dài workflow

Nếu interval quá nhỏ:

- summarizer chạy quá thường xuyên,
- có thêm chi phí summary,
- dễ mất chi tiết sớm.

Nếu interval quá lớn:

- context vẫn phình to,
- latency vẫn tăng.

Điểm bắt đầu hợp lý là compact theo các mốc workflow tự nhiên hoặc theo token threshold.

### Dùng summarizer riêng khi cần

Dùng custom summarizer nếu summary cần format cụ thể.

Ví dụ prompt summarizer có thể yêu cầu giữ:

- quyết định đã đưa ra,
- entity IDs,
- user preferences,
- pending tasks,
- tool results quan trọng,
- lỗi đã gặp,
- bước tiếp theo.

## Ví dụ kiến trúc hợp lý

```text
Agent session dài
  -> context compaction tóm tắt event cũ
  -> state giữ workflow flags và IDs quan trọng
  -> artifact giữ file/result lớn
  -> memory giữ thông tin dài hạn về user/domain
```

Mô hình này tránh phụ thuộc quá nhiều vào summary.

```text
Summary giữ mạch câu chuyện.
State/artifact/memory giữ dữ liệu quan trọng đúng chỗ.
```

## Ghi nhớ

Trang này trả lời câu hỏi:

> Khi session agent dài dần, làm sao giảm context gửi vào model mà vẫn giữ được bối cảnh chính?

Câu trả lời:

> Dùng context compaction để tóm tắt event history cũ thành summary.

Quy tắc nhanh:

```text
Session ngắn -> chưa cần compaction.
Session dài, nhiều event/tool response -> nên dùng compaction.
Cần nguyên văn dữ liệu cũ -> lưu riêng, đừng chỉ dựa vào summary.
```

Khác với caching:

```text
Caching xử lý context tĩnh lặp lại.
Compaction xử lý lịch sử session dài.
```

## Source

- [Context compression](https://adk.dev/context/compaction/)
