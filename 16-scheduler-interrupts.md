# Scheduler & Interrupt Internals

Deep-dive into Linux CPU scheduling, interrupt handling, and CPU isolation for latency-critical workloads.

## Scheduler Classes

Linux implements multiple scheduling classes with strict priority ordering:

```
Priority (highest to lowest):
SCHED_DEADLINE > SCHED_FIFO/RR > SCHED_NORMAL/BATCH/IDLE
     |                |                    |
  Real-time       Real-time             Normal
  (EDF-based)     (static prio)      (EEVDF/CFS)
```

### Quick Reference: Scheduling Policy Selection

| Policy | Use Case | Priority | Preemption |
|--------|----------|----------|------------|
| `SCHED_DEADLINE` | Hard RT: audio DSP, video frames, control loops | Highest | By deadline |
| `SCHED_FIFO` | Soft RT: deterministic latency, no time slice | Static 1-99 | Only by higher prio |
| `SCHED_RR` | Soft RT: round-robin among same priority | Static 1-99 | Time quantum + higher prio |
| `SCHED_NORMAL` | Default: interactive and batch | Dynamic (nice) | EEVDF virtual deadline |
| `SCHED_BATCH` | CPU-intensive: encoders, builds | Dynamic (nice) | Disfavored at wakeup |
| `SCHED_IDLE` | Background: backups, maintenance | Lowest | Only when idle |

### SCHED_DEADLINE: Guaranteed CPU Time

```bash
# Set deadline parameters: runtime, deadline, period (all in nanoseconds)
# Guarantee: runtime CPU time within deadline, repeating every period
chrt -d --sched-runtime 5000000 \
        --sched-deadline 10000000 \
        --sched-period 16666666 0 ./process

# Example: Video processing at 60 FPS (16.67ms period)
# - 5ms runtime guaranteed
# - Must complete within 10ms of period start
# - Period repeats every 16.67ms
chrt -d --sched-runtime 5000000 \
        --sched-deadline 10000000 \
        --sched-period 16666666 0 ffmpeg -i input.mp4 -c:v libx264 output.mp4

# Audio processing: 10ms buffer, 2ms processing time
chrt -d --sched-runtime 2000000 \
        --sched-deadline 8000000 \
        --sched-period 10000000 0 jackd -d alsa

# Check current deadline parameters
cat /proc/<pid>/sched | grep -E "dl_runtime|dl_deadline|dl_period"
```

**Constraint**: `runtime <= deadline <= period`

**Admission control**: Kernel rejects if total deadline utilization exceeds capacity:
```
sum(runtime_i / period_i) <= total_cpu_capacity
```

**When to use SCHED_DEADLINE**:
- Hard real-time with known WCET (worst-case execution time)
- Periodic tasks: audio, video, control loops
- Need guaranteed CPU regardless of other load

### SCHED_FIFO and SCHED_RR: Static Priority Real-Time

```bash
# SCHED_FIFO: Run until blocks or preempted by higher priority
chrt -f 80 ./latency_critical_app

# SCHED_RR: Same as FIFO but with time quantum rotation
chrt -r 50 ./soft_realtime_app

# Check time quantum for RR (default 100ms)
cat /proc/sys/kernel/sched_rr_timeslice_ms

# View process scheduling policy
chrt -p <pid>

# Priority range: 1 (lowest RT) to 99 (highest RT)
# Priority 99 reserved for kernel migration threads
# Use 1-98 for applications

# Set from within application (requires CAP_SYS_NICE)
# C: sched_setscheduler(0, SCHED_FIFO, &param)
# Python: os.sched_setscheduler(0, os.SCHED_FIFO, os.sched_param(80))
```

**SCHED_FIFO vs SCHED_RR**:
| Aspect | SCHED_FIFO | SCHED_RR |
|--------|------------|----------|
| Time slice | Unlimited | Default 100ms |
| Same-priority handling | FIFO queue | Round-robin |
| Use case | Single critical path | Multiple RT tasks |

**Warning**: RT tasks at priority 99 can starve kernel threads. Use priority 1-90 for applications.

### SCHED_BATCH and SCHED_IDLE: Background Work

```bash
# SCHED_BATCH: CPU-intensive, non-interactive
# Scheduler assumes CPU-bound, applies wakeup penalty
chrt -b 0 make -j$(nproc)
nice -n 19 chrt -b 0 ffmpeg -i video.mp4 -c:v libx265 output.mp4

# SCHED_IDLE: Only runs when system truly idle
# Gets ~1.4% CPU even under load (starvation protection)
chrt -i 0 updatedb
chrt -i 0 /usr/bin/btrfs-backup.sh

# Combine with ionice for full background mode
ionice -c 3 chrt -i 0 rsync -av /data /backup
```

**SCHED_BATCH use cases**:
- Video/audio encoding
- Database maintenance (VACUUM, compaction)
- Build systems, CI/CD
- Log analyzers, ETL jobs

**SCHED_IDLE use cases**:
- Index building (updatedb, mlocate)
- Backup scripts
- Garbage collection
- System maintenance

### EEVDF Scheduler (Kernel 6.6+)

EEVDF (Earliest Eligible Virtual Deadline First) replaced CFS as the default scheduler.

```bash
# Check kernel version
uname -r  # 6.6+ uses EEVDF

# Key behavior differences from CFS:
# - Virtual deadline instead of vruntime alone
# - Bounded latency guarantees
# - Simpler tuning (fewer heuristics)
# - Tasks can request shorter time slices

# Time slice range: 100us to 100ms
# Controlled via sched_setattr() syscall with sched_runtime parameter

# Lag tracking: positive lag = owed CPU time, negative = exceeded share
# Sleep handling: lag decays over virtual time (prevents sleep exploit)
```

**EEVDF vs CFS behavior**:

| Aspect | CFS | EEVDF |
|--------|-----|-------|
| Selection | Lowest vruntime | Earliest eligible deadline |
| Latency bound | Heuristic | Algorithmic |
| Sleep bonus | Explicit heuristics | Lag decay |
| Tuning knobs | Many (wakeup_granularity, etc.) | Fewer, cleaner |

**Tunable parameters (6.6+)**:
```bash
# Minimum time slice (default 3ms)
sysctl kernel.sched_min_granularity_ns=3000000

# Migration cost threshold
sysctl kernel.sched_migration_cost_ns=500000

# Disable autogroup for servers (group by session otherwise)
sysctl kernel.sched_autogroup_enabled=0
```

### CFS Bandwidth Throttling in Containers

```bash
# cgroup v1: Set quota and period
echo 250000 > /sys/fs/cgroup/cpu/mygroup/cpu.cfs_quota_us   # 250ms quota
echo 100000 > /sys/fs/cgroup/cpu/mygroup/cpu.cfs_period_us  # 100ms period
# Result: 2.5 CPUs worth of time (250ms / 100ms)

# cgroup v2: Single file
echo "250000 100000" > /sys/fs/cgroup/mygroup/cpu.max

# Docker equivalent
docker run --cpus=2.5 myimage
docker run --cpu-quota=250000 --cpu-period=100000 myimage

# Kubernetes equivalent
resources:
  limits:
    cpu: "2500m"  # 2.5 cores

# Check throttling stats (cgroup v1)
cat /sys/fs/cgroup/cpu/mygroup/cpu.stat
# nr_periods: total periods elapsed
# nr_throttled: periods with throttling
# throttled_time: total ns throttled

# cgroup v2
cat /sys/fs/cgroup/mygroup/cpu.stat
```

**Throttling pathology diagnosis**:
```bash
# If you see throttling with low CPU usage:
# - Common in multi-threaded apps (JVM GC, Go runtime)
# - All threads share the quota within period
# - Burst consumption exhausts quota early in period

# Symptom: nr_throttled >> 0 but %CPU < limit
cat /sys/fs/cgroup/cpu/*/cpu.stat | grep throttled

# Solutions:
# 1. Increase period (allows more burst)
echo 250000 > cpu.cfs_period_us  # Longer period
echo 625000 > cpu.cfs_quota_us   # Same ratio, more headroom

# 2. Remove limits, use requests only (K8s)
resources:
  requests:
    cpu: "2500m"
  # No limits specified

# 3. Use cpu.cfs_burst_us (kernel 5.14+)
echo 100000 > cpu.cfs_burst_us  # Allow 100ms burst accumulation
```

**Red flags for CFS throttling**:
- JVM STW GC pauses causing 100% quota consumption
- High throttled_time with low average CPU
- Latency spikes correlating with period boundaries

---

## CPU Saturation Patterns

### Run Queue Depth Interpretation

```bash
# vmstat: r column = processes waiting for CPU + currently running
vmstat 1
# procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
#  r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
#  5  0      0 123456  12345 234567    0    0     0     0 1000  500 50  5 45  0  0

# Interpretation:
# r > num_CPUs = CPU saturated
# r - num_CPUs = actual queue depth
# b > 0 = processes blocked on I/O (not CPU)

# Get CPU count for comparison
nproc

# Real queue depth (subtract running processes)
# r=12 on 8-core = 4 processes waiting

# BPF tool: precise run queue length distribution
runqlen 10 1  # 10 samples, 1 second
# Or with bcc:
/usr/share/bcc/tools/runqlen
```

**Run queue thresholds**:
| Ratio (r / nproc) | Status | Action |
|-------------------|--------|--------|
| < 1 | Healthy | None |
| 1-2 | Elevated | Monitor |
| 2-4 | Saturated | Investigate |
| > 4 | Severely saturated | Immediate action |

**Load average interpretation**:
```bash
uptime
# 10:00:00 up 30 days,  load average: 4.50, 3.20, 2.10

# Load = runnable + uninterruptible (D state) processes
# Includes I/O waiters, not just CPU demand

# Per-CPU load
awk '{print $1/'"$(nproc)"'}' /proc/loadavg

# Rule of thumb:
# load/nproc < 0.7  : Healthy headroom
# load/nproc = 1.0  : Fully utilized, no headroom
# load/nproc > 2.0  : Investigate immediately

# Caveat: load includes D-state processes
# High load + low CPU% = I/O bound, not CPU bound
```

### Context Switch Overhead Measurement

```bash
# System-wide context switches
vmstat 1
# cs column = context switches/second

# Per-process context switches
pidstat -w 1
# cswch/s  = voluntary (waiting for resource)
# nvcswch/s = involuntary (time slice expired)

# Example output:
# PID   cswch/s nvcswch/s  Command
# 1234   150.00    50.00   myapp

# High voluntary = I/O bound or lock contention
# High involuntary = CPU bound, being preempted

# Detailed per-thread
pidstat -w -t 1

# Process lifetime totals
cat /proc/<pid>/status | grep ctxt
# voluntary_ctxt_switches:    12345
# nonvoluntary_ctxt_switches: 6789

# perf stat for specific workload
perf stat -e context-switches,cpu-migrations ./myapp
```

**Context switch thresholds**:
| Rate | Assessment | Notes |
|------|------------|-------|
| < 10K/s | Normal | Typical server |
| 10K-50K/s | Elevated | Check if justified |
| 50K-100K/s | High | Likely performance impact |
| > 100K/s | Critical | Thread thrashing likely |

**Cost per context switch**: 1-10 microseconds depending on cache state

```bash
# Identify context switch causes
perf sched record -a sleep 5
perf sched latency --sort max

# High involuntary switches indicate:
# - Too many threads for available CPUs
# - Short time slices under contention
# - Aggressive RT processes preempting

# High voluntary switches indicate:
# - Frequent I/O or syscalls
# - Lock contention (mutex, futex)
# - Over-synchronization
```

### Load Balancing Issues on NUMA

```bash
# Check NUMA topology
numactl --hardware
# node 0 cpus: 0-15
# node 1 cpus: 16-31
# node distances:
# node   0   1
#   0:  10  21
#   1:  21  10

# Monitor cross-node scheduling
perf stat -e node-loads,node-stores ./myapp

# CPU migrations (cross-socket is expensive)
perf stat -e cpu-migrations ./myapp

# Per-process NUMA stats
numastat -p <pid>

# Check scheduler domains
cat /proc/sys/kernel/sched_domain/cpu0/domain*/name
# SMT, MC, NUMA

# Memory locality
cat /proc/<pid>/numa_maps | head
# Shows where memory pages are allocated vs accessed
```

**NUMA scheduling pathologies**:

```bash
# Symptom: High memory latency despite free local memory
# Diagnosis:
numastat -p <pid>
# Look for high "other_node" counts

# Fix 1: Bind to NUMA node
numactl --cpunodebind=0 --membind=0 ./myapp

# Fix 2: Disable auto NUMA balancing if it's hurting
sysctl kernel.numa_balancing=0

# Fix 3: CPU pinning to keep memory local
taskset -c 0-15 ./myapp  # Stay on node 0

# Verify NIC NUMA affinity
cat /sys/class/net/eth0/device/numa_node
# Schedule network processing on same node
```

**Cross-node migration cost**:
- Same socket, different core: ~50-100ns cache miss
- Different socket: ~100-300ns + cache invalidation

### perf sched Workflows

```bash
# Record scheduler events (warning: high overhead)
perf sched record -a -- sleep 10
# Or for specific command:
perf sched record -- ./myapp

# Latency report: per-task scheduling delays
perf sched latency
# Shows: max, avg scheduler latency per task
#  Task                  |   Runtime ms  | Switches | Average delay ms | Maximum delay ms |
# -----------------------------------------------------------------
#  stress:12345          |   8000.000 ms |     1234 | avg:    0.050 ms | max:   15.000 ms |

# Sort by worst offenders
perf sched latency --sort max

# Timeline view (since kernel 4.10)
perf sched timehist
# Columns: time, cpu, task, wait time, sch delay, runtime
# wait time = time sleeping
# sch delay = time runnable but not running (queue time)

# Add CPU visualization and migrations
perf sched timehist -MVw
# -M = show migrations
# -V = CPU visualization
# -w = wakeup events

# Summary only
perf sched timehist --summary

# CPU map view (context switches across CPUs)
perf sched map --compact
# Shows which task ran on which CPU at what time
# * = CPU with event, . = idle CPU

# Replay for analysis (experimental)
perf sched replay
```

**Interpreting perf sched latency output**:
```
# Good: max delay < 1ms for interactive
# Warning: max delay > 10ms
# Critical: max delay > 100ms

# High switches + low runtime = thrashing
# Low switches + high runtime = batch workload (normal)
```

**BPF-based scheduler analysis** (lower overhead):
```bash
# Run queue latency distribution
runqlat 10 1  # From bcc-tools
# Shows histogram of time spent waiting in run queue

# Run queue length over time
runqlen 10 1

# Off-CPU time analysis (why tasks aren't running)
offcputime -p <pid> 10
```

---

## Interrupt Handling

### IRQ Affinity for Network-Heavy Workloads

```bash
# View current IRQ assignments
cat /proc/interrupts
#            CPU0       CPU1       CPU2       CPU3
#  27:    1234567          0          0          0   IR-PCI-MSI  eth0-TxRx-0
#  28:          0    2345678          0          0   IR-PCI-MSI  eth0-TxRx-1

# Check NIC queue to IRQ mapping
ls /sys/class/net/eth0/queues/
# rx-0 rx-1 rx-2 rx-3 tx-0 tx-1 tx-2 tx-3

# View IRQ affinity (hex bitmask)
cat /proc/irq/27/smp_affinity
# 00000001 = CPU 0

# Set IRQ to specific CPU (hex mask)
# CPU 0 = 1, CPU 1 = 2, CPU 2 = 4, CPU 3 = 8
echo 2 > /proc/irq/27/smp_affinity  # CPU 1
echo f > /proc/irq/27/smp_affinity  # CPUs 0-3

# List format (easier)
echo 0-3 > /proc/irq/27/smp_affinity_list

# Per-NIC queue affinity
cat /sys/class/net/eth0/queues/rx-0/rps_cpus

# Stop irqbalance for manual control
systemctl stop irqbalance
systemctl disable irqbalance
```

**Multi-queue NIC strategy**:
```bash
# Ideal: 1 queue per CPU, spread across NUMA node
# Check number of queues
ethtool -l eth0
# Combined: 8 (current), max 16

# Increase queues if needed
ethtool -L eth0 combined 16

# Spread IRQs across CPUs on same NUMA node as NIC
cat /sys/class/net/eth0/device/numa_node
# 0

# Pin each queue IRQ to different CPU on node 0
for i in {27..34}; do
  cpu=$((i - 27))
  echo $cpu > /proc/irq/$i/smp_affinity_list
done

# Verify distribution
grep eth0 /proc/interrupts
```

**Receive Side Scaling (RSS) tuning**:
```bash
# Check RSS hash settings
ethtool -x eth0
# Shows indirection table and hash key

# Configure flow distribution
ethtool -X eth0 equal 8  # Spread across 8 queues evenly

# Set hash fields (for better distribution)
ethtool -N eth0 rx-flow-hash tcp4 sdfn  # src/dst IP + src/dst port
```

### Softirq Storms (ksoftirqd at 100%)

```bash
# Identify softirq load
top
# Look for ksoftirqd/N at high CPU%
# Or high %si in CPU line

# Detailed softirq breakdown
cat /proc/softirqs
#                    CPU0       CPU1       CPU2       CPU3
#          HI:          0          0          0          0
#       TIMER:   12345678   12345678   12345678   12345678
#      NET_TX:       1234       5678       9012       3456
#      NET_RX:  567890123  567890123  567890123  567890123  <- Network heavy
#       BLOCK:     123456     123456     123456     123456
#    IRQ_POLL:          0          0          0          0
#     TASKLET:          0          0          0          0
#       SCHED:    1234567    1234567    1234567    1234567
#     HRTIMER:          0          0          0          0
#         RCU:   12345678   12345678   12345678   12345678

# Watch rate of change
watch -d 'cat /proc/softirqs'

# If NET_RX dominates: network packet storm
# If TIMER dominates: too many timers (check application)
# If RCU dominates: RCU callback backlog
```

**Diagnosing NET_RX softirq storms**:
```bash
# Check softnet stats
cat /proc/net/softnet_stat
# Column meanings (per CPU):
# 1: packets processed
# 2: packets dropped (buffer full)
# 3: time_squeeze (ran out of budget)

# Columns 2,3 non-zero = problem
# Column 2: increase buffer
# Column 3: increase softirq budget

# Increase softirq budget
sysctl net.core.netdev_budget=600        # Default 300
sysctl net.core.netdev_budget_usecs=4000 # Default 2000

# Increase receive buffer
sysctl net.core.netdev_max_backlog=10000 # Default 1000
```

**RCU callback storms**:
```bash
# Check RCU callback queue
cat /sys/kernel/debug/rcu/rcu_preempt/rcudata
# Look for high cblist length

# If using nohz_full, ensure rcu_nocbs is set
cat /proc/cmdline | grep rcu_nocbs

# Monitor RCU grace periods
cat /sys/kernel/debug/rcu/rcu_preempt/rcudata
```

### Interrupt Coalescing: Latency vs Throughput

```bash
# View current coalescing settings
ethtool -c eth0
# rx-usecs: 50
# rx-frames: 0
# rx-usecs-irq: 0
# tx-usecs: 50

# Disable coalescing for lowest latency
ethtool -C eth0 rx-usecs 0 rx-frames 1
ethtool -C eth0 tx-usecs 0 tx-frames 1

# Enable adaptive coalescing (driver adjusts automatically)
ethtool -C eth0 adaptive-rx on adaptive-tx on

# High throughput settings (batching)
ethtool -C eth0 rx-usecs 100 rx-frames 64
ethtool -C eth0 tx-usecs 100 tx-frames 64

# Check for packet drops indicating too much coalescing
ethtool -S eth0 | grep -i drop
```

**Coalescing decision matrix**:
| Workload | rx-usecs | rx-frames | Notes |
|----------|----------|-----------|-------|
| Low latency trading | 0 | 1 | Every packet fires IRQ |
| Interactive (web) | 20-50 | 8-16 | Balance |
| Bulk transfer | 100-250 | 64-128 | Batch for throughput |
| Adaptive | adaptive-rx on | - | Driver decides |

**Latency impact**:
- `rx-usecs=0`: ~5-10us interrupt latency
- `rx-usecs=50`: up to 50us delay
- `rx-usecs=250`: up to 250us delay

**Warning**: Low coalescing = more interrupts = higher CPU usage = potential DoS vulnerability.

### /proc/interrupts and /proc/softirqs Interpretation

```bash
# /proc/interrupts format:
#  IRQ#   CPU0    CPU1    ...   Type          Source
#    0:    123       0          IO-APIC  2-edge    timer
#   27: 123456  234567          PCI-MSI  524288-edge eth0-TxRx-0

# Interrupt types:
# IO-APIC-edge: Edge-triggered, single signal
# IO-APIC-level: Level-triggered, sustained signal
# PCI-MSI: Message Signaled Interrupts (modern, preferred)
# IR-PCI-MSI: Interrupt Remapping MSI (with IOMMU)

# Red flags in /proc/interrupts:
# 1. All interrupts on CPU0 (irqbalance not running)
# 2. One NIC queue handling 10x more than others
# 3. Extremely high interrupt rate (>100K/s per CPU)

# Analyze interrupt rate
while true; do
  cat /proc/interrupts | grep eth0 >> /tmp/irq.log
  sleep 1
done
# Then: awk to calculate delta per second

# Or use sar
sar -I ALL 1 10

# /proc/softirqs analysis
# Expected pattern: relatively even distribution across CPUs
# Red flags:
# 1. One CPU handling 90%+ of NET_RX (affinity problem)
# 2. NET_RX >> NET_TX (asymmetric load or receive-heavy)
# 3. RCU counts vastly different across CPUs

# Quick health check
awk 'NR==1 || /NET_RX|NET_TX|TIMER/' /proc/softirqs
```

**Interrupt to CPU mapping best practices**:
```bash
# 1. Pin NIC IRQs to same NUMA node as NIC
# 2. Pin application to same NUMA node
# 3. Keep housekeeping on CPU 0 (or dedicated housekeeping CPUs)
# 4. For low-latency: isolate CPUs from IRQs entirely

# Example: 8-core dual-socket
# Node 0: CPU 0-3, Node 1: CPU 4-7
# NIC on Node 0
# Strategy:
#   CPU 0: housekeeping, irqbalance
#   CPU 1-3: NIC IRQs + application
#   CPU 4-7: isolated for latency-critical threads
```

---

## CPU Isolation

### isolcpus, nohz_full, rcu_nocbs Usage

```bash
# Kernel boot parameters (GRUB_CMDLINE_LINUX)
# Basic isolation:
isolcpus=2-7

# Full isolation (recommended combination):
isolcpus=managed_irq,domain,2-7 nohz_full=2-7 rcu_nocbs=2-7

# Complete low-latency setup:
isolcpus=managed_irq,domain,2-7 nohz_full=2-7 rcu_nocbs=2-7 \
  irqaffinity=0-1 rcu_nocb_poll skew_tick=1 \
  intel_pstate=disable nosoftlockup

# Parameter breakdown:
# isolcpus=managed_irq,domain,2-7
#   managed_irq: kernel manages IRQ affinity (keeps off isolated)
#   domain: remove from scheduler load balancing domains
#   2-7: CPU list to isolate

# nohz_full=2-7
#   Disable scheduler tick when only one runnable task
#   Requires: CONFIG_NO_HZ_FULL=y

# rcu_nocbs=2-7
#   Offload RCU callbacks to kthreads on non-isolated CPUs
#   Automatically implied by nohz_full on recent kernels

# Apply changes
update-grub && reboot
```

**Additional isolation parameters**:
```bash
# irqaffinity=0-1
#   Default IRQ affinity (housekeeping CPUs only)

# rcu_nocb_poll
#   RCU callback kthreads poll instead of waking (reduces IPI)

# skew_tick=1
#   Offset timer ticks across CPUs (reduces lock contention)

# intel_pstate=disable
#   Use acpi-cpufreq (more deterministic than intel_pstate)

# nosoftlockup
#   Disable soft lockup detector (fires on isolated CPUs)

# nmi_watchdog=0
#   Disable NMI watchdog (periodic interrupt)
```

### Latency-Critical Workload Pinning

```bash
# Pin to isolated CPUs using taskset
taskset -c 2-7 ./latency_critical_app

# Pin with cgroups (cpuset)
mkdir /sys/fs/cgroup/cpuset/realtime
echo 2-7 > /sys/fs/cgroup/cpuset/realtime/cpuset.cpus
echo 0 > /sys/fs/cgroup/cpuset/realtime/cpuset.mems
echo $$ > /sys/fs/cgroup/cpuset/realtime/tasks
./latency_critical_app

# Pin with systemd
# /etc/systemd/system/myapp.service
[Service]
CPUAffinity=2 3 4 5 6 7
ExecStart=/usr/bin/myapp

# Docker
docker run --cpuset-cpus="2-7" myimage

# Kubernetes (requires CPU manager)
# kubelet: --cpu-manager-policy=static
resources:
  requests:
    cpu: "4"
  limits:
    cpu: "4"
# Pod gets dedicated CPUs if Guaranteed QoS

# Verify pinning
taskset -p <pid>
# pid XXXXX's current affinity mask: fc (binary: 11111100 = CPUs 2-7)
```

**Thread-level pinning** (for multi-threaded apps):
```bash
# Find all threads
ls /proc/<pid>/task/

# Pin each thread
for tid in /proc/<pid>/task/*; do
  taskset -p -c 2 $(basename $tid)
done

# Or in application code:
# pthread_setaffinity_np() in C
# os.sched_setaffinity() in Python
```

### Verification That Isolation Works

```bash
# 1. Check isolcpus effective
cat /sys/devices/system/cpu/isolated
# 2-7

# 2. Verify scheduler domains
cat /proc/schedstat | head -20
# Isolated CPUs should show minimal activity

# 3. Check no unexpected processes on isolated CPUs
ps -eo pid,comm,psr | awk '$3 >= 2 && $3 <= 7 {print}'
# Should only show your pinned processes

# 4. Monitor timer interrupts on isolated CPUs
watch -d 'cat /proc/interrupts | grep -E "LOC|RES"'
# LOC (Local timer) should be nearly static on isolated CPUs
# Expect ~1 tick every 1-2 seconds with nohz_full

# 5. Trace interrupts hitting isolated CPUs
perf record -e irq:irq_handler_entry -C 2-7 -- sleep 10
perf report
# Should be minimal

# 6. BPF tracing for interrupt latency
cat > /tmp/trace.bt << 'EOF'
tracepoint:irq:irq_handler_entry /cpu >= 2 && cpu <= 7/ {
  @[cpu, args->name] = count();
}
EOF
bpftrace /tmp/trace.bt

# 7. Measure actual jitter
cyclictest -m -p 80 -t 1 -a 2 -i 1000 -l 10000
# -m: mlockall
# -p 80: priority 80 (SCHED_FIFO)
# -t 1: 1 thread
# -a 2: pin to CPU 2
# -i 1000: 1ms interval
# -l 10000: 10000 loops

# Good result: Max latency < 50us
# Acceptable: Max latency < 100us
# Problem: Max latency > 1ms
```

**Common verification failures**:
```bash
# Problem: Still seeing timer ticks
# Check: Only ONE runnable task per isolated CPU
ps -eo psr,comm | sort | uniq -c | sort -rn

# Problem: RCU callbacks still running
# Check: rcu_nocbs parameter effective
cat /sys/kernel/debug/rcu/rcu_preempt/rcudata
# Isolated CPUs should show "nocb" flag

# Problem: Kernel threads on isolated CPUs
# Fix: kthread_cpus=0-1 boot parameter
cat /proc/cmdline | grep kthread_cpus

# Problem: IRQs hitting isolated CPUs
# Check: /proc/irq/*/smp_affinity
for irq in /proc/irq/*/smp_affinity; do
  mask=$(cat $irq 2>/dev/null)
  # Convert to binary, check if bits 2-7 set
done
```

### Housekeeping CPU Configuration

```bash
# Designate CPUs 0-1 as housekeeping
# Boot parameters:
GRUB_CMDLINE_LINUX="isolcpus=managed_irq,domain,2-7 \
  nohz_full=2-7 rcu_nocbs=2-7 irqaffinity=0-1 \
  kthread_cpus=0-1"

# Move kernel threads to housekeeping CPUs
for pid in $(pgrep -f '\[.*\]'); do
  taskset -p -c 0-1 $pid 2>/dev/null
done

# Move irqbalance to housekeeping
taskset -c 0-1 irqbalance

# Move all IRQs to housekeeping
for irq in /proc/irq/*/smp_affinity; do
  echo 3 > $irq 2>/dev/null  # 0x3 = CPUs 0,1
done

# Systemd: restrict system slice
mkdir -p /etc/systemd/system/system.slice.d/
cat > /etc/systemd/system/system.slice.d/cpuset.conf << 'EOF'
[Slice]
AllowedCPUs=0-1
EOF
systemctl daemon-reload

# Verify nothing unexpected on isolated CPUs
watch 'ps -eo pid,psr,comm | awk "\$2 >= 2"'
```

**Housekeeping workload placement**:
| Task | CPUs | Notes |
|------|------|-------|
| systemd, logging | 0-1 | System services |
| irqbalance | 0-1 | IRQ distribution |
| sshd, monitoring | 0-1 | Management traffic |
| Kernel threads | 0-1 | kworker, ksoftirqd |
| Network IRQs | 0-1 | Unless app processes packets |
| Application | 2-7 | Isolated, pinned |

---

## Context Switches & Migration

### Voluntary vs Involuntary Context Switches

```bash
# Definition:
# Voluntary: Task blocks (I/O, lock, sleep)
# Involuntary: Task preempted (time slice expired, higher priority)

# Per-process monitoring
pidstat -w 1 5
#      PID   cswch/s nvcswch/s  Command
#     1234    150.00     50.00  myapp
#     5678     20.00    200.00  cpu_intensive

# Process lifetime totals
cat /proc/<pid>/status | grep ctxt
# voluntary_ctxt_switches:	12345
# nonvoluntary_ctxt_switches:	6789

# perf stat for specific workload
perf stat -e context-switches ./myapp
# Or detailed:
perf stat -e 'sched:sched_switch' ./myapp

# Ratio interpretation:
# High voluntary, low involuntary = I/O bound (normal)
# Low voluntary, high involuntary = CPU bound, contention
# Both high = thread thrashing or over-synchronization
```

**Pattern recognition**:
```
Pattern 1: High voluntary, low involuntary
  cswch/s: 5000    nvcswch/s: 50
  Diagnosis: I/O bound or lock contention
  Action: Profile I/O, check lock patterns

Pattern 2: Low voluntary, high involuntary
  cswch/s: 100     nvcswch/s: 2000
  Diagnosis: CPU bound, too many threads
  Action: Reduce thread count, increase priority

Pattern 3: Both extremely high
  cswch/s: 10000   nvcswch/s: 10000
  Diagnosis: Thread thrashing, over-synchronization
  Action: Thread pool, batch work, reduce sync points

Pattern 4: Both low
  cswch/s: 10      nvcswch/s: 5
  Diagnosis: Healthy, efficient execution
  Action: None needed
```

**Reducing unnecessary context switches**:
```bash
# 1. Thread pools instead of spawn-per-task
# 2. Batch small I/O operations
# 3. Use async I/O (io_uring, epoll)
# 4. Reduce lock granularity
# 5. Use lock-free structures where possible
# 6. Pin threads to reduce cache invalidation
```

### CPU Migration Cost on NUMA

```bash
# Measure migrations
perf stat -e cpu-migrations ./myapp

# Watch migrations in real-time
perf sched record -a -- sleep 5
perf sched timehist -M

# Per-process migration count
cat /proc/<pid>/sched | grep nr_migrations

# Migration cost by type:
# Same core (HT): ~free (shared L1/L2)
# Same socket, different core: 50-100ns (L3 hit)
# Different socket: 100-300ns + potential cache flush

# NUMA migration impact
numastat -p <pid>
# numa_miss: pages accessed from remote node
# numa_foreign: pages intended for this node, allocated elsewhere
```

**Measuring actual migration cost**:
```bash
# BPF: track migration latency
cat > /tmp/migrate.bt << 'EOF'
tracepoint:sched:sched_migrate_task {
  @migrations[args->pid] = count();
  @from_to[args->orig_cpu, args->dest_cpu] = count();
}

END {
  print(@migrations);
  print(@from_to);
}
EOF
bpftrace /tmp/migrate.bt

# Memory bandwidth test with/without migration
# Without migration (pinned):
numactl --cpunodebind=0 --membind=0 ./membw_test

# With migration (unpinned):
./membw_test

# Compare throughput
```

**Cross-NUMA migration red flags**:
```bash
# High remote memory access
perf stat -e node-loads,node-stores,node-load-misses ./myapp
# node-load-misses >> 0 indicates cross-node access

# Memory bandwidth saturation
perf stat -e LLC-load-misses,LLC-store-misses ./myapp

# Process moving between NUMA nodes
watch -n 1 'cat /proc/<pid>/sched | grep -E "nr_migrations|numa_faults"'
```

### Pinning vs Letting Scheduler Decide

**When to pin**:
```bash
# Pin when:
# 1. Latency-critical and need cache warmth
# 2. NUMA-aware application with local memory
# 3. Real-time requirements
# 4. Application understands its memory layout

# Example: Database with NUMA-aware memory allocator
numactl --cpunodebind=0 --membind=0 ./database

# Example: Network application pinned to NIC's NUMA node
nic_node=$(cat /sys/class/net/eth0/device/numa_node)
numactl --cpunodebind=$nic_node ./network_app
```

**When NOT to pin**:
```bash
# Don't pin when:
# 1. Many short-lived processes (let scheduler load balance)
# 2. Unknown/variable workload characteristics
# 3. Oversubscribed system (more threads than cores)
# 4. Application already NUMA-optimized internally

# The scheduler is usually smarter:
# - Tracks cache hotness
# - Balances load automatically
# - Adapts to changing conditions
```

**Hybrid approach**:
```bash
# Soft affinity: prefer but allow migration
# Linux lacks native soft affinity, but:

# 1. Use cpuset with multiple CPUs
mkdir /sys/fs/cgroup/cpuset/myapp
echo 0-3 > /sys/fs/cgroup/cpuset/myapp/cpuset.cpus
# Process can migrate within 0-3, but not outside

# 2. NUMA-aware allocation without CPU pinning
numactl --preferred=0 ./myapp
# Prefer node 0 memory, but allow CPU migration

# 3. Use scheduler hints
# SCHED_BATCH: hint that task is CPU-bound
chrt -b 0 ./myapp
# Scheduler will try to minimize migrations
```

**Monitoring pinning effectiveness**:
```bash
# Track if pinned process stays put
while true; do
  psr=$(ps -o psr= -p <pid>)
  echo "$(date +%s) CPU: $psr"
  sleep 0.1
done > /tmp/cpu_track.log

# Analyze
sort /tmp/cpu_track.log | uniq -c
# Should show single CPU if properly pinned

# Check for involuntary migration despite pinning
# (indicates pinning not working)
cat /proc/<pid>/status | grep nonvoluntary
```

---

## Quick Reference: Diagnostic Commands

### CPU Saturation
```bash
vmstat 1                          # r column > nproc = saturated
mpstat -P ALL 1                   # Per-CPU utilization
uptime                            # Load average / nproc
runqlat 10 1                      # Run queue latency distribution (BPF)
perf sched latency                # Per-task scheduler delays
```

### Context Switches
```bash
vmstat 1                          # cs column (system-wide)
pidstat -w 1                      # Per-process cswch/nvcswch
perf stat -e context-switches     # Specific workload
cat /proc/<pid>/status            # Lifetime totals
```

### Interrupts
```bash
cat /proc/interrupts              # IRQ distribution
cat /proc/softirqs                # Softirq counts
watch -d 'cat /proc/net/softnet_stat'  # Network softnet
sar -I ALL 1                      # Interrupt rate
mpstat -P ALL 1                   # %irq, %soft columns
```

### NUMA
```bash
numactl --hardware                # Topology
numastat -p <pid>                 # Per-process NUMA stats
perf stat -e node-loads           # Cross-node memory
cat /sys/class/net/*/device/numa_node  # NIC NUMA affinity
```

### Isolation Verification
```bash
cat /sys/devices/system/cpu/isolated  # Isolated CPUs
cyclictest -m -p 80 -t 1 -a <cpu>     # Latency test
cat /proc/interrupts | grep LOC       # Timer interrupts
ps -eo pid,psr,comm                    # Process placement
```

---

## Decision Trees

### "My application has latency spikes"

```
Is CPU saturated? (vmstat r > nproc)
├─ Yes → Scale out or optimize
└─ No
   ├─ Check context switches (pidstat -w)
   │  ├─ High involuntary → Too many threads, reduce or pin
   │  └─ High voluntary → I/O or lock contention
   ├─ Check scheduler latency (perf sched latency)
   │  └─ Max delay > 10ms → Priority/isolation issue
   ├─ Check interrupt distribution (/proc/interrupts)
   │  └─ Unbalanced → Configure IRQ affinity
   └─ Check CFS throttling (cpu.stat)
      └─ throttled > 0 → Increase quota or period
```

### "When should I use CPU isolation?"

```
Latency requirement < 100us?
├─ Yes → Full isolation: isolcpus + nohz_full + rcu_nocbs
└─ No
   ├─ Latency requirement < 1ms?
   │  ├─ Yes → isolcpus + careful IRQ placement
   │  └─ No → Standard pinning (taskset/cgroups)
   └─ Interactive workload?
      ├─ Yes → EEVDF default is usually fine
      └─ No → SCHED_BATCH for throughput
```

### "Which scheduling policy should I use?"

```
Hard real-time with known WCET?
├─ Yes → SCHED_DEADLINE
└─ No
   ├─ Soft real-time, need deterministic latency?
   │  ├─ Yes → SCHED_FIFO (single path) or SCHED_RR (multiple)
   │  └─ No → Continue
   ├─ Interactive/mixed workload?
   │  ├─ Yes → SCHED_NORMAL (default EEVDF)
   │  └─ No → Continue
   ├─ CPU-intensive batch job?
   │  ├─ Yes → SCHED_BATCH
   │  └─ No → Continue
   └─ Background maintenance?
      └─ Yes → SCHED_IDLE
```

---

## Thresholds Summary

| Metric | Healthy | Warning | Critical |
|--------|---------|---------|----------|
| Run queue (r/nproc) | < 1 | 1-2 | > 2 |
| Load average (/nproc) | < 0.7 | 0.7-1.5 | > 2 |
| Context switches/s | < 10K | 10K-50K | > 100K |
| Scheduler delay (max) | < 1ms | 1-10ms | > 100ms |
| CFS throttle ratio | 0% | 1-10% | > 25% |
| Softirq %CPU | < 5% | 5-20% | > 30% |
| Cross-NUMA migrations | < 100/s | 100-1000/s | > 1000/s |
| Timer interrupts (isolated CPU) | ~1/s | 10/s | > 100/s |

---

## Further Reading

- [EEVDF Scheduler - Kernel Docs](https://docs.kernel.org/scheduler/sched-eevdf.html)
- [Completing the EEVDF scheduler - LWN](https://lwn.net/Articles/969062/)
- [CFS Bandwidth Control - Kernel Docs](https://docs.kernel.org/scheduler/sched-bwc.html)
- [The container throttling problem - Dan Luu](https://danluu.com/cgroup-throttling/)
- [perf sched for Linux CPU scheduler analysis - Brendan Gregg](https://www.brendangregg.com/blog/2017-03-16/perf-sched.html)
- [Linux Run Queue Latency - Brendan Gregg](https://www.brendangregg.com/blog/2016-10-08/linux-bcc-runqlat.html)
- [Low Latency Tuning Guide - Erik Rigtorp](https://rigtorp.se/low-latency-guide/)
- [CPU Isolation with isolcpus, nohz_full, rcu_nocbs - Red Hat](https://access.redhat.com/articles/3720611)
- [IRQ Affinity and Network Performance - Red Hat](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/network_troubleshooting_and_performance_tuning/tuning-irq-balancing)
- [Monitoring and Tuning the Linux Networking Stack - Packagecloud](https://blog.packagecloud.io/monitoring-tuning-linux-networking-stack-receiving-data/)
- [SCHED_DEADLINE - Kernel Docs](https://docs.kernel.org/scheduler/sched-deadline.html)
- [sched_ext Schedulers - GitHub](https://github.com/sched-ext/scx)
- [Interrupt Coalescing - Jakub Kicinski](https://people.kernel.org/kuba/easy-network-performance-wins-with-irq-coalescing)
- [Understanding Interrupts and Softirqs - Netdata](https://www.netdata.cloud/blog/understanding-interrupts-softirqs-and-softnet-in-linux/)

---

## See Also

- [eBPF & Tracing](06-ebpf-tracing.md) - runqlat, sched_ext custom schedulers, CPU tracing
- [Containers & K8s](07-containers-k8s.md) - CFS bandwidth throttling detection, container CPU limits
- [Kernel Tuning](08-kernel-tuning.md) - EEVDF tuning, sched_* sysctls, CPU governor
- [Memory Subsystem](15-memory-subsystem.md) - NUMA memory placement, migration costs
- [Latency Analysis](13-latency-analysis.md) - Scheduling latency impact on tail latency
