# One-Liners Cheatsheet

Quick diagnostic commands for common scenarios. Organized by problem type with modern tool alternatives.

**Related chapters:** [Performance Profiling](../05-performance-profiling.md) | [eBPF & Tracing](../06-ebpf-tracing.md) | [Kernel Tuning](../08-kernel-tuning.md)

## First 60 Seconds

```bash
uptime
dmesg | tail
vmstat 1 5
mpstat -P ALL 1 3
pidstat 1 3
iostat -xz 1 3
free -m
sar -n DEV 1 3
sar -n TCP,ETCP 1 3
top -b -n 1
```

## CPU Problems

**Symptoms:** High load average, slow response, spinning processes

```bash
# High CPU process
ps aux --sort=-%cpu | head -10
# Modern: procs (if installed)

# Per-core utilization
mpstat -P ALL 1

# Context switches
vmstat 1 | awk '{print $12, $13}'

# Run queue length
vmstat 1 | awk '{print $1}'

# Load average over time
sar -q 1

# CPU stealing (VM)
vmstat 1 | awk '{print $15}'

# Which functions are hot (requires root)
sudo perf top -g
```

**eBPF alternatives:** See [eBPF & Tracing](../06-ebpf-tracing.md)
```bash
runqlat-bpfcc          # Run queue latency histogram
cpudist-bpfcc          # On-CPU time distribution
offcputime-bpfcc -p PID  # Where is time spent waiting?
```

## Memory Problems

**Symptoms:** OOM kills, swap activity, slow allocations

```bash
# Memory summary
free -wh

# Top memory processes
ps aux --sort=-%mem | head -10

# Memory by process (PSS - actual memory used)
smem -tk -s pss

# Buffer/cache breakdown
cat /proc/meminfo | grep -E 'Buffers|Cached|Slab'

# Swap usage by process
for f in /proc/*/status; do awk '/VmSwap|Name/{printf $2 " "}END{print ""}' $f 2>/dev/null; done | sort -k2 -n | tail -10

# OOM events
dmesg | grep -i "out of memory"

# Huge pages
grep Huge /proc/meminfo

# Memory pressure (kernel 4.20+)
cat /proc/pressure/memory
```

**eBPF alternatives:**
```bash
memleak-bpfcc -p PID   # Track allocations not freed
oomkill-bpfcc          # Trace OOM kills in real time
```

## Disk I/O Problems

**Symptoms:** High iowait, slow file operations, disk queues

```bash
# I/O wait percentage
iostat -xz 1 | awk '$1 !~ /^[a-z]/ {print $1, $14}'

# Top I/O processes
iotop -b -n 1 -o

# Disk latency (await = avg wait time in ms)
iostat -x 1 | awk 'NR>3 {print $1, $10, $11}'

# I/O per process
pidstat -d 1

# Which files are being accessed
lsof +D /path/to/directory

# I/O pressure (kernel 4.20+)
cat /proc/pressure/io
```

**eBPF alternatives:**
```bash
biolatency-bpfcc -D    # Block I/O latency by device
biosnoop-bpfcc         # Per-I/O tracing
fileslower-bpfcc 10    # Files taking >10ms
opensnoop-bpfcc -p PID # What files is process opening?
ext4slower-bpfcc       # Slow ext4 operations
```

## Network Problems

**Symptoms:** Connection failures, slow transfers, drops

```bash
# Listening ports
ss -tulpn

# Established connections
ss -tunap | grep ESTAB

# Connections per state
ss -tan | awk 'NR>1 {print $1}' | sort | uniq -c

# Top bandwidth processes
nethogs -t -c 3

# Packet errors
ip -s link | grep -A 2 errors

# TCP retransmits
ss -ti | grep retrans

# Connection tracking
conntrack -C

# DNS resolution time
dig +stats example.com | grep "Query time"

# Network pressure (kernel 4.20+)
# Note: No /proc/pressure/net exists; use ss -ti for backpressure
```

**eBPF alternatives:**
```bash
tcpconnect-bpfcc       # Trace new TCP connections
tcpaccept-bpfcc        # Trace TCP accepts
tcpretrans-bpfcc       # Trace retransmissions
tcpdrop-bpfcc          # Trace dropped packets
tcprtt-bpfcc           # TCP RTT distribution
```

## Process Problems

**Symptoms:** Hung processes, resource leaks, spawn storms

```bash
# Process tree
ps fauxww

# Threads of process
ps -T -p PID

# Open files by process
lsof -p PID | wc -l

# File descriptors
ls -la /proc/PID/fd | wc -l

# Process limits
cat /proc/PID/limits

# Process memory map
pmap -x PID

# Trace process syscalls (high overhead)
strace -c -p PID

# What is process waiting for?
cat /proc/PID/wchan
cat /proc/PID/stack  # requires root
```

**eBPF alternatives:**
```bash
execsnoop-bpfcc        # Trace new processes
exitsnoop-bpfcc        # Trace process exits
```

## System Problems

**Symptoms:** Kernel errors, service failures, resource exhaustion

```bash
# Kernel errors
dmesg -T --level=err,warn | tail -20

# System uptime/load
uptime

# Recent logins
last -n 10

# Failed services
systemctl --failed

# Disk space
df -h | grep -v tmpfs

# Inode usage
df -i | grep -v tmpfs

# File handle usage (allocated, free, max)
cat /proc/sys/fs/file-nr

# Kernel version and features
uname -a
cat /proc/version
```

## Performance Quick Checks

```bash
# Is CPU the bottleneck?
vmstat 1 | awk '$1 > 0 {print "runqueue:", $1}'

# Is I/O the bottleneck?
vmstat 1 | awk '$16 > 20 {print "iowait:", $16}'

# Is memory the bottleneck?
vmstat 1 | awk '$7 > 0 || $8 > 0 {print "swapping:", $7, $8}'

# Are there network issues?
ss -s | grep -i retrans

# Are there disk errors?
dmesg | grep -i error | tail
```

## eBPF One-Liners

**Requires:** bcc-tools package. See [eBPF & Tracing](../06-ebpf-tracing.md) for details.

```bash
# New processes
execsnoop-bpfcc

# File opens
opensnoop-bpfcc

# TCP connections
tcpconnect-bpfcc

# Block I/O latency
biolatency-bpfcc

# Run queue latency
runqlat-bpfcc

# Function counting
funccount-bpfcc 'vfs_*' -i 1

# Off-CPU analysis
offcputime-bpfcc -df -p PID 30 | flamegraph.pl > offcpu.svg
```

## bpftrace One-Liners

**Requires:** bpftrace 0.16+. See [eBPF & Tracing](../06-ebpf-tracing.md) for details.

```bash
# Syscall count by process
bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'

# Read size distribution
bpftrace -e 'tracepoint:syscalls:sys_exit_read /args->ret > 0/ { @bytes = hist(args->ret); }'

# Process exec
bpftrace -e 'tracepoint:sched:sched_process_exec { printf("%s -> %s\n", comm, str(args->filename)); }'

# Block I/O by device
bpftrace -e 'tracepoint:block:block_rq_complete { @[args->dev] = count(); }'

# TCP accepts by port
bpftrace -e 'kretprobe:inet_csk_accept { @[retval->__sk_common.skc_num] = count(); }'

# VFS latency histogram
bpftrace -e 'kprobe:vfs_read { @start[tid] = nsecs; } kretprobe:vfs_read /@start[tid]/ { @us = hist((nsecs - @start[tid]) / 1000); delete(@start[tid]); }'
```

## Container Quick Checks

```bash
# Container CPU/memory
docker stats --no-stream

# Container processes
docker top CONTAINER

# Container logs (last 100 + follow)
docker logs --tail 100 -f CONTAINER

# Container inspect (IP)
docker inspect -f '{{.NetworkSettings.IPAddress}}' CONTAINER

# K8s pod resources
kubectl top pods -A --sort-by=cpu

# K8s events
kubectl get events --sort-by='.lastTimestamp' | tail -20

# K8s pod logs
kubectl logs -f POD --tail=100

# Enter container namespace for debugging
nsenter -t $(docker inspect -f '{{.State.Pid}}' CONTAINER) -n ss -tulpn
```

See [Containers & K8s](../07-containers-k8s.md) for more container debugging.

## Java Quick Checks

```bash
# GC activity
jstat -gcutil PID 1000 5

# Thread dump
jstack -l PID

# Heap usage
jmap -heap PID

# Top heap consumers
jmap -histo PID | head -20

# Native memory (requires -XX:NativeMemoryTracking=summary)
jcmd PID VM.native_memory summary

# Flight recording (JDK 11+)
jcmd PID JFR.start duration=60s filename=recording.jfr
```

See [Java/JVM](../10-java-jvm.md) for JVM tuning and profiling.

## Find Problems Fast

```bash
# Why is the system slow?
vmstat 1 3; iostat -xz 1 3; free -m

# Why is the process slow? (30s profile)
timeout 30 perf record -g -p PID && perf report

# Why is the network slow?
ss -ti | head; mtr -rw -c 5 TARGET

# Why is disk slow?
iostat -xz 1; biolatency-bpfcc

# Why is memory high?
smem -tk; ps aux --sort=-%mem | head
```

## Tuning Verification

Verify your [Kernel Tuning](../08-kernel-tuning.md) settings are applied.

```bash
# Check sysctl values
sysctl net.ipv4.tcp_congestion_control
sysctl vm.swappiness
sysctl net.core.somaxconn

# Check CPU governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# Check I/O scheduler
cat /sys/block/sda/queue/scheduler
cat /sys/block/nvme0n1/queue/scheduler  # NVMe

# Check THP
cat /sys/kernel/mm/transparent_hugepage/enabled

# Check NUMA
numactl --hardware

# Check scheduler (6.6+ EEVDF vs CFS)
dmesg | grep -i scheduler
cat /sys/kernel/debug/sched/features 2>/dev/null

# Check sched_ext (6.12+)
cat /sys/kernel/sched_ext/root/ops 2>/dev/null
```

## Modern Tool Equivalents

| Classic | Modern | Install |
|---------|--------|---------|
| `top` | `btop`, `htop` | `apt install btop` |
| `ps aux` | `procs` | cargo install |
| `netstat` | `ss` | built-in |
| `ifconfig` | `ip addr` | built-in |
| `iotop` | `iotop-c` | `apt install iotop` |
| `lsof` | `lsof` | still best option |
| `strace` | `perf trace` | lower overhead |
| `tcpdump` | `termshark` | TUI interface |
| `du -sh` | `dust` | visual bars |
| `df -h` | `duf` | cleaner output |

## Quick Reference by Symptom

| Symptom | First Command | Deep Dive |
|---------|---------------|-----------|
| High load | `uptime; vmstat 1` | `mpstat -P ALL 1` |
| OOM kills | `dmesg \| grep -i oom` | `memleak-bpfcc` |
| Slow disk | `iostat -xz 1` | `biolatency-bpfcc` |
| Network drops | `ss -s` | `tcpdrop-bpfcc` |
| Process hang | `cat /proc/PID/stack` | `offcputime-bpfcc` |
| High syscalls | `strace -c -p PID` | `bpftrace` |
| Memory leak | `smem -tk` | `memleak-bpfcc -p PID` |
| Connection issues | `ss -tunap` | `tcpconnect-bpfcc` |
