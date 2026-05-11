# Bonus Challenge — Observe something real (UNGRADED)

> Vietnamese: [`BONUS-CHALLENGE.md`](BONUS-CHALLENGE.md)

**Type:** A space to **point a real observability stack at something you actually care about** — no points, no deadline, no rubric.
**Audience for you:** You play *SRE / Platform engineer* on call for one specific AI system, with one specific person (a teammate, a small-business owner, your future self) who pages you when it breaks.
**Effort target:** 4–8 hours. Pair work (2–3 people) encouraged. Brainstorm first, code second.
**Vibe coding encouraged:** AI handles dashboard JSON, query DSL, and YAML; *you* write what the system should care about and when to wake a human.

> This is where you stop treating observability as "fill in the lab dashboards" and start treating it as **answering one question for one person: how do they know their AI is still working at 2am?** The real reward: a portfolio piece you can point at and say "I instrumented X for Y, here's the dashboard, here's the alert, here's the postmortem" — not "I scraped `/metrics` for 60 seconds."

---

## 5 provocations — pick one, or invent your own

Each provocation has 4 axes: **Audience** (who gets paged) · **Domain knowledge** (what you bring) · **Application objective** (what the system should tell them) · **Real-world output** (a deliverable that ships).

### 1. Wire telemetry into a previous day's lab

> *"You already built a system across 6 days. Now make it visible."*

- **Audience:** Yourself, 3 months from now, debugging your own Day 19 vector store or Day 20 model serving when retrieval quality drops or p99 latency creeps up.
- **Domain knowledge:** You built that lab — you know which 3 numbers actually matter (retrieval recall@10? GPU memory? token/sec? cache hit-rate?) versus the 50 metrics Prometheus will happily emit.
- **Application objective:** A single Grafana dashboard + 1 multi-burn-rate alert that catches the *one failure mode you actually fear* in that prior system. Not all failure modes — just the one most likely to bite you.
- **Real-world output:**
  - Fork of one prior day's lab (Day 18/19/20) with OpenTelemetry instrumentation added (~30 lines)
  - `dashboards/<day-name>.json` — committed as code, not screenshotted
  - 1 SLO + 1 burn-rate alert rule that fires when the failure mode happens
  - `RUNBOOK.md` (~150 words): the alert fired — now what does on-call do?
  - Demo: trigger the failure, alert fires within 5 min, you follow your own runbook

**Brainstorming questions:**
- Of the metrics your prior lab already exposes, which 3 would you actually look at first during an incident? Why those three?
- What's the *non-obvious* failure mode — the one Prometheus default alerts won't catch? (data quality drift, cache-coherency, slow degradation, …)
- Could your alert have caught the failure 10 minutes earlier with a different burn-rate window?

---

### 2. Observability for a Vietnamese business actually using AI

> *"Pick one shop. Imagine their chatbot dying at 9am Monday. What dashboard do they wish they had?"*

- **Audience:** The owner of one specific business with at least one AI feature in their flow — a Hanoi cafe with a Zalo chatbot, a Lazada/Shopee seller running auto-replies, a homestay with OCR-based booking ingestion, a tutoring center using LLM-graded essays. You pick.
- **Domain knowledge:** You know what *they* would consider a bad day — losing 30 orders because the chatbot went down? Replying in English to VN customers? Returning the wrong room price? The technical metrics that matter follow from the business impact, not vice-versa.
- **Application objective:** 2 dashboards (one for the owner — plain language, big numbers; one for whoever maintains the bot — technical) + 2 alerts that route to *different channels* (owner gets SMS-style summary, dev gets full context).
- **Real-world output:**
  - `bonus/business-case.md` — who, what, why, sample failure modes (~300 words)
  - 2 `dashboards/*.json` committed
  - 2 alert rules in `prometheus/rules/`
  - 1 webhook-receiver script that translates the dev alert into a 1-sentence VN summary for the owner
  - Model card / "What I observed": 5 sample alerts the system would have fired this week if it had been running

**Brainstorming questions:**
- For a non-technical owner, *what does "the AI is broken" mean*? Slow? Wrong? Rude? Down? Pick the 2 that matter most and design alerts only for those.
- What metric would catch "the chatbot is replying, but the replies are wrong"? (Hint: it's not in the 4 golden signals.)
- How do you avoid waking up the owner for a transient blip? (multi-window burn-rate, anyone?)

---

### 3. Drift detection on a real Vietnamese dataset

> *"Pick a real dataset. Show me the day it shifts."*

- **Audience:** A team that just deployed a Vietnamese-language model (sentiment, classification, retrieval) and doesn't know yet how to tell when their training distribution stops matching production.
- **Domain knowledge:** You know enough about your dataset's domain (news, e-commerce reviews, weather, exam essays, …) to recognize what a "real" shift looks like — vocabulary drift around Tet, slang changes, new product categories, schema evolution.
- **Application objective:** A reproducible drift-detection pipeline on real public Vietnamese data (VnExpress headlines, Lazada reviews scrape, public weather feeds, …) that runs daily, computes PSI/KL/KS, and emits an alert that means something — not "PSI > 0.1 on feature_47" but "the top-5 most-mentioned product categories have shifted vs last week."
- **Real-world output:**
  - `bonus/dataset-card.md` — what dataset, why it's interesting, what shift you expect/observed
  - `drift/pipeline.py` — fetches, computes drift, emits Prometheus metrics
  - Grafana dashboard with at least 1 panel that an English-speaking grader could understand in 10 seconds
  - 1 alert rule + 1 example fire (real or synthetic injection)
  - Reflection: PSI vs KL vs KS — which surfaced the shift first on *your* data?

**Brainstorming questions:**
- For Vietnamese text, what's the equivalent of "feature drift"? Token frequency? Topic distribution? Embedding centroid distance?
- A Tet-related vocabulary shift is *expected*. How does your alert distinguish "expected seasonal drift" from "the model is now broken"?
- What's the action when the alert fires? Retrain? Re-collect data? Page the data scientist? Make this concrete in your runbook.

---

### 4. Cost observability for a model you actually run

> *"How much did your AI cost yesterday? To the cent. Right now."*

- **Audience:** A budget-conscious founder or course instructor running OpenAI/Anthropic API calls (or self-hosted vLLM/llama.cpp) where surprise bills hurt — small team, no FinOps department.
- **Domain knowledge:** You know what your workload's cost-per-request *should* be, what a spike means (debug loop? user growth? prompt got 3× longer?), and what's an acceptable monthly budget.
- **Application objective:** A real-time `$/request`, `$/user`, `$/day` dashboard with a *hard* budget alert that fires before damage is done — not "you spent $1000 last month" but "at current burn you'll hit budget in 3 days."
- **Real-world output:**
  - Instrumented client wrapper (OpenAI SDK, llama.cpp HTTP, vLLM): emits `tokens_input`, `tokens_output`, `cost_usd`, `model` labels — ~50 lines
  - Cost & tokens dashboard committed as JSON
  - 2 alerts: (a) burn-rate vs monthly budget, (b) per-request anomaly (cost > 3σ of last hour)
  - 1-page summary: 7 days of cost data, top 3 cost-driving endpoints, 1 recommendation that saves money (cache, smaller model, prompt compression)

**Brainstorming questions:**
- Should `model_name` be a Prometheus label? (Hint: cardinality.) How do you handle 5 vendors × 3 models × 100 endpoints?
- What's the *leading indicator* of a runaway-cost bug? (loop with no early-stop, retry storm, prompt-doubling-due-to-tool-recursion)
- For self-hosted models the "cost" is GPU-hours, not API dollars. How do you make those comparable on one dashboard?

---

### 5. Postmortem rehearsal — break your own system on purpose

> *"You can't write a good postmortem you've never lived through."*

- **Audience:** Future-you, your future on-call rotation, and the next person you onboard onto your system.
- **Domain knowledge:** You built the core lab. You know where the seams are.
- **Application objective:** Pick 3 failure modes, inject each one with a chaos script, time how long until detection, how long until mitigation, then write a real postmortem with timeline, root cause, action items.
- **Real-world output:**
  - `bonus/chaos/` — 3 scripts that break the system in 3 distinct ways (kill a service, fill disk, inject latency, poison data, exhaust quota, leak a secret, …)
  - For each: `bonus/postmortems/incident-NN.md` with **timeline**, **detection** (time + which signal), **mitigation**, **root cause**, **action items** — 5 of the 5 sections from §9 of the deck
  - 1 of the 3 incidents must result in a *real change* to your dashboards/alerts (i.e. the action item was implemented, not just listed)
  - Optional but encouraged: video screen-recording (~3 min) showing the alert fire, you opening the dashboard, finding the issue

**Brainstorming questions:**
- Which 3 failure modes? Don't pick 3 of the same kind. Mix infra (kill service) + data (drift) + dependency (slow LLM API).
- Time-to-detect is a metric. Can you make your detection time *shorter* on the second injection of the same failure?
- A good postmortem is blameless and changes the system. What concrete change did this incident force?

---

## What counts as "done"

- A `bonus/` folder in your fork with whichever deliverables your provocation demands.
- A 2-paragraph `bonus/REFLECTION.md` answering: **what surprised you?** and **what would you build next if you had another 8 hours?**
- Public GitHub URL. Submit alongside core lab.

That's it. No autograder. No checklist. Just one real artifact that says *I built this for X to do Y*.

## Why we made this ungraded

Because the moment we put points on it, you'd optimize for points. The skill we're trying to develop is **judgment** — what's worth alerting on, what isn't, who do you wake up, when, why — and grading judgment makes it perform-y instead of real. Build the thing you'd build for yourself if no one were watching.

If you ship something good, send the link. Best 3 of the cohort get featured in the next phase's intro deck.
