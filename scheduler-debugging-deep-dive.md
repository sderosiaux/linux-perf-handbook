# Linux Scheduler Debugging Deep Dive

Practical guide to diagnosing CFS scheduler bugs, wakeup latency, run queue attribution, and container scheduling pathologies using ftrace, eBPF, and Perfetto.

## Table of Contents

1. [CFS Scheduler Bugs & Detection](#cfs-scheduler-bugs--detection)
2. [Run Queue Latency Attribution](#run-queue-latency-attribution)
3. [ftrace Synthetic Events for Scheduler Analysis](#ftrace-synthetic-events-for-scheduler-analysis)
4. [Perfetto Visualization](#perfetto-visualization)
5. [Container/Cgroup Scheduler Interactions](#containercgroup-scheduler-interactions)
6. [Decision Trees for LLM-Driven Troubleshooting](#decision-trees-for-llm-driven-troubleshooting)

---

## CFS Scheduler Bugs & Detection

### The "Decade of Wasted Cores" Bugs

The [Linux Scheduler: A Decade of Wasted Cores](https://people.ece.ubc.ca/sasha/papers/eurosys16-final29.pdf) paper (EuroSys 2016) documented four critical CFS bugs that silently degraded performance without crashing the system:

1. **Group Imbalance Bug**: Autogroups + hierarchical load balancing created imbalanced load distribution
2. **Scheduling Group Construction Bug**: NUMA asymmetry in complex topologies caused incorrect group construction
3. **Overload-on-Wakeup Bug**: Threads wake up on overloaded CPUs while others idle
4. **Missing Scheduling Domains Bug**: NUMA-ness caused missing scheduling domains

**Why they persisted undetected:**
- No crashes or OOM conditions
- Introduced **microscopic idle periods** that moved between cores
- Tools like `htop`, `sar`, `perf` showed aggregate utilization, missing the problem
- Cache-coherency optimizations masked the symptoms

### Overload-on-Wakeup Bug (Most Critical)

**Symptom**: IO-bound thread wakes up, stays queued on original CPU (which is busy) even when other CPUs are idle. Results in **10-15ms stalls**.

**Detection commands:**
```bash
# Monitor per-CPU run queue depth
runqlat-bpfcc -C 10 1
# Look for: high latency on specific CPUs while others show low latency

# Trace wakeup events showing CPU affinity issues
bpftrace -e 'tracepoint:sched:sched_waking {
  printf("%s waking on CPU %d\n", comm, cpu);
}
tracepoint:sched:sched_switch {
  if (args->prev_state == 0) {
    printf("CPU %d: %s -> %s (waited %dus)\n",
      cpu, args->prev_comm, args->next_comm,
      (nsecs - @start[args->next_pid]) / 1000);
  }
}'

# Check CPU idle time distribution
mpstat -P ALL 1 10 | awk 'NR>3 {print $3, $13}'
# Look for: some CPUs at 100% busy, others with >10% idle
```

**Verification test:**
```bash
# Launch CPU-intensive tasks = num_CPUs - 1
for i in $(seq 1 $(($(nproc) - 1))); do
  stress-ng --cpu 1 --timeout 60s &
done

# Launch IO-intensive task
dd if=/dev/zero of=/tmp/test bs=1M count=1000 oflag=direct &

# Monitor scheduler distribution
perf sched record -a -- sleep 10
perf sched latency --sort max

# Red flag: IO task shows >10ms max delay while one CPU idle
```

**Root cause**: CFS's `select_idle_sibling()` prefers cache-warm CPUs over idle CPUs to avoid cache misses. For IO tasks (long sleep), cache is already cold, so the optimization backfires.

**Mitigation** (kernel 5.8+ improved, not eliminated):
```bash
# Force migration to idle CPUs for latency-sensitive tasks
taskset -c <idle_cpu> ./latency_sensitive_app

# Or use cpuset to spread across all cores
echo 0 > /sys/fs/cgroup/cpuset/<cgroup>/cpuset.sched_load_balance
# Disables CFS load balancing within this cgroup
```

### Detection Pattern: Microscopic Idle Periods

**Problem**: Standard tools aggregate over 1+ seconds, missing sub-millisecond idle periods.

**High-resolution detection:**
```bash
# Sample CPU state at 1000 Hz (1ms resolution)
perf stat -e cpu-clock -I 1 -a -- sleep 10

# Or use bpftrace to histogram idle periods
bpftrace -e 'kprobe:schedule {
  @start[cpu] = nsecs;
}
kretprobe:schedule /@start[cpu]/ {
  @idle_us = hist((nsecs - @start[cpu]) / 1000);
  delete(@start[cpu]);
}
interval:s:10 { exit(); }'

# Red flag: Histogram shows bimodal distribution (some CPUs idle, some not)
```

### Group Imbalance Bug

**Symptom**: Threads in different control groups unevenly distributed despite available capacity.

**Detection:**
```bash
# Check cgroup CPU distribution
for cg in /sys/fs/cgroup/cpu/*/; do
  name=$(basename $cg)
  usage=$(cat $cg/cpuacct.usage 2>/dev/null)
  echo "$name: $usage ns"
done | sort -t: -k2 -rn

# Look for: massive imbalance (10x difference) between cgroups

# Trace load balancing decisions
trace-cmd record -e sched:sched_migrate_task \
  -e sched:sched_stick_numa \
  -f 'cpu != prev_cpu' -- sleep 10
trace-cmd report | grep -E 'migrate|stick'
```

**Root cause**: CFS tries to balance load per-group, but autogroups (one per TTY session) + hierarchy creates competing balance constraints.

**Mitigation:**
```bash
# Disable autogroup for servers (not interactive workstations)
echo 0 > /proc/sys/kernel/sched_autogroup_enabled

# Or manually assign tasks to same cgroup
echo $PID > /sys/fs/cgroup/cpu/shared/tasks
```

---

## Run Queue Latency Attribution

**Core question**: Is high run queue latency due to **noisy neighbor** or **CPU quota exhausted**?

### Metrics to Distinguish

| Metric | Noisy Neighbor | Quota Exhausted |
|--------|----------------|-----------------|
| Preempted by other cgroup | High | Low |
| Preempted by same cgroup | Low | High |
| CPU throttle events | 0 | > 0 |
| PSI CPU pressure (full) | Low | High |

### eBPF-Based Attribution (Netflix Approach)

Netflix's [noisy neighbor detection](https://netflixtechblog.com/noisy-neighbor-detection-with-ebpf-64b1f4b3bbdd) uses three hooks:

```bash
# Pseudo-code BPF program
# Hook 1: Record wakeup timestamp
tracepoint:sched:sched_wakeup {
  @wakeup_ts[args->pid] = nsecs;
}

# Hook 2: Same for new tasks
tracepoint:sched:sched_wakeup_new {
  @wakeup_ts[args->pid] = nsecs;
}

# Hook 3: Calculate run queue latency + attribution
tracepoint:sched:sched_switch {
  $wakeup = @wakeup_ts[args->next_pid];
  if ($wakeup) {
    $latency_us = (nsecs - $wakeup) / 1000;

    # Get cgroup IDs
    $my_cgroup = cgroup_id();
    $prev_cgroup = cgroup_id_from_pid(args->prev_pid);

    # Attribute preemption
    if ($prev_cgroup != $my_cgroup) {
      @noisy_neighbor[$my_cgroup, $prev_cgroup] = hist($latency_us);
    } else {
      @self_contention[$my_cgroup] = hist($latency_us);
    }

    delete(@wakeup_ts[args->next_pid]);
  }
}
```

**Production-ready BCC tool** (runqlat with cgroup attribution):
```bash
# Run queue latency per cgroup (requires BCC with cgroup support)
runqlat-bpfcc -C 10 1

# Enhanced version with preemption tracking
cat > /tmp/runqlat_attribution.bt << 'EOF'
#include <linux/sched.h>

tracepoint:sched:sched_wakeup,
tracepoint:sched:sched_wakeup_new {
  @qtime[args->pid] = nsecs;
}

tracepoint:sched:sched_switch {
  if (@qtime[args->next_pid]) {
    $lat_us = (nsecs - @qtime[args->next_pid]) / 1000;

    # Simplified cgroup detection (requires kernel support)
    $cg_self = cgroup_id();
    $cg_prev = ((struct task_struct *)curtask)->cgroups->dfl_cgrp->kn->id;

    if ($lat_us > 0) {
      @runqlat_us = hist($lat_us);
      @runqlat_by_prev_task[args->prev_comm] = hist($lat_us);
    }

    delete(@qtime[args->next_pid]);
  }
}

interval:s:10 { exit(); }
EOF

bpftrace /tmp/runqlat_attribution.bt
```

### Kubernetes-Specific Attribution

**Problem**: In K8s, need to map cgroup → pod → namespace.

**Tool: Linnix** ([PSI tells you "What". Cgroups tell you "Who"](https://getlinnix.substack.com/p/psi-tells-you-what-cgroups-tell-you))

```bash
# Query: Which pods consumed CPU while payments-api was active?
curl http://localhost:8080/api/attribution/cpu?target=payments-api&duration=60s

# Response shows:
# {
#   "target_pod": "payments-api",
#   "top_consumers": [
#     {"pod": "batch-job-abc", "cpu_ms": 45000},
#     {"pod": "logging-agent", "cpu_ms": 12000}
#   ]
# }
```

**Manual K8s cgroup mapping:**
```bash
# Get pod cgroup path
POD="payments-api-xyz"
CONTAINER_ID=$(kubectl get pod $POD -o jsonpath='{.status.containerStatuses[0].containerID}' | cut -d/ -f3)

# Find cgroup
CGROUP_PATH=$(find /sys/fs/cgroup -name "*${CONTAINER_ID:0:12}*" -type d | head -1)

# Check CPU throttling (quota exhausted indicator)
cat $CGROUP_PATH/cpu.stat
# nr_throttled > 0 = quota issue, not noisy neighbor
# throttled_usec growing = latency directly from quota

# Check PSI
cat $CGROUP_PATH/cpu.pressure
# full avg10 > 5.0 = all tasks CPU-starved (quota or noisy neighbor)

# Correlate with system-wide sched events
trace-cmd record -e sched:sched_switch -f "next_comm ~ \"*$POD*\"" sleep 10
trace-cmd report --cpu $(taskset -p $(pgrep -f $POD | head -1) | awk '{print $NF}')
```

### Decision Logic: Noisy Neighbor vs Quota

```bash
#!/bin/bash
# Diagnose run queue latency cause

CGROUP_PATH="$1"
RQ_LATENCY_MS="$2"  # From runqlat or metrics

# Check throttling
nr_throttled=$(awk '/nr_throttled/ {print $2}' $CGROUP_PATH/cpu.stat)
throttled_usec=$(awk '/throttled_usec/ {print $2}' $CGROUP_PATH/cpu.stat)

# Check PSI
psi_cpu_full=$(awk '/full/ {print $2}' $CGROUP_PATH/cpu.pressure | grep -oP 'avg10=\K[0-9.]+')

if [ "$nr_throttled" -gt 0 ] && [ "$throttled_usec" -gt 1000000 ]; then
  echo "DIAGNOSIS: CPU quota exhausted"
  echo "  - nr_throttled: $nr_throttled"
  echo "  - throttled_time: $((throttled_usec / 1000000))s"
  echo "ACTION: Increase cpu.cfs_quota_us or remove limit"

elif (( $(echo "$psi_cpu_full > 5.0" | bc -l) )); then
  echo "DIAGNOSIS: Severe CPU contention (check noisy neighbors)"
  echo "  - PSI CPU full: ${psi_cpu_full}%"
  echo "ACTION: Use eBPF to identify preempting cgroups"

elif (( $(echo "$RQ_LATENCY_MS > 10" | bc -l) )); then
  echo "DIAGNOSIS: Scheduler issue (possibly Overload-on-Wakeup bug)"
  echo "  - Run queue latency: ${RQ_LATENCY_MS}ms"
  echo "ACTION: Check per-CPU distribution, consider CPU pinning"

else
  echo "DIAGNOSIS: Healthy (latency within normal range)"
fi
```

---

## ftrace Synthetic Events for Scheduler Analysis

### SQL-like Latency Analysis with sqlhist

**Tool**: `sqlhist` from libtracefs ([Event Histograms](https://docs.kernel.org/trace/histogram.html))

**Installation:**
```bash
# Debian/Ubuntu
apt install libtracefs-dev trace-cmd

# Build sqlhist example
git clone https://git.kernel.org/pub/scm/libs/libtrace/libtracefs.git
cd libtracefs
make
# Binary: utest/sqlhist
```

**Example: Wakeup latency distribution**
```bash
# SQL-like syntax to measure wakeup → schedule latency
sqlhist -e -n wakeup_latency \
  "SELECT start.pid, (end.TIMESTAMP_USECS - start.TIMESTAMP_USECS) AS latency_us
   FROM sched_waking AS start
   JOIN sched_switch AS end
   ON start.pid = end.next_pid
   WHERE latency_us > 100"

# Output: histogram of processes with >100us wakeup latency
# [PID] latency_us: count
# 1234  150: 45
# 1234  200: 23
# ...
```

**Example: Blocked syscall duration**
```bash
# Measure time processes spend blocked
sqlhist -e -n blocked_time \
  "SELECT enter.common_pid,
          (exit.TIMESTAMP_USECS - enter.TIMESTAMP_USECS) AS blocked_us
   FROM raw_syscalls:sys_enter AS enter
   JOIN raw_syscalls:sys_exit AS exit
   ON enter.common_pid = exit.common_pid
   WHERE blocked_us > 1000"
```

### Manual Synthetic Events (Low-Level)

**Create synthetic event for scheduler latency:**
```bash
T=/sys/kernel/tracing

# Define synthetic event
echo 'sched_lat u64 pid; u64 lat_us' > $T/synthetic_events

# Histogram 1: Save wakeup timestamp per PID
echo 'hist:keys=pid:ts0=common_timestamp.usecs' \
  > $T/events/sched/sched_waking/trigger

# Histogram 2: Calculate latency on switch, trigger synthetic event
echo 'hist:keys=next_pid:lat_us=common_timestamp.usecs-$ts0:onmatch(sched.sched_waking).sched_lat(next_pid,$lat_us)' \
  > $T/events/sched/sched_switch/trigger

# Histogram 3: Aggregate synthetic events
echo 'hist:keys=pid:vals=lat_us:sort=lat_us.desc' \
  > $T/events/synthetic/sched_lat/trigger

# Enable events
echo 1 > $T/events/sched/sched_waking/enable
echo 1 > $T/events/sched/sched_switch/enable
echo 1 > $T/events/synthetic/sched_lat/enable

# Wait for data
sleep 10

# Read results
cat $T/events/synthetic/sched_lat/hist
```

**Sample output:**
```
# event histogram
#
# trigger info: hist:keys=pid:vals=lat_us:sort=lat_us.desc
#

{ pid:   1234 } hitcount:    523  lat_us:   15234 (avg: 29us)
{ pid:   5678 } hitcount:   1245  lat_us:    8976 (avg: 7us)
{ pid:   9012 } hitcount:     89  lat_us:    1234 (avg: 13us)

Totals:
    Hits: 1857
    Entries: 3
```

### Latency Percentiles with Histogram Buckets

```bash
T=/sys/kernel/tracing

# Create latency histogram with buckets
echo 'hist:keys=lat_us.buckets=0,10,50,100,500,1000,5000,10000:vals=lat_us' \
  > $T/events/synthetic/sched_lat/trigger

# Result shows distribution:
# < 10us: 89% of wakeups
# 10-50us: 8%
# 50-100us: 2%
# >100us: 1% (RED FLAG for latency-sensitive workloads)
```

### Cleanup
```bash
T=/sys/kernel/tracing

# Disable events
echo 0 > $T/events/sched/sched_waking/enable
echo 0 > $T/events/sched/sched_switch/enable
echo 0 > $T/events/synthetic/sched_lat/enable

# Remove triggers
echo '!hist:keys=pid:ts0=common_timestamp.usecs' \
  > $T/events/sched/sched_waking/trigger
echo '!hist:keys=next_pid:lat_us=common_timestamp.usecs-$ts0:onmatch(sched.sched_waking).sched_lat(next_pid,$lat_us)' \
  > $T/events/sched/sched_switch/trigger
echo '!hist:keys=pid:vals=lat_us:sort=lat_us.desc' \
  > $T/events/synthetic/sched_lat/trigger

# Delete synthetic event
echo '!sched_lat u64 pid; u64 lat_us' > $T/synthetic_events
```

---

## Perfetto Visualization

**Perfetto** ([perfetto.dev](https://perfetto.dev/)) provides browser-based timeline visualization of scheduler traces with nanosecond accuracy.

### Recording Scheduler Traces

**Method 1: Perfetto native (Android/Linux)**
```bash
# Install perfetto
curl -LO https://get.perfetto.dev/perfetto
chmod +x perfetto

# Record scheduler events
./perfetto \
  -c - --txt \
  -o /tmp/trace.perfetto-trace << EOF
buffers {
  size_kb: 65536
}
data_sources {
  config {
    name: "linux.ftrace"
    ftrace_config {
      ftrace_events: "sched/sched_switch"
      ftrace_events: "sched/sched_waking"
      ftrace_events: "sched/sched_wakeup"
      ftrace_events: "sched/sched_process_exit"
      ftrace_events: "sched/sched_process_free"
    }
  }
}
duration_ms: 10000
EOF

# Open in browser
xdg-open https://ui.perfetto.dev
# Drag /tmp/trace.perfetto-trace into browser
```

**Method 2: Convert ftrace to Perfetto**
```bash
# Record with trace-cmd
trace-cmd record -e sched:sched_switch \
  -e sched:sched_waking \
  -e sched:sched_wakeup \
  sleep 10

# Convert to Perfetto format
trace-cmd report -R > /tmp/ftrace.txt

# Convert using traceconv (part of Perfetto SDK)
traceconv text /tmp/ftrace.txt /tmp/trace.perfetto-trace

# Or use online converter at ui.perfetto.dev
```

### Perfetto UI Features for Scheduler Analysis

**CPU Scheduling Tracks:**
- Each CPU gets a track showing which thread ran when
- Color-coded by thread/process
- Gaps = CPU idle time
- Click slice → see thread state transitions

**Thread State Tracks:**
- Running (green)
- Runnable/Queued (orange) ← **Key for detecting run queue latency**
- Sleeping (gray)
- Blocked on I/O (blue)

**Wakeup Arrows:**
- Shows which thread woke up another
- Follow chains: Thread A → Thread B → Thread C
- Useful for request tracing through microservices

**Key Metrics Panel:**
- Per-thread CPU time
- Scheduler latency (time in Runnable state)
- Context switches
- Migrations

### Analyzing Overload-on-Wakeup Bug

**Steps in Perfetto UI:**
1. Open trace in https://ui.perfetto.dev
2. Expand "CPU 0", "CPU 1", etc. tracks
3. Look for pattern:
   - CPU N: solid green (100% busy)
   - CPU M: intermittent green with gaps (idle periods)
   - Thread X: alternating orange (runnable) on CPU N, despite CPU M idle
4. Click orange "Runnable" slice → see duration (should be <1ms, if >10ms = bug)
5. Check "Waking CPU" vs "Running CPU" in details panel
   - If different AND waking CPU was idle = Overload-on-Wakeup bug

**SQL Query in Perfetto:**
```sql
-- Find threads with high runnable time (queued, not running)
SELECT
  thread.name,
  thread.tid,
  SUM(CASE WHEN state = 'R' THEN dur ELSE 0 END) / 1e6 AS runnable_ms,
  SUM(CASE WHEN state = 'Running' THEN dur ELSE 0 END) / 1e6 AS running_ms
FROM sched_slice
JOIN thread USING(utid)
GROUP BY utid
HAVING runnable_ms > 100
ORDER BY runnable_ms DESC;
```

### Detecting Noisy Neighbor in Perfetto

```sql
-- Find processes that preempted others frequently
SELECT
  preempting.name AS preemptor,
  preempted.name AS victim,
  COUNT(*) AS preemptions
FROM (
  SELECT
    ts,
    LEAD(ts) OVER (PARTITION BY cpu ORDER BY ts) AS next_ts,
    utid,
    LEAD(utid) OVER (PARTITION BY cpu ORDER BY ts) AS next_utid,
    cpu
  FROM sched_slice
) AS switches
JOIN thread AS preempting ON switches.next_utid = preempting.utid
JOIN thread AS preempted ON switches.utid = preempted.utid
WHERE preempting.name != preempted.name
GROUP BY preemptor, victim
ORDER BY preemptions DESC
LIMIT 20;
```

### Export Perfetto Metrics

```bash
# Run trace_processor (CLI tool from Perfetto SDK)
trace_processor /tmp/trace.perfetto-trace

# Interactive SQL queries
> SELECT * FROM sched_slice LIMIT 10;

# Export to CSV
> .mode csv
> .output /tmp/sched_metrics.csv
> SELECT thread.name, SUM(dur)/1e6 AS cpu_ms FROM sched_slice JOIN thread USING(utid) GROUP BY utid;
> .quit

# Programmatic queries
echo "SELECT thread.name, SUM(dur)/1e6 FROM sched_slice JOIN thread USING(utid) GROUP BY utid" \
  | trace_processor /tmp/trace.perfetto-trace --run-metrics > metrics.txt
```

---

## Container/Cgroup Scheduler Interactions

### CFS Bandwidth Throttling Pathologies

**Problem**: Multi-threaded apps (JVM GC, Go runtime) can exhaust CPU quota early in period despite low average CPU%.

**Symptom detection:**
```bash
CGROUP=/sys/fs/cgroup/system.slice/docker-abc123.scope

# Check throttling stats
cat $CGROUP/cpu.stat
# nr_periods: 100000
# nr_throttled: 25000  ← 25% of periods hit throttling
# throttled_usec: 5000000000  ← 5 seconds total throttled

# Calculate throttle rate
nr_periods=$(awk '/nr_periods/ {print $2}' $CGROUP/cpu.stat)
nr_throttled=$(awk '/nr_throttled/ {print $2}' $CGROUP/cpu.stat)
throttle_pct=$((nr_throttled * 100 / nr_periods))
echo "Throttle rate: ${throttle_pct}%"

# Red flag thresholds:
# >5%: noticeable latency impact
# >25%: severe performance degradation
# >50%: container essentially running at half capacity
```

**Root cause trace:**
```bash
# Trace throttling events
trace-cmd record -e cpu:cpu_frequency \
  -e cgroup:cgroup_freeze \
  -e cgroup:cgroup_unfreeze \
  -f "cgroup ~ \"*docker-abc123*\"" \
  sleep 30

trace-cmd report | grep -E 'freeze|unfreeze'

# Or use bpftrace
bpftrace -e 'tracepoint:cgroup:* /comm == "java"/ {
  printf("%s: %s\n", probe, comm);
}'
```

### GOMAXPROCS Mismatch (Go Containers)

**Problem**: Go runtime sets GOMAXPROCS to host cores, not container limit. Creates CPU quota exhaustion.

**Example:**
- 4-core host, container limited to 1 core
- Go spawns 4 OS threads (GOMAXPROCS=4)
- All 4 threads try to run, exhaust 1-core quota in 25% of period
- Remaining 75% of period: throttled → 75ms stalls per 100ms period

**Detection:**
```bash
# Inside Go container
go version
# Check GOMAXPROCS
go env GOMAXPROCS
# Compare to cgroup limit
cat /sys/fs/cgroup/cpu.max  # cgroup v2
# 100000 100000  ← 1 CPU (100ms quota / 100ms period)

# Mismatch = GOMAXPROCS > 1 here
```

**Fix:**
```go
import _ "go.uber.org/automaxprocs"
// Automatically sets GOMAXPROCS to match cgroup CPU limit
```

Or manual:
```bash
# Set GOMAXPROCS environment variable
docker run -e GOMAXPROCS=1 --cpus=1 myapp
```

### Kubernetes CPU Manager (Static vs Dynamic)

**Static Policy**: Guarantees dedicated CPUs for Guaranteed QoS pods.

**Enable:**
```yaml
# kubelet config
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cpuManagerPolicy: static
```

**Pod spec:**
```yaml
resources:
  requests:
    cpu: "4"
  limits:
    cpu: "4"  # Must match requests for Guaranteed QoS
```

**Verification:**
```bash
# Check CPU manager allocations
kubectl get pod $POD -o jsonpath='{.metadata.annotations.cpu-manager\.kubernetes\.io/cpus}'
# Output: "2,3,4,5"  ← Dedicated CPUs

# Verify from inside pod
taskset -p 1
# pid 1's current affinity mask: 3c  ← binary 00111100 = CPUs 2-5
```

### cgroup v2 cpu.idle Feature (Kernel 6.2+)

**Problem**: Container with low CPU request starves when other containers spike.

**Solution**: `cpu.idle=1` makes cgroup yield CPU to others when not using it.

```bash
# Set cpu.idle (cgroup v2)
echo 1 > /sys/fs/cgroup/<path>/cpu.idle

# Effect: Container gets minimum guaranteed CPU (from cpu.weight)
# But doesn't block others when idle
# Good for: sidecar containers, logging agents
```

### Scheduler Latency in cgroup Hierarchy

**Latency impact**: Deeper cgroup hierarchy = more scheduler overhead.

**Measure hierarchy overhead:**
```bash
# Count cgroup depth
CGROUP_PATH="/sys/fs/cgroup/kubepods.slice/kubepods-besteffort.slice/kubepods-pod123.slice/docker-abc.scope"
depth=$(echo $CGROUP_PATH | tr '/' '\n' | wc -l)
echo "Cgroup depth: $depth"

# Red flag: depth > 6 can add measurable latency (100s of ns per schedule)

# Trace scheduler time in cgroup updates
perf record -e sched:sched_switch -a -g -- sleep 10
perf report --stdio | grep cgroup
```

---

## Decision Trees for LLM-Driven Troubleshooting

### Tree 1: High Scheduler Latency Diagnosis

```
START: Application experiencing latency spikes

Is CPU saturated? (vmstat r > nproc)
├─ YES
│  └─ Scale out or optimize CPU usage
└─ NO
   ├─ Check run queue latency: runqlat-bpfcc 10 1
   │  ├─ P99 < 1ms: HEALTHY, look elsewhere
   │  └─ P99 > 10ms: Continue investigation
   │     ├─ Check per-CPU distribution: runqlat-bpfcc -C 10 1
   │     │  ├─ High latency on some CPUs, others low
   │     │  │  └─ DIAGNOSIS: Overload-on-Wakeup bug
   │     │  │     ACTION: Pin workload to specific CPUs (taskset)
   │     │  │             or upgrade to kernel 5.8+
   │     │  └─ High latency uniform across CPUs
   │     │     ├─ In container? Check cgroup throttling
   │     │     │  └─ cat /sys/fs/cgroup/*/cpu.stat
   │     │     │     ├─ nr_throttled > 0
   │     │     │     │  └─ DIAGNOSIS: CPU quota exhausted
   │     │     │     │     ACTION: Increase cpu.cfs_quota_us
   │     │     │     │             Check for GOMAXPROCS mismatch (Go)
   │     │     │     └─ nr_throttled = 0
   │     │     │        └─ Check PSI: cat cpu.pressure
   │     │     │           ├─ full avg10 > 5.0
   │     │     │           │  └─ DIAGNOSIS: Severe CPU contention
   │     │     │           │     ACTION: eBPF attribution (noisy neighbor)
   │     │     │           └─ full avg10 < 5.0
   │     │     │              └─ DIAGNOSIS: Scheduler bug or config issue
   │     │     │                 ACTION: Check sched_autogroup_enabled
   │     │     │                         Verify NUMA affinity
   │     │     └─ Not in container? Check system scheduler config
   │     │        └─ cat /proc/sys/kernel/sched_autogroup_enabled
   │     │           ├─ 1 (enabled)
   │     │           │  └─ DIAGNOSIS: Autogroup causing imbalance
   │     │           │     ACTION: echo 0 > sched_autogroup_enabled
   │     │           └─ 0 (disabled)
   │     │              └─ DIAGNOSIS: NUMA or topology issue
   │     │                 ACTION: numactl --hardware
   │     │                         Check CPU affinity with taskset -p
   └─ Check context switches: pidstat -w 1
       ├─ High voluntary (cswch/s > 1000)
       │  └─ DIAGNOSIS: I/O bound or lock contention
       │     ACTION: Profile I/O (iotop, offcputime-bpfcc)
       │             Check mutex contention (perf lock)
       └─ High involuntary (nvcswch/s > 1000)
          └─ DIAGNOSIS: Too many threads for CPU quota
             ACTION: Reduce thread count or increase CPU allocation
```

### Tree 2: Noisy Neighbor vs Quota Exhausted

```
START: Container experiencing CPU performance issues

Check throttling: cat /sys/fs/cgroup/*/cpu.stat
├─ nr_throttled > 0 AND throttled_usec growing
│  ├─ Calculate average CPU usage over last minute
│  │  ├─ Avg CPU% < CPU limit
│  │  │  └─ DIAGNOSIS: Bursty workload exhausting quota in period
│  │  │     ACTION: Increase cpu.cfs_period_us (e.g., 250ms)
│  │  │             OR enable cpu.cfs_burst_us (kernel 5.14+)
│  │  │             OR remove limits, use requests only
│  │  └─ Avg CPU% ≈ CPU limit
│  │     └─ DIAGNOSIS: Legitimately hitting limit
│  │        ACTION: Increase CPU quota or optimize application
│  └─ Check application type
│     ├─ Go application?
│     │  └─ Check GOMAXPROCS: go env GOMAXPROCS
│     │     └─ GOMAXPROCS > CPU limit
│     │        └─ DIAGNOSIS: Go spawning too many OS threads
│     │           ACTION: Import go.uber.org/automaxprocs
│     ├─ JVM application?
│     │  └─ Check GC threads: jstat -gcutil <pid>
│     │     └─ FGCT (Full GC time) high
│     │        └─ DIAGNOSIS: GC exhausting quota
│     │           ACTION: Tune GC or increase quota
│     └─ Other multi-threaded app
│        └─ Count threads: ls /proc/<pid>/task | wc -l
│           └─ Thread count >> CPU quota
│              └─ DIAGNOSIS: Thread pool oversized for quota
│                 ACTION: Reduce thread pool or increase quota
└─ nr_throttled = 0 BUT high run queue latency
   ├─ Run eBPF attribution: (Netflix approach)
   │  └─ bpftrace script tracking preemption by cgroup
   │     ├─ High preemption by other cgroups
   │     │  └─ DIAGNOSIS: Noisy neighbor
   │     │     ACTION: Identify culprit cgroup
   │     │             Request isolation (dedicated node, CPU pinning)
   │     │             OR file K8s priority/preemption
   │     └─ High self-preemption
   │        └─ DIAGNOSIS: Internal contention (too many threads)
   │           ACTION: Reduce thread count
   └─ Check PSI: cat /sys/fs/cgroup/*/cpu.pressure
      ├─ some avg10 > 20.0
      │  └─ DIAGNOSIS: CPU contention (noisy neighbor likely)
      │     ACTION: Correlate with node-level metrics
      │             Check other pods on node (kubectl top pods)
      └─ some avg10 < 10.0
         └─ DIAGNOSIS: Not CPU contention, look elsewhere
            ACTION: Check memory pressure, I/O, network
```

### Tree 3: Scheduler Bug Classification

```
START: Confirmed scheduler latency issue (not quota or noisy neighbor)

Check kernel version: uname -r
├─ Kernel < 5.8
│  └─ Likely affected by Overload-on-Wakeup bug
│     ACTION: Upgrade kernel OR manual CPU pinning
└─ Kernel ≥ 5.8
   ├─ Check CPU distribution: mpstat -P ALL 1 10
   │  ├─ Some CPUs 100% busy, others >10% idle
   │  │  └─ Load balancing failure
   │  │     ├─ Check NUMA topology: numactl --hardware
   │  │     │  └─ Asymmetric topology?
   │  │     │     └─ DIAGNOSIS: Scheduling Group Construction bug
   │  │     │        ACTION: Manually assign NUMA affinity
   │  │     └─ Check autogroup: cat /proc/sys/kernel/sched_autogroup_enabled
   │  │        └─ Enabled (1)
   │  │           └─ DIAGNOSIS: Group Imbalance bug
   │  │              ACTION: Disable autogroup (echo 0 > ...)
   │  └─ CPUs evenly utilized
   │     └─ Check cgroup hierarchy depth
   │        └─ Depth > 6
   │           └─ DIAGNOSIS: cgroup hierarchy overhead
   │              ACTION: Flatten hierarchy (K8s QoS classes)
   ├─ Check scheduler policy: chrt -p <pid>
   │  ├─ SCHED_NORMAL
   │  │  └─ Expected behavior for normal workloads
   │  │     ├─ Latency-sensitive?
   │  │     │  └─ ACTION: Upgrade to SCHED_FIFO or SCHED_DEADLINE
   │  │     └─ CPU-intensive batch?
   │  │        └─ ACTION: Switch to SCHED_BATCH
   │  └─ SCHED_FIFO/RR/DEADLINE
   │     └─ Check priority: cat /proc/<pid>/sched
   │        └─ Low priority (<50)
   │           └─ DIAGNOSIS: Preempted by higher-priority RT tasks
   │              ACTION: Increase priority (with caution)
   └─ Check scheduler tuning: sysctl -a | grep sched_
      ├─ sched_min_granularity_ns too high (>10ms)
      │  └─ DIAGNOSIS: Time slices too long
      │     ACTION: echo 3000000 > sched_min_granularity_ns
      └─ sched_migration_cost_ns too high (>5ms)
         └─ DIAGNOSIS: Sticky scheduling preventing migration
            ACTION: echo 500000 > sched_migration_cost_ns
```

### Tree 4: Container-Specific Quick Triage

```
START: Container performance issue (latency or throughput)

kubectl top pod <pod>
├─ CPU usage > 80% of limit
│  └─ Check throttling: See Tree 2
└─ CPU usage < 50% of limit
   ├─ Check memory: kubectl top pod <pod>
   │  └─ Memory > 90% of limit
   │     └─ OOM risk, not scheduler issue
   │        ACTION: Increase memory limit
   └─ Memory healthy
      ├─ Exec into pod: kubectl exec -it <pod> -- sh
      │  ├─ Run top/htop
      │  │  └─ High %wa (I/O wait)
      │  │     └─ DIAGNOSIS: I/O bottleneck, not CPU
      │  │        ACTION: Profile I/O (iotop, blktrace)
      │  └─ High %sy (system CPU)
      │     └─ Excessive syscalls
      │        ACTION: Profile syscalls (strace -c, perf top)
      └─ Check PSI from host:
         CGROUP=$(kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].containerID}' | cut -d/ -f3)
         cat /sys/fs/cgroup/*/*${CGROUP:0:12}*/cpu.pressure
         ├─ full avg10 > 5.0
         │  └─ DIAGNOSIS: CPU starvation
         │     ACTION: Use eBPF attribution (noisy neighbor check)
         └─ some avg10 > 20.0 BUT full < 5.0
            └─ DIAGNOSIS: Competing threads within container
               ACTION: Check thread count, reduce if excessive
```

---

## Metrics Interpretation Cheat Sheet

### Run Queue Latency

| P99 Latency | Assessment | Action |
|-------------|------------|--------|
| < 1ms | Healthy | None |
| 1-10ms | Elevated | Monitor, investigate if sustained |
| 10-50ms | Problematic | Active investigation required |
| > 50ms | Critical | Immediate action: likely scheduler bug |

### CPU Throttling

| Throttle Rate | Impact | Action |
|---------------|--------|--------|
| 0% | None | Quota is appropriate |
| 1-5% | Minor | May cause occasional latency blips |
| 5-25% | Moderate | Noticeable impact on latency-sensitive apps |
| > 25% | Severe | Container essentially running at reduced capacity |

### PSI CPU Pressure

| Metric | Threshold | Meaning |
|--------|-----------|---------|
| `some avg10` | > 10% | Contention beginning |
| `some avg60` | > 20% | Sustained contention |
| `full avg10` | > 5% | All tasks blocked (critical) |
| `full avg60` | > 10% | Persistent starvation |

### Context Switch Rates

| Rate (switches/sec) | Assessment | Likely Cause |
|---------------------|------------|--------------|
| < 1,000 | Healthy | Normal operation |
| 1K-10K | Moderate | Multi-threaded app, normal |
| 10K-50K | High | Over-synchronization or I/O bound |
| > 50K | Critical | Thread thrashing |

### Voluntary vs Involuntary Ratio

| Pattern | Diagnosis | Action |
|---------|-----------|--------|
| High voluntary, low involuntary | I/O or lock bound | Profile I/O, check locks |
| Low voluntary, high involuntary | CPU bound | Reduce threads or add CPUs |
| Both high | Thrashing | Architectural issue |
| Both low | Efficient | Healthy state |

---

## Command Quick Reference

### Detection
```bash
# Run queue latency distribution
runqlat-bpfcc 10 1

# Per-CPU run queue latency
runqlat-bpfcc -C 10 1

# CPU saturation
vmstat 1
mpstat -P ALL 1

# Context switches
pidstat -w 1

# Container throttling
cat /sys/fs/cgroup/*/cpu.stat

# PSI pressure
cat /sys/fs/cgroup/*/cpu.pressure
```

### Tracing
```bash
# Scheduler events with trace-cmd
trace-cmd record -e sched:sched_switch -e sched:sched_waking sleep 10
trace-cmd report

# ftrace synthetic events (wakeup latency)
sqlhist -e -n wakeup_lat \
  "SELECT start.pid, (end.TIMESTAMP_USECS - start.TIMESTAMP_USECS) AS lat
   FROM sched_waking AS start JOIN sched_switch AS end ON start.pid = end.next_pid"

# Perfetto timeline
./perfetto -c config.pbtxt -o trace.perfetto-trace
```

### Attribution
```bash
# eBPF noisy neighbor detection (pseudo-code)
bpftrace -e 'tracepoint:sched:sched_switch {
  # Track preemption by cgroup
}'

# K8s cgroup mapping
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[0].containerID}'
find /sys/fs/cgroup -name "*<container_id>*"
```

### Mitigation
```bash
# Disable autogroup
echo 0 > /proc/sys/kernel/sched_autogroup_enabled

# CPU pinning
taskset -c 2-5 ./app

# Increase CPU quota
echo 200000 > /sys/fs/cgroup/cpu/mygroup/cpu.cfs_quota_us

# Enable CPU burst (kernel 5.14+)
echo 100000 > /sys/fs/cgroup/cpu/mygroup/cpu.cfs_burst_us
```

---

## See Also

- [Scheduler & Interrupt Internals](16-scheduler-interrupts.md) - CFS mechanics, EEVDF, scheduling policies
- [ftrace for Production Debugging](17-ftrace-production.md) - Event tracing, histogram triggers, synthetic events
- [Containers & Kubernetes](07-containers-k8s.md) - cgroup metrics, PSI, container debugging
- [eBPF & Tracing](06-ebpf-tracing.md) - BPF-based scheduler analysis, runqlat, offcputime

## References

- [The Linux Scheduler: A Decade of Wasted Cores (2016)](https://people.ece.ubc.ca/sasha/papers/eurosys16-final29.pdf) - EuroSys paper documenting CFS bugs
- [The morning paper: Wasted Cores](https://blog.acolyer.org/2016/04/26/the-linux-scheduler-a-decade-of-wasted-cores/) - Accessible summary
- [Netflix: Noisy Neighbor Detection with eBPF](https://netflixtechblog.com/noisy-neighbor-detection-with-ebpf-64b1f4b3bbdd) - Production attribution approach
- [Event Histograms - Kernel Docs](https://docs.kernel.org/trace/histogram.html) - ftrace synthetic events and SQL-like analysis
- [Perfetto Documentation](https://perfetto.dev/docs/) - Timeline visualization and trace analysis
- [PSI tells you "What". Cgroups tell you "Who"](https://getlinnix.substack.com/p/psi-tells-you-what-cgroups-tell-you) - Kubernetes attribution
- [Brendan Gregg: Linux Run Queue Latency](https://www.brendangregg.com/blog/2016-10-08/linux-bcc-runqlat.html) - runqlat tool explanation
- [LWN: Inter-event (latency) support](https://lwn.net/Articles/737776/) - ftrace synthetic events development
- [CPU Scheduling events - Perfetto](https://perfetto.dev/docs/data-sources/cpu-scheduling) - Scheduler trace collection

---

**Document Status**: Production-ready reference for LLM-assisted scheduler debugging and decision-making.
