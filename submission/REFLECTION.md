# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** Hồ Xuân Phú
**Submission date:** 2026-05-12
**Lab repo URL:** _public GitHub URL_

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (29.4.0)
Compose v2:    OK  (5.1.1)
RAM available: 7.47 GB (OK)
Ports free:    BOUND: [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888]
Report written: D:\AI20K\Day23-Track2-Observability-Lab\00-setup\setup-report.json
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

```json
{"model": "llama3-mock", "input_tokens": 4, "output_tokens": 54, "quality": 0.667, "duration_seconds": 0.2439, "trace_id": "48366a8097b02f733a542401d71bf53d", "event": "prediction served", "level": "info", "timestamp": "2026-05-11T09:21:23.167637Z"}
```

Trace ID: `48366a8097b02f733a542401d71bf53d`

### Tail-sampling math

If your service produced N traces/sec, what fraction did the policy keep? Show the calculation.

Based on the `otel-config.yaml` and `sampling-policies.md`:
The policy keeps:
- 100% of errors
- 100% of slow traces (latency > 2s)
- 1% of healthy traces

Formula:
$$sampled = N \times (P(error) \times 1.0 + P(slow \land \neg error) \times 1.0 + P(healthy) \times 0.01)$$

For a typical traffic scenario (1% errors, 1% slow, 98% healthy):
$$sampled = N \times (0.01 + 0.01 + 0.98 \times 0.01) = N \times 0.0298 \approx 3\%$$

So the fraction kept is approximately **3%** (or a factor of $0.0298$).

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

- **`prompt_length`**: **KS Test**. Since prompt length is a continuous numerical feature (number of characters or tokens), the Kolmogorov-Smirnov (KS) test is ideal because it is non-parametric and does not assume a specific distribution. It is very sensitive to shifts in the shape of the distribution.
- **`embedding_norm`**: **KS Test** (or **MMD**). For the *norm* (a 1D scalar), **KS Test** is efficient and effective. However, if we were monitoring the full high-dimensional embeddings themselves, **MMD** (Maximum Mean Discrepancy) would be the correct choice as it is designed for multi-dimensional data.
- **`response_length`**: **PSI**. While KS also works, Population Stability Index (PSI) is often preferred in production because it provides a single, easily interpretable score with standard thresholds (e.g., > 0.2 indicates significant shift). It works well if we bin the response lengths into buckets.
- **`response_quality`**: **KL Divergence**. Quality scores are often treated as probability distributions or bounded continuous values. KL Divergence is excellent for measuring the information loss between the baseline and current distributions, making it sensitive to degradation in model output quality.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

Assuming we use the stub scripts because prior days were not running locally, the hardest metric to expose in a real production scenario would likely be the **Spark/Delta** metrics from Day 18. Spark metrics often require configuring a Prometheus servlet or using a JMX exporter, which is significantly more complex than simple HTTP scraping of JSON metrics or pushing metrics via a script.

---

## 6. The single change that mattered most

The single change that made the biggest difference was implementing **Tail-Sampling** in the OpenTelemetry Collector. Initially, capturing 100% of traces generates too much data and costs too much in a high-throughput system. By setting up a policy that keeps all errors and slow traces but only 1% of healthy traces, we ensured that we captured the high-information traces (the ones we need for debugging) while keeping infrastructure costs low.

This connects directly to the concept of **sampling strategies** discussed in the deck, specifically moving from head-sampling (which makes decisions before knowing if a trace is interesting) to tail-sampling (which makes decisions after the trace is complete), allowing us to preserve visibility into outliers without the cost of full retention.
