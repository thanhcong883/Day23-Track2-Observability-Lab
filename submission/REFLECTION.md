# Day 23 Lab Reflection

**Student:** Giang Thành Công
**Submission date:** 2026-06-29
**Lab repo URL:** https://github.com/gtcong12a03/Day23-Track2-Observability-Lab

---

## 1. Hardware + setup output

```
Docker:        OK  (29.4.0)
Compose v2:    OK  (5.1.1)
RAM available: 15.53 GB (OK)
Ports free:    OK
Report written: 00-setup/setup-report.json
```

Machine: Windows 11 Pro, 16 GB RAM, Docker Desktop 29.4.0.
All 7 containers fit comfortably; no port conflicts detected.

---

## 2. Track 02 — Dashboards & Alerts

### 3 dashboards provisioned automatically

Grafana auto-provisioned all three Day 23 dashboards from `/var/lib/grafana/dashboards`:
- **AI Service Overview (Day 23)** — RED metrics: request rate, error rate, p99 latency, active gauges, token counts, GPU utilization.
- **SLO Burn Rate (Day 23)** — 1h/5m multi-window burn rate panels against a 99.9% availability SLO.
- **Cost & Tokens (Day 23)** — $/hr estimate based on token throughput (input + output tokens × price per token).

### Overview dashboard panels

After 60 seconds of Locust load (10 users, 1058 requests, 0 errors):
- `inference_requests_total` rate ≈ 17.6 req/s
- `inference_latency_seconds` p99 ≈ 350 ms (mock backend)
- `inference_active_gauge` rose during load, returned to 0 after
- `inference_tokens_total` (input + output) accumulated throughout load
- `inference_quality_score` stable around 0.82
- `gpu_utilization_percent` (simulated) ~35–65%

### Alert fire + resolve

The `ServiceDown` alert fires when the app's `/healthz` returns non-200 for 30s.
`make alert` sends a `docker stop day23-app` → waits → `docker start day23-app`.
Alertmanager routes to `slack-default` receiver on `#observability`.

| When | What | Evidence |
|---|---|---|
| T0 | killed `day23-app` | `docker stop day23-app` |
| T0+30s | `ServiceDown` pending → firing | Alertmanager UI shows FIRING |
| T1 | restored app | `docker start day23-app` |
| T1+60s | alert resolved | Alertmanager shows RESOLVED |

### One thing that surprised me about Prometheus / Grafana

Prometheus exemplars (enabled via `--enable-feature=exemplar-storage`) allow linking a histogram data point directly to a Jaeger trace. Without exemplars, you would see a spike in p99 latency on the dashboard but have no fast path to the offending trace — you'd have to manually correlate timestamps. Wiring `exemplarTraceIdDestinations` in the Grafana datasource turns that latency spike into a one-click jump to the trace.

---

## 3. Track 03 — Tracing & Logs

### Trace from Jaeger

A `POST /predict` request produces a root server span (auto-instrumented by `FastAPIInstrumentor`) with three child spans:

```
POST /predict                          [root]
  └─ embed-text      (5 ms)           text.length attribute
  └─ vector-search   (10 ms)          k=5 attribute
  └─ generate-tokens (130–250 ms)     gen_ai.usage.input_tokens, gen_ai.usage.output_tokens, gen_ai.response.finish_reason
```

Span attributes follow the **GenAI semantic conventions** (OTel `gen_ai.*` namespace):
- `gen_ai.request.model = "llama3-mock"`
- `gen_ai.usage.input_tokens = 7`
- `gen_ai.usage.output_tokens = 51`
- `gen_ai.response.finish_reason = "stop"`

### Log line correlated to trace

```json
{"event": "prediction served", "log_level": "info", "timestamp": "2026-06-29T07:40:12.345Z",
 "model": "llama3-mock", "input_tokens": 7, "output_tokens": 51, "quality": 0.817,
 "duration_seconds": 0.1473, "trace_id": "acd8faeb5d2c0d53de778c033649ad39"}
```

The `trace_id` (`acd8faeb5d2c0d53de778c033649ad39`) is extracted from the active OTel span context via `format(span.get_span_context().trace_id, "032x")` and injected into structlog. In Grafana Loki, the derived-field regex `"trace_id":"([a-fA-F0-9]+)"` turns every log line into a clickable jump to Jaeger.

### Tail-sampling math

OTel Collector tail-sampling policy: **keep all traces with status=ERROR, keep 1% of healthy traces** (probabilistic sampler p=0.01 on non-error paths).

At 17.6 req/s under load:
- Error traces (forced via `fail=true`): 0 errors in this run → 0 error traces retained by this rule
- Healthy traces: 17.6/s × 0.01 = **0.176 traces/s** kept → ~10-11 traces per minute

Storage impact: instead of 1058 traces/min at full volume, tail-sampling stores ≈11/min — a **99% reduction** with 100% error coverage.

---

## 4. Track 04 — Drift Detection

### PSI scores

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

`prompt_length` shifted from N(50,15) → N(85,20): PSI=3.46 (severe drift, threshold >0.2).
`response_quality` shifted from Beta(8,2) → Beta(2,6): PSI=8.85 (extreme quality degradation).
`embedding_norm` and `response_length` are unchanged: PSI <0.02 (no drift).

### Which test fits which feature?

| Feature | Recommended test | Reason |
|---|---|---|
| `prompt_length` | **KS (Kolmogorov-Smirnov)** | Continuous, unbounded distribution. KS is non-parametric and sensitive to location shifts — ideal for detecting mean shifts in Gaussian-like features without assuming a specific distribution shape. |
| `embedding_norm` | **PSI** | Stable, bounded float. PSI gives a single scalar that is easy to threshold in production alerting. When the distribution is narrow and expected to remain stable, PSI is low-overhead and interpretable. |
| `response_length` | **PSI** | Same reasoning as embedding_norm: bounded positive integer, historically stable. PSI buckets naturally capture count-distribution shifts. |
| `response_quality` | **KL divergence** | Beta-distributed score in [0,1]. KL divergence is asymmetric and captures how much the reference distribution would be surprised by the current distribution — excellent for detecting degradation in quality scores where the reference is a known-good Beta distribution. |

**MMD (Maximum Mean Discrepancy)** is the right choice for high-dimensional embedding vectors (not scalar features) because it captures distributional differences in kernel space without binning — but it's computationally expensive and not used here for 1-D features.

---

## 5. Track 05 — Cross-Day Integration

The cross-day dashboard (`05-integration/full-stack-dashboard.json`) stubs panels for:
- Day 16 (cloud infra cost)
- Day 17 (pipeline throughput)
- Day 18 (lakehouse query latency)
- Day 19 (vector store recall@k)
- Day 20 (llama.cpp token/s)
- Day 22 (alignment evaluation pass rate)

### Which prior-day metric was hardest to expose?

**Day 19 (vector store / Qdrant recall@k)** would be the hardest. Qdrant does not natively expose recall@k as a Prometheus metric — recall is an offline evaluation metric computed by comparing approximate nearest-neighbor results to ground truth. Exposing it requires either: (a) a sidecar that runs periodic benchmark queries against Qdrant and exports results via `prometheus_client`, or (b) integrating it into the inference path and recording it as a custom histogram. The `monitor-day20-llama-cpp.py` script for llama.cpp is easier because llama.cpp's `/metrics` endpoint already speaks Prometheus natively.

---

## 6. The single change that mattered most

**Adding `trace_id` to every structured log line** was the single change that transformed the stack from a collection of loosely-coupled dashboards into a coherent observability system.

Before this change, when a latency spike appeared on the Grafana overview dashboard, the investigation path was: see spike → check Prometheus for which time window → manually scroll through container logs → cross-reference Jaeger by guessing time windows. Each hop was manual and error-prone.

After embedding `trace_id` in structlog output and wiring Loki's derived-field regex in the datasource config, the path collapsed to two clicks: latency spike on dashboard (via Prometheus exemplar) → jump to Jaeger trace → click the `trace_id` in the trace's log panel → see the exact structlog line with model, token counts, quality score, and duration. This is what the deck calls the **"three pillars of observability" convergence** (§7 Tracing + OTel-GenAI): metrics tell you *that* something is wrong, traces tell you *where* in the request it went wrong, logs tell you *why*. The `trace_id` is the foreign key that makes all three pillars queryable from a single entry point — without it, the pillars are siloed and the mean-time-to-resolution stays high even when all the data exists.
