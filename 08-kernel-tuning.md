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

## CPU Tuning

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

## Memory Tuning

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

## Kernel Parameters Reference

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
