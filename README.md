# Day 12 — Cloud Deployment (FastAPI + Docker + Railway/Render)

> Repo thực hành Day 12 (AICB-P1 · VinUniversity 2026).  
> Trọng tâm: **deploy một AI agent production-ready** theo 12-factor + Docker + Redis.

---

## Cấu trúc repo (đúng theo project này)

```
2A202600238-Nguyen-Van-Thuc-Day12/
├── 01-localhost-vs-production/
├── 02-docker/
├── 03-cloud-deployment/
├── 04-api-gateway/
├── 05-scaling-reliability/
├── 06-lab-complete/                 # Full project để deploy (khuyên dùng)
│   ├── app/                         # FastAPI backend
│   │   ├── main.py                  # API: /ask, /health, /ready, /ui
│   │   ├── config.py                # Settings từ env (12-factor)
│   │   ├── auth.py                  # X-API-Key
│   │   ├── rate_limiter.py          # Rate limit (Redis)
│   │   ├── redis_client.py          # Redis client (đọc REDIS_URL)
│   │   └── cost_guard.py            # Budget guard
│   ├── web/                         # UI test nhanh cho backend
│   │   ├── index.html
│   │   └── static/
│   │       ├── app.js
│   │       └── styles.css
│   ├── nginx/                       # (tuỳ chọn) reverse proxy / LB
│   ├── Dockerfile
│   ├── docker-compose.yml           # local stack: agent + redis
│   ├── railway.toml                 # deploy Railway
│   ├── render.yaml                  # deploy Render
│   ├── requirements.txt
│   ├── .env.example
│   └── check_production_ready.py
└── utils/                           # shared utilities (mock, helpers)
```

---

## Bắt đầu nhanh (khuyên dùng: `06-lab-complete`)

### Chạy local bằng Docker Compose

```bash
cd 06-lab-complete
cp .env.example .env
docker compose up --build
```

Mở các endpoint:
- `GET /health`: liveness
- `GET /ready`: readiness (đòi Redis OK)
- `GET /ui`: UI test nhanh (frontend gọi `POST /ask`)

### Test API bằng curl

```bash
curl -H "X-API-Key: dev-key-change-me-in-production" ^
  -H "Content-Type: application/json" ^
  -X POST http://localhost:8000/ask ^
  -d "{\"user_id\":\"test\",\"question\":\"xin chào\"}"
```

---

## Environment variables (backend)

Thiết lập tại `.env` (local) hoặc Railway/Render Variables:

- **`AGENT_API_KEY`**: bắt buộc (header `X-API-Key`)
- **`OPENAI_API_KEY`**: optional (nếu không có sẽ chạy mock)
- **`REDIS_URL`**: bắt buộc khi deploy cloud (Railway Redis)  
  Ví dụ: `redis://default:password@host:port/0`
- **`ALLOWED_ORIGINS`**: mặc định `*` (có thể set domain UI)
- **`LLM_MODEL`**: mặc định `gpt-4o-mini`

Ghi chú:
- UI có thể gửi OpenAI key theo header **`X-OpenAI-Key`** (backend không lưu key).
- Nếu Redis chưa kết nối được, `/ask` sẽ trả **503** thay vì 500.

---

## Deploy Railway

- **Backend service**: deploy thư mục `06-lab-complete/` (Dockerfile đã sẵn)
- **Add Redis** trong Railway, rồi đảm bảo backend có biến **`REDIS_URL`**

Biến tối thiểu cần set:
- `AGENT_API_KEY`
- `REDIS_URL`
- (optional) `OPENAI_API_KEY`

Sau deploy, test:
- `GET /health`
- `GET /ready`
- `POST /ask`

---

## Tài liệu đi kèm trong repo

- `QUICK_START.md`: chạy nhanh
- `CODE_LAB.md`: làm lab chi tiết
- `QUICK_REFERENCE.md`: cheat sheet
- `TROUBLESHOOTING.md`: xử lý lỗi thường gặp
- `DAY12_DELIVERY_CHECKLIST.md`: checklist nộp bài
