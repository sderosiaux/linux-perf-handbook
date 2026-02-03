# One-Liners Cheatsheet

Quick diagnostic commands for common scenarios.

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

## CPU

```bash
# High CPU process
ps aux --sort=-%cpu | head -10

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
```

## Memory

```bash
# Memory summary
free -wh

# Top memory processes
ps aux --sort=-%mem | head -10

# Memory by process (PSS)
smem -tk -s pss

# Buffer/cache breakdown
cat /proc/meminfo | grep -E 'Buffers|Cached|Slab'

# Swap usage by process
for f in /proc/*/status; do awk '/VmSwap|Name/{printf $2 " "}END{print ""}' $f 2>/dev/null; done | sort -k2 -n | tail -10

# OOM events
dmesg | grep -i "out of memory"

# Huge pages
grep Huge /proc/meminfo
```

## Disk I/O

```bash
# I/O wait
iostat -xz 1 | awk '$1 !~ /^[a-z]/ {print $1, $14}'

# Top I/O processes
iotop -b -n 1 -o

# Disk latency
iostat -x 1 | awk 'NR>3 {print $1, $10, $11}'

# I/O per process
pidstat -d 1

# Slow I/O operations
biolatency-bpfcc -D

# File access tracing
opensnoop-bpfcc -p PID
```

## Network

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
```

## Processes

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

# Trace process syscalls
strace -c -p PID
```

## System

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

# File handle usage
cat /proc/sys/fs/file-nr
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
netstat -s | grep -i retrans

# Are there disk errors?
dmesg | grep -i error | tail
```

## eBPF One-Liners

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
```

## bpftrace One-Liners

```bash
# Syscall count by process
bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'

# Read size distribution
bpftrace -e 'tracepoint:syscalls:sys_exit_read /args->ret > 0/ { @bytes = hist(args->ret); }'

# Process exec
bpftrace -e 'tracepoint:sched:sched_process_exec { printf("%s -> %s\n", comm, str(args->filename)); }'

# Block I/O by device
bpftrace -e 'tracepoint:block:block_rq_complete { @[args->dev] = count(); }'
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
```

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

# Native memory
jcmd PID VM.native_memory summary
```

## Find Problems Fast

```bash
# Why is the system slow?
vmstat 1 3; iostat -xz 1 3; free -m

# Why is the process slow?
strace -c -p PID; timeout 30 perf record -g -p PID; perf report

# Why is the network slow?
ss -ti | head; mtr -rw -c 5 TARGET

# Why is disk slow?
iostat -xz 1; biolatency-bpfcc

# Why is memory high?
smem -tk; ps aux --sort=-%mem | head
```

## Tuning Verification

```bash
# Check sysctl values
sysctl net.ipv4.tcp_congestion_control
sysctl vm.swappiness
sysctl net.core.somaxconn

# Check CPU governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# Check I/O scheduler
cat /sys/block/sda/queue/scheduler

# Check THP
cat /sys/kernel/mm/transparent_hugepage/enabled

# Check NUMA
numactl --hardware
```
