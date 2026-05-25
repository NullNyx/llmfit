# Deploy `llmfit` bằng Docker Compose

Tài liệu này dành cho DevOps. Mục tiêu: copy repo lên server, chạy 1 lệnh, mở dashboard, lọc ra model phù hợp nhất.

## 1. Quick start

Từ root repo chạy đúng 3 lệnh:

```sh
cd llmfit-web && npm ci && npm run build && cd ..
docker compose build --no-cache
docker compose up -d
```

Mở trình duyệt:

```text
http://<server-ip>:8787/
```

Kiểm tra API:

```sh
curl http://localhost:8787/api/v1/system
curl "http://localhost:8787/api/v1/models/top?limit=5&min_fit=good"
```

Nếu cần rebuild sạch:

```sh
docker compose down
docker compose build --no-cache
docker compose up -d
```

---

## 2. Kết quả sau khi chạy

Service `llmfit` sẽ:

- tự detect `RAM`, `CPU`, `GPU`
- chấm điểm hàng trăm model theo cấu hình server
- mở REST API tại cổng `8787`
- mở web dashboard tại `/`

URL mặc định:

- Dashboard: `http://<server-ip>:8787/`
- API system: `http://<server-ip>:8787/api/v1/system`
- API top models: `http://<server-ip>:8787/api/v1/models/top?limit=5&min_fit=good`

---

## 3. Yêu cầu trên server

Bắt buộc:

- cài `Docker`
- cài `Docker Compose` plugin (`docker compose`)

Tùy chọn theo phần cứng:

- **NVIDIA GPU**: cài `nvidia-container-toolkit`
- **AMD GPU / ROCm**: host phải expose device ROCm vào container
- **CPU-only**: không cần gì thêm

Lưu ý:

- Docker trên **Apple Silicon** không phù hợp để detect phần cứng chính xác cho use case này. Nếu server là Mac, nên chạy binary native thay vì container.

---

## 4. File đã chuẩn bị sẵn

Repo hiện có:

- `Dockerfile`
- `docker-compose.yml`
- `DEPLOY_DOCKER_COMPOSE.md`

Mode chạy mặc định trong compose:

```sh
llmfit serve --host 0.0.0.0 --port 8787
```

Mode này phù hợp nhất vì:

- chạy lâu dài
- có API cho tool khác gọi
- có dashboard để người không dùng CLI vẫn thao tác được

---

## 5. Cách dùng dashboard để chọn model nhanh

Sau khi mở `http://<server-ip>:8787/`, làm đúng flow này:

### Bước 1: nhìn cấu hình máy

Xem phần thông tin system ở đầu trang hoặc gọi:

```sh
curl http://localhost:8787/api/v1/system
```

Xác nhận các giá trị sau đúng với server:

- `RAM`
- `CPU cores`
- `GPU`
- `VRAM`

Nếu sai, xem mục 9 để override.

### Bước 2: lọc model phù hợp nhất

Dùng các filter sau trong dashboard:

- **Fit**: chọn `Good` hoặc `Perfect`
- **Use Case**: chọn theo nhu cầu
  - `Coding`
  - `Chat`
  - `Reasoning`
  - `General`
  - `Embedding`
- **Runtime**:
  - `GPU` nếu server có GPU
  - `CPU Only` nếu server không có GPU

### Bước 3: đọc 4 cột quan trọng nhất

Ưu tiên các cột này:

- `Fit`
  - `Perfect` = phù hợp nhất
  - `Good` = chạy ổn
  - `Marginal` = chạy được nhưng chật
- `Score`
  - càng cao càng tốt
- `tok/s`
  - càng cao càng nhanh
- `Mem%`
  - càng thấp càng dư tài nguyên

### Bước 4: rule chọn model đơn giản

DevOps có thể dùng rule này:

- ưu tiên model có `Fit = Perfect`
- nếu không có, lấy `Fit = Good`
- trong cùng mức fit, chọn model có `Score` cao hơn
- nếu cần tốc độ, ưu tiên `tok/s` cao hơn
- tránh model có `Mem%` quá sát 100%

### Bước 5: quick recommendation theo use case

Gợi ý lọc nhanh:

- chatbot nội bộ: `Use Case = Chat`
- code assistant: `Use Case = Coding`
- tác vụ tổng quát: `Use Case = General`
- vector/semantic search: `Use Case = Embedding`

---

## 6. Quick setup filter cho DevOps

### Trường hợp A: server có GPU, muốn model ổn định

Trên dashboard:

- `Fit = Good`
- `Runtime = GPU`
- sort theo `Score`

API tương đương:

```sh
curl "http://localhost:8787/api/v1/models/top?limit=10&min_fit=good"
```

### Trường hợp B: server GPU yếu, vẫn muốn chạy được

Trên dashboard:

- `Fit = Marginal`
- `Runtime = GPU`
- sort theo `Score`

API tương đương:

```sh
curl "http://localhost:8787/api/v1/models/top?limit=10&min_fit=marginal"
```

### Trường hợp C: server chỉ có CPU

Trên dashboard:

- `Runtime = CPU Only`
- `Fit = Good` hoặc `Marginal`
- ưu tiên model nhỏ, `tok/s` cao

### Trường hợp D: cần model coding

Trên dashboard:

- `Use Case = Coding`
- `Fit = Good`
- sort theo `Score`

API tương đương:

```sh
curl "http://localhost:8787/api/v1/models/top?limit=5&min_fit=good&use_case=coding"
```

### Trường hợp E: cần model chat nội bộ

API nhanh:

```sh
curl "http://localhost:8787/api/v1/models/top?limit=5&min_fit=good&use_case=chat"
```

---

## 7. API hữu ích nhất

### Lấy cấu hình máy detect được

```sh
curl http://localhost:8787/api/v1/system
```

### Lấy top model phù hợp nhất

```sh
curl "http://localhost:8787/api/v1/models/top?limit=10&min_fit=good"
```

### Lọc theo coding

```sh
curl "http://localhost:8787/api/v1/models/top?limit=5&min_fit=good&use_case=coding"
```

### Lọc theo chat

```sh
curl "http://localhost:8787/api/v1/models/top?limit=5&min_fit=good&use_case=chat"
```

### Lấy toàn bộ model đã chấm điểm

```sh
curl "http://localhost:8787/api/v1/models?limit=50&sort=score"
```

### Xem chi tiết một model

```sh
curl "http://localhost:8787/api/v1/models/Mistral"
```

API đầy đủ xem thêm ở `API.md`.

---

## 8. Cấu hình cho từng loại server

### 8.1 CPU-only server

Không cần sửa gì:

```sh
cd llmfit-web && npm ci && npm run build && cd ..
docker compose build --no-cache
docker compose up -d
```

### 8.2 NVIDIA server

Host phải cài `nvidia-container-toolkit` trước.

Kiểm tra host:

```sh
nvidia-smi
```

Kiểm tra Docker thấy GPU:

```sh
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

Sau đó chạy:

```sh
cd llmfit-web && npm ci && npm run build && cd ..
docker compose build --no-cache
docker compose up -d
```

### 8.3 AMD ROCm server

Sửa `docker-compose.yml`:

- bỏ block NVIDIA nếu có
- bật block ROCm devices

Mẫu:

```yaml
    devices:
      - /dev/kfd:/dev/kfd
      - /dev/dri:/dev/dri
```

Rồi chạy lại:

```sh
cd llmfit-web && npm ci && npm run build && cd ..
docker compose build --no-cache
docker compose up -d
```

---

## 9. Nếu detect phần cứng không đúng

Một số môi trường VM / container không expose GPU hoặc RAM đúng.

### Hướng 1: fix passthrough phần cứng

Ưu tiên nhất.

### Hướng 2: override bằng CLI args trong compose

Sửa `docker-compose.yml`.

Ví dụ server có:

- GPU VRAM: `24G`
- RAM: `64G`
- CPU cores: `16`

Đổi `command` thành:

```yaml
    command:
      [
        "--memory", "24G",
        "--ram", "64G",
        "--cpu-cores", "16",
        "serve", "--host", "0.0.0.0", "--port", "8787"
      ]
```

Rồi chạy lại:

```sh
docker compose down
docker compose build --no-cache
docker compose up -d
```

---

## 10. Lệnh vận hành cơ bản

Start / rebuild:

```sh
docker compose build --no-cache
docker compose up -d
```

Stop:

```sh
docker compose down
```

Restart:

```sh
docker compose restart llmfit
```

Xem log:

```sh
docker compose logs -f llmfit
```

Xem config render cuối cùng:

```sh
docker compose config
```

Vào shell container:

```sh
docker compose exec llmfit sh
```

Chạy thử CLI trong container:

```sh
docker compose exec llmfit llmfit system --json
```

---

## 11. Troubleshooting

### Port `8787` đã bị dùng

Sửa `docker-compose.yml`:

```yaml
ports:
  - "8087:8787"
```

Khi đó truy cập:

- `http://<server-ip>:8087/`

### Dashboard hiện trang fallback “Frontend assets are missing”

Nguyên nhân: image cũ hoặc chưa rebuild sau khi build frontend.

Fix đúng thứ tự:

```sh
cd llmfit-web && npm ci && npm run build && cd ..
docker compose down
docker compose build --no-cache
docker compose up -d
```

### API không lên

Kiểm tra:

```sh
docker compose logs -f llmfit
```

### Không thấy GPU

Kiểm tra trong host:

```sh
nvidia-smi
```

hoặc AMD:

```sh
rocm-smi
```

Rồi kiểm tra passthrough runtime.

### Dashboard lên nhưng kết quả không đúng kỳ vọng

Thử gọi trực tiếp:

```sh
curl http://localhost:8787/api/v1/system
```

Nếu RAM/GPU detect sai, dùng hardware override ở mục 9.

---

## 12. File liên quan

- `docker-compose.yml`
- `Dockerfile`
- `API.md`
- `README.md`
- `llmfit-tui/src/main.rs`
