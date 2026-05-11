# Day 23 — Track 2 — Observability Stack Lab

> AICB · Phase 2 · Track 2 · Day 8 (= program Day 23)
>
> *"Model chạy tốt hôm qua — hôm nay accuracy drop 20%. Bạn phát hiện được không, và bao lâu?"*

A 7-service Docker Compose lab that builds the **full open-source observability stack** for an AI service: Prometheus + Grafana + Loki + Jaeger + Alertmanager + OpenTelemetry Collector + an instrumented FastAPI app.

You will:

- Instrument a FastAPI service with **metrics + traces + logs** (RED + USE + AI-specific)
- Build **3 Grafana dashboards** (overview / SLO burn-rate / cost & tokens)
- Wire **2 multi-window multi-burn-rate alerts** → Slack
- Configure **tail-sampling** in OTel Collector (keep all errors + 1% healthy)
- Run **drift detection** (PSI / KL / KS) on a synthetic shifted dataset
- Integrate metrics from prior days (16 cloud, 17 pipelines, 18 lakehouse, 19 vector store, 20 serving, 22 alignment)

## Before you start

1. Read `rubric.md` — know what you're being graded on
2. Read `HARDWARE-GUIDE.md` — confirm your machine can run 7 containers
3. Read `VIBE-CODING.md` — pick a persona (SRE / Platform / Data) before you begin

## Quick start

```bash
git clone <your-fork> && cd Day23-Track2-Observability-Lab
cp .env.example .env       # then edit SLACK_WEBHOOK_URL
make setup                 # one-time: pulls 6 images, runs verify-docker.py
make up                    # start the 7-service stack
make smoke                 # verify all services healthy (~30s after up)
make load                  # run 60s of locust load
make alert                 # kill app → wait for fire → restore → wait for resolve
make drift                 # run PSI/KL/KS on synthetic shifted data
make verify                # rubric gate — exit 0 = ready to submit
make down                  # stop (preserves data)
```

## Track map

| Track | Focus | Time | Key deliverable |
|---|---|---|---|
| `00-setup/`               | Docker pre-flight                | 15 min | `setup-report.json` committed |
| `01-instrument-fastapi/`  | Metrics + traces + logs in app   | 30 min | `/metrics` exposes 6 metric families |
| `02-prometheus-grafana/`  | Scrape, 3 dashboards, alerts     | 45 min | Slack receives fire+resolve |
| `03-tracing-and-logs/`    | OTel Collector + Jaeger + Loki   | 30 min | end-to-end trace screenshot |
| `04-drift-detection/`     | Evidently + PSI/KL/KS math       | 20 min | drift-summary.json |
| `05-integration/`         | Wire prior days 16-22            | 20 min | cross-day dashboard |
| `BONUS-llm-native-obs/`   | Self-hosted Langfuse (optional)  | +30 min | LLM trace from LangChain |
| `BONUS-ebpf-profiling/`   | Pyroscope (Linux/WSL only)       | +30 min | flame graph for app |

**Total core time:** ~2 h. **Bonus:** +1 h.

## Slide → Track mapping

| Deck section | Lab track |
|---|---|
| §1 Evolution                       | reading only |
| §2 Three Pillars + RED/USE         | `01-instrument-fastapi/` |
| §3 Prometheus + Cardinality        | `02-prometheus-grafana/prometheus/` |
| §4 Grafana + Dashboards-as-Code    | `02-prometheus-grafana/grafana/` |
| §5 SLO + Burn-Rate                  | `02-prometheus-grafana/prometheus/rules/slo-burn-rate.yml` |
| §6 Tracing + OTel + Sampling        | `03-tracing-and-logs/` |
| §7 Drift + LLM-native               | `04-drift-detection/` + `BONUS-llm-native-obs/` |
| §8 Cost + Vendors                   | `02-prometheus-grafana/grafana/dashboards/cost-and-tokens.json` |
| §9 Postmortems + Runbooks           | `submission/REFLECTION.md` (write your own) |
| §10 Demo                            | `make demo` |

## What's NOT in the lab (and why)

- **Promtail / log shipping into Loki** — Loki is up but receives no logs by default. Add Promtail as homework or tail OTel filelog receiver. We didn't bake this in to keep the stack at 7 services and avoid Mac/Windows bind-mount fragility.
- **eBPF continuous profiling** — Linux-only kernel feature; lives in `BONUS-ebpf-profiling/` for those who can run it.
- **Multi-tenant security** — anonymous Grafana viewer is `Viewer` role with no real auth. Lab-grade only.

## Creative bonus (UNGRADED)

Past the rubric there's a separate **portfolio-style** bonus: point this observability stack at *something real you care about* (a prior day's lab, a VN business using AI, a real Vietnamese dataset for drift, a model whose cost you want to track, or a chaos-test postmortem of your own system). No points, no rubric — just one shippable artifact.

Full provocations: [`BONUS-CHALLENGE.md`](BONUS-CHALLENGE.md) (tiếng Việt) · [`BONUS-CHALLENGE-EN.md`](BONUS-CHALLENGE-EN.md) (English). Format: brainstorm-first, code-second, pairs OK. Output: 1 portfolio piece that lets you say "I instrumented X for Y, here's the dashboard, here's the alert, here's the postmortem."

## Submission

Public GitHub URL + commits in `submission/screenshots/` and `submission/REFLECTION.md`. Grader runs `make verify` expecting exit code 0.

## Repo structure

```
.
├── Makefile                     ← orchestration
├── docker-compose.yml           ← 7 services
├── README.md / rubric.md / HARDWARE-GUIDE.md / VIBE-CODING.md
├── BONUS-CHALLENGE.md           ← creative bonus brief (tiếng Việt, ungraded)
├── BONUS-CHALLENGE-EN.md        ← creative bonus brief (English, ungraded)
├── .env.example
├── pyproject.toml / requirements.txt
├── 00-setup/                    ← pre-flight
├── 01-instrument-fastapi/       ← FastAPI + Prometheus + OTel + structlog
├── 02-prometheus-grafana/       ← scrape config + alert rules + 3 dashboards
├── 03-tracing-and-logs/         ← OTel Collector + Loki configs
├── 04-drift-detection/          ← PSI/KL/KS + Evidently HTML report
├── 05-integration/              ← scrapers/stubs for Days 16-22
├── BONUS-ebpf-profiling/        ← Pyroscope (Linux/WSL)
├── BONUS-llm-native-obs/        ← self-hosted Langfuse
├── scripts/                     ← verify.py, trigger-alert.sh, lint-dashboards.py
└── submission/                  ← REFLECTION.md + screenshots/
```

## Why this matters

Day 23 is the **integrative day** of Track 2's operations chapter (CH.5 Vận Hành). Every prior day produces an artifact (cloud infra, pipelines, lakehouse, vector store, serving, evals). Day 23 is when you wire telemetry across all of them so Phase 3's enterprise placement starts from a system you can actually operate, not just deploy.
