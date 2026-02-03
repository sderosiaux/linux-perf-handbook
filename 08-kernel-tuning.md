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

### EEVDF Scheduler (Default 6.6+)

Linux 6.6 replaced CFS with EEVDF (Earliest Eligible Virtual Deadline First) as the default scheduler. EEVDF provides better latency guarantees while maintaining fairness.

**Key differences from CFS:**
- Uses virtual deadlines instead of vruntime alone
- Provides bounded latency guarantees
- Better behavior for latency-sensitive tasks
- Simpler tuning model

```bash
# Check if using EEVDF (kernel 6.6+)
# EEVDF has different tuning knobs than CFS

# Scheduler tuning (6.6+ EEVDF)
sysctl kernel.sched_min_granularity_ns=3000000    # Min slice (3ms default)
sysctl kernel.sched_migration_cost_ns=500000      # Migration cost
sysctl kernel.sched_autogroup_enabled=0           # Disable autogroup (servers)

# Legacy CFS tuning (pre-6.6)
# sysctl kernel.sched_wakeup_granularity_ns=1500000  # Removed in EEVDF
```

**Key insight:** EEVDF eliminates the need for manual latency tuning in most cases. Tasks naturally get bounded latency based on their share of CPU time.

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
# Swappiness (0-200, lower = less swap; >100 prefers swap over file reclaim, kernel 5.8+)
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

### Per-VMA Locks (6.4+)

Linux 6.4+ introduced per-VMA (Virtual Memory Area) locks, dramatically reducing mmap_lock contention in multi-threaded applications.

**Impact:**
- Reduces mmap_lock contention by 2-3x in high-concurrency workloads
- Particularly benefits malloc-heavy applications
- No configuration needed - enabled automatically

```bash
# Check if per-VMA locks are active (6.4+)
grep CONFIG_PER_VMA_LOCK /boot/config-$(uname -r)

# Monitor mmap_lock contention
perf lock record -a -- sleep 10
perf lock report
```

**Key insight:** If you're hitting mmap_lock bottlenecks on older kernels (visible in `perf top` as `__mmap_lock*`), upgrading to 6.4+ can provide significant throughput improvements without code changes.

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

## Memory Allocator Deep-Dive

Allocator choice causes 2-10x difference in latency tail and throughput. The default glibc malloc is adequate for most workloads but becomes a bottleneck under high concurrency or fragmentation pressure.

### Allocator Comparison

| Allocator | Strengths | Weaknesses | Best For |
|-----------|-----------|------------|----------|
| glibc malloc | Ubiquitous, no setup | Poor scalability, fragmentation | Default, low-concurrency |
| jemalloc | Proven at scale, profiling, low fragmentation | Complexity, memory overhead | Production servers, monitoring-heavy |
| tcmalloc | Simple integration, heap profiling | Less tunable than jemalloc | Google-style workloads |
| mimalloc | Bounded latency, low overhead, simple | Newer, less battle-tested | Latency-critical, general purpose |
| snmalloc | Message-passing free, batch operations | Specialized design | Thread-per-core architectures |

**Performance characteristics (synthetic benchmarks):**

| Allocator | Throughput (ops/s) | p99 Latency | Memory Overhead |
|-----------|-------------------|-------------|-----------------|
| glibc 2.35 | 1.0x baseline | 100-500us | ~8 bytes/alloc |
| jemalloc 5.3 | 1.5-2.5x | 10-50us | ~2% fragmentation |
| tcmalloc | 1.3-2.0x | 20-80us | ~1-2% |
| mimalloc 2.x | 1.8-3.0x | 5-30us | ~0.2% metadata |

**Selection criteria:**

```
High concurrency (>16 threads)    -> jemalloc or mimalloc
Latency-sensitive                 -> mimalloc
Need profiling/debugging          -> jemalloc (stats) or tcmalloc (pprof)
Memory-constrained                -> mimalloc (lowest overhead)
Thread-per-core architecture      -> snmalloc
Default/simple deployment         -> tcmalloc (LD_PRELOAD)
```

### glibc malloc Internals

glibc malloc uses per-thread arenas with bins for different size classes.

```
Architecture:
+----------------------------------------------------------+
|                     Main Arena                            |
|  +----------------------------------------------------+  |
|  | Fast Bins (16-80 bytes, LIFO, no coalescing)       |  |
|  +----------------------------------------------------+  |
|  | Small Bins (< 512 bytes, FIFO, exact size)         |  |
|  +----------------------------------------------------+  |
|  | Large Bins (>= 512 bytes, sorted by size)          |  |
|  +----------------------------------------------------+  |
|  | Unsorted Bin (recently freed, any size)            |  |
|  +----------------------------------------------------+  |
+----------------------------------------------------------+
|                   Thread Arena 1                          |
+----------------------------------------------------------+
|                   Thread Arena 2                          |
+----------------------------------------------------------+
```

**Tuning via environment variables:**

```bash
# Control arena count (default: 8 * num_cores)
export MALLOC_ARENA_MAX=4

# Trim threshold (bytes before returning to OS)
export MALLOC_TRIM_THRESHOLD_=131072

# Mmap threshold (allocations above this use mmap)
export MALLOC_MMAP_THRESHOLD_=131072

# Top pad (extra space to keep after trim)
export MALLOC_TOP_PAD_=0

# Per-thread cache (0 to disable)
export MALLOC_PERTURB_=0
```

**Diagnostic tools:**

```bash
# Memory statistics
export MALLOC_CHECK_=3          # Enable checking (slow)
malloc_stats()                   # Call from code or gdb
malloc_info(0, stdout)           # XML output

# Debug with gdb
(gdb) call malloc_stats()
(gdb) call malloc_info(0, stdout)

# Fragmentation indicator
cat /proc/buddyinfo              # Kernel buddy allocator state
```

**Limitations:**
- Lock contention on arena mutex under high thread count
- Fragmentation accumulates over long-running processes
- No built-in profiling beyond basic stats
- Memory not returned to OS aggressively

### jemalloc Tuning

jemalloc (used by Redis, Firefox, Redpanda) provides extensive tuning and profiling.

**Installation:**

```bash
# Build from source
git clone https://github.com/jemalloc/jemalloc
cd jemalloc && ./autogen.sh && ./configure --enable-prof && make -j
sudo make install

# Or use package manager
apt install libjemalloc2 libjemalloc-dev

# Use via LD_PRELOAD
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2 ./application

# Link at compile time
gcc -o app app.c -ljemalloc
```

**Architecture:**

```
+--------------------------------------------------------------+
|                         jemalloc                             |
|  +--------------------------------------------------------+  |
|  | Thread Cache (tcache) - per-thread, lock-free          |  |
|  |   Small bins: 8, 16, 32, ... 14336 bytes               |  |
|  +--------------------------------------------------------+  |
|              | overflow/underflow                            |
|              v                                               |
|  +--------------------------------------------------------+  |
|  | Arenas (default: 4 * num_cores)                        |  |
|  |   +--------------------------------------------------+ |  |
|  |   | Small allocations: slab allocator (runs/slabs)   | |  |
|  |   | Large allocations: extent-based                  | |  |
|  |   +--------------------------------------------------+ |  |
|  +--------------------------------------------------------+  |
|              |                                               |
|              v                                               |
|  +--------------------------------------------------------+  |
|  | Extent Allocator (manages chunks from OS)              |  |
|  |   Dirty pages -> Muzzy pages -> Clean pages -> OS      |  |
|  +--------------------------------------------------------+  |
+--------------------------------------------------------------+
```

**MALLOC_CONF options (comprehensive):**

```bash
# Arena configuration
export MALLOC_CONF="narenas:4"           # Number of arenas (reduce for low-thread)
export MALLOC_CONF="narenas:2,lg_tcache_max:15"  # Max cached size 32KB

# Dirty page decay (return memory to OS)
export MALLOC_CONF="dirty_decay_ms:1000"    # 1s decay (default 10s)
export MALLOC_CONF="dirty_decay_ms:0"       # Immediate return (high syscall cost)
export MALLOC_CONF="dirty_decay_ms:-1"      # Never decay (max performance, memory hog)

# Muzzy page decay (MADV_FREE'd pages)
export MALLOC_CONF="muzzy_decay_ms:0"       # Skip muzzy state entirely

# Background threads for decay
export MALLOC_CONF="background_thread:true" # Async page return

# Huge pages
export MALLOC_CONF="thp:always"             # Use THP when available
export MALLOC_CONF="metadata_thp:always"    # THP for metadata too

# Profiling (requires --enable-prof at build)
export MALLOC_CONF="prof:true,prof_prefix:jeprof"
export MALLOC_CONF="prof:true,lg_prof_interval:30"  # Sample every 1GB
export MALLOC_CONF="prof:true,lg_prof_sample:19"    # Sample rate (2^19 bytes)
export MALLOC_CONF="prof:true,prof_gdump:true"      # Dump on exit
export MALLOC_CONF="prof:true,prof_leak:true"       # Leak detection

# Combined production config
export MALLOC_CONF="narenas:4,dirty_decay_ms:5000,muzzy_decay_ms:5000,background_thread:true,thp:always"
```

**Statistics and profiling:**

```bash
# Enable stats
export MALLOC_CONF="stats_print:true"       # Print on exit

# Runtime stats via mallctl
#include <jemalloc/jemalloc.h>

// Refresh stats
uint64_t epoch = 1;
mallctl("epoch", NULL, NULL, &epoch, sizeof(epoch));

// Read stats
size_t allocated, active, mapped;
size_t sz = sizeof(size_t);
mallctl("stats.allocated", &allocated, &sz, NULL, 0);
mallctl("stats.active", &active, &sz, NULL, 0);
mallctl("stats.mapped", &mapped, &sz, NULL, 0);

// Fragmentation = (active - allocated) / active
printf("fragmentation: %.2f%%\n", 100.0 * (active - allocated) / active);
```

**Profile analysis:**

```bash
# Generate heap profile
export MALLOC_CONF="prof:true,prof_prefix:/tmp/heap"
./application

# Analyze with jeprof
jeprof --show_bytes --pdf ./application /tmp/heap.*.heap > heap.pdf
jeprof --show_bytes --text ./application /tmp/heap.*.heap

# Live profiling
jeprof --show_bytes --web ./application http://localhost:PORT/debug/pprof/heap
```

**Arena assignment strategies:**

```bash
# Fixed arena per thread (reduce contention)
export MALLOC_CONF="narenas:$(nproc),percpu_arena:percpu"

# Explicit arena binding (code)
unsigned arena_ind;
size_t sz = sizeof(arena_ind);
mallctl("arenas.create", &arena_ind, &sz, NULL, 0);
mallctl("thread.arena", NULL, NULL, &arena_ind, sizeof(arena_ind));
```

### tcmalloc (Google)

tcmalloc (Thread-Caching Malloc) from Google's gperftools.

**Installation:**

```bash
# From package manager
apt install libgoogle-perftools-dev google-perftools

# Use via LD_PRELOAD
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libtcmalloc.so.4 ./application

# Or link
gcc -o app app.c -ltcmalloc
```

**Architecture:**

```
+------------------------------------------------------------+
|                       tcmalloc                             |
|  +------------------------------------------------------+  |
|  | Front-end: Per-thread cache (no locks)               |  |
|  |   Size classes: 8, 16, 32, ... up to 256KB           |  |
|  +------------------------------------------------------+  |
|              | transfer batch                              |
|              v                                             |
|  +------------------------------------------------------+  |
|  | Middle-end: Transfer cache (shared, brief lock)      |  |
|  +------------------------------------------------------+  |
|              |                                             |
|              v                                             |
|  +------------------------------------------------------+  |
|  | Back-end: Page heap (spans of pages)                 |  |
|  |   Small: Central free lists                          |  |
|  |   Large: Page-level management                       |  |
|  +------------------------------------------------------+  |
+------------------------------------------------------------+
```

**Configuration:**

```bash
# Environment variables
export TCMALLOC_LARGE_ALLOC_REPORT_THRESHOLD=1073741824  # 1GB
export TCMALLOC_MAX_TOTAL_THREAD_CACHE_BYTES=33554432    # 32MB total thread cache
export TCMALLOC_RELEASE_RATE=1.0   # Aggressiveness of releasing memory (0-10)

# Sampling for profiling
export TCMALLOC_SAMPLE_PARAMETER=524288  # Sample every 512KB
```

**Heap profiling:**

```bash
# Enable heap profiling
export HEAPPROFILE=/tmp/heap

# Run application
./application

# Analyze profiles
pprof --show_bytes --pdf ./application /tmp/heap.0001.heap > heap.pdf
pprof --show_bytes --text ./application /tmp/heap.0001.heap

# Top allocators
pprof --show_bytes --top ./application /tmp/heap.0001.heap

# Diff two profiles
pprof --show_bytes --base=/tmp/heap.0001.heap --pdf ./application /tmp/heap.0010.heap > diff.pdf
```

**CPU profiling (bonus):**

```bash
# CPU profile
export CPUPROFILE=/tmp/cpu.prof
./application
pprof --pdf ./application /tmp/cpu.prof > cpu.pdf
```

**Huge page support:**

```bash
# tcmalloc with huge pages
export TCMALLOC_MEMFS_MALLOC_PATH=/mnt/hugepages

# Ensure hugetlbfs mounted
mount -t hugetlbfs none /mnt/hugepages
echo 1024 > /proc/sys/vm/nr_hugepages
```

**Runtime introspection:**

```c
#include <gperftools/malloc_extension.h>

// Get stats
char stats[4096];
MallocExtension_GetStats(stats, sizeof(stats));
printf("%s\n", stats);

// Release free memory to OS
MallocExtension_ReleaseFreeMemory();

// Get specific property
size_t value;
MallocExtension_GetNumericProperty("generic.current_allocated_bytes", &value);
```

### mimalloc (Microsoft)

mimalloc prioritizes bounded allocation times and low overhead.

**Installation:**

```bash
# Build from source
git clone https://github.com/microsoft/mimalloc
cd mimalloc && mkdir build && cd build
cmake .. && make -j
sudo make install

# Use via LD_PRELOAD
LD_PRELOAD=/usr/local/lib/libmimalloc.so ./application

# Statically link (recommended)
gcc -o app app.c -lmimalloc
```

**Design philosophy:**
- Free list sharding: Multiple free lists per page reduce fragmentation
- Single CAS for cross-thread frees (no locks)
- Eager page purging via `madvise(MADV_FREE)`
- ~0.2% metadata overhead (vs 2%+ for others)
- Bounded allocation time (no worst-case scans)

**Configuration:**

```bash
# Huge pages (8GB reserved)
export MIMALLOC_RESERVE_HUGE_OS_PAGES=8

# Page reset delay (ms, higher = better perf, more memory)
export MIMALLOC_PURGE_DELAY=100       # Default 10
export MIMALLOC_PURGE_DELAY=-1        # Never purge

# Arena count (per NUMA node)
export MIMALLOC_ARENA_EAGER_COMMIT=2  # Pre-commit arenas

# Debugging
export MIMALLOC_SHOW_STATS=1          # Print stats on exit
export MIMALLOC_VERBOSE=1             # Verbose output
export MIMALLOC_SHOW_ERRORS=1         # Show errors

# NUMA awareness
export MIMALLOC_USE_NUMA_NODES=1      # Respect NUMA topology
export MIMALLOC_ALLOW_LARGE_OS_PAGES=1  # Use huge pages if available

# Secure mode (guard pages, randomization)
export MIMALLOC_SECURE=1              # +10% overhead
```

**When mimalloc excels:**
- Latency-sensitive applications (trading, games)
- High allocation rate with varied sizes
- Memory-constrained environments
- Simple drop-in replacement

**Runtime API:**

```c
#include <mimalloc.h>

// Stats
mi_stats_print(NULL);
mi_stats_print_out(my_output_fn, NULL);

// Memory info
size_t peak = mi_peak_rss();
size_t current = mi_process_info(NULL, NULL, NULL, NULL, NULL, NULL, NULL);

// Force collection
mi_collect(true);  // Force full collection
```

### Fragmentation Analysis

Memory fragmentation degrades performance over time, especially for long-running services.

**Types of fragmentation:**

| Type | Description | Impact |
|------|-------------|--------|
| External | Free memory exists but non-contiguous | Large allocations fail |
| Internal | Allocated blocks have unused space | Memory waste |
| Cache | Hot/cold pages mixed | TLB misses, cache pollution |

**Detection methods:**

```bash
# 1. Buddy allocator state (kernel)
cat /proc/buddyinfo
# Node 0, zone Normal   1234  567  234   89   34   12   4   1   0   0   0
#                        4K   8K  16K  32K  64K 128K ...
# Many small orders, few large = fragmented

# 2. Memory stats comparison
cat /proc/meminfo | grep -E "MemFree|MemAvailable|Buffers|Cached"
# Large gap between MemFree and MemAvailable = fragmented

# 3. Process-level with pmap
pmap -x PID | tail -5
# Compare RSS vs actual heap usage

# 4. jemalloc fragmentation
# (active - allocated) / active
export MALLOC_CONF="stats_print:true"

# 5. /proc/PID/smaps analysis
awk '/^[0-9a-f]/ {region=$0} /Rss:/ {rss+=$2} /Pss:/ {pss+=$2} END {print "RSS:",rss,"PSS:",pss}' /proc/PID/smaps

# 6. Compaction score (kernel 5.17+)
cat /proc/vmstat | grep compact
```

**Monitoring script:**

```bash
#!/bin/bash
# fragmentation-monitor.sh
while true; do
    echo "=== $(date) ==="

    # Buddy fragmentation index
    cat /sys/kernel/debug/extfrag/extfrag_index 2>/dev/null || echo "extfrag not available"

    # Compaction stats
    grep -E "compact_|thp_" /proc/vmstat

    # Per-process (top 5 by RSS)
    ps aux --sort=-%mem | head -6 | awk 'NR>1 {print $2, $4"%", $6"K", $11}'

    sleep 60
done
```

**Mitigation strategies:**

```bash
# 1. Enable memory compaction
echo 1 > /proc/sys/vm/compact_memory        # Manual trigger
sysctl vm.compaction_proactiveness=20       # Proactive (0-100)

# 2. Use huge pages (reduces page table fragmentation)
echo always > /sys/kernel/mm/transparent_hugepage/enabled

# 3. Limit arena sprawl (jemalloc)
export MALLOC_CONF="narenas:4"

# 4. Aggressive page release
export MALLOC_CONF="dirty_decay_ms:1000,muzzy_decay_ms:0"

# 5. Restart long-running processes periodically
# (simplest mitigation for severe fragmentation)

# 6. Pre-allocate at startup
# Allocate working set upfront, avoid runtime growth
```

**Application-level fixes:**

```c
// 1. Use memory pools for frequent allocations
struct Pool {
    void *blocks[1024];
    int free_count;
};

// 2. Batch allocations
void *batch = malloc(count * size);
for (int i = 0; i < count; i++) {
    items[i] = batch + i * size;
}

// 3. Reuse allocations
// Instead of malloc/free in loop, maintain free list

// 4. Arena allocator for request handling
struct Arena {
    char *base;
    size_t offset;
    size_t capacity;
};
// Bump allocate during request, reset at end
```

### Allocator Profiling Tools

**perf for malloc analysis:**

```bash
# Count malloc calls
perf stat -e 'probe:malloc' ./application

# Add probe
perf probe -x /lib/x86_64-linux-gnu/libc.so.6 malloc
perf record -e probe:malloc -g ./application
perf report

# Memory access patterns
perf mem record ./application
perf mem report --sort=mem
```

**Valgrind massif (heap profiler):**

```bash
# Profile heap usage over time
valgrind --tool=massif --time-unit=B ./application

# View results
ms_print massif.out.PID

# Output:
#     MB
# 23.45^                                                       #
#      |                                                     ::#
#      |                                                   ::@:#
#      |                                                 ::@@:#:
# ...
```

**heaptrack (KDE):**

```bash
# Install
apt install heaptrack heaptrack-gui

# Profile
heaptrack ./application

# Analyze (GUI)
heaptrack_gui heaptrack.application.PID.gz

# Or text output
heaptrack_print heaptrack.application.PID.gz
```

**bpftrace for allocation tracing:**

```bash
# Count allocations by size
bpftrace -e 'uprobe:/lib/x86_64-linux-gnu/libc.so.6:malloc { @sizes = hist(arg0); }'

# Track large allocations
bpftrace -e 'uprobe:/lib/x86_64-linux-gnu/libc.so.6:malloc /arg0 > 1048576/ { printf("large alloc: %d bytes\n", arg0); print(ustack); }'

# Allocation rate
bpftrace -e 'uprobe:/lib/x86_64-linux-gnu/libc.so.6:malloc { @allocs = count(); } interval:s:1 { print(@allocs); clear(@allocs); }'
```

**eBPF memleak tool:**

```bash
# From bcc-tools
/usr/share/bcc/tools/memleak -p PID

# Sample output:
# [12:34:56] Top 10 stacks with outstanding allocations:
#     1024 bytes in 4 allocations from stack
#         malloc+0x1a [libc.so.6]
#         my_function+0x23 [application]
```

## NUMA Deep-Dive

NUMA (Non-Uniform Memory Access) architectures have multiple memory controllers with different access latencies. Proper NUMA awareness prevents 30-100ns additional latency per memory access.

### NUMA Topology Discovery

**Basic tools:**

```bash
# Hardware topology overview
numactl --hardware

# Output:
# available: 2 nodes (0-1)
# node 0 cpus: 0 1 2 3 4 5 6 7 16 17 18 19 20 21 22 23
# node 0 size: 65536 MB
# node 0 free: 45231 MB
# node 1 cpus: 8 9 10 11 12 13 14 15 24 25 26 27 28 29 30 31
# node 1 size: 65536 MB
# node 1 free: 52104 MB
# node distances:
# node   0   1
#   0:  10  21
#   1:  21  10

# CPU topology
lscpu | grep -E "NUMA|Socket|Core|Thread"
# NUMA node(s):          2
# NUMA node0 CPU(s):     0-7,16-23
# NUMA node1 CPU(s):     8-15,24-31
```

**Graphical topology (lstopo):**

```bash
# Install hwloc
apt install hwloc

# Text output
lstopo --of txt

# ASCII art
lstopo-no-graphics

# Generate image
lstopo --of png > topology.png

# Detailed with memory
lstopo --whole-system --no-io

# Filter specific info
lstopo --only numa
lstopo --only core
lstopo --only pu   # Processing units (threads)
```

**Sysfs exploration:**

```bash
# NUMA node details
ls /sys/devices/system/node/

# CPUs per node
cat /sys/devices/system/node/node0/cpulist
# 0-7,16-23

# Memory per node
cat /sys/devices/system/node/node0/meminfo
# Node 0 MemTotal:       67108864 kB
# Node 0 MemFree:        46333952 kB
# Node 0 MemUsed:        20774912 kB

# Distance matrix
cat /sys/devices/system/node/node0/distance
# 10 21

# Huge pages per node
cat /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
```

**Memory allocation by node:**

```bash
# View memory stats per node
numastat

# Per-process NUMA stats
numastat -p PID

# Output:
# Per-node process memory usage (in MBs)
# PID             Node 0  Node 1  Total
# ---------------  ------ ------ ------
# 12345 (app)       2048    156   2204

# Detailed per-node stats
numastat -m

# Watch changes
watch -n 1 'numastat -p PID'
```

### Memory Policies

NUMA memory policies control where memory is allocated relative to CPUs.

**Policy types:**

| Policy | Behavior | Use Case |
|--------|----------|----------|
| `default` | Local allocation, fallback to other nodes | General purpose |
| `bind` | Strict allocation on specified nodes | Pinned workloads |
| `interleave` | Round-robin across nodes | Large shared data |
| `preferred` | Prefer node, fallback allowed | Soft affinity |
| `local` | Allocate on current node | Most common |

**numactl usage:**

```bash
# Bind CPU and memory to same node
numactl --cpunodebind=0 --membind=0 ./application

# Prefer node 0, allow fallback
numactl --cpunodebind=0 --preferred=0 ./application

# Interleave across all nodes (databases, large caches)
numactl --interleave=all ./application

# Interleave on specific nodes
numactl --interleave=0,1 ./application

# Local allocation policy
numactl --localalloc ./application

# Specify physical CPUs directly
numactl --physcpubind=0-7 --membind=0 ./application

# Show current policy
numactl --show
```

**Programmatic control (set_mempolicy):**

```c
#include <numaif.h>
#include <numa.h>

// Interleave policy for current process
unsigned long nodemask = 3;  // Nodes 0 and 1
set_mempolicy(MPOL_INTERLEAVE, &nodemask, sizeof(nodemask) * 8);

// Bind to node 0
nodemask = 1;  // Node 0
set_mempolicy(MPOL_BIND, &nodemask, sizeof(nodemask) * 8);

// Preferred node
set_mempolicy(MPOL_PREFERRED, &nodemask, sizeof(nodemask) * 8);

// Get current policy
int policy;
unsigned long maxnode = 64;
unsigned long cur_nodemask;
get_mempolicy(&policy, &cur_nodemask, maxnode, NULL, 0);
```

**Per-range binding (mbind):**

```c
#include <numaif.h>
#include <sys/mman.h>

// Allocate memory
void *ptr = mmap(NULL, size, PROT_READ|PROT_WRITE,
                 MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);

// Bind to specific node
unsigned long nodemask = 1;  // Node 0
mbind(ptr, size, MPOL_BIND, &nodemask, sizeof(nodemask) * 8, MPOL_MF_MOVE);

// Flags:
// MPOL_MF_STRICT - fail if can't satisfy
// MPOL_MF_MOVE   - move existing pages
// MPOL_MF_MOVE_ALL - move all pages (root only)
```

**libnuma convenience functions:**

```c
#include <numa.h>

// Check NUMA availability
if (numa_available() < 0) {
    // No NUMA support
}

// Allocate on specific node
void *ptr = numa_alloc_onnode(size, node);

// Allocate interleaved
void *ptr = numa_alloc_interleaved(size);

// Allocate local
void *ptr = numa_alloc_local(size);

// Move pages to node
numa_tonode_memory(ptr, size, node);

// Set CPU affinity by node
numa_run_on_node(node);

// Free NUMA allocation
numa_free(ptr, size);
```

**Policy comparison:**

| Scenario | Recommended Policy |
|----------|-------------------|
| Single-threaded, local data | `--membind=N` (same as CPU) |
| Large shared hash table | `--interleave=all` |
| Database buffer pool | `--interleave=all` |
| Thread-per-core, private data | `--membind` per thread |
| Unknown access pattern | `default` (let AutoNUMA handle) |

### CPU Affinity with NUMA

Combine CPU pinning with memory binding for optimal locality.

**taskset + numactl:**

```bash
# Pin to CPUs 0-7 with memory on node 0
taskset -c 0-7 numactl --membind=0 ./application

# Or equivalently
numactl --cpunodebind=0 --membind=0 ./application

# Multiple processes with different affinities
numactl --cpunodebind=0 --membind=0 ./worker1 &
numactl --cpunodebind=1 --membind=1 ./worker2 &
```

**Optimal placement strategies:**

```
Strategy 1: Node-isolated processes
+-----------------------------+-----------------------------+
| Node 0                      | Node 1                      |
| +-------------------------+ | +-------------------------+ |
| | Process A               | | | Process B               | |
| | CPUs: 0-7               | | | CPUs: 8-15              | |
| | Memory: local           | | | Memory: local           | |
| +-------------------------+ | +-------------------------+ |
+-----------------------------+-----------------------------+

Strategy 2: Thread-per-core with local memory
+-------------------------------------------------------------+
| Process (multi-threaded)                                    |
| +---------+---------+---------+---------+                   |
| | T0:CPU0 | T1:CPU1 | T2:CPU8 | T3:CPU9 | ...               |
| | mem:N0  | mem:N0  | mem:N1  | mem:N1  |                   |
| +---------+---------+---------+---------+                   |
+-------------------------------------------------------------+

Strategy 3: Shared data interleaved, private data local
+-------------------------------------------------------------+
| Shared buffer pool: interleaved across all nodes            |
+-----------------------------+-------------------------------+
| Thread stacks: Node 0       | Thread stacks: Node 1         |
| Thread locals: Node 0       | Thread locals: Node 1         |
+-----------------------------+-------------------------------+
```

**Programmatic thread placement:**

```c
#define _GNU_SOURCE
#include <sched.h>
#include <numa.h>
#include <pthread.h>

void pin_thread_to_numa(int node) {
    // Get CPUs for this node
    struct bitmask *cpumask = numa_allocate_cpumask();
    numa_node_to_cpus(node, cpumask);

    // Pin to first CPU on node
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    for (int i = 0; i < numa_num_configured_cpus(); i++) {
        if (numa_bitmask_isbitset(cpumask, i)) {
            CPU_SET(i, &cpuset);
            break;  // Or set multiple for migration within node
        }
    }
    pthread_setaffinity_np(pthread_self(), sizeof(cpuset), &cpuset);

    // Bind memory
    numa_set_preferred(node);

    numa_free_cpumask(cpumask);
}

// In thread function
void *worker(void *arg) {
    int node = (intptr_t)arg;
    pin_thread_to_numa(node);
    // ... work
}
```

**Verification:**

```bash
# Check process NUMA distribution
numastat -p PID

# Check thread affinity
for tid in /proc/PID/task/*; do
    echo "Thread $(basename $tid): $(cat $tid/status | grep Cpus_allowed_list)"
done

# Memory locality per-page
/usr/share/bcc/tools/numamove  # eBPF tool
```

### AutoNUMA

AutoNUMA (Automatic NUMA Balancing) migrates pages and tasks to improve locality without manual configuration.

**How it works:**

```
1. Kernel periodically unmaps PTEs (makes pages "prot_none")
2. Access causes page fault -> "NUMA hint fault"
3. Kernel tracks which CPUs access which pages
4. Pages migrate toward accessing CPUs
5. Tasks may migrate toward their memory

Heuristic: Move page if accessed more from remote node
           Move task if most memory is remote
```

**Configuration:**

```bash
# Enable/disable
sysctl vm.numa_balancing=1   # Enable (default on NUMA systems)
sysctl vm.numa_balancing=0   # Disable

# Scan rate control
sysctl kernel.numa_balancing_scan_delay_ms=1000    # Initial scan delay
sysctl kernel.numa_balancing_scan_period_min_ms=1000  # Min scan period
sysctl kernel.numa_balancing_scan_period_max_ms=60000 # Max scan period
sysctl kernel.numa_balancing_scan_size_mb=256      # MB scanned per period
```

**Monitoring:**

```bash
# NUMA fault statistics
grep numa /proc/vmstat
# numa_hit         123456789   # Allocations on intended node
# numa_miss        12345       # Allocations on different node
# numa_foreign     12345       # Allocations intended elsewhere, served locally
# numa_interleave  1234        # Interleave policy allocations
# numa_local       123456789   # Local allocations
# numa_other       12345       # Remote allocations
# numa_pte_updates 12345       # PTE scans
# numa_huge_pte_updates 1234
# numa_hint_faults 12345       # Page faults triggering migration consideration
# numa_hint_faults_local 1234  # Local access hint faults
# numa_pages_migrated 123456   # Pages actually migrated

# Watch migration rate
watch -n 1 "grep numa_pages_migrated /proc/vmstat"

# Per-process migration
cat /proc/PID/numa_maps | grep migrate
```

**When to disable AutoNUMA:**

```bash
# 1. Already using numactl --membind (manual control)
# 2. Real-time workloads (migration adds latency jitter)
# 3. Observed excessive migration thrashing
# 4. THP + AutoNUMA can cause stalls

# Check if thrashing
vmstat 1 | grep -v procs  # Watch si/so columns
dmesg | grep -i numa
```

**Tuning for specific workloads:**

```bash
# High-frequency trading: disable entirely
sysctl vm.numa_balancing=0

# Database: reduce aggressiveness
sysctl kernel.numa_balancing_scan_period_min_ms=5000
sysctl kernel.numa_balancing_scan_size_mb=128

# General server: defaults usually good
# Let AutoNUMA learn access patterns
```

### Cross-NUMA Penalties

Quantifying cross-NUMA access costs is essential for optimization decisions.

**Latency measurement:**

```bash
# Intel Memory Latency Checker
mlc --latency_matrix

# Example output (Xeon 8280):
# Numa node    0       1
#    0        89.2   139.4
#    1       139.4    89.3
# ~50ns penalty for cross-node access

# numactl distance (relative, not absolute ns)
numactl --hardware | grep -A3 "node distances"
# node   0   1
#   0:  10  21   # 2.1x relative cost to access node 1 from node 0
#   1:  21  10

# lmbench
lat_mem_rd -P 1 -N 1 256m 512  # Local
numactl --cpunodebind=0 --membind=1 lat_mem_rd 256m 512  # Remote
```

**Bandwidth measurement:**

```bash
# STREAM benchmark
numactl --cpunodebind=0 --membind=0 ./stream  # Local
numactl --cpunodebind=0 --membind=1 ./stream  # Remote

# Typical results (DDR4-3200):
# Local:  ~180 GB/s aggregate
# Remote: ~90-120 GB/s (50-60% of local)

# Intel MLC bandwidth
mlc --bandwidth_matrix
```

**Detection tools:**

```bash
# perf NUMA events
perf stat -e 'node-loads,node-load-misses,node-stores,node-store-misses' ./application

# Output:
# 1,234,567,890 node-loads
#    12,345,678 node-load-misses  # 1% = acceptable, >10% = problem
# ...

# Detailed per-instruction
perf mem record ./application
perf mem report --sort=mem,local_weight

# eBPF NUMA tracing
/usr/share/bcc/tools/shmsnoop  # Shared memory access
```

**bpftrace script for NUMA misses:**

```bash
#!/usr/bin/bpftrace
// numa-misses.bt
tracepoint:migrate:mm_migrate_pages
{
    @migrations[comm] = count();
}

tracepoint:page_fault:page_fault_user
/args->error_code & 0x4/  // NUMA hint fault
{
    @hint_faults[comm] = count();
}

interval:s:1
{
    print(@migrations);
    print(@hint_faults);
    clear(@migrations);
    clear(@hint_faults);
}
```

**Impact summary table:**

| Access Type | Typical Latency | Bandwidth Impact |
|-------------|-----------------|------------------|
| L1 cache | 1-4 cycles | N/A |
| L2 cache | 10-15 cycles | N/A |
| L3 cache | 30-50 cycles | N/A |
| Local DRAM | 80-100 ns | 100% |
| Remote DRAM (1 hop) | 130-150 ns | 50-70% |
| Remote DRAM (2 hops) | 180-220 ns | 30-50% |
| CXL memory | 150-250 ns | 40-80% |

### CXL Memory Tiering

Compute Express Link (CXL) enables memory expansion and tiering beyond traditional NUMA.

**CXL overview:**

```
CXL 1.1/2.0: PCIe 5.0 based, adds memory semantics
CXL 3.0: Fabric support, memory pooling, switching

Device Types:
- Type 1: Accelerator (no memory)
- Type 2: Accelerator with device memory
- Type 3: Memory expander (most common for tiering)

+-------------------------------------------------------------+
|                        CPU Socket                           |
|  +-------------+  +-------------+  +---------------------+  |
|  | Local DDR5  |  | Local DDR5  |  | CXL Controller      |  |
|  | (fastest)   |  |             |  |   |                 |  |
|  +-------------+  +-------------+  +---|-----------------+  |
+---------------------------------------|---------------------+
                                        | PCIe/CXL
                          +-------------+-------------+
                          |     CXL Memory Device     |
                          |   (Type 3 expander)       |
                          |   Latency: +50-100ns      |
                          +---------------------------+
```

**Linux CXL support:**

```bash
# Check kernel support (5.18+)
modprobe cxl_acpi cxl_pci cxl_port cxl_mem

# CXL device discovery
ls /sys/bus/cxl/devices/

# View CXL memory regions
cat /sys/bus/cxl/devices/*/type
cat /sys/bus/cxl/devices/decoder*/size

# dmesg CXL messages
dmesg | grep -i cxl

# cxl-cli (ndctl package)
apt install ndctl
cxl list
cxl list -M  # Memdevs
cxl list -R  # Regions
```

**CXL memory appears as NUMA node:**

```bash
# CXL memory shows as additional NUMA node
numactl --hardware
# available: 3 nodes (0-2)
# node 2 cpus:           # No CPUs (CXL memory-only node)
# node 2 size: 262144 MB
# node 2 free: 261000 MB
# node distances:
# node   0   1   2
#   0:  10  21  30   # CXL is "farther" than remote NUMA
#   1:  21  10  30
#   2:  30  30  10
```

**Tiering configuration:**

```bash
# Manual tiering with numactl
# Hot data on local DRAM (node 0), cold data on CXL (node 2)
numactl --cpunodebind=0 --membind=0,2 --preferred=0 ./application

# Kernel auto-tiering (6.1+)
# Enable memory tiering
echo 1 > /sys/kernel/mm/numa/demotion_enabled

# Configure demotion path
# Node 2 (CXL) as demotion target for nodes 0,1
echo 2 > /sys/devices/system/node/node0/memory_tier
echo 2 > /sys/devices/system/node/node1/memory_tier

# DAMON-based tiering (recommended)
damo schemes --action demote \
    --access_pattern "0 0 5min max" \
    --quotas "1GB/s 5%"

damo schemes --action promote \
    --access_pattern "10 max 0 1s" \
    --quotas "500MB/s 2%"
```

**Performance characteristics:**

| Metric | Local DDR5 | Remote DDR5 | CXL Memory |
|--------|------------|-------------|------------|
| Latency | 80-100 ns | 130-150 ns | 150-250 ns |
| Bandwidth (read) | 50 GB/s/channel | 30-40 GB/s | 25-35 GB/s |
| Bandwidth (write) | 50 GB/s/channel | 30-40 GB/s | 20-30 GB/s |
| Cost/GB | $$$$ | $$$$ | $$ |

**Use cases:**

```
1. Memory capacity expansion
   - More memory without adding CPU sockets
   - Cost-effective for large working sets

2. Tiered memory
   - Hot data in DDR, cold in CXL
   - Automatic with DAMON or manual with mbind

3. Memory pooling (CXL 3.0)
   - Shared memory fabric
   - Dynamic allocation to hosts

4. Persistent memory replacement
   - CXL NVM as pmem successor
```

**Monitoring CXL:**

```bash
# PCIe/CXL link status
lspci -vvv | grep -A20 "CXL"

# Bandwidth utilization
perf stat -e 'cxl/read/,cxl/write/' ./application  # Requires PMU support

# NUMA stats show CXL node
numastat -n
grep node2 /proc/vmstat
```

**Application optimization for CXL:**

```c
// Explicit CXL placement
void *hot_data = numa_alloc_onnode(size, 0);     // Local DDR
void *cold_data = numa_alloc_onnode(size, 2);   // CXL

// Or use tiering hints
#include <linux/mempolicy.h>
madvise(ptr, size, MADV_COLD);     // Hint: demote to slower tier
madvise(ptr, size, MADV_PAGEOUT);  // Hint: can be paged out

// Access pattern optimization
// - Prefetch from CXL (higher latency needs more prefetch distance)
// - Batch accesses to CXL memory
// - Keep frequently accessed data in DDR
```

### NUMA Quick Reference

```bash
# Discovery
numactl --hardware                    # Topology overview
lscpu | grep NUMA                     # CPU distribution
lstopo                                # Visual topology

# Binding
numactl --cpunodebind=N --membind=N   # Strict binding
numactl --interleave=all              # Spread across nodes
numactl --preferred=N                 # Soft preference

# Monitoring
numastat -p PID                       # Process memory distribution
grep numa /proc/vmstat                # System NUMA stats
perf stat -e node-load-misses         # Cross-node access count

# Tuning
vm.numa_balancing=0                   # Disable auto-balancing
vm.zone_reclaim_mode=0                # Disable zone reclaim
kernel.numa_balancing_scan_size_mb    # AutoNUMA aggressiveness

# CXL tiering (6.1+)
/sys/kernel/mm/numa/demotion_enabled  # Enable demotion
damo schemes --action demote/promote  # DAMON-based tiering
```

| Problem | Solution |
|---------|----------|
| High numa_miss count | Pin processes to NUMA nodes |
| Memory imbalance | Use --interleave for shared data |
| Latency jitter | Disable numa_balancing |
| Memory pressure single node | Interleave or enable AutoNUMA |
| CXL underutilized | Enable demotion, use DAMON |

## See Also

- [Network Tuning](09-network-tuning.md) - TCP/UDP optimization, NUMA networking
- [eBPF & Tracing](06-ebpf-tracing.md) - sched_ext schedulers, tracing
- [Performance Profiling](05-performance-profiling.md) - perf, flame graphs
- [Disk & Storage](04-disk-storage.md) - I/O scheduler details, filesystem tuning
