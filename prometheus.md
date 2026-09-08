# Prometheus, Metrics & SLOs

## MELT — Where Metrics Fit in Observability

Observability is usually broken into four signal types, remembered as **MELT**:

| Signal | What it is |
|--------|-----------|
| **M**etrics | Numbers over time — CPU %, request count, latency |
| **L**ogs | A record of what happened on each individual request |
| **E**vents | A discrete change in the system (a deploy, a scaling event) |
| **T**races | The end-to-end journey of one request across services, made of **spans** |

A **trace-id** is created the moment a request enters the system (e.g. at the browser); every hop it makes downstream is a **span** under that same trace-id. This doc is about the first letter: **metrics**, and specifically **Prometheus** — the tool that collects, stores, and lets you query them, paired with **Grafana** for dashboards.

## Pull vs Push

Prometheus is a **pull-based** system: it scrapes each target on a schedule, rather than waiting for targets to push data at it.

| Model | How it works | Example |
|-------|--------------|---------|
| **Pull** (Prometheus) | Prometheus **periodically** hits each target's `/metrics` endpoint | Every 15s, scrape `backend-ip:8080/metrics` |
| **Push** | The source sends data **when an event happens** | A billing system pushing "order placed" |

Because it's pull-based, Prometheus needs to know *what* to scrape — that's **service discovery**. Hardcoding IPs doesn't survive autoscaling, so targets are usually discovered dynamically. In the expense project this is EC2 tag-based discovery, one job per tier:

```yaml
# expense-prometheus-grafana/userdata/prometheus.sh — prometheus.yml scrape_configs
- job_name: "node_exporter"
  ec2_sd_configs:
    - region: ${region}
      port: 9100
      filters:
        - name: "tag:Project"
          values: ["${project_tag}"]
  relabel_configs:
    - source_labels: [__meta_ec2_instance_state]
      regex: running
      action: keep
    - source_labels: [__meta_ec2_tag_Name]
      target_label: instance
```

Every scraped target produces an `up` metric — `1` if the last scrape succeeded, `0` if it didn't. It's the simplest possible health check, and shows up as a label:

```
up{app="prometheus", instance="localhost:9090", job="prometheus"}   1
```

`NodeDown` / `BackendDown` alerts (below) are just `up == 0` — you get "is this thing even reachable" for free from the scrape mechanism itself.

## The Metric Types: Counter, Gauge, Histogram

| Type | Behaviour | Real-world analogy | Example metric |
|------|-----------|---------------------|-----------------|
| **Counter** | Only ever goes **up** (or resets to 0 on restart) | A car's odometer; a power meter | `node_cpu_seconds_total`, `http_requests_total` |
| **Gauge** | Goes **up and down** — an instant snapshot | A speedometer | `node_load1`, `mysql_global_status_threads_connected` |
| **Histogram** | Sorts observations into **buckets**, so you can ask "how many requests finished within X seconds" | A duration bucketed by speed range | `http_request_duration_seconds_bucket` |

A counter alone isn't useful as a raw number — "the odometer reads 45,230 km" tells you nothing about *speed*. You need the **rate of change**.

## `rate()` — Turning a Counter Into a Speed

`rate()` only makes sense on a **counter**. It calculates how fast the counter is increasing, per second, over a time window:

```
odometer: 120km at minute 5, 125km at minute 10
→ 5km in 5min = 60 km/h
```

```yaml
# expense-prometheus-grafana/prometheus/recording_rules.yaml
- record: instance:node_cpu_utilisation:rate5m
  expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

Reading this: `node_cpu_seconds_total{mode="idle"}` is a counter of seconds spent idle. `rate(...[5m])` converts that into "idle-seconds gained per second, averaged over 5 minutes" — a fraction near 1 if the CPU is mostly idle. `100 - (that * 100)` flips idle-fraction into a **CPU utilisation %**. This is why "idle" is the metric CPU dashboards actually key off — busy is just "not idle."

## Instant Vector vs Range Vector

A PromQL query returns one of two shapes:

| Query | Returns |
|-------|---------|
| `mysql_up` | **Instant vector** — one value per series, right now |
| `mysql_up[5m]` | **Range vector** — every sample in the last 5 minutes, per series |

`rate()` and `histogram_quantile()` both need a **range vector** as input (`rate(counter[5m])`), because they need multiple samples over time to compute anything — an instant value alone has no rate.

## Histograms & Percentiles (p50 / p90 / p95 / p99)

A **percentile** answers "what value beats N% of the observations." If you scored 98/100 and 31 people scored higher:

```
percentile = (people who scored less than you / total) * 100
           = (100 - 31 - 1) / 100 * 100 = 68th percentile
```

Applied to request duration, `p95` means: **95% of requests finished at or below this duration.** Take 100 requests sorted by how long they took — the 95th one in that sorted order **is** p95.

Histograms make this queryable by pre-sorting observations into **buckets** (`le` = "less than or equal to" some duration), and `histogram_quantile()` interpolates the percentile from those buckets:

```yaml
# expense-prometheus-grafana/prometheus/recording_rules.yaml
- record: job_route:http_request_duration_seconds:p95
  expr: |
    histogram_quantile(0.95, sum by(job, route, le) (rate(http_request_duration_seconds_bucket[5m])))
```

| Percentile | Meaning |
|------------|---------|
| p50 (median) | Half of requests are faster than this |
| p90 | 90% of requests are faster than this |
| p95 | 95% of requests are faster than this |
| p99 | 99% of requests are faster than this |

**Why not just use the average?** One slow outlier (a 10s request among ninety-nine 1s requests) barely moves the average, but it's exactly the kind of pain a real user hits. Percentiles surface the tail; averages hide it. That's why the alert below gates on p95, not average latency:

```yaml
# expense-prometheus-grafana/prometheus/alerting-rules.yaml
- alert: BackendHighLatencyP95
  expr: job_route:http_request_duration_seconds:p95 > 1
  for: 2m
  labels:
    severity: warning
```

## Black-Box vs White-Box Monitoring

| | **Black-box** | **White-box** |
|---|----------------|----------------|
| Vantage point | External — like a public user | Internal — full infra/app access |
| Knows internals? | No | Yes |
| Checks | Is it up? SSL expiring? Response time? | Metrics, logs, traces, events |
| Tool here | `blackbox_exporter` | `node_exporter`, `mysqld_exporter`, app's own `/metrics` |

The expense project's `blackbox-https` job probes the live site from outside, exactly like a user would, and separately tracks the TLS cert's expiry as a metric:

```yaml
# expense-prometheus-grafana/userdata/prometheus.sh
- job_name: 'blackbox-https'
  metrics_path: /probe
  params:
    module: [http_2xx_tls]
  static_configs:
    - targets:
        - https://${domain_name}
```

```yaml
# expense-prometheus-grafana/prometheus/alerting-rules.yaml
- alert: SSLCertExpiringCritical
  expr: instance:probe_ssl_earliest_cert_expiry:hours_remaining < 72
  for: 1h
  labels:
    severity: critical
```

## Golden Signals, USE, and RED

Three overlapping mental models for "what should I actually alert on":

| Model | Signals | Used for |
|-------|---------|----------|
| **4 Golden Signals** (Google SRE) | Latency, Errors, Traffic, Saturation | General service health |
| **USE** | Utilisation, Saturation, Errors | Resources (CPU, disk, memory) |
| **RED** | Rate, Errors, Duration | Request-driven services (APIs) |

The expense project's backend alert group is explicitly RED-shaped — request **rate**, **error** ratio, request **duration** (p95):

```yaml
# expense-prometheus-grafana/prometheus/alerting-rules.yaml
- name: backend_alerts   # BACKEND APP (RED-based)
  rules:
    - alert: BackendHighErrorRate
      expr: job_route:http_request_errors:ratio5m > 0.05
```

...while the node-tier group (CPU/memory/disk) is USE-shaped — utilisation and saturation of a resource, `HighLoadAverage`, `DiskWillFillIn4Hours`, etc.

## SLI, SLO, SLA

| Term | Full name | What it is | Who it's for |
|------|-----------|-------------|---------------|
| **SLI** | Service Level Indicator | The actual **measured number** — e.g. current success rate, current latency | You, internally |
| **SLO** | Service Level Objective | Your **internal target** for that indicator — e.g. "99.5% of requests succeed" | Your team's own bar |
| **SLA** | Service Level Agreement | The number you've **promised a client**, with penalties for missing it | External, contractual |

The exam analogy from class: SLA is the pass mark you promised your parents; SLI is what you're personally aiming for — usually stricter than the SLA, so you have margin before you actually breach the promise. **SLA ≤ SLO** in practice: you target better than what you've contractually promised, so noise doesn't turn into a penalty.

## Error Budget & Burn Rate

If your SLO is 99.5% success, your **error budget** is the remaining 0.5% — the amount of failure you're *allowed* before you've broken your own objective.

Worked example — 100 requests, SLO 99.5%, 98 succeeded:

```
SLI = 98/100  = 0.98    (98% actual success)
SLO = 99.5/100 = 0.995  (99.5% target)

budget consumed  = (1 - SLI) / (1 - SLO)
                 = 0.02 / 0.005
                 = 4  → 400% of budget used

budget remaining = (SLI - SLO) / (1 - SLO) * 100
                 = (0.98 - 0.995) / 0.005 * 100
                 = -300%
```

Negative remaining budget means you've already blown through it 3x over — you're failing the SLO right now, not just at risk of it.

**Burn rate** is *how fast* you're consuming the budget relative to the SLO's time window (commonly 30 days). If your whole 30-day budget would be consumed in 10 days at the current failure rate, your burn rate is **3x** — you'll run out 3 times faster than the window allows. This is why alerting on burn rate (not just "budget currently negative") matters: a burn rate of 10x for one hour should page someone *immediately*, long before 30 days are up, because waiting for the SLO window to actually close means the damage (and the SLA breach) already happened.

## Recording Rules vs Alerting Rules

Prometheus rule files hold two different things, both loaded via `rule_files:` in `prometheus.yml`:

| | **Recording rule** | **Alerting rule** |
|---|----------------------|----------------------|
| Purpose | Pre-compute an expensive query into a new named metric | Fire a notification when a condition holds |
| Runs | Every `interval` (e.g. 30s) | Every evaluation, checks `for:` duration before firing |
| Example | `instance:node_cpu_utilisation:rate5m` | `HighCPUUsage: ... > 70` |

Recording rules exist so dashboards and alerts both query a cheap, pre-aggregated metric instead of recomputing `histogram_quantile(rate(...))` on every panel load — that's why `alerting-rules.yaml` mostly references names like `instance:node_cpu_utilisation:rate5m` rather than raw metrics: those names *are* recording rules defined in `recording_rules.yaml`.

```yaml
# expense-prometheus-grafana/prometheus/alerting-rules.yaml
- alert: HighCPUUsage
  expr: instance:node_cpu_utilisation:rate5m > 70
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: "CPU usage above 70% on {{ $labels.instance }}"
```

`for: 2m` matters: the condition must hold continuously for 2 minutes before the alert actually fires — a single noisy spike doesn't page anyone.

## Wiring Alerts to Notifications: Alertmanager

Prometheus only *evaluates* alert rules; **Alertmanager** is the separate process that receives firing alerts and decides *who* to notify, based on the `severity` label each rule sets:

```yaml
# expense-prometheus-grafana/userdata/prometheus.sh — alertmanager.yml
route:
  receiver: warning-alerts        # default: Slack only
  routes:
    - match:
        severity: critical
      receiver: critical-alerts   # Slack + email
```

Same alert, two blast radii — a `warning` only pings Slack, but a `critical` alert also emails, because someone needs to see it even if they're not watching Slack right now.

## Quick Reference

| Concept | One-liner |
|---------|-----------|
| **MELT** | Metrics · Logs · Events · Traces — the four observability signal types |
| **Pull model** | Prometheus scrapes targets on a schedule (vs push, where the source sends data) |
| **Service discovery** | Targets found dynamically (e.g. `ec2_sd_configs` by tag), not hardcoded |
| `up` | 1 if the last scrape succeeded, 0 if not — free health check per target |
| **Counter** | Only increases (odometer); needs `rate()` to be meaningful |
| **Gauge** | Instant value, up or down (speedometer) |
| **Histogram** | Buckets observations by `le` (≤) so percentiles can be computed |
| `rate(counter[5m])` | Per-second speed of a counter, averaged over the window |
| Instant vs range vector | One current value vs a window of samples — `rate`/`histogram_quantile` need range |
| **Percentile (pNN)** | NN% of observations are at or below this value |
| `histogram_quantile()` | Interpolates a percentile from histogram bucket counts |
| Why percentile > average | One slow outlier barely moves an average but ruins a user's experience |
| **Black-box monitoring** | Tests from outside, like a user — up/down, SSL expiry, response time |
| **White-box monitoring** | Full internal visibility — metrics, logs, traces |
| **4 Golden Signals** | Latency · Errors · Traffic · Saturation |
| **USE** | Utilisation · Saturation · Errors — for resources |
| **RED** | Rate · Errors · Duration — for request-driven services |
| **SLI** | The actual measured number right now |
| **SLO** | Your internal target for the SLI |
| **SLA** | The client-facing promise, with penalties for missing it |
| Error budget | The allowed failure margin: `1 − SLO` |
| Budget consumed | `(1 − SLI) / (1 − SLO)` |
| Budget remaining | `(SLI − SLO) / (1 − SLO) × 100` |
| **Burn rate** | How fast the budget is being consumed vs the SLO's time window — alert on this, not just current breach |
| **Recording rule** | Pre-computes a cheap named metric on an interval, reused by dashboards/alerts |
| **Alerting rule** | Fires when an expression holds true for `for:` duration |
| `severity` label | Routes the alert — e.g. `warning` → Slack, `critical` → Slack + email |
| **Alertmanager** | Separate process that turns firing alerts into notifications, routed by label |
