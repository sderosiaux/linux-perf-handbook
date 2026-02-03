# Memory Subsystem Deep Dive

Reference for diagnosing, interpreting, and fixing memory performance issues on Linux systems.

---

## 1. Page Faults and TLB

### Concepts

- **Minor fault (soft)**: Page exists in RAM but not mapped to process. Fast resolution (~1μs).
- **Major fault (hard)**: Page must be loaded from disk. Slow (~1-10ms for SSD, 10-100ms for HDD).
- **TLB (Translation Lookaside Buffer)**: CPU cache for virtual-to-physical address mappings. Limited entries (tens to hundreds).
- **TLB miss**: Requires page table walk. Cost: 10-100 cycles.
- **TLB shootdown**: Inter-processor interrupt to invalidate stale TLB entries. Causes latency spikes.

### Diagnosis Commands

**Check page fault rates:**
```bash
# System-wide from /proc/vmstat
awk '/pgfault|pgmajfault/ {print $1, $2}' /proc/vmstat

# Per-process
ps -o pid,min_flt,maj_flt,cmd -p <PID>

# Real-time monitoring
watch -n1 "awk '/pgfault|pgmajfault/ {print \$1, \$2}' /proc/vmstat"
```

**TLB statistics with perf:**
```bash
# TLB miss rates
perf stat -e dTLB-loads,dTLB-load-misses,iTLB-loads,iTLB-load-misses -p <PID> -- sleep 5

# Sample output interpretation:
# dTLB-load-misses / dTLB-loads = miss ratio
# > 1%: Consider huge pages
# > 5%: Significant TLB pressure
```

**Page fault tracing:**
```bash
# Trace major faults with stack
perf record -e page-faults -g -p <PID> -- sleep 10
perf report

# BPF-based tracing (more efficient)
bpftrace -e 'tracepoint:exceptions:page_fault_user { @[comm] = count(); }'
```

### Interpretation Thresholds

| Metric | Normal | Warning | Critical |
|--------|--------|---------|----------|
| Major faults/sec | < 10 | 10-100 | > 100 |
| Minor faults/sec | < 10000 | varies | context-dependent |
| dTLB miss ratio | < 1% | 1-5% | > 5% |
| iTLB miss ratio | < 0.5% | 0.5-2% | > 2% |

### Decision Tree

```
High major faults (pgmajfault > 100/sec)?
├─ YES → Check swap usage: `free -m`
│   ├─ Swap used heavily → Memory shortage (see Section 3)
│   └─ Swap minimal → mmap'd files being paged in
│       └─ Run: `perf record -e major-faults -g`
│       └─ Check file I/O patterns
└─ NO → Check minor faults
    ├─ Very high (> 100k/sec) → Possible memory churn
    │   └─ Profile allocations: `perf record -e page-faults`
    └─ Normal → Not a page fault issue
```

### Fixes

**Reduce TLB pressure:**
```bash
# Enable huge pages for specific app (2MB pages)
echo 1024 > /proc/sys/vm/nr_hugepages  # Reserve 1024 2MB pages

# Verify
grep Huge /proc/meminfo

# App must use mmap with MAP_HUGETLB or link with libhugetlbfs
```

**Prevent TLB shootdowns:**
```bash
# Causes of shootdowns: THP compaction, memory migration, KSM
# Check if THP is causing issues (see Section 6)
cat /sys/kernel/mm/transparent_hugepage/enabled

# Disable KSM if causing shootdowns
echo 0 > /sys/kernel/mm/ksm/run
```

**Lock critical process memory:**
```bash
# In application code: mlockall(MCL_CURRENT | MCL_FUTURE)
# Or use memlock ulimit
ulimit -l unlimited
```

---

## 2. Memory Allocator Internals

### SLAB/SLUB Concepts

- **Slab allocator**: Caches frequently-used kernel objects to avoid fragmentation.
- **SLUB**: Default modern allocator. Stores free pointer inside objects (100% utilization).
- **kmem cache**: Pool of pre-allocated objects of same size.
- **Fragmentation**: External (free chunks scattered) vs internal (wasted space in chunks).

### Diagnosis Commands

**Slab memory usage:**
```bash
# Summary
cat /proc/meminfo | grep -E "^(Slab|SReclaimable|SUnreclaim)"

# Example output:
# Slab:            500000 kB   ← Total slab memory
# SReclaimable:    300000 kB   ← Can be freed under pressure
# SUnreclaim:      200000 kB   ← Cannot be freed (problem if growing)

# Detailed per-cache stats (sorted by size)
slabtop -o -s c | head -20

# Alternative: /proc/slabinfo
cat /proc/slabinfo | sort -k4 -n -r | head -20
```

**Detect slab growth/leaks:**
```bash
# Watch over time
watch -n5 'cat /proc/meminfo | grep Slab'

# Track specific caches
watch -n5 'cat /proc/slabinfo | grep -E "dentry|inode|task_struct"'

# If dentry/inode growing: possible file handle leak
# If task_struct growing: fork bomb or process leak
```

**Memory fragmentation:**
```bash
# Buddy allocator state
cat /proc/buddyinfo
# Example: Node 0, zone Normal 123 456 234 89 34 12 5 2 1 0 0
# Numbers = free chunks of order 0,1,2...10 (4KB, 8KB, 16KB...4MB)
# LOW numbers at high orders = fragmentation

# Fragmentation index
cat /sys/kernel/debug/extfrag/extfrag_index

# Page type distribution
cat /proc/pagetypeinfo
```

### Interpretation

**Slab issues:**
```
SUnreclaim growing continuously?
├─ YES → Kernel memory leak
│   ├─ Check top consumers: slabtop
│   ├─ Common culprits: dentry, inode_cache, kmalloc-*
│   └─ May need kernel upgrade or parameter tuning
└─ NO → Normal behavior

Slab > 10% of MemTotal?
├─ YES → Investigate largest caches
│   └─ dentry/inode high: many files opened, consider vfs_cache_pressure
└─ NO → Typically fine
```

**Fragmentation issues:**
```
/proc/buddyinfo shows 0s in high orders (8+)?
├─ YES → High fragmentation
│   ├─ Check if THP enabled (causes compaction)
│   ├─ Consider: echo 1 > /proc/sys/vm/compact_memory
│   └─ Long-term: increase vm.min_free_kbytes
└─ NO → Fragmentation acceptable
```

### Fixes

**Reduce slab memory:**
```bash
# Increase cache pressure (default=100, higher=more aggressive reclaim)
sysctl vm.vfs_cache_pressure=150

# Drop caches (temporary, use with caution in prod)
sync && echo 3 > /proc/sys/vm/drop_caches
# 1=page cache, 2=dentries/inodes, 3=both
```

**Debug slab with kernel options:**
```bash
# Boot-time debugging (impacts performance)
# Add to kernel cmdline: slab_debug=FZP

# Per-cache debugging
echo "slab_debug=FZP" > /sys/kernel/slab/<cache>/debug

# Debug flags:
# F = consistency checks
# Z = red zoning (detect overflows)
# P = poisoning (detect use-after-free)
# U = user tracking (allocation traces)
```

**Fix fragmentation:**
```bash
# Increase minimum free memory (reserves more for high-order allocs)
# Default varies; increase by 50-100%
sysctl vm.min_free_kbytes=131072

# Trigger manual compaction
echo 1 > /proc/sys/vm/compact_memory

# Reduce NUMA fragmentation (if applicable)
# Consider disabling NUMA for small nodes
```

---

## 3. Page Reclaim and Swap

### Concepts

- **kswapd**: Background daemon that reclaims pages when free memory is low.
- **Direct reclaim**: Foreground reclaim blocking the allocating process. Causes latency.
- **Watermarks**: min (OOM threshold), low (wake kswapd), high (kswapd stops).
- **Anonymous pages**: Process heap/stack memory. Swappable.
- **File pages**: Page cache. Droppable or writeback if dirty.

### Diagnosis Commands

**Memory pressure check:**
```bash
# Quick overview
free -h

# Key metric: MemAvailable (not MemFree)
awk '/MemAvailable/ {avail=$2} /MemTotal/ {total=$2} END {printf "Available: %.1f%%\n", avail/total*100}' /proc/meminfo

# PSI (Pressure Stall Information) - requires kernel 4.20+
cat /proc/pressure/memory
# some avg10=0.00 avg60=0.00 avg300=0.00 total=0
# full avg10=0.00 avg60=0.00 avg300=0.00 total=0

# Interpretation:
# avg10 > 5%: Mild pressure
# avg10 > 20%: Significant pressure
# avg10 > 50%: Severe, likely impacting latency
# "full" > 0: All CPUs stalled waiting for memory
```

**Reclaim activity:**
```bash
# vmstat columns: si=swap-in, so=swap-out, r=runnable, b=blocked
vmstat 1
# Look for: so > 0 consistently (swap pressure)
#           b > 0 consistently (processes blocked)

# Detailed reclaim stats
awk '/pgscan|pgsteal|kswapd|direct/ {print $1, $2}' /proc/vmstat

# Key metrics:
# pgscan_direct > 0: Direct reclaim happening (bad for latency)
# pgsteal_kswapd vs pgsteal_direct: Ratio of background vs foreground reclaim
```

**Swap analysis:**
```bash
# Swap usage by process
for f in /proc/*/status; do
  awk '/^(Name|VmSwap)/{printf "%s ", $2}' "$f" 2>/dev/null && echo
done | sort -k2 -n | tail -10

# Swap I/O rate
vmstat 1 | awk '{print $7, $8}'  # si, so columns

# Swap device performance
iostat -x 1 | grep -E "Device|swap"
```

### Interpretation Thresholds

| Metric | Normal | Warning | Critical |
|--------|--------|---------|----------|
| MemAvailable | > 20% | 10-20% | < 10% |
| PSI memory avg10 | < 5% | 5-20% | > 20% |
| Swap out (so) | 0 | < 100 MB/s | > 100 MB/s |
| pgscan_direct | 0 | < 1000/sec | > 1000/sec |

### Swappiness

**What it actually controls:**
```bash
# Current value (default: 60)
cat /proc/sys/vm/swappiness

# Range: 0-200
# 0: Avoid swapping, prefer reclaiming file pages
# 60: Balance between anonymous and file pages
# 100: Treat anonymous and file pages equally
# >100: Prefer swapping over file page reclaim
```

**Tuning guidelines:**
```bash
# Databases, latency-sensitive apps: Minimize swapping
sysctl vm.swappiness=10

# Memory-constrained system: Allow swap
sysctl vm.swappiness=60

# Note: swappiness=0 can trigger OOM instead of swap
# Use swappiness=1 for "avoid swap but don't OOM"
```

### Decision Tree

```
MemAvailable < 10%?
├─ YES → Memory pressure
│   ├─ Check top consumers: ps aux --sort=-%mem | head -10
│   ├─ Single process > 50% RAM?
│   │   ├─ Expected: Tune app (heap size, caching)
│   │   └─ Unexpected: Memory leak, restart or debug
│   ├─ Slab high? → See Section 2
│   └─ Many processes → Scale horizontally or add RAM
└─ NO → Check PSI

PSI memory avg10 > 10%?
├─ YES → Latency from reclaim
│   ├─ Direct reclaim happening (pgscan_direct > 0)?
│   │   └─ Increase vm.min_free_kbytes to keep kswapd ahead
│   ├─ Swap I/O? → Add swap or use faster storage
│   └─ Consider vm.watermark_scale_factor increase
└─ NO → Memory subsystem healthy
```

### Fixes

**Reduce reclaim latency:**
```bash
# Wake kswapd earlier (default: 10, range 1-1000)
sysctl vm.watermark_scale_factor=200

# Keep more memory free (default varies, ~1% of RAM)
sysctl vm.min_free_kbytes=262144

# Reduce direct reclaim
sysctl vm.watermark_boost_factor=0
```

**Swap configuration:**
```bash
# Check current swap priority
swapon --show

# Add faster swap (SSD) with higher priority
mkswap /dev/nvme0n1p2
swapon -p 100 /dev/nvme0n1p2

# Use zram for compression (reduces swap I/O)
modprobe zram
echo lz4 > /sys/block/zram0/comp_algorithm
echo 4G > /sys/block/zram0/disksize
mkswap /dev/zram0
swapon -p 200 /dev/zram0
```

**Application-level mitigation:**
```bash
# Prevent specific process from swapping
echo -17 > /proc/<PID>/oom_score_adj  # Discourage OOM kill
# Note: can't fully prevent swap, but reduces priority

# Use mlock() in application code for critical memory
```

---

## 4. OOM Behavior

### Concepts

- **OOM killer**: Terminates processes when memory is exhausted and cannot be reclaimed.
- **oom_score**: Badness score (0-1000). Higher = more likely to be killed.
- **oom_score_adj**: User-adjustable modifier (-1000 to +1000).
- **cgroup OOM**: Container-level OOM, separate from host OOM.

### Diagnosis Commands

**Check current OOM scores:**
```bash
# All processes sorted by oom_score
for pid in /proc/[0-9]*; do
  [ -f "$pid/oom_score" ] && echo "$(cat $pid/oom_score) $(cat $pid/cmdline 2>/dev/null | tr '\0' ' ')"
done | sort -rn | head -20

# Specific process
cat /proc/<PID>/oom_score
cat /proc/<PID>/oom_score_adj
```

**Detect OOM events:**
```bash
# dmesg for OOM messages
dmesg -T | grep -i "out of memory\|oom-killer\|killed process"

# Example OOM log pattern:
# [timestamp] Out of memory: Kill process <PID> (<name>) score <score>

# Journalctl (systemd)
journalctl -k | grep -i oom

# cgroup OOM (container)
# Kernel message: "Memory cgroup out of memory: Kill process..."
```

**Container OOM detection:**
```bash
# cgroup v2: Check OOM events
cat /sys/fs/cgroup/<container-path>/memory.events
# oom: Number of OOM kills in this cgroup
# oom_kill: Same, more recent kernels

# cgroup v1
cat /sys/fs/cgroup/memory/<container-path>/memory.oom_control

# Kubernetes: Check pod status
kubectl describe pod <name> | grep -A5 "State\|Last State"
# Reason: OOMKilled, Exit Code: 137
```

### OOM Score Calculation

```
Base score = (process_memory / total_memory) * 1000
Adjustments:
  - Root processes: -30 points (3% bonus)
  - oom_score_adj: Added directly
  - oom_score_adj = -1000: Never kill (oom_score = 0)
  - oom_score_adj = +1000: Always kill first (oom_score = 1000)
```

### Decision Tree

```
OOM killed?
├─ Check scope: cgroup or host?
│   ├─ cgroup message → Container limit reached
│   │   ├─ Increase container memory limit
│   │   ├─ Fix application memory leak
│   │   └─ Scale horizontally
│   └─ Host message → System-wide memory exhaustion
│       ├─ Identify memory hogs
│       └─ Add RAM or tune applications
├─ Wrong process killed?
│   └─ Adjust oom_score_adj for critical processes
└─ Prevention needed?
    ├─ Early warning: Monitor MemAvailable, set alerts
    └─ Controlled degradation: Use memory.high in cgroups
```

### Fixes

**Protect critical processes:**
```bash
# Reduce OOM score (less likely to be killed)
echo -500 > /proc/<PID>/oom_score_adj

# Never kill (use sparingly, can cause system hang)
echo -1000 > /proc/<PID>/oom_score_adj

# Systemd service protection
# In unit file:
# [Service]
# OOMScoreAdjust=-500
```

**Target specific processes:**
```bash
# Increase OOM score for less critical processes
echo 500 > /proc/<PID>/oom_score_adj

# Kill this first if OOM
echo 1000 > /proc/<PID>/oom_score_adj
```

**Container memory management (cgroup v2):**
```bash
# View current limits
cat /sys/fs/cgroup/<path>/memory.max      # Hard limit (OOM trigger)
cat /sys/fs/cgroup/<path>/memory.high     # Soft limit (throttling)
cat /sys/fs/cgroup/<path>/memory.current  # Current usage

# Set soft limit at 90% of hard limit to enable throttling before OOM
echo $((LIMIT * 90 / 100)) > /sys/fs/cgroup/<path>/memory.high

# Monitor pressure
cat /sys/fs/cgroup/<path>/memory.pressure
```

**cgroup v1 vs v2 OOM behavior:**
```
cgroup v1:
  - May kill any child process (unpredictable)
  - Kubernetes may not detect partial kills

cgroup v2:
  - Kills entire cgroup (all or nothing)
  - More predictable, visible to orchestrators
  - singleProcessOOMKill flag for v1-like behavior
```

**Prevent OOM with early action:**
```bash
# earlyoom: Userspace OOM killer (acts before kernel OOM)
# Install: apt/yum install earlyoom
# Kills processes when MemAvailable < 10%

# Configure in /etc/default/earlyoom:
# EARLYOOM_ARGS="-r 60 -m 5 -s 5"
# -m 5: Kill when <5% memory available
# -s 5: Kill when <5% swap available
```

---

## 5. NUMA Effects

### Concepts

- **NUMA (Non-Uniform Memory Access)**: Memory attached to specific CPU sockets.
- **Local access**: Memory on same node as CPU. Fast.
- **Remote access**: Memory on different node. 50-100%+ slower.
- **Node distance**: Relative cost (local=10, remote=20+ typically).

### Diagnosis Commands

**Check if NUMA system:**
```bash
# Node count
numactl --hardware
# If only 1 node: NUMA not relevant

# Node distances
cat /sys/devices/system/node/node*/distance
# Example: 10 21 → Local=10, remote=21 (2.1x slower)
```

**NUMA memory allocation stats:**
```bash
# System-wide
numastat
# Key metrics:
# numa_hit: Pages allocated on intended node (good)
# numa_miss: Pages allocated elsewhere (bad)
# numa_foreign: Pages intended here, allocated elsewhere
# local_node: Pages allocated for local CPU
# other_node: Pages allocated for remote CPU

# Per-process
numastat -p <PID>

# Compact view (many nodes)
numastat -c -z

# Real-time monitoring
watch -n1 numastat
```

**NUMA memory access latency:**
```bash
# Using perf mem
perf mem record -a -- sleep 10
perf mem report --sort=mem

# NUMA benchmark
perf bench numa mem -a

# Check if processes cross NUMA boundaries
ps -eo pid,psr,comm | while read pid psr comm; do
  node=$(cat /sys/devices/system/cpu/cpu$psr/node* 2>/dev/null | head -1)
  echo "$pid CPU:$psr Node:$node $comm"
done
```

### Interpretation

**NUMA problem indicators:**
```bash
# Calculate miss ratio
numastat | awk '/numa_miss/ {miss=$2} /numa_hit/ {hit=$2} END {printf "Miss ratio: %.2f%%\n", miss/(hit+miss)*100}'

# Thresholds:
# < 1%: Excellent NUMA locality
# 1-5%: Acceptable
# 5-10%: Worth investigating
# > 10%: Significant performance impact
```

**Memory distribution check:**
```bash
# Memory per node
numastat -m
# Look for imbalanced "MemUsed" across nodes
# Large difference = poor distribution
```

### Decision Tree

```
Multi-node NUMA system?
├─ NO → Skip NUMA tuning
└─ YES → Check numa_miss ratio
    ├─ < 5% → NUMA working well
    └─ > 5% → Investigate
        ├─ Single large process?
        │   └─ Pin to single node: numactl --cpunodebind=0 --membind=0 <cmd>
        ├─ Multiple processes?
        │   └─ Distribute across nodes with numactl
        └─ Database/JVM?
            └─ Configure app-specific NUMA settings
```

### Fixes

**Pin process to NUMA node:**
```bash
# Run new process on node 0
numactl --cpunodebind=0 --membind=0 <command>

# Interleave memory across all nodes (good for some workloads)
numactl --interleave=all <command>

# Preferred node (allows overflow to others)
numactl --preferred=0 <command>
```

**Move existing process:**
```bash
# Migrate process memory to node 0
migratepages <PID> 0,1 0

# Note: May cause brief pause, memory is copied
```

**Systemd service NUMA binding:**
```ini
[Service]
# Bind to nodes 0 and 1
NumaPolicy=bind
NumaMask=0-1

# Or interleave
NumaPolicy=interleave
NumaMask=all
```

**Application-specific:**
```bash
# MySQL/MariaDB: numactl --interleave=all mysqld
# Or in my.cnf: innodb_numa_interleave=ON

# PostgreSQL: numactl --interleave=all postgres

# JVM: -XX:+UseNUMA

# Redis: numactl --cpunodebind=0 --membind=0 redis-server
```

**Kernel NUMA balancing:**
```bash
# Check if automatic NUMA balancing enabled
cat /proc/sys/kernel/numa_balancing
# 0=disabled, 1=enabled

# Disable if causing latency spikes (scans cause overhead)
sysctl kernel.numa_balancing=0

# Enable for workloads with changing locality
sysctl kernel.numa_balancing=1
```

---

## 6. Transparent Huge Pages

### Concepts

- **THP**: Kernel automatically uses 2MB pages instead of 4KB.
- **Benefit**: Fewer page faults, fewer TLB misses.
- **Cost**: Compaction overhead, memory fragmentation, latency spikes.
- **Compaction**: Process of moving pages to create contiguous 2MB blocks.

### Diagnosis Commands

**Current THP status:**
```bash
# THP mode
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never
# always: THP enabled system-wide
# madvise: Only for apps requesting via madvise()
# never: THP disabled

# Defrag mode
cat /sys/kernel/mm/transparent_hugepage/defrag
# [always] defer defer+madvise madvise never
```

**THP usage:**
```bash
# System-wide
grep -E "^(AnonHuge|Huge)" /proc/meminfo
# AnonHugePages: Amount of memory in THP

# Per-process
grep -E "AnonHuge|THP" /proc/<PID>/smaps_rollup
```

**Compaction activity (KEY METRIC):**
```bash
# Compaction stalls - direct indicator of THP latency impact
grep -E "compact_stall|compact_success|compact_fail" /proc/vmstat

# compact_stall: Times process blocked for compaction (BAD if high)
# compact_success: Successful compactions
# compact_fail: Failed compactions

# Monitor over time
watch -n1 'grep compact_stall /proc/vmstat'
```

**Detect THP-caused latency:**
```bash
# High sys CPU during compaction
top  # Look for high %sy during memory pressure

# Stack trace during stall
perf record -e sched:sched_switch -g -a -- sleep 10
perf report
# Look for: do_huge_pmd_anonymous_page, compact_zone, migration
```

### Interpretation

**When THP hurts (common cases):**
```
- Databases (MongoDB, Redis, MySQL, PostgreSQL)
- JVM applications with large heaps
- Applications with sparse memory access patterns
- Latency-sensitive services
- Systems under memory pressure
```

**When THP helps:**
```
- Scientific computing with sequential access
- Large matrix operations
- Applications explicitly designed for huge pages
- Memory-bound, throughput-focused workloads
```

**Symptoms of THP problems:**
```
- Sporadic latency spikes (P99 >> P50)
- High sys CPU in top during memory operations
- compact_stall growing in /proc/vmstat
- High khugepaged CPU usage
- Flame graphs showing compaction functions
```

### Decision Tree

```
Latency spikes correlating with memory operations?
├─ YES → Check THP
│   ├─ THP enabled? cat /sys/kernel/mm/transparent_hugepage/enabled
│   │   └─ YES → Check compact_stall
│   │       ├─ Growing → THP likely causing stalls
│   │       │   └─ Disable THP or use madvise mode
│   │       └─ Stable → THP probably not the issue
│   └─ NO → Not THP-related
└─ NO → THP not causing latency issues

Database workload?
├─ YES → Disable THP (recommended by MongoDB, Redis, Oracle, etc.)
└─ NO → Test with and without THP, measure P99 latency
```

### Fixes

**Disable THP (runtime):**
```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

**Disable THP (persistent):**
```bash
# GRUB method
# Add to /etc/default/grub GRUB_CMDLINE_LINUX:
# transparent_hugepage=never
# Then: grub2-mkconfig -o /boot/grub2/grub.cfg

# Systemd tmpfiles method
# Create /etc/tmpfiles.d/transparent_hugepages.conf:
# w /sys/kernel/mm/transparent_hugepage/enabled - - - - never
# w /sys/kernel/mm/transparent_hugepage/defrag - - - - never
```

**Use madvise mode (compromise):**
```bash
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled
echo madvise > /sys/kernel/mm/transparent_hugepage/defrag
# Apps must call madvise(addr, len, MADV_HUGEPAGE) to opt-in
```

**Per-process control (prctl):**
```bash
# Disable THP for specific process (kernel 5.0+)
# In code: prctl(PR_SET_THP_DISABLE, 1, 0, 0, 0)

# Check process THP status
cat /proc/<PID>/status | grep THP
```

**Reduce compaction overhead without disabling THP:**
```bash
# Change defrag to defer (background compaction only)
echo defer > /sys/kernel/mm/transparent_hugepage/defrag

# Reduce khugepaged aggressiveness
echo 10000 > /sys/kernel/mm/transparent_hugepage/khugepaged/scan_sleep_millisecs
# Default: 10000 (10s between scans)
```

**Explicit huge pages (alternative to THP):**
```bash
# Reserve fixed huge pages (no compaction needed)
echo 1024 > /proc/sys/vm/nr_hugepages

# Check
grep Huge /proc/meminfo

# Mount hugetlbfs for apps
mount -t hugetlbfs nodev /mnt/huge

# Use with: mmap(..., MAP_HUGETLB, ...)
# Or configure app (e.g., QEMU, DPDK, Oracle)
```

---

## 7. Quick Reference

### One-Liner Diagnostics

```bash
# Memory pressure quick check
awk '/MemAvailable/{a=$2}/MemTotal/{t=$2}END{printf "%.0f%% available\n",a/t*100}' /proc/meminfo

# Top memory consumers
ps aux --sort=-%mem | head -10

# Swap usage by process
awk '/VmSwap/{s+=$2}END{print s" kB total swap used"}' /proc/*/status 2>/dev/null

# Page fault rate
awk '/pgfault/{print "faults/sec:", $2}' /proc/vmstat

# OOM risk (high score processes)
for p in /proc/[0-9]*; do echo "$(cat $p/oom_score 2>/dev/null) $(cat $p/comm 2>/dev/null)"; done | sort -rn | head -5

# THP compaction stalls
awk '/compact_stall/{print "stalls:", $2}' /proc/vmstat

# NUMA miss ratio
numastat 2>/dev/null | awk '/numa_miss/{m=$2}/numa_hit/{h=$2}END{if(h+m>0)printf "%.2f%% miss\n",m/(h+m)*100}'

# Memory PSI pressure
cat /proc/pressure/memory 2>/dev/null | head -1
```

### Key Files

| File | Purpose |
|------|---------|
| `/proc/meminfo` | System memory summary |
| `/proc/vmstat` | VM subsystem statistics |
| `/proc/<PID>/status` | Process memory details |
| `/proc/<PID>/smaps` | Detailed memory mappings |
| `/proc/<PID>/oom_score` | Current OOM score |
| `/proc/<PID>/oom_score_adj` | OOM score adjustment |
| `/proc/buddyinfo` | Free memory fragmentation |
| `/proc/slabinfo` | Slab allocator details |
| `/proc/pressure/memory` | PSI memory pressure |
| `/sys/kernel/mm/transparent_hugepage/*` | THP settings |
| `/sys/fs/cgroup/*/memory.*` | cgroup memory controls |

### Key Tunables

| Tunable | Default | Typical Range | Effect |
|---------|---------|---------------|--------|
| `vm.swappiness` | 60 | 1-100 | Swap vs file cache reclaim |
| `vm.vfs_cache_pressure` | 100 | 50-200 | Dentry/inode cache reclaim |
| `vm.min_free_kbytes` | varies | 64K-256K | Reserved memory for allocations |
| `vm.watermark_scale_factor` | 10 | 10-200 | kswapd activation sensitivity |
| `vm.dirty_ratio` | 20 | 5-40 | Sync writeback threshold (%) |
| `vm.dirty_background_ratio` | 10 | 1-20 | Async writeback threshold (%) |
| `kernel.numa_balancing` | 1 | 0-1 | Automatic NUMA memory migration |

### Alert Thresholds

```yaml
# Prometheus/Grafana alerting rules pattern
MemoryAvailable:
  warning: < 20%
  critical: < 10%

MemoryPSI:
  warning: avg10 > 10%
  critical: avg10 > 30%

SwapUsage:
  warning: > 50%
  critical: > 80%

OOMKills:
  warning: > 0 in 5min
  critical: > 3 in 5min

MajorPageFaults:
  warning: > 100/sec
  critical: > 1000/sec

THPCompactionStalls:
  warning: > 10/sec increase
  critical: > 100/sec increase

NUMAMissRatio:
  warning: > 5%
  critical: > 15%
```

---

## 8. Troubleshooting Workflows

### Workflow: Application Running Slowly

```
1. Quick memory check
   $ free -h && cat /proc/pressure/memory

2. If MemAvailable < 10% or PSI > 10%:
   $ ps aux --sort=-%mem | head -10
   → Identify top consumer

3. Check for swap activity:
   $ vmstat 1 5
   → si/so columns > 0 = swap pressure

4. If specific process suspect:
   $ cat /proc/<PID>/status | grep -E "^(VmRSS|VmSwap|RssAnon)"
   → VmSwap high = process being swapped

5. Resolution:
   - Add RAM
   - Tune application memory settings
   - Restart leaking process
   - Reduce swappiness
```

### Workflow: Sporadic Latency Spikes

```
1. Check THP compaction:
   $ grep compact_stall /proc/vmstat
   → Monitor for growth during spikes

2. Check direct reclaim:
   $ grep pgscan_direct /proc/vmstat
   → Should be 0 or very low

3. Check NUMA (if multi-node):
   $ numastat | grep numa_miss
   → High miss = NUMA penalty

4. If THP compaction stalls growing:
   $ echo madvise > /sys/kernel/mm/transparent_hugepage/enabled

5. If direct reclaim happening:
   $ sysctl vm.min_free_kbytes=$((CURRENT * 2))
   $ sysctl vm.watermark_scale_factor=200
```

### Workflow: Container OOMKilled

```
1. Confirm OOM type:
   $ dmesg -T | grep -i oom | tail -5
   → "Memory cgroup" = container limit
   → "Out of memory" = host level

2. Check container memory limit:
   $ cat /sys/fs/cgroup/<path>/memory.max
   $ cat /sys/fs/cgroup/<path>/memory.current

3. Was it sudden or gradual?
   - Sudden: Traffic spike or batch job
   - Gradual: Memory leak

4. Check pressure before OOM:
   $ cat /sys/fs/cgroup/<path>/memory.events
   → high count = was being throttled

5. Resolution:
   - Increase memory.max
   - Set memory.high at 90% of max (throttle before kill)
   - Fix application memory leak
   - Add horizontal scaling
```

### Workflow: Kernel Memory Growing

```
1. Check slab usage:
   $ cat /proc/meminfo | grep -E "^Slab|^SUnreclaim"

2. Identify large slabs:
   $ slabtop -o -s c | head -15

3. Common patterns:
   - dentry/inode high: Too many files, increase vfs_cache_pressure
   - kmalloc-* growing: Possible kernel leak, check for patches
   - tcp_* growing: Connection leak, check netstat

4. Temporary relief:
   $ sync && echo 2 > /proc/sys/vm/drop_caches
   → Reclaims dentries/inodes

5. Long-term:
   $ sysctl vm.vfs_cache_pressure=150
   → Increase pressure on inode/dentry caches
```

---

## 9. Memory Profiling with perf

### Page Fault Profiling

```bash
# Record page faults with call stacks
perf record -e page-faults -g -p <PID> -- sleep 30
perf report --stdio

# Major faults only
perf record -e major-faults -g -p <PID> -- sleep 30
perf report --stdio

# Annotate source (if symbols available)
perf annotate <function_name>
```

### Memory Access Profiling

```bash
# Sample memory loads with latency
perf mem record -p <PID> -- sleep 30
perf mem report --sort=mem

# Output columns:
# Weight: Access latency in cycles
# Memory access: L1 hit, L2 hit, L3 hit, Local RAM, Remote RAM

# Filter by latency (>= 50 cycles)
perf mem record --ldlat=50 -p <PID> -- sleep 30
```

### Cache Miss Analysis

```bash
# L1 data cache misses
perf stat -e L1-dcache-loads,L1-dcache-load-misses -p <PID> -- sleep 10

# LLC (Last Level Cache) misses
perf stat -e LLC-loads,LLC-load-misses -p <PID> -- sleep 10

# Calculate miss ratios from output
```

### Memory Flame Graphs

```bash
# Record memory allocation stacks
perf record -e kmem:kmalloc -g -a -- sleep 30
# Or page faults
perf record -e page-faults -g -a -- sleep 30

# Generate flame graph
perf script > out.stacks
stackcollapse-perf.pl out.stacks > out.folded
flamegraph.pl out.folded > memory.svg
```

---

## 10. Container Memory Deep Dive

### cgroup v2 Memory Interface

```bash
# Key files in /sys/fs/cgroup/<container>/

memory.current      # Current usage in bytes
memory.max          # Hard limit (OOM trigger)
memory.high         # Soft limit (throttle trigger)
memory.low          # Memory protection (best-effort)
memory.min          # Memory protection (hard)
memory.swap.current # Current swap usage
memory.swap.max     # Swap limit
memory.pressure     # PSI metrics for this cgroup
memory.events       # Counter for OOM, high, max events
memory.stat         # Detailed statistics
```

### Reading Container Memory Stats

```bash
# Current memory usage
cat memory.current | numfmt --to=iec

# Memory breakdown
cat memory.stat | grep -E "^(anon|file|slab|sock|shmem|kernel)"

# Pressure
cat memory.pressure
# some avg10=0.45 avg60=0.23 avg300=0.12 total=123456789

# Event counters
cat memory.events
# low 0
# high 125      ← Times hit memory.high limit
# max 3         ← Times hit memory.max limit
# oom 1         ← OOM kills
# oom_kill 1
```

### Container Memory Troubleshooting

```bash
# Find container cgroup path (Docker)
docker inspect <container> --format '{{.HostConfig.CgroupParent}}'

# Or find by PID
cat /proc/<container_pid>/cgroup

# Check if being throttled
watch -n1 'cat /sys/fs/cgroup/<path>/memory.events'
# Look for "high" counter increasing

# Check actual limit vs usage
echo "Limit: $(cat memory.max | numfmt --to=iec)"
echo "Usage: $(cat memory.current | numfmt --to=iec)"
echo "Percent: $(awk -v u=$(cat memory.current) -v m=$(cat memory.max) 'BEGIN{printf "%.1f%%", u/m*100}')"
```

### Kubernetes Memory Debugging

```bash
# Get container memory limits
kubectl get pod <name> -o jsonpath='{.spec.containers[*].resources.limits.memory}'

# Check if OOMKilled
kubectl describe pod <name> | grep -A3 "Last State"

# Get memory metrics (requires metrics-server)
kubectl top pod <name>

# Exec into container for debugging
kubectl exec -it <pod> -- cat /sys/fs/cgroup/memory.current
```

---

## Sources

This chapter synthesizes information from:
- [Brendan Gregg's Linux perf Examples](https://www.brendangregg.com/perf.html)
- [Brendan Gregg: Analyzing High Rate of Paging](https://www.brendangregg.com/blog/2021-08-30/high-rate-of-paging.html)
- [Linux Kernel THP Documentation](https://docs.kernel.org/admin-guide/mm/transhuge.html)
- [Linux Kernel Memory Controller Documentation](https://docs.kernel.org/admin-guide/cgroup-v2.html)
- [Red Hat Memory Performance Documentation](https://access.redhat.com/solutions/46111)
- [Netdata cgroups Memory Diagnostics](https://www.netdata.cloud/academy/diagnosing-linux-cgroups/)
- [LinkedIn Engineering: Optimizing Linux Memory Management](https://engineering.linkedin.com/performance/optimizing-linux-memory-management-low-latency-high-throughput-databases)
- [Google Cloud GKE OOM Troubleshooting](https://cloud.google.com/kubernetes-engine/docs/troubleshooting/oom-events)
- [PingCAP: THP and Database Performance](https://www.pingcap.com/blog/transparent-huge-pages-why-we-disable-it-for-databases/)
- [Oracle Linux SLUB Documentation](https://blogs.oracle.com/linux/linux-slub-allocator-internals-and-debugging-1)
- [man7.org Linux Manual Pages](https://man7.org/linux/man-pages/)
- [Red Hat NUMA Performance Brief](https://access.redhat.com/sites/default/files/attachments/rhel7_numa_perf_brief.pdf)
- [LWN.net: Taming the OOM Killer](https://lwn.net/Articles/317814/)
- [systemd Memory Pressure Handling](https://systemd.io/MEMORY_PRESSURE/)
