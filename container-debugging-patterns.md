# Container-Specific Debugging Patterns & Gotchas

Production-ready debugging commands, detection patterns, and fixes for containerized environments. Focus on actionable diagnostics for LLM-assisted troubleshooting.

## Table of Contents

1. [cAdvisor File Walking CPU Spikes](#cadvisor-file-walking-cpu-spikes)
2. [Go GOMAXPROCS vs Kubernetes CPU Limits](#go-gomaxprocs-vs-kubernetes-cpu-limits)
3. [DHCP Socket Overhead with AF_PACKET](#dhcp-socket-overhead-with-af_packet)
4. [Container Scheduler Throttling Patterns](#container-scheduler-throttling-patterns)
5. [cgroup v2 Debugging Commands](#cgroup-v2-debugging-commands)
6. [PSI (Pressure Stall Information) Interpretation](#psi-pressure-stall-information-interpretation)
7. [Quick Decision Trees](#quick-decision-trees)

---

## cAdvisor File Walking CPU Spikes

### Problem

Kubelet's cAdvisor component walks container ephemeral filesystem layers every 60 seconds to gather metrics. With large file counts (npm node_modules, Python site-packages, Java JARs), this causes periodic CPU spikes.

### Detection

**Symptom**: Periodic CPU spikes on kubelet process, exactly every 60 seconds.

```bash
# Monitor kubelet CPU usage
pidstat -p $(pgrep kubelet) 1 60

# Flame graph during spike
perf record -F 99 -p $(pgrep kubelet) -g -- sleep 65
perf script | ./FlameGraph/stackcollapse-perf.pl | ./FlameGraph/flamegraph.pl > kubelet.svg

# Look for: fs_get_your_usage, filepath.Walk, or cadvisor/fs calls
grep -E "fs_get_your_usage|filepath.Walk|cadvisor" kubelet.svg

# Count files in container layers
CONTAINER_ID="abc123"
nsenter -t $(docker inspect -f '{{.State.Pid}}' $CONTAINER_ID) -m \
  find / -xdev -type f 2>/dev/null | wc -l
# Red flag: >100K files
```

**Direct cAdvisor metrics check:**
```bash
# cAdvisor runs on kubelet at :10255/metrics (older versions) or :10250/metrics/cadvisor
curl -s localhost:10250/metrics/cadvisor | grep container_fs

# Or check kubelet logs for slow filesystem walks
journalctl -u kubelet | grep -i "filesystem.*slow\|cadvisor.*duration"
```

### Root Cause

cAdvisor uses `du` equivalent operations to measure container disk usage. Each walk:
- Stats every file (syscall overhead)
- Traverses overlay/aufs layers (cache-unfriendly)
- Blocks on slow storage (NFS mounts, network volumes)

### Mitigation

**Option 1: Reduce file count (application level)**
```dockerfile
# Multi-stage builds: only copy runtime dependencies
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:18-slim
COPY --from=builder /app/node_modules ./node_modules
COPY . .
# Result: 80-90% fewer files in final image
```

**Option 2: Increase cAdvisor collection interval**
```yaml
# kubelet config (increase housekeeping interval)
# /var/lib/kubelet/config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
# This increases housekeeping interval indirectly by reducing GC frequency
```

**Option 3: Disable filesystem metrics (if not needed)**
```bash
# kubelet flag (requires restart)
--cadvisor-port=0  # Disables cAdvisor HTTP endpoint
# Note: Metrics still collected, just not exposed

# Or use --housekeeping-interval flag (deprecated, but works)
--housekeeping-interval=2m  # Default 1m
```

**Option 4: Use .dockerignore aggressively**
```.dockerignore
node_modules
.git
*.log
*.md
test/
docs/
.pytest_cache
__pycache__
*.pyc
```

### Verification

```bash
# After mitigation, confirm reduced CPU spikes
pidstat -p $(pgrep kubelet) 1 300 | awk '{if($7>10) print}'
# Should see fewer/smaller spikes

# Check container file count reduction
docker exec $CONTAINER_ID find / -xdev -type f 2>/dev/null | wc -l
```

---

## Go GOMAXPROCS vs Kubernetes CPU Limits

### Problem

Go runtime sets GOMAXPROCS to host core count at startup, not the container's CPU limit. This causes quota exhaustion and throttling.

**Example scenario:**
- Host: 4 cores
- Container: 1 CPU limit (`--cpus=1`)
- Go sets `GOMAXPROCS=4`
- Go spawns 4 OS threads
- All 4 threads try to run simultaneously
- Exhaust 100ms quota in 25ms
- Throttled for remaining 75ms → **75ms latency spikes every 100ms period**

### Detection

```bash
# Inside Go container, check GOMAXPROCS
docker exec $CONTAINER_ID go env GOMAXPROCS
# Output: 4 (host cores)

# Check container CPU limit (cgroup v2)
docker exec $CONTAINER_ID cat /sys/fs/cgroup/cpu.max
# Output: 100000 100000  (100ms quota / 100ms period = 1 CPU)

# Mismatch detected: GOMAXPROCS > CPU limit

# Check for throttling evidence
CGROUP_PATH="/sys/fs/cgroup/system.slice/docker-${CONTAINER_ID}.scope"
cat $CGROUP_PATH/cpu.stat
# nr_throttled: 12345   ← Non-zero = throttling
# throttled_usec: 567890123  ← Growing over time
```

**Symptom correlation:**
```bash
# Monitor throttling and latency together
while true; do
  throttled=$(awk '/nr_throttled/ {print $2}' $CGROUP_PATH/cpu.stat)
  echo "$(date +%T) throttled=$throttled"
  sleep 1
done

# In separate terminal, monitor app latency
# Expect latency spikes correlating with throttle increments
```

### Root Cause

Go scheduler uses `GOMAXPROCS` to determine parallelism. With `GOMAXPROCS=4` but only 1 CPU quota:
- Go creates 4 P (processor) contexts
- Each P wants to run
- All 4 compete for 1 CPU's worth of quota
- Burst consumption exhausts quota early in period
- CFS throttles remaining period

This is **not visible in average CPU usage** - shows 100% of 1 CPU, but delivered in bursts with gaps.

### Fix

**Option 1: Use automaxprocs (recommended)**
```go
package main

import (
    _ "go.uber.org/automaxprocs"
    // ^ Import automatically adjusts GOMAXPROCS at init()
)

func main() {
    // GOMAXPROCS now matches cgroup CPU limit
    // ...
}
```

**Verification:**
```bash
# Check logs for automaxprocs adjustment
docker logs $CONTAINER_ID 2>&1 | grep -i maxprocs
# Output: maxprocs: Updating GOMAXPROCS=1: determined from CPU quota
```

**Option 2: Set GOMAXPROCS manually**
```dockerfile
# Dockerfile
ENV GOMAXPROCS=1
```

Or in Kubernetes:
```yaml
env:
- name: GOMAXPROCS
  value: "1"  # Must match CPU limit
```

**Option 3: Remove CPU limits (use requests only)**
```yaml
resources:
  requests:
    cpu: "1"
  # No limits specified - Go can use host cores when available
  # Trade-off: Potential noisy neighbor issues
```

### Verification After Fix

```bash
# Inside container after fix
docker exec $CONTAINER_ID go env GOMAXPROCS
# Output: 1 (matches CPU limit)

# Check throttling disappears
watch -n 1 'cat $CGROUP_PATH/cpu.stat | grep throttled'
# nr_throttled should stop increasing
# throttled_usec should stabilize

# Monitor latency improvement
# Expect p99 latency spikes to disappear
```

### Advanced: Detection Script

```bash
#!/bin/bash
# go-quota-mismatch-detector.sh

for container in $(docker ps -q); do
  # Get GOMAXPROCS
  gomaxprocs=$(docker exec $container sh -c 'echo $GOMAXPROCS' 2>/dev/null || echo "unknown")

  # Get CPU limit
  cgroup_path="/sys/fs/cgroup/system.slice/docker-${container}.scope"
  if [ -f "$cgroup_path/cpu.max" ]; then
    cpu_max=$(cat $cgroup_path/cpu.max)
    quota=$(echo $cpu_max | awk '{print $1}')
    period=$(echo $cpu_max | awk '{print $2}')
    cpu_limit=$(awk "BEGIN {printf \"%.1f\", $quota/$period}")
  else
    cpu_limit="unlimited"
  fi

  # Check throttling
  if [ -f "$cgroup_path/cpu.stat" ]; then
    nr_throttled=$(awk '/nr_throttled/ {print $2}' $cgroup_path/cpu.stat)
  else
    nr_throttled=0
  fi

  # Report mismatches
  if [ "$gomaxprocs" != "unknown" ] && [ "$cpu_limit" != "unlimited" ]; then
    if (( $(echo "$gomaxprocs > $cpu_limit" | bc -l) )); then
      echo "MISMATCH: Container $container"
      echo "  GOMAXPROCS: $gomaxprocs"
      echo "  CPU limit: $cpu_limit"
      echo "  Throttled: $nr_throttled times"
      echo ""
    fi
  fi
done
```

---

## DHCP Socket Overhead with AF_PACKET

### Problem

`AF_PACKET` sockets (used by DHCP clients, tcpdump, monitoring tools) receive **every packet** at the link layer, even those not destined for the host. At >1M packets/sec, this causes CPU overhead processing and dropping unwanted packets.

### Detection

```bash
# List AF_PACKET sockets
ss --packet
# or
ss -A packet

# Example output:
# Netid  State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port
# p_raw  UNCONN  0       0       *:eth0             *

# Identify process holding packet socket
lsof -i -a -p $(ss --packet | grep -oP 'pid=\K[0-9]+' | head -1)

# Common culprits:
# - dhclient
# - systemd-networkd
# - tcpdump (left running)
# - packet capture tools
# - monitoring agents

# Check packet processing overhead
perf top -p $(pgrep dhclient)
# Look for: packet_rcv, tpacket_rcv, __netif_receive_skb

# Measure packet drops on AF_PACKET socket
cat /proc/net/packet
# tp_drops column > 0 indicates overload
```

**Network-wide packet rate:**
```bash
# Check if you're even in high packet rate scenario
sar -n DEV 1 10 | grep eth0
# rxpck/s column > 100K = potential issue
# rxpck/s > 1M = definitely a problem for AF_PACKET
```

### Root Cause

AF_PACKET sockets bypass the network stack's protocol demultiplexing. They receive:
- Broadcast packets
- Multicast packets
- Packets for other MAC addresses (if NIC in promiscuous mode)
- All protocol types

At 1M pps:
- Each packet copied to socket buffer
- Userspace daemon must wake, read, and drop
- CPU cycles wasted on irrelevant packets

### Mitigation

**Option 1: Disable DHCP after boot (AWS, GCP, Azure)**

Most cloud VMs have **permanent** private IPs. DHCP is only needed at first boot.

```bash
# Check if IP is static (won't change)
# AWS: Private IPs are fixed for instance lifetime
# GCP: Ephemeral IPs are fixed unless instance deleted
# Azure: Dynamic IPs rarely change unless NIC reattached

# Stop DHCP client
systemctl stop dhclient
systemctl disable dhclient

# Or for systemd-networkd
systemctl stop systemd-networkd
systemctl disable systemd-networkd

# Verify socket closed
ss --packet
# Should show no dhclient socket

# Configure static IP
# /etc/netplan/01-netcfg.yaml (Ubuntu)
network:
  version: 2
  ethernets:
    eth0:
      addresses:
        - 10.0.1.5/24  # Your current DHCP-assigned IP
      gateway4: 10.0.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]

netplan apply
```

**Option 2: Use BPF filter on packet socket**

If you must keep DHCP running, filter at kernel level:

```bash
# dhclient with BPF filter (requires patched dhclient or custom build)
# Filter: only UDP port 67/68 (DHCP)
# This reduces 1M pps to ~100 pps (only DHCP traffic)

# For tcpdump (example of proper filtering)
tcpdump -i eth0 'udp port 67 or udp port 68'  # DHCP only
# vs
tcpdump -i eth0  # BAD: processes all packets
```

**Option 3: Container-specific - disable network tooling**

```dockerfile
# Remove unnecessary network tools from production images
RUN apt-get remove -y tcpdump netcat-openbsd net-tools

# Use minimal base images (distroless, alpine)
FROM gcr.io/distroless/base-debian11
# No shell, no packet tools
```

### Verification

```bash
# Before mitigation: check CPU load from packet processing
perf top -p $(pgrep dhclient)

# After mitigation: verify socket closed
ss --packet | grep -c "p_raw\|p_dgram"
# Should be 0 or minimal

# Confirm no packet drops
cat /proc/net/packet
# tp_drops should be 0

# CPU usage reduction
pidstat -u | grep -E "dhclient|systemd-network"
# Should show process gone or <1% CPU
```

### Advanced: Identify Hidden Packet Sockets

```bash
#!/bin/bash
# packet-socket-audit.sh

echo "=== AF_PACKET Socket Audit ==="
echo ""

# List all packet sockets with owning process
ss --packet -p | while read line; do
  if [[ $line =~ pid=([0-9]+) ]]; then
    pid=${BASH_REMATCH[1]}
    cmd=$(ps -p $pid -o comm=)
    echo "Process: $cmd (PID: $pid)"
    echo "  $line"

    # Check CPU usage
    cpu=$(ps -p $pid -o %cpu= 2>/dev/null)
    echo "  CPU: ${cpu}%"

    # Check packet stats
    echo "  Packet stats:"
    cat /proc/net/packet | awk -v pid=$pid 'NR>1 && $9==pid {print "    RX: "$4" packets, Drops: "$5}'
    echo ""
  fi
done

# Check for promiscuous mode (amplifies problem)
echo "=== Promiscuous Mode Check ==="
ip link show | grep -i promisc
echo ""

# Packet rate per interface
echo "=== Current Packet Rates ==="
sar -n DEV 1 1 | grep -v Average | tail -n +3
```

---

## Container Scheduler Throttling Patterns

### Problem Types

1. **Bursty throttling**: Low average CPU, but periodic spikes hit quota
2. **Sustained throttling**: Legitimately hitting CPU limit
3. **Mismatch throttling**: Multi-threaded app vs single-core quota (GOMAXPROCS issue)
4. **GC throttling**: JVM/Go GC pauses exhaust quota

### Detection Commands

```bash
# Get container cgroup path
CONTAINER_ID="abc123"
CGROUP_PATH="/sys/fs/cgroup/system.slice/docker-${CONTAINER_ID}.scope"

# For Kubernetes
POD_NAME="mypod-abc123"
CONTAINER_NAME="mycontainer"
# Find cgroup path
kubectl get pod $POD_NAME -o jsonpath='{.status.containerStatuses[0].containerID}' | cut -d/ -f3
# Then look under /sys/fs/cgroup/kubepods.slice/...

# Check throttling stats
cat $CGROUP_PATH/cpu.stat
# Key fields:
#   nr_periods: total enforcement periods
#   nr_throttled: periods where throttled
#   throttled_usec: total microseconds throttled

# Calculate throttle percentage
nr_periods=$(awk '/nr_periods/ {print $2}' $CGROUP_PATH/cpu.stat)
nr_throttled=$(awk '/nr_throttled/ {print $2}' $CGROUP_PATH/cpu.stat)
throttle_pct=$((nr_throttled * 100 / nr_periods))
echo "Throttle rate: ${throttle_pct}%"

# Thresholds:
# 0-1%: Acceptable for bursty workloads
# 1-5%: Investigate if latency-sensitive
# 5-25%: Moderate throttling, impacts performance
# >25%: Severe throttling, immediate action needed
```

**Time-series monitoring:**
```bash
#!/bin/bash
# throttle-monitor.sh
CGROUP_PATH="/sys/fs/cgroup/system.slice/docker-abc123.scope"

prev_throttled=0
while true; do
  curr_throttled=$(awk '/nr_throttled/ {print $2}' $CGROUP_PATH/cpu.stat)
  throttled_usec=$(awk '/throttled_usec/ {print $2}' $CGROUP_PATH/cpu.stat)

  delta=$((curr_throttled - prev_throttled))

  if [ $delta -gt 0 ]; then
    echo "$(date +%T) THROTTLED: $delta periods, total wait: $((throttled_usec/1000))ms"
  fi

  prev_throttled=$curr_throttled
  sleep 1
done
```

### Pattern Recognition

**Pattern 1: Bursty Throttling**
```
Symptom: nr_throttled > 0, but average CPU% < limit
Example: 100% CPU limit, average 60% usage, but throttled 15% of periods

Diagnosis:
  - Multi-threaded app with bursts
  - Short compute spikes exhaust quota early in period
  - Remaining period: throttled

Fix:
  1. Increase cpu.cfs_period_us (allow more burst headroom)
     echo 250000 > cpu.cfs_period_us  # 250ms period
     echo 625000 > cpu.cfs_quota_us   # 2.5 CPUs over 250ms = same avg

  2. Enable cpu.cfs_burst_us (kernel 5.14+)
     echo 100000 > cpu.cfs_burst_us  # Allow 100ms burst accumulation

  3. Remove limits, use requests only (Kubernetes)
     resources:
       requests:
         cpu: "2"
       # No limits
```

**Pattern 2: GC Throttling**
```
Symptom: Throttling correlates with GC pauses (JVM/Go)

Detection (JVM):
  docker exec $CONTAINER_ID jstat -gcutil 1 1000
  # Watch for FGCT (Full GC Time) spikes correlating with throttling

Detection (Go):
  # Enable GC trace
  docker run -e GODEBUG=gctrace=1 myapp
  # Look for GC pauses > quota period

Fix:
  - Increase CPU quota to accommodate GC threads
  - Tune GC (fewer threads, concurrent instead of STW)
  - JVM: -XX:ParallelGCThreads=2 (match CPU limit)
  - Go: GOMAXPROCS=<cpu_limit> (see section above)
```

**Pattern 3: Noisy Neighbor**
```
Symptom: Throttled but shouldn't be (under quota)

Detection:
  # Check PSI (see PSI section below)
  cat $CGROUP_PATH/cpu.pressure
  # some avg10 > 10% = contention
  # full avg10 > 5% = severe starvation

  # eBPF attribution (see scheduler-debugging-deep-dive.md)
  # Identify which cgroup is preempting you

Fix:
  - CPU pinning (isolate from noisy neighbors)
  - QoS class adjustment (Kubernetes Guaranteed > Burstable > BestEffort)
  - Node anti-affinity rules
```

### Mitigation Strategy Decision Tree

```
START: Container experiencing throttling (nr_throttled > 0)

Calculate throttle rate: (nr_throttled / nr_periods) * 100

├─ <5% AND latency-insensitive app
│  └─ ACTION: Monitor, no change needed
│
├─ <5% AND latency-sensitive app
│  ├─ Check pattern: bursty or sustained?
│  │  ├─ Bursty (avg CPU < limit)
│  │  │  └─ ACTION: Increase period OR enable burst
│  │  └─ Sustained (avg CPU ≈ limit)
│  │     └─ ACTION: Increase quota OR optimize app
│
├─ 5-25%
│  ├─ Check application type
│  │  ├─ Multi-threaded (Go, JVM, etc.)
│  │  │  ├─ GOMAXPROCS > CPU limit?
│  │  │  │  └─ ACTION: Fix GOMAXPROCS (see above)
│  │  │  └─ GC/runtime threads > CPU limit?
│  │  │     └─ ACTION: Tune GC threads OR increase quota
│  │  └─ Single-threaded
│  │     └─ ACTION: Optimize hot path OR increase quota
│
└─ >25%
   └─ CRITICAL: Immediate action required
      ├─ Quick fix: Remove limits temporarily
      │  resources:
      │    requests: {cpu: "2"}
      │    # limits: commented out
      │
      └─ Long-term: Profile and optimize OR allocate more CPU
```

---

## cgroup v2 Debugging Commands

Comprehensive cgroup v2 interface reference for container debugging.

### CPU Debugging

```bash
CGROUP_PATH="/sys/fs/cgroup/system.slice/docker-abc123.scope"

# CPU quota and period
cat $CGROUP_PATH/cpu.max
# Format: quota_us period_us
# Example: 200000 100000 = 2 CPUs (200ms per 100ms period)
# "max 100000" = unlimited quota

# CPU weight (nice value equivalent, 1-10000)
cat $CGROUP_PATH/cpu.weight
# Default: 100
# Higher = more CPU share when contended

# CPU statistics
cat $CGROUP_PATH/cpu.stat
# usage_usec: total CPU time (all cores)
# user_usec: user mode time
# system_usec: kernel mode time
# nr_periods: total enforcement periods
# nr_throttled: periods with throttling
# throttled_usec: total throttled time
# nr_bursts: burst usage events (if burst enabled)
# burst_usec: total burst time

# CPU pressure (PSI)
cat $CGROUP_PATH/cpu.pressure
# some avg10=X.XX avg60=Y.YY avg300=Z.ZZ total=NNNNNN
# full avg10=X.XX avg60=Y.YY avg300=Z.ZZ total=NNNNNN

# CPU idle (kernel 6.2+)
cat $CGROUP_PATH/cpu.idle
# 0 = normal priority
# 1 = yield to other cgroups when idle
```

**CPU analysis script:**
```bash
#!/bin/bash
CGROUP_PATH="$1"

echo "=== CPU Analysis ==="

# Get limits
cpu_max=$(cat $CGROUP_PATH/cpu.max)
quota=$(echo $cpu_max | awk '{print $1}')
period=$(echo $cpu_max | awk '{print $2}')

if [ "$quota" = "max" ]; then
  echo "CPU limit: unlimited"
else
  cpu_limit=$(awk "BEGIN {printf \"%.2f\", $quota/$period}")
  echo "CPU limit: $cpu_limit cores"
fi

# Get usage
usage_usec=$(awk '/usage_usec/ {print $2}' $CGROUP_PATH/cpu.stat)
echo "Total CPU time: $((usage_usec / 1000000)) seconds"

# Get throttling
nr_periods=$(awk '/nr_periods/ {print $2}' $CGROUP_PATH/cpu.stat)
nr_throttled=$(awk '/nr_throttled/ {print $2}' $CGROUP_PATH/cpu.stat)
throttled_usec=$(awk '/throttled_usec/ {print $2}' $CGROUP_PATH/cpu.stat)

if [ $nr_periods -gt 0 ]; then
  throttle_pct=$((nr_throttled * 100 / nr_periods))
  echo "Throttling: $throttle_pct% of periods ($nr_throttled / $nr_periods)"
  echo "Throttled time: $((throttled_usec / 1000000)) seconds"

  if [ $throttle_pct -gt 25 ]; then
    echo "⚠️  CRITICAL: Severe throttling detected"
  elif [ $throttle_pct -gt 5 ]; then
    echo "⚠️  WARNING: Moderate throttling"
  fi
fi

# Get pressure
if [ -f "$CGROUP_PATH/cpu.pressure" ]; then
  echo ""
  echo "CPU Pressure:"
  cat $CGROUP_PATH/cpu.pressure
fi
```

### Memory Debugging

```bash
# Memory limits
cat $CGROUP_PATH/memory.max      # Hard limit (OOM trigger)
cat $CGROUP_PATH/memory.high     # Soft limit (throttling)
cat $CGROUP_PATH/memory.min      # Minimum guarantee
cat $CGROUP_PATH/memory.low      # Best-effort protection

# Current usage
cat $CGROUP_PATH/memory.current

# Memory breakdown
cat $CGROUP_PATH/memory.stat
# Key fields:
#   anon: anonymous memory (heap, stack)
#   file: page cache
#   kernel_stack: kernel stack usage
#   slab: kernel slab allocations
#   sock: socket buffer memory
#   shmem: shared memory
#   pgfault: minor page faults
#   pgmajfault: major page faults (disk I/O)

# Memory events
cat $CGROUP_PATH/memory.events
# low: crossed memory.low
# high: crossed memory.high (throttling started)
# max: crossed memory.max (OOM risk)
# oom: OOM kills
# oom_kill: processes killed by OOM

# Memory pressure
cat $CGROUP_PATH/memory.pressure
```

**Memory analysis script:**
```bash
#!/bin/bash
CGROUP_PATH="$1"

echo "=== Memory Analysis ==="

# Limits
max=$(cat $CGROUP_PATH/memory.max)
high=$(cat $CGROUP_PATH/memory.high 2>/dev/null || echo "not set")
current=$(cat $CGROUP_PATH/memory.current)

echo "Current: $((current / 1048576)) MB"
echo "High (throttle): $high"
echo "Max (OOM): $max"

if [ "$max" != "max" ]; then
  pct=$((current * 100 / max))
  echo "Usage: ${pct}%"

  if [ $pct -gt 90 ]; then
    echo "⚠️  CRITICAL: Near OOM limit"
  elif [ $pct -gt 75 ]; then
    echo "⚠️  WARNING: High memory usage"
  fi
fi

# Breakdown
echo ""
echo "Memory breakdown:"
anon=$(awk '/^anon / {print $2}' $CGROUP_PATH/memory.stat)
file=$(awk '/^file / {print $2}' $CGROUP_PATH/memory.stat)
slab=$(awk '/^slab / {print $2}' $CGROUP_PATH/memory.stat)
echo "  Anon (heap/stack): $((anon / 1048576)) MB"
echo "  File (page cache): $((file / 1048576)) MB"
echo "  Slab (kernel): $((slab / 1048576)) MB"

# Check for thrashing
pgmajfault=$(awk '/pgmajfault/ {print $2}' $CGROUP_PATH/memory.stat)
if [ $pgmajfault -gt 0 ]; then
  echo ""
  echo "⚠️  Major page faults: $pgmajfault (memory pressure)"
fi

# OOM events
oom=$(awk '/^oom / {print $2}' $CGROUP_PATH/memory.events)
oom_kill=$(awk '/^oom_kill / {print $2}' $CGROUP_PATH/memory.events)
if [ $oom_kill -gt 0 ]; then
  echo ""
  echo "⚠️  OOM kills: $oom_kill"
fi
```

### I/O Debugging

```bash
# I/O limits (max bytes/sec)
cat $CGROUP_PATH/io.max
# Format: MAJ:MIN rbps=X wbps=Y riops=Z wiops=W
# Example: 8:0 rbps=10485760 wbps=10485760 riops=1000 wiops=1000

# I/O weight
cat $CGROUP_PATH/io.weight
# Default: 100, range 1-10000

# I/O statistics
cat $CGROUP_PATH/io.stat
# Format: MAJ:MIN rbytes=X wbytes=Y rios=Z wios=W dbytes=A dios=B
# rbytes/wbytes: total bytes read/written
# rios/wios: total I/O operations
# dbytes/dios: discarded bytes/operations (TRIM)

# I/O pressure
cat $CGROUP_PATH/io.pressure
```

**I/O analysis script:**
```bash
#!/bin/bash
CGROUP_PATH="$1"

echo "=== I/O Analysis ==="

cat $CGROUP_PATH/io.stat | while read line; do
  dev=$(echo $line | awk '{print $1}')
  rbytes=$(echo $line | grep -oP 'rbytes=\K[0-9]+')
  wbytes=$(echo $line | grep -oP 'wbytes=\K[0-9]+')
  rios=$(echo $line | grep -oP 'rios=\K[0-9]+')
  wios=$(echo $line | grep -oP 'wios=\K[0-9]+')

  echo "Device $dev:"
  echo "  Read:  $((rbytes / 1048576)) MB ($rios ops)"
  echo "  Write: $((wbytes / 1048576)) MB ($wios ops)"

  # Calculate average I/O size
  if [ $rios -gt 0 ]; then
    avg_read=$((rbytes / rios))
    echo "  Avg read size: $avg_read bytes"

    if [ $avg_read -lt 4096 ]; then
      echo "  ⚠️  Small read I/O (< 4KB) - inefficient"
    fi
  fi

  if [ $wios -gt 0 ]; then
    avg_write=$((wbytes / wios))
    echo "  Avg write size: $avg_write bytes"

    if [ $avg_write -lt 4096 ]; then
      echo "  ⚠️  Small write I/O (< 4KB) - inefficient"
    fi
  fi

  echo ""
done
```

### Quick Health Check Script

```bash
#!/bin/bash
# cgroup-health-check.sh
# Usage: ./cgroup-health-check.sh /sys/fs/cgroup/path/to/container

CGROUP_PATH="$1"

if [ ! -d "$CGROUP_PATH" ]; then
  echo "Error: $CGROUP_PATH not found"
  exit 1
fi

echo "========================================="
echo "cgroup Health Check"
echo "Path: $CGROUP_PATH"
echo "========================================="
echo ""

# CPU
if [ -f "$CGROUP_PATH/cpu.stat" ]; then
  nr_throttled=$(awk '/nr_throttled/ {print $2}' $CGROUP_PATH/cpu.stat)
  nr_periods=$(awk '/nr_periods/ {print $2}' $CGROUP_PATH/cpu.stat)

  if [ $nr_periods -gt 0 ]; then
    throttle_pct=$((nr_throttled * 100 / nr_periods))
    echo "CPU: Throttled $throttle_pct% of periods"

    if [ $throttle_pct -gt 25 ]; then
      echo "  ❌ CRITICAL: Severe CPU throttling"
    elif [ $throttle_pct -gt 5 ]; then
      echo "  ⚠️  WARNING: Moderate CPU throttling"
    else
      echo "  ✅ OK"
    fi
  fi
fi

echo ""

# Memory
if [ -f "$CGROUP_PATH/memory.current" ]; then
  current=$(cat $CGROUP_PATH/memory.current)
  max=$(cat $CGROUP_PATH/memory.max)

  if [ "$max" != "max" ]; then
    pct=$((current * 100 / max))
    echo "Memory: ${pct}% of limit"

    if [ $pct -gt 90 ]; then
      echo "  ❌ CRITICAL: Near OOM"
    elif [ $pct -gt 75 ]; then
      echo "  ⚠️  WARNING: High usage"
    else
      echo "  ✅ OK"
    fi
  else
    echo "Memory: Unlimited"
  fi

  # Check OOM kills
  if [ -f "$CGROUP_PATH/memory.events" ]; then
    oom_kill=$(awk '/^oom_kill / {print $2}' $CGROUP_PATH/memory.events)
    if [ $oom_kill -gt 0 ]; then
      echo "  ❌ OOM kills: $oom_kill"
    fi
  fi
fi

echo ""

# PSI
echo "Pressure Stall Information:"
for resource in cpu memory io; do
  file="$CGROUP_PATH/${resource}.pressure"
  if [ -f "$file" ]; then
    echo "  $resource:"
    some_avg10=$(awk '/some/ {print $2}' $file | grep -oP 'avg10=\K[0-9.]+')
    full_avg10=$(awk '/full/ {print $2}' $file | grep -oP 'avg10=\K[0-9.]+')

    echo "    some: ${some_avg10}%"
    echo "    full: ${full_avg10}%"

    # Flag issues
    if (( $(echo "$full_avg10 > 5.0" | bc -l) )); then
      echo "    ❌ CRITICAL: Full pressure"
    elif (( $(echo "$some_avg10 > 10.0" | bc -l) )); then
      echo "    ⚠️  WARNING: Elevated pressure"
    fi
  fi
done

echo ""
echo "========================================="
```

---

## PSI (Pressure Stall Information) Interpretation

PSI provides real-time pressure metrics showing percentage of time tasks are stalled waiting for resources.

### Understanding PSI Metrics

```bash
# System-wide PSI
cat /proc/pressure/cpu
cat /proc/pressure/memory
cat /proc/pressure/io

# Per-cgroup PSI (cgroup v2)
cat /sys/fs/cgroup/<path>/cpu.pressure
cat /sys/fs/cgroup/<path>/memory.pressure
cat /sys/fs/cgroup/<path>/io.pressure

# Format:
# some avg10=X.XX avg60=Y.YY avg300=Z.ZZ total=NNNNN
# full avg10=X.XX avg60=Y.YY avg300=Z.ZZ total=NNNNN
```

**Metric breakdown:**

| Field | Meaning |
|-------|---------|
| `some` | % time **some** tasks stalled (at least one waiting) |
| `full` | % time **all** tasks stalled (complete blockage) |
| `avg10` | 10-second moving average (most responsive) |
| `avg60` | 60-second moving average (good for alerting) |
| `avg300` | 5-minute moving average (trend analysis) |
| `total` | Accumulated microseconds of pressure |

### Thresholds by Resource

**CPU Pressure:**
```
some avg10:
  < 5%: Healthy
  5-10%: Mild contention
  10-20%: Moderate contention
  > 20%: Severe contention

full avg10:
  < 1%: Healthy
  1-5%: Concerning (all tasks blocked periodically)
  > 5%: Critical (severe CPU starvation)

Interpretation:
  - some > 0: Tasks waiting for CPU (normal for busy systems)
  - full > 0: ALL tasks blocked (serious problem)
```

**Memory Pressure:**
```
some avg10:
  < 5%: Healthy
  5-20%: Mild pressure (some reclaim activity)
  20-50%: Moderate pressure (active reclaim)
  > 50%: Severe pressure (thrashing)

full avg10:
  < 1%: Healthy
  1-10%: Concerning (all tasks waiting on memory)
  > 10%: Critical (severe thrashing, OOM imminent)

Interpretation:
  - some > 0: Memory reclaim happening
  - full > 0: All tasks waiting for memory (thrashing)
```

**I/O Pressure:**
```
some avg10:
  < 10%: Healthy
  10-30%: Mild I/O wait
  30-60%: Moderate I/O bottleneck
  > 60%: Severe I/O bottleneck

full avg10:
  < 5%: Healthy
  5-20%: Concerning (all tasks I/O blocked)
  > 20%: Critical (disk completely saturated)

Interpretation:
  - some > 0: Some tasks waiting for I/O
  - full > 0: All tasks blocked on I/O
```

### PSI-Based Alerting

```bash
# Prometheus alert rules (example)
groups:
- name: psi_alerts
  rules:
  - alert: CPUPressureHigh
    expr: cgroup_cpu_pressure_some_avg10 > 20
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "CPU pressure high in {{ $labels.cgroup }}"

  - alert: CPUPressureCritical
    expr: cgroup_cpu_pressure_full_avg10 > 5
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "All tasks CPU-starved in {{ $labels.cgroup }}"

  - alert: MemoryPressureCritical
    expr: cgroup_memory_pressure_full_avg10 > 10
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Memory thrashing in {{ $labels.cgroup }}"
```

### PSI Monitoring Script

```bash
#!/bin/bash
# psi-monitor.sh - Continuous PSI monitoring

CGROUP_PATH="${1:-/sys/fs/cgroup}"
INTERVAL="${2:-1}"

echo "PSI Monitor - Path: $CGROUP_PATH"
echo "Interval: ${INTERVAL}s"
echo ""
printf "%-10s %-8s %-8s %-8s %-8s %-8s %-8s\n" \
  "TIME" "CPU-some" "CPU-full" "MEM-some" "MEM-full" "IO-some" "IO-full"
echo "--------------------------------------------------------------------"

while true; do
  timestamp=$(date +%H:%M:%S)

  # CPU
  cpu_some=$(awk '/some/ {print $2}' $CGROUP_PATH/cpu.pressure | grep -oP 'avg10=\K[0-9.]+' || echo "0")
  cpu_full=$(awk '/full/ {print $2}' $CGROUP_PATH/cpu.pressure | grep -oP 'avg10=\K[0-9.]+' || echo "0")

  # Memory
  mem_some=$(awk '/some/ {print $2}' $CGROUP_PATH/memory.pressure | grep -oP 'avg10=\K[0-9.]+' || echo "0")
  mem_full=$(awk '/full/ {print $2}' $CGROUP_PATH/memory.pressure | grep -oP 'avg10=\K[0-9.]+' || echo "0")

  # I/O
  io_some=$(awk '/some/ {print $2}' $CGROUP_PATH/io.pressure | grep -oP 'avg10=\K[0-9.]+' || echo "0")
  io_full=$(awk '/full/ {print $2}' $CGROUP_PATH/io.pressure | grep -oP 'avg10=\K[0-9.]+' || echo "0")

  # Color coding
  color_cpu=""
  color_mem=""
  color_io=""

  # CPU warnings
  if (( $(echo "$cpu_full > 5.0" | bc -l) )); then
    color_cpu="⚠️ "
  elif (( $(echo "$cpu_some > 20.0" | bc -l) )); then
    color_cpu="⚠️ "
  fi

  # Memory warnings
  if (( $(echo "$mem_full > 10.0" | bc -l) )); then
    color_mem="⚠️ "
  elif (( $(echo "$mem_some > 50.0" | bc -l) )); then
    color_mem="⚠️ "
  fi

  # I/O warnings
  if (( $(echo "$io_full > 20.0" | bc -l) )); then
    color_io="⚠️ "
  elif (( $(echo "$io_some > 60.0" | bc -l) )); then
    color_io="⚠️ "
  fi

  printf "%-10s %-8s %-8s %-8s %-8s %-8s %-8s\n" \
    "$timestamp" \
    "${color_cpu}${cpu_some}" \
    "${color_cpu}${cpu_full}" \
    "${color_mem}${mem_some}" \
    "${color_mem}${mem_full}" \
    "${color_io}${io_some}" \
    "${color_io}${io_full}"

  sleep $INTERVAL
done
```

### Facebook's oomd Integration

Facebook uses PSI `full` metrics to trigger preemptive OOM killing before system-wide OOM.

```bash
# oomd config example (Systemd v245+)
# /etc/systemd/system/-.slice.d/10-oomd.conf
[Slice]
ManagedOOMMemoryPressure=kill
ManagedOOMMemoryPressureLimit=80%

# When memory.pressure full avg10 > 80%, kill processes
# Prevents system-wide OOM by sacrificing high-pressure cgroups early
```

### PSI Decision Tree

```
Container slow, investigate with PSI:

Check cpu.pressure:
├─ full avg10 > 5%
│  └─ DIAGNOSIS: All tasks CPU-starved
│     ACTION: Check throttling (cpu.stat)
│             Check GOMAXPROCS mismatch
│             Increase CPU quota
│
├─ some avg10 > 20%
│  └─ DIAGNOSIS: Significant CPU contention
│     ACTION: Profile hot paths
│             Check for noisy neighbors
│             Optimize or scale out
│
└─ some/full < thresholds
   └─ CPU healthy, check memory.pressure

Check memory.pressure:
├─ full avg10 > 10%
│  └─ DIAGNOSIS: Memory thrashing
│     ACTION: Check memory.stat pgmajfault
│             Increase memory.max
│             Optimize memory usage
│
├─ some avg10 > 50%
│  └─ DIAGNOSIS: Heavy reclaim activity
│     ACTION: Increase memory.high
│             Check for memory leaks
│
└─ some/full < thresholds
   └─ Memory healthy, check io.pressure

Check io.pressure:
├─ full avg10 > 20%
│  └─ DIAGNOSIS: I/O completely saturated
│     ACTION: Check io.stat for patterns
│             Optimize I/O (larger buffers, batching)
│             Check disk health
│
├─ some avg10 > 60%
│  └─ DIAGNOSIS: Heavy I/O wait
│     ACTION: Reduce I/O frequency
│             Use faster storage
│
└─ some/full < thresholds
   └─ I/O healthy, problem elsewhere (network, locks)
```

---

## Quick Decision Trees

### Container Slow - Root Cause Analysis

```
START: Container experiencing slowness

1. Check CPU throttling:
   cat /sys/fs/cgroup/*/cpu.stat | grep throttled

   ├─ nr_throttled > 0
   │  └─ Go to: "Container Scheduler Throttling Patterns" section
   │
   └─ nr_throttled = 0
      └─ Continue to step 2

2. Check PSI pressure:
   cat /sys/fs/cgroup/*/cpu.pressure
   cat /sys/fs/cgroup/*/memory.pressure
   cat /sys/fs/cgroup/*/io.pressure

   ├─ cpu.pressure full avg10 > 5%
   │  └─ DIAGNOSIS: CPU starvation
   │     ACTION: Increase CPU quota
   │
   ├─ memory.pressure full avg10 > 10%
   │  └─ DIAGNOSIS: Memory thrashing
   │     ACTION: Increase memory limit
   │
   ├─ io.pressure full avg10 > 20%
   │  └─ DIAGNOSIS: I/O saturation
   │     ACTION: Optimize I/O or faster disk
   │
   └─ All PSI metrics low
      └─ Continue to step 3

3. Check for application-specific issues:
   ├─ Go application?
   │  └─ Check GOMAXPROCS: go env GOMAXPROCS
   │     Compare to CPU limit
   │     └─ Mismatch? Use automaxprocs
   │
   ├─ JVM application?
   │  └─ Check GC: jstat -gcutil
   │     └─ High FGCT? Tune GC or increase CPU
   │
   └─ Check network:
      └─ ss --packet
         └─ AF_PACKET sockets? Disable DHCP post-boot
```

### High kubelet CPU - Quick Diagnosis

```
START: kubelet using high CPU

1. Check for periodic spikes (every 60s):
   pidstat -p $(pgrep kubelet) 1 120

   ├─ Spikes every ~60s
   │  └─ DIAGNOSIS: cAdvisor file walking
   │     ACTION: Go to "cAdvisor File Walking" section
   │
   └─ Constant high CPU
      └─ Continue to step 2

2. Profile kubelet:
   perf top -p $(pgrep kubelet)

   ├─ Hotspot: fs_get_your_usage, filepath.Walk
   │  └─ DIAGNOSIS: cAdvisor (confirm with step 1)
   │
   ├─ Hotspot: packet_rcv, tpacket_rcv
   │  └─ DIAGNOSIS: AF_PACKET socket overhead
   │     Check: ss --packet
   │     ACTION: Disable DHCP post-boot
   │
   └─ Other hotspots
      └─ DIAGNOSIS: Cluster load, API calls, etc.
```

### Kubernetes Pod Throttled - Fix Selection

```
Pod reports throttling (kubectl top shows <100% but throttled)

1. Get container ID and check stats:
   kubectl get pod $POD -o jsonpath='{.status.containerStatuses[0].containerID}'
   cat /sys/fs/cgroup/kubepods.slice/.../cpu.stat

   Calculate: throttle_rate = nr_throttled / nr_periods

2. Based on throttle rate:

   ├─ 1-5% AND latency-insensitive
   │  └─ ACTION: Monitor, acceptable burst throttling
   │
   ├─ 1-5% AND latency-sensitive
   │  └─ ACTION: Increase cpu.limits slightly (1.5x current)
   │
   ├─ 5-25%
   │  ├─ Check: Is it bursty? (avg CPU < limit)
   │  │  ├─ Yes → ACTION: Remove cpu.limits, keep requests
   │  │  └─ No → ACTION: Increase limits to match actual usage
   │  │
   │  └─ Check: Multi-threaded app?
   │     ├─ Go → Check GOMAXPROCS
   │     ├─ JVM → Check GC threads
   │     └─ Other → Profile and optimize
   │
   └─ >25%
      └─ ACTION: CRITICAL
         Immediate: Remove limits temporarily
         Long-term: Optimize or increase quota significantly
```

---

## Command Reference Card

### Quick Diagnostics

```bash
# Container cgroup path
CGROUP_PATH="/sys/fs/cgroup/system.slice/docker-${CONTAINER_ID}.scope"

# CPU throttling check
cat $CGROUP_PATH/cpu.stat | grep throttled

# PSI check
cat $CGROUP_PATH/cpu.pressure
cat $CGROUP_PATH/memory.pressure
cat $CGROUP_PATH/io.pressure

# Memory usage
cat $CGROUP_PATH/memory.current
cat $CGROUP_PATH/memory.max

# GOMAXPROCS check (Go containers)
docker exec $CONTAINER_ID go env GOMAXPROCS

# AF_PACKET sockets
ss --packet

# cAdvisor file count
nsenter -t $(docker inspect -f '{{.State.Pid}}' $CONTAINER_ID) -m \
  find / -xdev -type f 2>/dev/null | wc -l
```

### Kubernetes Specific

```bash
# Get container ID
kubectl get pod $POD -o jsonpath='{.status.containerStatuses[0].containerID}'

# Find cgroup path
find /sys/fs/cgroup -name "*${CONTAINER_ID}*"

# Check throttling
kubectl exec $POD -- cat /sys/fs/cgroup/cpu.stat

# Check PSI
kubectl exec $POD -- cat /sys/fs/cgroup/cpu.pressure
```

### Mitigation Commands

```bash
# Disable DHCP (cloud VMs with static IPs)
systemctl stop dhclient && systemctl disable dhclient

# Fix GOMAXPROCS (add to Dockerfile)
ENV GOMAXPROCS=<cpu_limit>

# Increase CPU quota (cgroup v2)
echo "<new_quota> <period>" > /sys/fs/cgroup/.../cpu.max

# Enable CPU burst (kernel 5.14+)
echo 100000 > /sys/fs/cgroup/.../cpu.cfs_burst_us

# Remove CPU limits (Kubernetes)
# Edit pod spec, remove resources.limits.cpu
```

---

## Thresholds Summary

| Metric | Healthy | Warning | Critical |
|--------|---------|---------|----------|
| CPU throttle rate | 0-1% | 1-5% | > 5% |
| PSI CPU some avg10 | < 5% | 5-20% | > 20% |
| PSI CPU full avg10 | < 1% | 1-5% | > 5% |
| PSI memory some avg10 | < 5% | 5-50% | > 50% |
| PSI memory full avg10 | < 1% | 1-10% | > 10% |
| PSI I/O some avg10 | < 10% | 10-60% | > 60% |
| PSI I/O full avg10 | < 5% | 5-20% | > 20% |
| Memory usage (% of max) | < 75% | 75-90% | > 90% |
| Container file count | < 50K | 50K-100K | > 100K |

---

## See Also

- [Containers & Kubernetes](07-containers-k8s.md) - Container runtime basics, cgroup metrics, PSI per-cgroup
- [Scheduler & Interrupt Internals](16-scheduler-interrupts.md) - CFS bandwidth throttling mechanics, scheduler policies
- [Scheduler Debugging Deep Dive](scheduler-debugging-deep-dive.md) - Run queue attribution, noisy neighbor detection
- [Memory Subsystem](15-memory-subsystem.md) - PSI memory pressure, OOM scoring, cgroup memory
- [eBPF & Tracing](06-ebpf-tracing.md) - Container-aware tracing, cgroup filtering

---

## References

- [The container throttling problem - Dan Luu](https://danluu.com/cgroup-throttling/)
- [Noisy Neighbor Detection with eBPF - Netflix](https://netflixtechblog.com/noisy-neighbor-detection-with-ebpf-64b1f4b3bbdd)
- [PSI tells you "What". Cgroups tell you "Who" - Linnix](https://getlinnix.substack.com/p/psi-tells-you-what-cgroups-tell-you)
- [CFS Bandwidth Control - Kernel Docs](https://docs.kernel.org/scheduler/sched-bwc.html)
- [cgroup v2 - Kernel Docs](https://docs.kernel.org/admin-guide/cgroup-v2.html)
- [PSI - Pressure Stall Information - Kernel Docs](https://docs.kernel.org/accounting/psi.html)
- [uber-go/automaxprocs - GitHub](https://github.com/uber-go/automaxprocs)

---

**Document Status**: Production-ready reference for container debugging in Kubernetes and Docker environments.
