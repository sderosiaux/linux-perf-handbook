# Chapter 00: Troubleshooting Decision Framework

The entry point for performance debugging. Start here before diving into specific tools.

---

## 1. Methodology Overview

Two complementary frameworks for systematic performance analysis.

### USE Method (Resources)

Created by Brendan Gregg. For every **resource**, check three metrics:

```
┌─────────────────────────────────────────────────────────────────┐
│                         USE METHOD                              │
├─────────────────────────────────────────────────────────────────┤
│  U - Utilization    How busy is the resource? (% time busy)    │
│  S - Saturation     Work queued/waiting (queue length, delays) │
│  E - Errors         Error counts (often overlooked!)           │
└─────────────────────────────────────────────────────────────────┘
```

**Resources to check:** CPU, Memory, Network interfaces, Storage devices, Controllers

**Key thresholds:**
- 100% utilization = definite bottleneck
- 70%+ on I/O resources = likely bottleneck
- Any saturation > 0 = adds latency
- Any errors = investigate immediately

**USE Method Quick Reference:**

| Resource | Utilization | Saturation | Errors |
|----------|-------------|------------|--------|
| CPU | `vmstat` %us+%sy, `mpstat` | `vmstat` r > CPU count | `perf stat` |
| Memory | `free -m`, `/proc/meminfo` | `vmstat` si/so, OOM kills | `dmesg` |
| Network | `sar -n DEV`, `ip -s link` | `ifconfig` overruns/drops | `netstat -s` |
| Disk | `iostat -xz` %util | `iostat` avgqu-sz, await | `/sys/.../ioerr_cnt` |

### RED Method (Services)

Created by Tom Wilkie. For every **service**, check three metrics:

```
┌─────────────────────────────────────────────────────────────────┐
│                         RED METHOD                              │
├─────────────────────────────────────────────────────────────────┤
│  R - Rate        Requests per second (throughput)              │
│  E - Errors      Failed requests per second                    │
│  D - Duration    Latency distribution (p50, p95, p99)          │
└─────────────────────────────────────────────────────────────────┘
```

**Best for:** Request-driven services, microservices, APIs

**Not for:** Batch jobs, streaming pipelines, data processing

### When to Use Which

```
                    ┌──────────────────┐
                    │  What are you    │
                    │  investigating?  │
                    └────────┬─────────┘
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
    ┌──────────────────┐          ┌──────────────────┐
    │   Infrastructure │          │   Application    │
    │   (host, VM, HW) │          │   (service, API) │
    └────────┬─────────┘          └────────┬─────────┘
             │                              │
             ▼                              ▼
    ┌──────────────────┐          ┌──────────────────┐
    │   USE METHOD     │          │   RED METHOD     │
    │   CPU, Mem, Disk │          │   Rate, Errors   │
    │   Network I/O    │          │   Duration       │
    └──────────────────┘          └──────────────────┘
```

**Combine both:** Start with RED to identify slow service, then USE to find resource constraint.

---

## 2. The 60-Second Analysis (Modernized)

Netflix's checklist, updated with modern tool equivalents.

### Classic Commands (Always Available)

```bash
# 1. LOAD AVERAGES - tasks wanting CPU + blocked I/O
uptime
# Look for: load > CPU count, increasing trend

# 2. KERNEL MESSAGES - errors, OOM, hardware failures
dmesg -T | tail -20
# Look for: oom-killer, hardware errors, TCP drops

# 3. VIRTUAL MEMORY - system-wide activity
vmstat 1 5
# Look for: r > CPU count (CPU sat), si/so > 0 (swapping)

# 4. PER-CPU BREAKDOWN - detect imbalance
mpstat -P ALL 1 3
# Look for: one CPU at 100% (single-threaded bottleneck)

# 5. PER-PROCESS CPU - who's consuming
pidstat 1 3
# Look for: unexpected processes, high %system

# 6. DISK I/O - device-level stats
iostat -xz 1 3
# Look for: %util > 60%, await > 10ms, avgqu-sz > 1

# 7. MEMORY - usage breakdown
free -m
# Look for: low available, buff/cache not reclaimable

# 8. NETWORK THROUGHPUT - interface stats
sar -n DEV 1 3
# Look for: near line rate, high drop counts

# 9. TCP STATS - connection health
sar -n TCP,ETCP 1 3
# Look for: retrans/s > 0, high passive/active

# 10. OVERVIEW - interactive check
top -b -n 1 | head -20
# Look for: %wa (I/O wait), zombie processes
```

### Modern Tool Equivalents

| Classic | Modern | Why Upgrade |
|---------|--------|-------------|
| `top` | `btop` | Dashboard with graphs, mouse support |
| `top` | `glances` | Cross-platform, web UI, exports |
| `vmstat` | `below` (Meta) | Historical data, cgroups aware |
| `iostat` | `below replay` | Time-travel debugging |
| `sar` | `atop` | Historical playback, process accounting |
| `ps aux` | `procs` | Better formatting, tree view |

### One-Liner Health Check

```bash
# Quick system pulse (copy-paste friendly)
uptime; free -m; df -h / | tail -1; vmstat 1 2 | tail -1
```

### What Each Metric Reveals

```
┌────────────────────────────────────────────────────────────────────────┐
│ vmstat 1                                                               │
├──────────┬─────────────────────────────────────────────────────────────┤
│ r        │ Run queue. r > CPU count = CPU saturated                   │
│ b        │ Blocked on I/O. High b = storage bottleneck                │
│ si/so    │ Swap in/out. Non-zero = memory pressure                    │
│ us       │ User CPU time. High = application CPU-bound                │
│ sy       │ System CPU time. High = kernel/syscall overhead            │
│ wa       │ I/O wait. High = waiting for storage                       │
│ st       │ Steal time (VM). High = hypervisor contention              │
└──────────┴─────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│ iostat -xz 1                                                           │
├──────────┬─────────────────────────────────────────────────────────────┤
│ %util    │ Device busy time. >60% often problematic                   │
│ await    │ Average I/O time (ms). >10ms for SSD = investigate         │
│ avgqu-sz │ Average queue length. >1 = device saturated                │
│ r/s, w/s │ IOPS. Compare to device specs                              │
│ rkB/s    │ Throughput. Compare to device bandwidth                    │
└──────────┴─────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────┐
│ free -m                                                                │
├──────────┬─────────────────────────────────────────────────────────────┤
│ total    │ Physical RAM                                                │
│ used     │ Actually consumed (includes buffers/cache on old kernels)  │
│ free     │ Completely unused (often near zero, that's OK)             │
│ available│ Memory available for applications (KEY METRIC)             │
│ buff/cache│ Reclaimable on demand (not wasted)                        │
└──────────┴─────────────────────────────────────────────────────────────┘
```

---

## 3. Decision Trees

### Application Is Slow

```
                        ┌─────────────────────┐
                        │ Application is slow │
                        └──────────┬──────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │  Is CPU utilization high?   │
                    │  (top, mpstat, pidstat)     │
                    └──────────────┬──────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              ▼ YES                │ NO                 │
    ┌──────────────────┐          │         ┌──────────────────┐
    │ See: High CPU    │          │         │ Is memory under  │
    │ flowchart below  │          │         │ pressure?        │
    └──────────────────┘          │         │ (free, vmstat)   │
                                  │         └────────┬─────────┘
                                  │                  │
                                  │    ┌─────────────┼─────────────┐
                                  │    ▼ YES         │ NO          │
                                  │ ┌────────────┐   │   ┌────────────────┐
                                  │ │ See: Memory│   │   │ Is I/O wait    │
                                  │ │ flowchart  │   │   │ high? (%wa)    │
                                  │ └────────────┘   │   │ (iostat, iotop)│
                                  │                  │   └───────┬────────┘
                                  │                  │           │
                                  │                  │  ┌────────┼────────┐
                                  │                  │  ▼ YES    │ NO     │
                                  │                  │ ┌──────┐  │ ┌──────────────┐
                                  │                  │ │Disk  │  │ │ Network      │
                                  │                  │ │I/O   │  │ │ latency?     │
                                  │                  │ │chart │  │ │ (ss, netstat)│
                                  │                  │ └──────┘  │ └──────┬───────┘
                                  │                  │           │        │
                                  │                  │           │  ┌─────┼─────┐
                                  │                  │           │  ▼ YES │ NO  │
                                  │                  │           │ ┌────┐ │┌────────────┐
                                  │                  │           │ │Net │ ││Lock/sync  │
                                  │                  │           │ │flow│ ││contention?│
                                  │                  │           │ └────┘ ││perf, strace│
                                  │                  │           │        │└────────────┘
                                  │                  │           │        │
                                  │                  │           │        ▼
                                  │                  │           │  ┌────────────┐
                                  │                  │           │  │ Profile    │
                                  │                  │           │  │ with perf  │
                                  │                  │           │  │ off-CPU    │
                                  │                  │           │  └────────────┘
```

### High CPU Usage

```
                        ┌─────────────────────┐
                        │   High CPU Usage    │
                        │   (>80% sustained)  │
                        └──────────┬──────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
    ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
    │ High %user (us)  │ │ High %system (sy)│ │ High %iowait (wa)│
    └────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
             │                    │                    │
             ▼                    ▼                    ▼
    ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
    │ Application code │ │ Kernel overhead  │ │ Waiting for I/O  │
    │ is CPU-bound     │ │ or syscall-heavy │ │ (see Disk I/O)   │
    └────────┬─────────┘ └────────┬─────────┘ └──────────────────┘
             │                    │
             ▼                    ▼
    ┌──────────────────┐ ┌──────────────────┐
    │ Profile with:    │ │ Check for:       │
    │ - perf record    │ │ - High context   │
    │ - async-profiler │ │   switches       │
    │ - py-spy         │ │ - Syscall storms │
    │ Generate flame   │ │ - Lock contention│
    │ graph            │ │                  │
    └────────┬─────────┘ └────────┬─────────┘
             │                    │
             ▼                    ▼
    ┌──────────────────┐ ┌──────────────────┐
    │ TOOLS:           │ │ TOOLS:           │
    │ perf top         │ │ strace -c        │
    │ perf record -g   │ │ perf stat        │
    │ flamegraph.pl    │ │ vmstat (cs col)  │
    │ hotspot          │ │ funccount        │
    └──────────────────┘ └──────────────────┘

    ┌──────────────────┐
    │ High %steal (st) │  (VMs only)
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────────────────────────────┐
    │ Hypervisor is stealing CPU cycles        │
    │ - Other VMs on same host are busy        │
    │ - Overcommitted host                     │
    │ FIX: Migrate VM, resize, dedicated host  │
    └──────────────────────────────────────────┘
```

### Memory Pressure

```
                        ┌─────────────────────┐
                        │   Memory Pressure   │
                        │   (low available)   │
                        └──────────┬──────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │  Is swap being used?        │
                    │  (vmstat si/so, swapon -s)  │
                    └──────────────┬──────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              ▼ YES                │                    ▼ NO
    ┌──────────────────┐          │          ┌──────────────────┐
    │ Swapping active  │          │          │ Check buff/cache │
    │ Performance will │          │          │ Is it high?      │
    │ degrade severely │          │          └────────┬─────────┘
    └────────┬─────────┘          │                   │
             │                    │         ┌─────────┼─────────┐
             ▼                    │         ▼ YES     │         ▼ NO
    ┌──────────────────┐          │ ┌────────────┐    │ ┌────────────────┐
    │ Find memory hogs:│          │ │ Normal     │    │ │ Memory leak or │
    │ - top (sort by   │          │ │ File cache │    │ │ large working  │
    │   RES/VIRT)      │          │ │ Reclaimable│    │ │ set            │
    │ - smem           │          │ └────────────┘    │ └───────┬────────┘
    │ - pmap -x <pid>  │          │                   │         │
    └────────┬─────────┘          │                   │         ▼
             │                    │                   │ ┌────────────────┐
             ▼                    │                   │ │ Diagnose:      │
    ┌──────────────────┐          │                   │ │ - heaptrack    │
    │ OOM killer logs? │          │                   │ │ - valgrind     │
    │ dmesg | grep oom │          │                   │ │ - pmap changes │
    └────────┬─────────┘          │                   │ │   over time    │
             │                    │                   │ └────────────────┘
    ┌────────┴────────┐           │                   │
    ▼ YES             ▼ NO        │                   │
┌──────────────┐ ┌──────────────┐ │                   │
│ Check killed │ │ Approaching  │ │                   │
│ process, RSS │ │ OOM          │ │                   │
│ at kill time │ │ Add RAM or   │ │                   │
│              │ │ reduce usage │ │                   │
└──────────────┘ └──────────────┘ │                   │

    ┌──────────────────────────────────────────┐
    │           Memory Fragmentation           │
    ├──────────────────────────────────────────┤
    │ Symptoms:                                │
    │ - Allocation failures despite free RAM   │
    │ - /proc/buddyinfo shows no large blocks  │
    │                                          │
    │ Check: cat /proc/buddyinfo               │
    │        cat /sys/kernel/debug/extfrag/*   │
    │                                          │
    │ Fix (off-peak):                          │
    │ echo 1 > /proc/sys/vm/compact_memory     │
    └──────────────────────────────────────────┘
```

### Network Latency

```
                        ┌─────────────────────┐
                        │   Network Latency   │
                        │   (slow responses)  │
                        └──────────┬──────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
    ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
    │ DNS Resolution   │ │ TCP Connection   │ │ Application      │
    │                  │ │                  │ │ Layer            │
    └────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
             │                    │                    │
             ▼                    ▼                    ▼
    ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
    │ Test:            │ │ Test:            │ │ Test:            │
    │ dig @server name │ │ ss -ti           │ │ tcpdump + timing │
    │ time dig name    │ │ netstat -s       │ │ App-level traces │
    │                  │ │ tcpdump          │ │                  │
    └────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
             │                    │                    │
             ▼                    ▼                    ▼
    ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
    │ DNS slow?        │ │ Retransmissions? │ │ Serialization    │
    │ - Wrong resolver │ │ - Packet loss    │ │ delay?           │
    │ - Resolver down  │ │ - Congestion     │ │ - Large payloads │
    │ - DNSSEC issues  │ │ - MTU mismatch   │ │ - Slow backend   │
    │                  │ │                  │ │ - Query perf     │
    │ FIX:             │ │ FIX:             │ │                  │
    │ /etc/resolv.conf │ │ Check path with  │ │ FIX:             │
    │ systemd-resolved │ │ mtr, traceroute  │ │ Profile app code │
    └──────────────────┘ │ Tune TCP buffers │ │ Optimize queries │
                         └──────────────────┘ └──────────────────┘

    ┌────────────────────────────────────────────────────────────┐
    │                TCP Troubleshooting Checklist               │
    ├────────────────────────────────────────────────────────────┤
    │ ss -ti                # Show TCP internal state            │
    │ netstat -s | grep -i retrans  # Retransmission count       │
    │ ethtool -S eth0       # NIC-level errors                   │
    │ cat /proc/net/snmp    # Protocol counters                  │
    │                                                            │
    │ High retransmissions:                                      │
    │ - Network congestion: check intermediate hops              │
    │ - Packet drops: check ethtool stats, dmesg                 │
    │ - Firewall issues: check iptables DROP counts              │
    └────────────────────────────────────────────────────────────┘
```

### Disk I/O Bottleneck

```
                        ┌─────────────────────┐
                        │  Disk I/O Problem   │
                        │  (high %util, await)│
                        └──────────┬──────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │  Which device is saturated? │
                    │  (iostat -xz 1)             │
                    └──────────────┬──────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
    ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
    │ High IOPS        │ │ High Throughput  │ │ High Latency     │
    │ (r/s + w/s)      │ │ (MB/s near max)  │ │ (await >> normal)│
    └────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
             │                    │                    │
             ▼                    ▼                    ▼
    ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
    │ Random I/O       │ │ Sequential I/O   │ │ Queue backup or  │
    │ workload         │ │ workload         │ │ device issue     │
    │                  │ │                  │ │                  │
    │ Check:           │ │ Check:           │ │ Check:           │
    │ - iotop          │ │ - iotop          │ │ - avgqu-sz       │
    │ - biotop (bcc)   │ │ - Large file ops │ │ - Device health  │
    │                  │ │ - Backups running│ │ - smartctl       │
    └────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
             │                    │                    │
             ▼                    ▼                    ▼
    ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
    │ Solutions:       │ │ Solutions:       │ │ Solutions:       │
    │ - Add cache      │ │ - Schedule off-  │ │ - Replace disk   │
    │ - Faster storage │ │   peak           │ │ - Add RAID       │
    │ - Reduce IOPS    │ │ - Rate limit     │ │ - Fix I/O sched  │
    │ - Read-ahead     │ │ - Separate disk  │ │ - ionice procs   │
    └──────────────────┘ └──────────────────┘ └──────────────────┘

    ┌────────────────────────────────────────────────────────────┐
    │                 I/O Diagnosis Commands                     │
    ├────────────────────────────────────────────────────────────┤
    │ iotop -aoP           # Accumulated I/O by process          │
    │ iostat -xz 1         # Device stats every second           │
    │ pidstat -d 1         # Per-process I/O                     │
    │ biotop               # eBPF: block I/O by process          │
    │ biosnoop             # eBPF: trace each I/O with latency   │
    │ blktrace -d /dev/sda # Detailed block tracing              │
    │                                                            │
    │ Filesystem level:                                          │
    │ fileslower 10        # Files taking >10ms                  │
    │ filelife             # File create/delete events           │
    │ ext4slower / xfs*    # FS-specific latency                 │
    └────────────────────────────────────────────────────────────┘
```

---

## 4. Tool Selection Matrix

### Symptom to Tool Mapping

| Symptom | First Check | Deep Dive | Overhead |
|---------|-------------|-----------|----------|
| **High CPU** | `top`, `mpstat` | `perf record -g`, flame graph | Low/Med |
| **CPU imbalance** | `mpstat -P ALL` | `perf record -C <cpu>` | Low |
| **Slow syscalls** | `strace -c` | `perf trace`, `funclatency` | Med/High |
| **Memory leak** | `top`, `pmap` | `heaptrack`, `valgrind` | High |
| **OOM events** | `dmesg`, `journalctl` | `/proc/<pid>/oom_score` | None |
| **Disk slow** | `iostat -xz` | `biolatency`, `biotop` | Low |
| **Network drops** | `netstat -s`, `ss` | `tcpdump`, `dropwatch` | Med |
| **DNS slow** | `dig +trace` | `tcpdump port 53` | Low |
| **TCP retrans** | `ss -ti`, `netstat -s` | `tcpretrans` (bcc) | Low |
| **Lock contention** | `perf top` | `lockstat`, `offcputime` | Med |
| **Context switches** | `vmstat`, `pidstat -w` | `perf sched` | Med |

### Tool Overhead Reference

```
┌────────────────────────────────────────────────────────────────────┐
│                      TOOL OVERHEAD GUIDE                           │
├──────────────┬─────────────┬───────────────────────────────────────┤
│ Overhead     │ Tools       │ Notes                                 │
├──────────────┼─────────────┼───────────────────────────────────────┤
│ Negligible   │ uptime      │ Safe for any production system        │
│ (<0.1%)      │ free        │                                       │
│              │ df          │                                       │
│              │ vmstat      │                                       │
├──────────────┼─────────────┼───────────────────────────────────────┤
│ Low          │ top/htop    │ Generally safe for production         │
│ (<1%)        │ iostat      │                                       │
│              │ sar         │                                       │
│              │ ss/netstat  │                                       │
│              │ pidstat     │                                       │
│              │ mpstat      │                                       │
├──────────────┼─────────────┼───────────────────────────────────────┤
│ Medium       │ perf record │ Use sampling, limit duration          │
│ (1-5%)       │ tcpdump     │ Use filters, write to RAM disk        │
│              │ strace -c   │ Summary mode OK; -f can be heavy      │
│              │ bcc tools   │ Most are low, some higher             │
├──────────────┼─────────────┼───────────────────────────────────────┤
│ High         │ strace -f   │ Avoid in production if possible       │
│ (5-20%+)     │ valgrind    │ Dev/test only                         │
│              │ heaptrack   │ Dev/test only                         │
│              │ gdb         │ Stops process                         │
└──────────────┴─────────────┴───────────────────────────────────────┘
```

### Quick Reference: I See X, Use Y

```
I see high load average          → vmstat (check r and b columns)
I see high CPU but app is slow   → perf top, check %system vs %user
I see high %iowait               → iostat -xz, iotop
I see high %steal                → Contact cloud provider / migrate VM
I see swap usage                 → free -m, top (sort by RES)
I see OOM kills                  → dmesg, check process RSS limits
I see network timeouts           → ss -ti, mtr, tcpdump
I see TCP retransmissions        → netstat -s, check path with mtr
I see slow DNS                   → dig +trace, check /etc/resolv.conf
I see disk errors                → smartctl, dmesg, check /sys/block/*/stat
I see process in D state         → cat /proc/<pid>/stack, check I/O
I see high context switches      → pidstat -w, reduce thread count
```

---

## 5. Common Anti-Patterns

### Misdiagnoses and Corrections

```
┌────────────────────────────────────────────────────────────────────┐
│  LOOKS LIKE            │  ACTUALLY IS           │  HOW TO CONFIRM  │
├────────────────────────┼────────────────────────┼──────────────────┤
│ High CPU usage         │ High I/O wait          │ Check %wa in top │
│ (load average high)    │ (processes blocked)    │ vmstat 'b' column│
├────────────────────────┼────────────────────────┼──────────────────┤
│ Memory leak            │ Normal file cache      │ 'available' in   │
│ (used memory high)     │ growth                 │ free -m is fine  │
├────────────────────────┼────────────────────────┼──────────────────┤
│ Network problem        │ DNS resolution slow    │ ping by IP works │
│ (connections timeout)  │                        │ dig shows delay  │
├────────────────────────┼────────────────────────┼──────────────────┤
│ Disk is slow           │ RAID rebuild or        │ Check dmesg,     │
│ (high latency)         │ bad sector retry       │ smartctl         │
├────────────────────────┼────────────────────────┼──────────────────┤
│ App is CPU-bound       │ Lock contention        │ perf shows       │
│ (100% CPU one thread)  │ (spinning on mutex)    │ futex/mutex waits│
├────────────────────────┼────────────────────────┼──────────────────┤
│ Out of memory          │ Memory fragmentation   │ /proc/buddyinfo  │
│ (alloc fails)          │ (no contiguous blocks) │ shows fragments  │
├────────────────────────┼────────────────────────┼──────────────────┤
│ Network saturated      │ TCP buffer too small   │ ss -m shows tiny │
│ (low throughput)       │                        │ buffer sizes     │
├────────────────────────┼────────────────────────┼──────────────────┤
│ Storage bottleneck     │ Filesystem journal     │ Check journal    │
│ (sync writes slow)     │ on same device         │ device location  │
├────────────────────────┼────────────────────────┼──────────────────┤
│ Application bug        │ Noisy neighbor (VM)    │ Check %steal in  │
│ (intermittent slow)    │                        │ top/mpstat       │
├────────────────────────┼────────────────────────┼──────────────────┤
│ Kernel bug             │ Misconfigured limits   │ Check ulimit -a  │
│ (ENOMEM errors)        │ (file descriptors)     │ /proc/sys/fs/*   │
└────────────────────────┴────────────────────────┴──────────────────┘
```

### iowait Trap

```
WARNING: iowait can be misleading!

┌────────────────────────────────────────────────────────────────────┐
│ iowait = CPU idle time when there is outstanding I/O              │
│                                                                    │
│ TRAP 1: If you add CPU work, iowait drops (but I/O is still slow)│
│ TRAP 2: On multi-core systems, I/O-bound process shows low %wa   │
│ TRAP 3: NVMe is fast enough that elevated iowait may be CPU-side │
│                                                                    │
│ BETTER METRICS:                                                    │
│ - iostat await (actual I/O latency)                               │
│ - iostat avgqu-sz (queue depth)                                   │
│ - Process state: count 'D' state processes                        │
└────────────────────────────────────────────────────────────────────┘
```

### Load Average Misinterpretation

```
Load average includes:
  - Processes running on CPU (R state)
  - Processes in uninterruptible sleep (D state) - usually I/O

Load 8.0 on 4-core system could be:
  - 8 CPU-bound processes (CPU saturated)
  - 8 I/O-blocked processes (storage saturated, CPU idle)
  - Mix of both

ALWAYS correlate with:
  - vmstat: 'r' for run queue, 'b' for blocked
  - %user + %system vs %iowait
```

---

## 6. Quick Reference Card

```
┌────────────────────────────────────────────────────────────────────┐
│           LINUX PERFORMANCE TROUBLESHOOTING CHEATSHEET            │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  METHODOLOGIES                                                     │
│  ─────────────                                                     │
│  USE: Resources → Utilization, Saturation, Errors                 │
│  RED: Services  → Rate, Errors, Duration                          │
│                                                                    │
│  60-SECOND COMMANDS                                                │
│  ──────────────────                                                │
│  uptime          Load averages (1, 5, 15 min)                     │
│  dmesg -T|tail   Kernel errors, OOM, hardware                     │
│  vmstat 1        r=runnable, b=blocked, si/so=swap                │
│  mpstat -P ALL 1 Per-CPU breakdown, find imbalance                │
│  pidstat 1       Per-process CPU                                  │
│  iostat -xz 1    Disk: %util, await, avgqu-sz                     │
│  free -m         Memory: focus on 'available'                     │
│  sar -n DEV 1    Network throughput                               │
│  sar -n TCP,ETCP TCP stats, retransmissions                       │
│  top             Interactive overview                             │
│                                                                    │
│  KEY THRESHOLDS                                                    │
│  ──────────────                                                    │
│  CPU:    load > cores = investigate                               │
│  Memory: available < 10% = pressure                               │
│  Disk:   %util > 60% = likely bottleneck                          │
│          await > 10ms (SSD) = investigate                         │
│  Network: drops/errors > 0 = investigate                          │
│  Swap:   si/so > 0 = memory pressure                              │
│                                                                    │
│  PROCESS STATES                                                    │
│  ──────────────                                                    │
│  R = Running/runnable                                              │
│  S = Sleeping (interruptible)                                     │
│  D = Disk sleep (uninterruptible) ← often I/O blocked            │
│  Z = Zombie (needs parent wait())                                 │
│  T = Stopped (signal or trace)                                    │
│                                                                    │
│  QUICK DIAGNOSTICS                                                 │
│  ─────────────────                                                 │
│  CPU hog?        pidstat 1 | sort by %CPU                         │
│  Memory hog?     top, sort by RES (Shift+M)                       │
│  I/O hog?        iotop -aoP                                       │
│  Network hog?    iftop, nethogs                                   │
│  Open files?     lsof -p <pid> | wc -l                            │
│  File descriptors? ls /proc/<pid>/fd | wc -l                      │
│  Syscalls?       strace -c -p <pid>                               │
│  Stack trace?    cat /proc/<pid>/stack                            │
│                                                                    │
│  EMERGENCY COMMANDS                                                │
│  ──────────────────                                                │
│  echo 1 > /proc/sys/vm/drop_caches    # Free page cache          │
│  echo 1 > /proc/sys/vm/compact_memory # Defrag memory            │
│  kill -STOP <pid>                     # Pause process            │
│  ionice -c3 -p <pid>                  # Deprioritize I/O         │
│  renice +19 -p <pid>                  # Deprioritize CPU         │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 7. Google SRE Troubleshooting Workflow

For incident response, follow this structured approach:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SRE TROUBLESHOOTING WORKFLOW                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. TRIAGE                                                          │
│     └─→ How many users affected? Which regions? Severity?          │
│                                                                     │
│  2. MITIGATE FIRST                                                  │
│     └─→ Stop the bleeding BEFORE root-causing                      │
│         - Roll back recent changes                                  │
│         - Divert traffic                                            │
│         - Disable problematic feature                               │
│         - Scale up if capacity issue                                │
│                                                                     │
│  3. EXAMINE                                                         │
│     └─→ Start with metrics/dashboards (breadth-first)              │
│         - Error rates, latency spikes                               │
│         - Recent deployments                                        │
│         - Upstream/downstream health                                │
│         - Resource utilization (USE method)                         │
│                                                                     │
│  4. DIAGNOSE                                                        │
│     └─→ Drill into logs/traces only AFTER identifying component    │
│         - Correlate with changes                                    │
│         - Bisect the stack                                          │
│         - Test with known inputs                                    │
│                                                                     │
│  5. FIX & DOCUMENT                                                  │
│     └─→ Apply fix, verify recovery                                  │
│         - Write postmortem                                          │
│         - Update runbooks                                           │
│         - Add monitoring for this failure mode                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

KEY PRINCIPLE: "Fly the airplane first"
               Stabilize before investigating root cause
```

---

## Sources

- [Brendan Gregg's USE Method](https://www.brendangregg.com/usemethod.html)
- [USE Method: Linux Performance Checklist](https://www.brendangregg.com/USEmethod/use-linux.html)
- [Netflix: Linux Performance Analysis in 60,000 Milliseconds](https://netflixtechblog.com/linux-performance-analysis-in-60-000-milliseconds-accc10403c55)
- [The RED Method for Microservices](https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/)
- [Google SRE Book: Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/)
- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
