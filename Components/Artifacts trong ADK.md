---
title: Artifacts trong ADK
tags:
  - adk
  - components
  - artifacts
  - storage
created: 2026-06-07
up:
  - "[[Components]]"
source:
  - https://adk.dev/artifacts/
---

# Artifacts trong ADK

Liên kết: [[Components]]

## Artifacts là gì?

`Artifacts` trong ADK là cơ chế để lưu và quản lý dữ liệu dạng file hoặc binary data.

Ví dụ:

- PDF.
- Ảnh PNG/JPG.
- Audio.
- Video.
- CSV, Excel.
- File user upload.
- File agent sinh ra, ví dụ report PDF hoặc chart image.

Nói ngắn:

```text
Artifact = file/blob có tên, có version, gắn với session hoặc user.
```

Artifact không phải là text response thông thường của agent.

Artifact cũng không phải là `state`.

## Vì sao cần Artifacts?

Session `state` phù hợp với dữ liệu nhỏ:

```text
state["order_id"] = "ORD-123"
state["step"] = "waiting_for_approval"
state["summary"] = "Tóm tắt ngắn..."
```

Nhưng `state` không phù hợp để lưu file lớn:

```text
state["contract_pdf"] = toàn bộ bytes của file PDF
```

Nếu lưu raw file vào state:

- session phình to,
- history nặng,
- khó cleanup,
- khó version,
- khó dùng lại giữa session,
- tốn memory và storage không cần thiết.

Artifacts giải quyết chuyện này bằng cách tách file/blob ra khỏi session.

```text
Session:
  - message
  - state nhỏ
  - metadata/event về artifact

ArtifactService:
  - file bytes
  - mime_type
  - version
  - storage thật phía sau
```

## Hình dung tổng thể

Ví dụ user upload một file:

```text
contract.pdf
```

Flow hợp lý:

```text
User upload file
  -> app/backend nhận bytes
  -> save_artifact("contract.pdf")
  -> ArtifactService lưu file vào storage
  -> session chỉ giữ metadata/tham chiếu
```

Khi user hỏi về file:

```text
Tóm tắt điều khoản thanh toán trong contract.pdf
```

Flow:

```text
Agent cần đọc file
  -> load_artifact("contract.pdf")
  -> ArtifactService lấy bytes về
  -> nội dung file được đưa vào request hiện tại
  -> agent đọc và trả lời
```

Điểm quan trọng:

```text
Raw file không tự động nằm vĩnh viễn trong session history.
Loaded artifact chủ yếu phục vụ lượt xử lý hiện tại.
Nếu turn sau cần đọc lại, agent load lại artifact.
```

## Ví dụ cụ thể: user upload PDF

User upload:

```text
contract.pdf
```

App/backend nhận bytes của file.

Sau đó lưu thành artifact:

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

Sau khi save:

```text
filename = contract.pdf
version = 0
mime_type = application/pdf
data = bytes của PDF
scope = session hiện tại, nếu filename không có user:
```

Nếu user upload lại cùng tên `contract.pdf`, ADK tạo version mới:

```text
contract.pdf version 0
contract.pdf version 1
contract.pdf version 2
```

Khi load không chỉ định version:

```python
artifact = await context.load_artifact("contract.pdf")
```

ADK lấy version mới nhất.

## Ví dụ cụ thể: agent tạo report

User hỏi:

```text
Phân tích dữ liệu bán hàng và xuất report PDF.
```

Tool của agent tạo PDF bytes.

Thay vì trả bytes vào câu chat, tool lưu artifact:

```python
report_part = types.Part.from_bytes(
    data=report_pdf_bytes,
    mime_type="application/pdf",
)

version = await context.save_artifact(
    filename="sales_report.pdf",
    artifact=report_part,
)
```

Agent có thể trả lời:

```text
Mình đã tạo sales_report.pdf.
```

Ứng dụng của bạn có thể cho user tải file đó từ artifact storage.

Lần sau user hỏi:

```text
Mở lại sales_report.pdf và tóm tắt phần kết luận.
```

Agent load artifact:

```python
report = await context.load_artifact("sales_report.pdf")
```

## Artifact được biểu diễn bằng gì?

Trong ADK, artifact content được biểu diễn bằng `google.genai.types.Part`.

Phần binary nằm trong `inline_data`.

Ví dụ:

```python
image_part = types.Part.from_bytes(
    data=image_bytes,
    mime_type="image/png",
)
```

Một artifact có hai phần quan trọng:

```text
data      = bytes thật của file
mime_type = loại file
```

Ví dụ MIME type:

```text
application/pdf
image/png
image/jpeg
audio/mpeg
text/csv
application/json
```

MIME type rất quan trọng.

File extension như `.pdf` giúp người đọc hiểu, nhưng ADK/tool/model cần `mime_type` để diễn giải dữ liệu đúng.

## ArtifactService là gì?

`ArtifactService` là lớp chịu trách nhiệm lưu, load, list, delete và version artifact.

Artifacts không được lưu trực tiếp trong agent.

Artifacts cũng không được lưu trực tiếp trong session state.

ADK dùng service riêng:

```text
BaseArtifactService
  -> InMemoryArtifactService
  -> GcsArtifactService
  -> custom implementation nếu bạn tự viết
```

Khi tạo `Runner`, phải truyền `artifact_service`.

Ví dụ:

```python
from google.adk.runners import Runner
from google.adk.artifacts import InMemoryArtifactService
from google.adk.sessions import InMemorySessionService

artifact_service = InMemoryArtifactService()

runner = Runner(
    agent=my_agent,
    app_name="my_artifact_app",
    session_service=InMemorySessionService(),
    artifact_service=artifact_service,
)
```

Nếu không có `artifact_service`, các thao tác sau có thể lỗi:

```python
save_artifact()
load_artifact()
list_artifacts()
```

## BaseArtifactService cần làm gì?

Theo doc, artifact service cần hỗ trợ các thao tác chính:

- Save artifact.
- Load artifact.
- List artifact keys.
- Delete artifact.
- List versions.

Nói cách khác, ADK không bắt buộc file phải nằm trong session.

ADK chỉ cần một service biết:

```text
filename này nằm ở đâu?
version mới nhất là version nào?
load bytes về như thế nào?
list các file có trong session/user ra sao?
delete file như thế nào?
```

## InMemoryArtifactService

`InMemoryArtifactService` lưu artifact trong memory của process.

Ưu điểm:

- dễ dùng,
- nhanh,
- không cần cloud setup,
- hợp để test và demo.

Nhược điểm:

- app restart là mất dữ liệu,
- không phù hợp production,
- không hợp lưu nhiều file lớn lâu dài.

Dùng khi:

```text
local dev
unit test
demo ngắn
prototype
```

Ví dụ:

```python
from google.adk.artifacts import InMemoryArtifactService

artifact_service = InMemoryArtifactService()
```

## GcsArtifactService

`GcsArtifactService` lưu artifact vào Google Cloud Storage.

Ưu điểm:

- persistent,
- phù hợp production,
- scale tốt,
- file còn sau khi app restart/deploy,
- nhiều instance app có thể dùng chung bucket.

Nhược điểm:

- cần GCS bucket,
- cần credentials,
- cần IAM permissions,
- có chi phí storage/network.

Dùng khi:

```text
production
file upload thật
generated reports
long-term user data
multi-instance deployment
```

Ví dụ:

```python
from google.adk.artifacts import GcsArtifactService

artifact_service = GcsArtifactService(
    bucket_name="your-gcs-bucket-for-adk-artifacts",
)
```

## Có dùng MinIO, Cloudinary, S3 được không?

Có thể dùng, nhưng cần hiểu rõ:

Doc built-in nhắc đến:

```text
InMemoryArtifactService
GcsArtifactService
```

Nếu muốn dùng backend khác như:

- MinIO,
- Cloudinary,
- AWS S3,
- Azure Blob,
- Supabase Storage,
- database riêng,

thì bạn cần tự viết custom implementation của `BaseArtifactService`.

Đây là suy luận từ kiến trúc `BaseArtifactService`: ADK làm việc qua interface, còn backend thật có thể là storage khác nếu bạn implement đủ các method.

## MinIO làm artifact backend

MinIO khá hợp với artifact vì nó là object storage kiểu S3.

Bạn có thể map artifact thành object key.

Ví dụ key:

```text
artifacts/{app_name}/{user_id}/{session_id}/{filename}/{version}
```

Ví dụ thật:

```text
artifacts/my_app/user_123/session_abc/contract.pdf/0
artifacts/my_app/user_123/session_abc/contract.pdf/1
```

Khi `save_artifact("contract.pdf")`:

```text
1. Tìm version mới nhất hiện có.
2. Tính version tiếp theo.
3. Upload bytes lên MinIO.
4. Lưu metadata nếu cần.
5. Trả version về ADK.
```

Khi `load_artifact("contract.pdf")`:

```text
1. Nếu không truyền version, tìm version mới nhất.
2. Download bytes từ MinIO.
3. Tạo types.Part với data + mime_type.
4. Trả về cho ADK.
```

## Cloudinary làm artifact backend

Cloudinary cũng dùng được, nhưng hợp nhất với media:

- image,
- video,
- audio,
- file cần CDN URL,
- file cần transformation.

Khi dùng Cloudinary, bạn cần tự map:

```text
filename -> public_id
version -> public_id hoặc metadata
mime_type -> metadata/resource type
load -> tải bytes hoặc lấy secure_url rồi fetch
list -> list resources theo prefix/tag
delete -> delete resource
```

Cloudinary phù hợp nếu app của bạn chủ yếu xử lý ảnh/video.

Nếu cần artifact private, version rõ ràng, PDF/document nhiều, MinIO/S3/GCS thường tự nhiên hơn.

## Chỉ lưu URL trong session có được không?

Có, nhưng lúc đó bạn không thật sự dùng artifact abstraction của ADK.

Ví dụ:

```text
state["contract_url"] = "https://storage.example.com/contract.pdf"
```

Cách này chạy được, nhưng bạn phải tự xử lý:

- list file,
- load file,
- version,
- permission,
- cleanup,
- user/session scope,
- mapping URL với file.

Nếu dùng ArtifactService:

```text
Agent chỉ cần biết filename/version.
Storage URL thật nằm sau ArtifactService.
```

Thiết kế sạch hơn:

```text
User upload file
  -> ArtifactService.save_artifact()
      -> backend thật là GCS/MinIO/Cloudinary
  -> agent dùng load_artifact/list_artifacts
```

## Session có lưu raw file không?

Không nên.

Theo cách dùng artifact chuẩn:

```text
Artifact bytes nằm ở ArtifactService/storage.
Session chỉ giữ hội thoại, state nhỏ, và record/metadata về artifact.
```

Khi agent load artifact:

```text
load_artifact("contract.pdf")
  -> lấy file từ storage
  -> đưa vào context hiện tại cho model/tool đọc
  -> phục vụ lượt trả lời hiện tại
```

Nội dung file không tự động được lưu vĩnh viễn vào session history.

Nhưng có ngoại lệ do bạn tự thiết kế:

- Nếu custom tool return toàn bộ nội dung file dưới dạng text, text đó có thể đi vào model context.
- Nếu agent trả lời bằng summary, summary nằm trong conversation như bình thường.
- Nếu bạn tự ghi raw content vào `state`, nó sẽ nằm trong state.

Best practice:

```text
Không lưu raw file vào session.
Không lưu file text dài vào state.
Chỉ lưu filename/version/metadata.
Khi cần thì load artifact lại.
Nếu cần nhớ lâu, lưu summary ngắn vào state hoặc memory.
```

## LoadArtifactsTool

`LoadArtifactsTool` cho phép model tự quyết định khi nào cần load artifact.

Ví dụ:

```python
from google.adk.agents import LlmAgent
from google.adk.tools.load_artifacts_tool import LoadArtifactsTool

root_agent = LlmAgent(
    name="artifact_reader",
    model="gemini-flash-latest",
    instruction=(
        "Answer questions about available user files. "
        "Call load_artifacts before answering when you need file contents."
    ),
    tools=[
        LoadArtifactsTool(),
    ],
)
```

Nếu user hỏi:

```text
Tóm tắt contract.pdf
```

Agent có thể gọi `load_artifacts`.

ADK tạm đưa nội dung artifact được chọn vào request hiện tại.

Điểm quan trọng:

```text
Loaded artifact content không được save vĩnh viễn vào session history.
Nếu turn sau cần cùng artifact, model nên gọi load_artifacts lại.
```

## Session scope và user scope

Artifact có hai scope chính.

### Session scope

Filename bình thường:

```text
contract.pdf
summary.txt
chart.png
```

Artifact gắn với:

```text
app_name + user_id + session_id + filename
```

Nó chỉ thuộc session hiện tại.

Dùng cho:

- file trong một cuộc chat cụ thể,
- report tạm,
- output chỉ cần trong session.

### User scope

Filename có prefix `user:`.

Ví dụ:

```text
user:profile.png
user:settings.json
user:avatar.jpg
```

Artifact gắn với:

```text
app_name + user_id + filename
```

Nó có thể được dùng qua nhiều session của cùng user.

Dùng cho:

- avatar,
- file cấu hình cá nhân,
- profile document,
- data user muốn dùng lại lâu dài.

## Versioning

Mỗi lần save cùng filename, ADK tạo version mới.

Ví dụ:

```python
await context.save_artifact("report.pdf", report_v1)  # version 0
await context.save_artifact("report.pdf", report_v2)  # version 1
await context.save_artifact("report.pdf", report_v3)  # version 2
```

Load latest:

```python
latest = await context.load_artifact("report.pdf")
```

Load version cụ thể:

```python
old = await context.load_artifact(
    filename="report.pdf",
    version=0,
)
```

Cách nhớ:

```text
same filename + save nhiều lần = nhiều version
load không truyền version = latest
```

## List artifacts

Agent/tool có thể list các artifact hiện có.

Ví dụ:

```python
files = await context.list_artifacts()
```

Kết quả có thể là:

```text
contract.pdf
sales_report.pdf
chart.png
user:settings.json
```

Việc list artifact giúp agent biết user/session hiện có những file nào.

## Delete artifacts

`BaseArtifactService` có method delete artifact.

Nhưng doc lưu ý delete không được expose trực tiếp qua context object cho an toàn.

Nếu cần cleanup, nên có:

- admin tool riêng,
- lifecycle policy nếu dùng GCS,
- scheduled cleanup job,
- convention filename cho artifact tạm.

## Artifact khác State khác Memory

| Cơ chế | Dùng để lưu | Ví dụ |
|---|---|---|
| `state` | Dữ liệu nhỏ, điều khiển workflow | `step`, `order_id`, `validation_status` |
| `artifact` | File/blob/binary data | PDF, image, audio, CSV |
| `memory` | Tri thức/ngữ cảnh dài hạn dạng semantic | sở thích user, facts cần nhớ |

Cách chọn:

```text
Nhỏ và cần điều khiển flow -> state
File hoặc bytes -> artifact
Tri thức lâu dài để retrieve -> memory
```

## Best practices

### Dùng filename rõ nghĩa

Nên:

```text
monthly_report_2026_06.pdf
contract_acme_v1.pdf
chart_revenue_q2.png
```

Không nên:

```text
file.pdf
abc
temp
```

### Set đúng MIME type

Luôn set đúng:

```python
types.Part.from_bytes(
    data=pdf_bytes,
    mime_type="application/pdf",
)
```

Không set sai kiểu:

```text
PDF nhưng mime_type là image/png
```

### Dùng user: cẩn thận

Chỉ dùng `user:` nếu artifact thật sự thuộc user và cần dùng qua nhiều session.

Ví dụ hợp:

```text
user:profile.png
user:settings.json
```

Không nên dùng `user:` cho file chỉ liên quan một cuộc chat.

### Không lưu raw file vào state

Nên:

```text
ArtifactService lưu file.
State lưu filename/version nếu cần.
```

Không nên:

```text
state["file_content"] = toàn bộ PDF text/bytes
```

### Production nên dùng persistent backend

Dev/test:

```text
InMemoryArtifactService
```

Production:

```text
GcsArtifactService
Custom MinIO/S3/Cloudinary ArtifactService
```

### Cần cleanup strategy

Persistent artifact không tự biến mất.

Cần nghĩ đến:

- file tạm hết hạn,
- report cũ,
- upload lỗi,
- artifact nhiều version,
- chi phí storage.

## Ghi nhớ

Artifacts trả lời câu hỏi:

> Agent làm việc với file như PDF, ảnh, audio, CSV thì lưu ở đâu?

Câu trả lời:

> Lưu qua ArtifactService, không nhét raw file vào session state.

Mô hình đúng:

```text
Session giữ context nhỏ.
ArtifactService giữ file.
Agent cần đọc thì load artifact.
Loaded artifact chỉ phục vụ request hiện tại, không tự lưu raw content vào session.
```

Với backend:

```text
Built-in: InMemory, GCS.
Khác GCS: được nếu tự implement BaseArtifactService.
```

## Source

- [Artifacts](https://adk.dev/artifacts/)
