# Bonus Challenge — Observe một thứ thật (UNGRADED)

> English: [`BONUS-CHALLENGE-EN.md`](BONUS-CHALLENGE-EN.md)

**Loại:** Sân chơi để **chĩa observability stack thật vào một thứ bạn thật sự quan tâm** — không điểm, không deadline, không rubric.
**Đối tượng:** Bạn vào vai *SRE / Platform engineer* on-call cho một AI system cụ thể, với một người cụ thể (đồng đội, chủ shop, chính bạn sau 3 tháng) sẽ page bạn khi nó hỏng.
**Effort target:** 4–8 giờ. Khuyến khích pair (2–3 người), brainstorm trước, code sau.
**Vibe coding khuyến khích:** dashboard JSON, query DSL, YAML — AI lo; *bạn* quyết định system nên quan tâm cái gì và khi nào nên đánh thức một con người.

> Đây là chỗ bạn ngừng coi observability như "tô màu vào dashboard lab" mà coi nó như **trả lời 1 câu hỏi cho 1 người: làm sao họ biết AI của họ còn chạy lúc 2 giờ sáng?** Phần thưởng thực sự: 1 portfolio piece có thể chỉ vào nói "tôi instrument X cho Y, đây là dashboard, đây là alert, đây là postmortem" — không phải "tôi scrape `/metrics` được 60 giây."

---

## 5 provocations — chọn 1, hoặc invent your own

Mỗi provocation có 4 phần: **Audience** (ai bị page) · **Domain knowledge** (bạn đem gì vào) · **Application objective** (system nên nói gì với họ) · **Real-world output** (deliverable ship được).

### 1. Gắn telemetry vào lab của một ngày trước

> *"Bạn đã build 1 system qua 6 ngày. Giờ làm cho nó visible."*

- **Audience:** Chính bạn, 3 tháng sau, đang debug Day 19 vector store hoặc Day 20 model serving của chính mình khi retrieval quality drop hoặc p99 latency tăng dần.
- **Domain knowledge:** Bạn build cái lab đó — bạn biết 3 con số nào thật sự quan trọng (retrieval recall@10? GPU memory? token/sec? cache hit-rate?) chứ không phải 50 metrics Prometheus emit ra theo default.
- **Application objective:** Một Grafana dashboard + 1 multi-burn-rate alert duy nhất bắt được *failure mode bạn thật sự sợ* trong system cũ đó. Không phải mọi failure — chỉ failure dễ cắn nhất.
- **Real-world output:**
  - Fork lab của 1 ngày trước (Day 18/19/20) thêm instrumentation OpenTelemetry (~30 dòng)
  - `dashboards/<tên-ngày>.json` — commit dưới dạng code, không phải screenshot
  - 1 SLO + 1 burn-rate alert rule fire được khi failure mode xảy ra
  - `RUNBOOK.md` (~150 từ): alert đã fire — on-call làm gì tiếp?
  - Demo: trigger failure, alert fire trong vòng 5 phút, bạn follow runbook của chính mình

**Câu hỏi để brainstorm:**
- Trong các metrics lab cũ đang expose, 3 metrics nào bạn *thật sự* xem đầu tiên khi có incident? Vì sao là 3 metrics đó?
- *Failure mode không hiển nhiên* là gì — cái mà default alert của Prometheus sẽ không bắt? (data quality drift, cache-coherency, slow degradation, …)
- Alert của bạn có thể bắt failure đó *sớm hơn 10 phút* không nếu đổi burn-rate window?

---

### 2. Observability cho 1 doanh nghiệp Việt thật đang dùng AI

> *"Chọn 1 shop. Tưởng tượng chatbot của họ chết lúc 9h sáng thứ Hai. Họ ước họ có dashboard nào?"*

- **Audience:** Chủ một doanh nghiệp cụ thể có ít nhất 1 AI feature trong flow — quán cafe Hà Nội có Zalo chatbot, người bán Lazada/Shopee dùng auto-reply, homestay dùng OCR booking, trung tâm dạy thêm dùng LLM chấm bài luận. Bạn chọn.
- **Domain knowledge:** Bạn biết với *họ* một ngày tồi tệ là gì — mất 30 đơn vì chatbot chết? Trả lời tiếng Anh cho khách VN? Báo sai giá phòng? Metric kỹ thuật cần quan tâm theo sau business impact, không phải ngược lại.
- **Application objective:** 2 dashboards (1 cho chủ shop — ngôn ngữ thường, số to; 1 cho người bảo trì bot — kỹ thuật) + 2 alerts đi vào *kênh khác nhau* (chủ shop nhận tóm tắt kiểu SMS, dev nhận full context).
- **Real-world output:**
  - `bonus/business-case.md` — ai, cái gì, vì sao, sample failure modes (~300 từ)
  - 2 `dashboards/*.json` đã commit
  - 2 alert rules trong `prometheus/rules/`
  - 1 webhook-receiver script dịch dev alert thành 1 câu VN cho chủ shop
  - Model card / "Tôi đã quan sát gì": 5 sample alerts hệ thống sẽ fire trong tuần này nếu nó chạy thật

**Câu hỏi để brainstorm:**
- Với chủ shop không kỹ thuật, *"AI bị hỏng" nghĩa là gì*? Chậm? Sai? Cộc lốc? Down? Chọn 2 cái quan trọng nhất rồi chỉ thiết kế alert cho 2 cái đó.
- Metric nào bắt được "chatbot vẫn trả lời, nhưng trả lời sai"? (Hint: nó không nằm trong 4 golden signals đâu.)
- Làm sao tránh đánh thức chủ shop vì 1 blip thoáng qua? (multi-window burn-rate, ai đó?)

---

### 3. Drift detection trên 1 dataset Việt thật

> *"Chọn 1 dataset thật. Chỉ cho tôi ngày nó shift."*

- **Audience:** Một team vừa deploy 1 Vietnamese model (sentiment, classification, retrieval) chưa biết cách nhận ra khi nào training distribution không khớp với production nữa.
- **Domain knowledge:** Bạn hiểu đủ về domain dataset (news, e-commerce reviews, weather, exam essays, …) để nhận ra "shift thật" trông như thế nào — vocabulary drift quanh Tết, slang thay đổi, danh mục sản phẩm mới, schema evolution.
- **Application objective:** Một pipeline drift-detection có thể reproduce trên public Vietnamese data (headlines VnExpress, scrape Lazada review, public weather feed, …) chạy hàng ngày, tính PSI/KL/KS, emit alert có ý nghĩa — không phải "PSI > 0.1 trên feature_47" mà "top-5 product categories được mention nhiều nhất đã shift so với tuần trước."
- **Real-world output:**
  - `bonus/dataset-card.md` — dataset gì, vì sao thú vị, shift bạn expect/observe được
  - `drift/pipeline.py` — fetch, compute drift, emit Prometheus metrics
  - Grafana dashboard với ít nhất 1 panel mà grader nói tiếng Anh có thể hiểu trong 10 giây
  - 1 alert rule + 1 lần fire ví dụ (thật hoặc inject synthetic)
  - Reflection: PSI vs KL vs KS — cái nào surface được shift sớm nhất trên dataset *của bạn*?

**Câu hỏi để brainstorm:**
- Với text tiếng Việt, "feature drift" tương đương cái gì? Token frequency? Topic distribution? Embedding centroid distance?
- Vocabulary shift quanh Tết là *bình thường*. Alert của bạn phân biệt "drift theo mùa expected" với "model giờ đang hỏng" như thế nào?
- Khi alert fire thì action là gì? Retrain? Re-collect data? Page data scientist? Cụ thể hoá trong runbook của bạn.

---

### 4. Cost observability cho 1 model bạn thật sự đang chạy

> *"Hôm qua AI của bạn tốn bao nhiêu? Đến từng cent. Ngay bây giờ."*

- **Audience:** Một founder hoặc giảng viên tiết kiệm đang gọi API OpenAI/Anthropic (hoặc self-host vLLM/llama.cpp) — team nhỏ, không có FinOps department, hoá đơn bất ngờ là đau.
- **Domain knowledge:** Bạn biết workload của bạn cost-per-request *nên là bao nhiêu*, spike nghĩa là gì (debug loop? user tăng? prompt dài gấp 3?), và budget tháng chấp nhận được là bao nhiêu.
- **Application objective:** Một dashboard real-time `$/request`, `$/user`, `$/day` với 1 budget alert *cứng* fire trước khi thiệt hại xảy ra — không phải "tháng trước bạn xài $1000" mà "với mức burn hiện tại, 3 ngày nữa bạn chạm budget."
- **Real-world output:**
  - Client wrapper đã instrument (OpenAI SDK, llama.cpp HTTP, vLLM): emit `tokens_input`, `tokens_output`, `cost_usd`, `model` labels — ~50 dòng
  - Cost & tokens dashboard commit dưới dạng JSON
  - 2 alerts: (a) burn-rate so với monthly budget, (b) per-request anomaly (cost > 3σ giờ trước)
  - Tóm tắt 1 trang: 7 ngày cost data, top 3 endpoints tốn tiền nhất, 1 đề xuất tiết kiệm (cache, model nhỏ hơn, prompt compression)

**Câu hỏi để brainstorm:**
- `model_name` có nên là Prometheus label không? (Hint: cardinality.) Xử lý 5 vendors × 3 models × 100 endpoints thế nào?
- *Leading indicator* của bug runaway-cost là gì? (loop không early-stop, retry storm, prompt-doubling-due-to-tool-recursion)
- Với self-host model "cost" là GPU-hours, không phải API dollars. Đưa 2 thứ đó lên cùng 1 dashboard cho so sánh được như thế nào?

---

### 5. Diễn tập postmortem — chủ động phá system của chính mình

> *"Bạn không thể viết postmortem hay nếu chưa từng sống qua nó."*

- **Audience:** Bạn-trong-tương-lai, on-call rotation tương lai, và người tiếp theo bạn onboard vào system.
- **Domain knowledge:** Bạn build core lab. Bạn biết các vết nối nằm ở đâu.
- **Application objective:** Chọn 3 failure modes, inject mỗi cái bằng chaos script, đo time-to-detection, time-to-mitigation, rồi viết 1 postmortem thật với timeline, root cause, action items.
- **Real-world output:**
  - `bonus/chaos/` — 3 scripts phá system theo 3 cách khác nhau (kill service, fill disk, inject latency, poison data, exhaust quota, leak secret, …)
  - Cho mỗi cái: `bonus/postmortems/incident-NN.md` với **timeline**, **detection** (thời điểm + signal nào), **mitigation**, **root cause**, **action items** — đủ 5/5 sections từ §9 của deck
  - 1 trong 3 incidents phải dẫn tới *thay đổi thật* dashboard/alert (action item đã implement, không chỉ liệt kê)
  - Optional nhưng khuyến khích: video screen-record (~3 phút) cho thấy alert fire, bạn mở dashboard, tìm ra issue

**Câu hỏi để brainstorm:**
- 3 failure modes nào? Đừng chọn 3 cái cùng loại. Mix infra (kill service) + data (drift) + dependency (LLM API chậm).
- Time-to-detect là 1 metric. Lần inject thứ 2 cùng failure đó, time-to-detect có thể *ngắn hơn* không?
- Một postmortem tốt là blameless và *thay đổi system*. Incident này ép bạn thay đổi cái gì cụ thể?

---

## Cái gì tính là "xong"

- 1 folder `bonus/` trong fork của bạn, với deliverables tương ứng provocation đã chọn.
- 1 `bonus/REFLECTION.md` 2 đoạn trả lời: **bạn ngạc nhiên cái gì?** và **nếu có thêm 8 giờ nữa bạn sẽ build cái gì tiếp?**
- Public GitHub URL. Submit chung với core lab.

Hết. Không autograder. Không checklist. Chỉ 1 artifact thật nói *tôi build cái này cho X để làm Y*.

## Vì sao bonus này ungraded

Vì khoảnh khắc đặt điểm vào, bạn sẽ tối ưu cho điểm. Kỹ năng chúng tôi muốn bạn phát triển là **judgment** — cái gì đáng alert, cái gì không, đánh thức ai, khi nào, vì sao — và chấm điểm judgment làm nó trở nên perform-y thay vì thật. Build thứ bạn sẽ tự build cho chính mình nếu không ai nhìn.

Nếu ship được thứ ngon, gửi link. 3 cái xuất sắc nhất cohort sẽ được feature trong intro deck của phase tiếp theo.
