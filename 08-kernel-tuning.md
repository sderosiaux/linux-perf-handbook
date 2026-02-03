# Kernel Tuning

sysctl parameters, CPU, memory, and I/O optimization.

## sysctl Basics

```bash
sysctl -a                      # Show all parameters
sysctl parameter               # Show specific
sysctl -w parameter=value      # Set value (runtime)
sysctl -p                      # Reload /etc/sysctl.conf
sysctl -p /etc/sysctl.d/99-custom.conf  # Reload specific file
```

Persistent settings: `/etc/sysctl.conf` or `/etc/sysctl.d/*.conf`

## CPU Scheduling

### CPU Governor

```bash
# Check current
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# Available governors
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors
# conservative ondemand userspace powersave performance schedutil

# Set performance (all CPUs)
echo performance | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# Using cpupower
cpupower frequency-set -g performance
```

| Governor | Use Case |
|----------|----------|
| `performance` | Maximum frequency, latency-sensitive |
| `powersave` | Minimum frequency, battery |
| `schedutil` | Default, utilization-based |
| `ondemand` | Ramp on demand |

### CPU Isolation

```bash
# Boot parameters (GRUB_CMDLINE_LINUX)
isolcpus=2-7                   # Isolate from scheduler
nohz_full=2-7                  # Disable timer ticks
rcu_nocbs=2-7                  # Offload RCU callbacks

# Pin process to isolated CPUs
taskset -c 2-7 ./application
```

### IRQ Affinity

```bash
# View IRQ affinities
cat /proc/irq/*/smp_affinity

# Set IRQ to specific CPU (hex mask, CPU 0 = 1, CPU 1 = 2, etc.)
echo 1 > /proc/irq/IRQ_NUM/smp_affinity

# Disable irqbalance for manual control
systemctl stop irqbalance
```

### Scheduler Parameters

```bash
# Scheduler tuning
sysctl kernel.sched_min_granularity_ns=1000000    # Min timeslice
sysctl kernel.sched_wakeup_granularity_ns=1500000 # Wakeup granularity
sysctl kernel.sched_migration_cost_ns=500000      # Migration cost
sysctl kernel.sched_autogroup_enabled=0           # Disable autogroup (servers)
```

### sched_ext (SCX): Pluggable BPF Schedulers

sched_ext allows loading custom schedulers as BPF programs at runtime without kernel recompilation. Available since kernel 6.12.

```bash
# Install sched_ext schedulers (from scx repo)
git clone https://github.com/sched-ext/scx
cd scx && meson build && ninja -C build

# Load latency-optimized scheduler for gaming
sudo scx_lavd

# Load for better throughput
sudo scx_rusty

# Stop and revert to default scheduler
# Ctrl+C or kill the scheduler process
```

**Gaming performance metrics:**
- Track P99 frame times, not just average FPS
- Consistent 55 FPS beats variable 30-70 FPS
- Frame time = 1000ms / FPS (16.6ms for 60 FPS)

**Key insight:** Latency-sensitive tasks (games, compositors) benefit from deadline-based scheduling that prioritizes short CPU bursts over fairness.

### Custom Scheduler for Concurrency Testing

sched_ext enables deliberately erratic scheduling to expose race conditions that are hard to reproduce.

```bash
# Simple SCX scheduler structure
# init(): create dispatch queue
# enqueue(): add task to queue with timeslice
# dispatch(): consume from queue to CPU

# Force race conditions by:
# - Random sleep delays before scheduling
# - Very short timeslices
# - Aggressive preemption

# SCX safety: auto-eject after 30s without scheduling
# Soft lockup detector kicks in at ~45s
```

**Key insight:** Bad schedulers expose latent bugs. Use SCX to deliberately delay scheduling and make race conditions reproducible.

### PREEMPT_RT Real-Time Kernel

PREEMPT_RT merged into mainline kernel 6.12+ for x86, ARM64, and RISC-V. Converts spinlocks to preemptible mutexes for bounded latency.

```bash
# Check if running RT kernel
uname -a  # Look for PREEMPT_RT

# Enable at build time
make menuconfig
# General setup -> Preemption Model -> Fully Preemptible (RT)

# Real-time task setup
#include <sched.h>
struct sched_param param = { .sched_priority = 80 };
sched_setscheduler(0, SCHED_FIFO, &param);
mlockall(MCL_CURRENT | MCL_FUTURE);  # Lock memory

# Test latency with cyclictest
cyclictest -p 80 -t 4 -n -l 1000000
# Target: <100us on x86, <200us on embedded
```

**Key insight:** RT is not about speed, it's about bounded worst-case latency. Expect ~80us max latency on x86 with proper configuration.

### SCHED_DEADLINE Real-Time Scheduling

SCHED_DEADLINE provides hard real-time guarantees with admission control.

```c
#include <linux/sched.h>

struct sched_attr attr = {
    .size = sizeof(attr),
    .sched_policy = SCHED_DEADLINE,
    .sched_runtime = 10000000,   // 10ms worst-case runtime
    .sched_deadline = 30000000,  // 30ms deadline
    .sched_period = 100000000,   // 100ms period
};
sched_setattr(0, &attr, 0);
```

**Key insight:** Kernel performs admission test on sched_setattr(). If total utilization exceeds capacity, the call fails. Exceeding runtime triggers SIGXCPU.

### NUMA

```bash
# View NUMA topology
numactl --hardware
lstopo

# Pin to NUMA node
numactl --cpunodebind=0 --membind=0 ./application

# Interleave memory
numactl --interleave=all ./application

# Disable NUMA balancing (for pinned workloads)
sysctl vm.numa_balancing=0
```

## Memory Management

### Virtual Memory

```bash
# Swappiness (0-100, lower = less swap)
sysctl vm.swappiness=10                    # Desktop/DB
sysctl vm.swappiness=1                     # Almost no swap

# Cache pressure (higher = reclaim sooner)
sysctl vm.vfs_cache_pressure=50            # Default 100

# Dirty page settings
sysctl vm.dirty_ratio=15                   # % memory before sync write
sysctl vm.dirty_background_ratio=5         # % memory before async write
sysctl vm.dirty_expire_centisecs=3000      # 30s max dirty age
sysctl vm.dirty_writeback_centisecs=500    # 5s writeback interval

# For fast storage (NVMe)
sysctl vm.dirty_ratio=40
sysctl vm.dirty_background_ratio=10
```

### Overcommit

```bash
# Overcommit modes
sysctl vm.overcommit_memory=0              # Heuristic (default)
sysctl vm.overcommit_memory=1              # Always allow
sysctl vm.overcommit_memory=2              # Strict (swap + ratio*RAM)
sysctl vm.overcommit_ratio=80              # With mode 2
```

### Transparent Huge Pages

```bash
# Check status
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never

# Disable THP (recommended for databases)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# Enable for madvise only
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled
```

### Huge Pages (explicit)

```bash
# Reserve huge pages
sysctl vm.nr_hugepages=1024                # 1024 * 2MB = 2GB

# Check huge page size
cat /proc/meminfo | grep Hugepagesize

# Mount hugetlbfs
mount -t hugetlbfs none /mnt/hugepages
```

### OOM Killer

```bash
# Adjust OOM score (-1000 to 1000, lower = less likely to kill)
echo -500 > /proc/PID/oom_score_adj

# Disable OOM killer (dangerous)
sysctl vm.panic_on_oom=1
sysctl vm.oom_kill_allocating_task=1

# View OOM score
cat /proc/PID/oom_score
```

### Zone Reclaim

```bash
# Disable zone reclaim (NUMA optimization)
sysctl vm.zone_reclaim_mode=0
```

### DAMON: Data Access Monitoring

DAMON monitors memory access patterns with configurable accuracy/overhead tradeoff. Uses region-based sampling instead of per-page tracking for scalability.

```bash
# Install DAMON userspace tools
apt install damon-tools

# Start monitoring a process
damo start --target_pid <PID>

# Show access pattern (regions, frequency, age)
damo show

# Proactive reclaim: evict cold pages before memory pressure
damo schemes --action pageout \
    --access_pattern "0 0 5min max" \
    --quotas "100M/s 2%"

# DAMON schemes for memory tiering
# Move hot pages to faster NUMA node
damo schemes --action migrate_hot \
    --access_pattern "10 max 0 1s"
```

**Key insight:** DAMON enables proactive memory management based on actual access patterns, not just LRU heuristics. Typical overhead is 3-4% single CPU usage.

### Per-CPU Memory Allocation (librseq)

Per-CPU data avoids cache line bouncing between cores. librseq provides userspace per-CPU allocator with restartable sequences.

```bash
# Check rseq support
cat /proc/self/rseq

# rseq concurrency ID (Linux 6.3+)
# Provides process-local CPU index for compact per-CPU arrays
```

**Design patterns:**
```c
// Anti-pattern: array indexed by CPU
data[sched_getcpu()];  // Cache line bouncing

// Better: per-CPU memory pools
// All CPU0 data together, all CPU1 data together
// Stride = pool_size, not element_size

// rseq critical section
// If preempted/migrated, aborts and retries
```

**Key insight:** Per-CPU allocation with copy-on-write init values avoids touching memory for unused CPUs in containers.

### Buddy Allocator with Bitmap Indexing

Buddy allocator using pure bitmaps instead of linked lists. Fixed overhead: 2 bits per minimum page.

```
Overhead calculation:
- 4KB pages: 0.006% overhead (65GB to manage 1PB)
- 256B pages: 0.1% overhead

Key operations (all O(1)):
- Free: set bit
- Take: clear bit
- Split: take parent, free right child
- Merge: check buddy bit, take if free, recurse up
```

**Key insight:** For disk/cache allocators, bitmap-based buddy allocator enables defragmentation through selective eviction until pages coalesce.

## I/O Tuning

### I/O Scheduler

```bash
# Check current scheduler
cat /sys/block/sda/queue/scheduler

# Set scheduler
echo mq-deadline > /sys/block/sda/queue/scheduler
```

| Scheduler | Use Case |
|-----------|----------|
| `none` | NVMe, fast storage |
| `mq-deadline` | Default, balanced |
| `bfq` | Desktop, fair queuing |
| `kyber` | Low latency |

### Queue Parameters

```bash
# Queue depth
echo 256 > /sys/block/sda/queue/nr_requests

# Read-ahead (KB)
echo 4096 > /sys/block/sda/queue/read_ahead_kb  # HDD
echo 256 > /sys/block/nvme0n1/queue/read_ahead_kb  # NVMe

# Max sectors per request
echo 1024 > /sys/block/sda/queue/max_sectors_kb

# Disable I/O merging (low latency)
echo 0 > /sys/block/sda/queue/nomerges
```

### Filesystem Options

```bash
# Mount options
mount -o noatime,nodiratime /dev/sda1 /mnt  # Skip access time
mount -o discard /dev/sda1 /mnt              # TRIM for SSD

# ext4 specific
mount -o barrier=0 ...                        # Disable barriers (risky)
mount -o data=writeback ...                   # Writeback mode (risky)

# XFS specific
mount -o nobarrier,logbufs=8,logbsize=256k ...
```

## Security Parameters

```bash
# Address Space Layout Randomization
sysctl kernel.randomize_va_space=2           # Full ASLR (default)

# Restrict dmesg
sysctl kernel.dmesg_restrict=1

# Restrict kernel pointers
sysctl kernel.kptr_restrict=2

# Restrict perf
sysctl kernel.perf_event_paranoid=2          # Restrict unprivileged

# Core dumps
sysctl kernel.core_pattern=/var/crash/core.%e.%p
sysctl fs.suid_dumpable=0                    # No SUID core dumps
```

## File Descriptors

```bash
# System-wide limits
sysctl fs.file-max=2000000                   # Max open files
sysctl fs.nr_open=2000000                    # Per-process hard limit

# Check current usage
cat /proc/sys/fs/file-nr
# allocated  free  max

# Per-process (ulimit or limits.conf)
ulimit -n 65535
```

**/etc/security/limits.conf:**
```
*    soft    nofile    65535
*    hard    nofile    65535
*    soft    nproc     65535
*    hard    nproc     65535
```

## Kernel Livepatching

TuxTape automates CVE patch generation and deployment using kpatch. Tracks Linux CNA (CVE Naming Authority) for fix commits.

```bash
# kpatch redirects vulnerable functions via ftrace
# Patches must not change stack frame size

# CVE triage workflow:
# 1. Parse Linux CNA YAML files
# 2. Generate git diff of fix commit
# 3. Check if affected files in kernel config
# 4. Convert to kpatch-compatible module

# Profile kernel build to find included files
remake --profile  # JSON output of compiled files
```

**Key insight:** Not all CVEs affect your kernel. Profile build to identify which source files are actually compiled into your config.

## HPC Optimizations

Tested AVX-512, O3, and profile-guided optimization (PGO) on HPC workloads.

```bash
# Kernel compile with native CPU features
make KCFLAGS="-march=native"

# Profile-guided optimization
# 1. Build with coverage
make KCFLAGS="-fprofile-generate"
# 2. Run workload
# 3. Rebuild with profile
make KCFLAGS="-fprofile-use"

# Results:
# - AVX-512: negligible average improvement
# - O3 vs O2: mixed results, some regressions
# - PGO: 1-3% improvement on targeted workloads
```

**Key insight:** Kernel instruction set optimization shows minimal gains for well-optimized userspace HPC code. PGO provides modest improvement but requires workflow-specific profiling.

## sysctl Reference

### Critical for Performance

```bash
# Memory
vm.swappiness=10
vm.dirty_ratio=15
vm.dirty_background_ratio=5
vm.vfs_cache_pressure=50

# File handles
fs.file-max=2000000
fs.nr_open=2000000

# NUMA (if pinning)
vm.numa_balancing=0
vm.zone_reclaim_mode=0
```

### Persistent Configuration

**/etc/sysctl.d/99-performance.conf:**
```bash
# Memory
vm.swappiness = 10
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5
vm.vfs_cache_pressure = 50

# File descriptors
fs.file-max = 2000000

# NUMA
vm.numa_balancing = 0
vm.zone_reclaim_mode = 0

# Kernel
kernel.sched_autogroup_enabled = 0
```

## Tuned Profiles

```bash
# Install tuned
apt install tuned

# List profiles
tuned-adm list

# Set profile
tuned-adm profile throughput-performance
tuned-adm profile latency-performance
tuned-adm profile network-latency
tuned-adm profile virtual-guest

# Current profile
tuned-adm active

# Recommend profile
tuned-adm recommend
```

## Quick Reference

| Task | Setting |
|------|---------|
| Reduce swap | `vm.swappiness=10` |
| Fast I/O | `vm.dirty_ratio=40` |
| Max performance CPU | `scaling_governor=performance` |
| Isolate CPUs | `isolcpus=2-7` (boot param) |
| Disable THP | `echo never > /sys/.../enabled` |
| Pin to NUMA | `numactl --cpunodebind=0 --membind=0` |
| NVMe scheduler | `echo none > /sys/block/nvme0n1/queue/scheduler` |
| Disable IRQ balance | `systemctl stop irqbalance` |
| More file descriptors | `fs.file-max=2000000` |
