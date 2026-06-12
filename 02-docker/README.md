# Section 2 — Docker: Đóng Gói Agent Thành Container

## Mục tiêu học
- Hiểu container là gì và tại sao cần nó
- Viết Dockerfile đúng cách (single vs multi-stage)
- Dùng Docker Compose để chạy multi-service stack
- Tối ưu image size xuống dưới 500 MB

---

## Ví dụ Basic — Dockerfile Đơn Giản

```
develop/
├── app.py
├── Dockerfile          # Single-stage, dễ hiểu
├── .dockerignore
└── requirements.txt
```

### Chạy thử
```bash
# IMPORTANT: Build from project root!
cd ../..  # Go to project root

# Build image
docker build -f 02-docker/develop/Dockerfile -t agent-develop .

# Xem size
docker images agent-develop

# Chạy container
# -p 8000:8000 : map port 8000 của máy host → port 8000 trong container
# -d           : chạy ở chế độ detached (nền), terminal không bị block
# agent-develop: tên image đã build ở bước trên
docker run -p 8000:8000 -d agent-develop

# Xem container đang chạy
docker ps

# Truy cập vào bên trong container đang chạy (mở shell tương tác)
# -i: giữ kết nối stdin mở | -t: cấp terminal giả (pseudo-TTY)
# Hữu ích để debug, kiểm tra file, xem log, hoặc chạy lệnh trực tiếp trong container

docker exec -it <container-id> sh

# Test
curl http://localhost:8000/health
```

---

## Ví dụ Advanced — Multi-Stage + Docker Compose

```
production/
├── app.py
├── Dockerfile              # Multi-stage build → image nhỏ hơn nhiều
├── docker-compose.yml      # Full stack: agent + vector store + redis
├── nginx/
│   └── nginx.conf          # Reverse proxy
├── .dockerignore
└── requirements.txt
```

### Chạy thử
```bash
# From project root
cd ../..  # if not already there

# Khởi động toàn bộ stack (1 lệnh!)
docker compose -f 02-docker/production/docker-compose.yml up

# Xem các service đang chạy
docker compose -f 02-docker/production/docker-compose.yml ps

# Test agent qua Nginx
curl http://localhost/health

# Dừng toàn bộ
docker compose -f 02-docker/production/docker-compose.yml down
```

### So sánh image size:

```bash
# Basic vs Advanced
docker images | grep agent
# agent-basic    ~  800 MB  ← python:3.11 base
# agent-advanced ~  160 MB  ← python:3.11-slim + multi-stage
```

---

## Lý thuyết: Tại Sao Multi-Stage?

```dockerfile
# Stage 1: Builder — có đầy đủ tools để compile deps
FROM python:3.11 AS builder   # 1 GB
RUN pip install ...            # thêm deps vào layer này

# Stage 2: Runtime — chỉ copy những gì cần chạy
FROM python:3.11-slim          # 150 MB ← bắt đầu từ image sạch
COPY --from=builder ...        # copy chỉ /site-packages
```

**Kết quả:** Final image chỉ có runtime, không có pip, không có build tools → nhỏ và an toàn hơn.

---

## Câu hỏi thảo luận

1. Tại sao `COPY requirements.txt .` rồi `RUN pip install` TRƯỚC khi `COPY . .`?
2. `.dockerignore` nên chứa những gì? Tại sao `venv/` và `.env` quan trọng?
3. Nếu agent cần đọc file từ disk, làm sao mount volume vào container?

---

## Trả lời câu hỏi thảo luận (Answers)

**Q1 — Why `COPY requirements.txt .` + `RUN pip install` BEFORE `COPY . .`?**
Docker builds in **layers** and **caches** each one; it only rebuilds a layer if that
layer's inputs changed. Dependencies change **rarely**, but your code changes
**constantly**. By copying `requirements.txt` and installing *first* (an earlier layer),
then copying your code *after* (a later layer), editing your code doesn't invalidate the
slow `pip install` layer — Docker reuses it from cache. Result: rebuilds drop from
**minutes to seconds**. If you copied all the code first, *any* code edit would bust the
cache and force a full reinstall every time.

**Q2 — What goes in `.dockerignore`, and why are `venv/` and `.env` important?**
`.dockerignore` lists files to **keep out** of the build context (and therefore out of
the image): caches (`__pycache__/`), `venv/`, IDE folders, `.git/`, docs, tests, and —
critically — `.env`.
- **`venv/`** is your local virtual environment: it's large *and* built for **your** OS
  (e.g. Windows), so it's useless and even harmful inside a Linux container. Excluding it
  shrinks the build context and speeds builds. (The container installs its own deps via
  `pip install` anyway.)
- **`.env`** holds **secrets** (API keys, DB passwords). If it got copied into the image,
  anyone who pulls that image from a registry could read your secrets. **Never bake
  secrets into an image** — inject them at runtime via environment variables instead.

**Q3 — If the agent needs to read files from disk, how do you mount a volume?**
Use a **volume** so data lives *outside* the container and survives restarts (containers
are disposable — their internal disk vanishes when removed).
- **CLI:** `docker run -v mydata:/app/data agent-production`
  (left of `:` = volume name on the host, right = path inside the container)
- **Compose:** under the service add `volumes: ["mydata:/app/data"]`, and declare
  `mydata:` in the top-level `volumes:` block — exactly how `redis_data` and `qdrant_data`
  work in this lab's `docker-compose.yml`.
- For mounting a **local folder** directly (e.g. live config), use a bind mount:
  `-v ./localdir:/app/data` (or the `./nginx/nginx.conf:/etc/nginx/nginx.conf:ro` pattern
  already used for Nginx, where `:ro` means read-only).
