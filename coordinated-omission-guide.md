# Coordinated Omission: Detection and Mitigation Guide

Comprehensive guide to identifying, measuring, and preventing coordinated omission in load testing and production monitoring.

## Executive Summary

**Coordinated omission** is when your measurement system inadvertently synchronizes with the system under test, causing it to avoid measuring during periods of degraded performance. This creates latency measurements that are off by **orders of magnitude** - making systems appear 100-1000x better than reality.

**Impact**: Gil Tene's canonical example shows measured P99.99 of 1ms when actual P99.99 is 100 seconds - a **5+ order of magnitude error**.

**Root cause**: Traditional closed-loop load generators wait for responses before sending the next request. During latency spikes, the generator stops sending requests, coordinating with the server to omit bad measurements.

## The Problem Explained

### Service Time vs Response Time

The fundamental confusion:

```
Response Time = Queue Time + Service Time

┌─────────────────────────────────────┐
│  Response Time (what users see)     │
├──────────────────┬──────────────────┤
│   Queue Time     │   Service Time   │
│ (waiting to run) │ (actual work)    │
└──────────────────┴──────────────────┘
```

**Coordinated omission measures service time but reports it as response time.**

During a server stall:
- Queue time: 0ms → 5000ms (requests pile up)
- Service time: 10ms → 10ms (unchanged)
- Closed-loop generator only measures service time

### Visual Example

```
Timeline with 10 req/s target rate:

Actual arrival schedule (what should happen):
t=0ms    t=100ms  t=200ms  t=300ms  t=400ms  t=500ms
  ↓        ↓        ↓        ↓        ↓        ↓
[ Req 1 ][ Req 2 ][ Req 3 ][ Req 4 ][ Req 5 ][ Req 6 ]

Server stalls at t=200ms for 500ms:

Closed-loop (WRONG):
Req 1: |--10ms--|
Req 2:          |--10ms--|
Req 3:                   |------------ 500ms ------------|
Req 4:                                                    |--10ms--|

Measured: 3 @ 10ms, 1 @ 500ms → P99 = 10ms ✗
Missing: Requests 4-6 queued during stall


Open-loop (CORRECT):
Req 1: |--10ms--|
Req 2:          |--10ms--|
Req 3:                   |------------ 500ms ------------|
Req 4:                   |------------ 500ms ------------|  (queued)
Req 5:                   |------------ 500ms ------------|  (queued)
Req 6:                   |------------ 500ms ------------|  (queued)

Measured: 2 @ 10ms, 4 @ 500ms → P99 = 500ms ✓
```

### Mathematical Impact on Percentiles

Gil Tene's example from "How NOT to Measure Latency":

```
System runs:
- 100 seconds at 100 req/s = 10,000 requests at 1ms each
- 100 second stall
- Total: 10,001 samples over 200 seconds

Closed-loop results (WRONG):
- 10,000 @ 1ms
- 1 @ 100,000ms
- Average: 10.9ms (should be 25 seconds)
- P50: 1ms
- P75: 1ms (should be 50 seconds)
- P99.99: 1ms (should be 100 seconds)

Open-loop results (CORRECT):
- Would capture ~10,000 additional requests queued during stall
- P99.9 captures the stall properly
- Percentiles reflect user experience
```

**Real-world example** from wrk vs wrk2 comparison:
- Traditional wrk P99: 12ms
- wrk2 P99: 2,400ms (200x difference)

## Detection Methods

### 1. Client vs Server Metric Divergence

**Signal**: Server-side P99 looks healthy, client-side P99 is terrible.

```bash
# Server metrics (from APM/instrumentation)
curl -s http://localhost:9090/api/v1/query?query='histogram_quantile(0.99,http_server_duration_seconds_bucket)' | jq .

# Client metrics (from load test)
wrk -t4 -c100 -d60s http://localhost:8080/api  # Suspect if matches server

wrk2 -t4 -c100 -d60s -R1000 http://localhost:8080/api  # Reality check
```

**Why this works**: Server measures when it's running (service time), client measures from send to receive (response time including queue).

**Red flag pattern**:
```
Server P99:  12ms   ✓ Looks great
Client P99:  15ms   ✓ Looks reasonable
wrk2 P99:    1200ms ✗ Reality
```

### 2. Request Timestamp Gap Analysis

**Signal**: Long gaps in request timestamps during latency spikes.

```python
# Analyze request logs
import pandas as pd

df = pd.read_csv('requests.log', parse_dates=['timestamp'])
df['gap'] = df['timestamp'].diff().dt.total_seconds()

# Coordinated omission signature: gaps correlate with high latency
high_latency = df[df['latency'] > 1000]  # > 1s
print(f"Avg gap before slow requests: {df.loc[high_latency.index, 'gap'].mean()}s")
print(f"Avg gap overall: {df['gap'].mean()}s")

# Red flag: 10x+ higher gaps before slow requests
```

Example output:
```
Avg gap before slow requests: 5.2s   ← Generator backed off
Avg gap overall: 0.01s               ← Should be constant
```

### 3. Load Generator Queue Depth

**Signal**: Load generator's request queue grows during server stalls.

```python
# Monitor generator internals
class LoadGenerator:
    def __init__(self, target_rps):
        self.pending = 0  # Track inflight requests

    def send_request(self):
        self.pending += 1
        # Log if pending > threshold
        if self.pending > 100:
            logger.warning(f"Queue depth: {self.pending} - possible CO")
```

**Red flag**: Queue depth spikes when latency spikes = closed-loop behavior.

### 4. Arrival Rate vs Throughput Divergence

**Signal**: Commanded arrival rate doesn't match actual send rate.

```bash
# Target: 1000 req/s
# Closed-loop will drop to ~500 req/s during stalls
# Open-loop maintains 1000 req/s (requests just queue)

# Check actual rate from metrics
rate(http_requests_total[1m])  # Should match target in open-loop
```

**Detection query**:
```promql
# Prometheus: flag when actual < target
(rate(http_requests_total[1m]) / on() target_rps) < 0.9
```

### 5. The "Should" Test

Ask: **"When should request #12345 have been sent?"**

If you can answer precisely (e.g., "t = 12.345 seconds from start"), you have a static schedule (good).

If you answer "after request #12344 completes" (dynamic schedule), you have coordinated omission.

```python
# Static schedule (CORRECT)
send_times = [start_time + i * (1.0 / target_rps) for i in range(n_requests)]
for t in send_times:
    sleep_until(t)
    send_request()

# Dynamic schedule (WRONG - exhibits CO)
for i in range(n_requests):
    send_request()
    wait_for_response()  # Coordinates with server
```

## Production Monitoring Techniques

### 1. Timestamp Injection Pattern

Inject intended send time into requests to measure true latency.

```go
// Client-side
type Request struct {
    IntendedSendTime time.Time  // When it SHOULD have been sent
    ActualSendTime   time.Time  // When it WAS sent
    Payload          []byte
}

func sendRequest(scheduledTime time.Time) {
    req := Request{
        IntendedSendTime: scheduledTime,
        ActualSendTime:   time.Now(),
    }
    // Server computes: latency = response_time - intended_send_time
}
```

Server-side calculation:
```go
func handleRequest(req Request) {
    responseTime := time.Now()

    // Traditional (WRONG - includes client queueing)
    serviceLat := responseTime.Sub(req.ActualSendTime)

    // Correct (includes intended queue time)
    responseLat := responseTime.Sub(req.IntendedSendTime)

    metrics.RecordLatency(responseLat)  // What user experienced
}
```

### 2. HdrHistogram with Expected Interval

Automatically fills in missing samples.

```java
import org.HdrHistogram.Histogram;

Histogram hist = new Histogram(3_600_000_000L, 3); // 1h max, 3 digits
long expectedInterval = 1_000_000; // 1ms in nanoseconds (1000 req/s)

// Record with coordinated omission correction
long latencyNanos = measureLatency();
hist.recordValueWithExpectedInterval(latencyNanos, expectedInterval);

// Histogram auto-generates missing samples:
// If latency = 100ms and interval = 1ms, it records:
// - 1 @ 100ms (actual)
// - 1 @ 99ms  (interpolated - would have queued)
// - 1 @ 98ms  (interpolated)
// - ... down to expected interval
```

**How it works**:
```
Latency: 100ms, Expected interval: 1ms, Target rate: 1000/s

Missing samples = latency / interval = 100
Records: 100ms, 99ms, 98ms, ..., 2ms, 1ms

This reconstructs the queue that formed.
```

### 3. Dual Metric Tracking

Track both service time and response time separately.

```python
# Prometheus instrumentation
from prometheus_client import Histogram

# Service time (without queue)
service_time = Histogram('http_service_seconds',
                         'Time spent processing',
                         buckets=[.001, .005, .01, .05, .1, .5, 1, 5])

# Response time (with queue)
response_time = Histogram('http_response_seconds',
                          'Total time including queue',
                          buckets=[.001, .005, .01, .05, .1, .5, 1, 5, 10, 30])

@app.route('/api')
def handler():
    arrival = time.time()
    wait_for_worker()  # Queue time
    start = time.time()
    result = process()
    end = time.time()

    service_time.observe(end - start)     # Just processing
    response_time.observe(end - arrival)  # Total user experience
```

**Alert on divergence**:
```promql
# When P99 response time >> P99 service time
histogram_quantile(0.99, http_response_seconds_bucket) /
histogram_quantile(0.99, http_service_seconds_bucket) > 5
```

### 4. Saturation Metrics

Track queue depth and utilization to detect backpressure.

```python
# Track concurrency
inflight = 0
max_concurrent = 100

@app.before_request
def before():
    global inflight
    inflight += 1
    metrics.gauge('http_inflight_requests', inflight)

    # Load shed if saturated
    if inflight > max_concurrent * 0.9:
        abort(503, "Server saturated")

@app.after_request
def after(response):
    global inflight
    inflight -= 1
    return response
```

**Queueing theory check**:
```python
# Little's Law: L = λ × W
# If L grows while λ constant → W increasing (queue forming)

avg_inflight = metrics.gauge('http_inflight_requests').avg()
arrival_rate = metrics.counter('http_requests_total').rate()
implied_latency = avg_inflight / arrival_rate

if implied_latency > p99_latency * 2:
    alert("Queue forming - coordinated omission likely in external monitors")
```

## Load Testing Tools Comparison

### Tools That Avoid Coordinated Omission

| Tool | Open-Loop | Constant Rate | HDR Histogram | Production Ready |
|------|-----------|---------------|---------------|------------------|
| wrk2 | ✓ | ✓ (-R flag) | ✓ | ✓ |
| vegeta | ✓ | ✓ (-rate) | ✗ | ✓ |
| Gatling | ✓ | ✓ (open model) | ✗ | ✓ |
| k6 | ✓ | ✓ (scenarios) | ✗ | ✓ |
| Artillery | ✓ | ✓ (arrivalRate) | ✗ | ✓ |
| Locust | ⚠ | ⚠ (constant_throughput) | ✗ | ⚠ |
| Hyperfoil | ✓ | ✓ | ✓ | ✓ |

### Tools That Exhibit Coordinated Omission

| Tool | Why | Fix |
|------|-----|-----|
| wrk | Closed-loop by design | Use wrk2 |
| ab | Closed-loop | Use wrk2/vegeta |
| JMeter | Default thread groups are closed | Use Constant Throughput Timer |
| Locust | Default users wait for response | Use constant_throughput wait time |
| hey | Fixed concurrency without rate limit | Use -q flag for rate limiting |

### wrk2 - The Gold Standard

```bash
# Install
git clone https://github.com/giltene/wrk2
cd wrk2 && make

# Basic usage
./wrk -t4 -c100 -d60s -R10000 http://localhost:8080/api
#      │   │    │     │
#      │   │    │     └─ Rate: 10k req/s (CRITICAL)
#      │   │    └─────── Duration: 60s
#      │   └──────────── Connections: 100
#      └──────────────── Threads: 4

# Output includes corrected percentiles
Running 1m test @ http://localhost:8080/api
  4 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    12.43ms   45.21ms   2.40s    95.23%
    Req/Sec       -        -        -        -
  600000 requests in 1.00m, 120.00MB read
Requests/sec:  10000.00
Transfer/sec:      2.00MB
```

**Key difference from wrk**:
```bash
# wrk (WRONG)
./wrk -t4 -c100 -d60s http://localhost:8080/api
# No -R flag = closed loop = coordinated omission

# wrk2 (CORRECT)
./wrk -t4 -c100 -d60s -R10000 http://localhost:8080/api
# -R flag = constant throughput = accurate measurement
```

### vegeta

```bash
# Install
go install github.com/tsenart/vegeta@latest

# Constant rate attack
echo "GET http://localhost:8080/api" | \
  vegeta attack -rate=1000/s -duration=60s | \
  vegeta report

# Output
Requests      [total, rate, throughput]  60000, 1000.00, 998.32
Duration      [total, attack, wait]      60.1s, 60s, 100ms
Latencies     [mean, 50, 95, 99, max]    12ms, 10ms, 45ms, 123ms, 2.1s
Success       [ratio]                    99.8%

# JSON format for post-processing
echo "GET http://localhost:8080/api" | \
  vegeta attack -rate=1000/s -duration=60s | \
  vegeta encode | \
  vegeta plot > latency.html
```

### Gatling (Open Model)

```scala
import io.gatling.core.Predef._
import io.gatling.http.Predef._

class LoadTest extends Simulation {
  val httpProtocol = http.baseUrl("http://localhost:8080")

  val scn = scenario("Open Loop")
    .exec(http("request").get("/api"))

  setUp(
    // CORRECT: Open model with constant arrival rate
    scn.inject(
      constantUsersPerSec(1000) during(60 seconds)
    )

    // WRONG: Closed model - would exhibit CO
    // scn.inject(atOnceUsers(100))
  ).protocols(httpProtocol)
}
```

### k6

```javascript
import http from 'k6/http';
import { check } from 'k6';

export let options = {
  // Open model - constant arrival rate
  scenarios: {
    constant_request_rate: {
      executor: 'constant-arrival-rate',
      rate: 1000,        // 1000 req/s
      timeUnit: '1s',
      duration: '60s',
      preAllocatedVUs: 100,
      maxVUs: 200,
    },
  },

  // WRONG: per-vu-iterations = closed loop
  // vus: 100,
  // iterations: 10000,
};

export default function () {
  const res = http.get('http://localhost:8080/api');
  check(res, { 'status 200': (r) => r.status === 200 });
}
```

## Correction Techniques

### 1. Post-Recording Correction

For data already collected with coordinated omission.

```java
// HdrHistogram post-correction
Histogram raw = loadHistogramFromFile("co_affected.hdr");
Histogram corrected = raw.copyCorrectedForCoordinatedOmission(1_000_000L); // 1ms interval

System.out.println("Raw P99: " + raw.getValueAtPercentile(99.0) + "μs");
System.out.println("Corrected P99: " + corrected.getValueAtPercentile(99.0) + "μs");
```

**Algorithm**:
```python
def correct_coordinated_omission(samples, expected_interval):
    """Fill in missing samples during high latency periods."""
    corrected = []
    for latency in samples:
        corrected.append(latency)  # Original sample

        # If latency > interval, add missing samples
        missing_count = int(latency / expected_interval)
        for i in range(1, missing_count):
            interpolated = latency - (i * expected_interval)
            corrected.append(max(interpolated, expected_interval))

    return corrected

# Example
samples = [10, 10, 10, 5000, 10]  # ms, one stall
interval = 10  # ms (100 req/s)
corrected = correct_coordinated_omission(samples, interval)

# Original: [10, 10, 10, 5000, 10] → P99 = 10ms
# Corrected: [10, 10, 10, 5000, 4990, 4980, ..., 20, 10, 10] → P99 = 4900ms
```

### 2. At-Recording Correction

Capture correctly from the start.

```java
// HdrHistogram at-recording correction
Histogram hist = new Histogram(3_600_000_000L, 3);
long expectedInterval = 1_000_000L; // 1ms

while (running) {
    long intended = startNanos + (requestNum * expectedInterval);
    sleep_until(intended);

    long start = System.nanoTime();
    sendRequest();
    waitResponse();
    long latency = System.nanoTime() - start;

    // Records latency AND fills in missing samples
    hist.recordValueWithExpectedInterval(latency, expectedInterval);
    requestNum++;
}
```

### 3. Static Schedule with Queueing

Best practice for production-grade load generators.

```go
// Target: 1000 req/s for 60s = 60,000 requests
target_rps := 1000.0
duration := 60 * time.Second
interval := time.Duration(1e9 / target_rps) // nanoseconds

start := time.Now()
for i := 0; i < int(target_rps * duration.Seconds()); i++ {
    scheduled := start.Add(time.Duration(i) * interval)

    // Sleep until scheduled time (or skip if late)
    time.Sleep(time.Until(scheduled))

    // Send request asynchronously (don't wait for response)
    go func(intended time.Time) {
        sent := time.Now()
        resp := sendRequest()
        received := time.Now()

        // True latency includes client queueing
        latency := received.Sub(intended)
        metrics.RecordLatency(latency)

        // Also track queue delay
        queueDelay := sent.Sub(intended)
        if queueDelay > 0 {
            metrics.RecordQueueDelay(queueDelay)
        }
    }(scheduled)
}
```

### 4. Poisson vs Uniform Arrival

Most systems face Poisson arrivals (random, bursty).

```python
import numpy as np

def generate_schedule(target_rps, duration_sec):
    """Generate request schedule."""

    # Uniform (unrealistic but easier to analyze)
    n_requests = int(target_rps * duration_sec)
    uniform = np.linspace(0, duration_sec, n_requests)

    # Poisson (realistic - exponential inter-arrival)
    poisson = []
    t = 0
    while t < duration_sec:
        inter_arrival = np.random.exponential(1.0 / target_rps)
        t += inter_arrival
        poisson.append(t)

    return uniform, poisson

# Uniform: 0.001, 0.002, 0.003, ... (evenly spaced)
# Poisson: 0.0005, 0.0031, 0.0035, ... (bursty)
```

**Key insight**: Poisson arrivals create natural queuing even under light load, making coordinated omission detection harder but more important.

## Production Monitoring Anti-Patterns

### Anti-Pattern 1: Client-Side Only Metrics

```python
# WRONG: Only measure from client perspective
def call_service():
    start = time.time()
    response = http.get(url)
    latency = time.time() - start  # Includes network, queue, processing
    metrics.observe(latency)
```

**Fix**: Inject correlation ID and measure both client and server.

```python
# Client
def call_service():
    correlation_id = generate_id()
    client_start = time.time()
    response = http.get(url, headers={'X-Request-ID': correlation_id})
    client_latency = time.time() - client_start

    # Compare client vs server latency
    server_latency = float(response.headers.get('X-Server-Latency', 0))
    queue_latency = client_latency - server_latency

    metrics.observe('client_latency', client_latency)
    metrics.observe('queue_latency', queue_latency)

# Server
@app.before_request
def before():
    g.server_start = time.time()

@app.after_request
def after(response):
    server_latency = time.time() - g.server_start
    response.headers['X-Server-Latency'] = str(server_latency)
    return response
```

### Anti-Pattern 2: Sampling During High Latency

```python
# WRONG: Adaptive sampling drops samples when busy
if random.random() < (1.0 / current_load):
    metrics.observe(latency)  # Drops more when latency high
```

**Fix**: Always record, use HdrHistogram for efficient storage.

```python
# CORRECT: Record everything, aggregate efficiently
histogram = HdrHistogram(max_value=3600_000_000, sig_figs=3)
histogram.record(latency_nanos)  # O(1), ~3-6ns overhead
```

### Anti-Pattern 3: Percentile Aggregation

```python
# WRONG: Aggregating percentiles across servers
global_p99 = mean([server1.p99, server2.p99, ...])  # Nonsensical
```

**Fix**: Merge histograms or use centralized raw data.

```python
# CORRECT: Merge histograms
from hdrh.histogram import HdrHistogram

merged = HdrHistogram(3_600_000_000, 3)
for server_hist in server_histograms:
    merged.add(server_hist)  # Additive property

global_p99 = merged.get_value_at_percentile(99.0)
```

### Anti-Pattern 4: Closed-Loop Health Checks

```python
# WRONG: Health check waits for response
def health_check():
    try:
        response = http.get(url, timeout=5)
        return response.status == 200
    except:
        return False

# During server stall, health check waits 5s, then fails
# Only ONE failure recorded for entire stall period
```

**Fix**: Independent timer checks expected throughput.

```python
# CORRECT: Monitor expected vs actual throughput
class HealthCheck:
    def __init__(self, expected_rps):
        self.expected = expected_rps
        self.actual_count = 0

    def record_request(self):
        self.actual_count += 1

    def check(self):
        actual_rps = self.actual_count / time_window
        health = actual_rps / self.expected
        self.actual_count = 0
        return health > 0.9  # Alert if < 90% expected throughput
```

## Key Formulas

### Coordinated Omission Impact

```
Let:
- N = requests per stall period
- t_stall = stall duration
- t_service = normal service time
- interval = 1 / target_rps

Closed-loop measures:
- 1 sample at t_stall (the request that hit the stall)
- Reported P99 ≈ t_service

Open-loop measures:
- N = t_stall / interval samples
- Reported P99 ≈ t_stall (if stall > P99 threshold)

Error factor = t_stall / t_service

Example:
- t_stall = 5s, t_service = 10ms, interval = 10ms
- Closed: 1 @ 5s, rest @ 10ms → P99 = 10ms
- Open: 500 @ 5s → P99 = 5s
- Error factor = 5000ms / 10ms = 500x
```

### Required Percentile for N Dependencies

```
Target SLO: P(fast) = p
Number of dependencies: N

Required per-service percentile:
p_service = p^(1/N)

Example:
- Target: P99 (p = 0.99)
- Dependencies: N = 100
- Required: 0.99^(1/100) = 0.99990 = P99.99 per service

If any service has coordinated omission error, target is impossible.
```

### Little's Law with Queue Detection

```
L = λ × W

L = concurrent requests (queue depth)
λ = arrival rate (req/s)
W = average latency (s)

Detection:
If L_measured > λ_target × W_p99 × 1.5:
    # Queue forming, coordinated omission likely in external tests
```

## Troubleshooting Playbook

### Scenario 1: Load Test Shows Low Latency, Production is Slow

**Symptoms**:
- Load test P99: 50ms
- Production P99: 2000ms
- RPS identical

**Diagnosis**:
```bash
# Check if load test uses open loop
grep -i "rate\|throughput\|arrival" load_test_config.yml

# Compare request timestamp gaps
awk '{print $1}' load_test.log | awk 'NR>1 {print $1-prev} {prev=$1}' | sort -n | tail -20

# Long gaps → closed loop → coordinated omission
```

**Fix**:
```bash
# Switch to wrk2/vegeta with explicit rate
wrk2 -t4 -c100 -d60s -R1000 http://target/api
```

### Scenario 2: P99 Improves After Adding Capacity

**Symptoms**:
- 10 servers: P99 = 500ms
- 20 servers: P99 = 50ms
- RPS/server unchanged

**Diagnosis**: Queue time dominates, hitting M/M/1 knee.

```python
# Check utilization
utilization = arrival_rate / service_rate

# M/M/1 response time
response_time = service_time / (1 - utilization)

# If utilization > 0.8, response_time explodes
```

**Fix**: Target 60-70% utilization. This isn't coordinated omission - it's proper queueing theory.

### Scenario 3: Client P99 Matches Server P99

**Symptoms**:
- Client P99: 100ms
- Server P99: 100ms
- But users report 5s delays

**Diagnosis**: Client library exhibits coordinated omission.

```python
# Check client implementation
if "wait_for_response()" in client_code:
    # Closed loop - exhibits CO

# Check for retry logic
if retries and not track_total_latency:
    # Only measures last attempt, not total user wait
```

**Fix**:
```python
# Track from first attempt
def request_with_retries(url, max_retries=3):
    start = time.time()  # User's perspective starts here
    for attempt in range(max_retries):
        try:
            response = http.get(url, timeout=1)
            break
        except Timeout:
            continue
    end = time.time()

    total_latency = end - start  # Include all retries
    metrics.observe(total_latency)
```

### Scenario 4: Percentiles Look Bimodal

**Symptoms**:
- P50: 10ms
- P90: 12ms
- P99: 2000ms (huge jump)

**Diagnosis**: Might be legitimate bimodal behavior OR coordinated omission masking a mode.

```bash
# Generate latency heatmap
trace2heatmap.pl --unitstime=us requests.log > heatmap.svg

# If heatmap shows two clear bands → legitimate bimodal
# If heatmap shows scattered high latencies → investigate CO
```

**Investigation**:
```python
# Check if high latencies cluster in time
high_lat = df[df['latency'] > 1000]
time_clusters = high_lat.groupby(high_lat['timestamp'].dt.floor('10s')).size()

# If clusters → server events (GC, disk stall)
# If uniform distribution → potential CO
```

## References and Further Reading

### Foundational Papers
- Gil Tene, "How NOT to Measure Latency" (2015) - Strange Loop Conference
- Dean & Barroso, "The Tail at Scale" (2013) - CACM
- ScyllaDB, "On Coordinated Omission" (2021)

### Tools
- wrk2: https://github.com/giltene/wrk2
- HdrHistogram: https://github.com/HdrHistogram/HdrHistogram
- vegeta: https://github.com/tsenart/vegeta
- Hyperfoil: https://hyperfoil.io/

### Implementation Guides
- HdrHistogram Java API: http://hdrhistogram.github.io/HdrHistogram/JavaDoc/
- Gatling Open vs Closed Models: https://gatling.io/docs/
- k6 Scenarios: https://k6.io/docs/using-k6/scenarios/

### Community Discussions
- Mechanical Sympathy (Gil Tene): https://groups.google.com/g/mechanical-sympathy/c/icNZJejUHfE
- Brave New Geek, "Everything You Know About Latency Is Wrong": https://bravenewgeek.com/everything-you-know-about-latency-is-wrong/

---

**TL;DR for LLM Troubleshooting**:

1. **Detection**: Server P99 looks great but client P99 is terrible → coordinated omission
2. **Fix**: Use wrk2 (not wrk) with -R flag for constant throughput
3. **Production**: Inject intended send time, measure response_time - intended_time
4. **Math**: 100ms stall at 10 req/s = 1 sample (closed) vs 1000 samples (open) = 1000x error
5. **Red flag**: Request timestamp gaps during latency spikes = generator backed off

Sources:
- [Coordinated Omission - Mechanical Sympathy](https://groups.google.com/g/mechanical-sympathy/c/icNZJejUHfE/m/BfDekfBEs_sJ)
- [On Coordinated Omission - ScyllaDB](https://www.scylladb.com/2021/04/22/on-coordinated-omission/)
- [wrk2 - GitHub](https://github.com/giltene/wrk2)
- [Your Load Generator is Probably Lying to You - High Scalability](http://highscalability.com/blog/2015/10/5/your-load-generator-is-probably-lying-to-you-take-the-red-pi.html)
- [Everything You Know About Latency Is Wrong - Brave New Geek](https://bravenewgeek.com/everything-you-know-about-latency-is-wrong/)
- [Understanding workload models - Artillery](https://www.artillery.io/blog/load-testing-workload-models)
- [Open and closed models - Grafana k6](https://grafana.com/docs/k6/latest/using-k6/scenarios/concepts/open-vs-closed/)
- [HdrHistogram Java Documentation](https://hdrhistogram.github.io/HdrHistogram/JavaDoc/org/HdrHistogram/Histogram.html)
