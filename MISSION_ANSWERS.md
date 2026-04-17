# Day 12 Lab - Mission Answers

> **Student Name:** Nguyễn Văn Thức  
> **Student ID:** 2A202600238  
> **Date:** 17/04/2026

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found

Dựa trên `01-localhost-vs-production/develop/app.py` (basic/anti-patterns), các vấn đề chính:

1. **Hardcoded secrets trong code**: `OPENAI_API_KEY = "sk-..."` (push lên GitHub là lộ secret).
2. **Hardcoded credentials nhạy cảm khác**: `DATABASE_URL = "postgresql://admin:password123@localhost:..."`.
3. **Log secrets ra console**: `print(f"... Using key: {OPENAI_API_KEY}")`.
4. **Không có health check endpoint** (`GET /health`) → platform không biết khi nào cần restart/stop routing.
5. **Port bị hardcode** (`8000`) và **không đọc từ `PORT` env** → deploy Railway/Render dễ fail.
6. **Bind host `localhost`** (`host="localhost"`) → chạy trong container/cloud không truy cập được từ ngoài.
7. **Bật debug reload cứng** (`reload=True`) → không phù hợp production (tốn tài nguyên, rủi ro).
8. **Không có graceful shutdown** (không xử lý SIGTERM / không chờ in-flight requests).
9. **Không có input validation rõ ràng** (endpoint nhận `question: str` kiểu query/body lẫn lộn, dễ lỗi client).

### Exercise 1.3: Comparison table

| Feature | Develop | Production | Why Important? |
|---------|---------|------------|----------------|
| Config | Hardcode trong code | Lấy từ env (12-factor) | Dễ đổi giữa env, không commit secrets, deploy cloud ổn định |
| Secrets | Có thể lộ qua code/log | Không hardcode, không log secrets | Tránh rò rỉ key, tránh bị trừ tiền/bị hack |
| Host/Port | `localhost:8000` cố định | `0.0.0.0` + `PORT` env | Container/cloud cần bind all interfaces và nhận port do platform cấp |
| Health check | Không có | Có `/health` | Platform monitoring/restart + alerting |
| Readiness | Không có | Có `/ready` | LB chỉ route traffic khi instance sẵn sàng |
| Logging | `print()` + log raw | JSON structured logs | Dễ search/parse trong log aggregator, debug production nhanh hơn |
| Shutdown | Tắt đột ngột | Graceful shutdown | Tránh mất request đang xử lý, tránh lỗi khi rolling deploy |
| CORS | Mặc định/không kiểm soát | Cấu hình origins/headers | Giảm rủi ro security, UI gọi API ổn định |

---

## Part 2: Docker

### Exercise 2.1: Dockerfile questions

Dựa trên `02-docker/develop/Dockerfile` và `02-docker/production/Dockerfile`:

1. **Base image**:  
   - Develop: `python:3.11`  
   - Production: `python:3.11-slim` (builder + runtime đều slim)
2. **Working directory**:  
   - Develop: `WORKDIR /app`  
   - Production: `WORKDIR /app`
3. **Vì sao copy `requirements.txt` trước khi copy code?**  
   - Để tận dụng Docker layer cache: nếu code đổi nhưng dependencies không đổi thì không phải cài lại packages → build nhanh hơn.
4. **CMD vs ENTRYPOINT (ý nghĩa thực tế)**:  
   - `CMD` là default command, có thể override khi `docker run ... <cmd>`  
   - `ENTRYPOINT` thường “cố định” binary chính; kết hợp ENTRYPOINT + CMD để truyền args. (Repo này chủ yếu dùng `CMD`.)
5. **Tại sao production dùng multi-stage?**  
   - Stage builder có build tools để cài deps; stage runtime chỉ copy phần cần chạy → image nhỏ hơn, ít attack surface hơn.
6. **Tại sao nên chạy non-root user trong container?**  
   - Best practice security: giảm rủi ro nếu app bị khai thác, tránh quyền root trong container.

### Exercise 2.3: Image size comparison

- Develop: **1.66** GB  
- Production: **236.44** MB  
- Difference: **85.7** %

lệnh để đo (chạy từ project root):

```bash
docker build -f 02-docker/develop/Dockerfile -t agent-develop .
docker build -f 02-docker/production/Dockerfile -t agent-production .
docker images agent-develop
docker images agent-production
```

---

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment

- URL: `https://day-12-cloud-deployment-production.up.railway.app`
- Screenshot: ![screenshot](https://res.cloudinary.com/dczdnu2ba/image/upload/v1776440665/railway_deploy_ple4o4.jpg)

---

## Part 4: API Security

### Exercise 4.1-4.3: Test results

#### 4.1 API key authentication (basic)

Dựa trên `04-api-gateway/develop/app.py`:
- Endpoint `/ask` yêu cầu header `X-API-Key`
- Thiếu key → `401`
- Sai key → `403`
- Đúng key → `200`

Test commands (local):

```bash
AGENT_API_KEY=my-secret-key python 04-api-gateway/develop/app.py

# With key → 200
curl -H "X-API-Key: my-secret-key" -X POST \
  "http://localhost:8000/ask?question=hello"

# Missing key → 401
curl -X POST "http://localhost:8000/ask?question=hello"
```

**Test output (paste here):**  
```text
missing key → 401
{"detail":"Missing API key. Include header: X-API-Key: <your-key>"}

wrong key → 403
{"detail":"Invalid API key."}

valid key → 200
{"question":"hello","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}
```

#### 4.2 JWT authentication (advanced)

Dựa trên `04-api-gateway/production/auth.py` + `04-api-gateway/production/app.py`:
- `POST /auth/token` nhận username/password → trả JWT
- `POST /ask` yêu cầu header `Authorization: Bearer <token>`
- Token hết hạn/invalid → 401/403

Test commands (local):

```bash
python 04-api-gateway/production/app.py

# Get token
curl -X POST http://localhost:8000/auth/token \
  -H "Content-Type: application/json" \
  -d '{"username":"student","password":"demo123"}'

# Call /ask with token
curl -X POST http://localhost:8000/ask \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"question":"what is docker?"}'
```

**Test output (paste here):**  
```text
login → 200
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "bearer",
  "expires_in_minutes": 60,
  "hint": "Include in header: Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."
}

no token → 401
{"detail":"Authentication required. Include: Authorization: Bearer <token>"}

with token → 200
{
  "question": "what is docker?",
  "answer": "Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!",
  "usage": {
    "requests_remaining": 9,
    "budget_remaining_usd": 1.9e-05
  }
}
```

#### 4.3 Rate limiting (advanced)

Dựa trên `04-api-gateway/production/rate_limiter.py`:
- Sliding window in-memory limiter
- User: 10 req/min (`rate_limiter_user`)
- Admin: 100 req/min (`rate_limiter_admin`)
- Vượt limit → `429` + headers `Retry-After`, `X-RateLimit-*`

**Test output (paste here):**  
```text
after exceeding limit → 429
{
  "detail": {
    "error": "Rate limit exceeded",
    "limit": 10,
    "window_seconds": 60,
    "retry_after_seconds": 59
  }
}

response headers (sample):
Retry-After: 59
X-RateLimit-Limit: 10
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1776440934
```

### Exercise 4.4: Cost guard implementation

Dựa trên `04-api-gateway/production/cost_guard.py`:

- **Mục tiêu**: tránh “bill shock” khi gọi LLM
- **Cách làm trong repo**:
  - Lưu usage per-user theo ngày (`UsageRecord`): `input_tokens`, `output_tokens`, `request_count`
  - Tính chi phí theo công thức \(cost = input\_tokens/1000 * price\_in + output\_tokens/1000 * price\_out\)
  - `check_budget(user_id)` chạy **trước** khi gọi LLM:
    - Nếu vượt per-user daily budget → trả **402 Payment Required**
    - Nếu vượt global daily budget → trả **503 Service temporarily unavailable**
  - Sau khi có response, gọi `record_usage(...)` để cộng dồn tokens/cost
  - Có warning khi dùng vượt `warn_at_pct` (mặc định 80%)

---

## Part 5: Scaling & Reliability

### Exercise 5.1-5.5: Implementation notes

#### 5.1 Health check & readiness check

Dựa trên `05-scaling-reliability/develop/app.py` và final project `06-lab-complete/app/main.py`:
- `/health` (liveness): trả 200 khi service sống + có uptime/version/checks
- `/ready` (readiness): trả 200 khi sẵn sàng nhận traffic, trả 503 khi chưa ready hoặc dependency (Redis) chưa OK

**Test output:**  
```text
05-scaling-reliability (develop)
GET /health → 200
{
  "status": "ok",
  "uptime_seconds": 0.2,
  "version": "1.0.0",
  "environment": "development",
  "checks": { "memory": { "status": "ok", "used_percent": 86.7 } }
}

GET /ready → 200
{"ready":true,"in_flight_requests":1}

06-lab-complete (Railway public URL)
GET https://day-12-cloud-deployment-production.up.railway.app/health → 200
{"status":"ok","version":"1.0.0","environment":"development","checks":{"redis":true,"llm":"mock"},...}

GET https://day-12-cloud-deployment-production.up.railway.app/ready → 200
{"ready":true}
```

#### 5.2 Graceful shutdown (SIGTERM)

Dựa trên `05-scaling-reliability/develop/app.py`:
- Dùng lifespan shutdown để set `_is_ready = False`
- Chờ `_in_flight_requests` về 0 (tối đa 30s) trước khi shutdown
- Có handler log khi nhận SIGTERM/SIGINT

**Test output:**  
```text
Chạy server: python 05-scaling-reliability/develop/app.py (PORT=8125)
Gửi shutdown signal (Windows): CTRL_BREAK_EVENT

Log trích xuất:
INFO:     Shutting down
INFO:     Waiting for application shutdown.
... INFO 🔄 Graceful shutdown initiated...
... INFO ✅ Shutdown complete
INFO:     Application shutdown complete.
```

#### 5.3 Stateless design

Dựa trên `05-scaling-reliability/production/app.py`:
- Stateless = không lưu session/history trong memory của 1 instance
- Lưu session/history vào Redis (`session:{session_id}`) để instance nào cũng đọc/ghi được
- Nếu Redis không có (demo) thì fallback in-memory và **không scalable** (được log cảnh báo)

#### 5.4 Run load balanced stack

Trong `05-scaling-reliability/production/test_stateless.py`:
- Kịch bản gửi nhiều requests vào `BASE_URL` và kiểm tra trường `served_by`
- Kết quả lý tưởng: thấy nhiều instance khác nhau cùng phục vụ nhưng history vẫn giữ nguyên (nhờ Redis)

**Test output:**  
```text
Chạy test bằng FastAPI TestClient (single instance):

POST /chat → 200
{ "session_id": "...", "served_by": "instance-4404ce", "storage": "in-memory" }

POST /chat (same session_id) → 200
{ "session_id": "...", "served_by": "instance-4404ce", "storage": "in-memory" }

GET /chat/{session_id}/history → 200
{ "count": 4 }

Ghi chú:
- Môi trường test hiện tại không có Redis server, nên app fallback `storage: in-memory` (đúng như code cảnh báo).
- Khi chạy đúng “load balanced stack” theo bài (docker compose + scale), `served_by` sẽ thay đổi giữa các instance và `storage` phải là `redis`.
```

#### 5.5 Test stateless design

Tiêu chí kiểm chứng:
- Request 1 tạo session và lưu history
- Kill 1 instance / route sang instance khác
- Request tiếp theo vẫn đọc được history/session từ Redis

**Test output:**  
```text
Kết quả kiểm chứng session/history (trên 05 production app):
- Sau 2 lần gọi /chat cùng session_id, gọi /history trả count = 4 (2 user + 2 assistant)
```

