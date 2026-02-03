# Meta/Facebook eBPF and Systems Engineering

Curated technical insights from Meta Engineering on BPF/eBPF, kernel development, and infrastructure at hyperscale.

---

## 1. Katran: BPF-Based L4 Load Balancer

**Source**: [Open-sourcing Katran](https://engineering.fb.com/2018/05/22/open-source/open-sourcing-katran-a-scalable-network-load-balancer/)

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Network Switch                            │
│                    (ECMP via ExaBGP VIP)                        │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                     XDP BPF Program                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐  │
│  │ 5-tuple     │───▶│ Maglev Hash │───▶│ IP-in-IP Encap      │  │
│  │ Extraction  │    │ (L1 cache)  │    │ (DSR mode)          │  │
│  └─────────────┘    └─────────────┘    └─────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
                      Backend Servers
```

### Key Techniques

| Technique | Implementation | Benefit |
|-----------|----------------|---------|
| XDP hook | Packet processing immediately after NIC receipt | Bypasses kernel stack |
| Per-CPU maps | Lockless data structures | Linear scaling with RX queues |
| Maglev hash | Extended consistent hashing on 5-tuple | No cross-LB synchronization |
| LRU cache | Connection tracking with eviction | Hash computation cheaper than L3 cache lookup |
| RSS-friendly encap | Different outer src IPs per flow | Proper Receive Side Scaling alignment |

### Design Constraints

- Direct Server Return (DSR) mode only
- No IP fragmentation support
- No IP options support
- 3.5 KB maximum packet size
- L3-based network topology required
- Single-interface ingress/egress ("LB on a stick")

### Deployment Pattern

Collocate load balancers with backend servers on same hosts. Benefits:
- Improved LB capacity ratios
- Increased resilience to host failures
- Eliminated dedicated L4LB tier

---

## 2. Strobelight: Fleet-Wide Profiling

**Source**: [Strobelight Profiling Service](https://engineering.fb.com/2025/01/21/production-engineering/strobelight-a-profiling-service-built-on-open-source-technology/)

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Strobelight Agent                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │ 42 Profilers │  │ bpftrace    │  │ Dynamic Sampling     │   │
│  │ (eBPF-based) │  │ ad-hoc      │  │ (auto-adjust daily)  │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Symbolization Pipeline                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │ gsym         │  │ blazesym    │  │ DWARF/ELF parsing    │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Analysis Layer                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │ Scuba (SQL)  │  │ Tracery     │  │ Flame Graphs         │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 42 Profilers Include

- Memory profilers (jemalloc integration)
- Function call count profilers
- Event-based profilers (Python, Java, Erlang)
- AI/GPU profilers
- Off-CPU time trackers
- Service request latency profilers
- Last Branch Record (LBR) profiler → FDO pipeline

### Ad-Hoc Debugging Pattern

```
# Engineers write bpftrace scripts
# Deploy across fleet within hours
# No standard review cycle for rapid debugging
```

### Key Features

| Feature | Implementation |
|---------|----------------|
| Dynamic Sampling | Auto-adjust run probabilities daily per-service |
| Weighted Normalization | Cross-host comparisons via sample weighting |
| Stack Schemas | DSL for tagging/filtering call stacks |
| Strobemeta | Thread-local metadata for endpoint/latency attribution |

### Performance Impact

- LBR profiler feeds Feedback-Directed Optimization (FDO)
- 10-20% CPU reduction in top services via FDO

---

## 3. Millisampler: Fine-Grained Network Analysis

**Source**: [Millisampler Network Traffic Analysis](https://engineering.fb.com/2023/04/17/networking-traffic/millisampler-network-traffic-analysis/)

### Implementation

```c
// eBPF tc filter - runs at packet ingress/egress
// "among the first programmable steps on receipt,
//  near last step on transmission"

// Per-CPU variables to eliminate contention
// Fixed memory: 2000 x 64-bit counters per CPU per metric
```

### Sampling Configuration

| Interval | Samples | Observation Window |
|----------|---------|-------------------|
| 100 μs | 2,000 | 200 ms (sub-RTT) |
| 1 ms | 2,000 | 2 s |
| 10 ms | 2,000 | 20 s (cross-region RTT) |

### Metrics Captured

- Ingress/egress bytes
- ECN-marked bytes
- Active flow counts (128-bit sketch)
- Retransmission patterns
- In-region vs cross-region classification

### Real-World Discoveries

- Synchronized traffic bursts at fine time scales
- NIC driver bug: stopped delivering packets for milliseconds

---

## 4. SSLWall: Encryption Enforcement via eBPF

**Source**: [Enforcing Encryption at Scale](https://engineering.fb.com/2021/07/12/security/enforcing-encryption/)

### BPF Hook Points

```
┌─────────────────────────────────────────────────────────────────┐
│                       Kernel Space                               │
│                                                                  │
│  ┌──────────────────┐         ┌──────────────────────────────┐  │
│  │ kprobe:          │         │ tc-bpf filter                │  │
│  │ tcp_connect      │────────▶│ (packet inspection)          │  │
│  │ tcp_v6_destroy   │         │                              │  │
│  └──────────────────┘         └──────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Optimization Strategies

| Challenge | Solution |
|-----------|----------|
| High QPS services | Early exit conditions |
| CPU impact | BPF arrays over LRU maps where possible |
| TCP Fast Open | Flow tracking via kprobe prehandler |
| Size/complexity limits | Many optimization cycles |

### Architecture Pattern

- Daemon bundles BPF programs
- Config via Configerator
- Logs via perf events → Scribe
- Eliminates user/kernel version compatibility issues

---

## 5. NetEdit: eBPF Program Management at Scale

**Source**: [NetEdit Managing eBPF at Scale](https://blog.apnic.net/2025/06/05/building-netedit-managing-ebpf-programs-at-scale-at-meta/)

### Challenge

Meta operates 10+ kernel versions simultaneously.

### Framework Components

```
┌─────────────────────────────────────────────────────────────────┐
│                        NetEdit                                   │
│                                                                  │
│  ┌──────────────────┐    ┌──────────────────────────────────┐   │
│  │ BPFAdapter       │    │ PolicyEngine                     │   │
│  │ ─────────────────│    │ ─────────────────────────────────│   │
│  │ • Kernel abstraction   │ • tuningFeature granularity      │   │
│  │ • Explicit GC    │    │ • Dynamic loading decisions      │   │
│  │ • Lazy loading   │    │ • Stability > optimality         │   │
│  │ • Shared maps    │    │                                  │   │
│  └──────────────────┘    └──────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Pain Points Addressed

| Problem | BPFAdapter Solution |
|---------|---------------------|
| Kernel heuristics | Explicit garbage collection |
| Kernel-BPF interface changes | Abstraction layer per hookpoint |
| BPF-BPF composition bugs | Modular, tested components |

### Deployed tuningFeatures (12+ over 5 years)

| Feature | Result |
|---------|--------|
| Initial CWND tuning | 30% faster image generation |
| TCP receiver window | 100% faster storage reads |
| BPF-based DCTCP | 75% reduction in network retransmits |

### Development Acceleration

~6 months → ~weeks for new tuningFeatures via capability store.

---

## 6. Linux Kernel Components (Open Source)

**Source**: [Facebook Open-Sources Linux Kernel Components](https://engineering.fb.com/2018/10/30/open-source/linux/)

### Production Components

| Component | Purpose | Key Benefit |
|-----------|---------|-------------|
| **Btrfs** | COW filesystem | Eliminated priority inversions from journaling |
| **Cgroup2** | Resource control | Multi-tenancy via memory overcommit handling |
| **PSI** | Pressure Stall Information | Canonical resource shortage metrics |
| **Oomd** | Userspace OOM killer | Flexible plugin system, pre-kernel OOM action |
| **Netconsd** | UDP netconsole daemon | Rapid kernel log diagnosis |

### PSI Metrics

```
# /proc/pressure/cpu
# /proc/pressure/memory
# /proc/pressure/io

# Format: some avg10=X.XX avg60=X.XX avg300=X.XX total=XXXXX
#         full avg10=X.XX avg60=X.XX avg300=X.XX total=XXXXX
```

---

## 7. Zoomer: AI Performance Optimization

**Source**: [Zoomer AI Performance](https://engineering.fb.com/2025/11/21/data-infrastructure/zoomer-powering-ai-performance-meta-intelligent-debugging-optimization/)

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Zoomer Platform                               │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Infrastructure: Manifold (distributed storage)          │    │
│  │                 Fault-tolerant trace processing         │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Analytics: GPU metrics (SM util, mem BW, Tensor Core)   │    │
│  │            PyTorch Profiler + Kineto                    │    │
│  │            StrobeLight CPU profiling                    │    │
│  │            NCCL collective analysis                     │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Automation: Straggler detection across ranks            │    │
│  │             Bottleneck classification                   │    │
│  │             Critical path identification                │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Results

| Optimization | Impact |
|--------------|--------|
| Ads model training | 75% time reduction |
| Power consumption | 78% reduction (ads), 10-45% (general) |
| QPS improvements | 2-50% via single-click workflows |

---

## 8. sched_ext + LAVD: BPF-Based Scheduler

**Source**: [Meta SCX-LAVD Deployment](https://www.phoronix.com/news/Meta-SCX-LAVD-Steam-Deck-Server)

### Background

- LAVD = Latency-Aware Virtual Deadline
- Originally developed for Valve's Steam Deck
- Implemented in BPF + Rust via sched_ext framework
- Linux kernel 6.12+ required

### Meta's Adaptations

| Challenge | Solution |
|-----------|----------|
| Multi-core contention | Improved scheduling queue management |
| Pinned tasks | Better interference handling |
| Network interrupt overhead | Treat interrupt-heavy cores as "slower" CPUs |
| Cache locality | Enhanced CCX/LLC boundary awareness |

### Deployment

- Default fleet scheduler across diverse hardware
- Messaging backends, caching services
- No specialized scheduler needed for most use cases

---

## 9. BPF Tools and Tracing

### bpftrace (Production Use)

```bash
# One-liner examples used at Meta/Netflix

# Trace block I/O latency
bpftrace -e 'tracepoint:block:block_rq_complete { @us = hist(args->sector); }'

# Count syscalls by process
bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'

# Profile CPU stack samples
bpftrace -e 'profile:hz:99 { @[kstack] = count(); }'
```

### BCC Tools

Meta contributes to and uses BCC tools including:
- `opensnoop` - trace file opens
- `execsnoop` - trace new processes
- `tcpconnect` - trace TCP connections
- `biolatency` - block I/O latency histogram
- `funccount` - count function calls

### Key Libraries

| Library | Purpose |
|---------|---------|
| libbpf | BPF program loading and interaction |
| BTF | Type information for CO-RE |
| blazesym | Symbolization |
| gsym | Compact symbol format |

---

## 10. Kernel Parameters (Derived from Practices)

### Network Tuning (inferred from Katran/Millisampler)

```bash
# RSS/RPS configuration
/sys/class/net/<dev>/queues/rx-*/rps_cpus

# XDP mode
ip link set dev <dev> xdp obj katran.o

# Increase socket buffers for high-throughput
net.core.rmem_max
net.core.wmem_max
net.ipv4.tcp_rmem
net.ipv4.tcp_wmem
```

### BPF Limits (inferred from SSLWall)

```bash
# Increase BPF complexity limit (kernel 5.2+)
# Default: 1M instructions
/proc/sys/kernel/bpf_jit_limit

# Memory limits for maps
/proc/sys/kernel/bpf_stats_enabled
```

### Memory/OOM (inferred from Oomd/PSI)

```bash
# Enable PSI
CONFIG_PSI=y

# PSI monitoring thresholds for oomd
memory.pressure_stall_some_threshold
memory.pressure_stall_full_threshold
```

---

## Summary: Meta's BPF Philosophy

1. **XDP for packet processing** - Hook as early as possible
2. **Per-CPU data structures** - Eliminate lock contention
3. **Hash > cache lookup** - When data fits in L1
4. **Abstraction layers** - BPFAdapter for kernel version isolation
5. **Fleet-wide profiling** - 42 profilers, dynamic sampling
6. **bpftrace for ad-hoc** - Deploy custom debugging in hours
7. **BPF schedulers** - sched_ext + LAVD as default fleet scheduler
8. **PSI + Oomd** - Proactive resource management
9. **Shared infrastructure** - Collocate LBs with backends

---

## References

- [Katran Open Source](https://github.com/facebookincubator/katran)
- [BCC Tools](https://github.com/iovisor/bcc)
- [bpftrace](https://github.com/bpftrace/bpftrace)
- [sched_ext](https://github.com/sched-ext/scx)
- [Brendan Gregg's BPF Tools](https://www.brendangregg.com/ebpf.html)
