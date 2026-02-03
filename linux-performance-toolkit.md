# Linux Performance Toolkit
> Extracted from Brendan Gregg's methodology and tools. Actionable commands, checklists, and diagnostic approaches.

---

## Table of Contents
1. [Methodologies](#methodologies)
2. [USE Method Checklist](#use-method-checklist)
3. [perf Commands](#perf-commands)
4. [eBPF/BPF Tools](#ebpfbpf-tools)
5. [Flame Graphs](#flame-graphs)
6. [Off-CPU Analysis](#off-cpu-analysis)
7. [Load Average Interpretation](#load-average-interpretation)
8. [Tracer Selection Guide](#tracer-selection-guide)

---

## Methodologies

### Primary Methods (Use These)

| Method | When to Use | Core Steps |
|--------|-------------|------------|
| **USE** | Finding resource bottlenecks | For each resource: check Utilization, Saturation, Errors |
| **TSA** | Thread latency analysis | Measure time in each thread state, investigate most frequent first |
| **RED** | Service monitoring | Track Request rate, Errors, Duration |
| **CPU Profile** | Identifying CPU consumption | Generate flame graph, understand code paths >1% |
| **Off-CPU** | Thread wait latencies | Profile off-CPU time with stacks, study by duration |
| **Drill-Down** | Hierarchical systems | Start high-level, examine details, select interesting paths, repeat |

### Anti-Methodologies (Avoid)

- **Streetlight**: Using only familiar tools without systematic approach
- **Random Change**: Altering system attributes haphazardly
- **Traffic Light**: Assuming green dashboards mean no problems
- **Blame-Someone-Else**: Redirecting without investigation

---

## USE Method Checklist

**Principle**: For every resource, check Utilization, Saturation, Errors.

### CPU

| Metric | Command |
|--------|---------|
| Utilization | `vmstat 1`, `mpstat -P ALL 1`, `sar -u 1` |
| Saturation | `vmstat 1` (r column), `sar -q 1` |
| Errors | `dmesg`, `perf stat -e cs` (context switches), MCE logs |

### Memory

| Metric | Command |
|--------|---------|
| Utilization | `free -m`, `vmstat 1` (free column), `sar -r 1` |
| Saturation | `vmstat 1` (si/so columns), `sar -B 1` (pgscank/pgscand) |
| Errors | `dmesg | grep -i "out of memory"`, `sar -B 1` (pgfault) |

### Disk I/O

| Metric | Command |
|--------|---------|
| Utilization | `iostat -xz 1` (%util column) |
| Saturation | `iostat -xz 1` (avgqu-sz), `sar -d 1` |
| Errors | `/sys/devices/.../ioerr_cnt`, `smartctl`, `dmesg` |

### Network

| Metric | Command |
|--------|---------|
| Utilization | `sar -n DEV 1`, `ip -s link` |
| Saturation | `netstat -s` (retransmits), `ss -ti` |
| Errors | `ip -s link` (errors/drops), `netstat -s`, `dmesg` |

### Quick Diagnostic Script

```bash
#!/bin/bash
# USE Method Quick Check
echo "=== CPU ==="
vmstat 1 3
echo -e "\n=== Memory ==="
free -m
vmstat 1 3 | awk '{print $7,$8}'  # si/so
echo -e "\n=== Disk ==="
iostat -xz 1 3
echo -e "\n=== Network ==="
sar -n DEV 1 3 2>/dev/null || ip -s link
```

---

## perf Commands

### CPU Profiling

```bash
# Basic CPU stats
perf stat command
perf stat -d command                    # Detailed with cache events
perf stat -a sleep 5                    # System-wide for 5 seconds

# CPU sampling (99 Hz avoids lockstep bias)
perf record -F 99 command
perf record -F 99 -p PID
perf record -F 99 -a -g -- sleep 10     # All CPUs with call graphs
perf record -F 99 -a --call-graph dwarf sleep 10  # DWARF unwinding

# Real-time profiling
perf top -F 49                          # Live CPU sampling
perf top -F 49 -ns comm,dso            # By command and library
```

### Hardware Counters

```bash
# Cache analysis
perf stat -e cycles,instructions,cache-references,cache-misses -a sleep 10
perf stat -e L1-dcache-loads,L1-dcache-load-misses command
perf stat -e LLC-loads,LLC-load-misses,LLC-stores command

# TLB analysis
perf stat -e dTLB-loads,dTLB-load-misses command

# Cache miss sampling
perf record -e L1-dcache-load-misses -c 10000 -ag -- sleep 5
perf record -e LLC-load-misses -c 100 -ag -- sleep 5
```

### Tracing

```bash
# List available events
perf list
perf list 'sched:*'

# System calls
perf stat -e 'syscalls:sys_enter_*' -p PID
perf stat -e raw_syscalls:sys_enter -I 1000 -a   # Per-second rate

# Scheduler
perf record -e sched:sched_switch -a
perf record -e context-switches -c 1 -a          # Trace all

# Block I/O
perf record -e block:block_rq_issue -ag          # Disk I/O with stacks
perf record -e block:block_rq_complete --filter 'nr_sector > 200'

# New processes
perf record -e sched:sched_process_exec -a

# Page faults
perf record -e page-faults -ag
perf record -e minor-faults -c 1 -ag             # Trace all
```

### Dynamic Probes

```bash
# Add kernel probes
perf probe --add tcp_sendmsg
perf probe --add 'tcp_sendmsg size'              # With variable
perf probe -V tcp_sendmsg                        # Show available variables
perf probe -L tcp_sendmsg                        # Show line numbers
perf probe -d tcp_sendmsg                        # Remove probe
perf probe -l                                    # List probes

# Record with filter
perf record -e probe:tcp_sendmsg --filter 'size > 100' -a

# User-level probes
perf probe -x /lib64/libc.so.6 malloc
```

### Reporting

```bash
perf report                             # Interactive
perf report --stdio                     # Text output
perf report -n                          # Include sample counts
perf report --stdio -n -g folded        # For flame graph processing
perf script                             # List all events
perf script -F comm,pid,tid,cpu,time,event,ip,sym,dso
```

### Flame Graph Generation

```bash
# Record
perf record -F 99 -a -g -- sleep 30

# Generate
perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

### Quick One-Liners

```bash
# CPU hotspots
perf record -F 99 -a -g -- sleep 10 && perf report

# What syscalls is process making?
perf stat -e 'syscalls:sys_enter_*' -p PID

# Trace disk I/O
perf record -e block:block_rq_issue -ag && perf report

# Context switch overhead
perf record -e context-switches -a sleep 10 && perf report

# Memory allocation patterns
perf record -e minor-faults -ag && perf report

# Cache-to-cache false sharing (Linux 4.10+)
perf c2c record -a -- sleep 10 && perf c2c report
```

### Flag Reference

| Flag | Purpose |
|------|---------|
| `-a` | System-wide (all CPUs) |
| `-p PID` | Specific process |
| `-g` | Capture call graphs |
| `-F Hz` | Sampling frequency |
| `-c count` | Sample every Nth event |
| `-e event` | Specify event |
| `--call-graph dwarf` | DWARF stack unwinding |
| `--call-graph lbr` | Last Branch Record stacks |
| `--filter` | Event filtering |
| `-n` | Display sample counts |
| `--stdio` | Text output |
| `-d` | Detailed mode |

---

## eBPF/BPF Tools

### Installation (Ubuntu/Debian)

```bash
sudo apt-get install bpfcc-tools linux-headers-$(uname -r)
# Tools installed to /usr/share/bcc/tools/ or /usr/sbin/*-bpfcc
```

### Essential Single-Purpose Tools

| Tool | Purpose | Example |
|------|---------|---------|
| `execsnoop` | Trace new processes | `execsnoop-bpfcc` |
| `opensnoop` | Trace file opens | `opensnoop-bpfcc` |
| `biolatency` | Block I/O latency histogram | `biolatency-bpfcc` |
| `biosnoop` | Block I/O per-event | `biosnoop-bpfcc` |
| `ext4slower` | Slow ext4 operations | `ext4slower-bpfcc 10` (>10ms) |
| `tcpconnect` | Trace outbound connections | `tcpconnect-bpfcc` |
| `tcpaccept` | Trace inbound connections | `tcpaccept-bpfcc` |
| `tcpretrans` | TCP retransmissions | `tcpretrans-bpfcc` |
| `tcplife` | TCP session summary | `tcplife-bpfcc` |
| `runqlat` | Scheduler run queue latency | `runqlat-bpfcc` |
| `runqlen` | Scheduler run queue length | `runqlen-bpfcc` |
| `profile` | CPU profiler | `profile-bpfcc -F 99 10` |
| `offcputime` | Off-CPU time with stacks | `offcputime-bpfcc 10` |
| `cachestat` | Page cache stats | `cachestat-bpfcc` |
| `cachetop` | Page cache top-style | `cachetop-bpfcc` |
| `filetop` | File reads/writes by process | `filetop-bpfcc` |
| `gethostlatency` | DNS lookup latency | `gethostlatency-bpfcc` |
| `bashreadline` | Trace bash commands | `bashreadline-bpfcc` |

### Multi-Purpose Tools

```bash
# funccount: Count function calls
funccount-bpfcc 'vfs_*'
funccount-bpfcc -i 1 'tcp_send*'

# funclatency: Function latency histogram
funclatency-bpfcc do_sys_open
funclatency-bpfcc -u vfs_read        # Microseconds

# stackcount: Count stack traces to function
stackcount-bpfcc submit_bio

# trace: Custom tracing
trace-bpfcc 'do_sys_open "%s", arg2'
trace-bpfcc 'r::do_sys_open "ret=%d", retval'
```

### bpftrace One-Liners

```bash
# Trace open syscalls
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%d %s\n", pid, str(args->filename)); }'

# Count syscalls by process
bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'

# Read latency histogram
bpftrace -e 'kprobe:vfs_read { @start[tid] = nsecs; }
             kretprobe:vfs_read /@start[tid]/ { @ns = hist(nsecs - @start[tid]); delete(@start[tid]); }'

# Trace process execution
bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("%s -> %s\n", comm, str(args->filename)); }'

# Block I/O size histogram
bpftrace -e 'tracepoint:block:block_rq_issue { @bytes = hist(args->bytes); }'

# Files opened by process
bpftrace -e 'tracepoint:syscalls:sys_enter_openat /comm == "nginx"/ { printf("%s\n", str(args->filename)); }'
```

### BTF/CO-RE (Modern Approach)

For portable tools without Clang/LLVM dependency:
- Requires kernel `CONFIG_DEBUG_INFO_BTF=y`
- Tools compile once, run across kernel versions
- bcc Python tools now deprecated in favor of libbpf C

---

## Flame Graphs

### Generation Workflow

```bash
# 1. Record with perf
perf record -F 99 -a -g -- sleep 30

# 2. Generate folded stacks
perf script | stackcollapse-perf.pl > out.folded

# 3. Create SVG
flamegraph.pl out.folded > flame.svg
```

### Flame Graph Types

| Type | Use Case | Generation |
|------|----------|------------|
| CPU | Where CPU time is spent | `perf record -F 99 -ag` |
| Off-CPU | Where threads block/wait | `offcputime -f 30` |
| Memory | Allocation patterns | `--color=mem` |
| Differential | Compare two profiles | `difffolded.pl` |

### Interpretation

- **Width** = time/frequency (wider = more samples)
- **Y-axis** = stack depth (bottom = entry point)
- **X-axis** = alphabetical sort (NOT time)
- **Color** = random for differentiation (or semantic with options)
- **Top edge** = functions consuming CPU

### Off-CPU Flame Graphs

```bash
# Collect off-CPU stacks
offcputime-bpfcc -f 30 > out.offcpustacks

# Generate with I/O coloring
flamegraph.pl --color=io --countname=us < out.offcpustacks > offcpu.svg
```

### Differential Flame Graphs

```bash
# Profile 1 (before)
perf script > out.stacks1
stackcollapse-perf.pl out.stacks1 > out.folded1

# Profile 2 (after)
perf script > out.stacks2
stackcollapse-perf.pl out.stacks2 > out.folded2

# Generate diff (red=growth, blue=reduction)
difffolded.pl out.folded1 out.folded2 | flamegraph.pl > diff.svg

# Normalize for different sample counts
difffolded.pl -n out.folded1 out.folded2 | flamegraph.pl > diff.svg
```

### Language-Specific Setup

| Language | Requirement |
|----------|-------------|
| Java | `-XX:+PreserveFramePointer` (JDK 8u60+) |
| Node.js | `--perf-basic-prof` or `--perf-prof` |
| Python | py-spy, pyflame |
| Go | Native support |
| C/C++ | Compile with `-fno-omit-frame-pointer` |

---

## Off-CPU Analysis

### Concept

Off-CPU analysis captures time threads spend blocked (not running on CPU):
- I/O waits
- Lock contention
- Sleep calls
- Paging/swapping
- Network waits

Combined with CPU profiling, explains 100% of thread time.

### Tools

```bash
# Off-CPU time with stacks (bcc)
offcputime-bpfcc 30 > out.stacks
offcputime-bpfcc -p PID 30           # Specific process
offcputime-bpfcc -uf 30              # User stacks only, folded

# Off-CPU time histogram
cpudist-bpfcc -O                     # Off-CPU distribution

# Wake-up analysis
offwaketime-bpfcc 30                 # What woke threads
wakeuptime-bpfcc 30                  # Time to wake
```

### Thread State Analysis (TSA)

Investigate states in order of time spent:

| State | What It Means | Tools |
|-------|--------------|-------|
| Executing | Running on CPU | perf, profile |
| Runnable | Waiting for CPU | runqlat, runqlen |
| Sleeping | Blocked on I/O/events | offcputime, trace |
| Lock | Waiting for lock | lockstat, trace |
| Paging | Waiting for memory | vmstat, sar -B |
| Idle | Waiting for work | Application-level |

**Rule**: If Runnable or Paging >10%, investigate immediately.

---

## Load Average Interpretation

### What Linux Load Averages Include

Unlike other Unix systems, Linux includes:
- Threads running on CPU (R state)
- Threads waiting for CPU (R state, runnable)
- Threads in uninterruptible sleep (D state) - disk I/O, locks

### The Three Numbers

```
$ uptime
 14:30  up 3 days,  load average: 1.50, 2.00, 1.75
                                   ^     ^     ^
                                  1min  5min  15min
```

**These are exponentially-damped moving averages**, not simple averages.

### Interpretation Guidelines

| Pattern | Meaning |
|---------|---------|
| 1-min > 5-min > 15-min | Load increasing |
| 1-min < 5-min < 15-min | Load decreasing |
| Value > CPU count | Potential bottleneck (but context matters) |
| High load, low CPU% | I/O or lock contention |

### Better Metrics

```bash
# Per-CPU utilization (better than load average)
mpstat -P ALL 1

# Scheduler latency (how long threads wait)
runqlat-bpfcc

# Pressure Stall Information (Linux 4.20+)
cat /proc/pressure/cpu
cat /proc/pressure/io
cat /proc/pressure/memory
```

---

## Tracer Selection Guide

### Decision Tree

```
Need simple CPU profiling?
  -> perf record -F 99 -ag

Need to count/trace kernel functions?
  -> bcc tools (execsnoop, biolatency, etc.)
  -> bpftrace one-liners

Need custom in-kernel analysis?
  -> bpftrace (simpler)
  -> bcc (more powerful)

Need maximum flexibility?
  -> SystemTap (but more complex)

Need lowest overhead event collection?
  -> LTTng
```

### Tracer Comparison

| Tracer | Strengths | Weaknesses |
|--------|-----------|------------|
| **perf** | Official, well-maintained, CPU profiling | Not programmable in-kernel |
| **bcc/eBPF** | Production-safe, in-kernel aggregation | Requires Linux 4.x+ |
| **bpftrace** | Easy one-liners, rapid prototyping | Less flexible than bcc |
| **ftrace** | Built-in, no dependencies | Fiddly interface |
| **SystemTap** | Most powerful | Safety concerns, needs debuginfo |
| **LTTng** | Fastest, safest | No in-kernel programming |

### Recommended Learning Order

1. **perf** - CPU profiling, flame graphs
2. **bcc tools** - Run pre-built tools (execsnoop, biolatency)
3. **bpftrace** - Custom one-liners
4. **bcc development** - Build custom tools

---

## Quick Reference Cards

### 60-Second Analysis

```bash
uptime                    # Load averages
dmesg -T | tail          # Kernel errors
vmstat 1 5               # Virtual memory
mpstat -P ALL 1 5        # Per-CPU
pidstat 1 5              # Per-process
iostat -xz 1 5           # Disk I/O
free -m                  # Memory
sar -n DEV 1 5          # Network I/O
sar -n TCP,ETCP 1 5     # TCP stats
```

### CPU Analysis

```bash
# Utilization
mpstat -P ALL 1
top -1                    # Press '1' for per-CPU

# Profiling
perf record -F 99 -ag -- sleep 10
perf report

# Flame graph
perf script | stackcollapse-perf.pl | flamegraph.pl > cpu.svg

# Scheduler latency
runqlat-bpfcc
```

### Memory Analysis

```bash
# Utilization
free -m
vmstat 1

# Saturation (swapping)
vmstat 1                  # si/so columns
sar -B 1                  # pgscank, pgscand

# Page cache
cachestat-bpfcc
cachetop-bpfcc

# Per-process
ps aux --sort=-%mem | head
pidstat -r 1
```

### Disk I/O Analysis

```bash
# Utilization
iostat -xz 1

# Latency histogram
biolatency-bpfcc

# Per-event tracing
biosnoop-bpfcc

# Slow operations
ext4slower-bpfcc 10       # >10ms operations
```

### Network Analysis

```bash
# Interface stats
sar -n DEV 1
ip -s link

# TCP connections
tcpconnect-bpfcc         # Outbound
tcpaccept-bpfcc          # Inbound
tcplife-bpfcc            # Session summary

# Retransmissions
tcpretrans-bpfcc
ss -ti                    # Per-socket

# DNS latency
gethostlatency-bpfcc
```

---

## Resources

- **FlameGraph tools**: https://github.com/brendangregg/FlameGraph
- **bcc tools**: https://github.com/iovisor/bcc
- **bpftrace**: https://github.com/iovisor/bpftrace
- **perf wiki**: https://perf.wiki.kernel.org
- **Brendan Gregg's site**: https://www.brendangregg.com
