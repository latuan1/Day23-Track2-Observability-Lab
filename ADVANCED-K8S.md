# ADVANCED-K8S.md — Bài Tập Nâng Cao Kubernetes (UNGRADED)

> **Loại:** Bonus assignment, không tính điểm, không deadline. Sân chơi để bạn đẩy day23 observability stack — và kỹ năng infra của mình — lên một tầng nữa.
> **Đối tượng:** Bạn đã chạy được lab Compose ở mức cơ bản và muốn biết *"production thật sự trông như thế nào?"* — không chỉ về observability, mà còn về model serving và agent workloads.
> **Effort target:** 6–20 giờ tuỳ track chọn. Khuyến khích pair (2–3 người), brainstorm trước, YAML sau.
> **Vibe coding khuyến khích:** YAML manifests, Helm values, kustomization patches — *AI lo*. **Bạn lo** kiến trúc, blast radius, và quyết định "thứ này có nên chạy trên k8s không?"

> Mục tiêu cuối: 1 portfolio piece bạn có thể chỉ vào và nói *"đây là cluster tôi setup, đây là observability stack tôi deploy, đây là model tôi serve, đây là agent tôi run — tất cả production-shaped."*

---

## §0 — Vì sao K8s? (Why bother)

Compose tuyệt cho lab — 1 file, 7 services, `make up`, xong. Nhưng nó **dừng lại** ở những chỗ production bắt đầu:

| Compose dừng ở | K8s mở ra |
|---|---|
| 1 host, 1 user | Multi-node, multi-tenant, RBAC |
| Restart container = downtime | Rolling update, surge, pod disruption budget |
| Cấu hình hardcoded | ConfigMap, Secret, external secret stores |
| Scale = sửa file tay | HPA, VPA, cluster autoscaler |
| "Crash thì sao?" | Self-healing qua replica + probes |
| Observability bolted-on | ServiceMonitor / PodMonitor *là* idiom |

Đây không phải lý do để mọi project chuyển sang k8s — Compose vẫn đúng cho lab và prototype. Nhưng nếu bạn muốn làm **SRE / Platform / MLOps** thật, k8s là ngôn ngữ chung của production từ 2018 tới giờ.

---

## §1 — Foundation: K8s trong 1 buổi sáng

### 1.1 — Core objects (chỉ 7 cái phải nhớ)

| Object | Vai trò 1 câu | Khi bạn cần |
|---|---|---|
| **Pod** | Đơn vị schedulable nhỏ nhất, 1+ container chia chung network namespace | Hầu như không bao giờ tạo trực tiếp — luôn qua Deployment |
| **Deployment** | Stateless workload: N replica, rolling update, history | Web service, OTel collector, Grafana |
| **StatefulSet** | Workload có identity bền (`pod-0`, `pod-1`), stable storage | Prometheus, Loki, Tempo, vector DB, agent có sticky session |
| **DaemonSet** | 1 pod / 1 node, deploy tự động cho mọi node | node-exporter, log shipper, GPU device plugin |
| **Service** | DNS + load balancer nội bộ cho 1 set of pod (selector by label) | Cách 1 service gọi service khác |
| **Ingress** | HTTP/HTTPS reverse proxy vào cluster, TLS termination | Expose Grafana, app API ra ngoài |
| **ConfigMap / Secret** | Key-value config, mount thành env hoặc file | Prometheus config, API keys, model paths |
| **PVC** (PersistentVolumeClaim) | Yêu cầu storage bền (tách khỏi pod lifecycle) | Model weights, Prometheus TSDB, Loki chunks |

**Quy tắc nhớ:** *"Deployment cho stateless, StatefulSet cho stateful, DaemonSet cho per-node, Job/CronJob cho batch."* Mọi thứ khác là wiring.

### 1.2 — CRD & Operator — vì sao thắng cuộc

Một CRD (Custom Resource Definition) cho phép bạn extend k8s API với object riêng. Một Operator là controller chạy trong cluster, watch CRD đó, và reconcile state.

Ví dụ: thay vì viết 200 dòng YAML cho Prometheus + Alertmanager + rule files + ServiceMonitor scrape configs, bạn write:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata: { name: main, namespace: monitoring }
spec:
  replicas: 2
  serviceMonitorSelector: { matchLabels: { release: main } }
  retention: 30d
  storage:
    volumeClaimTemplate:
      spec: { storageClassName: ssd, resources: { requests: { storage: 100Gi } } }
```

…và Prometheus Operator tự generate tất cả các object dưới (StatefulSet, Service, ConfigMap, RBAC, …). **Đây là k8s idiom production**.

### 1.3 — Probes: tầng observability đầu tiên

Mỗi container nên expose 3 probe:

```yaml
livenessProbe:    # "Tôi còn sống không? Nếu fail → restart pod."
  httpGet: { path: /healthz, port: 8000 }
  initialDelaySeconds: 10
  periodSeconds: 10
readinessProbe:   # "Tôi sẵn sàng nhận traffic chưa? Nếu fail → bỏ khỏi Service endpoints."
  httpGet: { path: /readyz, port: 8000 }
  periodSeconds: 5
startupProbe:     # "Tôi đang khởi động (chậm). Đừng kill tôi vội."
  httpGet: { path: /healthz, port: 8000 }
  failureThreshold: 30
  periodSeconds: 10
```

Liveness sai → cascading restart loop. Readiness sai → traffic vào pod chưa load model xong. Startup probe là cứu cánh cho LLM mà mất 60–120s để load weight.

### 1.4 — Local cluster: chọn cái nào?

| Tool | Ưu | Nhược | Phù hợp |
|---|---|---|---|
| **kind** | Nhanh, multi-node ảo, CI-friendly | Không hỗ trợ LoadBalancer mặc định | Lab, CI test |
| **minikube** | Lâu đời, có addon (ingress, metrics-server có sẵn) | Nặng hơn kind, driver phụ thuộc OS | Người mới |
| **k3d** | Cực nhẹ (k3s trong Docker), nhanh | k3s ≠ k8s 100% trên một số CRD | Edge demo, dev nhanh |
| **Docker Desktop k8s** | Bật 1 click | Single-node, không reproduce production | Nhanh nhất để chạy thử |
| **Orbstack** (macOS) | Cực nhanh, ăn ít RAM | macOS-only | Lab cá nhân trên Mac |

**Gợi ý:** dùng **kind** cho assignment này — nó cho phép tạo cluster 3-node ảo giống production hơn.

```bash
kind create cluster --config - <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF
```

### 1.5 — Vibe coding với kubectl + AI

K8s YAML chính là loại toil mà AI giải quyết tuyệt vời. Workflow gợi ý:

1. **Mô tả intent** cho AI: *"Generate a Deployment + Service + ConfigMap for an OTel Collector receiving OTLP on 4317/4318, exporting to a Prometheus remote_write at http://prometheus.monitoring:9090/api/v1/write"*.
2. **AI sinh YAML** → bạn `kubectl apply --dry-run=client -f -` để check syntax.
3. **`kubectl explain <resource>.<field>`** để verify từng field AI sinh ra là *thật*, không phải hallucinated.
4. **`kubectl diff -f file.yaml`** trước khi apply lên cluster có state.

Tools đáng học: **k9s** (TUI cho cluster), **kubectx/kubens** (switch context nhanh), **stern** (multi-pod log tail), **kubectl-ai** (LLM-aware kubectl).

> **Anti-pattern:** copy YAML từ AI rồi `kubectl apply` mà không đọc. K8s sẽ vui vẻ apply một Deployment với `replicas: 1000` nếu bạn không nhìn.

---

## §A — Port the Day-23 Observability Stack to K8s

**Bài toán:** chuyển 7-service Compose stack (`app`, `prometheus`, `alertmanager`, `grafana`, `loki`, `jaeger`, `otel-collector`) lên Kubernetes — từ raw manifests tới production-grade với Operator pattern.

### Level A1 — kind cluster + raw manifests (target: 3–6 giờ)

Mục tiêu: stack chạy được trên 1 kind cluster, 1 Deployment + 1 Service cho mỗi service Compose.

**Mapping Compose → K8s:**

| Compose | K8s |
|---|---|
| `service: app` | Deployment + Service (ClusterIP) + ConfigMap (env) |
| `service: prometheus` | StatefulSet + Service + ConfigMap (prometheus.yml) + PVC |
| `service: grafana` | Deployment + Service + Secret (admin pass) + PVC (datasources, dashboards) |
| `service: loki` | StatefulSet + Service + PVC |
| `service: jaeger` | Deployment + Service (one for collector, one for query UI) |
| `service: otel-collector` | DaemonSet *hoặc* Deployment + Service (OTLP receivers exposed on 4317/4318) |
| `service: alertmanager` | StatefulSet + Service + ConfigMap |
| `volumes:` | PVC với StorageClass `standard` (kind default) |
| `networks: obs` | Tất cả trong namespace `monitoring`, dùng K8s DNS |

**Việc cần làm:**
- Tạo namespace `monitoring`.
- Viết manifest cho từng service (vibe-code ok, nhưng `kubectl explain` từng field bạn không hiểu).
- Verify từng service healthy: `kubectl get pods -n monitoring`.
- Port-forward Grafana: `kubectl port-forward -n monitoring svc/grafana 3000:3000`.
- Xác minh trace + metric + log của app vẫn flow end-to-end (giống Compose).

**Câu hỏi để brainstorm:**
- Tại sao Prometheus phải là StatefulSet mà không phải Deployment?
- Service `ClusterIP` vs `NodePort` vs `LoadBalancer` — bạn dùng cái nào, vì sao?
- Khi pod restart, dữ liệu Prometheus còn không? Còn nếu có PVC. Mất nếu không.

### Level A2 — Helm chart (target: +2–4 giờ)

Đóng gói tất cả manifest A1 vào **một** Helm chart custom (`charts/day23-obs/`), với:
- `values.yaml` cho mọi config (port, replica count, retention, password)
- `templates/` cho mỗi service
- `Chart.yaml` với version semver
- Helm helper: `_helpers.tpl` cho label / selector

Verify: `helm install obs ./charts/day23-obs -n monitoring --create-namespace` thay được toàn bộ A1.

**Bonus:** publish chart lên ChartMuseum hoặc GitHub Pages, install bằng `helm repo add`.

### Level A3 — Prometheus Operator + ServiceMonitor (target: +3–5 giờ)

Đây là idiom production. Thay vì tự maintain Prometheus StatefulSet:

```bash
helm install kube-prom-stack prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  --set grafana.adminPassword=lab123
```

Stack này deploy: Prometheus Operator, Prometheus, Alertmanager, Grafana, kube-state-metrics, node-exporter, default dashboards, default alerts — tất cả 1 lệnh.

**Việc cần làm:**
- Convert app của bạn để Prometheus Operator scrape: gắn label `release: kube-prom-stack` và tạo `ServiceMonitor`:
  ```yaml
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata: { name: inference-api, labels: { release: kube-prom-stack } }
  spec:
    selector: { matchLabels: { app: inference-api } }
    endpoints: [{ port: http, path: /metrics, interval: 15s }]
  ```
- Convert alert rule Compose của bạn sang CRD `PrometheusRule`:
  ```yaml
  apiVersion: monitoring.coreos.com/v1
  kind: PrometheusRule
  metadata: { name: inference-api-rules, labels: { release: kube-prom-stack } }
  spec:
    groups:
      - name: inference-api.slo
        rules:
          - alert: HighErrorBurnRate
            expr: |
              (sum(rate(http_requests_total{status=~"5..",app="inference-api"}[5m]))
               / sum(rate(http_requests_total{app="inference-api"}[5m]))) > 0.05
            for: 5m
            labels: { severity: page }
            annotations: { summary: "Error budget burning at >5% over 5m" }
  ```
- Tự upgrade Operator (`helm upgrade`) và verify zero-downtime cho Prometheus.

### Level A4 — Telemetry stack hoàn chỉnh (target: +2–3 giờ)

Thêm các thành phần "vô danh anh hùng" mà mỗi cluster production đều có:

- **kube-state-metrics** (cluster object inventory: bao nhiêu pod Pending? Deployment nào unavailable?) — đã sẵn trong A3.
- **node-exporter** dạng DaemonSet (CPU, mem, disk, network từng node) — đã sẵn trong A3.
- **cAdvisor** (per-container metrics, đã built vào kubelet — bạn chỉ cần scrape `/metrics/cadvisor` qua kubelet endpoint).
- **OpenTelemetry Operator** thay cho OTel Collector chạy tay:
  ```bash
  helm install opentelemetry-operator open-telemetry/opentelemetry-operator -n monitoring
  ```
  Sau đó:
  ```yaml
  apiVersion: opentelemetry.io/v1beta1
  kind: OpenTelemetryCollector
  metadata: { name: gateway, namespace: monitoring }
  spec:
    mode: deployment
    config:
      receivers: { otlp: { protocols: { grpc: {}, http: {} } } }
      exporters: { prometheusremotewrite: { endpoint: http://kube-prom-stack-prometheus:9090/api/v1/write }, otlp/jaeger: { endpoint: jaeger-collector:4317, tls: { insecure: true } } }
      service: { pipelines: { metrics: { receivers: [otlp], exporters: [prometheusremotewrite] }, traces: { receivers: [otlp], exporters: [otlp/jaeger] } } }
  ```

- **Loki** qua Helm chart `grafana/loki-stack` + Promtail DaemonSet để ship log từ mọi pod.

### Level A5 — Real cloud cluster (target: +4–8 giờ, có chi phí ~$10–30)

Move stack lên 1 cluster cloud thật (GKE/EKS/DOKS/Linode/Civo) — đây là phần *thật sự* phân biệt người làm lab và người làm production.

**Việc cần làm:**
- Provision cluster (Terraform khuyến khích — chính là chỗ tiếp theo của vibe coding).
- Setup Ingress controller (nginx hoặc Traefik).
- TLS qua cert-manager + Let's Encrypt.
- Expose Grafana qua Ingress với BasicAuth hoặc OAuth proxy.
- Persistent storage qua StorageClass của cloud (GP3, pd-ssd, …).
- Test cluster autoscaler: deploy 50 replica của app, xem worker node tăng lên.
- **Tắt cluster khi xong** — đừng để chạy qua đêm để hôm sau nhận hóa đơn.

> **Vibe coding khuyến khích:** Terraform / OpenTofu, kustomize overlays, ArgoCD application manifests. AI lo syntax; bạn lo blast radius (`destroy` của bạn xoá gì?).

---

## §B — AI/LLM Serving trên K8s

**Bài toán:** chạy 1 LLM (hoặc model nhỏ) trên cluster, observe nó qua stack §A, scale nó theo metric thật.

### B.1 — GPU scheduling (nếu có GPU)

Nếu cluster có node GPU:
- Cài **NVIDIA device plugin** (DaemonSet).
- Pod request `nvidia.com/gpu: 1` trong resources.
- Dùng `nodeSelector` hoặc `nodeAffinity` để pin lên GPU node.
- Dùng `taint`/`toleration` để ngăn workload thường schedule lên GPU node.

Nếu không có GPU (cluster lab): vẫn làm được mọi bài tập dưới với model nhỏ (TinyLlama 1.1B, Qwen 0.5B) trên CPU + llama.cpp/Ollama.

### B.2 — Model weights: 3 chiến lược

| Chiến lược | Pros | Cons |
|---|---|---|
| **Bake vào image** | Đơn giản, immutable | Image phình to (5–30GB), pull chậm |
| **PVC mount** | Image nhỏ, share giữa replica | Cần ReadWriteMany hoặc init-copy logic |
| **Init container download từ S3/HF** | Linh hoạt, chọn model qua env | Cold start chậm, phụ thuộc bandwidth |

Production thường: **OCI image** cho model nhỏ, **PVC** cho model lớn share giữa nhiều replica.

### B.3 — Operators serving AI

| Operator | Phù hợp | Đặc trưng |
|---|---|---|
| **KServe** | Production multi-model | InferenceService CRD, autoscale to zero, canary rollout có sẵn |
| **NVIDIA Triton** | High-perf, multi-framework | Concurrent model execution, dynamic batching |
| **vLLM** (chart) | LLM-only, high throughput | PagedAttention, prefix caching |
| **Ray Serve** | Custom pipeline (RAG, agent) | Python-native, deployment graph |
| **TGI** (HuggingFace) | LLM, đơn giản | Continuous batching, dễ deploy |

### B.4 — Tasks

#### B.4.1 — Deploy vLLM as Deployment (Level 1, target: 2–3 giờ)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: vllm, namespace: serving }
spec:
  replicas: 1
  selector: { matchLabels: { app: vllm } }
  template:
    metadata: { labels: { app: vllm } }
    spec:
      containers:
        - name: vllm
          image: vllm/vllm-openai:latest
          args: ["--model", "Qwen/Qwen2.5-0.5B-Instruct", "--port", "8000"]
          ports: [{ containerPort: 8000, name: http }]
          resources:
            requests: { cpu: "1", memory: "4Gi" }
            limits: { cpu: "2", memory: "8Gi" }
          readinessProbe:
            httpGet: { path: /health, port: 8000 }
            initialDelaySeconds: 60   # model load
            periodSeconds: 10
```

- Tạo Service + Ingress, hit endpoint từ ngoài.
- Verify Prometheus scrape `/metrics` của vLLM (qua ServiceMonitor).

#### B.4.2 — HPA trên custom metric (Level 2, target: 2–4 giờ)

HPA mặc định scale theo CPU. Bạn muốn scale theo **p95 latency** hoặc **requests/sec**.

- Cài `prometheus-adapter` để expose Prometheus query làm custom metric API.
- Define rule trong adapter config:
  ```yaml
  rules:
    - seriesQuery: 'vllm_request_latency_seconds{namespace!="",pod!=""}'
      resources: { overrides: { namespace: { resource: namespace }, pod: { resource: pod } } }
      name: { matches: "(.*)_latency_seconds", as: "${1}_p95_latency" }
      metricsQuery: 'histogram_quantile(0.95, sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (le, <<.GroupBy>>))'
  ```
- Tạo HPA:
  ```yaml
  apiVersion: autoscaling/v2
  kind: HorizontalPodAutoscaler
  metadata: { name: vllm, namespace: serving }
  spec:
    scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: vllm }
    minReplicas: 1
    maxReplicas: 5
    metrics:
      - type: Pods
        pods: { metric: { name: vllm_request_p95_latency }, target: { type: AverageValue, averageValue: "2" } }
  ```
- Load test bằng `hey` hoặc `k6`, watch HPA scale.

#### B.4.3 — Canary với Argo Rollouts (Level 3, target: 3–5 giờ)

Replace Deployment bằng `Rollout` CRD:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata: { name: vllm, namespace: serving }
spec:
  replicas: 5
  strategy:
    canary:
      steps:
        - setWeight: 20
        - pause: { duration: 5m }
        - analysis: { templates: [{ templateName: success-rate }] }
        - setWeight: 50
        - pause: { duration: 5m }
        - setWeight: 100
```

Define `AnalysisTemplate` query Prometheus cho success rate; rollout tự rollback nếu < threshold.

#### B.4.4 — KServe ModelMesh, multi-model (Level 4, target: 4–6 giờ)

Cài KServe, deploy 3 model qua `InferenceService`, scale to zero khi idle, route traffic thông minh.

### B.5 — Câu hỏi để brainstorm

- Cold start của LLM (60–120s) đánh bại HPA reactive scaling. Bạn predictive-scale như thế nào — request rate trend? thời gian trong ngày?
- Một replica vLLM giữ KV-cache của session — load balancer round-robin sẽ phá hỏng prefix-cache hit-rate. Bạn xử lý sao? (Hint: sticky session, consistent hash.)
- Model weights 30GB. Mỗi pod restart phải tải lại? Bạn share qua đâu (ReadOnly PVC? sidecar cache?)?

---

## §C — Agent Workloads trên K8s

**Bài toán:** chạy agent (LangGraph, AutoGen, custom) như production workload, không phải script chạy trên laptop.

### C.1 — Nature of agents: stateful, lâu, bursty

Agent ≠ web request. Đặc trưng:
- **Stateful:** giữ context, memory, tool state qua nhiều step.
- **Long-running:** 1 task có thể chạy 30s, 5min, 1h.
- **Bursty:** parallel sub-task tooling (search, code-exec, vector query) trong cùng 1 agent run.
- **Failure modes phức tạp:** stuck in loop, tool timeout, LLM hallucination → cần observability layer riêng.

→ Mapping k8s primitive không trivial. Vài lựa chọn:

| Agent pattern | K8s primitive | Khi dùng |
|---|---|---|
| One-shot batch agent | **Job** | "Chạy agent này 1 lần, gen 1 report, xong" |
| Recurring batch agent | **CronJob** | "Mỗi 1h, agent scrape news, gen summary" |
| Long-running stateful | **StatefulSet** + PVC | Agent giữ memory bền (vector store local, conversation history) |
| Worker pool processing queue | **Deployment** + queue (Redis, NATS, RabbitMQ) | Multiple agent worker pull từ task queue |
| Sub-agent tooling (sandboxed code exec) | **Job** spawn dynamic, hoặc Knative | Mỗi tool call = 1 pod ngắn |

### C.2 — Tasks

#### C.2.1 — Single-shot agent as Job (Level 1, target: 2–3 giờ)

Chọn 1 agent đã viết (lab18/19/20/22 nếu có), wrap thành Docker image, deploy như `Job`:

```yaml
apiVersion: batch/v1
kind: Job
metadata: { name: research-agent-run-001, namespace: agents }
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: agent
          image: my-research-agent:v1
          env:
            - { name: OPENAI_API_KEY, valueFrom: { secretKeyRef: { name: api-keys, key: openai } } }
            - { name: OTEL_EXPORTER_OTLP_ENDPOINT, value: "http://otel-collector.monitoring:4317" }
            - { name: TASK, value: "Research: top 5 vector DBs in 2026" }
```

- Verify Job complete, log retrieved qua `kubectl logs`.
- Verify trace của agent flow tới Jaeger UI.

#### C.2.2 — Multi-step agent as StatefulSet (Level 2, target: 3–5 giờ)

Agent giữ state (conversation history, sub-task progress) trong volume bền:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: chat-agent, namespace: agents }
spec:
  serviceName: chat-agent
  replicas: 3
  selector: { matchLabels: { app: chat-agent } }
  template:
    metadata: { labels: { app: chat-agent } }
    spec:
      containers:
        - name: agent
          image: my-chat-agent:v1
          volumeMounts: [{ name: state, mountPath: /data }]
  volumeClaimTemplates:
    - metadata: { name: state }
      spec: { accessModes: [ReadWriteOnce], resources: { requests: { storage: 10Gi } } }
```

- Mỗi replica (`chat-agent-0`, `-1`, `-2`) có PVC riêng → state persistent.
- Service `headless` (`clusterIP: None`) để client target pod cụ thể qua DNS (`chat-agent-0.chat-agent.agents.svc.cluster.local`).
- Sticky-session router: dùng Istio/Linkerd hoặc nginx Ingress với `consistent-hash` để stable user → stable pod.

#### C.2.3 — Agent telemetry vào stack §A (Level 3, target: 2–4 giờ)

Instrument agent với OpenTelemetry:
- **Trace:** mỗi tool call = 1 span. Mỗi LLM call = 1 span với attribute `gen_ai.system`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`.
- **Metric:** `agent_tool_call_duration_seconds`, `agent_tokens_used_total`, `agent_iterations_per_task`.
- **Log:** structured JSON với `trace_id` để correlate.

Tạo Grafana dashboard riêng cho agent: p95 latency per tool, tokens spent per task type, % task hit max iteration (stuck-in-loop detector).

#### C.2.4 — Worker pool + queue (Level 4, target: 4–6 giờ)

Production agent ở scale: agent run mất 1–10 phút, có 100 task/h. Pattern:
- **NATS** hoặc **Redis Streams** làm task queue.
- **KEDA** scale Deployment theo queue depth.
- Mỗi worker pop 1 task, run agent, push result.
- Idempotency: nếu pod chết giữa run, task quay lại queue (ack-after-complete).

### C.3 — Câu hỏi để brainstorm

- Agent của bạn chạy 5 phút. Pod bị evict ở phút 3 (node maintenance) — bạn lose work hay resume được? Bạn cần checkpoint pattern gì?
- 1 tool call agent là `subprocess.run("rm -rf /tmp/work")`. Bạn cho phép code exec trong pod thế nào *an toàn*? (Hint: gVisor, Kata, ephemeral pod per call.)
- Agent gọi LLM 50 lần/run × $0.002/call → mỗi run $0.10. 100 task/h → $10/h. Bạn observe cost theo team/user/task-type bằng cách nào? (Hint: attribute `team` trên span, sum by attribute.)

---

## §D — Cách nộp / chia sẻ (optional)

Không có rubric. Nhưng nếu bạn muốn cái này thành portfolio piece, nên có:

1. **Repo GitHub public**: `day23-k8s-bonus-<your-handle>`.
2. **README.md** top-level: bạn làm những Level nào, screenshot Grafana / k9s, link Helm chart.
3. **Architecture diagram** (mermaid hoặc excalidraw): cluster topology, namespace, traffic flow.
4. **Demo video 3–5 phút**: chạy 1 incident (kill 1 pod, watch self-heal), 1 deploy (rolling update), 1 scale event (HPA trigger).
5. **`POSTMORTEM.md`** từ 1 lần stack hỏng trong lúc làm: "tôi đã apply cái gì → cluster trông như nào → root cause → cách fix → bài học". Đây là phần *quý nhất* — production engineer được đo bằng postmortem chứ không phải uptime.

---

## §E — Anti-patterns hay gặp

Cảnh báo từ những người đã ngã:

| Anti-pattern | Vì sao tệ | Cách đúng |
|---|---|---|
| `kubectl apply` từ AI mà không đọc | YAML AI sinh có thể có `replicas: 1000`, `imagePullPolicy: Always` trên image 30GB, hoặc privilege escalation | `--dry-run=client`, `kubectl diff`, review từng file |
| Chạy Prometheus như Deployment | Restart = mất hết TSDB data | Luôn StatefulSet + PVC |
| Không set resource limits | 1 pod ăn hết node RAM, OOM kill cả cluster | Luôn `requests` + `limits`, đặc biệt cho LLM workload |
| `latest` tag image | Cluster restart pull image khác, không reproduce được | Pin SHA digest hoặc semver tag |
| Secret trong ConfigMap | Plaintext trên etcd, dump bằng `kubectl get cm -o yaml` | Secret object + tốt nhất là External Secrets Operator |
| 1 namespace cho tất cả | RBAC scope, network policy, resource quota không kiểm soát được | Tách `monitoring`, `serving`, `agents`, `app` |
| HPA scale theo CPU cho LLM | LLM CPU-bound không đại diện cho latency. Cold start 60s không kịp HPA reactive | Scale theo custom metric (queue depth, p95 latency) + predictive |
| Log mọi thing vào Loki | Cost explosion ở scale | Sampling, structured log, drop debug log ở production |
| Operator hand-installed everywhere | Drift giữa env | GitOps (ArgoCD/Flux) — cluster state = git state |

---

## §F — Resources khuyến khích

| Resource | Loại | Dùng cho |
|---|---|---|
| [kubernetes.io/docs](https://kubernetes.io/docs/) | Doc gốc | Reference khi `kubectl explain` không đủ |
| [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) | Tutorial | Hiểu k8s từ dưới lên — *highly recommended* trước khi vibe-code YAML |
| [kind quick start](https://kind.sigs.k8s.io/docs/user/quick-start/) | Tool | Local cluster cho assignment này |
| [Prometheus Operator docs](https://prometheus-operator.dev/) | Operator | Level A3 |
| [kube-prometheus-stack chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) | Helm chart | Level A3-A4 |
| [OpenTelemetry Operator](https://github.com/open-telemetry/opentelemetry-operator) | Operator | Level A4 |
| [KServe docs](https://kserve.github.io/website/) | Operator | Level B4.4 |
| [vLLM on k8s guide](https://docs.vllm.ai/en/latest/deployment/k8s.html) | Guide | Level B4.1 |
| [Argo Rollouts](https://argoproj.github.io/argo-rollouts/) | CRD-based deploy | Level B4.3 |
| [KEDA](https://keda.sh) | Event-driven autoscale | Level C2.4 |
| [k9s](https://k9scli.io/) | TUI | Quality-of-life upgrade massive cho mọi level |
| [Lens](https://k8slens.dev/) | GUI | Nếu bạn thích GUI hơn TUI |
| [kubectl-ai](https://github.com/sozercan/kubectl-ai) | Plugin | LLM-aware kubectl, perfect cho vibe coding |
| [ArgoCD](https://argo-cd.readthedocs.io/) | GitOps | Bonus: deploy stack qua git, không phải `helm install` tay |

---

## §G — Đo độ thành công

Bạn xong không phải khi mọi level tick xong. Bạn xong khi bạn có thể trả lời được những câu này *không phải search*:

1. Khi 1 pod CrashLoopBackOff, bạn debug theo thứ tự nào? (5 lệnh đầu tiên)
2. ServiceMonitor không scrape — checklist root cause của bạn là gì?
3. HPA không scale dù load cao — 3 thứ bạn check đầu tiên?
4. Cluster bạn có 50 worker, 1 worker mất kết nối. Pod trên đó sẽ ra sao? Sau bao lâu?
5. Bạn deploy v2 của vLLM, nó crash khi load model. Rollback của bạn mất bao lâu? *Mới*-deploy có rollback được không nếu PVC schema đổi?
6. Một junior hỏi: *"sao mình không deploy mọi thứ qua Compose cho gọn?"* — bạn trả lời thế nào trong 2 câu?

Nếu bạn trả lời được 4/6 — assignment này là portfolio piece thật của bạn. Nếu < 4 — quay lại Level đầu của mỗi track, đọc kỹ hơn lần 2.

---

*File này là tài liệu sống. K8s evolve nhanh — review mỗi 6 tháng. Operator pattern, CRDs, và GitOps là lớp dày nhất; tools (kind, k9s, KServe) đổi nhanh nhất.*

*Câu hỏi cuối cùng từ tinh thần vibe-coding của day23:*

> *"AI có thể sinh hết YAML cho bạn. Nhưng AI không quyết định được: namespace này có nên tồn tại không? Workload này có nên ở cluster này không? Blast radius của `kubectl apply` lần tới đây là gì?"*
>
> *Đó là phần còn lại của con người. Giữ chặt nó.*
