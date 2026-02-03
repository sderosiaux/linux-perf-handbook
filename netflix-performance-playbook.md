# Netflix Performance Engineering Playbook

Extracted from Netflix Tech Blog. Concrete tuning parameters, tools, methodologies, and benchmarks.

---

## 1. Linux Performance Analysis (60-Second Checklist)

Source: [Linux Performance Analysis in 60,000 Milliseconds](https://netflixtechblog.com/linux-performance-analysis-in-60-000-milliseconds-accc10403c55)

### The 10 Commands

```bash
# 1. Load averages - tasks wanting to run
uptime

# 2. Kernel errors and OOM events
dmesg | tail

# 3. Virtual memory stats (1s intervals)
vmstat 1

# 4. Per-CPU utilization (detect single-threaded bottlenecks)
mpstat -P ALL 1

# 5. Per-process CPU usage (rolling view)
pidstat 1

# 6. Block device I/O stats
iostat -xz 1

# 7. Memory usage (buffers/cache)
free -m

# 8. Network interface throughput
sar -n DEV 1

# 9. TCP connection stats
sar -n TCP,ETCP 1

# 10. Interactive process view
top
```

### USE Method (Utilization, Saturation, Errors)

| Resource | Utilization | Saturation | Errors |
|----------|-------------|------------|--------|
| CPU | `mpstat -P ALL` | run queue length | - |
| Memory | `free -m` | swap usage, OOM | `dmesg` |
| Disk | `iostat -xz` | await, avgqu-sz | device errors |
| Network | `sar -n DEV` | drops, overruns | `sar -n EDEV` |

---

## 2. JVM Garbage Collection

### Generational ZGC (JDK 21+)

Source: [Bending pause times to your will with Generational ZGC](https://netflixtechblog.com/bending-pause-times-to-your-will-with-generational-zgc-256629c9386b)

**Netflix default GC for JDK 21+.** >50% of critical streaming services migrated.

```bash
# Enable Generational ZGC
java -XX:+UseZGC -XX:+ZGenerational -Xmx<size> ...

# Non-generational ZGC (legacy)
java -XX:+UseZGC -Xmx<size> ...
```

#### Key Flags

| Flag | Purpose | Default |
|------|---------|---------|
| `-Xmx<size>` | Max heap (primary tuning knob) | - |
| `-XX:SoftMaxHeapSize=<size>` | Target heap ZGC tries to stay below | - |
| `-XX:ZUncommitDelay=<sec>` | Delay before returning memory to OS | 300s |
| `-Xms=<size>` | Set equal to `-Xmx` to prevent uncommit latency | - |

#### Benchmarks

| Metric | G1GC | ZGC (Gen) | Delta |
|--------|------|-----------|-------|
| Pause times | ms-seconds | microseconds | 1000x+ |
| CPU utilization (worst case) | baseline | -10% vs G1 | improved |
| Memory overhead | lower | +3% of heap | acceptable |
| Heap <32G compressed refs | yes | no (colored pointers) | N/A for ZGC |

**Key insight:** Losing compressed references on small heaps doesn't matter for ZGC because concurrent collection amortizes allocation rate increases.

### jvmquake: Proactive JVM Health

Source: [Introducing jvmquake](https://netflixtechblog.medium.com/introducing-jvmquake-ec944c60ba70)

Kills JVMs stuck in GC death spirals before they become grey failures.

```bash
# Attach jvmquake agent
java -agentpath:/path/to/libjvmquake.so=<threshold>,<runtime_weight>,<action> ...

# Default: 30 second debt threshold
java -agentpath:/path/to/libjvmquake.so=30,1,0 ...
```

#### Debt Model

```
debt_counter = Σ(gc_pause_time) - Σ(runtime × weight)
# floor at 0, cannot go negative
# if debt > threshold after GC: kill JVM
```

**Default threshold:** 30 seconds
**Action modes:** 0=SIGKILL, 1=trigger OOM for heap dump, 2=SIGABRT for core dump

#### Why Not Standard JVM Options?

| Option | Problem |
|--------|---------|
| `GCTimeLimit` | Requires 5 consecutive full GCs to trigger |
| `GCHeapFreeLimit` | Hard to tune, inconsistent across GCs |
| `+UseGCOverheadLimit` | Doesn't work reliably on all collectors |

**Production deployment:** All Netflix Cassandra and Elasticsearch JVMs.

---

## 3. CPU Flame Graphs

### Mixed-Mode Java Flame Graphs

Source: [Java in Flames](https://netflixtechblog.com/java-in-flames-e763b3d32166)

```bash
# 1. Enable frame pointers (required for perf)
java -XX:+PreserveFramePointer ...

# 2. Record CPU samples
sudo perf record -F 99 -a -g -- sleep 30

# 3. Generate JIT symbol map (run immediately after perf record)
# Using perf-map-agent
java -jar perf-map-agent/out/attach-main.jar <pid>

# 4. Generate flame graph
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

#### JVM Profiling Flags

| Flag | Purpose |
|------|---------|
| `-XX:+PreserveFramePointer` | Enable stack walking (JDK8u60+) |
| `-XX:InlineSmallCode=500` | Reduce inlining for more frames |
| `-XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints` | Better symbol resolution |

### FlameScope: Subsecond Analysis

Source: [Netflix FlameScope](https://netflixtechblog.com/netflix-flamescope-a57ca19d47bb)

Visualizes variance, perturbations, single-threaded execution patterns.

```bash
# Clone and run FlameScope
git clone https://github.com/Netflix/flamescope
cd flamescope
pip install -r requirements.txt
python run.py
# Access at http://localhost:5000
```

**Workflow:**
1. Generate heat map from profile data
2. Select time ranges visually
3. Generate flame graph for specific subsecond ranges
4. Compare before/after or normal/anomaly periods

**Supported formats:** Linux perf, Trace Event Format (Chrome), DTrace

### Impact Example

Source: [Saving 13 Million Computational Minutes](https://netflixtechblog.com/saving-13-million-computational-minutes-per-day-with-flame-graphs-d95633b6d01f)

- Identified method consuming 25% of CPU samples
- After fix: reduced to <1%
- **Savings: 13M CPU-minutes/day (~25 CPU-years/day)**

---

## 4. Container CPU Isolation

Source: [Predictive CPU Isolation of Containers at Netflix](https://netflixtechblog.com/predictive-cpu-isolation-of-containers-91f014d856c7)

### titus-isolate Architecture

```
┌─────────────────────────────────────────────┐
│                 titus-isolate               │
├─────────────────────────────────────────────┤
│  GBRT Model (P95 CPU prediction, 10min)     │
│  ↓                                          │
│  MIP Solver (cvxpy → CBC/Gurobi)            │
│  ↓                                          │
│  cpuset cgroup manipulation                 │
└─────────────────────────────────────────────┘
```

### Mechanism

```bash
# titus-isolate modifies cpuset cgroups per container
echo "0-3" > /sys/fs/cgroup/cpuset/<container>/cpuset.cpus

# Events that trigger reoptimization:
# - add: new container allocated
# - remove: container terminated
# - rebalance: CPU usage changed
```

### ML Model Features

- **Contextual:** owner, image, memory config, app name
- **Time-series:** last hour of CPU usage
- **Target:** P95 CPU usage in next 10 minutes (quantile regression)
- **Retraining:** every few hours on weeks of platform data

### Results

| Workload | Improvement |
|----------|-------------|
| Batch jobs | Multi-% runtime reduction |
| Runtime variance | Right-tail outliers eliminated |
| Middleware service | 13% capacity reduction (1000+ containers) |

---

## 5. Distributed Tracing (Edgar)

Source: [Building Netflix's Distributed Tracing Infrastructure](https://netflixtechblog.com/building-netflixs-distributed-tracing-infrastructure-bb856c319304)

### Architecture

```
┌──────────────┐    ┌─────────────┐    ┌───────────────┐
│ Tracer Libs  │───▶│   Mantis    │───▶│   Cassandra   │
│ (OpenZipkin) │    │ (Streaming) │    │   (Storage)   │
└──────────────┘    └─────────────┘    └───────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │  Edgar UI   │
                    └─────────────┘
```

### Sampling Strategy

| Type | Rate | Use Case |
|------|------|----------|
| Random (head-based) | 0.1% at Zuul | Default |
| 100% sampling | configurable | Critical streaming services |
| Tail-based filtering | N/A | Retain errors/warnings/retries |

**Tail-based sampling result:** 20% volume reduction, no UX impact.

### Storage Optimization

| Change | Impact |
|--------|--------|
| Elasticsearch → Cassandra | Handle high write rates |
| Zstd block compression | 50% storage reduction |
| Tiered storage (Cassandra → S3) | Cost optimization |

**Cost reduction:** 71% less to operate, 35x more data stored.

---

## 6. Network Observability (eBPF)

Source: [How Netflix Uses eBPF Flow Logs at Scale](https://netflixtechblog.com/how-netflix-uses-ebpf-flow-logs-at-scale-for-network-insight-e3ea997dca96)

### FlowExporter

```
┌─────────────────────────────────────────────────────────┐
│                    FlowExporter Sidecar                 │
├─────────────────────────────────────────────────────────┤
│  eBPF Programs (TCP tracepoints)                        │
│  ├─ tcp_connect                                         │
│  ├─ tcp_close                                           │
│  └─ socket state changes                                │
│                                                         │
│  Output: IP, ports, timestamps, socket stats            │
└─────────────────────────────────────────────────────────┘
```

### Scale

- **5 million flow records/second**
- **Billions of records/hour ingested**
- **<1% CPU and memory overhead**

### IP Attribution

```
IPManAgent → eBPF map → FlowExporter
(IP → workload-ID mapping written on container launch)
```

---

## 7. Metrics & Monitoring

### Atlas (Time Series Database)

Source: [Introducing Atlas](https://netflixtechblog.com/introducing-atlas-netflixs-primary-telemetry-platform-bd31f4d8ed9a)

| Metric | Value |
|--------|-------|
| 2011 metrics | 2 million |
| 2014 metrics | 1.2 billion |
| Query throughput | Billions of datapoints/second |
| Hot data window | 6 hours (in-memory) |

**Architecture:** In-memory dimensional TSDB, sharded across machines.

**Cost controls:**
- Single replica for older data (50% savings)
- Automatic rollup after 6 hours
- Drop node-level dimensions on old data

### Telltale (Application Health)

Source: [Telltale: Netflix Application Monitoring Simplified](https://netflixtechblog.com/telltale-netflix-application-monitoring-simplified-5c08bfa780ba)

- Learns typical health automatically (no alert tuning)
- Correlates anomalies across upstream/downstream services
- Ingests SLI metrics (latency, error rates)
- Suggests probable causes during incidents

**Scale:** Thousands of services monitored via Atlas Streaming.

---

## 8. Load Shedding & Concurrency

Source: [Performance Under Load](https://netflixtechblog.medium.com/performance-under-load-3e6fa9a60581), [Prioritized Load Shedding](https://netflixtechblog.com/keeping-netflix-reliable-using-prioritized-load-shedding-6cc827b02f94)

### Adaptive Concurrency Limits

```xml
<!-- Maven dependency -->
<dependency>
  <groupId>com.netflix.concurrency-limits</groupId>
  <artifactId>concurrency-limits-core</artifactId>
</dependency>
```

```java
// Server-side: Vegas limit (delay-based)
Limiter<Context> limiter = SimpleLimiter.newBuilder()
    .limit(VegasLimit.newBuilder().build())
    .build();

// Client-side: AIMD limit (loss-based)
Limiter<Context> limiter = SimpleLimiter.newBuilder()
    .limit(AIMDLimit.newBuilder().build())
    .build();
```

### Priority-Based Load Shedding

| Priority Score | Traffic Type | Behavior Under Load |
|----------------|--------------|---------------------|
| 0 (highest) | User-initiated playback | Always served |
| 50 | Interactive requests | Shed at high load |
| 100 (lowest) | Telemetry, prefetch | Shed first |

**Implementation:** Dynamic priority threshold at Zuul API gateway.

### Partitioned Concurrency

```java
// Two partitions: user-initiated gets 100% guarantee
// prefetch uses only excess capacity
PartitionedLimiter.newBuilder()
    .partition("user-initiated", 1.0)  // guaranteed 100%
    .partition("prefetch", 0.0)        // excess only
    .build();
```

**Thresholds example:**
- Non-critical shedding starts: 60% CPU
- Critical shedding starts: 80% CPU

---

## 9. Kernel Debugging Case Study

Source: [Investigation of a Cross-regional Network Performance Issue](https://netflixtechblog.com/investigation-of-a-cross-regional-network-performance-issue-422d6218fdf1)

### Problem

- Performance regression after kernel upgrade 6.5.13 → 6.6.10
- ~14,000 commits between versions

### Methodology

1. Binary search (git bisect) across kernel versions
2. Identified commit with "tcp" in message
3. Root cause: TCP receive window scaling changed

### Fix

```bash
# Increase initial scaling_ratio from 25% to 50%
# Makes receive window backward compatible with original sysctl_tcp_adv_win_scale

# Kafka configuration
receive.buffer.bytes=-1  # Let Linux auto-tune receive window
```

---

## 10. Netflix Performance Tools Summary

| Tool | Purpose | Status |
|------|---------|--------|
| [Atlas](https://github.com/Netflix/atlas) | Dimensional TSDB | Active |
| [Vector](https://github.com/Netflix/vector) | On-host perf monitoring | Deprecated → Grafana |
| [FlameScope](https://github.com/Netflix/flamescope) | Subsecond flame graph analysis | Active |
| [jvmquake](https://github.com/Netflix-Skunkworks/jvmquake) | JVM health agent | Active |
| [concurrency-limits](https://github.com/Netflix/concurrency-limits) | Adaptive load shedding | Active |
| [titus-isolate](https://github.com/Netflix-Skunkworks/titus-isolate) | Container CPU isolation | Active |
| [Spectator](https://github.com/Netflix/spectator) | Metrics instrumentation | Active |

---

## Key Takeaways

1. **Measure first:** 60-second checklist before deep diving
2. **USE Method:** Systematically check utilization, saturation, errors
3. **ZGC on JDK 21+:** Sub-millisecond pauses, minimal tuning needed
4. **Flame graphs:** Visualize CPU time, find optimization targets
5. **jvmquake:** Fail fast on GC spirals (30s default threshold)
6. **Adaptive limits:** TCP congestion control concepts for services
7. **Priority shedding:** Protect critical paths under load
8. **eBPF:** Low-overhead network observability at scale
9. **Tail-based sampling:** Keep interesting traces, drop noise

---

## Sources

- [Linux Performance Analysis in 60,000 Milliseconds](https://netflixtechblog.com/linux-performance-analysis-in-60-000-milliseconds-accc10403c55)
- [Bending pause times to your will with Generational ZGC](https://netflixtechblog.com/bending-pause-times-to-your-will-with-generational-zgc-256629c9386b)
- [Introducing jvmquake](https://netflixtechblog.medium.com/introducing-jvmquake-ec944c60ba70)
- [Java in Flames](https://netflixtechblog.com/java-in-flames-e763b3d32166)
- [Netflix FlameScope](https://netflixtechblog.com/netflix-flamescope-a57ca19d47bb)
- [Saving 13 Million Computational Minutes](https://netflixtechblog.com/saving-13-million-computational-minutes-per-day-with-flame-graphs-d95633b6d01f)
- [Predictive CPU Isolation of Containers](https://netflixtechblog.com/predictive-cpu-isolation-of-containers-at-netflix-91f014d856c7)
- [Building Netflix's Distributed Tracing Infrastructure](https://netflixtechblog.com/building-netflixs-distributed-tracing-infrastructure-bb856c319304)
- [How Netflix Uses eBPF Flow Logs at Scale](https://netflixtechblog.com/how-netflix-uses-ebpf-flow-logs-at-scale-for-network-insight-e3ea997dca96)
- [Introducing Atlas](https://netflixtechblog.com/introducing-atlas-netflixs-primary-telemetry-platform-bd31f4d8ed9a)
- [Telltale: Netflix Application Monitoring Simplified](https://netflixtechblog.com/telltale-netflix-application-monitoring-simplified-5c08bfa780ba)
- [Performance Under Load](https://netflixtechblog.medium.com/performance-under-load-3e6fa9a60581)
- [Keeping Netflix Reliable Using Prioritized Load Shedding](https://netflixtechblog.com/keeping-netflix-reliable-using-prioritized-load-shedding-6cc827b02f94)
- [Investigation of a Cross-regional Network Performance Issue](https://netflixtechblog.com/investigation-of-a-cross-regional-network-performance-issue-422d6218fdf1)
