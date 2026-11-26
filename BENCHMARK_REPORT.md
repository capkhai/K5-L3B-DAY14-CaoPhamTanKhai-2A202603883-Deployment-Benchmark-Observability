# Benchmark & Observability Report — Day 14

> Mọi con số dưới đây được lấy trực tiếp từ các file JSON trong `reports/`, CSV trong `benchmark-results/` và ảnh chụp trong `screenshots/`.

**Họ tên:** Cao Phạm Tấn Khải — MSSV: 2A202603883  
**Ngày chạy:** 26/11/2026  
**Môi trường:** Windows 11 + WSL2 (Ubuntu, Python 3.14.4), Docker Desktop + Docker Compose v5.7.0

---

## 1. SLO của agent

| SLI | SLO | Lý do chọn ngưỡng này |
|---|---|---|
| p95 latency `/ask` | < 500 ms | 95% người dùng nhận phản hồi dưới nửa giây (trải nghiệm hội thoại mượt mà), phù hợp với baseline mock LLM (~180–230 ms ở tải thường). |
| Error rate `/ask` | < 2.0 % | Chấp nhận sai số nhỏ từ lỗi upstream ngẫu nhiên (`MOCK_ERROR_RATE=0.01`, ~1%), tương ứng error budget 2% để tránh alert fatigue khi có spike nhỏ. |
| Accuracy trên golden dataset | ≥ 85 % | Đảm bảo agent trả lời chính xác phần lớn câu hỏi cốt lõi trước khi deploy; ngăn regression trong CI/CD (`.github/workflows/eval.yml`). |

---

## 2. Benchmark hiệu năng (develop, `load_test.py`)

### 2.1 Concurrency vs latency (agent v2)

*(Nguồn: `reports/load_v2_c1.json`, `load_v2_c10.json`, `load_v2_c50.json`, `load_v2_c100.json` — mỗi mức 300 requests)*

| Concurrency | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Error rate | SLO |
|---|---|---|---|---|---|---|
| 1 | 4.12 | 194.1 | 223.9 | 1707.2 | 2.00% | PASS |
| 10 | 33.43 | 191.2 | 228.4 | 1701.3 | 1.00% | PASS |
| 50 | 93.95 | 225.7 | 532.8 | 1740.8 | 0.67% | FAIL |
| 100 | 61.23 | 554.5 | 3203.8 | 3537.2 | 2.00% | FAIL |

**Nhận xét:** RPS **không tăng tuyến tính** theo concurrency. Từ concurrency 1 → 10 → 50, RPS tăng mạnh (4.12 → 33.43 → 93.95 req/s) vì hệ thống tận dụng tốt async I/O. Tuy nhiên đến concurrency = 50, p95 đã bắt đầu vượt ngưỡng SLO 500 ms (532.8 ms); và lên concurrency = 100 thì hệ thống bị bão hoà (1 tiến trình uvicorn xử lý quá nhiều kết nối đồng thời, tranh chấp event loop/socket và hàng đợi phình to) khiến p50 nhảy lên 554.5 ms, p95 vọt lên 3203.8 ms còn RPS tụt ngược xuống còn 61.23 req/s.

### 2.2 Tail latency

*(Nguồn: `reports/load_v2_slow_0.json`, `load_v2_slow_0.03.json`, `load_v2_slow_0.1.json` — 300 requests, concurrency = 20)*

| `MOCK_SLOW_RATE` | avg (ms) | p95 (ms) | p99 (ms) |
|---|---|---|---|
| 0 | 190.8 | 225.1 | 242.0 |
| 0.03 (mặc định) | 257.5 | 249.6 | 1711.0 |
| 0.1 | 344.6 | 1698.0 | 1716.5 |

**Nhận xét:** **p99 và p95** là hai chỉ số nhạy nhất với request chậm (tail latency). Khi `MOCK_SLOW_RATE` tăng từ `0` lên `0.03`, `avg` chỉ tăng nhẹ từ `190.8 ms` lên `257.5 ms` (+66.7 ms, trông vẫn rất ổn), nhưng `p99` nhảy vọt hơn 7 lần từ `242.0 ms` lên `1711.0 ms`. Khi tăng lên `0.1` (> 5% request chậm), cả `p95` cũng vọt lên `1698.0 ms` trong khi `avg` mới là `344.6 ms`. Vì **average che giấu nhóm người dùng chịu độ trễ cao**, SLO bắt buộc phải đặt theo percentile (`p95` / `p99`) thay vì trung bình.

---

## 3. Benchmark chất lượng (develop, `eval/run_eval.py`)

*(Nguồn: `reports/eval_v1.json`, `reports/eval_v2.json`, `reports/eval_v2_fixed.json` — 23 câu hỏi trong `golden_dataset.jsonl`)*

| Chỉ số | Agent v1 | Agent v2 | Agent v2 (sau khi sửa KB) |
|---|---|---|---|
| Accuracy | 4.35% (1/23) | 91.30% (21/23) | 100.00% (23/23) |
| Avg tokens / câu | 57.3 | 109.1 | 113.8 |
| Tổng cost (USD) | \$0.000408 | \$0.000762 | \$0.000792 |
| Cost / câu đúng (USD) | \$0.00040755 | \$0.00003627 | \$0.00003443 |
| Quality gate | FAIL | PASS | PASS (0 regressions) |

**Những câu v2 trả lời sai ban đầu và cách bạn sửa:**

| ID | Keyword bị thiếu | Bạn đã sửa gì |
|---|---|---|
| `security-02` | `token`, `hết hạn`, `chữ ký` | Thêm entry với từ khoá `["jwt"]` giải thích sự khác biệt giữa API key và JWT (token có chữ ký số, chứa thông tin user và thời điểm hết hạn) vào `KNOWLEDGE_BASE` trong `develop/utils/mock_llm.py` (và đồng bộ sang `production/utils/mock_llm.py`). |
| `scaling-03` | `cpu`, `memory`, `custom metrics` | Thêm entry với từ khoá `["hpa", "kubernetes", "autoscal"]` mô tả Kubernetes HPA tự động scale pod dựa trên CPU, memory hoặc custom metrics vào `KNOWLEDGE_BASE`. |

**Quality gate trong CI:** Đã tạo workflow `.github/workflows/eval.yml` chạy tự động trên mỗi Pull Request: khởi động agent v2 (`MOCK_ERROR_RATE=0`) và chạy `python eval/run_eval.py --url http://localhost:8002 --min-accuracy 0.85`. Khi accuracy dưới ngưỡng hoặc có regression, `run_eval.py` thoát với **exit code 1** làm job CI fail và chặn merge PR.

**Nhận xét về trade-off chất lượng / chi phí / latency giữa v1 và v2:**
- Agent v2 bổ sung bước retrieval (RAG) khiến độ trễ tăng nhẹ (`p50` từ `167.6 ms` → `193.0 ms`, `p95` từ `184.4 ms` → `213.0 ms`) và lượng token trung bình gần gấp đôi (`57.3` → `109.1` → `113.8` tokens/câu) do phải nhồi ngữ cảnh vào prompt, làm tổng chi phí 23 câu tăng gần gấp đôi (`\$0.000408` → `\$0.000792`).
- Tuy nhiên, độ chính xác nhảy vọt từ `4.35%` lên `91.30%` (và `100%` sau khi bổ sung KB), khiến **chi phí trên mỗi câu trả lời đúng (cost/correct) giảm hơn 11 lần** (từ `\$0.00040755` xuống `\$0.00003443`). Đây là sự đánh đổi hoàn toàn xứng đáng trong production.

---

## 4. Observability stack (production)

### 4.1 Screenshot

- [x] Dashboard Grafana khi đang có tải ([screenshots/grafana-dashboard.png](screenshots/grafana-dashboard.png)) — hiển thị rõ 4 Golden Signals: Traffic `52.0 req/s`, Error rate `1.19%`, Latency p95 `447.32 ms`, Saturation (in-flight) `19`, Instances up `2`, LLM cost 1h `\$0.1110`, cùng biểu đồ p50/p95/p99 và Requests/s theo status code.
- [x] Một trace chậm trong Jaeger ([screenshots/jaeger-slow-trace.png](screenshots/jaeger-slow-trace.png)) — `POST /ask` (`200`), tổng thời gian `1.7s`, thấy rõ span `agent.retrieve_context` (`14.8 ms`) và `chat mock-llm-small` chiếm gần toàn bộ thời gian.
- [x] Một trace lỗi trong Jaeger ([screenshots/jaeger-error-trace.png](screenshots/jaeger-error-trace.png)) — `POST /ask` (`503`, trace `28a40ba`, `98.5 ms`), lỗi phát sinh tại span `chat mock-llm-small` (`72.7 ms`, icon đỏ lỗi upstream).
- [x] Trang Alerts của Prometheus với **2 alert cùng đang FIRING** ([screenshots/prometheus-alert-firing.png](screenshots/prometheus-alert-firing.png)): `AgentHighErrorRate` và `LLMCostBurnRateHigh`.

**Trả lời câu hỏi phân tích Exercise 7.4 (Grafana & Nginx):**
1. **Khoảng cách giữa `avg` và `p95`/`p99`:** Trên biểu đồ Latency `/ask`, đường `avg`/`p50` nằm quanh ~200–250 ms còn `p95` ở ~447 ms và `p99` nhảy lên ~1.7–1.8 s. Khoảng cách lớn giữa `avg` và `p99` cho thấy hệ thống có **tail latency** rõ rệt (do ~3% request gọi LLM bị chậm thêm 1500 ms).
2. **Input tokens/s lớn hơn output tokens/s:** Ở agent v2 (RAG), mỗi lần gọi LLM phải gửi kèm `SYSTEM_PROMPT` + đoạn tài liệu retrieve từ `KNOWLEDGE_BASE` + câu hỏi của người dùng làm input, trong khi câu trả lời output ngắn gọn hơn nên số input token luôn lớn hơn output token.
3. **Vì sao `GET /metrics` qua Nginx (`http://localhost:8080/metrics`) trả về `404`:** Trong `nginx/nginx.conf`, route `/metrics` bị chặn cố tình (`return 404`) để không phơi bày số liệu nội bộ (lưu lượng, chi phí, cấu trúc route, thông tin instance) ra internet công cộng; chỉ Prometheus nằm trong mạng nội bộ Docker (`obs_net`) mới được scrape trực tiếp `agent:8000/metrics`.

### 4.2 Thí nghiệm "phá hệ thống"

Chạy tải liên tục bằng Locust (30 users, ramp-up 10) trên stack `k5-day14-observability` (`agent=2`), thay đổi `.env` và quan sát tại `http://localhost:9090/alerts` (ảnh chụp cả 2 alert cùng firing tại [screenshots/prometheus-alert-firing.png](screenshots/prometheus-alert-firing.png)):

| # | Thay đổi | Alert nào firing | Thời gian từ lúc đổi → firing | Panel nào phản ánh rõ nhất |
|---|---|---|---|---|
| A | `MOCK_ERROR_RATE=0.2` | `AgentHighErrorRate` (Error rate ~18.7% > 5%, `for: 1m`) | ~1 phút 15 giây (đạt điều kiện sau ~15s scrape + giữ `for: 1m`) | Golden Signals: **Error rate** chuyển đỏ; **Requests/s theo status code** xuất hiện đường `5xx`; Jaeger có nhiều trace `error=true`. |
| B | `MOCK_SLOW_RATE=0.3` | `AgentHighLatencyP95` (`p95 > 1s`, `for: 2m`) | ~2 phút 15 giây (theo cấu hình `for: 2m` trong `alert_rules.yml`) | **Latency p95** chuyển đỏ; đồ thị p50/p95/p99 nhảy vọt đường p95 lên ~1.7s. |
| C | `LLM_MODEL=mock-llm-large` | `LLMCostBurnRateHigh` (chi phí/giờ vượt ngưỡng, `for: 2m`) | ~2 phút 15 giây (cần cửa sổ `rate(...[5m])` + `for: 2m`) | **LLM cost (1h)** và **Cost USD/giờ** tăng mạnh (~16×), **LLM p95** tăng gấp ~2.5×. |
| D | `docker pause` 1 instance | `AgentInstanceDown` (`up{job="agent"} == 0`) | ~30–60 giây (container vẫn còn trong DNS nên Prometheus scrape fail → `up == 0`) | **Instances up** giảm đi 1; Locust có thể gặp `504 Gateway Timeout` từ Nginx nhưng panel **Error rate** của agent vẫn xanh vì lỗi 504 xảy ra ở tầng Nginx trước khi agent ghi metric. |
| E | `AGENT_VERSION=v1` | **Không có alert nào firing** | Không firing | **Retrieval hit ratio** mất dữ liệu, **tokens/s** giảm, bảng **version** đổi sang `v1` — mọi Golden Signal hạ tầng vẫn xanh. |

**Thí nghiệm E không làm alert nào firing. Bạn sẽ phát hiện loại sự cố này trong production bằng cách nào?**
- Các metric hạ tầng (RPS, latency, error rate 5xx) chỉ cho biết **hệ thống có đang chạy không**, không biết **câu trả lời có đúng hay không** (khi hạ xuống `v1`, agent thậm chí còn nhanh hơn và rẻ hơn nhưng accuracy tụt từ 100% xuống 4.35%).
- Để phát hiện trong production cần:
  1. **Quality gate trong CI/CD:** chạy `eval/run_eval.py` trên golden dataset trước mỗi lần merge/deploy (đã cấu hình trong `.github/workflows/eval.yml`, fail build khi accuracy < 85% hoặc có regression).
  2. **Online evaluation & giám sát chất lượng:** lấy mẫu traffic thật chạy LLM-as-a-judge định kỳ, theo dõi tín hiệu người dùng (thumbs up/down, retry rate), và đặt alert cho các metric đặc thù RAG như **retrieval hit ratio tụt bất thường** hoặc `agent_info{version}` thay đổi ngoài dự kiến.

### 4.3 Debug bằng trace

1. **Trace chậm nhất tìm được trong Jaeger** ([screenshots/jaeger-slow-trace.png](screenshots/jaeger-slow-trace.png)):
   - `trace_id` (prefix hiển thị trên Jaeger UI): `79a038c`
   - Tổng thời gian: `1.7s` (~`1700 ms`)
   - Span chiếm nhiều thời gian nhất: `chat mock-llm-small` (~`1685 ms`, chiếm **~99%** tổng thời gian request; trong khi `agent.retrieve_context` chỉ mất `14.8 ms` — chứng tỏ độ trễ do upstream LLM tail latency chứ không phải do code retrieval hay FastAPI).

2. **Liên kết log JSON ↔ trace (`trace_id`):** trích từ `docker compose -p k5-day14-observability logs agent | grep '"llm_error"'` (trace lỗi `503` tương tự [screenshots/jaeger-error-trace.png](screenshots/jaeger-error-trace.png), có thể dán trực tiếp `trace_id` vào ô *Lookup by Trace ID* của Jaeger để mở toàn bộ cây span của request):
   - `trace_id`: `a3f4de610dafba422a06a9b00da1dba1`
   - `span_id`: `b0d8f7be022bcaf1`
   - Dòng log JSON thực tế:

```json
{"ts": "2026-11-26T16:11:56.189950+00:00", "level": "INFO", "service": "observable-agent", "instance": "ff5c3e11cf9a", "event": "llm_error", "trace_id": "a3f4de610dafba422a06a9b00da1dba1", "span_id": "b0d8f7be022bcaf1", "request_id": "723c52e16016", "error": "Upstream LLM error: 503 Service Unavailable (mock)"}
```

---

## 5. Benchmark scaling (production, Locust headless, 100 users, 2 phút)

*(Nguồn: `benchmark-results/scale_1_stats.csv` và `benchmark-results/scale_3_stats.csv` — Locust `--headless -u 100 -r 10 -t 2m`, tổng hợp `Aggregated` và chi tiết `POST /ask`)*

| Số instance | RPS (Aggregated / `/ask`) | p50 (ms) | p95 (ms) | p99 (ms) | Failures |
|---|---|---|---|---|---|
| 1 | 64.56 (58.44) | 200 (210) | 410 (430) | 1700 (1700) | 132 / 7662 (1.72%) — `/ask`: 125, `/health`: 7 |
| 3 | 64.93 (59.13) | 200 (210) | 370 (380) | 1700 (1700) | 68 / 7702 (0.88%) — `/ask`: 68, `/health`: 0 |

**Giải thích kết quả:**
- Khi scale từ **1 → 3 instance**, **p95 giảm nhẹ** từ `410 ms` → `370 ms` (riêng `/ask`: `430 ms` → `380 ms`) và lỗi timeout ở `/health` giảm hẳn từ `7` về `0` (tổng tỉ lệ lỗi giảm gần một nửa từ `1.72%` xuống `0.88%`) nhờ giảm tải CPU/hàng đợi trên từng container (mỗi container bị giới hạn `0.5 CPU`).
- Tuy nhiên, **RPS (~64.6 → 64.9 req/s), p50 (200 ms) và p99 (1700 ms) gần như không đổi**. Nút thắt không nằm ở CPU của agent mà nằm ở **độ trễ I/O của upstream LLM** (`MOCK_LATENCY_MS=120ms`, jitter `60ms`, và tail latency `MOCK_SLOW_MS=1500ms` ở 3% request) cộng với `wait_time` giữa các request của 100 user mô phỏng trong Locust. Vì mỗi instance agent là async (chủ yếu chờ I/O), tăng số instance giúp tăng khả năng chịu tải và giảm queueing/lỗi dưới tải cao nhưng không thể rút ngắn latency nội tại của mỗi lần gọi LLM.

---

## 6. Kết luận

- **3 điều học được về benchmark & observability:**
  1. **Không bao giờ đánh giá hiệu năng bằng latency trung bình:** `avg` che giấu tail latency; phải dùng `p95`/`p99` để đặt SLO và phát hiện nhóm người dùng đang phải chờ lâu.
  2. **Ba trụ cột Metrics + Traces + Logs phải liên kết với nhau:** Prometheus/Grafana phát hiện *có vấn đề* (4 Golden Signals), Jaeger trace chỉ ra *chậm/lỗi ở đúng span nào* (retrieval vs LLM), và `trace_id` nối thẳng sang JSON log để điều tra chi tiết.
  3. **Hạ tầng xanh không đồng nghĩa chất lượng AI tốt:** metric hệ thống không bắt được sự cố giảm chất lượng câu trả lời (thí nghiệm E với `v1`), nên bắt buộc phải có **eval trên golden dataset + quality gate trong CI/CD** cùng chỉ số `cost / câu đúng`.
- **Nếu đưa agent này lên production thật, sẽ thêm các metric / alert:**
  - **Metric chất lượng & RAG:** `agent_retrieval_hit_ratio` (alert khi tụt dưới ngưỡng), điểm eval tự động (LLM-as-a-judge) trên mẫu traffic thật, tỉ lệ người dùng bấm feedback tiêu cực.
  - **Metric tầng Load Balancer / Nginx & phụ thuộc ngoài:** tỉ lệ lỗi `502`/`504` ở Nginx (bắt các lỗi mà metric của agent không thấy như thí nghiệm D), Redis latency / connection errors, và tỷ lệ request bị chặn `429` (rate limit) / `402` (cost guard).
  - **Alerting & Routing:** kết nối **Alertmanager** gửi cảnh báo theo mức độ (`critical` → PagerDuty/Telegram, `warning` → Slack) và đặt budget alert theo ngày/tháng.
