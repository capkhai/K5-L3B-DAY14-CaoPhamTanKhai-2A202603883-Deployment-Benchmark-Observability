# Day 14 Lab — Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found in `01-localhost-vs-production/develop/app.py`

1. **Hardcode secrets trong mã nguồn (dòng 17–18):** `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"` và `DATABASE_URL = "postgresql://admin:password123@localhost:5432/mydb"`. Nếu push lên GitHub public, khóa và mật khẩu DB bị lộ ngay lập tức và nằm mãi trong lịch sử Git.
2. **Không có cấu hình qua biến môi trường (dòng 21–22, 51–52):** `DEBUG = True`, `MAX_TOKENS = 500`, `host="localhost"`, `port=8000` đều gán cứng; không thể thay đổi giữa dev/staging/production mà không sửa code, và trên cloud platform (Railway/Render) `PORT` được inject qua env var.
3. **Dùng `print()` thay cho structured logging và log lộ secret (dòng 33–38):** `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")` vừa khó parse/search trên hệ thống log tập trung (Loki, Datadog), vừa làm lộ API key ra log.
4. **Không có endpoint health check & readiness (`/health`, `/ready`):** Cloud platform / load balancer không biết container còn sống hay đã sẵn sàng nhận traffic để tự động restart hoặc ngắt route traffic khi gặp sự cố.
5. **Không có graceful shutdown (không xử lý `SIGTERM` / `lifespan`):** Khi container bị dừng/redeploy, process bị cắt ngang đột ngột làm mất request đang xử lý và không đóng kết nối tài nguyên sạch sẽ.
6. **Bind `host="localhost"` và bật `reload=True` (dòng 51, 53):** Bind `localhost` khiến container không nhận được traffic từ bên ngoài; `reload=True` tốn tài nguyên và không an toàn cho production.

### Exercise 1.2: Kết quả chạy bản `develop` và `production`

- **Bản `develop`:** chạy `python app.py` trong `01-localhost-vs-production/develop` và gọi `POST /ask?question=Hello` thành công.
- **Bản `production`:** chạy `python app.py` trong `01-localhost-vs-production/production` (đọc cấu hình từ `.env`), kết quả kiểm tra 3 endpoint:
  - `GET /health`:
    ```json
    {"status":"ok","uptime_seconds":11.8,"version":"1.0.0","environment":"development","timestamp":"2026-11-26T07:58:09.634873+00:00"}
    ```
  - `GET /ready`:
    ```json
    {"ready":true}
    ```
  - `POST /ask` (`{"question": "Hello"}`):
    ```json
    {"question":"Hello","answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé.","model":"gpt-4o-mini"}
    ```

### Exercise 1.3: Bảng so sánh `develop` vs `production`

| Feature | Develop (`develop/app.py`) | Production (`production/app.py` + `config.py`) | Tại sao quan trọng? |
|---|---|---|---|
| **Config & Secrets** | Hardcode `OPENAI_API_KEY`, `DATABASE_URL`, `PORT=8000` ngay trong code | Tập trung trong `Settings` (`config.py`), đọc từ biến môi trường (`os.getenv`), fail-fast khi thiếu secret ở `production` (`validate()`), chỉ commit `.env.example` | Tránh lộ secret lên Git; cùng một bản build chạy được trên dev/staging/prod chỉ bằng cách đổi env vars (12-Factor III). |
| **Health / Readiness check** | Không có | Có `GET /health` (liveness, trả `status`, `uptime_seconds`, `version`) và `GET /ready` (readiness, trả `503` khi `is_ready=False`) | Platform biết khi nào container chết để restart (`/health`) và load balancer biết khi nào sẵn sàng nhận traffic (`/ready`) tránh gửi request vào instance chưa khởi động xong. |
| **Logging** | `print()` tự do, thậm chí in cả `OPENAI_API_KEY` | Structured JSON logging qua `logging`, không log secret, ghi rõ `event`, `question_length`, `client_ip` | Log dạng JSON dễ thu thập, tìm kiếm, lọc và vẽ biểu đồ trên công cụ tập trung (Datadog, Loki, Cloud Logging) mà không làm rò rỉ dữ liệu nhạy cảm. |
| **Shutdown** | Tắt đột ngột, không có lifecycle | Dùng `lifespan` của FastAPI + bắt tín hiệu `SIGTERM` (`signal.signal`), đặt `is_ready = False` rồi chờ hoàn thành request đang chạy | Không làm đứt request của người dùng khi deploy bản mới hoặc scale down (zero-downtime deployment). |
| **Network binding & Debug** | `host="localhost"`, `port=8000`, `reload=True` cứng | `host="0.0.0.0"`, `port=int(os.getenv("PORT", 8000))`, `reload=settings.debug` | `0.0.0.0` cho phép nhận kết nối từ bên ngoài container/cloud; tắt `reload` ở production giúp ổn định và tiết kiệm CPU/RAM. |

**Ví dụ đối chiếu cụ thể dòng code đã sửa:**
- `develop/app.py` dòng 17 (`OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"`) → `production/config.py` dòng 27 (`openai_api_key: str = field(default_factory=lambda: os.getenv("OPENAI_API_KEY", ""))`).

---

## Part 2: Docker

### Exercise 2.1: Dockerfile questions (`02-docker/develop/Dockerfile`)

1. **Base image:** `python:3.11` — image Linux (Debian) cài sẵn Python 3.11 đầy đủ kèm các công cụ build (`gcc`, thư viện hệ thống...), kích thước lớn (~1 GB).
2. **Working directory:** `WORKDIR /app` — thư mục làm việc mặc định bên trong container cho mọi lệnh `COPY`, `RUN`, `CMD` phía sau.
3. **Tại sao `COPY requirements.txt` và `RUN pip install` trước khi `COPY` source code:** Để tận dụng **Docker layer cache**. `requirements.txt` ít đổi hơn `app.py`; khi sửa source code rồi build lại, Docker dùng lại cache của bước `pip install` và chỉ chạy lại từ bước `COPY app.py`, giúp build nhanh hơn rất nhiều.
4. **`CMD` vs `ENTRYPOINT`:**
   - `CMD` là lệnh chạy **mặc định**, có thể bị ghi đè dễ dàng khi truyền lệnh mới ở cuối `docker run <image> <cmd>`.
   - `ENTRYPOINT` là lệnh chính **cố định** luôn được thực thi; đối số truyền ở `docker run` được nối thêm vào sau `ENTRYPOINT` (chỉ ghi đè được khi dùng `--entrypoint`).

### Exercise 2.2: Build và chạy container `develop`

- Build từ gốc repo: `docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .`
- Chạy container và test `GET /health` + `POST /ask?question=What%20is%20Docker?`:
  ```json
  {"status":"ok","uptime_seconds":26.8,"container":true}
  {"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}
  ```

### Exercise 2.3: Multi-stage build & Image size comparison

- **Stage 1 (`builder`):** Dùng `python:3.11-slim`, cài công cụ biên dịch (`gcc`, `libpq-dev`) và cài Python dependencies bằng `pip install --no-cache-dir --user -r requirements.txt` vào `/root/.local`.
- **Stage 2 (`runtime`):** Bắt đầu từ `python:3.11-slim` sạch, chỉ copy các package đã cài (`/root/.local` → `/home/appuser/.local`) cùng `main.py` và `utils/mock_llm.py`, chạy bằng user non-root `appuser` và khai báo `HEALTHCHECK`.
- **Tại sao image nhỏ hơn:** Không mang theo `gcc`, `libpq-dev`, apt cache hay các file trung gian của bước build, đồng thời dùng base `python:3.11-slim` thay vì `python:3.11` đầy đủ.

**Kết quả đo thực tế bằng `docker images my-agent`:**
- **Develop (`my-agent:develop`):** Disk usage `1.67 GB` (1670 MB) · Content size `426 MB`
- **Production (`my-agent:advanced`):** Disk usage `240 MB` · Content size `57.4 MB`
- **Chênh lệch:** Giảm **85.6%** disk usage (giảm **86.5%** content size — nhỏ hơn gần 7 lần).

### Exercise 2.4: Docker Compose stack (`02-docker/production/docker-compose.yml`)

**Sơ đồ kiến trúc (mạng nội bộ `internal`):**
```text
Client ──▶ Nginx (:80 / :443, reverse proxy + rate limit 10r/s)
             └──▶ Agent (:8000, không expose port ra host)
                    ├──▶ Redis (:6379, cache / rate limit, volume redis_data)
                    └──▶ Qdrant (:6333, vector DB, volume qdrant_data)
```

- Các service chạy: `nginx`, `agent`, `redis`, `qdrant` (tất cả đều `healthy` trên mạng bridge `internal`).
- Kết quả test qua Nginx (`http://localhost`):
  - `GET /health`:
    ```json
    {"status":"ok","uptime_seconds":31.8,"version":"2.0.0","timestamp":"2026-11-26T08:29:44.092052"}
    ```
  - `POST /ask` (`{"question": "Explain microservices"}`):
    ```json
    {"answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}
    ```

---

## Part 3: Cloud Deployment

*(Thực hiện ở CP6 sau khi hoàn thành Part 6)*

---

## Part 4: API Security

### Exercise 4.1: API Key Authentication (`04-api-gateway/develop/app.py`)

- **Kiểm tra ở đâu:** Trong dependency `verify_api_key` dùng `APIKeyHeader(name="X-API-Key", auto_error=False)` gắn vào endpoint `POST /ask` qua `Depends(verify_api_key)`.
- **Khi thiếu / sai key:** Thiếu header trả `401 Unauthorized`; sai key trả `403 Forbidden` (ở Part 6 chuẩn hóa cả hai về `401`).
- **Cách rotate key:** Thay đổi giá trị biến môi trường `AGENT_API_KEY` trên nền tảng cloud/secret manager và restart/redeploy; trong production lớn có thể hỗ trợ đồng thời 2 key (cũ + mới) trong thời gian chuyển tiếp rồi thu hồi key cũ.

**Kết quả test `curl -i`:**
- Không có `X-API-Key` → **401 Unauthorized**:
  ```http
  HTTP/1.1 401 Unauthorized
  date: Thu, 26 Nov 2026 08:36:33 GMT
  server: uvicorn
  content-type: application/json

  {"detail":"Missing API key. Include header: X-API-Key: <your-key>"}
  ```
- Có `X-API-Key: secret-key-123` → **200 OK**:
  ```http
  HTTP/1.1 200 OK
  date: Thu, 26 Nov 2026 08:36:34 GMT
  server: uvicorn
  content-type: application/json
  ```

### Exercise 4.2 & 4.3: JWT Authentication & Rate Limiting (`04-api-gateway/production/`)

- **JWT flow (`auth.py`):** Client đăng nhập tại `POST /auth/token` (`student` / `demo123`) → server ký JWT (`HS256`, hạn 60 phút, payload chứa `sub`, `role`, `iat`, `exp`). Các request sau gửi `Authorization: Bearer <REDACTED_TOKEN>` → `verify_token` kiểm tra chữ ký và hạn dùng mà không cần truy vấn DB.
- **Rate limiting (`rate_limiter.py`):**
  - **Thuật toán:** Sliding Window Counter dùng `deque` lưu timestamp các request trong cửa sổ `window_seconds = 60` giây, xoá các timestamp cũ hơn `now - 60`.
  - **Giới hạn:** User thường `10 requests/phút` (`rate_limiter_user`), Admin `100 requests/phút` (`rate_limiter_admin`).
  - **Bypass / nâng hạn mức cho admin:** Dựa vào `role` giải mã từ JWT để chọn limiter riêng (hoặc bỏ qua check nếu `role == "admin"`).

**Kết quả test JWT + vòng lặp Rate Limit (token đã che):**
- Gọi hợp lệ với `Authorization: Bearer <REDACTED_TOKEN>` → **200 OK**:
  ```http
  HTTP/1.1 200 OK
  content-type: application/json

  {"question":"Test 429","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.","usage":{"requests_remaining":9,"budget_remaining_usd":3.5e-05}}
  ```
- Gọi 10 lần liên tiếp + lần thứ 11 → **429 Too Many Requests**:
  ```text
  Request 1 : HTTP 200
  ...
  Request 10 : HTTP 200
  ```
  ```http
  HTTP/1.1 429 Too Many Requests
  date: Thu, 26 Nov 2026 08:46:39 GMT
  server: uvicorn
  x-ratelimit-limit: 10
  x-ratelimit-remaining: 0
  x-ratelimit-reset: 1795682859
  retry-after: 59
  content-type: application/json

  {"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":59}}
  ```

### Exercise 4.4: Cost Guard (`cost_guard.py`)

- **Cách thiết kế với Redis (stateless, dùng cho production/Part 6):**
  - Dùng key theo tháng hoặc theo ngày trong Redis (ví dụ `budget:{user_id}:{YYYY-MM}` hoặc `cost:daily:{YYYY-MM-DD}`).
  - Trước khi gọi LLM (`check_budget`): đọc tổng chi phí hiện tại từ Redis; nếu `current + estimated_cost > limit` (hoặc `current >= daily_budget_usd`) thì trả **`402 Payment Required`** (hoặc `503` khi vượt ngân sách toàn hệ thống).
  - Sau khi gọi LLM (`record_cost`): tính chi phí từ `input_tokens` và `output_tokens` theo đơn giá model, cộng dồn nguyên tử bằng `INCRBYFLOAT` và đặt `EXPIRE` để tự reset sang chu kỳ mới.
- **Vì sao cần cả Rate Limit và Cost Guard?**
  - **Rate limit** bảo vệ **tốc độ & tài nguyên server** (chống spam/DDoS theo số request/phút).
  - **Cost guard** bảo vệ **ngân sách tiền LLM** (một user vẫn có thể gửi dưới ngưỡng rate limit nhưng mỗi câu prompt cực dài/chọn model đắt gây đốt tiền liên tục). Cần cả hai lớp bảo vệ độc lập.

---

## Part 5: Scaling & Reliability

### Exercise 5.1: Health check (`/health`) vs Readiness check (`/ready`)

- **`GET /health` (Liveness probe):** Kiểm tra process ứng dụng còn sống không (không bị treo/deadlock). Nếu trả non-200 hoặc timeout, platform (Docker, Railway, Render, Kubernetes) sẽ tự **restart container**. Do đó `/health` phải luôn trả `200` khi process còn chạy, kể cả khi Redis tạm mất kết nối.
- **`GET /ready` (Readiness probe):** Kiểm tra instance đã **sẵn sàng nhận traffic** chưa (đã load xong model, chưa bước vào shutdown, và các dependency bắt buộc như Redis đang kết nối được qua `redis.ping()`). Nếu chưa sẵn sàng, trả **`503 Service Unavailable`** để Load Balancer tạm ngừng điều hướng request vào instance đó thay vì restart nó một cách vô ích.

### Exercise 5.2: Graceful shutdown (`05-scaling-reliability/develop/app.py`)

- Khi container bị dừng/redeploy, platform gửi tín hiệu **`SIGTERM`**.
- Cơ chế xử lý trong `develop/app.py`:
  1. Middleware `track_requests` đếm số request đang xử lý (`_in_flight_requests`).
  2. `signal.signal(signal.SIGTERM, handle_sigterm)` ghi log khi nhận tín hiệu, sau đó `uvicorn` (với `timeout_graceful_shutdown=30`) tự kích hoạt phần shutdown của `lifespan`.
  3. Trong `lifespan` shutdown: đặt `_is_ready = False` (để `/ready` trả `503`, Load Balancer ngừng gửi request mới), chờ `_in_flight_requests == 0` (tối đa 30s) để hoàn thành nốt các request đang chạy, rồi đóng kết nối và thoát với exit code `0`.

### Exercise 5.3: Stateless design (`05-scaling-reliability/production/app.py`)

- **Vì sao state trong RAM bị hỏng khi scale ngang:** Mỗi instance container có vùng nhớ RAM riêng biệt. Nếu lưu `conversation_history` trong dict Python trên RAM, request 1 của user đi vào Instance 1 (lưu ở RAM 1), request 2 bị Load Balancer chuyển sang Instance 2 → Instance 2 không có lịch sử hội thoại đó.
- **Giải pháp:** Tách toàn bộ session/history ra **Redis** (`save_session` dùng `setex` với TTL 3600s, `load_session` dùng `get`). Nhờ vậy mọi instance đều đọc/ghi cùng một nguồn dữ liệu chung và chịu được việc restart hoặc scale lên/xuống tự do.

### Exercise 5.4 & 5.5: Load balancing (Nginx + 3 instances) & `test_stateless.py`

Chạy stack với `docker compose -p k5-day14-scaling up -d --build --scale agent=3` (Nginx lắng nghe cổng `8080`, round-robin sang 3 instance agent dùng chung Redis), rồi chạy `python test_stateless.py`:

```text
Session ID: 69afb582-f986-4bf6-9376-63dc60247a33

Request 1: [instance-5b0b5c]  Q: What is Docker?
Request 2: [instance-5130e6]  Q: Why do we need containers?
Request 3: [instance-85e4f8]  Q: What is Kubernetes?
Request 4: [instance-5b0b5c]  Q: How does load balancing work?
Request 5: [instance-5130e6]  Q: What is Redis used for?

Total requests: 5
Instances used: {'instance-5b0b5c', 'instance-85e4f8', 'instance-5130e6'}
✅ All requests served despite different instances!

--- Conversation History ---
Total messages: 10
✅ Session history preserved across all instances via Redis!
```

**Giải thích kết quả:** 5 request liên tiếp của cùng một `session_id` được Nginx phân phối xoay vòng (round-robin) tới cả 3 instance khác nhau (`instance-5b0b5c`, `instance-5130e6`, `instance-85e4f8`). Khi gọi `GET /chat/{session_id}/history`, toàn bộ 10 tin nhắn (5 câu hỏi + 5 câu trả lời) vẫn còn đầy đủ vì toàn bộ trạng thái phiên được lưu tập trung trong Redis chứ không nằm trong RAM từng container.

---

## Part 6: Final Project — `check_production_ready.py` Output

*(Dán phần "📋 Checklist Deliverable" sau khi đạt `PASS 12/12` ở CP5)*

---

## AI Log

| Thời điểm | Phần | Hỏi AI gì | Nhận được gì | Tự sửa & tự kiểm tra thế nào |
|---|---|---|---|---|
| Day 14 | CP0–CP1 | Giải thích lỗi 422 / 404 khi gọi `/ask`, lỗi `ModuleNotFoundError: fastapi` do lệch `pip`/`python`, và tổng hợp câu trả lời Part 1 từ code đã đọc | Hướng dẫn dùng query param cho bản `develop`, tạo `.venv`, chạy `curl.exe` trên Windows/WSL2 và cấu trúc bảng so sánh Part 1 | Tự chạy `develop/app.py` và `production/app.py`, tự kiểm tra 3 endpoint `/health`, `/ready`, `/ask` nhận kết quả thực tế ở trên và đối chiếu trực tiếp dòng code trong 2 file. |
| Day 14 | CP2 | Hỏi cách giải quyết lỗi build Docker (sai build context), lỗi Docker Hub 429 Too Many Requests khi `docker compose up --build`, và giải thích 4 câu hỏi Dockerfile | Hướng dẫn build từ gốc repo, dùng `docker tag` + `--no-build` tận dụng image đã build để tránh 429, và giải thích layer cache / multi-stage | Tự chạy build 2 image, tự đo `docker images my-agent` (1.67GB vs 240MB), tự khởi động stack 4 container và gọi `/health`, `/ask` qua Nginx. |
| Day 14 | CP3 | Hỏi lệnh bật `.venv` và cách viết vòng lặp test rate limit trên PowerShell | Đường dẫn kích hoạt `.venv` và đoạn script PowerShell test 10 request + 429 | Tự chạy cả bản `develop` (API key: 401/200) và `production` (JWT + rate limit: 10×200 → 429), đối chiếu code `auth.py`, `rate_limiter.py`, `cost_guard.py`. |
| Day 14 | CP4 | Xác nhận vị trí Part 3 theo lộ trình K5, hỏi về test script và hướng dẫn chạy stack scale 3 instance | Giải thích lộ trình đưa Part 3 sang CP6 và cụm lệnh chạy `docker compose -p k5-day14-scaling --scale agent=3` + `test_stateless.py` | Tự chạy stack 3 instance + Redis + Nginx, tự chạy `test_stateless.py` xác nhận 3 instance khác nhau chia tải nhưng vẫn giữ đủ 10 tin nhắn trong Redis. |
