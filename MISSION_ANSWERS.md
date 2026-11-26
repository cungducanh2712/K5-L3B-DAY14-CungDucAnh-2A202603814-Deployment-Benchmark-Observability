# Day 14 Lab - Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found (`01-localhost-vs-production/develop/app.py`)

Dưới đây là các anti-pattern tìm được trong `develop/app.py` cùng cách `production/app.py` và `production/config.py` đã khắc phục:

1. **Hardcode secrets trong mã nguồn**
   - **Develop (`develop/app.py:17-18`):** Gán trực tiếp `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"` và `DATABASE_URL = "postgresql://admin:password123@localhost:5432/mydb"` trong code.
   - **Production (`production/config.py:27, 42-43`):** Đọc qua `os.getenv("OPENAI_API_KEY", "")` và `os.getenv("AGENT_API_KEY", "")`, đồng thời có hàm `validate()` để fail-fast nếu thiếu `AGENT_API_KEY` khi `ENVIRONMENT=production`.

2. **Cấu hình cứng, không quản lý qua environment variables**
   - **Develop (`develop/app.py:21-22, 51-53`):** Hardcode `DEBUG = True`, `MAX_TOKENS = 500`, `host="localhost"`, `port=8000`, `reload=True`. Khi đưa lên container (`localhost` không nhận traffic từ ngoài) hoặc cloud (Railway/Render inject `PORT` động), app sẽ không chạy đúng.
   - **Production (`production/config.py:17-35`, `production/app.py:194-199`):** Tập trung toàn bộ cấu hình trong dataclass `Settings` đọc từ biến môi trường (`HOST=0.0.0.0`, `PORT`, `DEBUG`, `ENVIRONMENT`, `MAX_TOKENS`, `ALLOWED_ORIGINS`), `reload=settings.debug`.

3. **Dùng `print()` thay cho structured logging và làm lộ secret ra log**
   - **Develop (`develop/app.py:33-34, 38`):** Dùng `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")`, vừa không có cấu trúc JSON để hệ thống thu thập log (Loki/Datadog) parse, vừa làm lộ trực tiếp API key ra stdout.
   - **Production (`production/app.py:28-32, 111-122`):** Cấu hình `logging.basicConfig` định dạng JSON (`{"time":...,"level":...,"msg":...}`), chỉ log metadata an toàn như `question_length`, `client_ip`, `response_length`.

4. **Không có Health Check / Readiness endpoint**
   - **Develop (`develop/app.py`):** Không có `/health` hay `/ready` (gọi `GET /health` trả `404 Not Found`), khiến nền tảng cloud hoặc load balancer không biết container còn sống hay đã sẵn sàng nhận traffic.
   - **Production (`production/app.py:135-161`):** Bổ sung `GET /health` (liveness probe trả `200` kèm `uptime_seconds`, `version`, `environment`) và `GET /ready` (readiness probe trả `200` hoặc `503` khi `not is_ready`).

5. **Không có Graceful Shutdown / Lifecycle management**
   - **Develop (`develop/app.py`):** Không xử lý tín hiệu `SIGTERM` hay vòng đời startup/shutdown; khi platform tắt container, các request đang xử lý bị ngắt đột ngột.
   - **Production (`production/app.py:40-67, 180-189`):** Dùng `lifespan` (`@asynccontextmanager`) để chuyển `is_ready = False` khi shutdown, chờ request hiện tại hoàn thành, và đăng ký `signal.signal(signal.SIGTERM, handle_sigterm)`.

6. **Nhận input qua query parameter và thiếu CORS/Validation**
   - **Develop (`develop/app.py:30-31`):** `ask_agent(question: str)` nhận câu hỏi qua query string (`?question=Hello`), dễ lộ nội dung nhạy cảm trong URL/access log và giới hạn độ dài URL.
   - **Production (`production/app.py:77-82, 102-108`):** Cấu hình `CORSMiddleware` theo `ALLOWED_ORIGINS`, nhận dữ liệu qua JSON body và trả `422` nếu thiếu trường `question`.

---

### Exercise 1.2: Kết quả chạy bản `develop`

Chạy trong WSL2 (`Ubuntu`) tại `01-localhost-vs-production/develop`:

```bash
$ curl -sS -i http://localhost:8000/
HTTP/1.1 200 OK
content-type: application/json

{"message":"Hello! Agent is running on my machine :)"}

$ curl -sS -i -X POST 'http://localhost:8000/ask?question=Hello'
HTTP/1.1 200 OK
content-type: application/json

{"answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}

$ curl -sS -i http://localhost:8000/health
HTTP/1.1 404 Not Found
content-type: application/json

{"detail":"Not Found"}
```

**Log server ghi nhận được (bị lộ secret và bật reloader):**
```text
INFO:     Uvicorn running on http://localhost:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [506] using WatchFiles
Starting agent on localhost:8000...
[DEBUG] Got question: Hello
[DEBUG] Using key: sk-hardcoded-fake-key-never-do-this
[DEBUG] Response: Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.
```

---

### Exercise 1.3: So sánh Develop vs Production

Kết quả kiểm thử nhanh bản `production` (WSL2 Ubuntu, `ENVIRONMENT=development`):
- `GET /health` → `200 OK`: `{"status":"ok","uptime_seconds":2.7,"version":"1.0.0","environment":"development","timestamp":"2026-11-26T07:34:06.525983+00:00"}`
- `GET /ready` → `200 OK`: `{"ready":true}`
- `POST /ask` (JSON `{"question": "Hello"}`) → `200 OK`: `{"question":"Hello","answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic.","model":"gpt-4o-mini"}`

| Feature | Develop (Basic) | Production (Advanced) | Tại sao quan trọng? |
|---|---|---|---|
| **Config & Secrets** | Hardcode `OPENAI_API_KEY`, `DATABASE_URL`, `DEBUG=True` trong `app.py` | Đọc từ biến môi trường qua `Settings` (`config.py`), tách `.env.example`, có `validate()` fail-fast | Tránh lộ secret lên Git; thay đổi cấu hình giữa dev/staging/prod không cần sửa code |
| **Network & Port** | `host="localhost"`, `port=8000`, `reload=True` cố định | `host="0.0.0.0"`, `port=PORT` từ env, `reload=settings.debug` | `localhost` không nhận kết nối từ ngoài container; cloud platform cấp `PORT` động và không dùng `reload=True` ở prod |
| **Health check** | Không có (`GET /health` → `404`) | `GET /health` (liveness) & `GET /ready` (readiness, trả `503` nếu chưa sẵn sàng) | Platform/load balancer dùng để tự restart container lỗi và chỉ route traffic khi app sẵn sàng |
| **Logging** | `print()` tự do, in cả `OPENAI_API_KEY` ra stdout | `logging` định dạng JSON có `time`, `level`, `msg`, không log secret | Log aggregator (Datadog, Loki) dễ tìm kiếm, lọc theo trường JSON và đảm bảo an toàn thông tin |
| **Shutdown** | Tắt đột ngột, không bắt `SIGTERM` | `lifespan` + `signal.SIGTERM` handler, hạ `is_ready=False` rồi mới dừng | Không làm rớt request đang xử lý (in-flight requests) khi deploy phiên bản mới hoặc scale down |
| **API Input & CORS** | Nhận `question` qua query param, không có CORS | Nhận JSON body, validate trả `422`, cấu hình `CORSMiddleware` | Tránh lộ prompt/dữ liệu trên URL log và kiểm soát origin được phép gọi API |

**Câu hỏi thảo luận:**
1. **Push code có API key hardcode lên GitHub public:** Các bot quét secret tự động trên GitHub sẽ phát hiện trong vài giây/phút và dùng key đó để gọi API trái phép, gây thiệt hại chi phí lớn hoặc rò rỉ dữ liệu. Ngoài ra, chỉ xóa file ở commit sau là chưa đủ vì key vẫn nằm trong lịch sử Git — bắt buộc phải **revoke/rotate key ngay lập tức**.
2. **Tại sao stateless quan trọng khi scale:** Khi chạy nhiều instance sau load balancer, các request liên tiếp của cùng một user có thể rơi vào các instance khác nhau. Nếu lưu state trong RAM của từng instance, instance sau sẽ mất ngữ cảnh/session của user và mất dữ liệu khi container restart.
3. **12-Factor "dev/prod parity" trong thực tế:** Giữ môi trường dev, staging và production giống nhau nhất có thể về runtime (ví dụ cùng dùng Docker/WSL2 Linux, cùng Python version, cùng cách đọc env vars và các dịch vụ phụ trợ như Redis) để loại bỏ lỗi *"it works on my machine"*.

---

## Part 2: Docker

### Exercise 2.1: Dockerfile cơ bản (`02-docker/develop/Dockerfile`)

1. **Base image là gì?**
   `python:3.11` (dòng 8) — bản phân phối Python đầy đủ (Debian + nhiều công cụ build hệ thống nên dung lượng lớn, ~1 GB+).
2. **Working directory là gì?**
   `/app` (dòng 11: `WORKDIR /app`). Mọi lệnh `COPY`, `RUN`, `CMD` sau đó đều thực thi tương đối từ `/app`.
3. **Tại sao `COPY requirements.txt` phải đứng trước `COPY . .` (hoặc `COPY app.py`)?**
   Để tận dụng **Docker layer cache**. `requirements.txt` ít thay đổi hơn mã nguồn; khi chỉ sửa code (`app.py`), Docker tái sử dụng lại layer `RUN pip install -r requirements.txt` đã cache ở trên mà không cần cài lại toàn bộ thư viện từ đầu.
4. **`CMD` vs `ENTRYPOINT` khác nhau thế nào?**
   - `CMD` cung cấp lệnh/tham số mặc định và **có thể bị ghi đè** dễ dàng bằng tham số truyền thêm khi `docker run <image> <command>`.
   - `ENTRYPOINT` cố định file thực thi chính của container; tham số truyền thêm ở `docker run` sẽ được nối vào làm đối số cho `ENTRYPOINT` (chỉ bị ghi đè nếu dùng cờ `--entrypoint`).

---

### Exercise 2.2 & 2.3: Build, Run và So sánh Multi-stage Build

**Phân tích `02-docker/production/Dockerfile`:**
- **Stage 1 (`builder`, từ `python:3.11-slim`):** Cài build tools (`gcc`, `libpq-dev`) để biên dịch dependencies và chạy `pip install --no-cache-dir --user -r requirements.txt` vào `/root/.local`. Image stage này không được đưa vào sản phẩm cuối.
- **Stage 2 (`runtime`, từ `python:3.11-slim`):** Tạo user non-root `appuser`, chỉ `COPY --from=builder /root/.local /home/appuser/.local` (lấy thư viện đã cài) và source code (`main.py`, `utils/mock_llm.py`), bỏ lại toàn bộ `gcc`, `libpq-dev`, cache build. Cấu hình `HEALTHCHECK` và chạy `uvicorn` với 2 workers dưới quyền `appuser`.
- **Tại sao image nhỏ hơn?** Dùng base image `slim` gọn nhẹ hơn bản `python:3.11` đầy đủ và loại bỏ hoàn toàn các trình biên dịch (`gcc`, dev headers) khỏi stage `runtime`.

**Kết quả đo kích thước image thực tế (`docker images`):**
```text
REPOSITORY:TAG                        SIZE
my-agent:develop                      1.67GB
my-agent:advanced                     240MB
```
- **Develop:** `1.67 GB` (~`1670 MB`)
- **Production (`advanced`):** `240 MB`
- **Chênh lệch:** Giảm `1430 MB` (**giảm ~85.6%**, nhỏ hơn gần 7 lần).

**Kết quả chạy & test container `my-agent:advanced` (cổng `8001:8000`):**
```text
HTTP/1.1 200 OK
content-type: application/json

{"status":"ok","uptime_seconds":1.4,"version":"2.0.0","timestamp":"2026-11-26T08:25:23.704143"}
```

---

### Exercise 2.4: Docker Compose stack (`02-docker/production/docker-compose.yml`)

**Sơ đồ kiến trúc các service:**
```text
Client (HTTP :80 / :443)
   │
   ▼
[ nginx ] (nginx:alpine, reverse proxy & security headers)
   │  (internal bridge network)
   ▼
[ agent ] (FastAPI + uvicorn, port 8000, depends_on redis & qdrant healthy)
   ├──► [ redis ]  (redis:7-alpine, port 6379, cache / session / rate limit)
   └──► [ qdrant ] (qdrant/qdrant:v1.9.0, port 6333, vector database)
```

**Các service khởi chạy và giao tiếp:**
- Chạy `docker compose -p k5-day14-docker up -d --build` (sau khi tạo `.env.local`).
- Tất cả 4 service (`nginx`, `agent`, `redis`, `qdrant`) đều ở trạng thái `Up` / `healthy`, giao tiếp qua mạng nội bộ `internal` (bridge); chỉ `nginx` mở cổng `80`/`443` ra ngoài.

**Kết quả test qua Nginx (`http://127.0.0.1`):**
```bash
$ curl -sS -i http://127.0.0.1/health
HTTP/1.1 200 OK
Server: nginx
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block

{"status":"ok","uptime_seconds":7.1,"version":"2.0.0","timestamp":"2026-11-26T08:27:01.997843"}

$ curl -sS -i -X POST http://127.0.0.1/ask -H "Content-Type: application/json" -d '{"question": "Explain microservices"}'
HTTP/1.1 200 OK
Server: nginx

{"answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé."}
```

---

## Part 4: API Security

### Exercise 4.1: API Key authentication (`04-api-gateway/develop/app.py`)

- **Kiểm tra ở đâu:** Dependency `verify_api_key` (dòng 39–54) dùng `APIKeyHeader(name="X-API-Key", auto_error=False)` và được inject vào `POST /ask` qua `_key: str = Depends(verify_api_key)` (dòng 70).
- **Thiếu hoặc sai key:** Thiếu header `X-API-Key` → `401 Unauthorized`; sai key → `403 Forbidden`.
- **Cách rotate key:** Không hardcode key; đọc từ biến môi trường `AGENT_API_KEY`. Khi rotate ở production, hỗ trợ danh sách key hợp lệ (key cũ + key mới) trong thời gian chuyển tiếp, cập nhật env var ở client rồi gỡ key cũ.

**Output test bản `develop` (chạy với `AGENT_API_KEY=secret-key-123`):**
```bash
# Không có key → 401
$ curl -sS -i -X POST http://127.0.0.1:8000/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
HTTP/1.1 401 Unauthorized
content-type: application/json

{"detail":"Missing API key. Include header: X-API-Key: <your-key>"}

# Có key → 200
$ curl -sS -i -X POST http://127.0.0.1:8000/ask -H "X-API-Key: secret-key-123" -H "Content-Type: application/json" -d '{"question":"Hello"}'
HTTP/1.1 200 OK
content-type: application/json

{"question":"Hello","answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic."}
```

---

### Exercise 4.2 & 4.3: JWT Authentication & Rate Limiting (`04-api-gateway/production/`)

- **JWT flow (`auth.py`):** Client gọi `POST /auth/token` (`student` / `demo123`) để nhận JWT ký bằng `HS256` chứa `sub`, `role`, `iat`, `exp` (60 phút). Khi gọi `/ask`, `verify_token` giải mã và kiểm tra chữ ký/hạn dùng từ header `Authorization: Bearer <token>` mà không cần truy vấn DB mỗi request.
- **Rate limiter (`rate_limiter.py`):**
  - **Algorithm:** **Sliding Window Counter** dùng `deque` lưu timestamp của từng `user_id`; mỗi request mới sẽ `popleft()` các timestamp cũ hơn `now - 60s`.
  - **Giới hạn & phân quyền:** `user` (`student`) giới hạn **10 requests/phút** (`rate_limiter_user`); `admin` (`teacher`) giới hạn **100 requests/phút** (`rate_limiter_admin`) dựa theo `role` trong JWT payload.

**Output `curl -i` kiểm thử bản `production` (token đã che):**
```bash
# 1. Lấy token ở POST /auth/token (student / demo123)
Token received (length=168, masked=eyJhbGciOiJI...<REDACTED>)

# 2. Không có token → 401 Unauthorized
HTTP/1.1 401 Unauthorized
www-authenticate: Bearer
x-content-type-options: nosniff
x-frame-options: DENY
x-xss-protection: 1; mode=block
referrer-policy: strict-origin-when-cross-origin

{"detail":"Authentication required. Include: Authorization: Bearer <token>"}

# 3. Có token hợp lệ (trong hạn mức 10 req/phút) → 200 OK
HTTP/1.1 200 OK
x-content-type-options: nosniff
x-frame-options: DENY

{"question":"Sample 1","answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé.","usage":{"requests_remaining":8,"budget_remaining_usd":3.5e-05}}

# 4. Vòng lặp vượt quá 10 req/phút (Request 10, 11, 12) → 429 Too Many Requests
HTTP/1.1 429 Too Many Requests
x-ratelimit-limit: 10
x-ratelimit-remaining: 0
x-ratelimit-reset: 1795682168
retry-after: 59

{"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":59}}
```

---

### Exercise 4.4: Cost Guard (`cost_guard.py`)

- **Cách hoạt động:**
  1. **Trước khi gọi LLM (`check_budget`):** Kiểm tra chi phí tích lũy theo ngày/tháng của user (vượt ngưỡng → trả **`402 Payment Required`**) và ngân sách tổng hệ thống (vượt ngưỡng → trả **`503`**); cảnh báo log khi chạm 80% budget.
  2. **Sau khi gọi LLM (`record_usage`):** Tính chi phí từ `input_tokens * PRICE_PER_1K_INPUT_TOKENS + output_tokens * PRICE_PER_1K_OUTPUT_TOKENS` rồi cộng dồn.
  3. **Production với Redis:** Không lưu trong RAM (`dict`) vì mất dữ liệu khi restart và không đồng bộ giữa nhiều instance. Dùng Redis key theo user/chu kỳ (ví dụ `budget:{user_id}:{YYYY-MM}`), đọc bằng `GET`, cộng dồn nguyên tử bằng `INCRBYFLOAT` và đặt `EXPIRE` (32 ngày cho chu kỳ tháng).
- **Tại sao cần cả Rate Limit và Cost Guard?**
  - **Rate limit** bảo vệ **tốc độ/tài nguyên hệ thống** (chống spam/DDoS theo số request/phút).
  - **Cost guard** bảo vệ **ngân sách tiền LLM** (một request hợp lệ nhưng prompt/output rất dài vẫn có thể đốt nhiều token và tiền dù chưa chạm rate limit).

---

## Part 5: Scaling & Reliability

### Exercise 5.1 & 5.2: Health checks & Graceful shutdown (`05-scaling-reliability/develop/app.py`)

- **Phân biệt Liveness (`/health`) vs Readiness (`/ready`):**
  - `GET /health` (**Liveness probe**): Kiểm tra tiến trình app còn sống hay bị treo. Luôn trả `200` khi process đang chạy (kể cả khi dependency như Redis tạm mất kết nối, tránh platform restart container liên tục một cách vô ích).
  - `GET /ready` (**Readiness probe**): Kiểm tra app đã sẵn sàng nhận traffic chưa (ví dụ đã khởi tạo xong, kết nối được Redis/DB, chưa vào trạng thái shutdown). Nếu chưa sẵn sàng thì trả `503` để load balancer tạm ngừng route traffic vào instance đó.
- **Graceful shutdown:** Khi nhận `SIGTERM` (signal 15), app chuyển `_is_ready = False` để ngừng nhận request mới, theo dõi `_in_flight_requests` qua middleware `track_requests` và chờ request đang xử lý hoàn thành (tối đa 30s) trước khi tắt.

**Output kiểm thử bản `develop`:**
```bash
$ curl -sS -i http://127.0.0.1:8000/health
HTTP/1.1 200 OK
{"status":"ok","uptime_seconds":2.4,"version":"1.0.0","environment":"development","timestamp":"2026-11-26T08:41:19.020292+00:00","checks":{"memory":{"status":"ok","note":"psutil not installed"}}}

$ curl -sS -i http://127.0.0.1:8000/ready
HTTP/1.1 200 OK
{"ready":true,"in_flight_requests":1}
```
**Log khi gửi `kill -TERM`:**
```text
INFO:     Shutting down
INFO:     Waiting for application shutdown.
2026-11-26 08:41:19,176 INFO 🔄 Graceful shutdown initiated...
2026-11-26 08:41:19,176 INFO ✅ Shutdown complete
INFO:     Application shutdown complete.
2026-11-26 08:41:19,176 INFO Received signal 15 — uvicorn will handle graceful shutdown
```

---

### Exercise 5.3–5.5: Stateless Design, Load Balancing & Test (`05-scaling-reliability/production/`)

- **Vì sao state trong RAM hỏng khi chạy nhiều instance?**
  Mỗi container `agent` có bộ nhớ RAM riêng biệt. Nếu lưu `conversation_history` trong RAM của instance 1, khi Nginx phân tán request tiếp theo của cùng session sang instance 2, instance 2 sẽ không thấy lịch sử trò chuyện trước đó. Đưa state ra **Redis** dùng chung giúp mọi instance đều đọc/ghi cùng một nguồn dữ liệu và không mất session khi restart.
- **Kết quả chạy stack 3 replicas (`docker compose -p k5-day14-scaling up -d --build --scale agent=3`) và `python test_stateless.py` (cổng `8080`, `POST /chat`):**
  ```text
  Session ID: 5cc9425b-25d4-4f0d-9c7c-3a27c3fdf3b4

  Request 1: [instance-c4266a]  Q: What is Docker?
  Request 2: [instance-26b5b7]  Q: Why do we need containers?
  Request 3: [instance-041903]  Q: What is Kubernetes?
  Request 4: [instance-c4266a]  Q: How does load balancing work?
  Request 5: [instance-26b5b7]  Q: What is Redis used for?

  Total requests: 5
  Instances used: {'instance-041903', 'instance-26b5b7', 'instance-c4266a'}
  ✅ All requests served despite different instances!

  --- Conversation History ---
  Total messages: 10
  ✅ Session history preserved across all instances via Redis!
  ```
- **Giải thích:** Nginx cân bằng tải đều 5 request qua cả 3 instance khác nhau (`instance-c4266a`, `instance-26b5b7`, `instance-041903`), nhưng `GET /chat/{session_id}/history` vẫn trả về trọn vẹn 10 tin nhắn (5 câu hỏi + 5 câu trả lời) nhờ lưu trữ stateless trong Redis.

---

## Part 6: Final Project (`06-lab-complete/`)

Kết quả chạy `python check_production_ready.py` (đầy đủ với Redis & Docker trong WSL2):

```text
================================================================
  📋 Checklist Deliverable
     ✅ Dockerfile (multi-stage, < 500 MB)
     ✅ docker-compose.yml (agent + redis)
     ✅ .dockerignore
     ✅ Health check endpoint (GET /health)
     ✅ Readiness endpoint (GET /ready)
     ✅ API Key authentication
     ✅ Rate limiting
     ✅ Cost guard
     ✅ Config từ environment variables
     ✅ Structured logging
     ✅ Graceful shutdown
     ✅ Public URL ready (Railway / Render config)
----------------------------------------------------------------
  Result: 12/12 tiêu chí đạt · 42/42 checks passed
  🎉 PASS 12/12 — PRODUCTION READY! Deploy nào!
================================================================
```

- Kích thước Docker image production đo thực tế: **`297 MB`** (< 500 MB).
- Thời gian `docker stop` (graceful shutdown): **`0.9s`** (exit code `0`).

---

## Part 3: Cloud Deployment

### Exercise 3.1 & 3.2: Deploy Agent Part 6 lên Render

- **Nền tảng chính:** Render (`Web Service` với `Docker` + `Key Value / Redis` nội bộ).
- **Public URL:** `https://day14-production-agent-7rjo.onrender.com`
- **Screenshots:** `screenshots/dashboard.png`, `screenshots/running.png`, `screenshots/test.png` (xem chi tiết tại [DEPLOYMENT.md](DEPLOYMENT.md)).
- **Kết quả tự kiểm tra trên URL public:**
  ```text
  GET  /health              → HTTP/2 200  {"status":"ok","environment":"production"}
  GET  /ready               → HTTP/2 200  {"status":"ready"}
  POST /ask (no X-API-Key)  → HTTP/2 401  {"detail":"Invalid or missing API key"}
  ```

**So sánh `railway.toml` với `render.yaml`:**
- **Định dạng & phạm vi:** `railway.toml` dùng định dạng TOML để cấu hình build/deploy cho **một service** (`[build] builder = "DOCKERFILE"`, `[deploy] healthcheckPath = "/health"`), còn các biến môi trường và Redis thường cấu hình qua Railway Dashboard hoặc CLI (`railway variables set`). Trong khi đó, `render.yaml` dùng định dạng YAML theo mô hình **Blueprint (Infrastructure as Code)**, khai báo được danh sách nhiều `services:` cùng lúc (`type: web`, `runtime: docker`, `healthCheckPath: /health`) và khai báo rõ từng `envVars` ngay trong file (hỗ trợ `generateValue: true` để tự sinh secret hoặc `sync: false` để nhập tay trên dashboard mà không commit secret vào Git).
- **Vì sao khác nhau:** Railway ưu tiên trải nghiệm CLI/UI linh hoạt cho từng service rời, còn Render Blueprint hướng tới việc mô tả toàn bộ hạ tầng + biến môi trường một cách khai báo (declarative) trong 1 file để tái tạo nhanh từ GitHub.

### Exercise 3.3: GCP Cloud Run CI/CD Pipeline (`03-cloud-deployment/production-cloud-run/`)

Đọc `cloudbuild.yaml` và `service.yaml`, pipeline CI/CD trên Google Cloud đi qua **4 bước tuần tự** khi có commit mới lên nhánh `main`:
1. **`test`:** Dùng image `python:3.11-slim` cài dependencies và chạy `pytest tests/`; nếu test fail thì pipeline dừng ngay.
2. **`build`:** Build Docker image từ source code, gắn 2 tag (`$COMMIT_SHA` để truy vết phiên bản và `latest` để làm `--cache-from`).
3. **`push`:** Đẩy Docker image đã gắn tag lên Google Container Registry (`gcr.io/$PROJECT_ID/ai-agent`).
4. **`deploy`:** Gọi `gcloud run deploy` để triển khai image theo `$COMMIT_SHA` lên Cloud Run (`asia-southeast1`), cấu hình autoscaling (`min-instances=1` tránh cold start, `max-instances=10`), tài nguyên (`512Mi` RAM, `1` CPU), và nạp secret (`OPENAI_API_KEY`) an toàn từ Google Secret Manager (tương tự cơ chế `secretKeyRef` và `livenessProbe` `/health`, `startupProbe` `/ready` trong `service.yaml`).

---

## AI log

| Mục | Hỏi AI gì | Nhận được gì | Tự sửa & tự kiểm tra |
|---|---|---|---|
| Setup & Part 1 | Hướng dẫn thiết lập môi trường WSL2 và thực hiện kiểm thử Part 1 (`develop` vs `production`) | Tạo virtualenv `~/.venvs/day14` trong WSL2 Ubuntu, cài `requirements.txt`, chạy thực tế cả hai server và tổng hợp bảng so sánh | Đã trực tiếp chạy `develop/app.py` và `production/app.py` trong WSL2, kiểm tra các endpoint `/`, `/ask`, `/health`, `/ready`, đối chiếu đúng số dòng code trong repo |
| Part 2 (Docker) | Hỗ trợ cấu hình WSL Integration, khắc phục lỗi Docker Hub 429 & đầy ổ C:, build & test stack Part 2 | Build 2 image `my-agent:develop` và `my-agent:advanced`, chạy stack Compose `k5-day14-docker` qua Nginx và tính % giảm dung lượng | Đã dọn ổ C:, reset WSL2, chạy `docker images` đo thực tế (`1.67GB` vs `240MB`), gọi thực tế `/health` và `/ask` qua Nginx trả `200 OK`, và dọn stack sau khi test |
| Part 4 (API Security) | Hỗ trợ cài `pyjwt` và chạy kiểm thử `develop` & `production` cho Exercise 4.1–4.4 | Chạy kiểm thử API Key, lấy JWT (`student`/`demo123`), chạy vòng lặp kích hoạt `429` và tổng hợp phần giải thích Cost Guard | Đã tự kiểm tra output thật cho cả 3 mã HTTP `401`, `200`, `429`, che JWT token trước khi ghi vào `MISSION_ANSWERS.md` |
| Part 5 (Scaling & Reliability) | Hỗ trợ kiểm thử `develop` (`/health`, `/ready`, `SIGTERM`) và chạy Compose stack `k5-day14-scaling` (`--scale agent=3`) + `test_stateless.py` | Bật đủ 3 agent replicas + Redis + Nginx, chạy `test_stateless.py` xác nhận 5 requests chia đều cho 3 instance và giữ đủ 10 messages trong Redis | Đã tự kiểm tra `docker compose -p k5-day14-scaling ps` tất cả healthy, đối chiếu output `test_stateless.py`, rồi chạy `down` và xóa `.env.local` tạm |
| Part 6 (Final Project) | Triển khai toàn bộ các module TODO trong `06-lab-complete/` (`config`, `auth`, `redis_client`, `rate_limiter`, `cost_guard`, `main`, `Dockerfile`, `docker-compose.yml`, `.dockerignore`, `.env.example`, `render.yaml`) | Hoàn thiện code đạt chuẩn production, multi-stage Docker `< 500 MB` (`297 MB`), sliding window & cost guard trên Redis | Đã chạy trực tiếp `check_production_ready.py` đầy đủ với Redis & Docker trong WSL2 và đạt `PASS 12/12` (`42/42 checks passed`) |
| Part 3 (Cloud Deployment) | Hướng dẫn cấu hình Redis & Web Service trên Render, kiểm tra URL public, so sánh `railway.toml` vs `render.yaml` và phân tích Cloud Run CI/CD | Deploy thành công `06-lab-complete/` lên Render, tạo `DEPLOYMENT.md` và hoàn thiện mục Part 3 | Đã gọi trực tiếp `https://day14-production-agent-7rjo.onrender.com` kiểm tra `/health` (200), `/ready` (200), `/ask` không key (401) |
