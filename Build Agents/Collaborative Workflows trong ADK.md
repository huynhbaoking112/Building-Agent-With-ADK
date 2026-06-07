---
title: Collaborative Workflows trong ADK
tags:
  - adk
  - build-agents
  - workflows
  - collaboration
created: 2026-06-07
up:
  - "[[Build Agents]]"
source:
  - https://adk.dev/workflows/collaboration/
---

# Collaborative Workflows trong ADK

Liên kết: [[Build Agents]]

## Collaborative Workflows là gì?

`Collaborative workflows` trong ADK là cách xây một team nhiều agent, trong đó có một agent chính đóng vai trò **coordinator** và nhiều subagent chuyên xử lý từng loại nhiệm vụ.

Ý tưởng chính:

```text
User
  -> Coordinator Agent
      -> giao task cho subagent phù hợp
      -> subagent làm việc
      -> subagent hoàn thành task
      -> control quay lại Coordinator Agent
  -> User
```

Cách này phù hợp với các bài toán phức tạp, có nhiều phần việc lớn, mỗi phần cần một agent có trách nhiệm riêng.

Ví dụ:

- `travel_planner`: agent chính lên kế hoạch chuyến đi.
- `weather_checker`: agent kiểm tra thời tiết.
- `flight_booker`: agent tìm hoặc đặt vé máy bay.
- `hotel_finder`: agent tìm khách sạn.

`travel_planner` không cần tự làm mọi thứ. Nó có thể giao từng phần cho subagent phù hợp.

## Vấn đề mà collaborative workflow giải quyết

Nếu chỉ có một agent lớn làm tất cả, instruction dễ dài và khó kiểm soát.

Ví dụ một agent vừa phải:

- hiểu yêu cầu du lịch,
- hỏi thông tin còn thiếu,
- kiểm tra thời tiết,
- tìm chuyến bay,
- tìm khách sạn,
- tổng hợp lịch trình,
- trả lời user.

Khi tách thành nhiều subagent, mỗi agent chỉ cần tập trung vào một trách nhiệm rõ ràng.

```text
travel_planner
  -> weather_checker
  -> flight_booker
  -> hotel_finder
```

Coordinator agent chịu trách nhiệm điều phối.

Subagent chịu trách nhiệm làm tốt phần việc của nó.

## Ví dụ code trong doc

```python
from google.adk import Agent

weather_agent = Agent(
    name="weather_checker",
    mode="single_turn",  # no user interaction
    tools=[get_weather, user_info, geocode_address],
)

flight_agent = Agent(
    name="flight_booker",
    mode="task",  # can ask user questions
    input_schema=FlightInput,
    output_schema=FlightResult,
    tools=[search_flights, book_flight],
)

root = Agent(
    name="travel_planner",  # coordinator agent
    sub_agents=[weather_agent, flight_agent],
)
```

Trong ví dụ này:

- `travel_planner` là coordinator agent.
- `weather_checker` là subagent chạy một lượt, không hỏi user.
- `flight_booker` là subagent xử lý một task, có thể hỏi user nếu cần làm rõ.
- Khi subagent hoàn thành task, control quay lại parent agent.

## Mode của subagent

Điểm quan trọng nhất trong collaborative workflows là `mode`.

`mode` định nghĩa cách subagent hoạt động khi được coordinator gọi.

ADK có ba mode:

- `chat`
- `task`
- `single_turn`

## Mode chat

`chat` là mode mặc định.

Với `chat`, subagent có thể tương tác đầy đủ với user.

```python
agent = Agent(
    name="support_agent",
    mode="chat",
)
```

Đặc điểm:

- User có thể chat tự do với subagent.
- Subagent giữ control sau khi được chuyển sang.
- Muốn quay lại parent thì cần handoff thủ công.
- Đây là hành vi giống subagent truyền thống.

Flow:

```text
Coordinator
  -> chuyển control cho chat subagent
      -> subagent trả lời user
      -> subagent tiếp tục giữ conversation
      -> cần transfer thủ công nếu muốn quay lại parent
```

Nên dùng khi muốn subagent trực tiếp trò chuyện với user như một agent độc lập.

## Mode task

`task` dùng khi subagent nhận một nhiệm vụ cụ thể từ coordinator.

```python
flight_agent = Agent(
    name="flight_booker",
    mode="task",
    input_schema=FlightInput,
    output_schema=FlightResult,
    tools=[search_flights, book_flight],
)
```

Đặc điểm:

- Subagent xử lý một task cụ thể.
- Có thể hỏi user để làm rõ thông tin.
- Khi hoàn thành, subagent tự động quay lại parent.
- Control không bị kẹt mãi ở subagent.

Flow:

```text
Coordinator
  -> request task cho subagent
      -> subagent xử lý
      -> nếu thiếu thông tin, subagent hỏi user
      -> subagent hoàn thành task
  -> control quay lại Coordinator
```

Ví dụ:

User hỏi:

```text
Đặt giúp tôi chuyến bay đi Đà Nẵng cuối tuần này.
```

`travel_planner` giao task cho `flight_booker`.

`flight_booker` có thể hỏi:

```text
Bạn muốn bay từ sân bay nào?
```

Sau khi đủ thông tin, `flight_booker` tìm chuyến bay và hoàn thành task. Control quay lại `travel_planner`.

## Mode single_turn

`single_turn` dùng khi subagent chỉ cần chạy một lượt và không được hỏi user.

```python
weather_agent = Agent(
    name="weather_checker",
    mode="single_turn",
    tools=[get_weather],
)
```

Đặc điểm:

- Không tương tác với user.
- Nhận task, xử lý, trả kết quả ngay.
- Có thể chạy song song nhiều task.
- Phù hợp với tác vụ ngắn, rõ input, rõ output.

Flow:

```text
Coordinator
  -> gọi single_turn subagent
      -> subagent xử lý một lượt
      -> trả kết quả
  -> control quay lại Coordinator
```

Ví dụ:

- kiểm tra thời tiết,
- lấy tỷ giá,
- phân tích nhanh một đoạn text,
- kiểm tra trạng thái đơn hàng nếu đã có mã đơn.

## So sánh ba mode

| Tiêu chí | `chat` | `task` | `single_turn` |
|---|---|---|---|
| Tương tác với user | Đầy đủ | Chỉ khi cần làm rõ | Không |
| Control flow | Subagent giữ control | Quay lại parent khi task xong | Quay lại parent ngay sau một lượt |
| Handoff về parent | Thủ công | Tự động | Tự động |
| Chạy song song | Không hỗ trợ | Không hỗ trợ | Có thể |
| Phù hợp với | Subagent chat trực tiếp | Task có nhiều bước | Task ngắn, rõ ràng |

## Cách hiểu bằng ví dụ travel planner

Giả sử user hỏi:

```text
Lên kế hoạch đi Đà Nẵng cuối tuần này, kiểm tra thời tiết và tìm vé máy bay.
```

Thiết kế có thể là:

```text
travel_planner
  sub_agents:
    - weather_checker mode="single_turn"
    - flight_booker mode="task"
```

`weather_checker` dùng `single_turn` vì:

- chỉ cần kiểm tra thời tiết,
- không cần hỏi user,
- làm xong trả kết quả ngay.

`flight_booker` dùng `task` vì:

- có thể thiếu thông tin chuyến bay,
- có thể cần hỏi user thêm,
- có quy trình riêng,
- làm xong mới trả kết quả về coordinator.

Sau đó `travel_planner` nhận kết quả từ các subagent và tiếp tục xử lý.

## Control flow khi dùng với workflow node

Doc có nhấn mạnh rằng cùng một task agent có thể được gọi trong hai ngữ cảnh khác nhau.

### Khi là node trong workflow graph

Nếu task agent được đặt trong workflow graph, ví dụ trong `SequentialAgent` hoặc `ParallelAgent`, thì khi agent hoàn thành task, workflow sẽ đi tiếp sang node kế tiếp theo logic của graph.

```text
Workflow Node A
  -> task agent
      -> task done
  -> Workflow Node B
```

Tức là control đi theo graph.

### Khi được transfer từ LlmAgent

Nếu parent `LlmAgent` chuyển task cho subagent, subagent chạy cho đến khi hoàn thành task. Sau đó control quay lại agent đã gọi nó.

```text
Parent LlmAgent
  -> task subagent
      -> complete task
  -> quay lại Parent LlmAgent
```

Tức là control quay lại agent gốc đã request task.

## Context isolation

Với `task` và `single_turn`, mỗi subagent chạy trong một session branch riêng.

Điều này có nghĩa là:

- Subagent chỉ thấy context trong branch của nó.
- Nếu nhiều subagent chạy song song, chúng không thấy event của nhau.
- Khi các branch hoàn thành, parent nhận kết quả đã gom lại.

Ví dụ:

```text
travel_planner
  -> weather_checker branch
  -> hotel_finder branch
  -> flight_booker branch
```

Trong lúc chạy:

- `weather_checker` không thấy `hotel_finder` đang làm gì.
- `hotel_finder` không thấy `flight_booker` đang làm gì.
- `travel_planner` nhận kết quả sau khi các branch hoàn thành.

Context isolation giúp workflow dễ dự đoán hơn, đặc biệt khi chạy song song.

## Lưu ý quan trọng

`mode` chỉ dùng cho subagent.

Không nên đặt `mode` cho root agent.

Sai:

```python
root = Agent(
    name="travel_planner",
    mode="task",
)
```

Đúng:

```python
flight_agent = Agent(
    name="flight_booker",
    mode="task",
)

root = Agent(
    name="travel_planner",
    sub_agents=[flight_agent],
)
```

Root agent là coordinator. Subagent mới là nơi đặt `mode`.

## Limitation trong ADK Python v2.0.0

Theo doc, trong ADK Python v2.0.0:

- `task` mode đang bị disable khi dùng trong graph-based workflows.
- Task mode agent phải là leaf agent.

Leaf agent nghĩa là task agent không nên có subagent con.

Ví dụ nên tránh:

```text
travel_planner
  -> flight_booker mode="task"
      -> seat_picker
      -> payment_agent
```

Nếu `flight_booker` là task mode agent, nó nên là leaf agent, không phải parent của các subagent khác.

## Khi nào nên dùng collaborative workflow

Nên dùng khi:

- Bài toán có nhiều phần việc lớn.
- Mỗi phần việc có trách nhiệm riêng.
- Coordinator cần giao việc cho nhiều specialist.
- Một số specialist có thể hỏi user để làm rõ.
- Muốn subagent tự quay lại parent khi xong task.
- Muốn team agent có hành vi dự đoán hơn nhờ `mode`.

Ví dụ phù hợp:

- travel planning,
- customer support,
- research workflow,
- booking workflow,
- refund workflow,
- onboarding workflow,
- sales assistant nhiều bước.

## Khi nào không cần dùng

Không cần dùng collaborative workflow nếu:

- Chỉ có một agent đơn giản là đủ.
- Chỉ cần một vài function tool.
- Không có vai trò subagent rõ ràng.
- Không cần delegate task.
- Flow cố định và nên dùng graph rõ ràng hơn.

Nếu workflow là một pipeline cố định, dùng graph route có thể dễ kiểm soát hơn.

Nếu workflow có loop, resume, checkpoint, dynamic branching phức tạp, dynamic workflow có thể hợp hơn.

## Ghi nhớ

Collaborative workflow trả lời câu hỏi:

> Làm sao xây một team agent, trong đó coordinator giao việc cho subagent chuyên trách, và subagent tự quay lại parent khi làm xong?

Ba mode cần nhớ:

```text
chat        = subagent chat tự do, handoff thủ công
task        = subagent làm task, có thể hỏi user, xong tự quay lại parent
single_turn = subagent chạy một lượt, không hỏi user, xong trả kết quả ngay
```

## Source

- [Collaborative workflows](https://adk.dev/workflows/collaboration/)
