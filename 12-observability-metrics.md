# Observability & Metrics

Production observability: telemetry architecture, SLI/SLO methodology, distributed tracing, and eBPF-based instrumentation.

## Table of Contents

1. [Observability vs Monitoring](#observability-vs-monitoring)
2. [Three Pillars](#three-pillars-metrics-logs-traces)
3. [OpenTelemetry Architecture](#opentelemetry-architecture)
4. [SLI/SLO Methodology](#slislo-methodology)
5. [Distributed Tracing](#distributed-tracing)
6. [Metrics Best Practices](#metrics-best-practices)
7. [eBPF-Based Observability](#ebpf-based-observability)
8. [Cost of Observability](#cost-of-observability)

---

## Observability vs Monitoring

**Monitoring:** Predefined questions about known-unknowns. Is CPU > 80%? Is the service responding?

**Observability:** Ask arbitrary new questions without shipping code. Debug unknown-unknowns.

| Aspect | Monitoring | Observability |
|--------|------------|---------------|
| Questions | Predefined | Ad-hoc |
| Data | Aggregated | High-cardinality |
| Cardinality | Low (< 100 labels) | High (user_id, request_id) |

**The Cardinality Divide:** Traditional tools (Prometheus, Datadog) optimize for low cardinality. Observability 2.0 tools (Honeycomb, ClickHouse-based) store wide structured events where high cardinality is default.

**Charity Majors' test:** "Do you handle high-cardinality dimensions? If no, you are not doing observability."

---

## Three Pillars: Metrics, Logs, Traces

### Metrics

Numerical measurements aggregated over time. Cheap to store.

```promql
rate(http_requests_total[5m])                                    # Request rate
histogram_quantile(0.99, rate(http_request_duration_bucket[5m])) # p99 latency
```

**Types:** Counter (monotonic), Gauge (point-in-time), Histogram (distribution), Summary (pre-calculated quantiles)

### Logs

Discrete events. Always use structured format:

```json
{"level": "error", "msg": "payment failed", "trace_id": "abc123", "user_id": "user_789"}
```

### Traces

End-to-end request flow. A **trace** contains **spans** linked by trace_id/parent_span_id:

```
[Frontend] ──> [API Gateway] ──> [Order Service] ──> [Payment Service]
  span-1         span-2            span-3             span-4
                    trace-id: abc123
```

Each span contains: trace_id, span_id, parent_span_id, timestamps, attributes, and events.

**Dapper model (Google):** Root span creation ~204ns, non-root ~176ns. Traces stored by trace_id enabling efficient lookups.

---

## OpenTelemetry Architecture

CNCF standard for telemetry. Provides APIs, SDKs, and Collector.

```
┌─────────────────────────────────────────────┐
│  Application + Instrumentation Libraries    │
├─────────────────────────────────────────────┤
│  OpenTelemetry SDK (Traces/Metrics/Logs)    │
└──────────────────┬──────────────────────────┘
                   │ OTLP
┌──────────────────▼──────────────────────────┐
│  Collector: Receivers → Processors → Exporters │
└─────────────────────────────────────────────┘
```

### Auto-Instrumentation

```bash
# Python
opentelemetry-instrument python app.py

# Java
java -javaagent:opentelemetry-javaagent.jar -jar app.jar

# Node.js
node --require @opentelemetry/auto-instrumentations-node/register app.js
```

### Manual Instrumentation

```python
from opentelemetry import trace
tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("process_order") as span:
    span.set_attribute("order.id", order_id)
    with tracer.start_as_current_span("validate_payment"):
        validate(payment)
```

### Collector Config

```yaml
receivers:
  otlp:
    protocols:
      grpc: { endpoint: 0.0.0.0:4317 }

processors:
  batch: { timeout: 5s, send_batch_size: 1000 }
  memory_limiter: { limit_mib: 1000 }

exporters:
  otlp: { endpoint: tempo:4317 }
  prometheus: { endpoint: 0.0.0.0:8889 }

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlp]
```

**Patterns:** Agent (per-host daemonset) or Gateway (centralized behind LB).

### Connectors

Connectors join pipelines, enabling cross-signal processing:

```yaml
connectors:
  spanmetrics:
    histogram:
      explicit:
        buckets: [100ms, 250ms, 500ms, 1s]
    dimensions:
      - name: http.method
      - name: http.route

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [spanmetrics]
    metrics:
      receivers: [spanmetrics]
      exporters: [prometheus]
```

---

## SLI/SLO Methodology

### Service Level Indicators (SLIs)

| Type | Example |
|------|---------|
| Availability | 99.9% requests return non-5xx |
| Latency | 95% requests < 200ms |
| Throughput | 10k req/s sustained |

```promql
# Availability SLI
sum(rate(http_requests_total{status!~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

# Latency SLI
sum(rate(http_request_duration_seconds_bucket{le="0.2"}[5m])) / sum(rate(http_request_duration_seconds_count[5m]))
```

### Error Budgets

Error budget = 100% - SLO. A 99.9% SLO = 0.1% budget.

Over 28 days with 1M requests: 1,000 allowed failures.

- Budget available → Ship features
- Budget exhausted → Freeze changes, fix reliability

### Burn Rate Alerting

Burn rate = error_rate / (1 - SLO). Rate of 10x = budget exhausted in 1/10th of window.

**Multi-window, multi-burn-rate:**

| Severity | Burn Rate | Long Window | Short Window |
|----------|-----------|-------------|--------------|
| Page | 14.4x | 1h | 5m |
| Page | 6x | 6h | 30m |
| Ticket | 3x | 24h | 2h |

```promql
# Fast burn alert (14.4x)
(sum(rate(http_requests_total{status=~"5.."}[1h])) / sum(rate(http_requests_total[1h]))) > (14.4 * 0.001)
and
(sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))) > (14.4 * 0.001)
```

**Why two windows:** Long window prevents temporary spikes from alerting. Short window ensures fast recovery after fix.

### SLO Document Example

```yaml
service: payment-api
owner: payments-team
slos:
  - name: availability
    description: "Successful request ratio"
    target: 99.9%
    window: 28d
    sli: |
      sum(rate(http_requests_total{status!~"5.."}[5m]))
      / sum(rate(http_requests_total[5m]))
    alerts:
      - burn_rate: 14.4
        window: 1h
        severity: page
      - burn_rate: 6
        window: 6h
        severity: page
```

---

## Distributed Tracing

### Context Propagation

**W3C Trace Context:**
```http
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
```
Format: `version-traceid(128bit)-parentid(64bit)-flags`

**B3 (Zipkin):**
```http
X-B3-TraceId: 463ac35c9f6413ad48485a3953bb6124
X-B3-SpanId: 0020000000000001
```

### Backend Comparison

| Feature | Jaeger | Zipkin | Tempo |
|---------|--------|--------|-------|
| Storage | Cassandra, ES | Cassandra, ES, MySQL | Object storage |
| Query | ID, service, tags | ID, service | Trace ID only |
| Cost | Medium | Medium | Low |

### Jaeger Setup

```yaml
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports: ["16686:16686", "4317:4317"]
    environment:
      COLLECTOR_OTLP_ENABLED: "true"
```

### Tempo Setup

```yaml
distributor:
  receivers:
    otlp:
      protocols: { grpc: {}, http: {} }
storage:
  trace:
    backend: s3
    s3: { bucket: tempo-traces, region: us-east-1 }
```

### Sampling Strategies

**Head-based:** Decision at trace start. Simple, may miss rare errors.

**Tail-based:** Decision after completion. Keeps errors and slow requests.

```yaml
processors:
  tail_sampling:
    policies:
      - name: errors
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: slow
        type: latency
        latency: { threshold_ms: 1000 }
      - name: probabilistic    # Must be last
        type: probabilistic
        probabilistic: { sampling_percentage: 10 }
```

**Adaptive sampling:** Jaeger adjusts rates per-service based on traffic volume for consistent coverage.

### Trace Query Examples

```bash
# Jaeger API - find traces by service and duration
curl "http://jaeger:16686/api/traces?service=payment-api&minDuration=500ms"

# Tempo - query by trace ID (primary pattern)
curl "http://tempo:3200/api/traces/1234567890abcdef"

# TraceQL (Grafana Tempo)
{ resource.service.name = "payment-api" && duration > 500ms }
```

---

## Metrics Best Practices

### Cardinality Management

Cardinality = unique label combinations. Each creates a time series.

```python
# BAD: 1M users × 100 endpoints = 100M series
counter.labels(user_id=user.id, endpoint=path).inc()

# GOOD: 100 series
counter.labels(endpoint=path).inc()
```

**Rules:**
- Never use: user_id, email, request_id, IP
- Normalize paths: `/users/123` → `/users/{id}`
- Max 10 values per label on high-volume metrics

### Finding Issues

```promql
topk(10, count by (__name__)({__name__=~".+"}))  # Top metrics by cardinality
count by (path) (http_requests_total)            # High cardinality labels
```

### Histogram Buckets

Align with SLOs. Include bucket at SLO threshold.

```python
# API latency (SLO: 99% < 500ms)
buckets=[0.01, 0.025, 0.05, 0.1, 0.2, 0.3, 0.5, 0.75, 1.0, 2.0]
```

10-15 buckets typical. More = accuracy. Fewer = performance.

### Relabeling

```yaml
metric_relabel_configs:
  - source_labels: [user_id]
    action: labeldrop
  - source_labels: [path]
    regex: '/users/[0-9]+'
    replacement: '/users/{id}'
    target_label: path
```

---

## eBPF-Based Observability

### Beyla (OpenTelemetry eBPF)

Zero-code instrumentation. Language agnostic.

```bash
export BEYLA_OPEN_PORT=8080
export OTEL_EXPORTER_OTLP_ENDPOINT="http://localhost:4317"
./beyla
```

```yaml
# Kubernetes DaemonSet
containers:
  - name: beyla
    image: grafana/beyla:latest
    securityContext: { privileged: true }
    env:
      - name: OTEL_EXPORTER_OTLP_ENDPOINT
        value: "http://otel-collector:4317"
```

**Requirements:** Linux 5.8+ with BTF, or RHEL 4.18.0-348+.

### Pixie

Kubernetes-native. In-cluster processing (< 5% CPU).

```bash
px deploy
px run px/cluster      # Cluster overview
px run px/http_data    # HTTP traffic
```

Protocol decoding: HTTP, MySQL, Postgres, Redis, Kafka.

**PxL script:**
```python
import px
df = px.DataFrame(table='http_events', start_time='-5m')
df = df[df.latency > 100000000]  # nanoseconds
df = df[['time_', 'source', 'destination', 'req_path', 'latency']]
px.display(df)
```

### Parca (Continuous Profiling)

Always-on CPU profiling. Differential flamegraphs.

```yaml
containers:
  - name: parca-agent
    image: ghcr.io/parca-dev/parca-agent:latest
    args: ["--remote-store-address=parca:7070"]
    securityContext: { privileged: true }
```

---

## Cost of Observability

### Overhead

| Method | CPU Overhead | Latency |
|--------|--------------|---------|
| OTel SDK (manual) | 1-3% | < 1ms |
| OTel auto-instrumentation | 3-8% | 1-5ms |
| eBPF (Beyla) | < 2% | < 1ms |

Research: up to 8.4% throughput decrease, 20-30% latency increase in some cases.

### Sampling Trade-offs

| Strategy | Coverage | Cost |
|----------|----------|------|
| 100% traces | Full | Very high |
| Tail sampling (errors + slow) | Targeted | Medium |
| 1% + tail | Balanced | Low |

### Cost Optimization

1. Tail sample: 100% errors, 1-10% success
2. Tiered retention: Hot (7d) → Cold (1y)
3. Drop debug logs in production
4. Normalize labels at collection time

```yaml
processors:
  tail_sampling:
    policies:
      - name: always-errors
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: sample-rest
        type: probabilistic
        probabilistic: { sampling_percentage: 5 }
  filter:
    logs:
      exclude:
        severity_texts: ["DEBUG", "TRACE"]
```

**Cardinality cost:** One unbounded label (user_id) scraped every 15s can cost $10k+/month on commercial platforms.

### Vendor Cost Comparison (10M spans/day, 100M metrics, 100GB logs)

| Vendor | Est. Monthly Cost |
|--------|-------------------|
| Self-hosted (Tempo+Prometheus+Loki) | ~$1,000 |
| Grafana Cloud | ~$1,500 |
| Datadog | ~$5,500 |
| New Relic | ~$4,000 |

---

## Quick Reference

### Environment Variables

```bash
export OTEL_SERVICE_NAME="my-service"
export OTEL_EXPORTER_OTLP_ENDPOINT="http://collector:4317"
export OTEL_TRACES_SAMPLER="parentbased_traceidratio"
export OTEL_TRACES_SAMPLER_ARG="0.1"
```

### Essential PromQL

```promql
rate(http_requests_total[5m])                                              # Request rate
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))  # Error rate
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))   # p99
```

### Sampling Decision Tree

```
Error? → YES → 100%
       → NO  → Latency > SLO? → YES → 100%
                              → NO  → Critical path? → YES → 10-25%
                                                     → NO  → 1-5%
```

### Cardinality Checklist

- [ ] No user IDs, emails, request IDs in labels
- [ ] Paths normalized
- [ ] Max 10-15 histogram buckets
- [ ] Relabeling drops unused labels

---

## References

- [OpenTelemetry Specification](https://opentelemetry.io/docs/specs/)
- [Google SRE Book - SLOs](https://sre.google/sre-book/service-level-objectives/)
- [Google SRE Workbook - Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [Prometheus Best Practices](https://prometheus.io/docs/practices/)
- [Honeycomb Observability Manifesto](https://www.honeycomb.io/blog/observability-a-manifesto)
- [Dapper Paper](https://research.google/pubs/dapper-a-large-scale-distributed-systems-tracing-infrastructure/)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [Grafana Beyla](https://github.com/grafana/beyla)
- [Pixie](https://px.dev/)
- [Parca](https://www.parca.dev/)
