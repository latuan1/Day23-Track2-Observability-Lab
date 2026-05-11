# Day 23 Lab Reflection

**Student:** Lương Anh Tuấn
**Submission date:** 2026-05-11  
**Lab repo URL:** https://github.com/latuan1/Day23-Track2-Observability-Lab.git

---

## 1. Hardware + setup output

Kết quả chạy kiểm tra setup:

```text
Docker:        OK  (29.4.1)
Compose v2:    OK  (5.1.3)
RAM available: 7.62 GB (OK)
Ports free:    OK
Report written: C:\Users\nak11\python\aithucchien\Track-2\Day23-Track2-Observability-Lab\00-setup\setup-report.json
```

File `00-setup/setup-report.json` xác nhận Docker, Compose v2, RAM và các port cần thiết đều đạt yêu cầu để chạy stack quan sát gồm FastAPI, Prometheus, Grafana, Alertmanager, Loki, Jaeger và OTel Collector.

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels

Evidence: `submission/screenshots/dashboard-overview.png`

Dashboard Overview đã hiển thị đủ 6 panel chính: request rate, latency P50/P95/P99, error rate, GPU utilization, token throughput và in-flight requests. Sau khi chạy load, các metric như request rate, latency, token throughput và GPU utilization có dữ liệu, chứng minh Prometheus đang scrape `/metrics` của service `inference-api`.

### Burn-rate panel

Evidence: `submission/screenshots/slo-burn-rate.png`

SLO dashboard dùng các recording rule như `inference:fail_ratio:rate5m`, `rate30m`, `rate1h` và `rate6h`. Với `make load` bình thường, service chủ yếu tạo request `status="ok"`, nên burn-rate có thể chưa có dữ liệu lỗi. Để có dữ liệu burn-rate rõ hơn, cần chạy thêm error load bằng `ERROR_RATE=0.2` để sinh `inference_requests_total{status="error"}`.

### Cost and tokens

Evidence: `submission/screenshots/cost-and-tokens.png`

Cost dashboard hiển thị token throughput và ước tính chi phí theo giờ. Điểm quan trọng ở đây là biến kỹ thuật `inference_tokens_total` được chuyển thành tín hiệu vận hành dễ hiểu hơn: token/giây và USD/giờ.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| T0 | Dừng `day23-app` để Prometheus không scrape được service | `submission/screenshots/alertmanager-firing.png` |
| T0+90s | `ServiceDown` chuyển sang firing trong Alertmanager và Slack nhận thông báo | `submission/screenshots/slack-firing.png` |
| T1 | Khởi động lại `day23-app` | `docker start day23-app` |
| T1+60s | Alert resolve và Slack nhận resolved message | `submission/screenshots/slack-resolved.png` |

### One thing surprised me about Prometheus / Grafana

Điều làm tôi bất ngờ nhất là Grafana có thể hiển thị “No data” dù ứng dụng thật sự đã nhận request. Nguyên nhân không nằm ở Locust hay FastAPI, mà nằm ở cấu hình datasource UID: dashboard dùng UID `prometheus`, còn datasource ban đầu được Grafana tạo với UID tự sinh. Sau khi thêm `uid: prometheus` vào provisioning, các panel đọc đúng datasource và dữ liệu bắt đầu hiện ra.

Một điểm nữa là các panel dùng `rate(...[1m])` hoặc `rate(...[5m])` rất phụ thuộc vào time range và thời điểm chụp màn hình. Vì vậy khi làm evidence cho dashboard, cần chụp ngay trong lúc load hoặc ngay sau load, và đặt Grafana về “Last 15 minutes”.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Evidence: `submission/screenshots/jaeger-trace.png`

Jaeger trace cho request `POST /predict` hiển thị span cha `predict` và 3 span con:

```text
predict
├── embed-text
├── vector-search
└── generate-tokens
```

Evidence cho GenAI semantic attributes: `submission/screenshots/jaeger-genai-attrs.png`

Trong span `generate-tokens`, các tag GenAI quan trọng gồm:

```text
gen_ai.operation.name = chat
gen_ai.request.model = llama3-mock
gen_ai.response.model = llama3-mock
gen_ai.response.finish_reason = stop
gen_ai.system = mock
gen_ai.usage.input_tokens = 8
gen_ai.usage.output_tokens = 8
```

Các attribute này giúp trace không chỉ nói “request chậm”, mà còn cho biết model nào, loại thao tác GenAI nào, số token input/output và lý do kết thúc generation.

### Log line correlated to trace

Structured JSON log line có `trace_id`:

```json
{"model":"llama3-mock","input_tokens":8,"output_tokens":8,"quality":0.855,"duration_seconds":0.1647,"trace_id":"ce198bc40c5820734352b6e20c53ab11","event":"prediction served","level":"info","timestamp":"2026-05-11T14:46:16.889664Z"}
```

Log này hữu ích vì cùng một `trace_id` có thể dùng để đi từ log sang trace. Khi debugging production, tôi sẽ bắt đầu từ log có lỗi hoặc latency cao, lấy `trace_id`, rồi mở Jaeger để xem request đã tốn thời gian ở bước embedding, vector search hay generation.

### Tail-sampling math

OTel Collector dùng tail sampling với 3 policy:

```text
keep-errors: giữ 100% trace có status_code = ERROR
keep-slow: giữ 100% trace có latency > 2s
probabilistic-1pct: giữ 1% healthy trace
```

Nếu service tạo `N` traces/giây, tỷ lệ trace được giữ có thể viết như sau:

```text
sampled = N × (P(error) × 1.0 + P(slow ∧ not error) × 1.0 + P(healthy) × 0.01)
```

Ví dụ với 1% error, 1% slow và 98% healthy:

```text
sampled = N × (0.01 + 0.01 + 0.98 × 0.01)
        = N × 0.0298
        ≈ 3% tổng số trace
```

Điều này giảm chi phí lưu trữ khoảng 97% so với giữ toàn bộ trace, nhưng vẫn giữ lại những trace quan trọng nhất: lỗi và request chậm. Trong demo, healthy trace thường bị drop vì chỉ giữ 1%, còn forced-error trace hoặc slow trace được giữ lại theo policy 100%.

---

## 4. Track 04 — Drift Detection

### PSI scores

Nội dung `04-drift-detection/reports/drift-summary.json`:

```json
{
  "prompt_length": {
    "psi": 3.461,
    "kl": 1.7982,
    "ks_stat": 0.702,
    "ks_pvalue": 0.0,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.0187,
    "kl": 0.0324,
    "ks_stat": 0.052,
    "ks_pvalue": 0.133853,
    "drift": "no"
  },
  "response_length": {
    "psi": 0.0162,
    "kl": 0.0178,
    "ks_stat": 0.056,
    "ks_pvalue": 0.086899,
    "drift": "no"
  },
  "response_quality": {
    "psi": 8.8486,
    "kl": 13.5011,
    "ks_stat": 0.941,
    "ks_pvalue": 0.0,
    "drift": "yes"
  }
}
```

Evidently HTML report đã được sinh tại `04-drift-detection/reports/drift-report.html`.

### Which test fits which feature?

Với `prompt_length`, tôi chọn PSI trong production vì đây là feature số một chiều, dễ bucket theo độ dài prompt. PSI phù hợp để theo dõi population shift theo thời gian và dễ diễn giải với threshold vận hành như `<0.1`, `0.1-0.2`, `>0.2`.

Với `embedding_norm`, tôi chọn KS nếu chỉ giám sát norm một chiều, vì KS so sánh trực tiếp hai phân phối liên tục qua CDF mà không phụ thuộc quá nhiều vào cách chia bin. Nếu giám sát toàn bộ vector embedding thay vì norm, tôi sẽ chọn MMD vì MMD phù hợp hơn cho drift đa chiều trong không gian embedding.

Với `response_length`, tôi chọn PSI vì độ dài response có thể bucket tự nhiên và rất dễ đưa lên dashboard theo tuần/ngày. Nếu response length tăng mạnh, chi phí token và latency cũng tăng, nên PSI là tín hiệu vận hành đơn giản để cảnh báo sớm.

Với `response_quality`, tôi chọn KS vì quality score là một giá trị liên tục bị chặn trong khoảng `[0, 1]`. KS giúp phát hiện phân phối quality bị kéo xuống, kể cả khi trung bình chưa giảm quá nhiều. KL cũng có ích để so sánh phân phối, nhưng nhạy với binning và zero-probability hơn, nên tôi sẽ dùng KL như metric bổ sung thay vì metric cảnh báo chính.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

Evidence: `submission/screenshots/prometheus-stub-targets.png` và `submission/screenshots/cross-day-dashboard.png`

Tôi đã dùng stub cho Day 19 và Day 20. Prometheus scrape được:

```text
day19_qdrant_collections = 3
day20_llamacpp_tokens_per_second ≈ 17-25 tokens/sec
```

Metric khó expose nhất là Day 20 llama.cpp tokens/sec. Lý do là llama.cpp HTTP server không phải lúc nào cũng có Prometheus metrics native, nên cần sidecar hoặc stub exporter để biến throughput, queue depth và completion count thành metric có shape ổn định. Day 19 Qdrant dễ hơn vì vector database thường có endpoint `/metrics` rõ ràng hơn, hoặc ít nhất có thể stub bằng số collections và search counter.

Cross-day dashboard render đủ 6 panel. Days 16, 17, 18 và 22 có thể hiển thị “No Data” vì không chạy prior-day services, nhưng Day 19 và Day 20 có dữ liệu từ stub, đủ chứng minh dashboard fail-soft và vẫn tích hợp được ít nhất một nguồn prior-day.

---

## 6. The single change that mattered most

Thay đổi quan trọng nhất là làm cho metric, trace và log dùng cùng một mô hình nhận diện request: Prometheus có `model` và `status`, log có `trace_id`, còn Jaeger có span cha `predict` với các span con `embed-text`, `vector-search`, `generate-tokens`. Trước khi nối các tín hiệu này lại, dashboard chỉ cho biết “hệ thống có traffic” hoặc “request đang chậm”. Sau khi nối lại, tôi có thể đi từ một triệu chứng vận hành như latency/token cost/error rate sang đúng phần của pipeline gây ra vấn đề.

Điểm này gắn trực tiếp với khái niệm “observability không chỉ là thu thập telemetry, mà là khả năng đặt câu hỏi mới về hệ thống”. Metric cho biết có vấn đề ở mức aggregate, trace cho biết request đi qua bước nào và tốn thời gian ở đâu, log cho biết sự kiện cụ thể và `trace_id` để liên kết hai phần còn lại. Với AI service, tôi thấy hữu ích nhất là thêm GenAI semantic attributes và token metrics, vì chúng biến một request LLM từ hộp đen thành một workflow có thể đo được: model nào chạy, bao nhiêu token được xử lý, generation kết thúc thế nào, và chi phí ước tính ra sao.
