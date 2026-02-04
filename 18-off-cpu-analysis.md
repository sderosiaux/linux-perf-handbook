# Off-CPU Analysis & Wall-Clock Profiling

Understanding where your application spends time NOT running on CPU - the hidden 60-90% of latency in I/O-bound systems.

## The Core Problem

**CPU profilers lie for I/O-bound applications.**

```
CPU Profiler View (10s profile):
    parse_json()     40%
    handle_request() 30%
    validate()       20%
    other            10%

Reality (wall-clock):
    [Blocked on I/O]  85%
    parse_json()       6%
    handle_request()   4.5%
    validate()         3%
    other              1.5%
```

**Key insight:** If your service spends 85% of wall-clock time waiting for I/O, CPU profilers only show you the 15% on-CPU portion. You're optimizing the wrong code.

## Decision Tree: Which Profiler?

```
                        ┌─────────────────────┐
                        │  What workload type?│
                        └──────────┬──────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
    ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
    │ CPU-bound        │ │ I/O-bound        │ │ Mixed/Unknown    │
    │ (compute heavy)  │ │ (network, disk)  │ │                  │
    └────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
             │                    │                    │
             ▼                    ▼                    ▼
    ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
    │ Use CPU profiler │ │ Use OFF-CPU      │ │ Use WALL-CLOCK   │
    │ - perf record    │ │ profiler         │ │ profiler         │
    │ - async-profiler │ │ - offcputime     │ │ (on+off combined)│
    │ - py-spy         │ │ - xtop           │ │ - xtop           │
    └──────────────────┘ └────────┬─────────┘ └────────┬─────────┘
                                   │                    │
                                   ▼                    ▼
                    ┌────────────────────────────────────────┐
                    │ Reveals time spent:                    │
                    │ - Waiting for I/O                      │
                    │ - Blocked on locks                     │
                    │ - Sleeping/waiting for events          │
                    │ - Network latency                      │
                    │ - Database query response              │
                    └────────────────────────────────────────┘
```

### Quick Check: Do I Need Off-CPU Analysis?

```bash
# If %idle is high but application is slow → off-CPU problem
top -b -n 1 | head -5

# If run queue is empty but latency is high → off-CPU problem
vmstat 1 3  # Look at 'r' column (should be near 0)

# If I/O wait is high → definitely off-CPU problem
mpstat 1 3  # Look at %iowait

# If context switches are high relative to CPU usage → likely off-CPU
pidstat -w 1 3  # High cswch/s with low %CPU
```

**Thresholds:**
- `%iowait > 20%` → Start with off-CPU analysis
- `cswch/s > 10000` → Likely lock contention or I/O blocking
- `%idle > 50%` AND `p95 latency > 100ms` → Off-CPU bottleneck

## Tools Comparison

| Tool | Overhead | Kernel | Best For | Output |
|------|----------|--------|----------|--------|
| **xtop** | Ultra-low (<1%) | 5.11+ | Production, continuous monitoring | Real-time dashboard |
| **offcputime** | Low-Medium (1-5%) | 4.9+ | Detailed investigation | Flame graph |
| **bpftrace** | Low-Medium (1-5%) | 4.9+ | Custom investigations | Histograms, traces |
| **async-profiler** (Java) | Low (<2%) | Any | Java apps, wall-clock | Flame graph, JFR |
| **perf sched** | Medium (5-10%) | Any | Scheduler analysis | Text report |

**Production safety:**
- **xtop**: Safe for always-on production use
- **offcputime**: Test first, scheduler events can exceed 1M/s
- **bpftrace**: Fine for 30-60s samples, avoid long runs
- **perf sched**: Development/staging only

## Method 1: xtop (0x.tools) - Recommended for Production

**Why xtop?** Samples /proc at 20Hz - no kernel tracing overhead, no eBPF, works everywhere.

### Installation

```bash
# Install from PyPI
pip install 0x-tools

# Or from GitHub
git clone https://github.com/tanelpoder/0xtools
cd 0xtools
pip install -e .
```

**Requirements:** Linux 5.11+ (eBPF task storage)

### Basic Usage

```bash
# Real-time wall-clock top (20Hz sampling)
xtop -c 20

# Focus on specific process
xtop -p $(pgrep myapp)

# Show thread-level detail
xtop -t

# Sample for 60 seconds and exit
xtop -c 20 -d 60
```

### Reading xtop Output

```
PID    COMM           STATE   %WALL  %CPU   %SYS
12345  myapp          R       45%    45%    2%     ← CPU-bound, on-CPU
12346  worker-1       S       30%    5%     1%     ← Mostly blocked (30-5=25%)
12347  worker-2       D       20%    0%     0%     ← 100% I/O wait
12348  gc-thread      R       5%     4%     1%     ← Normal
```

**Key columns:**
- `%WALL` = Wall-clock time percentage (what matters to users)
- `%CPU` = On-CPU time (what CPU profilers see)
- `%SYS` = Kernel mode CPU time
- `STATE` = Current state (R=running, S=sleeping, D=disk wait)

**Gap = Off-CPU time:**
```
Off-CPU% = %WALL - %CPU

worker-1: 30% - 5% = 25% off-CPU (blocked)
worker-2: 20% - 0% = 20% off-CPU (I/O wait)
```

### Actionable Thresholds

```bash
# High off-CPU time → Investigate blocking
if (%WALL - %CPU) > 50%:
    → Check what thread is blocked on

# High %SYS → Syscall-heavy
if %SYS > 20%:
    → Use strace -c or perf trace

# State 'D' (uninterruptible sleep) → I/O problem
if STATE == 'D' and duration > 10ms:
    → Check iostat, biolatency
```

### Production Dashboard Pattern

```bash
# Continuous monitoring with 1-minute snapshots
while true; do
    timestamp=$(date +%s)
    xtop -c 20 -d 60 | tee logs/xtop-$timestamp.log
done

# Alert if any process shows >70% off-CPU time
# (indicates severe blocking)
```

## Method 2: offcputime (BCC Tools)

**Use when:** You need stack traces showing WHERE blocking occurs.

### Installation

```bash
# Ubuntu/Debian
apt install bpfcc-tools

# RHEL/CentOS
yum install bcc-tools
```

### Basic Usage

```bash
# System-wide off-CPU profiling for 30 seconds
offcputime-bpfcc 30

# Specific process
offcputime-bpfcc -p $(pgrep myapp) 30

# User + kernel stacks
offcputime-bpfcc -u -k 30

# Folded format for flame graphs
offcputime-bpfcc -df 30 > offcpu.folded

# Filter by minimum blocking time (1ms)
offcputime-bpfcc -M 1000 30  # microseconds
```

### Generate Flame Graph

```bash
# Clone Brendan Gregg's FlameGraph tools
git clone https://github.com/brendangregg/FlameGraph

# Capture off-CPU stacks (30s)
offcputime-bpfcc -df -p $(pgrep myapp) 30 > offcpu.folded

# Generate flame graph (blue color for I/O)
./FlameGraph/flamegraph.pl --color=io --title="Off-CPU Flame Graph" \
    --countname="microseconds" < offcpu.folded > offcpu.svg

# Open in browser
firefox offcpu.svg
```

### Reading Off-CPU Flame Graphs

```
Stack trace shows WHERE blocked:
  main() → handle_request() → query_db() → read() → [kernel] → schedule()
                                                            ↑
                                                    Blocked here

Width shows HOW LONG blocked:
  Wider = more total off-CPU time spent in this code path
```

**Look for:**
- **Wide `read()`/`write()` stacks** → I/O bottleneck
- **Wide `futex()` stacks** → Lock contention
- **Wide `poll()`/`epoll_wait()`** → Waiting for events (may be normal)
- **Wide database client code** → Slow queries

### Production Example

```bash
# P99 latency spike investigation
# 1. Capture during high latency period
offcputime-bpfcc -df -p $(pgrep api-server) 60 > spike.folded

# 2. Capture during normal period
offcputime-bpfcc -df -p $(pgrep api-server) 60 > normal.folded

# 3. Diff the two
./FlameGraph/difffolded.pl normal.folded spike.folded | \
    ./FlameGraph/flamegraph.pl --negate --title="Regression" > diff.svg

# Red = more blocking during spike
# Blue = less blocking during spike
```

## Method 3: bpftrace Custom Scripts

**Use when:** You need precise control or custom filtering.

### Off-CPU Time Histogram

```bash
# Show distribution of off-CPU time
bpftrace -e '
tracepoint:sched:sched_switch {
    if (args->prev_state == TASK_RUNNING) { next; }
    @start[args->prev_pid] = nsecs;
}

tracepoint:sched:sched_switch /@start[args->next_pid]/ {
    @offcpu_us = hist((nsecs - @start[args->next_pid]) / 1000);
    delete(@start[args->next_pid]);
}

END { clear(@start); }'
```

### Off-CPU by Process

```bash
# Track total off-CPU time per process (10s)
bpftrace -e '
tracepoint:sched:sched_switch {
    if (args->prev_state == TASK_RUNNING) { next; }
    @start[args->prev_pid] = nsecs;
}

tracepoint:sched:sched_switch /@start[args->next_pid]/ {
    $delta = nsecs - @start[args->next_pid];
    @offcpu_ms[args->next_comm] = sum($delta / 1000000);
    delete(@start[args->next_pid]);
}

interval:s:10 { exit(); }
END { clear(@start); }'
```

### Filter by Specific Blocking Reasons

```bash
# Track off-CPU time by wait reason
bpftrace -e '
#include <linux/sched.h>

tracepoint:sched:sched_switch {
    $prev_state = args->prev_state;
    if ($prev_state == 0) { next; }  # TASK_RUNNING

    @start[args->prev_pid] = nsecs;

    # Classify wait reason
    if ($prev_state & 1) {        # TASK_INTERRUPTIBLE
        @wait_type[args->prev_pid] = "sleep";
    } else if ($prev_state & 2) { # TASK_UNINTERRUPTIBLE
        @wait_type[args->prev_pid] = "io_wait";
    }
}

tracepoint:sched:sched_switch /@start[args->next_pid]/ {
    $delta = (nsecs - @start[args->next_pid]) / 1000;
    @offcpu_us[@wait_type[args->next_pid]] = hist($delta);

    delete(@start[args->next_pid]);
    delete(@wait_type[args->next_pid]);
}'
```

## Method 4: Wall-Clock Profiling (Combined On+Off CPU)

**Use when:** You need the complete picture in a single flame graph.

### Python Wall-Clock Profiler (eBPF)

```bash
# Clone wall-clock profiler
git clone https://github.com/eunomia-bpf/bpf-developer-tutorial
cd bpf-developer-tutorial/src/32-wallclock-profiler

# Profile specific process for 30s at 99Hz
sudo python3 wallclock_profiler.py $(pgrep myapp) -d 30 -f 99

# Output: wallclock_<pid>.svg
# Red = on-CPU, Blue = off-CPU (blocked)
```

### Java: async-profiler Wall Mode

```bash
# Download async-profiler
wget https://github.com/async-profiler/async-profiler/releases/latest/download/async-profiler-linux-x64.tar.gz
tar xzf async-profiler-linux-x64.tar.gz

# Wall-clock profiling (includes blocking time)
./asprof -e wall -d 60 -f wall.html $(pgrep java)

# Compare: CPU vs Wall
./asprof -e cpu -d 60 -f cpu.html $(pgrep java)
./asprof -e wall -d 60 -f wall.html $(pgrep java)

# If wall >> cpu → I/O-bound, optimize blocking operations
# If wall ≈ cpu → CPU-bound, optimize compute
```

**Key insight:** async-profiler's wall mode samples all threads at fixed intervals regardless of state. Shows true time distribution.

## Load-Scaling Bottleneck Detection

**Problem:** Flame graphs show what's slow, but not what LIMITS throughput under load.

### The Technique (from P99 CONF)

1. **Instrument pipeline stages with timestamps**
2. **Increase load progressively** (10%, 20%, 50%, 100%)
3. **Track wall-clock % per stage at each load level**
4. **The stage consuming MORE % under load is the bottleneck**

### Example

```python
# Python example: instrument each stage
import time

class PipelineMetrics:
    def __init__(self):
        self.stage_times = defaultdict(float)
        self.total_time = 0

    def record_stage(self, name, start, end):
        self.stage_times[name] += (end - start)

    def print_report(self):
        for stage, duration in self.stage_times.items():
            pct = (duration / self.total_time) * 100
            print(f"{stage}: {pct:.1f}%")

# In your code
metrics = PipelineMetrics()

request_start = time.time()

# Stage 1: Parse
parse_start = time.time()
parse_request(data)
parse_end = time.time()
metrics.record_stage("parse", parse_start, parse_end)

# Stage 2: Database
db_start = time.time()
fetch_from_db(query)
db_end = time.time()
metrics.record_stage("database", db_start, db_end)

# Stage 3: Process
process_start = time.time()
process_results(results)
process_end = time.time()
metrics.record_stage("process", process_start, process_end)

metrics.total_time = time.time() - request_start
```

### Load Test Results Pattern

```
Load Level │ Parse % │ Database % │ Process % │ Bottleneck
───────────┼─────────┼────────────┼───────────┼────────────
  10%      │   20%   │    30%     │    50%    │ process
  20%      │   20%   │    35%     │    45%    │ database (growing)
  50%      │   15%   │    50%     │    35%    │ database (limit)
  100%     │   10%   │    65%     │    25%    │ database (saturated)
          ↑         ↑            ↑
      Shrinks    GROWS       Shrinks
```

**Key insight:** Database % INCREASED from 30% → 65% under load. This is the off-CPU bottleneck limiting throughput. Flame graphs would mislead you to optimize "process" (50% at low load).

### Automated Detection

```bash
# Run load test at different rates
for rate in 100 500 1000 2000; do
    echo "Testing at $rate req/s"
    wrk2 -t4 -c100 -d60s -R$rate http://localhost:8080/api | tee load-$rate.log

    # Capture off-CPU profile during test
    offcputime-bpfcc -df -p $(pgrep myapp) 60 > offcpu-$rate.folded &
    sleep 60

    # Analyze stage percentages from app logs/metrics
    analyze_stage_percentages.py $rate
done

# Compare: which stage's % grows most?
```

## Common Blocking Patterns and Fixes

### Pattern 1: Database Query Waiting

**Symptoms:**
```
Off-CPU flame graph shows:
  query_database() → postgres_client_read() → read() → schedule()

xtop output:
  %WALL=80%, %CPU=10% → 70% off-CPU
```

**Investigation:**
```bash
# Check database query latency
psql -c "SELECT query, mean_exec_time FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;"

# Check connection pool exhaustion
# (High queueing time in app metrics)
```

**Fixes:**
- Add database indexes (check `EXPLAIN ANALYZE`)
- Increase connection pool size (if waiting for connections)
- Add caching layer (Redis, Memcached)
- Denormalize frequently joined tables

### Pattern 2: Lock Contention

**Symptoms:**
```
Off-CPU flame graph shows:
  acquire_lock() → futex() → schedule()

High context switches: pidstat -w shows cswch/s > 10000
```

**Investigation:**
```bash
# Java: Lock profiling
./asprof -e lock -d 60 -f lock.html $(pgrep java)

# C/C++: perf lock
perf lock record -p $(pgrep myapp) -- sleep 30
perf lock report

# Python: py-spy can't see locks directly, use thread dumps
kill -SIGUSR1 $(pgrep python)  # Dump threads
```

**Fixes:**
- Reduce critical section size (hold locks for less time)
- Use read-write locks (many readers, few writers)
- Lock-free data structures (atomic operations)
- Partition data to reduce contention (shard locks)

### Pattern 3: Network I/O

**Symptoms:**
```
Off-CPU flame graph shows:
  http_client_call() → recv() → schedule()

xtop: STATE='S' (sleeping)
```

**Investigation:**
```bash
# Check network latency to backend
curl -w "DNS:%{time_namelookup}s Connect:%{time_connect}s TTFB:%{time_starttransfer}s\n" \
    -o /dev/null -s http://backend/api

# Check TCP retransmissions
ss -ti | grep retrans

# Monitor connection state
ss -s  # Check time-wait, close-wait counts
```

**Fixes:**
- Connection pooling (reuse connections)
- HTTP keep-alive (persistent connections)
- Reduce round-trips (batch requests)
- Move services closer (same AZ/region)
- Increase TCP buffer sizes (high latency networks)

### Pattern 4: File I/O

**Symptoms:**
```
Off-CPU flame graph shows:
  read_file() → read() → [kernel] → schedule()

iostat shows: await > 10ms, %util > 60%
```

**Investigation:**
```bash
# Which files are slow?
fileslower-bpfcc 10  # Files taking >10ms

# Which processes doing I/O?
iotop -aoP

# Block device latency
biolatency-bpfcc -m
```

**Fixes:**
- Add file caching (in-memory cache)
- Use memory-mapped files (mmap)
- Enable filesystem read-ahead
- Upgrade to faster storage (NVMe SSD)
- Use O_DIRECT to bypass page cache (if cache ineffective)

### Pattern 5: Event Polling (May Be Normal)

**Symptoms:**
```
Off-CPU flame graph shows:
  event_loop() → epoll_wait() → schedule()
```

**Question:** Is this a problem?

**Investigation:**
```bash
# Check if event loop is starved
# (Events arriving but not processed quickly)

# Histogram of epoll_wait durations
bpftrace -e '
kprobe:do_epoll_wait { @start[tid] = nsecs; }
kretprobe:do_epoll_wait /@start[tid]/ {
    @epoll_wait_us = hist((nsecs - @start[tid]) / 1000);
    delete(@start[tid]);
}'

# If all waits are >100ms → normal idle waiting
# If waits are <1ms → tight loop, possible CPU waste
```

**Fixes:**
- If idle waiting → Normal, no fix needed
- If tight loop → Increase timeout, use edge-triggered epoll
- If many events → Increase worker threads

## Thresholds and Alerts

### Off-CPU Time Thresholds

```yaml
# Production monitoring rules
rules:
  - alert: HighOffCPUTime
    expr: (wall_clock_time - cpu_time) / wall_clock_time > 0.7
    for: 5m
    annotations:
      summary: "Process spending >70% time blocked ({{ $labels.process }})"
      action: "Check offcputime flame graph for blocking source"

  - alert: IOWaitHigh
    expr: node_cpu_seconds_total{mode="iowait"} / node_cpu_seconds_total > 0.2
    for: 5m
    annotations:
      summary: "System spending >20% time in I/O wait"
      action: "Check iostat and biolatency for storage bottleneck"

  - alert: HighContextSwitches
    expr: rate(node_context_switches_total[1m]) > 100000
    for: 5m
    annotations:
      summary: ">100K context switches/sec (likely lock contention)"
      action: "Check lock profiling or off-CPU analysis"
```

### Investigation Decision Tree

```
High Latency Detected
        │
        ▼
┌─────────────────┐
│ Check CPU usage │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
  Low      High
    │         │
    ▼         ▼
┌─────────┐ ┌──────────┐
│ Off-CPU │ │ On-CPU   │
│ Problem │ │ Problem  │
└────┬────┘ └──────────┘
     │         Use CPU profiler
     │         (perf, async-profiler)
     │
     ▼
Check iowait:
     │
┌────┴────┐
│         │
High    Low
│         │
▼         ▼
I/O      Lock/Event
(iostat) (offcputime)
```

## Production Deployment Checklist

### Phase 1: Baseline (Low Load)

```bash
# Capture baseline off-CPU profile
offcputime-bpfcc -df -p $(pgrep myapp) 60 > baseline.folded

# Capture baseline wall-clock distribution
xtop -p $(pgrep myapp) -c 20 -d 60 > baseline-xtop.log

# Document expected patterns
# - What % off-CPU is normal?
# - Which blocking sources are expected?
```

### Phase 2: Load Testing

```bash
# Progressive load test
for load in 100 500 1000 2000; do
    echo "Testing at $load req/s"

    # Start load generator
    vegeta attack -rate=$load -duration=120s -targets=targets.txt &

    # Capture off-CPU profile mid-test
    sleep 30
    offcputime-bpfcc -df -p $(pgrep myapp) 60 > load-$load.folded

    # Capture xtop snapshot
    xtop -p $(pgrep myapp) -c 20 -d 60 > xtop-$load.log

    wait  # For load generator
    sleep 10
done

# Compare flame graphs: which blocking sources grew?
```

### Phase 3: Production Monitoring

```bash
# Option A: Continuous xtop (ultra-low overhead)
xtop -p $(pgrep myapp) -c 20 | tee -a /var/log/xtop/production.log

# Option B: Periodic snapshots
*/5 * * * * offcputime-bpfcc -df -p $(pgrep myapp) 30 > /var/log/offcpu/$(date +\%s).folded

# Option C: On-demand via trigger
if p99_latency > threshold:
    trigger_offcpu_capture()
```

## Quick Reference Commands

### xtop (0x.tools)

```bash
xtop -c 20                        # Real-time wall-clock top
xtop -p $(pgrep myapp)            # Specific process
xtop -t                           # Thread-level detail
xtop -c 20 -d 60                  # 60-second sample
```

### offcputime (BCC)

```bash
offcputime-bpfcc 30                           # System-wide, 30s
offcputime-bpfcc -p PID 30                    # Specific process
offcputime-bpfcc -df 30 > out.folded          # Folded for flame graph
offcputime-bpfcc -M 1000 30                   # Min 1ms blocking
flamegraph.pl --color=io < out.folded > offcpu.svg  # Generate graph
```

### bpftrace Off-CPU

```bash
# Off-CPU histogram
bpftrace -e 'tracepoint:sched:sched_switch { ... }'

# Off-CPU by process
bpftrace -e 'tracepoint:sched:sched_switch { @[comm] = sum(...) }'

# Filter by wait reason
bpftrace -e 'tracepoint:sched:sched_switch { if (prev_state & 2) ... }'
```

### Wall-Clock Profiling

```bash
# Java async-profiler
./asprof -e wall -d 60 -f wall.html PID

# Python/eBPF wall-clock profiler
sudo python3 wallclock_profiler.py PID -d 30 -f 99
```

## Key Formulas

```
Off-CPU Time % = (Wall-Clock Time - CPU Time) / Wall-Clock Time

If Off-CPU% > 50% → I/O-bound, optimize blocking operations
If Off-CPU% < 20% → CPU-bound, optimize compute

Load Scaling Bottleneck:
    Stage% at High Load > Stage% at Low Load
    → This stage limits throughput
```

## Troubleshooting Decision Matrix

| Symptom | Investigation | Tool | Fix |
|---------|--------------|------|-----|
| High latency, low CPU | Off-CPU analysis | `offcputime`, `xtop` | Optimize blocking operations |
| High iowait% | Disk I/O analysis | `iostat`, `biolatency` | Faster storage, caching |
| High cswch/s | Lock contention | `offcputime`, `perf lock` | Reduce critical sections |
| Network timeouts | Network latency | `ss -ti`, `mtr` | Connection pooling, reduce RTT |
| Event loop slow | Epoll analysis | `bpftrace` (epoll_wait) | Increase workers, tune timeouts |

## Sources and Further Reading

- [0x.tools - X-Ray vision for Linux systems](https://0x.tools/)
- [0x.tools GitHub Repository](https://github.com/tanelpoder/0xtools)
- [Brendan Gregg: Off-CPU Analysis](https://www.brendangregg.com/offcpuanalysis.html)
- [Brendan Gregg: Linux eBPF Off-CPU Flame Graph](https://www.brendangregg.com/blog/2016-01-20/ebpf-offcpu-flame-graph.html)
- [eBPF Tutorial: Wall Clock Profiling with Combined On-CPU and Off-CPU Analysis](https://eunomia.dev/tutorials/32-wallclock-profiler/)
- [Tanel Poder: xtop - Top for Wall-Clock Time](https://tanelpoder.com/posts/xtop-top-for-wall-clock-time/)
- [LWN: Kernel analysis with bpftrace](https://lwn.net/Articles/793749/)
- [bcc tools: offcputime examples](https://github.com/iovisor/bcc/blob/master/tools/offcputime_example.txt)

## See Also

- [05-performance-profiling](05-performance-profiling.md) - CPU profiling, flame graphs
- [06-ebpf-tracing](06-ebpf-tracing.md) - BCC tools, bpftrace fundamentals
- [13-latency-analysis](13-latency-analysis.md) - Tail latency, coordinated omission
- [16-scheduler-interrupts](16-scheduler-interrupts.md) - Scheduler analysis, run queue latency
