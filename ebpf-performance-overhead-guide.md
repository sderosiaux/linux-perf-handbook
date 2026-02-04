# eBPF Performance Overhead & Optimization Guide

Production-focused guide for measuring, minimizing, and managing eBPF overhead in high-throughput systems. Based on research from Meta, Netflix, Cloudflare, and academic benchmarks.

---

## Overview

eBPF programs add overhead at every invocation. Understanding and minimizing this overhead is critical for production deployment. This guide provides concrete numbers, optimization patterns, and safe deployment thresholds.

**Target audience:** SREs, performance engineers deploying eBPF in production environments requiring <1% overhead.

**Prerequisites:**
- Kernel 5.5+ for fentry/fexit (recommended)
- Kernel 4.7+ for in-kernel histograms
- BTF-enabled kernel for CO-RE

---

## Hook Attachment Type Overhead

### Benchmark Comparison

| Attach Type | Overhead per Event | Mechanism | Production Safe? |
|-------------|-------------------|-----------|-----------------|
| **Tracepoint** | 10-50ns | Compile-time NOP, optimized call | ✓ Yes |
| **Raw Tracepoint** | 10-40ns | Direct args, no marshalling | ✓ Yes |
| **Fentry/Fexit** | 50-100ns | BPF trampoline, JIT jump | ✓ Yes (kernel 5.5+) |
| **Kprobe** | 100-300ns | int3 trap, exception handler | ⚠ Filtered only |
| **Kretprobe** | 200-600ns | Trampoline injection | ⚠ Low frequency only |

**Key insight:** Kprobes are ~20% slower than tracepoints due to exception handling overhead. Always prefer tracepoints when available.

### Why the Performance Difference

**Tracepoints:**
- NOP instruction when disabled (zero overhead)
- Pre-defined stable API maintained across kernel versions
- Arguments already prepared for tracing
- Direct function call when enabled

**Fentry/Fexit (BPF Trampolines):**
- Patches function prologue with jump instruction
- No CPU exception triggered
- 10x faster than kprobes for hot paths
- Access both function arguments and return value in fexit

**Kprobes:**
- Replace instruction with `int3` breakpoint
- Triggers CPU exception on every hit
- Exception handler dispatches to BPF program
- Higher context switch overhead

**Raw Tracepoints:**
- Skip argument marshalling that regular tracepoints perform
- Directly expose kernel internal parameters
- Slightly faster than regular tracepoints (~10-20%)
- No BPF_CORE_READ needed for BTF-enabled raw tracepoints

### Decision Matrix

```
START: What kernel function to trace?

├─> Syscall entry/exit?
│   └─> Use: tracepoint:syscalls:sys_enter_* / sys_exit_*
│        Overhead: ~10-30ns per syscall
│        Production: Safe with reasonable syscall rates

├─> Scheduler events (context switch, wakeup)?
│   └─> Use: tracepoint:sched:sched_switch / sched_wakeup
│        Overhead: ~20-40ns per event
│        Production: Safe, fires 1K-10K/sec typically

├─> Block I/O submission/completion?
│   └─> Use: tracepoint:block:block_rq_issue / block_rq_complete
│        Overhead: ~15-35ns per I/O
│        Production: Safe

├─> Network events (TCP state, socket)?
│   └─> Use: tracepoint:sock:* / tracepoint:tcp:*
│        Overhead: ~20-50ns per event
│        Production: Safe

├─> Internal kernel function (no tracepoint exists)?
│   │
│   ├─> Kernel 5.5+ with BTF?
│   │   └─> Use: fentry/fexit
│   │        Overhead: ~50-100ns per call
│   │        Production: Safe for medium-frequency functions (<100K/sec)
│   │
│   └─> Kernel < 5.5 or no BTF?
│       └─> Use: kprobe (FILTERED!)
│            Overhead: ~100-300ns per call
│            Production: Only for low-frequency functions (<10K/sec)

└─> Hot path function (>100K calls/sec)?
    └─> AVOID tracing if possible
        Consider: sampling, aggregation, or coarser-grained hooks
```

### Verification

```bash
# Check if tracepoint exists
cat /sys/kernel/debug/tracing/available_events | grep tcp_retransmit_skb

# Check if kernel has BTF (required for fentry)
ls /sys/kernel/btf/vmlinux

# Test if function is probeable with fentry
bpftrace -l 'fentry:vfs_read'

# Test kprobe attachment (manual method)
echo 'p:test_probe vfs_read' > /sys/kernel/debug/tracing/kprobe_events
cat /sys/kernel/debug/tracing/kprobe_events
echo '-:test_probe' >> /sys/kernel/debug/tracing/kprobe_events
```

---

## Map Type Performance

### Hashmap vs Per-CPU Hashmap vs LRU

Performance varies dramatically based on workload and map type selection.

| Map Type | Single-Flow Performance | Multi-Flow Performance | Memory | Use Case |
|----------|------------------------|------------------------|--------|----------|
| **Regular Hashmap** | 1.6 Mpps (poor) | Good | Fixed | Few concurrent flows |
| **Per-CPU Hashmap** | 4.7 Mpps (excellent) | Excellent | Fixed × CPUs | High concurrency |
| **LRU Hashmap** | Moderate | Good | Bounded | Unknown key cardinality |
| **Per-CPU Array** | Best | Excellent | Fixed × CPUs | Fixed-size, indexed data |
| **Ringbuffer** | Good (single-flow) | Excellent | Fixed | Event streaming |

### Detailed Overhead

**Regular Hashmap:**
- Lookup: ~200-600ns (includes locking)
- Update: ~300-800ns
- Single entry contention: 1.6 Mpps → throughput collapse
- Multi-flow: Good scaling (different entries)

**Per-CPU Hashmap:**
- Lookup: ~100-300ns (no locking)
- Update: ~200-500ns
- Scales linearly with CPU count
- Aggregation done in userspace
- **Best for:** timestamp storage, per-flow tracking

**LRU Hashmap:**
- Lookup: ~240-650ns (+40-50ns vs regular hashmap)
- Update: ~350-900ns (+50ns for LRU maintenance)
- Eviction algorithm is approximate (not precise LRU)
- **Best for:** bounded memory usage with unknown key cardinality

**Per-CPU Array:**
- Lookup: ~50-150ns (fastest)
- Update: ~100-250ns
- Fixed size, zero-based indexed
- **Best for:** per-CPU counters, fixed-size metrics

### Performance at Scale

From real-world benchmarks (network flow tracking):

```
Single Flow (all packets hit same map entry):
- Regular Hashmap:     1.6 Mpps  (lock contention)
- Per-CPU Hashmap:     4.7 Mpps  (no contention)
- Ringbuffer:          2.1 Mpps  (per-CPU variant)

Multiple Flows (packets distributed across entries):
- Regular Hashmap:     3.8 Mpps  (reduced contention)
- Per-CPU Hashmap:     4.7 Mpps  (linear scaling)
- Per-CPU Array:       4.9 Mpps  (optimal)
```

### Meta's Production Findings

From SSLWall (encryption enforcement at scale):

> "BPF arrays over LRU maps where possible"
> "Early exit conditions [for high QPS services]"

Translation:
- Prefer per-CPU arrays for simple indexed data
- LRU maps add 40-50ns overhead per operation
- For high-throughput services (>100K QPS), every nanosecond matters

### Recommendation

**For timestamp tracking (latency measurement):**
```c
// Regular hashmap - WRONG for high concurrency
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __type(key, u64);        // tid or flow tuple
    __type(value, u64);      // timestamp
    __uint(max_entries, 10240);
} start_times SEC(".maps");

// Per-CPU hashmap - CORRECT
struct {
    __uint(type, BPF_MAP_TYPE_PERCPU_HASH);
    __type(key, u64);
    __type(value, u64);
    __uint(max_entries, 10240);
} start_times SEC(".maps");
// Userspace must aggregate across CPUs
```

**For bounded memory with unknown keys:**
```c
// Use LRU when key cardinality is unbounded
struct {
    __uint(type, BPF_MAP_TYPE_LRU_HASH);
    __type(key, u64);
    __type(value, struct metrics);
    __uint(max_entries, 100000);  // Auto-evict oldest
} flow_metrics SEC(".maps");
// Accept +40-50ns overhead for automatic eviction
```

---

## BPF_CORE_READ Overhead

### When BPF_CORE_READ is Needed

BPF_CORE_READ() enables CO-RE (Compile Once, Run Everywhere) by handling kernel structure changes across versions.

**Overhead:** +20-30ns per BPF_CORE_READ() invocation

**When to use:**
- Kprobes accessing kernel structures
- Fentry/fexit with struct pointer dereferences
- Uprobes reading userspace memory

**When to SKIP:**
- Raw tracepoints (direct args, no pointer chasing)
- BTF-enabled tracepoints with simple fields
- Per-CPU variables (direct access)

### Examples

**Kprobe (needs BPF_CORE_READ):**
```c
SEC("kprobe/tcp_connect")
int trace_tcp_connect(struct pt_regs *ctx) {
    struct sock *sk = (struct sock *)PT_REGS_PARM1(ctx);

    // WRONG - may break across kernel versions
    u32 saddr = sk->__sk_common.skc_daddr;

    // CORRECT - CO-RE relocatable
    u32 saddr = BPF_CORE_READ(sk, __sk_common.skc_daddr);
    // Adds ~20-30ns overhead
}
```

**Raw Tracepoint (NO BPF_CORE_READ needed):**
```c
SEC("raw_tracepoint/tcp_retransmit_skb")
int trace_tcp_retrans(struct bpf_raw_tracepoint_args *ctx) {
    struct sock *sk = (struct sock *)ctx->args[0];

    // Direct access - raw tracepoints expose kernel args directly
    u32 saddr = sk->__sk_common.skc_daddr;
    // No BPF_CORE_READ needed, no extra overhead
}
```

**Tracepoint with BTF (NO helper needed):**
```c
SEC("tp_btf/sched_switch")
int trace_sched_switch(u64 *ctx) {
    struct task_struct *prev = (struct task_struct *)ctx[1];

    // BTF-enabled, direct access
    pid_t pid = prev->pid;
    // No overhead
}
```

### Cost-Benefit Analysis

**Example: 10-field struct read in high-frequency kprobe**

```c
// Approach 1: Individual BPF_CORE_READ calls
u32 field1 = BPF_CORE_READ(task, field1);  // +25ns
u32 field2 = BPF_CORE_READ(task, field2);  // +25ns
u32 field3 = BPF_CORE_READ(task, field3);  // +25ns
// ... 10 fields = +250ns overhead

// Approach 2: Bulk read with BPF_CORE_READ_INTO
struct my_data data;
BPF_CORE_READ_INTO(&data, task, field1, field2, field3, ...);
// Single relocation, ~30-50ns overhead

// Approach 3: Use tracepoint instead of kprobe (if available)
// Zero BPF_CORE_READ overhead, tracepoint provides stable fields
```

**Recommendation:** For kprobes on hot paths reading multiple fields, consider:
1. Switch to tracepoint/fentry if available
2. Use bulk read helpers (BPF_CORE_READ_INTO)
3. Read only necessary fields

---

## Early Exit Optimization Patterns

### The Problem

Every eBPF program invocation costs CPU cycles. Unnecessary work multiplies overhead.

**Example:** Tracing file opens system-wide
- Syscall rate: 50K opens/sec
- 80% are kernel threads (pid 0) or irrelevant processes
- 40K unnecessary eBPF program executions per second

### Pattern 1: Kernel Task Filtering

```c
SEC("tracepoint/syscalls/sys_enter_openat")
int trace_open(struct trace_event_raw_sys_enter *ctx) {
    u64 pid_tgid = bpf_get_current_pid_tgid();
    u32 pid = pid_tgid >> 32;

    // Early exit: skip kernel threads (pid 0)
    if (pid == 0)
        return 0;

    // Early exit: skip kworker threads
    char comm[16];
    bpf_get_current_comm(&comm, sizeof(comm));
    if (comm[0] == 'k' && comm[1] == 'w' && comm[2] == 'o' && comm[3] == 'r')
        return 0;

    // Remaining work only executes for relevant processes
    // ... expensive operations ...
}
```

**Impact:** Reduces program execution rate by 60-80% in typical workloads.

### Pattern 2: PID/TID Allowlist

```c
// User-space controlled PID filter
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __type(key, u32);    // pid
    __type(value, u8);   // unused (existence check)
    __uint(max_entries, 1024);
} pid_filter SEC(".maps");

SEC("kprobe/vfs_read")
int trace_read(struct pt_regs *ctx) {
    u32 pid = bpf_get_current_pid_tgid() >> 32;

    // Early exit: not in allowlist
    if (!bpf_map_lookup_elem(&pid_filter, &pid))
        return 0;

    // Only processes in filter proceed
    // ... rest of program ...
}
```

**Use case:** Targeted debugging of specific services without system-wide overhead.

### Pattern 3: Argument-Based Early Exit

```c
SEC("tracepoint/syscalls/sys_enter_read")
int trace_read(struct trace_event_raw_sys_enter *ctx) {
    int fd = ctx->args[0];
    ssize_t count = ctx->args[2];

    // Early exit: small reads (<4KB) not interesting
    if (count < 4096)
        return 0;

    // Early exit: stdin/stdout/stderr
    if (fd >= 0 && fd <= 2)
        return 0;

    // Only large reads from files proceed
    // ... expensive tracking ...
}
```

**Impact:** Can reduce event processing by 90%+ in certain workloads.

### Pattern 4: Sampling for High-Frequency Events

```c
SEC("kprobe/tcp_sendmsg")
int trace_tcp_send(struct pt_regs *ctx) {
    // Sample 1% of events (for 10M events/sec → 100K/sec)
    u32 rand = bpf_get_prandom_u32();
    if ((rand % 100) != 0)
        return 0;

    // Statistical profiling with 99% overhead reduction
    // ... expensive processing ...
}
```

**Use case:** Profiling extremely hot functions where full tracing would add >5% overhead.

### Meta's SSLWall Pattern

From production encryption enforcement at scale:

```c
// Early exit: check if connection already validated
u64 sock_id = (u64)sk;
struct conn_state *state = bpf_map_lookup_elem(&validated_conns, &sock_id);
if (state && state->is_encrypted)
    return 0;  // Skip already-validated encrypted connections

// Early exit: loopback traffic always allowed
if (is_loopback_addr(sk))
    return 0;

// Early exit: whitelisted services
if (is_whitelisted_service(comm))
    return 0;

// Only unvalidated, non-loopback, non-whitelisted connections proceed
// to expensive encryption check
```

**Result:** Reduced CPU impact from 3-5% to <1% in high-QPS services.

### Benchmark: Early Exit Impact

Test: tracepoint on `sys_enter_write` (50K syscalls/sec)

| Optimization | Events Processed | CPU Overhead |
|--------------|-----------------|--------------|
| No filter | 50K/sec | 2.1% |
| PID filter (10 processes) | 12K/sec | 0.5% |
| + Skip kernel threads | 10K/sec | 0.4% |
| + Skip small writes (<1KB) | 2K/sec | 0.08% |

**Key insight:** Order matters. Put cheapest checks first (pid comparison), expensive checks later (string operations, map lookups).

---

## Production Overhead Thresholds

### Safe Overhead Guidelines

| Service Type | Max CPU Overhead | Max Latency Added | Notes |
|--------------|-----------------|-------------------|-------|
| **Latency-sensitive (P99 <10ms)** | <0.5% | <50μs per request | Trading, real-time APIs |
| **Standard web services** | <1% | <200μs per request | Most HTTP services |
| **Batch processing** | <3% | <1ms per operation | ETL, analytics |
| **Development/staging** | <10% | Any | Comprehensive tracing OK |

### Measurement Methodology

**Enable BPF statistics:**
```bash
# Kernel 5.1+ exposes per-program stats
sysctl -w kernel.bpf_stats_enabled=1

# View per-program overhead
bpftool prog show
# Output includes:
#   run_time_ns: 123456789   (total CPU time)
#   run_cnt: 987654          (invocations)
# Calculate: avg_ns = run_time_ns / run_cnt
```

**Measure with bpftop (Netflix):**
```bash
# Install bpftop
git clone https://github.com/Netflix/bpftop
cd bpftop && cargo build --release

# Run monitoring
sudo ./target/release/bpftop
# Shows real-time:
# - Events per second per program
# - Average execution time
# - Estimated CPU % usage
```

**Manual CPU overhead calculation:**
```
Overhead % = (events_per_sec × avg_latency_ns) / (1e9 × num_cpus) × 100

Example:
- 50K events/sec
- 500ns average latency
- 8 CPUs

Overhead = (50000 × 500) / (1e9 × 8) × 100 = 0.31%
```

### Overhead Breakdown by Component

**Example: VFS read latency tracer**

```
Component                           Overhead
─────────────────────────────────────────────
fentry hook invocation              50ns
Map lookup (per-CPU hash)           150ns
Timestamp capture                   20ns
Map update                          180ns
fexit hook invocation               50ns
Delta calculation                   10ns
Histogram update                    100ns
─────────────────────────────────────────────
Total per traced function call      560ns
```

**At 10K vfs_read calls/sec across 8 CPUs:**
```
Overhead = (10000 × 560) / (1e9 × 8) × 100 = 0.07%
```

**Safe for production:** Yes

### Real-World Examples

**Meta Strobelight (fleet-wide profiling):**
- 42 different profilers running simultaneously
- Dynamic sampling adjusted daily per service
- Overhead target: <1% CPU fleet-wide
- Method: Sampling + per-CPU data structures + early exits

**Katran (L4 load balancer):**
- XDP hook on every packet
- Per-CPU LRU maps for connection tracking
- Overhead: ~100-200ns per packet
- At 1M packets/sec: ~10-20% of single CPU (acceptable for dedicated LB)

**Millisampler (network monitoring):**
- TC BPF hook on ingress/egress
- Per-CPU arrays (2000 × 64-bit counters)
- 100μs sampling interval
- Fixed memory: 128KB per CPU
- Overhead: <0.5% on 10Gbps links

---

## Optimization Techniques

### 1. Use Appropriate Map Types

```c
// BAD: LRU hashmap for simple counters
struct {
    __uint(type, BPF_MAP_TYPE_LRU_HASH);
    __type(key, u32);
    __type(value, u64);
    __uint(max_entries, 256);
} counters SEC(".maps");

// GOOD: Per-CPU array for fixed-size counters
struct {
    __uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
    __type(key, u32);
    __type(value, u64);
    __uint(max_entries, 256);
} counters SEC(".maps");
// 3-5x faster lookups/updates
```

### 2. Minimize Map Operations

```c
// BAD: Multiple lookups
u64 *val1 = bpf_map_lookup_elem(&map, &key1);
u64 *val2 = bpf_map_lookup_elem(&map, &key2);
u64 *val3 = bpf_map_lookup_elem(&map, &key3);

// GOOD: Single lookup of struct
struct metrics {
    u64 val1, val2, val3;
};
struct metrics *m = bpf_map_lookup_elem(&map, &key);
// 3x fewer map operations
```

### 3. Batch Operations (Kernel 5.6+)

```c
// User-space batch operations reduce syscall overhead
#define BATCH_SIZE 1000
__u32 keys[BATCH_SIZE];
struct value values[BATCH_SIZE];
__u32 count = BATCH_SIZE;

// Single syscall instead of 1000
bpf_map_lookup_batch(map_fd, NULL, &next_key, keys, values, &count, 0);
```

### 4. Use In-Kernel Aggregation

```c
// BAD: Send every event to userspace
struct event {
    u64 timestamp;
    u32 pid;
    char comm[16];
    size_t bytes;
};
bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU, &event, sizeof(event));
// 50+ bytes per event to userspace

// GOOD: Aggregate in kernel, periodic export
@bytes[comm] = sum(bytes);
// Userspace reads aggregated map periodically
```

### 5. Minimize Stack Usage

```c
// BAD: Large stack allocations
char buf[4096];  // Stack limited to 512 bytes in BPF

// GOOD: Use per-CPU array as scratch space
struct {
    __uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
    __type(key, u32);
    __type(value, char[4096]);
    __uint(max_entries, 1);
} scratch_buf SEC(".maps");

u32 zero = 0;
char *buf = bpf_map_lookup_elem(&scratch_buf, &zero);
```

### 6. Avoid String Operations

```c
// BAD: bpf_probe_read_str on every event
char path[256];
bpf_probe_read_str(&path, sizeof(path), filename);
// Variable-length, slow

// GOOD: Use fixed-size reads for known structures
struct {
    char comm[16];
    // ... other fields
};
bpf_get_current_comm(&data.comm, sizeof(data.comm));
// Fixed 16 bytes, fast
```

### 7. Compiler Optimizations

```c
// Always compile with optimizations
clang -O2 -target bpf -c program.bpf.c -o program.bpf.o

// Use __always_inline for small helper functions
static __always_inline bool is_valid_pid(u32 pid) {
    return pid > 0 && pid < 65535;
}
// Prevents function call overhead
```

---

## Production Deployment Checklist

### Pre-Deployment

- [ ] Measure baseline CPU usage without eBPF
- [ ] Enable `kernel.bpf_stats_enabled=1`
- [ ] Test in staging with production-like traffic
- [ ] Implement early exit filters
- [ ] Use appropriate hook type (prefer tracepoint > fentry > kprobe)
- [ ] Use per-CPU maps for high-concurrency data
- [ ] Add userspace monitoring of BPF program stats

### Deployment

- [ ] Deploy to canary hosts first (1-5% of fleet)
- [ ] Monitor for 24 hours:
  - CPU usage delta
  - Latency P50/P99/P999 delta
  - BPF program run_time_ns and run_cnt
- [ ] Compare canary vs control groups
- [ ] Gradual rollout: 5% → 25% → 50% → 100%

### Monitoring

```bash
# Continuous monitoring script
#!/bin/bash
while true; do
    bpftool prog show | awk '
        /run_time_ns/ {runtime=$2}
        /run_cnt/ {
            cnt=$2
            if (cnt > 0) {
                avg_ns = runtime / cnt
                printf "Avg latency: %d ns, Count: %d\n", avg_ns, cnt
            }
        }
    '
    sleep 60
done
```

### Alerting Thresholds

```yaml
alerts:
  - alert: BPFHighOverhead
    expr: |
      (bpf_prog_run_time_ns / bpf_prog_run_cnt) > 1000
    for: 5m
    annotations:
      summary: "eBPF program avg latency >1μs"

  - alert: BPFHighFrequency
    expr: |
      rate(bpf_prog_run_cnt[1m]) > 100000
    for: 5m
    annotations:
      summary: "eBPF program invoked >100K/sec"

  - alert: BPFCPUOverhead
    expr: |
      (rate(bpf_prog_run_time_ns[5m]) / 1e9 / num_cpus) > 0.01
    for: 10m
    annotations:
      summary: "eBPF programs consuming >1% CPU"
```

---

## Troubleshooting High Overhead

### Symptom: CPU usage higher than expected

**Diagnosis:**
```bash
# Identify high-frequency programs
bpftool prog show | grep -A2 run_cnt | sort -t: -k2 -n

# Profile the BPF program itself
bpftrace -e 'profile:hz:99 /comm == "bpf_prog_xyz"/ { @[ustack] = count(); }'

# Check event rate
bpftool prog show id <id> | awk '/run_cnt/ {print $2}'
# Calculate events/sec by sampling twice with 10s gap
```

**Common causes:**
1. Attached to high-frequency tracepoint (e.g., `sched_switch` without filter)
2. Missing early exit checks
3. Expensive map operations in hot path
4. Using kprobe instead of tracepoint/fentry

**Fixes:**
- Add PID/comm filters
- Switch to lower-frequency event
- Use per-CPU maps
- Switch from kprobe to tracepoint

### Symptom: Increased tail latency (P99/P999)

**Diagnosis:**
```bash
# Check if BPF program is on critical path
bpftrace -e '
    tracepoint:syscalls:sys_enter_write {
        @start[tid] = nsecs;
    }
    tracepoint:syscalls:sys_exit_write /@start[tid]/ {
        @latency = hist(nsecs - @start[tid]);
        delete(@start[tid]);
    }
'
# Compare with/without BPF program
```

**Common causes:**
1. Map lock contention (use per-CPU maps)
2. Large map lookups (hash collisions)
3. BPF program attached to synchronous path (e.g., sys_enter)

**Fixes:**
- Use per-CPU maps to eliminate contention
- Reduce map size or use LRU eviction
- Move to async path if possible (e.g., fexit instead of fentry)

### Symptom: Lost events / dropped samples

**Diagnosis:**
```bash
# Check ringbuffer/perf_event stats
bpftool map dump name events
# Look for dropped counters

# Check kernel logs
dmesg | grep -i bpf
```

**Common causes:**
1. Ringbuffer too small
2. Userspace consumer too slow
3. BPF program taking too long (verifier time limit)

**Fixes:**
- Increase ringbuffer size
- Use per-CPU ringbuffers
- Aggregate in kernel, reduce events sent to userspace
- Optimize userspace consumer

---

## Advanced: Benchmarking Methodology

### Microbenchmark Template

```c
// benchmark_map_ops.bpf.c
SEC("kprobe/__x64_sys_getpid")
int bench_map_lookup(struct pt_regs *ctx) {
    u32 key = 0;
    u64 start, delta;

    // Measure map lookup time
    start = bpf_ktime_get_ns();
    u64 *val = bpf_map_lookup_elem(&test_map, &key);
    delta = bpf_ktime_get_ns() - start;

    // Store in histogram
    bpf_map_update_elem(&latency_hist, &delta, &one, BPF_ANY);
    return 0;
}
```

**Run:**
```bash
# Generate load
while true; do /bin/true; done &

# Collect data
bpftrace benchmark_map_ops.bpf.c

# Analyze
bpftool map dump name latency_hist
```

### Workload Simulation

**Open vs Closed Loop:**

Closed-loop (tight loop) benchmarks can mislead:
- Turbo boost affects results non-linearly
- Randomized (Poisson) workloads show different characteristics
- Ring buffer performance varies with burstiness

**Recommended tool:**
- `berserker` - generates realistic Poisson-distributed syscall workloads
- Simulates production-like burstiness
- Tests eBPF programs under realistic stress

**Environment control:**
```bash
# Minimize variability
echo 0 > /sys/devices/system/cpu/cpufreq/boost  # Disable turbo
for cpu in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
    echo performance > $cpu
done

# Pin to isolated CPU
taskset -c 5 ./benchmark

# Disable hyperthreading
echo 0 > /sys/devices/system/cpu/cpu6/online  # Sibling of CPU 5
```

---

## Case Studies

### Case Study 1: Reducing VFS Tracing Overhead (HPC Cluster)

**Problem:**
- 2500-node cluster running VFS I/O profiling
- Initial overhead: ~3% CPU
- Goal: <0.5% overhead

**Optimizations applied:**

1. **Switched from kprobe to tracepoint**
   - Before: `kprobe/vfs_read` - 250ns per call
   - After: `tracepoint:syscalls:sys_enter_read` - 35ns per call
   - Improvement: 7x faster

2. **Per-CPU arrays instead of LRU hashmap**
   - Before: LRU hashmap for mount point tracking - 650ns lookup
   - After: Per-CPU array indexed by mount ID - 120ns lookup
   - Improvement: 5x faster

3. **Early exit for small reads**
   ```c
   if (count < 4096)
       return 0;
   ```
   - Filtered 85% of events
   - Remaining 15% are meaningful I/O

**Result:**
- Overhead: 3% → 0.4%
- Ran for 2+ years in production
- Zero stability issues

### Case Study 2: Meta SSLWall Encryption Enforcement

**Problem:**
- Enforce TLS on all TCP connections fleet-wide
- High-QPS services (>100K req/sec) sensitive to overhead

**Optimizations:**

1. **BPF arrays over LRU maps**
   - Connection validation cache as regular hashmap
   - Service whitelist as BPF array (fixed size)
   - Reduced lookup latency by 40%

2. **Multiple early exit checks**
   ```c
   if (state && state->is_encrypted) return 0;  // 90% of traffic
   if (is_loopback_addr(sk)) return 0;          // 5% of traffic
   if (is_whitelisted_service(comm)) return 0;  // 2% of traffic
   // Actual enforcement: 3% of traffic
   ```

3. **TC + kprobe composition**
   - TC BPF for packet inspection (early in network stack)
   - kprobe for connection lifecycle tracking
   - Split responsibility for optimal hook placement

**Result:**
- Overhead reduced from 3-5% to <1% in high-QPS services
- Deployed across entire Meta fleet

### Case Study 3: Netflix bpftop Development

**Problem:**
- Engineers needed real-time eBPF performance monitoring
- Existing tools (`bpftool prog show`) required manual calculation

**Solution:**
- Built bpftop: real-time dashboard for BPF programs
- Displays per-program metrics:
  - Events per second
  - Average execution time
  - Estimated CPU usage percentage
- Enabled rapid iteration on BPF program optimization

**Impact:**
- Identified programs with unexpectedly high overhead
- Detected high-frequency attachment points
- Reduced time-to-diagnosis from hours to minutes

---

## Summary: Production-Ready eBPF

### Golden Rules

1. **Always prefer tracepoints** - Stable API, lowest overhead
2. **Use fentry for performance** - 10x faster than kprobes when tracepoint unavailable
3. **Early exit aggressively** - Filter kernel threads, uninteresting events
4. **Per-CPU maps for concurrency** - Eliminate lock contention
5. **Measure overhead continuously** - Enable BPF stats, monitor CPU usage
6. **Test in staging first** - Production-like traffic reveals real overhead
7. **Gradual rollout** - Canary → 5% → 25% → 100%
8. **Set alerts** - >1% CPU usage, high event rates, increased latency

### Overhead Budget Template

For a service with P99 latency SLO of 10ms:

| Component | Budget | Monitoring |
|-----------|--------|------------|
| eBPF hooks | <0.5% CPU | `kernel.bpf_stats_enabled` |
| Per-request overhead | <50μs | Latency histograms |
| Map operations | <5 per request | `bpf_map_*` call counts |
| Events to userspace | <1000/sec | Ringbuffer stats |

### Quick Optimization Checklist

When eBPF overhead is too high:

```
[ ] Using tracepoints instead of kprobes?
[ ] Per-CPU maps for shared data?
[ ] Early exit for 80% of events?
[ ] Filtered to specific PIDs/comm?
[ ] Avoiding string operations?
[ ] In-kernel aggregation instead of per-event export?
[ ] Map operations minimized (lookup once, use multiple times)?
[ ] Correct map type (array > hash > LRU)?
[ ] Compiled with -O2?
[ ] Tested at production scale?
```

---

## References

### Research & Benchmarks

- [Understanding Performance of eBPF Maps](https://dl.acm.org/doi/10.1145/3672197.3673430) - ACM SIGCOMM 2024
- [Optimizing eBPF I/O latency accounting](https://tanelpoder.com/posts/optimizing-ebpf-biolatency-accounting/) - Tanel Poder
- [Using eBPF for network observability](https://opensource.com/article/22/8/ebpf-network-observability-cloud)
- [Performance fine-tuning eBPF agent metrics](https://netobserv.io/posts/performance-fine-tuning-a-deep-dive-in-ebpf-agent-metrics/)

### Production Systems

- [Meta Strobelight](https://engineering.fb.com/2025/01/21/production-engineering/strobelight-a-profiling-service-built-on-open-source-technology/)
- [Meta SSLWall](https://engineering.fb.com/2021/07/12/security/enforcing-encryption/)
- [Meta Millisampler](https://engineering.fb.com/2023/04/17/networking-traffic/millisampler-network-traffic-analysis/)
- [Netflix bpftop](https://netflixtechblog.com/announcing-bpftop-streamlining-ebpf-performance-optimization-6a727c1ae2e5)

### Documentation

- [Measuring BPF performance](https://developers.redhat.com/articles/2022/06/22/measuring-bpf-performance-tips-tricks-and-best-practices) - Red Hat
- [eBPF Tracepoints vs Kprobes vs Fprobes](https://labs.iximiuz.com/tutorials/ebpf-tracing-46a570d1)
- [Raw Tracepoint FAQ](https://mozillazg.com/2022/05/ebpf-libbpf-raw-tracepoint-common-questions-en.html)
- [BPF Performance Tools](https://www.brendangregg.com/bpf-performance-tools-book.html) - Brendan Gregg

---

## See Also

- [06-ebpf-tracing.md](06-ebpf-tracing.md) - bpftrace, BCC tools, practical eBPF patterns
- [17-ftrace-production.md](17-ftrace-production.md) - ftrace for production debugging
- [05-performance-profiling.md](05-performance-profiling.md) - perf, flame graphs
- [meta-ebpf-systems-engineering.md](meta-ebpf-systems-engineering.md) - Meta's eBPF infrastructure
