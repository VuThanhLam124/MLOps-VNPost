# Bộ bài toán / tình huống thực hành Docker và Docker Compose cho người mới

Tài liệu này dành cho người mới bắt đầu học Docker/Docker Compose theo hướng **luyện tư duy hệ thống**, không chỉ học cú pháp lệnh. Mục tiêu là giúp người học trả lời được các câu hỏi cốt lõi:

- Khi nào cần container?
- Image và container khác nhau ở đâu?
- Vì sao ứng dụng chạy được trên máy này nhưng hỏng trên máy khác?
- Dữ liệu nằm ở đâu, mất khi nào, giữ khi nào?
- Khi có nhiều service, chúng nói chuyện với nhau như thế nào?
- Khi hệ thống lỗi, nên debug theo thứ tự nào?

---

# 1. Cách dùng bộ bài tập này

Mỗi bài nên được làm theo cùng một quy trình:

1. **Đọc bài toán**: hiểu nhu cầu nghiệp vụ.
2. **Phân tách thành thành phần**: app, database, cấu hình, dữ liệu, port, network, volume.
3. **Xác định ranh giới container**: cái gì nằm trong image, cái gì nằm ngoài.
4. **Viết giả thuyết trước khi làm**: ví dụ “nếu rebuild image thì code mới vào container”, “nếu không mount volume thì dữ liệu DB sẽ mất”.
5. **Triển khai**.
6. **Quan sát**: container status, logs, network, filesystem, exit code.
7. **Giải thích lại bằng lời**: tại sao chạy được / tại sao lỗi.

Người mới không nên chỉ chạy lệnh cho xong. Mỗi bài nên trả lời thêm ba câu:

- **Hệ thống đang có những thành phần nào?**
- **Dữ liệu di chuyển như thế nào?**
- **State nằm ở đâu?**

---

# 2. Các năng lực cần luyện

Bộ bài tập được thiết kế để luyện 6 nhóm tư duy:

## 2.1. Tư duy đóng gói môi trường
Hiểu rằng container giúp đóng gói:

- runtime
- dependency
- file hệ thống cần thiết
- entrypoint/cách khởi động

Nhưng **không tự động** giải quyết:

- dữ liệu bền vững
- cấu hình bí mật
- scale hệ thống
- phân phối tài nguyên GPU

## 2.2. Tư duy image vs container
Phải phân biệt rõ:

- **Image**: bản mẫu bất biến để tạo container
- **Container**: instance đang chạy từ image

Sai lầm phổ biến của người mới là sửa file trong container rồi nghĩ thay đổi đó sẽ tồn tại mãi.

## 2.3. Tư duy stateful vs stateless
Người mới cần học cách hỏi:

- service này có dữ liệu cần giữ lâu dài không?
- nếu container chết thì dữ liệu mất không?
- cần volume không?

## 2.4. Tư duy network service-to-service
Không phải service nào cũng dùng `localhost` để nói chuyện với nhau.

- từ máy host vào container: có thể dùng `localhost:PORT`
- giữa các container trong cùng compose network: dùng **service name**

## 2.5. Tư duy build-time vs run-time
Một số thứ thuộc lúc build:

- cài package
- copy source
- tạo artifact

Một số thứ thuộc lúc chạy:

- biến môi trường
- volume mount
- port mapping
- secrets/config

## 2.6. Tư duy debug theo lớp
Khi lỗi, debug theo thứ tự:

1. Container có chạy không?
2. Process chính có sống không?
3. Logs nói gì?
4. Port có expose/map đúng không?
5. App có bind `0.0.0.0` hay chỉ `127.0.0.1`?
6. Service phụ thuộc (DB, Redis) có sẵn chưa?
7. Network resolution có đúng không?
8. Volume có mount đè code/config không?

---

# 3. Bộ bài tập theo cấp độ

---

# Cấp độ 1 — Làm quen với container

## Bài 1. Chạy container đầu tiên và giải thích được chuyện gì đã xảy ra

### Mục tiêu
Hiểu dòng chảy cơ bản:

- pull image
- create container
- start process
- process kết thúc thì container dừng

### Bài toán
Chạy một container `nginx`, sau đó chạy một container `ubuntu` để thử vào shell.

### Yêu cầu
- Chạy `nginx`
- Kiểm tra container đang chạy
- Truy cập web từ trình duyệt hoặc `curl`
- Chạy một container Ubuntu tương tác
- Giải thích vì sao Ubuntu container thoát nếu không có process foreground phù hợp

### Câu hỏi tư duy
- Vì sao `nginx` chạy nền được còn `ubuntu` thường thoát ngay?
- Process chính của container là gì?
- Nếu process PID 1 chết thì chuyện gì xảy ra?

### Kỹ năng luyện được
- `docker run`
- `docker ps`
- `docker logs`
- hiểu lifecycle cơ bản

---

## Bài 2. Image khác container ở đâu?

### Bài toán
Tạo container từ image `python:3.11-slim`, vào bên trong cài tạm một package bằng `pip`. Sau đó xóa container và tạo lại container mới từ cùng image.

### Mục tiêu
Tự chứng minh rằng thay đổi trong container không đồng nghĩa thay đổi image gốc.

### Câu hỏi tư duy
- Vì sao container mới không có package vừa cài?
- Khi nào cần `Dockerfile` thay vì sửa tay trong container?
- Khi nào `docker commit` là giải pháp xấu?

### Kết luận người học cần tự rút ra
- Image là nguồn chân lý cho tái lập môi trường.
- Chỉnh tay trong container không phải workflow chuẩn.

---

## Bài 3. Port mapping và nhầm lẫn `localhost`

### Bài toán
Chạy một web app đơn giản trong container trên cổng 8000, map ra host cổng 8080.

### Yêu cầu
- App phải nghe ở `0.0.0.0:8000`
- Host truy cập ở `localhost:8080`

### Tình huống lỗi để tự thử
- Cho app bind vào `127.0.0.1:8000` bên trong container
- Quan sát vì sao host không truy cập được

### Câu hỏi tư duy
- `127.0.0.1` bên trong container là của ai?
- Port mapping thực ra đang nối cái gì với cái gì?

---

# Cấp độ 2 — Tự viết Dockerfile

## Bài 4. Dockerize một script Python tối giản

### Bài toán
Có một file `app.py` in ra “Hello Docker” và đọc biến môi trường `APP_ENV`.

### Yêu cầu
- Viết `Dockerfile`
- Build image
- Run container với biến môi trường khác nhau

### Điều cần giải thích
- Vì sao `COPY` cần trước hay sau `pip install` sẽ ảnh hưởng cache?
- `CMD` để làm gì?
- Khác gì với `RUN`?

### Mục tiêu tư duy
Phân biệt:

- `RUN`: chạy lúc build image
- `CMD`: lệnh mặc định lúc container start

---

## Bài 5. Dockerize một API Flask/FastAPI

### Bài toán
Đóng gói một API trả về JSON.

### Yêu cầu
- Chạy được trên container
- Truy cập được từ host
- Có health endpoint `/health`

### Điểm cần luyện
- cấu trúc thư mục gọn
- requirements file
- expose port hợp lý
- chạy bằng production server phù hợp nếu cần

### Câu hỏi tư duy
- Vì sao không nên dùng Flask development server cho production?
- `EXPOSE` có thực sự publish port ra ngoài không?

---

## Bài 6. Tối ưu Dockerfile với layer cache

### Bài toán
So sánh hai Dockerfile:

- bản A: `COPY . .` rồi mới `pip install`
- bản B: `COPY requirements.txt` trước, cài package, sau đó mới `COPY . .`

### Mục tiêu
Hiểu cache build giúp tiết kiệm thời gian như thế nào.

### Câu hỏi tư duy
- Vì sao sửa 1 dòng code không nên làm invalid layer cài dependencies?
- Docker build cache là cache của cái gì?

---

## Bài 7. Multi-stage build

### Bài toán
Đóng gói một app Node.js hoặc Go/Python có bước build artifact, sao cho image cuối nhẹ hơn.

### Yêu cầu
- stage 1 để build
- stage 2 chỉ chứa artifact/runtime cần thiết

### Mục tiêu tư duy
Hiểu rằng image runtime không nhất thiết phải chứa compiler, build tool, test file.

### Câu hỏi tư duy
- Vì sao image nhỏ hơn thường tốt hơn?
- Có phải image càng nhỏ luôn càng tốt không?

---

# Cấp độ 3 — Dữ liệu và cấu hình

## Bài 8. Volume: dữ liệu có mất không?

### Bài toán
Chạy PostgreSQL hoặc MySQL trong container.

### Thực hiện 2 lần

#### Trường hợp A: không mount volume
- tạo database
- thêm dữ liệu
- xóa container
- tạo lại container
- kiểm tra dữ liệu

#### Trường hợp B: có mount volume
- lặp lại quy trình trên
- so sánh kết quả

### Mục tiêu tư duy
Hiểu dữ liệu bền vững nằm ở đâu.

### Câu hỏi tư duy
- Container filesystem khác volume ở đâu?
- Khi nào bind mount, khi nào named volume?

---

## Bài 9. Bind mount cho dev workflow

### Bài toán
Mount source code từ host vào container để sửa code trên máy và app reload ngay.

### Mục tiêu
Hiểu bind mount tiện cho dev nhưng có thể phá tính nhất quán của image runtime.

### Câu hỏi tư duy
- Vì sao compose dev thường dùng bind mount?
- Vì sao production không nên mount source code kiểu này?

---

## Bài 10. ENV, ARG và secrets

### Bài toán
Tạo app cần:

- biến `APP_ENV`
- biến `DB_HOST`
- token bí mật giả lập

### Yêu cầu
- dùng `ARG` cho thông tin build-time
- dùng `ENV` cho runtime config
- không hard-code secret vào image

### Câu hỏi tư duy
- Tại sao không nên nhét password vào `Dockerfile`?
- Secret bị lộ qua image history theo cách nào?
- `.env` tiện nhưng có đủ an toàn không?

---

# Cấp độ 4 — Docker Compose và hệ nhiều service

## Bài 11. App + Database với Docker Compose

### Bài toán
Có một API backend phụ thuộc PostgreSQL.

### Yêu cầu
- Viết `compose.yaml` với 2 service: `app`, `db`
- app kết nối DB bằng service name `db`
- mount volume cho DB
- thêm env phù hợp

### Điều người học phải giải thích được
- vì sao app không nên dùng `localhost` để nối DB
- network nội bộ của compose hoạt động ra sao
- volume cho DB dùng để làm gì

### Câu hỏi tư duy
- Nếu đổi service name từ `db` sang `postgres`, app phải đổi gì?
- Nếu DB chưa sẵn sàng khi app start thì chuyện gì xảy ra?

---

## Bài 12. `depends_on` không phải thuốc tiên

### Bài toán
Dựng hệ `app + db`, bật app ngay khi DB còn chưa ready.

### Mục tiêu
Hiểu rằng `depends_on` chủ yếu điều khiển thứ tự start tương đối, không đảm bảo service thật sự sẵn sàng nhận kết nối.

### Cách luyện
- cố tình cho DB khởi động chậm
- quan sát app fail
- thêm retry logic hoặc healthcheck hợp lý

### Câu hỏi tư duy
- readiness khác startup order ở đâu?
- vấn đề này liên hệ thế nào với Kubernetes readiness probe sau này?

---

## Bài 13. Reverse proxy phía trước nhiều service

### Bài toán
Có 2 service web:

- `frontend`
- `api`

Dùng Nginx container làm reverse proxy để route:

- `/` -> frontend
- `/api` -> api

### Mục tiêu tư duy
Hiểu kiến trúc nhiều container trong một ứng dụng thực tế.

### Câu hỏi tư duy
- Vì sao không expose tất cả service trực tiếp ra host?
- reverse proxy giải quyết những việc gì?

---

## Bài 14. Redis cache trong Compose

### Bài toán
App đọc dữ liệu chậm từ DB, bổ sung Redis để cache.

### Mục tiêu
Bắt đầu học cách thêm 1 service phụ trợ mà không làm hệ thống rối.

### Câu hỏi tư duy
- Redis là stateful hay stateless?
- Nếu Redis mất dữ liệu có chấp nhận được không?
- Điều này ảnh hưởng cách mount volume thế nào?

---

# Cấp độ 5 — Debug và quan sát

## Bài 15. Container cứ restart liên tục

### Bài toán
Tạo một container app lỗi cấu hình khiến process crash ngay sau khi start.

### Mục tiêu
Luyện debug:

- `docker ps -a`
- `docker logs`
- exit code
- env/config check

### Câu hỏi tư duy
- restart loop thường đến từ lớp nào: app, config, network hay permission?
- nên đọc cái gì trước: logs hay sửa bừa?

---

## Bài 16. “Chạy local được, vào container thì hỏng”

### Bài toán
Một app dùng đường dẫn file tương đối, local chạy được nhưng trong container lỗi `file not found`.

### Mục tiêu
Hiểu:

- working directory
- `COPY` path
- filesystem trong image

### Câu hỏi tư duy
- `WORKDIR` thay đổi điều gì?
- Vì sao dependency file hoặc config file hay bị thiếu khi build?

---

## Bài 17. Permission denied trong container

### Bài toán
App cần ghi file vào thư mục mounted từ host nhưng bị lỗi quyền.

### Mục tiêu
Làm quen với:

- `USER`
- uid/gid
- permission giữa host và container

### Câu hỏi tư duy
- Vì sao chạy container bằng root thường tiện nhưng không nên lạm dụng?
- host permission và container permission liên hệ ra sao?

---

## Bài 18. Không truy cập được service dù container đang chạy

### Bài toán
Container chạy bình thường, logs không lỗi, nhưng host không vào được web app.

### Checklist cần tự áp dụng
- app có bind đúng `0.0.0.0` không?
- port map đúng chưa?
- process có thật sự listen cổng đó không?
- firewall/host network có chặn không?

### Mục tiêu
Học cách debug theo tầng thay vì đoán mò.

---

# Cấp độ 6 — Thiết kế hệ thống nhỏ bằng Docker Compose

## Bài 19. Hệ thống blog mini

### Đề bài
Thiết kế stack gồm:

- web app
- PostgreSQL
- Redis
- Nginx

### Nhiệm vụ
- vẽ sơ đồ thành phần
- xác định service nào stateful
- xác định service nào cần volume
- xác định cổng nào public, cổng nào private
- viết compose

### Điều quan trọng
Bài này không chỉ là viết YAML. Bài này luyện tư duy thiết kế.

### Câu hỏi tư duy
- Có cần expose PostgreSQL ra host không?
- Nếu chỉ Nginx public thì lợi ích là gì?
- Nếu sau này scale web app thì nút thắt ở đâu?

---

## Bài 20. Môi trường dev và prod khác nhau thế nào?

### Đề bài
Cùng một ứng dụng, hãy đề xuất 2 cấu hình:

#### Dev
- bind mount source code
- hot reload
- verbose logs
- có thể expose DB để debug

#### Prod
- image immutable
- không mount source code
- chỉ expose reverse proxy
- secret/config tách biệt hơn

### Mục tiêu tư duy
Phân biệt “chạy được” với “vận hành được”.

---

## Bài 21. Triển khai ML inference service tối giản

### Đề bài
Tạo một service Python phục vụ model inference đơn giản, kèm một service Redis hàng đợi hoặc cache.

### Yêu cầu
- Dockerize app
- compose cho nhiều service
- mount thư mục model hoặc build model vào image
- giải thích lựa chọn đó

### Câu hỏi tư duy
- Model file nên nằm trong image hay volume?
- Nếu model rất lớn thì build image có còn hợp lý không?
- Nếu phải dùng GPU sau này thì thiết kế hiện tại cần đổi gì?

---

# 4. Các tình huống tư duy thực chiến

Dưới đây là các case ngắn để người mới tập phân tích trước khi viết lệnh.

## Case A. Researcher làm việc bằng notebook, môi trường thường xuyên vỡ dependency

### Bài toán
Mỗi người cài package khác nhau, server chung bị loạn môi trường.

### Câu hỏi phân tích
- Docker giải quyết phần nào?
- Docker không giải quyết phần nào?
- Có cần volume cho dữ liệu notebook không?
- Có cần Jupyter trong container không?

### Insight nên rút ra
Docker giúp cô lập môi trường runtime, nhưng chưa đủ để quản trị tài nguyên nhiều người dùng. Đây là điểm nối sang JupyterHub/Kubernetes sau này.

---

## Case B. Một web app cần DB và cache

### Câu hỏi phân tích
- service nào phải khởi động trước?
- state nằm ở đâu?
- service nào có thể chết rồi chạy lại mà không sao?
- service nào cần backup?

### Insight
Người học bắt đầu phân loại được:

- stateless compute
- stateful storage
- infra phụ trợ

---

## Case C. Cùng một app, local chạy được nhưng server thì không

### Hãy yêu cầu người học tự hỏi
- local đang có file/config nào mà image không có?
- local đang dùng Python version nào?
- local có mount thư mục mà server không mount không?
- server có port mapping khác không?

### Insight
Container không chỉ là “gói app”, mà là công cụ buộc người học mô tả hệ thống một cách tường minh.

---

## Case D. DB container bị xóa, dữ liệu mất hết

### Câu hỏi phân tích
- lỗi do Docker hay do cách dùng?
- volume đang ở đâu?
- backup có được nghĩ tới chưa?

### Insight
Người học phải hiểu rằng container không đồng nghĩa persistent storage.

---

## Case E. Một AI service chuẩn bị chuyển từ CPU sang GPU

### Câu hỏi phân tích
- Dockerfile có đổi không?
- runtime có cần image base CUDA không?
- host cần chuẩn bị gì?
- vì sao chuyện này sẽ liên quan tới NVIDIA Container Toolkit và sau đó là GPU Operator nếu lên K8s?

### Insight
Docker là tầng đóng gói ứng dụng; quản lý GPU ở quy mô lớn cần thêm tầng orchestration và operator.

---

# 5. Bộ câu hỏi phản xạ để luyện tư duy

Sau mỗi bài, nên tự trả lời:

1. **Container chính chạy process gì?**
2. **Image được build từ đâu, phụ thuộc những layer nào?**
3. **Dữ liệu nào sẽ mất khi xóa container?**
4. **Dữ liệu nào phải được giữ bằng volume?**
5. **Service giao tiếp qua host port hay internal network?**
6. **Thành phần nào thuộc build-time, thành phần nào thuộc run-time?**
7. **Nếu app lỗi, log nên đọc ở đâu trước?**
8. **Nếu đổi máy khác, hệ thống còn tái lập được không?**
9. **Có secret nào đang bị nhét sai chỗ không?**
10. **Có phần nào đang “chạy được nhưng vận hành kém” không?**

---

# 6. Gợi ý mini-project cho repo học tập

## Mini-project 1. Python API + Postgres

### Mục tiêu
Hoàn thành trọn vẹn chu trình:

- Dockerfile cho app
- compose cho app + db
- volume cho db
- env config
- healthcheck

### Kỹ năng thu được
Nền tảng rất tốt trước khi học Kubernetes.

---

## Mini-project 2. Jupyter cho cá nhân bằng Docker

### Mục tiêu
- chạy Jupyter Lab trong container
- mount thư mục notebook
- cố định phiên bản thư viện

### Kỹ năng thu được
Hiểu cách đóng gói môi trường data/ML cá nhân.

---

## Mini-project 3. Inference service có model file

### Mục tiêu
- đóng gói model server
- quyết định model nằm trong image hay volume
- benchmark thời gian build/start ở hai cách

### Kỹ năng thu được
Chuẩn bị tư duy cho MLOps và GPU serving.

---

## Mini-project 4. Full stack tối giản

### Thành phần
- frontend
- backend
- db
- redis
- nginx

### Mục tiêu
Hiểu vì sao Docker Compose rất hợp để học kiến trúc đa service quy mô nhỏ.

---

# 7. Lộ trình học đề xuất cho người mới

## Giai đoạn 1: hiểu Docker căn bản
Làm các bài 1–7.

Mục tiêu:
- hiểu image/container
- biết viết Dockerfile
- biết port, logs, env

## Giai đoạn 2: hiểu dữ liệu và cấu hình
Làm các bài 8–10.

Mục tiêu:
- hiểu volume
- hiểu stateful/stateless
- tránh nhét secret sai chỗ

## Giai đoạn 3: hiểu đa service với Compose
Làm các bài 11–14.

Mục tiêu:
- hiểu internal network
- biết dựng app + db + cache
- hiểu startup vs readiness

## Giai đoạn 4: hiểu debug và thiết kế
Làm các bài 15–21.

Mục tiêu:
- có quy trình debug
- biết đọc lỗi có hệ thống
- bắt đầu có tư duy kiến trúc

---

# 8. Cấu trúc repo markdown gợi ý

```text
repo/
├── 00-roadmap/
│   └── docker-learning-path.md
├── 01-docker/
│   ├── docker-basics.md
│   ├── dockerfile.md
│   ├── image-vs-container.md
│   ├── volumes-and-storage.md
│   ├── networking.md
│   └── debugging.md
├── 02-compose/
│   ├── docker-compose-basics.md
│   ├── app-db-compose.md
│   ├── reverse-proxy-compose.md
│   └── compose-debugging.md
├── 03-exercises/
│   ├── beginner-exercises.md
│   ├── mini-projects.md
│   └── thought-questions.md
├── 04-k8s/
│   └── ...
└── 05-gpu-operator/
    └── ...
```

---

# 9. Tiêu chí đánh giá người mới đã “thật sự hiểu” chưa

Không đánh giá bằng việc nhớ lệnh nhiều, mà bằng việc người học có thể:

- tự giải thích image/container/volume/network bằng lời của mình
- tự suy luận vì sao app không connect được DB
- tự nhận ra khi nào cần rebuild image
- tự phân biệt state nằm trong container hay ngoài container
- tự vẽ sơ đồ một stack nhỏ với app/db/cache/proxy
- tự debug theo checklist thay vì sửa ngẫu nhiên

---

# 10. Kết luận

Người mới học Docker/Docker Compose thường mắc một lỗi lớn: học như học một danh sách lệnh. Cách học hiệu quả hơn là học theo **bài toán hệ thống**:

- ứng dụng gồm những thành phần gì
- mỗi thành phần cần gì để chạy
- dữ liệu nằm ở đâu
- thành phần nào public, thành phần nào private
- lỗi phát sinh ở lớp nào

Khi đã nắm vững các bài tập trên, bước chuyển sang Kubernetes sẽ tự nhiên hơn rất nhiều, vì Kubernetes thực chất vẫn xoay quanh những khái niệm nền tảng:

- containerized workload
- network
- storage
- config/secrets
- health/readiness
- scheduling

Chỉ khác là ở quy mô lớn hơn và có control plane điều phối.
