# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** Nguyen Hoang Nghia   
**Submission date:** 2026-05-11. 
**Lab repo URL:** https://github.com/Hoang21011/Day23-Track2-Observability-Lab     

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (29.2.1)
Compose v2:    OK  (5.1.0)
RAM available: 7.65 GB (OK)
Ports free:    BOUND: [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888]
Report written: /Users/nghia/Documents/Day23/00-setup/setup-report.json
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/dashboard-overview.png`.
(Note: Dashboard verified via Grafana API in make verify)

### Burn-rate panel

Drop `submission/screenshots/slo-burn-rate.png`.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| _T0_ | killed `day23-app`         | ServiceDown alert active in Alertmanager |
| _T0+90s_ | `ServiceDown` fired   | Slack notification (simulated via webhook log) |
| _T1_ | restored app              | — |
| _T1+60s_ | alert resolved        | Alertmanager status: resolved |

### One thing surprised me about Prometheus / Grafana

The simplicity of provisioning dashboards-as-code using JSON files and Grafana's automatic discovery was very impressive. It allows for a repeatable, version-controlled observability stack that can be spun up in minutes.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `predict -> embed-text → vector-search → generate-tokens` spans.

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```json
{"model": "llama3-mock", "input_tokens": 4, "output_tokens": 54, "quality": 0.82, "duration_seconds": 0.1616, "trace_id": "1b4fb8abc68cc4b4e82a3769514a2d3a", "event": "prediction served", "level": "info", "timestamp": "2026-05-13T10:56:37.882664Z"}
```
**Trace ID:** `1b4fb8abc68cc4b4e82a3769514a2d3a`

### Tail-sampling math

If your service produced 1000 traces/sec, with 5% errors (50) and 2% slow requests (20), the policy would keep:
- **keep-errors:** 50 traces (all errors)
- **keep-slow:** 20 traces (all slow)
- **probabilistic-1pct:** 1% of the remaining 930 healthy/fast traces = 9.3 traces
**Total kept:** ~79.3 traces/sec.
**Retention rate:** 7.93%.

---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

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

### Which test fits which feature?

For each of `prompt_length`, `embedding_norm`, `response_length`, `response_quality`, name the test (PSI / KL / KS / MMD) you'd choose in production and why.

- **prompt_length:** **PSI** (Population Stability Index). It's great for monitoring input volume shifts across buckets (e.g., users starting to send much longer prompts).
- **embedding_norm:** **MMD** (Maximum Mean Discrepancy). Since embeddings are high-dimensional, MMD is better at detecting shifts in the latent space than univariate tests.
- **response_length:** **KS** (Kolmogorov-Smirnov). A non-parametric test that's very sensitive to any change in the distribution shape of continuous numerical data.
- **response_quality:** **KL Divergence**. Useful for measuring how much the current quality distribution (as a probability distribution) deviates from the reference baseline.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

The GPU utilization metrics would be the hardest to expose realistically in a containerized environment without proper NVIDIA Docker runtime and exporter setup. In this lab, we simulated it, but in production, it requires careful hardware-level integration.

---

## 6. The single change that mattered most

> **Grader reads this closest.** What one thing about your stack design — a metric you added, a label you dropped, a panel you reorganized, an alert threshold you tuned — made the biggest difference between "works" and "useful"? Write 1-2 paragraphs. Connect it to a concept from the deck.

The single most impactful change was the **explicit injection of `trace_id` into the structured JSON logs**. While OpenTelemetry provides the "three pillars" (metrics, traces, logs), they are often siloed. By ensuring every log line emitted during an inference request contains the same `trace_id` seen in Jaeger, we enable "seamless jumping" between pillars. When an alert fires in Grafana due to a latency spike (metrics), we can immediately find the specific slow requests in Jaeger (traces), and then see the detailed execution context and error messages in Loki (logs) for those exact same requests.

This design choice directly implements the concept of **High Cardinality Observability** discussed in the deck. Instead of just knowing that "the system is slow," we can pinpoint exactly *which* request was slow and *why* by following the breadcrumbs across different telemetry data types. This reduces the Mean Time to Detection (MTTD) and Mean Time to Resolution (MTTR) significantly.
