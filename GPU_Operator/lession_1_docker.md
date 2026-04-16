# Docker: Hướng dẫn chi tiết từ nền tảng đến thực hành

## 1. Mục tiêu của tài liệu

Tài liệu này cung cấp hiểu biết toàn diện về **Docker**, bao gồm:

- Docker là gì, giải quyết bài toán gì.
- Các thành phần cốt lõi của Docker.
- Cách image, container, volume, network vận hành.
- Cách viết `Dockerfile` đúng tư duy kỹ thuật.
- Các best practice khi phát triển, kiểm thử và triển khai.
- Những lỗi tư duy phổ biến khi mới học container.

Tài liệu được viết theo hướng **giải thích bản chất**, không chỉ liệt kê lệnh.

---

## 2. Docker là gì?

Docker là một nền tảng để **đóng gói, phân phối và chạy ứng dụng trong môi trường cô lập** gọi là **container**.

Nói ngắn gọn:

- Ứng dụng cần code.
- Code cần runtime.
- Runtime cần thư viện, cấu hình, biến môi trường, file hệ thống.
- Docker cho phép đóng gói toàn bộ các thành phần đó thành một đơn vị có thể chạy lặp lại tương đối nhất quán trên nhiều máy khác nhau.

### 2.1. Bài toán Docker giải quyết

Trước Docker, một ứng dụng rất hay gặp các vấn đề:

- Máy dev chạy được nhưng máy test không chạy được.
- Cùng một project nhưng khác phiên bản Python, CUDA, libc, OpenSSL, Node, Java.
- Nhiều ứng dụng trên cùng một server xung đột dependency.
- Việc tái tạo môi trường cho người mới trong team mất nhiều thời gian.
- Việc scale nhiều instance của cùng một service rất khó quản lý nếu cài trực tiếp lên host.

Docker giảm mạnh các vấn đề đó bằng cách chuẩn hóa môi trường chạy theo **image** và **container**.

---

## 3. Container khác gì máy ảo?

Đây là điểm nền tảng cần nắm rất chắc.

### 3.1. Máy ảo (Virtual Machine)

Máy ảo mô phỏng một máy tính hoàn chỉnh:

- có kernel riêng,
- có hệ điều hành riêng,
- có tài nguyên được cấp phát riêng ở mức lớn hơn.

Ưu điểm:

- cô lập mạnh,
- phù hợp multi-tenant ở mức hạ tầng,
- linh hoạt OS.

Nhược điểm:

- nặng,
- khởi động chậm,
- tốn RAM/CPU/storage hơn.

### 3.2. Container

Container **không phải là máy ảo đầy đủ**. Container thường **chia sẻ kernel của host OS** và chỉ cô lập tiến trình, filesystem, network, namespace, cgroup. Bản chất 1 Container là 1 process, hay còn gọi là 1 tiến trình.

Ưu điểm:

- nhẹ hơn VM,
- khởi động rất nhanh,
- phù hợp microservice, CI/CD, reproducible environment,
- dễ scale số lượng lớn.

Nhược điểm:

- mức cô lập không giống VM,
- phụ thuộc kernel host,
- cần hiểu rõ networking, volumes, permissions để tránh lỗi vận hành.

### 3.3. So sánh trực giác

- **VM**: giống như xây nhiều căn hộ, mỗi căn có hệ thống điện nước riêng.
- **Container**: giống như chia các phòng chức năng trong cùng một tòa nhà, vẫn dùng chung hạ tầng nền nhưng tách biệt đủ để vận hành độc lập.

---

## 4. Docker hoạt động như thế nào?

Muốn dùng Docker tốt, cần hiểu mô hình thành phần.

### 4.1. Docker Engine

Docker Engine là runtime và bộ công cụ chính để tạo, chạy và quản lý container.

Các thành phần logic chính:

- **Docker CLI**: giao diện dòng lệnh, ví dụ `docker build`, `docker run`, `docker ps`.
- **Docker daemon (`dockerd`)**: tiến trình nền thực hiện build image, create container, manage network, volume.
- **Docker API**: lớp giao tiếp giữa CLI và daemon.
- **container runtime**: phần chịu trách nhiệm thực thi container.

### 4.2. Docker Desktop và Docker Engine

- **Docker Engine** thường dùng trên server Linux.
- **Docker Desktop** là bộ công cụ đóng gói sẵn cho máy cá nhân, hỗ trợ development trên Windows/macOS/Linux, thường đã kèm Docker Engine, CLI và Compose.

### 4.3. Các object chính trong Docker

#### Image
Image là **bản mẫu bất biến** để tạo container. 

Nó chứa:

- base OS layer tối giản,
- runtime,
- dependencies,
- code ứng dụng,
- một số metadata khởi chạy.

#### Container
Container là **instance đang chạy** của image. Có thể hiểu thì Image giống như khuôn gói bánh, còn Container là những chiếc bánh được làm theo kích thước của Image, tuy cùng 1 hình dạng nhưng có thể có nhiều loại nhân bánh khác nhau.

Một image có thể tạo ra nhiều container.

Về mặt bộ nhớ, Image được lưu trong ổ đĩa, còn Container sẽ được lưu trong RAM (tương tự 1 tiến trình).

Ví dụ:

- image: `python:3.11-slim`
- container A chạy API
- container B chạy worker

#### Volume
Volume dùng để **lưu dữ liệu bền vững**, không mất khi container bị xóa.

Dùng cho:

- database data,
- model cache,
- artifact,
- log cần giữ lâu hơn vòng đời container.

#### Network
Network dùng để kết nối các container với nhau hoặc với bên ngoài.

Docker tạo network để:

- container gọi nhau qua service name,
- publish port ra host,
- cô lập lưu lượng giữa các nhóm ứng dụng.

#### Registry
Registry là nơi lưu image.

Ví dụ:

- Docker Hub,
- GitHub Container Registry,
- Harbor,
- registry nội bộ.

---

## 5. Image và layer: cơ chế rất quan trọng

Docker image thường có cấu trúc **layered filesystem**.

Ví dụ một image Python app có thể gồm:

1. base layer từ Debian/Alpine,
2. layer cài Python runtime,
3. layer cài pip package,
4. layer copy source code,
5. layer cấu hình command mặc định.

### 5.1. Tại sao layer quan trọng?

Vì layer giúp:

- cache build,
- tiết kiệm thời gian rebuild,
- giảm băng thông khi pull/push,
- tái sử dụng các phần giống nhau giữa nhiều image.

### 5.2. Hệ quả thực tế

Nếu `requirements.txt` không đổi, nhưng source code đổi, Docker có thể chỉ rebuild các layer cuối thay vì build lại từ đầu.

Đây là lý do tại sao thứ tự trong `Dockerfile` ảnh hưởng mạnh đến tốc độ build.

---

## 6. Vòng đời cơ bản của Docker

Luồng chuẩn thường là:

1. Viết `Dockerfile`
2. Build image
3. Chạy container từ image
4. Quan sát logs / exec vào container
5. Dừng, xóa hoặc tái tạo container
6. Push image lên registry nếu cần deploy

### 6.1. Một số lệnh cốt lõi

```bash
docker version
docker info
docker images
docker ps
docker ps -a
docker build -t myapp:1.0 .
docker run -d --name myapp -p 8000:8000 myapp:1.0
docker logs -f myapp
docker exec -it myapp /bin/sh
docker stop myapp
docker rm myapp
docker rmi myapp:1.0
```

### 6.2. Ý nghĩa thực tế

- `docker build`: tạo image từ source.
- `docker run`: tạo và khởi động container.
- `docker ps`: xem container đang chạy.
- `docker logs`: đọc stdout/stderr của process chính.
- `docker exec`: vào trong container để debug.
- `docker stop`: dừng process chính.
- `docker rm`: xóa container.
- `docker rmi`: xóa image.

---

## 7. Dockerfile: tư duy đúng khi viết

`Dockerfile` là file mô tả cách build image.

### 7.1. Ví dụ tối giản cho Python API

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["python", "app.py"]
```

### 7.2. Phân tích từng dòng

#### `FROM`
Chọn image nền.

```dockerfile
FROM python:3.11-slim
```

Nghĩa là build image này dựa trên image Python 3.11 bản slim.

#### `WORKDIR`
Đặt thư mục làm việc mặc định trong container.

```dockerfile
WORKDIR /app
```

Sau đó các lệnh `COPY`, `RUN`, `CMD` sẽ mặc định làm việc từ đây.

#### `COPY`
Sao chép file từ host vào image.

```dockerfile
COPY requirements.txt .
```

Dòng này copy file `requirements.txt` vào `/app/requirements.txt`.

#### `RUN`
Chạy lệnh trong lúc build image.

```dockerfile
RUN pip install --no-cache-dir -r requirements.txt
```

Lệnh này được thực thi ở build-time, không phải runtime.

#### `EXPOSE`
Khai báo cổng mà ứng dụng dự định sử dụng.

```dockerfile
EXPOSE 8000
```

Đây chủ yếu là metadata; muốn truy cập từ host vẫn cần publish port khi chạy container.

#### `CMD`
Command mặc định khi container start.

```dockerfile
CMD ["python", "app.py"]
```

### 7.3. `RUN` khác `CMD` như thế nào?

- `RUN`: chạy khi **build image**.
- `CMD`: chạy khi **container start**.

Ví dụ:

- `RUN apt-get install -y curl` là cài package vào image.
- `CMD ["python", "app.py"]` là lệnh khởi động app.

### 7.4. `ENTRYPOINT` khác gì `CMD`?

- `ENTRYPOINT` dùng khi muốn container luôn chạy như một executable cố định.
- `CMD` thường dùng để cung cấp command mặc định hoặc default arguments.

Ví dụ:

```dockerfile
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8000"]
```

Khi đó container mặc định chạy:

```bash
python app.py --port 8000
```

### 7.5. `COPY` và `ADD`

Thông thường nên ưu tiên `COPY`.

`ADD` có thêm khả năng như tự giải nén một số archive cục bộ hoặc lấy file từ URL, nhưng chính vì nhiều chức năng hơn nên dễ làm hành vi build khó đoán hơn. Với phần lớn use case tiêu chuẩn, `COPY` rõ ràng và an toàn hơn.

### 7.6. `ARG` và `ENV`

#### `ARG`
Biến chỉ tồn tại trong quá trình build.

```dockerfile
ARG PYPI_INDEX_URL
```

#### `ENV`
Biến tồn tại trong image/container runtime.

```dockerfile
ENV APP_ENV=production
```

### 7.7. `USER`
Rất quan trọng cho bảo mật. Không nên mặc định chạy process bằng root nếu không cần.

```dockerfile
RUN useradd -m appuser
USER appuser
```

### 7.8. `HEALTHCHECK`
Dùng để Docker biết service còn sống hay không.

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1
```

---

## 8. Multi-stage build: kỹ thuật rất nên dùng

Một sai lầm phổ biến là nhét mọi thứ vào một image lớn, gồm compiler, build tools, source và artifact cuối cùng. Điều này làm image nặng, dễ lộ bề mặt tấn công và chậm deploy.

### 8.1. Ý tưởng

Dùng một stage để build, một stage khác để chạy.

### 8.2. Ví dụ

```dockerfile
FROM python:3.11 AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip wheel --wheel-dir=/wheels -r requirements.txt

FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /wheels /wheels
RUN pip install --no-cache-dir /wheels/*
COPY . .
CMD ["python", "app.py"]
```

### 8.3. Lợi ích

- image runtime nhỏ hơn,
- ít tool dư thừa hơn,
- bảo mật tốt hơn,
- thời gian pull/push giảm.

---

## 9. `.dockerignore`: file nhỏ nhưng tác động lớn

`.dockerignore` giúp loại các file không cần thiết khỏi build context.

Ví dụ:

```gitignore
.git
__pycache__/
*.pyc
.env
venv/
node_modules/
data/
checkpoints/
logs/
.ipynb_checkpoints/
```

### 9.1. Vì sao quan trọng?

Nếu không có `.dockerignore`:

- build context phình to,
- build chậm,
- dễ lộ file nhạy cảm,
- cache layer kém hiệu quả.

---

## 10. Persist dữ liệu: bind mount và volume

Đây là điểm rất hay gây nhầm lẫn.

### 10.1. Bind mount

Bind mount map trực tiếp thư mục từ host vào container.

Ví dụ:

```bash
docker run -v $(pwd):/app myapp
```

Ưu điểm:

- tiện cho development,
- sửa code trên host và container thấy ngay.

Nhược điểm:

- phụ thuộc cấu trúc host,
- kém portable hơn,
- dễ sinh lỗi permission.

### 10.2. Named volume

Named volume do Docker quản lý.

Ví dụ:

```bash
docker volume create pgdata
docker run -v pgdata:/var/lib/postgresql/data postgres:16
```

Ưu điểm:

- bền vững,
- phù hợp dữ liệu database,
- ít phụ thuộc layout thư mục host.

### 10.3. Khi nào dùng cái nào?

- **Dev source code**: thường dùng bind mount.
- **Database/data quan trọng**: ưu tiên named volume.
- **Artifact chia sẻ giữa container**: tùy use case, nhưng named volume thường sạch hơn.

---

## 11. Docker networking: hiểu đúng để tránh lỗi kết nối

### 11.1. Bridge network mặc định

Khi chạy container thông thường, Docker thường dùng bridge network.

Container trên cùng một network có thể giao tiếp với nhau.

### 11.2. Publish port

Ví dụ:

```bash
docker run -p 8080:80 nginx
```

Ý nghĩa:

- host port `8080`
- map vào container port `80`

Khi đó truy cập `localhost:8080` trên host sẽ đi vào service port `80` trong container.

### 11.3. `localhost` bên trong container là gì?

Đây là lỗi tư duy cực phổ biến.

- `localhost` trong **host** là host.
- `localhost` trong **container A** là chính container A.
- container A không thể dùng `localhost` để gọi container B.

Muốn A gọi B, cần dùng:

- cùng network,
- tên service/container hoặc DNS phù hợp,
- đúng port nội bộ.

### 11.4. Ví dụ thực tế

Nếu có:

- service `api`
- service `db`

Thì trong container `api`, chuỗi kết nối DB thường là:

```text
postgresql://user:pass@db:5432/appdb
```

không phải:

```text
postgresql://user:pass@localhost:5432/appdb
```

---

## 12. Những lỗi phổ biến khi mới dùng Docker

### 12.1. App chỉ bind `127.0.0.1`

Nếu app trong container chỉ listen `127.0.0.1`, host bên ngoài có thể không truy cập được dù đã map port. Thường phải bind `0.0.0.0`.

### 12.2. Dùng `localhost` sai ngữ cảnh

Đã nói ở trên: `localhost` trong container không phải host hay container khác.

### 12.3. Mất dữ liệu database

Nguyên nhân thường là:

- không mount volume,
- xóa container và tưởng dữ liệu nằm trong host,
- dùng `docker compose down -v` mà không để ý.

### 12.4. Image build rất chậm

Thường do:

- build context quá lớn,
- không có `.dockerignore`,
- copy source trước rồi mới cài dependency,
- cache layer bị vô hiệu hóa liên tục.

### 12.5. Permission lỗi khi mount file từ host

Đặc biệt dễ gặp trên Linux khi UID/GID giữa host và container không khớp.

### 12.6. Container chạy nhưng service lỗi

Container "running" không đồng nghĩa service "ready". Đây là lý do healthcheck rất quan trọng.

---

## 13. Best practices khi viết Dockerfile

### 13.1. Chọn base image nhỏ nhưng hợp lý

- `slim` thường là điểm cân bằng tốt.
- Không nên cực đoan chọn image quá nhỏ nếu làm debugging, build native package hoặc compatibility trở nên khó khăn.

### 13.2. Tối ưu cache build

Thường nên:

1. copy file dependency trước,
2. install dependency,
3. sau đó mới copy source code.

### 13.3. Không nhét secret vào image

Không hard-code:

- API keys,
- password,
- token,
- private cert.

### 13.4. Không chạy bằng root nếu không cần

Giảm rủi ro bảo mật.

### 13.5. Dùng multi-stage build

Đặc biệt tốt cho:

- Go,
- Node frontend build,
- Java,
- Python có build artifact,
- C/C++ extension.

### 13.6. Ghi rõ version quan trọng

Ví dụ:

- runtime version,
- package manager behavior,
- major dependency.

Việc này giúp reproducibility cao hơn.

---

## 14. Checklist học Docker theo thứ tự hợp lý

### Giai đoạn 1: nền tảng

- image là gì,
- container là gì,
- khác VM như thế nào,
- build vs run,
- volume vs bind mount,
- port mapping,
- network cơ bản.

### Giai đoạn 2: thực hành Dockerfile

- viết Dockerfile cho app đơn giản,
- tối ưu layer cache,
- thêm `.dockerignore`,
- dùng non-root user,
- hiểu `CMD` và `ENTRYPOINT`.

---

## 15. Tóm tắt ngắn gọn bản chất Docker

### Docker

Docker là công cụ chuẩn hóa môi trường chạy ứng dụng bằng container.

### Dockerfile

Dockerfile là công thức build image.

### Image

Image là template bất biến.

### Container

Container là instance đang chạy của image.

### Volume

Volume giữ dữ liệu bền vững.

### Network

Network cho container giao tiếp đúng topology.

---

## 16. Nên nhớ điều gì quan trọng nhất?

1. Docker không phải chỉ là vài lệnh CLI; bản chất là chuẩn hóa môi trường chạy.
2. Container không phải VM; nó nhẹ hơn vì chia sẻ kernel host.
3. Muốn dùng Docker tốt phải hiểu image, layer, volume, network.
4. Với team AI/ML, Docker là bước nền gần như bắt buộc.

---
