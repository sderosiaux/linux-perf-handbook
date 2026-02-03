# Curated System Tools

Linux performance, debugging, and system administration toolkit.

## Quick Navigation

| Category | Description |
|----------|-------------|
| [Modern CLI Replacements](01-modern-cli-replacements.md) | Rust/Go tools replacing classic Unix utils |
| [System Monitoring](02-system-monitoring.md) | CPU, memory, process monitoring |
| [Network Analysis](03-network-analysis.md) | Traffic analysis, DNS, HTTP testing |
| [Disk & Storage](04-disk-storage.md) | I/O benchmarking, filesystem tools |
| [Performance Profiling](05-performance-profiling.md) | perf, flame graphs, profilers |
| [eBPF & Tracing](06-ebpf-tracing.md) | BCC tools, bpftrace, ftrace |
| [Containers & K8s](07-containers-k8s.md) | Docker, Kubernetes debugging |
| [Kernel Tuning](08-kernel-tuning.md) | sysctl, CPU, memory, I/O tuning |
| [Network Tuning](09-network-tuning.md) | TCP, congestion control, NUMA |
| [Java/JVM](10-java-jvm.md) | JVM profiling and tuning |

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

### Classic → Modern Replacements

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
Application  →  async-profiler (Java), py-spy (Python), rbspy (Ruby)
     ↓
Userspace    →  perf, valgrind, heaptrack
     ↓
Syscalls     →  strace, ltrace
     ↓
Kernel       →  eBPF/BCC, bpftrace, ftrace
     ↓
Hardware     →  perf stat, turbostat, intel_gpu_top
```

### Network Stack

```
L7 (HTTP)    →  httpie, xh, hey, wrk, k6, vegeta
     ↓
L4 (TCP)     →  ss, netstat, tcpdump, termshark
     ↓
L3 (IP)      →  mtr, traceroute, ping, gping
     ↓
L2 (Link)    →  ethtool, ip link
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

## Resources

- [Brendan Gregg's Linux Performance](http://www.brendangregg.com/linuxperf.html)
- [Linux Tracing in 15 Minutes](http://www.brendangregg.com/blog/2016-12-27/linux-tracing-in-15-minutes.html)
- [BCC Tools](https://github.com/iovisor/bcc)
- [perf-tools](https://github.com/brendangregg/perf-tools)
