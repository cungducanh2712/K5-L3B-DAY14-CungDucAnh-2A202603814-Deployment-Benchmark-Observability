# Benchmark & Observability Report — Day 14

> Copy file này thành `BENCHMARK_REPORT.md` trong repo của bạn và điền đầy đủ.
> Mọi con số phải lấy từ report JSON / CSV / screenshot thật, không ước lượng.

**Họ tên:** Cung Đức Anh (MSSV: 2A202603814)
**Ngày chạy:** 2026-11-26
**Môi trường:** Windows 11 + WSL2 Ubuntu (Python 3.14.4) + Docker Desktop

---

## 1. SLO của agent

| SLI | SLO | Lý do chọn ngưỡng này |
|---|---|---|
| p95 latency `/ask` | ≤ 500 ms | 95% người dùng nhận phản hồi dưới 0.5s (trải nghiệm mượt, khớp ngưỡng mặc định trong `load_test.py`) |
| Error rate `/ask` | ≤ 2.0 % | Chấp nhận 1% lỗi ngẫu nhiên từ upstream mock LLM (`MOCK_ERROR_RATE=0.01`), vẫn đảm bảo error budget 2% |
| Accuracy trên golden dataset | ≥ 85 % (mục tiêu sau sửa KB: 100%) | Đảm bảo agent trả lời đúng kiến thức cốt lõi trước khi cho phép merge/deploy qua CI quality gate |

---

## 2. Benchmark hiệu năng (develop, `load_test.py`)

### 2.1 Concurrency vs latency (agent v2)

| Concurrency | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Error rate | SLO |
|---|---|---|---|---|---|---|
| 1 | 4.48 | 190.8 | 219.7 | 1682.8 | 1.67% | PASS |
| 10 | 38.21 | 187.6 | 218.7 | 1680.7 | 0.67% | PASS |
| 50 | 103.74 | 196.2 | 276.8 | 1699.7 | 1.00% | PASS |
| 100 | 59.5 | 387.3 | 3133.1 | 3460.2 | 1.33% | FAIL (p95 > 500ms) |

**Nhận xét:** RPS **không** tăng tuyến tính mãi theo concurrency. Từ `c=1` → `c=10` → `c=50`, RPS tăng mạnh (`4.48` → `38.21` → `103.74 req/s`) nhờ xử lý bất đồng bộ; nhưng tới `c=100`, hệ thống chạm ngưỡng bão hòa CPU/event loop của 1 process uvicorn đơn lẻ nên các request phải xếp hàng chờ → RPS tụt xuống `59.5 req/s` và `p95` vọt lên `3133.1 ms` (vi phạm SLO).

### 2.2 Tail latency

| `MOCK_SLOW_RATE` | avg (ms) | p95 (ms) | p99 (ms) |
|---|---|---|---|
| 0 | 188.2 | 217.3 | 224.0 |
| 0.03 (mặc định) | 233.6 | 217.4 | 1700.7 |
| 0.1 | 376.0 | 1700.5 | 1713.4 |

**Nhận xét:** `p99` (và `p95` khi tỉ lệ slow đạt 10%) nhạy nhất với request chậm, trong khi `avg` tăng rất ít (`188.2 ms` → `233.6 ms` ở mức 3%, vẫn trông "bình thường"). Không nên đặt SLO theo `average` vì trung bình cộng bị pha loãng bởi đa số request nhanh, che giấu trải nghiệm tệ của nhóm người dùng gặp tail latency.

---

## 3. Benchmark chất lượng (develop, `eval/run_eval.py`)

| Chỉ số | Agent v1 | Agent v2 | Agent v2 (sau khi sửa KB) |
|---|---|---|---|
| Accuracy | 4.3% (1/23) | 91.3% (21/23) | 100.0% (23/23) |
| Avg tokens / câu | 56.7 | 109.0 | 113.8 |
| Tổng cost (USD) | $0.000399 | $0.000760 | $0.000792 |
| Cost / câu đúng (USD) | $0.00039855 | $0.00003619 | $0.00003443 |
| Quality gate | FAIL | PASS (≥ 80%) | PASS (≥ 100%, 0 regression) |

**Những câu v2 trả lời sai ban đầu và cách bạn sửa:**

| ID | Keyword bị thiếu | Bạn đã sửa gì |
|---|---|---|
| `security-02` | `token`, `hết hạn`, `chữ ký` | Thêm entry về JWT vs API key (`["jwt"]`) vào `KNOWLEDGE_BASE` trong `develop/utils/mock_llm.py` (và đồng bộ sang `production/utils/mock_llm.py`) |
| `scaling-03` | `cpu`, `memory`, `custom metrics` | Thêm entry về Kubernetes HPA (`["hpa", "kubernetes", "autoscal"]`) vào `KNOWLEDGE_BASE` |

**Nhận xét về trade-off chất lượng / chi phí / latency giữa v1 và v2:**
- `v2` (có retrieval/RAG) tốn gần gấp đôi số token/câu (`109.0`–`113.8` vs `56.7`) và tổng cost cao hơn (`$0.000792` vs `$0.000399`), nhưng accuracy tăng vọt từ `4.3%` lên `100.0%`. Nếu tính theo **cost / câu trả lời đúng**, `v2` lại **rẻ hơn hơn 11 lần** (`$0.00003443` vs `$0.00039855`).
- **CI Quality Gate:** Đã cấu hình `.github/workflows/eval.yml` chạy `run_eval.py --min-accuracy 0.85` trên mỗi PR; vì `run_eval.py` trả về **exit code 1** khi accuracy dưới ngưỡng hoặc có câu bị regression, GitHub Actions sẽ đánh fail job và chặn merge PR.

---

## 4. Observability stack (production)

### 4.1 Screenshot

- [x] Dashboard Grafana khi đang có tải (`screenshots/grafana-dashboard.png`)
- [x] Một trace chậm trong Jaeger (`screenshots/jaeger-slow-trace.png`)
- [x] Một trace lỗi trong Jaeger (`screenshots/jaeger-error-trace.png`)
- [x] Trang Alerts của Prometheus với ít nhất 2 alert đang **firing** (`screenshots/prometheus-alert-firing.png`)

### 4.2 Thí nghiệm "phá hệ thống"

| # | Thay đổi | Alert nào firing | Thời gian từ lúc đổi → firing | Panel nào phản ánh rõ nhất |
|---|---|---|---|---|
| A | `MOCK_ERROR_RATE=0.2` | `AgentHighErrorRate` | ~1 phút 10 giây (`for: 1m` + chu kỳ scrape 10s) | `LLM calls theo kết quả` (đường `error` tăng vọt) |
| B | `MOCK_SLOW_RATE=0.3` | *(Chưa chạy riêng — `for: 2m`)* | — | `LLM latency p95 theo model` |
| C | Load cao (`100` users liên tục ở `scale_1`/`scale_3`) | `LLMCostBurnRateHigh` | ~2 phút 30 giây (`for: 2m` khi rate > $2/giờ) | `Cost burn rate (USD/giờ)` (vượt ngưỡng $2 lên ~$6/giờ) |
| D | `docker pause` 1 instance (`agent-2`) | `AgentInstanceDown` | ~35 giây (`for: 30s` + chu kỳ scrape) | `Requests/s theo instance` (1 đường rơi về 0) |
| E | `AGENT_VERSION=v1` | Không có alert nào firing | Không firing | `Retrieval hit ratio (agent v2)` (giảm về 0%) |

**Thí nghiệm E không làm alert nào firing. Bạn sẽ phát hiện loại sự cố này trong production bằng cách nào?**

- Chạy **eval định kỳ / online eval** trên mẫu traffic thật (hoặc golden dataset trong CI quality gate trước khi deploy) để đo accuracy/relevance.
- Theo dõi thêm metric nghiệp vụ/RAG trên Grafana như `Retrieval hit ratio` (khi chuyển nhầm sang `v1`, tỉ lệ retrieval giảm về 0%) hoặc tỉ lệ user thumbs-down / retry.

### 4.3 Debug bằng trace

Chọn 1 trace chậm nhất bạn tìm được:

- `trace_id`: `1eaf97b6c809ab7c77e536086acdc577`
- Tổng thời gian: `1665.0 ms` (~`1.7s` trên Jaeger)
- Span chiếm nhiều thời gian nhất: `chat mock-llm-small` (`1647.5 ms`, `98.9%`) *(trong khi `agent.retrieve_context` chỉ mất `18.7 ms`)*
- Dòng log tương ứng (grep theo `trace_id`):

```json
{"ts": "2026-11-26T10:14:31.473020+00:00", "level": "INFO", "service": "observable-agent", "instance": "aae1b15e0331", "event": "llm_call", "trace_id": "1eaf97b6c809ab7c77e536086acdc577", "span_id": "9ed1beb12b158f57", "request_id": "175271ed002b", "model": "mock-llm-small", "agent_version": "v2", "input_tokens": 83, "output_tokens": 44, "cost_usd": 3.9e-05, "llm_ms": 1647.5}
```

---

## 5. Benchmark scaling (production, Locust headless, 100 users, 2 phút)

*(Số liệu tổng hợp Aggregated từ `benchmark-results/scale_1_stats.csv` và `benchmark-results/scale_3_stats.csv`)*

| Số instance | RPS | p50 (ms) | p95 (ms) | p99 (ms) | Failures |
|---|---|---|---|---|---|
| 1 | 65.90 | 190 | 240 | 1700 | 92 (1.17% / 7890 reqs) |
| 3 | 65.69 | 190 | 230 | 1700 | 83 (1.06% / 7863 reqs) |

**Giải thích kết quả:** Scale từ 1 lên 3 instance chỉ giảm nhẹ `p95` (`240 ms` → `230 ms`) và gần như không tăng RPS (`65.90` vs `65.69 req/s`). Nút thắt không nằm ở CPU của agent mà nằm ở **chính tải sinh ra từ Locust** (`100` users với `wait_time = between(0.5, 1.5)` chỉ tạo tối đa ~66 req/s, và 1 instance FastAPI bất đồng bộ đã xử lý dư sức mức tải này). Còn `p99 = 1700 ms` đến từ `MOCK_SLOW_RATE=0.03` nội tại của mock LLM nên scale thêm instance cũng không làm mất tail latency của từng call chậm.

---

## 6. Kết luận

- 3 điều bạn học được về benchmark & observability:
  1. **Average che giấu tail latency:** Cần đặt SLO và theo dõi theo `p95`/`p99` thay vì trung bình cộng.
  2. **Chất lượng phải đo bằng Eval & Quality Gate:** Observability hạ tầng (CPU, HTTP 200, latency) không phát hiện được suy giảm chất lượng trả lời (như tụt từ `v2` xuống `v1`), cần golden dataset + CI gate.
  3. **3 trụ cột liên kết nhau:** Metrics (Grafana) báo triệu chứng → Trace (Jaeger) chỉ ra đúng span chậm/lỗi → Log JSON (`trace_id`) cho thấy chi tiết payload/nguyên nhân.
- Nếu đưa agent này lên production thật, bạn sẽ thêm metric / alert nào nữa?
  - Alert cho **Retrieval hit ratio giảm đột ngột** (phát hiện lỗi RAG/vector DB) và **Token/request bất thường** (chống prompt injection hoặc loop).
  - Thêm metric **Time-to-First-Token (TTFT)** cho streaming response và kết nối Alertmanager gửi cảnh báo tới Slack/PagerDuty.
