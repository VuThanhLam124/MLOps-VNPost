# Docker Compose: Hướng dẫn chi tiết từ nền tảng đến thực hành

## 1. Mục tiêu của tài liệu

Tài liệu này cung cấp hiểu biết toàn diện về **Docker Compose**, bao gồm:

- Docker Compose là gì, giải quyết bài toán gì.
- Cách mô hình hóa ứng dụng nhiều service.
- Cách viết `compose.yaml` đúng tư duy kỹ thuật.
- Các command thường dùng và workflow phát triển.
- Cách sử dụng trong bối cảnh MLOps/AI.
- Các best practice khi phát triển, kiểm thử và triển khai.

Tài liệu được viết theo hướng **giải thích bản chất**, không chỉ liệt kê lệnh.

---

## 2. Docker Compose là gì?

Docker Compose dùng để **định nghĩa và chạy ứng dụng nhiều container bằng một file YAML**.

Thay vì chạy từng lệnh `docker run` dài và khó nhớ, có thể mô tả cả stack ứng dụng trong một file như:

- web service,
- database,
- redis,
- volume,
- network,
- env vars,
- healthcheck,
- startup order.

Compose đặc biệt hữu ích cho:

- local development,
- integration testing,
- staging nhỏ,
- demo môi trường,
- tái hiện topology nhiều service.

---

## 3. Tư duy mô hình ứng dụng trong Compose

Compose mô hình hóa ứng dụng quanh vài thực thể chính:

- **services**: các container logic của app
- **networks**: cách các service kết nối
- **volumes**: nơi lưu dữ liệu bền vững
- **configs/secrets**: dữ liệu cấu hình nhạy cảm hoặc file cấu hình

Một project Compose thường tương ứng với **một application stack**.

Ví dụ một hệ thống ML platform nhỏ có thể có:

- `api`
- `worker`
- `postgres`
- `redis`
- `minio`
- `mlflow`

---

## 4. File `compose.yaml` cơ bản

### 4.1. Ví dụ hoàn chỉnh dễ hiểu

```yaml
services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: demo_web
    ports:
      - "8000:8000"
    environment:
      APP_ENV: development
      DB_HOST: db
      DB_PORT: 5432
      DB_NAME: appdb
      DB_USER: appuser
      DB_PASSWORD: secret
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - ./:/app
    command: python app.py
    restart: unless-stopped

  db:
    image: postgres:16
    container_name: demo_db
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

volumes:
  pgdata:
```

---

## 5. Giải thích chi tiết các trường trong Compose

### 5.1. `services`

Mỗi service là một thành phần ứng dụng.

Ví dụ:

- `web`: API server
- `db`: PostgreSQL

Mỗi service có thể được tạo từ:

- `image` có sẵn
- hoặc `build` từ source code

### 5.2. `build`

```yaml
build:
  context: .
  dockerfile: Dockerfile
```

Ý nghĩa:

- lấy source ở thư mục hiện tại làm build context,
- dùng file `Dockerfile` để build image.

### 5.3. `image`

```yaml
image: postgres:16
```

Nghĩa là dùng image có sẵn từ registry.

### 5.4. `ports`

```yaml
ports:
  - "8000:8000"
```

Map `host:container`.

### 5.5. `environment`

```yaml
environment:
  APP_ENV: development
```

Khai báo biến môi trường cho service.

### 5.6. `depends_on`

`depends_on` thể hiện phụ thuộc giữa service. Tuy nhiên cần hiểu đúng:

- nó hỗ trợ thứ tự start ở mức orchestration cơ bản,
- nhưng không thay thế hoàn toàn readiness logic của ứng dụng,
- tốt nhất nên dùng kèm `healthcheck` nếu service A thực sự cần service B sẵn sàng.

### 5.7. `volumes`

```yaml
volumes:
  - pgdata:/var/lib/postgresql/data
```

Named volume `pgdata` được mount vào thư mục data của PostgreSQL.

### 5.8. `command`

Override command mặc định của image.

### 5.9. `restart`

Ví dụ:

- `no`
- `always`
- `on-failure`
- `unless-stopped`

Dùng để định nghĩa hành vi restart khi process chết.

### 5.10. `healthcheck`

Rất hữu ích để đánh dấu service healthy/unhealthy.

---

## 6. Các lệnh Docker Compose cần dùng thường xuyên

### 6.1. Khởi động stack

```bash
docker compose up
```

### 6.2. Chạy nền

```bash
docker compose up -d
```

### 6.3. Build lại image và chạy

```bash
docker compose up -d --build
```

### 6.4. Xem container trong project

```bash
docker compose ps
```

### 6.5. Xem logs

```bash
docker compose logs -f
```

Hoặc theo service:

```bash
docker compose logs -f web
```

### 6.6. Exec vào service

```bash
docker compose exec web /bin/sh
```

### 6.7. Dừng và xóa resource của project

```bash
docker compose down
```

### 6.8. Dừng và xóa cả volume

```bash
docker compose down -v
```

### 6.9. Pull image mới

```bash
docker compose pull
```

### 6.10. Validate file compose

```bash
docker compose config
```

Lệnh này rất hữu ích để debug YAML và xem cấu hình sau khi được resolve.

---

## 7. Development workflow với Compose

Compose rất mạnh trong local dev.

### 7.1. Mô hình điển hình

- source code để trên host,
- mount vào container,
- app trong container chạy autoreload,
- database chạy ở service riêng,
- cache/message broker cũng chạy riêng.

### 7.2. Ví dụ cho FastAPI

```yaml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - ./:/app
    command: uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

Lợi ích:

- môi trường dev gần với production hơn so với chạy trực tiếp trên host,
- không làm bẩn máy local bởi dependency của từng project,
- onboarding người mới nhanh hơn.

---

## 8. Production mindset: Compose dùng được tới đâu?

Compose rất tốt cho:

- dev,
- test,
- demo,
- staging nhỏ,
- các hệ thống nội bộ quy mô vừa và ít thay đổi.

Nhưng Compose **không phải orchestration platform đầy đủ** như Kubernetes.

### 8.1. Compose không mạnh ở đâu?

- tự động scheduling đa node,
- rolling update phức tạp,
- self-healing mạnh ở cấp cluster,
- autoscaling chuẩn production lớn,
- policy và multi-tenant sâu.

### 8.2. Khi nào vẫn dùng Compose cho production?

Có thể dùng nếu hệ thống:

- ít node,
- ít service,
- team nhỏ,
- cần triển khai đơn giản,
- chấp nhận quản trị ở mức host-based.

Ví dụ:

- công cụ nội bộ,
- monitor service đơn giản,
- demo environment,
- PoC hạ tầng.

---

## 9. Ví dụ đầy đủ hơn: web + postgres + redis

```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      APP_ENV: production
      REDIS_HOST: redis
      REDIS_PORT: 6379
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: mydb
      DB_USER: myuser
      DB_PASSWORD: mypass
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    restart: unless-stopped
    networks:
      - backend

  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypass
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - backend

  redis:
    image: redis:7
    restart: unless-stopped
    networks:
      - backend

volumes:
  postgres_data:

networks:
  backend:
```

### 9.1. Phân tích kiến trúc

- `api` là service chính expose ra ngoài.
- `postgres` và `redis` chỉ ở backend network.
- DB data dùng volume riêng để dữ liệu không mất.
- `depends_on + healthcheck` giúp `api` chờ DB khỏe hơn trước khi khởi động.

---

## 10. Biến môi trường, file `.env`, và secrets

### 10.1. Dùng `.env`

Compose hỗ trợ dùng biến môi trường từ file `.env`.

Ví dụ:

```env
POSTGRES_DB=appdb
POSTGRES_USER=appuser
POSTGRES_PASSWORD=secret
APP_PORT=8000
```

Và trong `compose.yaml`:

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

### 10.2. Cảnh báo quan trọng

- `.env` tiện, nhưng không phải secret manager mạnh.
- Không nên commit secret thật lên Git.
- Với production nghiêm túc, nên dùng cơ chế secret management phù hợp hơn.

---

## 11. Logging và debugging

### 11.1. Xem logs

```bash
docker compose logs -f api
```

### 11.2. Xem inspect container

```bash
docker inspect <container_name>
```

### 11.3. Xem network

```bash
docker network ls
docker network inspect <network_name>
```

### 11.4. Xem volume

```bash
docker volume ls
docker volume inspect <volume_name>
```

### 11.5. Vào container để debug

```bash
docker compose exec api /bin/sh
```

Sau đó có thể kiểm tra:

- file có tồn tại không,
- env vars đã đúng chưa,
- app bind đúng `0.0.0.0` hay chưa,
- có resolve được DNS sang service khác không.

---

## 12. Best practices khi dùng Compose

### 12.1. Tách dev và prod bằng override hoặc file riêng

Ví dụ:

- `compose.yaml`
- `compose.dev.yaml`
- `compose.prod.yaml`

Chạy:

```bash
docker compose -f compose.yaml -f compose.dev.yaml up -d
```

### 12.2. Đặt tên service có nghĩa

Đừng đặt `app1`, `app2` nếu có thể đặt `api`, `worker`, `scheduler`, `db`, `redis`.

### 12.3. Giảm việc publish port không cần thiết

Nếu DB chỉ dùng nội bộ giữa các service, không nhất thiết publish ra host.

### 12.4. Dùng healthcheck cho dependency quan trọng

Nhất là:

- database,
- queue,
- cache,
- inference server.

### 12.5. Giữ volume rõ nghĩa

Ví dụ:

- `postgres_data`
- `mlflow_artifacts`
- `minio_data`

---

## 13. Compose trong bối cảnh MLOps / AI

Với đội AI/ML, Docker và Compose đặc biệt hữu ích trong các tình huống sau:

### 13.1. Chuẩn hóa môi trường huấn luyện và suy luận

Ví dụ một repo có thể đóng gói:

- Python + CUDA-compatible userspace,
- PyTorch/TensorFlow,
- model server,
- tracker như MLflow,
- database,
- object storage.

### 13.2. Dựng stack nhanh cho nghiên cứu

Một `compose.yaml` có thể dựng đồng thời:

- notebook service,
- MLflow,
- PostgreSQL,
- MinIO,
- Redis,
- FastAPI inference service.

### 13.3. Tách biệt dependency giữa các dự án

Thay vì để nhiều researcher dùng chung một Python môi trường trên server và gây xung đột package, mỗi project có thể có image hoặc service riêng.

### 13.4. Cảnh báo

Compose hỗ trợ container hóa rất tốt, nhưng quản trị GPU multi-user, fair scheduling, quota, isolation mạnh, autoscaling đa node và policy vận hành vẫn là câu chuyện lớn hơn, thường cần tới Kubernetes cùng NVIDIA GPU Operator hoặc các nền tảng điều phối tương đương. Những điều này tôi sẽ đề cập trong các bài viết sau.

---

## 14. Docker Compose và Kubernetes: quan hệ như thế nào?

Không nên nhìn chúng như hai công cụ cạnh tranh trực diện trong mọi ngữ cảnh.

- **Docker/Compose**: mạnh ở đóng gói và mô tả stack nhẹ.
- **Kubernetes**: mạnh ở orchestration cấp cluster.

Một luồng tự nhiên là:

1. viết app,
2. containerize bằng Docker,
3. test topology bằng Compose,
4. sau đó mới đưa lên Kubernetes nếu cần scale và governance cao hơn.

Nói cách khác:

- Docker giải quyết bài toán **đóng gói và runtime container**,
- Compose giải quyết bài toán **quản lý multi-container application ở mức đơn giản**,
- Kubernetes giải quyết bài toán **điều phối và vận hành container ở cấp cụm**.

---

## 15. Checklist học Docker Compose theo thứ tự hợp lý

### Giai đoạn 3: thực hành Compose

- dựng web + db,
- dùng volume,
- dùng env vars,
- dùng healthcheck,
- xem logs và debug network.

### Giai đoạn 4: production mindset

- tách dev/prod config,
- backup volume/data,
- registry,
- security cơ bản,
- resource limits,
- logging/monitoring.

---

## 16. Tóm tắt ngắn gọn Docker Compose

### Docker Compose

Compose là cách mô tả và vận hành nhiều service bằng một file YAML.

### Cấu trúc cơ bản

Services, networks, volumes.

### Workflow

Code → Compose file → Deploy locally → Test → Push.

---

## 17. Nên nhớ điều gì quan trọng nhất?

1. Compose không phải chỉ là vài lệnh CLI; bản chất là mô tả cấu trúc ứng dụng toàn bộ.
2. Compose rất hợp cho dev, test, staging nhỏ.
3. Compose không thay thế Kubernetes khi bước sang bài toán cluster, multi-node, policy và GPU orchestration.
4. Với team AI/ML, Docker Compose là bước nền gần như bắt buộc trước khi đi lên Kubernetes và GPU Operator.

---