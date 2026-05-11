# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** _your name_
**Submission date:** _YYYY-MM-DD_
**Lab repo URL:** _public GitHub URL_

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (29.4.1)
Compose v2:    OK  (5.1.3)
RAM available: 7.62 GB (OK)
Ports free:    OK
Report written: C:\Users\nak11\python\aithucchien\Track-2\Day23-Track2-Observability-Lab\00-setup\setup-report.json
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/dashboard-overview.png`.

### Burn-rate panel

Drop `submission/screenshots/slo-burn-rate.png`.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| _T0_ | killed `day23-app`         | screenshot `alertmanager-firing.png` |
| _T0+90s_ | `ServiceDown` fired   | screenshot `slack-firing.png` |
| _T1_ | restored app              | — |
| _T1+60s_ | alert resolved        | screenshot `slack-resolved.png` |

### One thing surprised me about Prometheus / Grafana

_(2-3 sentences)_

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```
... paste here ...
```

### Tail-sampling math

If your service produced N traces/sec, what fraction did the policy keep? Show the calculation.

---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

```json
... paste here ...
```

### Which test fits which feature?

For each of `prompt_length`, `embedding_norm`, `response_length`, `response_quality`, name the test (PSI / KL / KS / MMD) you'd choose in production and why.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

_(2-3 sentences. If you didn't have prior days running, write about which one would be hardest based on the integration scripts.)_

---

## 6. The single change that mattered most

> **Grader reads this closest.** What one thing about your stack design — a metric you added, a label you dropped, a panel you reorganized, an alert threshold you tuned — made the biggest difference between "works" and "useful"? Write 1-2 paragraphs. Connect it to a concept from the deck.
