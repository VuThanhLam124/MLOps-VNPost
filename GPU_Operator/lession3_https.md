# Giao thức gốc HTTP
## Góc nhìn dành cho người mới trong ngành AI / Data Science / MLOps

---

## 1) Vì sao dân AI / DS  phải hiểu phần này?

Nhiều người mới học AI thường tập trung vào model, training, GPU, notebook, vector database, serving. Nhưng khi đưa hệ thống vào thực tế, gần như mọi thành phần đều nói chuyện với nhau qua **HTTP** hoặc **HTTPS**:

- frontend gọi API inference
- service nội bộ gọi model server
- Các hệ thống tracking server nhận log experiment qua HTTP
- FastAPI nhận request dự đoán
- Prometheus scrape metrics qua HTTP endpoint
- webhook từ GitHub / CI/CD gửi event sang server
- object storage, registry, artifact server, model registry thường đều có API web
- hệ thống xác thực, gateway, reverse proxy, load balancer đều xoay quanh request/response

Nếu không hiểu HTTP/HTTPS và ngữ nghĩa của method như `GET`, `POST`, `PUT`, `PATCH`, người làm AI/DS/MLOps sẽ rất dễ:

- gọi sai API
- thiết kế API khó dùng
- debug nhầm lỗi model trong khi lỗi thật nằm ở network / auth / header
- cập nhật tài nguyên không đúng semantics
- gây lỗi cache, retry, hoặc race condition
- để lộ token / dữ liệu nhạy cảm vì dùng sai giao thức

Nói ngắn gọn:  
**Model giải quyết bài toán học máy, nhưng HTTP/HTTPS là một phần quan trọng giúp hệ thống hoạt động được trong thực tế.**

---

## 2) HTTP là gì?

**HTTP** = **HyperText Transfer Protocol**.  
Đây là **giao thức tầng ứng dụng** dùng để client và server trao đổi dữ liệu với nhau.

Ban đầu HTTP chủ yếu dùng cho web page, nhưng hiện nay nó là nền tảng của phần lớn **REST API**, web service, microservice, model serving API, logging API, monitoring API.

Ví dụ:

- notebook gửi request đến FastAPI server để lấy dự đoán
- UI nội bộ gọi endpoint `/predict`
- service A gọi service B để lấy metadata model
- CI/CD gọi API để deploy model mới

HTTP không quan tâm “model là CNN hay LLM”. Nó chỉ quan tâm:

- gửi yêu cầu gì
- đến địa chỉ nào
- bằng method nào
- kèm header gì
- có body hay không
- server trả về status code, header, body gì

---

## 3) HTTPS là gì?

**HTTPS** = **HTTP + TLS/SSL**.

Nghĩa là vẫn là HTTP, nhưng dữ liệu được truyền qua một kênh **mã hóa**.

### 3.1 Mục tiêu chính của HTTPS

HTTPS giúp đạt ba tính chất quan trọng:

- **Confidentiality**: người ngoài không đọc được nội dung truyền đi
- **Integrity**: dữ liệu không bị sửa lén trên đường truyền
- **Authentication**: client có thể kiểm tra đang nói chuyện với đúng server

### 3.2 Nếu chỉ dùng HTTP thì sao?

Nếu dùng HTTP thuần:

- token có thể bị lộ
- password có thể bị lộ
- request chứa dữ liệu khách hàng có thể bị sniff
- model API key nội bộ có thể bị đánh cắp
- dữ liệu truyền đi có thể bị sửa hoặc chèn

Trong môi trường AI/MLOps, đây là vấn đề rất thực tế:

- endpoint inference có thể nhận dữ liệu nhạy cảm
- artifact/model registry có thể chứa model quan trọng
- API nội bộ có bearer token hoặc service account
- dashboard monitoring có thể lộ thông tin hạ tầng

### 3.3 Ví dụ thường gặp

- gọi `http://model.company.local/predict`
- gửi `Authorization: Bearer ...`
- truyền dữ liệu khách hàng không mã hóa

---

## 4) Client, Server, Request, Response

HTTP hoạt động theo mô hình:

- **Client**: phía gửi yêu cầu
- **Server**: phía nhận yêu cầu và trả lời

### 4.1 Client là gì?

Client không nhất thiết là trình duyệt, client có thể là:

- Python script dùng `requests`
- notebook
- frontend React
- service backend
- cronjob
- Airflow task
- ML pipeline step
- Prometheus
- CI/CD runner

### 4.2 Server là gì?

Server là tiến trình lắng nghe request và trả response:

- FastAPI
- Flask
- MLflow server
- Grafana API
- Prometheus endpoint
- model serving service như Triton / vLLM / custom API
- Nginx / reverse proxy

---

## 5) Request là gì?

**Request** là gói thông tin client gửi sang server để yêu cầu một hành động hoặc lấy dữ liệu.

Một request HTTP thường có:

- **method**: GET, POST, PUT, PATCH, DELETE...
- **URL**: địa chỉ tài nguyên
- **headers**: metadata mô tả request
- **body**: dữ liệu gửi kèm, nếu có

### 5.1 Ví dụ request

```http
POST /predict HTTP/1.1
Host: ml-api.company.local
Content-Type: application/json
Authorization: Bearer abc123

{
  "text": "Sản phẩm này rất tốt"
}
```

Ý nghĩa:

- `POST`: muốn gửi dữ liệu để server xử lý
- `/predict`: endpoint dự đoán
- `Content-Type: application/json`: body là JSON
- `Authorization`: gửi token
- body chứa dữ liệu input

---

## 6) Response là gì?

**Response** là thứ server trả về sau khi xử lý request.

Response thường gồm:

- **status code**
- **headers**
- **body**

### 6.1 Ví dụ response

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "label": "positive",
  "score": 0.9821
}
```

### 6.2 Ví dụ response lỗi

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
  "detail": "Invalid token"
}
```

---

## 7) URL là gì trong API?

Một URL thường có dạng:

```text
https://api.company.com/models/123/versions/5?verbose=true
```

Tách ra:

- `https` → scheme / protocol
- `api.company.com` → host
- `/models/123/versions/5` → path
- `?verbose=true` → query parameters

### 7.1 Góc nhìn AI/MLOps

Ví dụ:

- `/predict`
- `/health`
- `/metrics`
- `/models`
- `/models/{model_id}`
- `/runs/{run_id}`
- `/artifacts/...`

---

## 8) Header là gì?

Header là metadata đi kèm request hoặc response.

### 8.1 Header phổ biến trong request

- `Content-Type`: kiểu dữ liệu body
- `Authorization`: token xác thực
- `Accept`: client muốn nhận kiểu dữ liệu gì
- `User-Agent`: client nào đang gọi
- `X-Request-ID`: định danh request để trace log

### 8.2 Header phổ biến trong response

- `Content-Type`
- `Content-Length`
- `Cache-Control`
- `Set-Cookie`
- `ETag`

### 8.3 Với dân MLOps

Header rất quan trọng trong:

- auth qua bearer token
- trace request qua gateway, ingress
- correlation id để debug distributed system
- versioning API
- cache / no-cache cho endpoint metadata

---

## 9) Body là gì?

Body là dữ liệu chính gửi kèm request hoặc response.

Ví dụ AI API:

```json
{
  "instances": [
    {"text": "xin chào"},
    {"text": "hôm nay trời đẹp"}
  ]
}
```

Hoặc:

```json
{
  "image_url": "s3://bucket/path/image.jpg"
}
```

Hoặc response:

```json
{
  "predictions": [
    {"label": "greeting", "score": 0.95},
    {"label": "weather", "score": 0.88}
  ]
}
```

---

## 10) Status code: cách đọc nhanh

Status code là mã phản hồi từ server.

### 10.1 Nhóm phổ biến

- **2xx**: thành công
- **3xx**: chuyển hướng
- **4xx**: lỗi phía client
- **5xx**: lỗi phía server

### 10.2 Các mã nên nhớ

- `200 OK`: thành công
- `201 Created`: tạo mới thành công
- `204 No Content`: thành công nhưng không trả body
- `400 Bad Request`: request sai format / thiếu field
- `401 Unauthorized`: chưa xác thực
- `403 Forbidden`: có xác thực nhưng không có quyền
- `404 Not Found`: không thấy tài nguyên
- `409 Conflict`: xung đột trạng thái
- `422 Unprocessable Entity`: dữ liệu đúng format nhưng không hợp lệ về mặt logic
- `429 Too Many Requests`: bị rate limit
- `500 Internal Server Error`: server lỗi
- `502 Bad Gateway`: gateway nhận lỗi từ upstream
- `503 Service Unavailable`: service tạm unavailable
- `504 Gateway Timeout`: upstream timeout

### 10.3 Debug trong AI/MLOps

- `400` thường là input schema sai
- `401/403` thường là auth hoặc RBAC
- `404` thường là sai endpoint hoặc sai model id
- `422` thường là payload đúng JSON nhưng sai validation
- `429` thường là bị rate limit ở API gateway / provider
- `500` có thể là model server crash, bug code, load model fail
- `503/504` thường là downstream quá tải, startup chưa xong, readiness chưa pass

---

## 11) GET là gì?

`GET` dùng để **lấy dữ liệu** từ server.

### 11.1 Đặc điểm

- thường **không làm thay đổi trạng thái** server
- thường **không có body** trong thực hành phổ biến
- phù hợp cho thao tác đọc
- có thể cache trong nhiều trường hợp

### 11.2 Ví dụ

```http
GET /models HTTP/1.1
Host: api.company.com
```

Hoặc:

```http
GET /runs/123
GET /health
GET /metrics
GET /datasets?page=2&page_size=50
```

### 11.3 Ví dụ AI/MLOps

- lấy danh sách model đã đăng ký
- lấy trạng thái deployment
- lấy health của service
- lấy metrics hiện tại
- lấy metadata của experiment run

### 11.4 Tư duy đúng

Nếu mục tiêu là **đọc**, chưa tạo gì mới, chưa cập nhật gì, hãy ưu tiên nghĩ đến `GET`.

---

## 12) POST là gì?

`POST` thường dùng để **tạo mới tài nguyên** hoặc **gửi dữ liệu để server xử lý**.

### 12.1 Khi nào dùng POST?

- tạo experiment run mới
- upload file
- gửi input cho endpoint inference
- submit job training
- tạo deployment
- trigger một hành động có side effect

### 12.2 Ví dụ

```http
POST /predict
POST /runs
POST /train-jobs
POST /deployments
```

### 12.3 Ví dụ inference API

```http
POST /predict HTTP/1.1
Content-Type: application/json

{
  "text": "đây là ví dụ"
}
```

### 12.4 Vì sao inference thường dùng POST?

Vì input dự đoán thường:

- dài
- phức tạp
- có JSON body
- có batch nhiều sample
- không phù hợp nhét vào query string

Ngoài ra, inference có thể không thuần “read” theo nghĩa HTTP thuần túy, vì server có thể:

- log request
- update counter
- ghi audit trail
- trigger downstream logic

Nên dùng `POST` cho `/predict` là lựa chọn thực tế phổ biến.

---

## 13) PUT là gì?

`PUT` thường dùng để **thay thế toàn bộ tài nguyên** tại một địa chỉ xác định.

### 13.1 Ý tưởng chính

Nếu gọi:

```http
PUT /models/123
```

thì thường hiểu là:

> đây là phiên bản đầy đủ mới của tài nguyên `models/123`; hãy thay thế tài nguyên cũ bằng bản này

### 13.2 Ví dụ

Giả sử tài nguyên cũ là:

```json
{
  "id": 123,
  "name": "fraud-detector",
  "owner": "ml-team",
  "replicas": 2
}
```

Gửi:

```http
PUT /models/123
Content-Type: application/json

{
  "id": 123,
  "name": "fraud-detector-v2",
  "owner": "ml-team",
  "replicas": 4
}
```

Thì thường hiểu là tài nguyên được thay bằng toàn bộ object mới.

### 13.3 Lưu ý

Nhiều hệ thống ngoài đời không tuân thủ quá nghiêm ngặt semantics REST, nhưng về mặt tư duy:

- `PUT` = cập nhật kiểu **full replacement**
- client gửi **toàn bộ trạng thái mong muốn**

### 13.4 Bối cảnh MLOps

Ví dụ hợp lý:

- thay toàn bộ config của một deployment
- thay toàn bộ metadata của một resource
- apply full manifest của một object quản lý nội bộ

---

## 14) PATCH là gì?

`PATCH` dùng để **cập nhật một phần** của tài nguyên.

### 14.1 Ý tưởng chính

Không thay toàn bộ object. Chỉ sửa những field cần sửa.

Ví dụ:

```http
PATCH /models/123
Content-Type: application/json

{
  "replicas": 4
}
```

Ý nghĩa:

- không động đến `name`, `owner`
- chỉ sửa `replicas`

### 14.2 Khi nào PATCH phù hợp?

- update status
- đổi tag
- scale số replicas
- bật/tắt cờ cấu hình
- sửa một vài field metadata

### 14.3 Bối cảnh AI/MLOps

Ví dụ:

- đổi `stage` của model từ `staging` sang `production`
- cập nhật `description`
- scale inference replicas từ 2 lên 4
- bật `canary_enabled = true`

---

## 15) So sánh nhanh GET / POST / PUT / PATCH

| Method | Mục đích chính | Có side effect không? | Body thường dùng? | Ví dụ AI/MLOps |
|---|---|---:|---:|---|
| GET | Lấy dữ liệu | Thường không | Hiếm | lấy model list, health, metrics |
| POST | Tạo mới hoặc submit xử lý | Có | Có | predict, create run, submit job |
| PUT | Thay toàn bộ tài nguyên | Có | Có | thay full config deployment |
| PATCH | Cập nhật một phần tài nguyên | Có | Có | sửa replicas, stage, metadata |

---

## 16) Phân biệt PUT và PATCH cho dễ nhớ

Một cách nhớ đơn giản:

- **PUT** = “đây là bản đầy đủ mới, hãy thay hết”
- **PATCH** = “chỉ sửa vài chỗ này thôi”

### 16.1 Ví dụ đời thường

Giả sử có hồ sơ user:

```json
{
  "name": "An",
  "email": "an@company.com",
  "role": "admin"
}
```

#### PUT
Gửi object đầy đủ mới:

```json
{
  "name": "An Nguyen",
  "email": "an@company.com",
  "role": "admin"
}
```

#### PATCH
Chỉ đổi email:

```json
{
  "email": "an.nguyen@company.com"
}
```

---

## 17) Idempotent là gì, và vì sao cần biết?

Đây là khái niệm rất quan trọng trong hệ thống production.

Một operation gọi là **idempotent** nếu gọi lặp lại nhiều lần với cùng dữ liệu thì kết quả cuối cùng giống nhau.

### 17.1 Ví dụ

- `GET /models/123` gọi 10 lần vẫn chỉ đọc
- `PUT /config/abc` với cùng payload gọi 10 lần, trạng thái cuối cùng vẫn vậy
- `PATCH /resource/1` đặt `enabled=true` nhiều lần, nếu semantics đúng thì kết quả cuối vẫn giống nhau

### 17.2 POST thường không idempotent

Ví dụ:

```http
POST /runs
```

gọi 2 lần có thể tạo ra 2 run khác nhau.

### 17.3 Vì sao MLOps cần quan tâm?

Trong production:

- request có thể bị retry
- network có thể timeout
- load balancer có thể gửi lại
- job runner có thể retry step

Nếu thiết kế sai method, rất dễ:

- tạo duplicate training jobs
- ghi duplicate prediction logs
- tạo nhiều deployment ngoài ý muốn

---

## 18) Query parameter, path parameter, body: dùng cái nào?

### 18.1 Path parameter

Dùng để chỉ định **đúng tài nguyên nào**.

Ví dụ:

- `/models/123`
- `/runs/abc`
- `/deployments/recommender-v1`

### 18.2 Query parameter

Dùng cho:

- filter
- sort
- pagination
- option đọc thêm

Ví dụ:

- `/models?stage=production`
- `/runs?page=2&page_size=50`
- `/metrics?window=5m`

### 18.3 Body

Dùng khi:

- dữ liệu lớn
- dữ liệu có cấu trúc phức tạp
- thao tác tạo/cập nhật
- input inference

Ví dụ:

```json
{
  "instances": [...],
  "threshold": 0.7
}
```

---

## 19) Ví dụ thực tế với FastAPI dành cho dân AI

### 19.1 API đơn giản

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class PredictRequest(BaseModel):
    text: str

@app.get("/health")
def health():
    return {"status": "ok"}

@app.post("/predict")
def predict(req: PredictRequest):
    score = 0.92
    label = "positive" if score > 0.5 else "negative"
    return {"label": label, "score": score}

@app.put("/deployments/{deployment_id}")
def replace_deployment(deployment_id: str, config: dict):
    return {"deployment_id": deployment_id, "mode": "full_replace", "config": config}

@app.patch("/deployments/{deployment_id}")
def patch_deployment(deployment_id: str, patch: dict):
    return {"deployment_id": deployment_id, "mode": "partial_update", "patch": patch}
```

### 19.2 Tư duy

- `/health` dùng `GET` vì chỉ đọc trạng thái
- `/predict` dùng `POST` vì gửi input để xử lý
- `/deployments/{id}` với `PUT` dùng khi thay full config
- `/deployments/{id}` với `PATCH` dùng khi sửa một phần config

---

## 20) Ví dụ curl để nhìn trực tiếp

### 20.1 GET

```bash
curl -X GET https://api.company.com/health
```

### 20.2 POST

```bash
curl -X POST https://api.company.com/predict \
  -H "Content-Type: application/json" \
  -d '{"text":"sản phẩm tốt"}'
```

### 20.3 PUT

```bash
curl -X PUT https://api.company.com/deployments/reco-v1 \
  -H "Content-Type: application/json" \
  -d '{"replicas":4,"cpu":"2","memory":"4Gi"}'
```

### 20.4 PATCH

```bash
curl -X PATCH https://api.company.com/deployments/reco-v1 \
  -H "Content-Type: application/json" \
  -d '{"replicas":6}'
```

---

## 21) Sai lầm phổ biến của người mới

### Sai lầm 1: tưởng GET/POST chỉ là “cách gọi cho vui”
Không phải. Chúng mang **ngữ nghĩa** và ảnh hưởng đến:

- cache
- retry
- audit
- semantics hệ thống
- tài liệu API
- cách người khác hiểu API

### Sai lầm 2: dùng GET cho thao tác có side effect
Ví dụ:

```http
GET /run-training-now
```

Đây là thiết kế kém, vì `GET` đáng lẽ dùng để đọc.  
Nên đổi thành:

```http
POST /training-jobs
```

### Sai lầm 3: dùng POST cho mọi thứ
Nhiều người mới thấy POST tiện nên dùng POST cho tất cả.  
Kết quả là API khó hiểu, khó maintain, khó retry đúng cách.

### Sai lầm 4: nhét dữ liệu lớn vào query string
Input inference, config lớn, metadata phức tạp nên đưa vào body JSON, không nên dồn vào URL.

### Sai lầm 5: không hiểu 401 và 403 khác nhau
- `401`: chưa được xác thực
- `403`: đã xác thực nhưng không đủ quyền

### Sai lầm 6: nghĩ HTTPS chỉ cần cho web public
Sai. Hệ thống nội bộ cũng nên mã hóa, nhất là khi có:

- token
- dữ liệu người dùng
- model quan trọng
- metadata hạ tầng

---

## 22) Mapping sang các tình huống AI / DS / MLOps

### 22.1 Model inference service

- `GET /health`
- `GET /version`
- `POST /predict`
- `POST /embeddings`

### 22.2 MLflow-like tracking service

- `GET /experiments`
- `POST /runs`
- `PATCH /runs/{id}` để update status
- `POST /artifacts/upload`

### 22.3 Model registry

- `GET /models`
- `GET /models/{id}`
- `POST /models`
- `PATCH /models/{id}` để sửa metadata
- `PUT /models/{id}/config` nếu thay toàn bộ config

### 22.4 Serving deployment controller

- `GET /deployments`
- `POST /deployments`
- `PATCH /deployments/{id}` để scale replicas
- `PUT /deployments/{id}` để thay full spec

### 22.5 Data pipeline service

- `POST /jobs`
- `GET /jobs/{id}`
- `PATCH /jobs/{id}` để đổi priority hoặc status

---

## 23) Docker, reverse proxy, gateway liên quan thế nào?

Giả sử có hệ thống:

- container A: FastAPI model server chạy port 8000
- container B: Nginx reverse proxy chạy port 443
- bên ngoài gọi HTTPS đến Nginx
- Nginx forward HTTP nội bộ sang FastAPI

Luồng:

1. client gửi `HTTPS request`
2. reverse proxy terminate TLS
3. proxy chuyển tiếp request vào app
4. app trả response
5. proxy gửi response ngược ra ngoài

Trong môi trường production, model server thường không tự xử lý TLS trực tiếp. TLS thường do:

- Nginx
- Traefik
- Envoy
- cloud load balancer

---

## 24) Phân tích một request end-to-end trong hệ thống AI

Ví dụ user gọi API sentiment:

```bash
curl -X POST https://ml.company.com/predict \
  -H "Authorization: Bearer xyz" \
  -H "Content-Type: application/json" \
  -d '{"text":"dịch vụ rất tốt"}'
```

### Điều gì xảy ra?

1. DNS phân giải `ml.company.com`
2. client kết nối TLS đến server
3. HTTPS handshake hoàn tất
4. request `POST /predict` được gửi
5. gateway kiểm tra auth
6. reverse proxy route request đến đúng service
7. app parse JSON body
8. app chạy model
9. app trả `200 OK` + JSON prediction
10. gateway / proxy trả response ra ngoài

Nếu lỗi có thể xảy ra ở bất kỳ lớp nào:

- DNS
- TLS certificate
- auth token
- routing
- app validation
- model runtime
- timeout
- upstream overload

Người làm MLOps phải biết tách lớp để debug.

---

## 25) Checklist debug khi API lỗi
5
### Nếu 4xx
Nghĩ trước về phía client:

- sai endpoint?
- sai method?
- thiếu header?
- sai token?
- JSON body sai schema?
- thiếu field bắt buộc?
- query param sai?

### Nếu 5xx
Nghĩ thêm về phía server:

- app crash?
- model chưa load xong?
- hết RAM / VRAM?
- downstream timeout?
- lỗi DB / Redis / artifact store?
- config sai?
- network nội bộ lỗi?

### Nếu chỉ lỗi với HTTPS
Kiểm tra:

- cert hợp lệ không?
- host name có match không?
- proxy có terminate TLS đúng không?
- có gọi nhầm `http://` sang cổng HTTPS không?

---

## 26) Một số câu hỏi luyện tư duy
6
### Câu 1
`/health` nên dùng method nào? Vì sao?

**Gợi ý:** thao tác đọc trạng thái, không nên gây side effect.

### Câu 2
Endpoint nhận input văn bản dài để suy luận sentiment nên dùng `GET` hay `POST`? Vì sao?

**Gợi ý:** dữ liệu dài, có body JSON, có xử lý trên server.

### Câu 3
Scale deployment từ 2 replicas lên 4 replicas. Dùng `PUT` hay `PATCH`?

**Gợi ý:** nếu chỉ sửa 1 field, `PATCH` thường hợp lý hơn.

### Câu 4
Thay toàn bộ spec của deployment bằng một config mới đầy đủ. Dùng gì?

**Gợi ý:** `PUT`.

### Câu 5
Vì sao không nên gửi bearer token qua HTTP thuần?

**Gợi ý:** bị sniff, mất confidentiality.

### Câu 6
`401` và `403` khác nhau thế nào?

### Câu 7
Một request timeout nhưng server có thể đã xử lý xong. Vì sao idempotency lại quan trọng trong retry?

---

## 27) Bài tập thực hành nhỏ
7
### Bài 1: Nhận diện method
Cho các hành động sau, chọn `GET`, `POST`, `PUT`, hoặc `PATCH`:

1. lấy danh sách model
2. gửi dữ liệu ảnh để predict
3. thay toàn bộ config deployment
4. đổi `replicas` từ 2 thành 5
5. lấy metrics Prometheus-style
6. tạo training job mới

### Bài 2: Thiết kế API mini
Thiết kế API cho một service quản lý model với các chức năng:

- tạo model
- xem danh sách model
- xem chi tiết model
- cập nhật mô tả model
- thay full config model serving
- gửi input để dự đoán

Viết ra endpoint + method phù hợp.

### Bài 3: Debug mindset
Một request `POST /predict` trả `422 Unprocessable Entity`.  
Liệt kê ít nhất 5 nguyên nhân có thể.

### Bài 4: HTTPS reasoning
Nêu 3 rủi ro cụ thể nếu gọi inference API nội bộ bằng HTTP thay vì HTTPS.

---

## 28) Tóm tắt cực ngắn để nhớ

- **HTTP**: giao thức trao đổi dữ liệu giữa client và server
- **HTTPS**: HTTP được mã hóa bằng TLS
- **Request**: thứ client gửi đi
- **Response**: thứ server trả về
- **GET**: lấy dữ liệu
- **POST**: tạo mới hoặc gửi dữ liệu để xử lý
- **PUT**: thay toàn bộ tài nguyên
- **PATCH**: cập nhật một phần tài nguyên

Một cách nhớ thực dụng cho AI/MLOps:

- health, metrics, metadata → thường `GET`
- predict, submit job, create run → thường `POST`
- replace full config/spec → thường `PUT`
- scale replica, đổi trạng thái, sửa metadata → thường `PATCH`

---

## 29) Kết luận
Người làm AI/DS/MLOps không nhất thiết phải là backend engineer chuyên sâu, nhưng phải hiểu **ngữ nghĩa giao tiếp của hệ thống**. HTTP/HTTPS, request/response, GET/POST/PUT/PATCH là phần cốt lõi của ngôn ngữ đó.

Hiểu chắc phần này giúp:

- đọc tài liệu API nhanh hơn
- debug chính xác hơn
- thiết kế service sạch hơn
- vận hành hệ thống model an toàn hơn
- trao đổi với backend / platform / DevOps hiệu quả hơn

Khi chuyển từ notebook sang production, đây là một trong những lớp kiến thức cần nắm sớm.


