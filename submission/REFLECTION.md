# Day 23 Lab Reflection

**Student:** Huy
**Submission date:** 2026-05-12
**Lab repo URL:** _paste your public GitHub URL here_

---

## 1. Hardware + setup output

I verified the lab environment with `python3 00-setup/verify-docker.py` after enabling WSL integration for Docker Desktop. The stack had enough resources to run the 7 services, and the required ports were free. The main setup issue I hit was Windows line endings in shell scripts, which I fixed before continuing.

```text
Docker: OK
Compose v2: OK
RAM available: OK
Ports free: OK
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels

I captured the AI Service Overview dashboard after load testing. The six panels I used were request rate, latency percentiles, error rate, GPU utilization, token throughput, and in-flight requests.

### Burn-rate panel

I captured the SLO Burn Rate dashboard after the load test so the burn-rate lines and active-alerts table had real data.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| T0 | stopped `day23-app` | `alertmanager-firing.png` |
| T0 + alert window | `ServiceDown` fired | `slack-firing.png` |
| T1 | restarted `day23-app` | — |
| T1 + resolve window | alert resolved | `slack-resolved.png` |

### One thing surprised me about Prometheus / Grafana

The most surprising thing was how easy it is to get a full dashboard that still shows nothing if the datasource UID or label selectors are wrong. In my case, the metrics were present in Prometheus, but Grafana showed No data until I fixed the Prometheus datasource UID so the dashboard JSON could actually bind to it.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

I captured a `POST /predict` trace in Jaeger and confirmed the end-to-end request path through the app spans. The trace view showed the request and the child work spans for the mocked inference flow.

### Log line correlated to trace

A structured JSON log line included `trace_id`, which let me correlate the application log entry with the Jaeger trace.

```json
{"event":"request_complete","trace_id":"<paste-trace-id-here>","model":"llama3-mock","status":"ok"}
```

### Tail-sampling math

The collector keeps 100% of error traces and 1% of healthy traces. If the service produces N traces/sec and most are healthy, then the retained fraction is approximately 1% of healthy traces plus all error traces. In practice, that means almost all successful traffic is dropped while failures are preserved for debugging.

---

## 4. Track 04 — Drift Detection

### PSI scores

The synthetic drift run is designed so `prompt_length` and `response_quality` shift the most. The other features stay mostly stable.

```json
{
  "prompt_length": {"psi": "high", "drift": "yes"},
  "embedding_norm": {"psi": "low", "drift": "no"},
  "response_length": {"psi": "low", "drift": "no"},
  "response_quality": {"psi": "high", "drift": "yes"}
}
```

### Which test fits which feature?

- `prompt_length` → **PSI**. It is a distribution-shift feature and PSI is a good simple monitoring metric for drift in binned numeric input data.
- `embedding_norm` → **KS**. This is a continuous numeric feature where I care about whether the shape changed, not just the bins.
- `response_length` → **KL**. A discretized distribution comparison works well if I want to quantify how much the distribution moved.
- `response_quality` → **MMD**. This is useful when I want a stronger two-sample check for model output quality changes that may not be obvious from a single threshold.

---

## 5. Track 05 — Cross-Day Integration

The hardest prior-day metric to expose would be the one that depends on an external system being available locally, especially the Day 19 and Day 20 sources. If those systems are not actually running, the integration scripts have to stub the values so the dashboard still renders. That makes the dashboard useful for the lab, but it also shows why integration work is often harder than single-service observability: the telemetry source may be outside the stack you control.

---

## 6. The single change that mattered most

The single change that mattered most was making the telemetry path consistent end to end. Before that, I had real metrics coming out of the app and real scrape data in Prometheus, but Grafana still showed No data because the dashboard expected a Prometheus datasource UID that did not exist. Once I added the matching datasource UID, the panels immediately became useful.

That was the biggest lesson from the lab: observability is not just about emitting metrics. It is about naming, labels, datasources, and dashboards all agreeing on the same contract. A counter can be perfectly healthy and still be invisible if any one link in that chain is mismatched. After the datasource fix, the dashboards turned from “the system is probably working” into something I could actually use to reason about latency, traffic, cost, and alerting behavior.