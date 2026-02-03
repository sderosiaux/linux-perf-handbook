# Latency Analysis & Tail Performance

Understanding, measuring, and mitigating latency in production systems.

## Why Tail Latency Matters

### The Problem with Averages

Average latency hides the true user experience. If your average is 50ms but 1% of users wait 5 seconds, you have a latency problem invisible to mean-based monitoring.

```
Distribution Example:
    Count
  1000|****
   100|******
    10|********                  *
     1|***********       **********
      +---+----+----+----+----+----+
        10   50  100  500 1000 5000 ms
```

### Percentile Interpretation

| Percentile | Meaning | Use Case |
|------------|---------|----------|
| P50 | Typical experience | General health |
| P95 | 1 in 20 users | Common SLO target |
| P99 | 1 in 100 users | Real tail behavior |
| P99.9 | 1 in 1000 users | High-traffic services |

**Key insight**: At 1M requests/day, P99.9 affects 1,000 requests. At Google scale, P99.99 affects millions of users.

### User Experience Thresholds

- **< 100ms**: Instantaneous
- **100-300ms**: Noticeable delay
- **300ms-1s**: User attention shift
- **> 1s**: Context switch, abandonment risk
- **> 10s**: Task abandonment likely

Google found that 500ms increase in search latency reduced traffic by 20%. Amazon reported every 100ms of latency cost 1% in sales.

## Tail Latency Amplification

### The Fan-Out Problem

In distributed systems, a single request fans out to multiple backends. Overall latency is bounded by the slowest component.

```
                    +---> Service A (P99: 10ms) ----+
User Request ---+---+---> Service B (P99: 10ms) ----+---> Response
                    +---> Service C (P99: 10ms) ----+

Overall P99 != 10ms (much worse)
```

### The Math: P99 with N Dependencies

```
P(all services fast) = p^N

For P99 (p = 0.99) with N services:
- N = 1:   99.0% fast
- N = 10:  90.4% fast  (P90 becomes your P99!)
- N = 100: 36.6% fast

Required per-service percentile = target^(1/N)
For N=100 services: 0.99^(1/100) = 0.99990 (P99.99 per service!)
```

Dean & Barroso's "The Tail at Scale" (2013) showed tail-tolerant techniques are essential at scale because eliminating all latency variability is impractical.

## Coordinated Omission Problem

### What It Is

Coordinated omission occurs when your load generator coordinates with the system under test to avoid measuring bad results.

```
Closed-Loop Load Generator (WRONG):
    Request 1  |----10ms----|
    Request 2               |----10ms----|
    Request N-1                          |---5000ms---|
    Request N                                         |----10ms----|

Measured: Most = 10ms, one = 5000ms
Reality:  Requests N-50 to N were WAITING during 5s stall
```

### Why Most Benchmarks Lie

Traditional generators wait for a response before sending the next request. During a latency spike, the generator stops sending, so only ONE slow request is recorded.

Gil Tene's example:
```
System runs 100s at 100 req/s, then stalls 100s:
- Measured P99.99: 1ms (WRONG!)
- Actual P99.99: 100s
- Off by 5+ orders of magnitude
```

### Open vs Closed Workloads

```
Closed Loop (WRONG):         Open Loop (CORRECT):
+--------+    +--------+     +--------+    +--------+
| Client |--->| Server |     | Client |--->| Server |
|        |<---|        |     | Timer  |--->|        |
+--------+    +--------+     +--------+<---|        |
Wait before next request     Send at constant rate
```

### Mitigation

```bash
# wrk2: Correct latency measurement (-R = constant throughput)
wrk2 -t4 -c100 -d60s -R10000 http://localhost:8080/api

# vegeta: Built-in open loop
echo "GET http://localhost:8080/api" | vegeta attack -rate=1000 -duration=60s | vegeta report

# hey: Rate-limited
hey -z 60s -q 1000 http://localhost:8080/api
```

## Measurement Tools

### HdrHistogram

High Dynamic Range Histogram: fixed memory, O(1) recording (~3-6ns), additive across nodes/time.

```java
Histogram histogram = new Histogram(3600000000L, 3); // 1hr max, 3 sig figs
histogram.recordValue(latencyMicros);
// With coordinated omission correction:
histogram.recordValueWithExpectedInterval(latencyMicros, expectedIntervalMicros);
histogram.getValueAtPercentile(99.0);  // P99
```

**Bindings**: Java, Go (`hdrhistogram-go`), Rust (`hdrhistogram`), Python

### wrk2

```bash
git clone https://github.com/giltene/wrk2 && cd wrk2 && make
./wrk -t4 -c100 -d60s -R10000 http://localhost:8080/

# Output includes corrected percentiles via HdrHistogram
```

### Latency Heat Maps (Brendan Gregg)

Show latency distribution over time, revealing patterns invisible to percentile lines.

```bash
git clone https://github.com/brendangregg/HeatMap
iosnoop -ts > out.iosnoop
./trace2heatmap.pl --unitstime=us out.iosnoop > io_latency.svg
```

```
Heat Map Interpretation:
Time --->
    |         **       ********     <-- Bimodal (two modes)
    |        ****     **********
    |*****************************
    +-----------------------------
Dark = more samples. Patterns reveal GC pauses, cache effects.
```

### bpftrace Latency Histograms

```bash
# Syscall latency
bpftrace -e 'tracepoint:syscalls:sys_enter_read { @start[tid] = nsecs; }
             tracepoint:syscalls:sys_exit_read /@start[tid]/ {
                 @usecs = hist((nsecs - @start[tid]) / 1000);
                 delete(@start[tid]); }'

# Block I/O latency
bpftrace -e 'tracepoint:block:block_rq_issue { @start[args->dev, args->sector] = nsecs; }
             tracepoint:block:block_rq_complete /@start[args->dev, args->sector]/ {
                 @usecs = hist((nsecs - @start[args->dev, args->sector]) / 1000);
                 delete(@start[args->dev, args->sector]); }'
```

## Latency Breakdown Methodology

```
Total Request Latency
+-- Network: DNS, TCP handshake, TLS, transit, kernel stack
+-- Compute: Parsing, business logic, GC pauses, CPU scheduling
+-- I/O: Disk, database, external services
+-- Contention: Locks, thread pool, connection pool exhaustion
```

### Network Analysis

```bash
dig +stats example.com | grep "Query time"
curl -w "DNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTLS: %{time_appconnect}s\nTTFB: %{time_starttransfer}s\n" -o /dev/null -s https://example.com
ss -ti  # TCP socket info with RTT, cwnd
```

### Compute Analysis

```bash
runqlat           # CPU scheduler latency (eBPF)
offcputime -df -p PID 10 | flamegraph.pl --color=io > offcpu.svg
grep "Pause" gc.log | awk '{print $NF}' | sort -n | tail -20  # GC pauses
```

### I/O Analysis

```bash
biolatency -m               # Block I/O histogram
biosnoop -p PID             # Per-process I/O
ext4slower 1                # FS operations > 1ms
```

### Lock Contention

```bash
futexctn -p PID                        # User-space locks (eBPF)
async-profiler -e lock -d 30 -f lock.html PID  # Java
perf lock record -p PID && perf lock report    # Kernel
```

## Queueing Theory Essentials

### Little's Law

```
L = λ × W

L = items in system
λ = arrival rate (items per unit time)
W = average time in system
```

**Practical applications**:
```
Connection pool sizing:
- Target: 1000 req/s at 100ms avg latency
- L = 1000 × 0.1 = 100 concurrent connections needed

Thread pool sizing:
- Target: 500 req/s, 200ms per request
- L = 500 × 0.2 = 100 concurrent threads minimum
```

### M/M/1 Queue

```
ρ = λ/μ = utilization (must be < 1)
Response time: W = 1/(μ - λ) = service_time / (1 - utilization)
```

### Why Utilization > 80% Kills Latency

```
Response Time (multiple of service time):
    10 |                        *
     5 |                    *
     2 |              ****
     1 |*********
       +----+----+----+----+----+
          20%  40%  60%  80% 100%
              Utilization

50% util: 2x    80% util: 5x    90% util: 10x    95% util: 20x
```

**Target 60-70% utilization** for predictable latency. Scale BEFORE 80%.

### The "Killer Microseconds" Problem

Barroso et al. (2017) identified that microsecond-scale latencies (1-100us) are poorly handled by modern systems:

- **Nanoseconds**: CPU caches, registers - hardware handles it
- **Milliseconds**: Disk I/O, network calls - OS scheduling works
- **Microseconds**: Falls in the gap - too slow for spinning, too fast for sleeping

Common microsecond sources: NVMe SSDs (10-100us), RDMA networking (1-10us), in-memory databases. Traditional thread-per-request models fail here.

## Mitigation Patterns

### Hedged Requests

Send duplicate requests to replicas; use first response. Hedge at P95 = only 5% overhead.

```
Request to Replica A: |-------- 200ms (slow) --------|
Hedge at 50ms:               |---- 30ms ----|
                                           ^ use this
Result: 80ms instead of 200ms
```

```go
func hedgedRequest(ctx context.Context, replicas []string) (*Response, error) {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()
    results := make(chan *Response, len(replicas))
    go func() { resp, _ := doRequest(ctx, replicas[0]); if resp != nil { results <- resp } }()
    select {
    case resp := <-results: return resp, nil
    case <-time.After(50 * time.Millisecond):
        go func() { resp, _ := doRequest(ctx, replicas[1]); if resp != nil { results <- resp } }()
    }
    return <-results, nil
}
```

### Tied Requests

Both replicas know about each other. When one starts processing, it cancels the other.

```
Tied Request Flow:
Client ---> Load Balancer
               |
        +------+------+
        v             v
    Replica A     Replica B
    (queue: 5)    (queue: 0)
        |             |
        |        starts work
        X <---- cancel message
                      |
                  response
```

Key insight: Most tail latency comes from queueing, not processing. Tied requests optimize for shortest queue.

### Latency Budgets

Allocate time per component; shed load when exhausted.

```go
func handleRequest(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 500*time.Millisecond)
    defer cancel()
    user, err := userService.Get(ctx, userID)  // Respects remaining budget
    if err != nil { http.Error(w, "timeout", 504); return }
}
```

### Load Shedding

```go
type LoadShedder struct { maxConcurrent, current int64 }
func (ls *LoadShedder) TryAcquire() bool {
    for {
        c := atomic.LoadInt64(&ls.current)
        if c >= ls.maxConcurrent { return false }
        if atomic.CompareAndSwapInt64(&ls.current, c, c+1) { return true }
    }
}
```

## SLO Design for Latency

### Choosing Percentile Targets

```
            Low Traffic      High Traffic
P95         Good default     Acceptable
P99         Ambitious        Good default
P99.9       Overkill         Recommended
```

### Error Budget

```
SLO: 99.9% requests < 200ms over 30 days
Total: 100M requests
Budget: 0.1% = 100,000 slow requests allowed (~3,333/day)
```

### Multi-Window SLOs

```yaml
objectives:
  - window: 1h,  target: 99.0%   # Acute issues
  - window: 24h, target: 99.5%   # Operational health
  - window: 30d, target: 99.9%   # Strategic planning
```

### Prometheus SLI

```promql
sum(rate(http_request_duration_seconds_bucket{le="0.2"}[5m]))
/ sum(rate(http_request_duration_seconds_count[5m]))
```

### Error Budget Policy

| Budget Status | Action |
|---------------|--------|
| > 50% | Normal velocity |
| 25-50% | Cautious deploys |
| 10-25% | Freeze risky changes |
| < 10% | Incident mode |
| Exhausted | No deploys |

## Quick Reference

| Task | Command |
|------|---------|
| Correct HTTP benchmark | `wrk2 -t4 -c100 -d60s -R10000 URL` |
| Rate-limited load test | `vegeta attack -rate=1000 -duration=60s` |
| Block I/O latency | `biolatency -m` |
| Run queue latency | `runqlat` |
| Off-CPU analysis | `offcputime -df -p PID 30 \| flamegraph.pl` |
| TCP timing | `curl -w "..." -o /dev/null -s URL` |
| Heat map | `trace2heatmap.pl data > heatmap.svg` |

## Key Formulas

```
Tail Amplification:    P(all N fast) = p^N
Little's Law:          L = λ × W
M/M/1 Response:        W = service_time / (1 - utilization)
Error Budget:          (1 - SLO) × requests × time_window
```

## References

- Dean & Barroso, "The Tail at Scale" (2013) - CACM
- Barroso et al., "Attack of the Killer Microseconds" (2017) - CACM
- Gil Tene, "How NOT to Measure Latency" - Strange Loop
- HdrHistogram: github.com/HdrHistogram/HdrHistogram
- Brendan Gregg, "Visualizing System Latency" (2010) - ACMQ
- wrk2: github.com/giltene/wrk2
