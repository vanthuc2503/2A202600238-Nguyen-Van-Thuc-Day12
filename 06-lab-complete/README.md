# Lab 06 — Complete Production Agent (Deploy được ngay)

Lab này là “bản hoàn chỉnh” kết hợp các concept của Day 12 vào **một service có thể deploy**:
- **FastAPI backend** + validation (Pydantic)
- **Auth bằng API key** (header `X-API-Key`)
- **Redis** cho rate-limit + history + budget guard
- **Health/Ready probes** (`/health`, `/ready`)
- **UI test nhanh** tại `/ui` (gọi backend qua fetch)
- **Docker multi-stage** + chạy non-root
- File cấu hình deploy sẵn cho **Railway** và **Render**

---

## 1) Cấu trúc thư mục `06-lab-complete/`

```
06-lab-complete/
├── app/
│   ├── main.py             # FastAPI app + endpoints + middleware logging
│   ├── config.py           # Settings đọc từ environment (12-factor)
│   ├── auth.py             # Verify API key (X-API-Key)
│   ├── redis_client.py     # Redis client (đọc REDIS_URL, lazy init + ping)
│   ├── rate_limiter.py     # Sliding-window rate limit (Redis zset)
│   └── cost_guard.py       # Budget guard theo tháng (Redis)
├── web/
│   ├── index.html          # UI tại /ui
│   └── static/
│       ├── app.js          # gọi /health /ready /ask
│       └── styles.css
├── nginx/
│   └── nginx.conf          # reverse proxy local (port 80)
├── Dockerfile              # multi-stage, non-root, HEALTHCHECK
├── docker-compose.yml      # local stack: nginx + agent + redis
├── railway.toml            # Railway deploy config
├── render.yaml             # Render blueprint config
├── requirements.txt
├── .env.example
├── .dockerignore
└── check_production_ready.py
```

---

## 2) Kiến trúc chạy local (Docker Compose)

Khi chạy `docker compose up`, bạn có 3 service:
- **`redis`**: lưu rate-limit / history / budget keys
- **`agent`**: FastAPI chạy port 8000 (internal)
- **`nginx`**: public port **80** → reverse proxy vào `agent:8000`

Vì vậy khi test local, bạn gọi:
- `http://localhost/health`
- `http://localhost/ready`
- `http://localhost/ui`
- `http://localhost/ask`

---

## 3) Environment variables (cần biết để chạy đúng)

File mẫu: `.env.example` (copy thành `.env` khi chạy local)

- **`AGENT_API_KEY`**: bắt buộc (backend sẽ trả 401 nếu thiếu/sai)
- **`OPENAI_API_KEY`**: optional (nếu không có, backend sẽ dùng mock)
- **`REDIS_URL`**: bắt buộc để `/ready` và `/ask` hoạt động ổn định
  - Local compose dùng: `redis://redis:6379/0`
  - Deploy Railway: set theo **Redis service URL** (thường Railway inject `REDIS_URL`)
- **`ALLOWED_ORIGINS`**: CORS (mặc định `*`)
- **`LLM_MODEL`**: model dùng khi gọi OpenAI (mặc định `gpt-4o-mini`)
- **`RATE_LIMIT_PER_MINUTE`**: giới hạn request / phút / `user_id`
- **`MONTHLY_BUDGET_USD`**: budget guard / tháng / `user_id`

Ghi chú quan trọng (đúng theo UI hiện tại):
- UI có thể gửi OpenAI key qua header **`X-OpenAI-Key`** cho mỗi request (backend không lưu key).

---

## 4) Chạy local (khuyến nghị)

### Bước 1: Chuẩn bị env

```bash
cd 06-lab-complete
cp .env.example .env
```

### Bước 2: Start stack

```bash
docker compose up --build
```

### Bước 3: Test nhanh

Health/Ready:

```bash
curl http://localhost/health
curl http://localhost/ready
```

Test `/ask` (nhớ có `user_id`):

```bash
curl -H "X-API-Key: dev-key-change-me-in-production" ^
  -H "Content-Type: application/json" ^
  -X POST http://localhost/ask ^
  -d "{\"user_id\":\"test\",\"question\":\"xin chào\"}"
```

### Bước 4: Test bằng UI

Mở:
- `http://localhost/ui`

Trong Settings:
- `Base URL`: để trống nếu đang mở UI trên đúng domain
- `Backend API key (X-API-Key)`: điền `AGENT_API_KEY`
- `OpenAI API key (X-OpenAI-Key)`: optional
- `user_id`: ví dụ `test`

---

## 5) Deploy Railway (backend + Redis)

### Bước 1: Deploy backend từ repo

Railway có thể build Dockerfile trong `06-lab-complete/`.
Sau khi tạo service backend, set variables:
- **`AGENT_API_KEY`** (bắt buộc)
- **`REDIS_URL`** (bắt buộc để tránh lỗi connect `localhost:6379`)
- **`OPENAI_API_KEY`** (optional — hoặc dùng `X-OpenAI-Key` từ UI)

### Bước 2: Add Redis service

Trong Railway:
- Add → Database → Redis
- Đảm bảo backend service **nhìn thấy** biến `REDIS_URL` (hoặc copy URL Redis vào `REDIS_URL`)

### Bước 3: Test sau deploy

- `GET /health` phải trả `{"status":"ok", ...}`
- `GET /ready` phải trả `{"ready": true}` (nếu Redis OK)
- `GET /ui` mở UI test
- `POST /ask` chạy được và không còn 500 Redis

Nếu thấy `503 Redis unavailable...`:
- kiểm tra lại `REDIS_URL` đã set đúng trên backend service chưa

---

## 6) Deploy Render (tuỳ chọn)

File `render.yaml` đã có blueprint cơ bản.
Bạn vẫn cần set thêm:
- `AGENT_API_KEY`
- `OPENAI_API_KEY` (optional)
- **`REDIS_URL`** (nếu bạn dùng Redis ngoài; Render blueprint hiện tại chưa khai báo Redis service)

---

## 7) Kiểm tra “production readiness”

```bash
python check_production_ready.py
```

Script sẽ kiểm tra nhanh:
- file cần thiết (Dockerfile/compose/env/deploy files)
- `.env` có bị track không
- endpoints `/health`, `/ready`
- các dấu hiệu “production-ready” trong Dockerfile

---

## 8) Tiêu chí hoàn thành Lab 06 (gợi ý chấm)

- **Chạy local** bằng Docker Compose và test `/ask` OK
- **Deploy Railway** có public URL
- `/health` OK, `/ready` OK
- UI `/ui` chat được (mock hoặc OpenAI)
- Không hardcode secret, config lấy từ env, log có JSON structured events
