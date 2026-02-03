# sysctl Reference

Quick reference for common kernel parameters.

## Network - TCP

| Parameter | Default | Tuned | Description |
|-----------|---------|-------|-------------|
| `net.ipv4.tcp_congestion_control` | cubic | bbr | Congestion algorithm |
| `net.core.default_qdisc` | pfifo_fast | fq | Queue discipline (for BBR) |
| `net.ipv4.tcp_fastopen` | 1 | 3 | TCP Fast Open (3=client+server) |
| `net.ipv4.tcp_slow_start_after_idle` | 1 | 0 | Disable slow start after idle |
| `net.ipv4.tcp_no_metrics_save` | 0 | 1 | Don't cache TCP metrics |
| `net.ipv4.tcp_mtu_probing` | 0 | 1 | Enable MTU probing |
| `net.ipv4.tcp_sack` | 1 | 1 | Selective ACK |
| `net.ipv4.tcp_timestamps` | 1 | 1 | TCP timestamps |
| `net.ipv4.tcp_window_scaling` | 1 | 1 | Window scaling |

## Network - Buffers

| Parameter | Default | Tuned | Description |
|-----------|---------|-------|-------------|
| `net.core.rmem_max` | 212992 | 16777216 | Max receive buffer |
| `net.core.wmem_max` | 212992 | 16777216 | Max send buffer |
| `net.core.rmem_default` | 212992 | 262144 | Default receive buffer |
| `net.core.wmem_default` | 212992 | 262144 | Default send buffer |
| `net.ipv4.tcp_rmem` | 4096 131072 6291456 | 4096 87380 16777216 | TCP receive buffer (min/default/max) |
| `net.ipv4.tcp_wmem` | 4096 16384 4194304 | 4096 65536 16777216 | TCP send buffer (min/default/max) |

## Network - Connections

| Parameter | Default | Tuned | Description |
|-----------|---------|-------|-------------|
| `net.core.somaxconn` | 4096 | 65535 | Listen backlog |
| `net.ipv4.tcp_max_syn_backlog` | 1024 | 65535 | SYN backlog |
| `net.core.netdev_max_backlog` | 1000 | 65535 | Network device backlog |
| `net.ipv4.ip_local_port_range` | 32768 60999 | 1024 65535 | Ephemeral port range |
| `net.ipv4.tcp_fin_timeout` | 60 | 15 | FIN-WAIT-2 timeout |
| `net.ipv4.tcp_tw_reuse` | 2 | 1 | Reuse TIME-WAIT sockets |
| `net.ipv4.tcp_max_tw_buckets` | 65536 | 1440000 | Max TIME-WAIT sockets |

## Network - Keep-Alive

| Parameter | Default | Tuned | Description |
|-----------|---------|-------|-------------|
| `net.ipv4.tcp_keepalive_time` | 7200 | 600 | Seconds before keep-alive probe |
| `net.ipv4.tcp_keepalive_intvl` | 75 | 60 | Interval between probes |
| `net.ipv4.tcp_keepalive_probes` | 9 | 3 | Number of probes |

## Network - Connection Tracking

| Parameter | Default | Tuned | Description |
|-----------|---------|-------|-------------|
| `net.nf_conntrack_max` | 65536 | 1048576 | Max tracked connections |
| `net.netfilter.nf_conntrack_tcp_timeout_established` | 432000 | 86400 | Established timeout (sec) |
| `net.netfilter.nf_conntrack_tcp_timeout_time_wait` | 120 | 30 | TIME-WAIT timeout |

## Network - NUMA/Performance

| Parameter | Default | Description |
|-----------|---------|-------------|
| `net.core.busy_poll` | 0 | Busy polling timeout (μs) |
| `net.core.busy_read` | 0 | Busy read timeout (μs) |
| `net.core.rps_sock_flow_entries` | 0 | RFS flow entries |

## Memory - VM

| Parameter | Default | Tuned | Description |
|-----------|---------|-------|-------------|
| `vm.swappiness` | 60 | 10 | Swap tendency (0-200, >100 prefers swap) |
| `vm.vfs_cache_pressure` | 100 | 50 | Cache reclaim pressure |
| `vm.dirty_ratio` | 20 | 15 | % RAM before sync write |
| `vm.dirty_background_ratio` | 10 | 5 | % RAM before async write |
| `vm.dirty_expire_centisecs` | 3000 | 3000 | Max dirty age (centisec) |
| `vm.dirty_writeback_centisecs` | 500 | 500 | Writeback interval |
| `vm.overcommit_memory` | 0 | 0 | Overcommit mode (0/1/2) |
| `vm.overcommit_ratio` | 50 | 50 | Overcommit % (with mode 2) |

## Memory - NUMA

| Parameter | Default | Tuned | Description |
|-----------|---------|-------|-------------|
| `vm.numa_balancing` | 1 | 0 | Auto NUMA balancing (disable if pinning) |
| `vm.zone_reclaim_mode` | 0 | 0 | Zone reclaim (0=disabled) |

## Memory - Huge Pages

| Parameter | Description |
|-----------|-------------|
| `vm.nr_hugepages` | Number of huge pages to reserve |
| `vm.hugetlb_shm_group` | Group allowed to use hugetlb |

## File System

| Parameter | Default | Tuned | Description |
|-----------|---------|-------|-------------|
| `fs.file-max` | varies | 2000000 | Max system-wide file handles |
| `fs.nr_open` | 1048576 | 2000000 | Max per-process file handles |
| `fs.inotify.max_user_watches` | 8192 | 524288 | Max inotify watches |
| `fs.inotify.max_user_instances` | 128 | 1024 | Max inotify instances |

## Kernel

| Parameter | Default | Description |
|-----------|---------|-------------|
| `kernel.sched_min_granularity_ns` | 1000000 | Min scheduler timeslice |
| `kernel.sched_wakeup_granularity_ns` | 1000000 | Wakeup granularity |
| `kernel.sched_migration_cost_ns` | 500000 | Migration cost |
| `kernel.sched_autogroup_enabled` | 1 | Auto-grouping (disable for servers) |
| `kernel.pid_max` | 32768 | Max PID value |
| `kernel.threads-max` | varies | Max threads |
| `kernel.perf_event_paranoid` | 2 | Perf access level |

## Security

| Parameter | Default | Description |
|-----------|---------|-------------|
| `kernel.randomize_va_space` | 2 | ASLR (0/1/2) |
| `kernel.dmesg_restrict` | 0 | Restrict dmesg |
| `kernel.kptr_restrict` | 1 | Restrict kernel pointers |
| `kernel.core_pattern` | core | Core dump pattern |
| `fs.suid_dumpable` | 0 | SUID core dumps |

## Quick Copy-Paste Configs

### High-Performance Server

```bash
# /etc/sysctl.d/99-server.conf

# Network
net.ipv4.tcp_congestion_control = bbr
net.core.default_qdisc = fq
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.core.netdev_max_backlog = 65535
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.ip_local_port_range = 1024 65535

# Buffers
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# Memory
vm.swappiness = 10
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5

# File handles
fs.file-max = 2000000
```

### Database Server

```bash
# /etc/sysctl.d/99-database.conf

# Memory (reduce swapping)
vm.swappiness = 1
vm.dirty_ratio = 40
vm.dirty_background_ratio = 10
vm.dirty_expire_centisecs = 500
vm.dirty_writeback_centisecs = 100

# Disable NUMA balancing (manual pinning)
vm.numa_balancing = 0
vm.zone_reclaim_mode = 0

# Disable THP (in /sys, not sysctl)
# echo never > /sys/kernel/mm/transparent_hugepage/enabled

# Large shared memory
kernel.shmmax = 68719476736
kernel.shmall = 4294967296
```

### Low-Latency

```bash
# /etc/sysctl.d/99-low-latency.conf

# Network
net.core.busy_poll = 50
net.core.busy_read = 50
net.ipv4.tcp_fastopen = 3

# Disable autogroup
kernel.sched_autogroup_enabled = 0

# Disable NUMA balancing
vm.numa_balancing = 0
```

## Apply Changes

```bash
# Reload all
sysctl --system

# Reload specific file
sysctl -p /etc/sysctl.d/99-server.conf

# Set temporarily
sysctl -w parameter=value

# Verify
sysctl parameter
```
