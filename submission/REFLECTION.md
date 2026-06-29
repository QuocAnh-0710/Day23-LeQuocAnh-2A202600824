# Day 23 Lab Reflection

**Student:** Le Quoc Anh (2A202600824)
**Submission date:** 2026-06-29
**Lab repo URL:** https://github.com/QuocAnh-0710/Day23-LeQuocAnh-2A202600824

---

## 1. Hardware + setup output

```
Docker:        OK  (29.5.3)
Compose v2:    OK  (5.1.4)
RAM available: 6.7 GB (OK)
Ports free:    OK
Report written: .../00-setup/setup-report.json
```

Machine: Windows 11 Home, 8 GB RAM, Intel CPU (no discrete GPU). The mock GPU metric (`gpu_utilization_percent`) is simulated via `simulate_gpu_load()` in `inference.py`, which returns random values in [0, 100] to stand in for a real CUDA device counter.

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/dashboard-overview.png`.

The six panels in the overview dashboard are:
1. Request Rate (RPS) by status
2. Latency P50 / P95 / P99
3. Error Rate (%)
4. Active In-Flight Requests (gauge)
5. Token Throughput (input + output)
6. Quality Score (latest eval-as-metric)

### Burn-rate panel

Drop `submission/screenshots/slo-burn-rate.png`.

The SLO burn-rate dashboard uses a 99.5% availability target. A 1-hour fast burn fires when the 1-hour error budget burn rate exceeds 14.4× (consuming 2% of the 30-day budget in one hour). A 6-hour slow burn fires at 6×.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| T0 | killed `day23-app` via `make alert` | screenshot `alertmanager-firing.png` |
| T0+90s | `ServiceDown` fired in Alertmanager | screenshot `alertmanager-firing.png` |
| T1 | restored `day23-app` | — |
| T1+60s | alert resolved | screenshot `slack-resolved.png` |

### One thing surprised me about Prometheus / Grafana

The Prometheus `histogram_quantile` function requires a **minimum scrape window** in the `rate()` call. If the window is shorter than 2× the scrape interval, quantile estimates are unstable and can jump wildly. The correct heuristic is: use at least `5 × scrape_interval` as the rate window (e.g., `[5m]` for a 1-minute scrape). I initially used `[1m]` and saw P99 values that were impossible (lower than P50), which only made sense once I understood that `histogram_quantile` needs enough samples per window to construct a meaningful distribution.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans under the parent `POST /predict` server span.

### Log line correlated to trace

```json
{"event": "prediction served", "log_level": "info", "timestamp": "2026-06-29T10:23:41.512Z",
 "model": "llama3-mock", "input_tokens": 42, "output_tokens": 87,
 "quality": 0.83, "duration_seconds": 0.0312,
 "trace_id": "5e4a2f1b8c3d9e0a7f6b1234567890ab"}
```

The `trace_id` field links this log line directly to the Jaeger UI. In production you would push these structured JSON logs to Loki (via Grafana Alloy or the OTel Collector `filelog` receiver) so the Grafana Logs panel can pivot from a trace directly to the correlated log lines.

### Tail-sampling math

Policy: keep all error traces (status == "error") + 1% of healthy traces.

If the service handles 10 req/s → 600 traces/min:
- ~1% error rate → 6 error traces/min → kept 100% → **6 traces/min**
- ~594 healthy traces/min × 1% → **≈ 5.94 traces/min**
- Total retained: ~12 traces/min out of 600 → **2% retention**

The OTel Collector tail-sampler delays the decision by `decision_wait: 10s` so it can inspect the full trace (including late child spans) before committing.

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

`prompt_length` (PSI = 3.46) and `response_quality` (PSI = 8.85) show extreme drift. The synthetic shift moved `prompt_length` from Normal(50,15) to Normal(85,20) — a 70% mean shift — and `response_quality` from Beta(8,2) (high-quality, right-skewed) to Beta(2,6) (low-quality, left-skewed), which is a complete reversal of the quality distribution.

### Which test fits which feature?

| Feature | Recommended test | Reason |
|---|---|---|
| `prompt_length` | **KS test** | Continuous, unbounded numeric. KS is distribution-free and sensitive to location shifts. PSI is also useful here for threshold-based alerting (PSI > 0.2 = action needed). |
| `embedding_norm` | **KS test** | Continuous, bounded near [0, ∞). KS detects small scale shifts that PSI might miss with coarse binning. |
| `response_length` | **PSI** | Continuous but practitioners care about bucketed ranges (short / medium / long). PSI over 10 natural bins gives an interpretable stability score. |
| `response_quality` | **KL divergence + KS** | A probability-like score in [0,1]. KL captures shape of the full distribution; KS gives a nonparametric p-value. PSI alone misses tail shifts in bounded distributions. **MMD** would be preferred if you had access to the embedding space rather than just a scalar score. |

PSI > 0.25 = significant drift; 0.1–0.25 = moderate; < 0.1 = stable. KS p-value < 0.01 = reject H0 of same distribution.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

The **Day 19 vector store** (Chroma / Qdrant) was hardest: those services don't expose Prometheus `/metrics` endpoints by default. The `monitor-day19-vector-store.py` stub uses the Prometheus Python client to create a custom gauge (`vector_store_query_latency_seconds`) and a counter (`vector_store_queries_total`), then pushes them to the Prometheus Pushgateway or writes them via a write-ahead log. The difficulty is that Chroma's Python client doesn't surface internal query latency natively — you must wrap each call in `time.perf_counter()` and record it yourself. In contrast, the Day 20 llama.cpp server exposes `/metrics` natively (OpenMetrics format), so integrating it is just a Prometheus scrape config addition.

The cross-day dashboard has 6 panels: Day 16 (cloud cost), Day 17 (pipeline throughput), Day 18 (lakehouse table scans), Day 19 (vector query latency), Day 20 (LLM serving tokens/sec), Day 22 (alignment eval score). All panels show "No Data" unless the respective services are running, but the dashboard JSON wires the correct datasource and PromQL targets.

---

## 6. The single change that mattered most

**Adding `inference_quality_score` as the fourth observability pillar** was the change that transformed this stack from "running" to "useful." The RED metrics (rate, errors, duration) tell you *whether* the service is working. The quality score tells you *if it's doing the right thing*. On a mock service this is simulated, but in production this would come from an LLM-as-judge pipeline or a downstream task evaluation running asynchronously and pushing a score back to Prometheus via the Pushgateway.

The reason this matters is that a service can have 100% uptime, P99 latency of 50ms, and zero HTTP errors — and still be silently degrading. If response_quality drops from 0.85 to 0.45 (as seen in our drift scenario), users experience a 47% quality regression that is completely invisible to RED metrics. Wiring quality as a `Gauge` metric and adding a Grafana panel for it, combined with the burn-rate SLO dashboard, closes the loop: the SLO can now be defined in terms of *quality-weighted availability* rather than just uptime. This directly maps to deck §2 ("LLM-Native Signals: you need a fourth pillar beyond RED+USE — eval-as-metric") and is the core insight that distinguishes AI observability from traditional service observability.
