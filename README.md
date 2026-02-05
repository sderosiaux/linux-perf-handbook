# Linux Perf Handbook

Performance engineering, debugging, and system tuning reference.

**Target Kernel:** Linux 6.6+ (EEVDF scheduler, modern eBPF features)
**Last Updated:** 2026-02

## How to Use This Handbook

1. **Start with the 60-second checklist** below for quick triage
2. **Use the Quick Navigation** to find specific topics
3. **Check the cheatsheets** for copy-paste commands
4. **Refer to curated sources** for deep dives on specific areas

For investigation workflow:
```
Symptom → 60-Second Analysis → Identify resource bottleneck → Deep dive chapter
```

## Quick Navigation

| Chapter | Description |
|---------|-------------|
| [01 - Modern CLI Replacements](01-modern-cli-replacements.md) | Rust/Go tools replacing classic Unix utils |
| [02 - System Monitoring](02-system-monitoring.md) | CPU, memory, process monitoring |
| [03 - Network Analysis](03-network-analysis.md) | Traffic analysis, DNS, HTTP testing |
| [04 - Disk & Storage](04-disk-storage.md) | I/O benchmarking, filesystem tools |
| [05 - Performance Profiling](05-performance-profiling.md) | perf, flame graphs, profilers |
| [06 - eBPF & Tracing](06-ebpf-tracing.md) | BCC tools, bpftrace, ftrace, sched_ext |
| [07 - Containers & K8s](07-containers-k8s.md) | Docker, Kubernetes debugging |
| [08 - Kernel Tuning](08-kernel-tuning.md) | sysctl, EEVDF scheduler, memory, I/O |
| [09 - Network Tuning](09-network-tuning.md) | TCP, BBR, io_uring, NUMA networking |
| [10 - Java/JVM](10-java-jvm.md) | JVM profiling and tuning |
| [11 - GPU & HPC](11-gpu-hpc.md) | GPU profiling, MIG, HPC tracing |
| [12 - Observability & Metrics](12-observability-metrics.md) | Prometheus, Grafana, OpenTelemetry |
| [13 - Latency Analysis](13-latency-analysis.md) | Tail latency, coordinated omission, P99 |
| [14 - Database Profiling](14-database-profiling.md) | PostgreSQL, MySQL, query optimization |
| [Database Production Debugging](database-production-debugging.md) | Hot partitions, cache pollution, admission control, lock analysis |
| [15 - Memory Subsystem](15-memory-subsystem.md) | NUMA, huge pages, memory profiling |
| [16 - Scheduler & Interrupts](16-scheduler-interrupts.md) | CPU scheduling, context switching |
| [Scheduler Debugging Deep Dive](scheduler-debugging-deep-dive.md) | CFS bugs, run queue attribution, Perfetto, noisy neighbor detection |
| [17 - Ftrace Production](17-ftrace-production.md) | Function tracing, trace-cmd |
| [18 - VDSO & Clock Source](18-vdso-clock-source-tuning.md) | Time syscalls, TSC, cloud VM performance |
| [18 - Off-CPU Analysis](18-off-cpu-analysis.md) | Wall-clock profiling, blocking detection, load-scaling bottlenecks |
| [19 - Storage Engine Patterns](19-storage-engine-patterns.md) | LMDB, RocksDB, LSM, columnar, vectorized engines |

### Advanced Topics & Production Patterns

| Guide | Description |
|-------|-------------|
| [Coordinated Omission Guide](coordinated-omission-guide.md) | Load testing correctness, wrk2, timestamp injection |
| [eBPF Performance Overhead](ebpf-performance-overhead-guide.md) | Hook overhead, map types, production deployment |
| [Container Debugging Patterns](container-debugging-patterns.md) | cAdvisor, GOMAXPROCS, PSI, cgroup v2 debugging |
| [TCP Edge Cases & Load Balancers](tcp-edge-cases-and-load-balancer-behavior.md) | SYN retry, LB buffering, timeout hierarchies |
| [Bryan Cantrill Debugging Methodology](bryan-cantrill-debugging-methodology.md) | Questions-first, state preservation, systematic elimination |

### Cheatsheets

| Cheatsheet | Description |
|------------|-------------|
| [One-Liners](cheatsheets/one-liners.md) | Quick diagnostic commands by problem type |
| [Sysctl Reference](cheatsheets/sysctl-reference.md) | Key kernel parameters |
| [VDSO/Clock Troubleshooting](cheatsheets/vdso-clock-troubleshooting.md) | Quick detection and fixes for time-related performance |

## 60-Second Analysis

From [Brendan Gregg's Linux Performance Analysis in 60,000 Milliseconds](http://techblog.netflix.com/2015/11/linux-performance-analysis-in-60s.html):

```bash
uptime                           # load averages
dmesg | tail                     # kernel errors
vmstat 1                         # system-wide stats
mpstat -P ALL 1                  # CPU balance
pidstat 1                        # process CPU
iostat -xz 1                     # disk I/O
free -m                          # memory usage
sar -n DEV 1                     # network I/O
sar -n TCP,ETCP 1               # TCP stats
top                              # overview
```

## Tool Categories

### Classic -> Modern Replacements

| Classic | Modern | Why |
|---------|--------|-----|
| `ls` | `eza` | Git status, icons, tree view |
| `cat` | `bat` | Syntax highlighting |
| `find` | `fd` | 5x faster, simpler syntax |
| `grep` | `ripgrep` | 10x faster, .gitignore aware |
| `du` | `dust` | Visual bars |
| `df` | `duf` | Clean tables |
| `top` | `btop` | Dashboard UI |
| `dig` | `dog`/`doggo` | DoH/DoT, colors |
| `curl` | `xh` | Human-friendly HTTP |
| `sed` | `sd` | Sane regex |
| `cd` | `zoxide` | Frecency-based jump |

### Performance Stack

```
Application  ->  async-profiler (Java), py-spy (Python), rbspy (Ruby)
     |
Userspace    ->  perf, valgrind, heaptrack
     |
Syscalls     ->  strace, ltrace
     |
Kernel       ->  eBPF/BCC, bpftrace, ftrace
     |
Hardware     ->  perf stat, turbostat, intel_gpu_top
```

### Network Stack

```
L7 (HTTP)    ->  httpie, xh, hey, wrk, k6, vegeta
     |
L4 (TCP)     ->  ss, netstat, tcpdump, termshark
     |
L3 (IP)      ->  mtr, traceroute, ping, gping
     |
L2 (Link)    ->  ethtool, ip link
```

## Quick Install (Debian/Ubuntu)

```bash
# Modern CLI tools
sudo apt install ripgrep fd-find bat eza fzf btop git-delta zoxide duf gping

# Performance tools
sudo apt install linux-tools-common linux-tools-$(uname -r) bpfcc-tools bpftrace

# Network tools
sudo apt install mtr-tiny tcpdump nmap iperf3 netcat-openbsd

# Monitoring
sudo apt install sysstat htop iotop
```

## Version Requirements

| Component | Minimum | Recommended | Notes |
|-----------|---------|-------------|-------|
| Linux Kernel | 5.15 | 6.6+ | EEVDF scheduler, modern eBPF |
| bpftrace | 0.16 | 0.20+ | BTF support, newer features |
| bcc-tools | 0.25 | 0.28+ | Latest BPF features |
| perf | matches kernel | - | Install linux-tools-$(uname -r) |
| iproute2 | 6.0 | 6.7+ | netkit, newer tc features |

### Key Kernel Features by Version

| Version | Feature |
|---------|---------|
| 5.15+ | io_uring maturity, BTF by default |
| 6.1+ | MGLRU, improved memory management |
| 6.4+ | Per-VMA locks, reduced mmap contention |
| 6.6+ | EEVDF scheduler (replaces CFS) |
| 6.7+ | Netkit stable |
| 6.9+ | BPF Arena, new kfuncs |
| 6.12+ | sched_ext merged, PREEMPT_RT mainline |

## Curated Sources

### Essential Reading

| Source | Focus | Link |
|--------|-------|------|
| Brendan Gregg | Performance methodology, eBPF | [brendangregg.com](https://www.brendangregg.com/) |
| Julia Evans | Debugging, Linux internals | [jvns.ca](https://jvns.ca/) |
| Netflix Tech Blog | Production performance | [netflixtechblog.com](https://netflixtechblog.com/) |
| Cloudflare Blog | Network performance, eBPF | [blog.cloudflare.com](https://blog.cloudflare.com/) |
| Meta Engineering | eBPF at scale, kernel | [engineering.fb.com](https://engineering.fb.com/) |
| Dan Luu | Systems analysis, measurement | [danluu.com](https://danluu.com/) |

### In This Repository

Curated extracts from these sources with actionable insights:

- [Netflix Performance Playbook](netflix-performance-playbook.md) - ZGC, flame graphs, load shedding
- [Cloudflare Network Performance](cloudflare-network-performance.md) - TCP tuning, XDP, DDoS
- [Meta eBPF & Systems Engineering](meta-ebpf-systems-engineering.md) - Katran, Strobelight, sched_ext
- [Dan Luu Systems Insights](dan-luu-systems-insights.md) - Latency, measurement, caching
- [Julia Evans Systems Debugging](julia-evans-systems-debugging.md) - strace, debugging methodology

### Tools & References

- [BCC Tools](https://github.com/iovisor/bcc) - eBPF-based Linux tools
- [bpftrace](https://github.com/bpftrace/bpftrace) - High-level tracing language
- [perf-tools](https://github.com/brendangregg/perf-tools) - Performance analysis tools
- [FlameGraph](https://github.com/brendangregg/FlameGraph) - Stack trace visualizer
- [sched_ext](https://github.com/sched-ext/scx) - BPF schedulers

## Resources

- [Brendan Gregg's Linux Performance](http://www.brendangregg.com/linuxperf.html)
- [Linux Tracing in 15 Minutes](http://www.brendangregg.com/blog/2016-12-27/linux-tracing-in-15-minutes.html)
- [BCC Tools](https://github.com/iovisor/bcc)
- [perf-tools](https://github.com/brendangregg/perf-tools)
