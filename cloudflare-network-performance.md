# Cloudflare Network Performance Tuning Reference

Extracted from Cloudflare Blog - production network tuning, eBPF, XDP, and high-throughput optimization.

---

## TCP Buffer & Window Management

### Production TCP Buffer Configuration

```bash
# High-throughput WAN optimization (Cloudflare production)
net.ipv4.tcp_rmem = 8192 262144 536870912
net.ipv4.tcp_wmem = 4096 16384 536870912
net.ipv4.tcp_adv_win_scale = -2
net.ipv4.tcp_collapse_max_bytes = 6291456
net.ipv4.tcp_notsent_lowat = 131072
```

**Parameter Breakdown:**
- `tcp_rmem`: min/default/max receive buffer (512MB max for 128MiB TCP window)
- `tcp_wmem`: min/default/max send buffer
- `tcp_adv_win_scale = -2`: 25% of buffer becomes advertised window
- `tcp_collapse_max_bytes = 6291456`: Allow collapse up to 6MB (~2ms latency)
- `tcp_notsent_lowat = 131072`: Threshold for unsent data notification

**Benchmarks Achieved:**
- Iowa-Marseille: 276 → 6600 mbps (24x)
- Melbourne-Marseille: 120 → 3800 mbps (32x)

---

## BBR Congestion Control

### Enable BBR (Linux 4.9+)

```bash
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
net.ipv4.tcp_notsent_lowat = 16384
```

**Performance Impact (HTTP/2):**
- Rendering start: 10.2s → 1.8s (5.6x faster)
- Visually complete: 10.7s → 4.5s (2.3x faster)

**Why BBR:**
- Model-based vs loss-based (CUBIC)
- Prevents bufferbloat
- Maintains accurate congestion window
- Critical for HTTP/2 multiplexed streams

---

## Mobile Web Performance Stack

```bash
# Disable slow start after idle
sysctl -w net.ipv4.tcp_slow_start_after_idle=0

# Initial window sizing (route-level)
ip route | while read p; do
  ip route change $p initcwnd 10 initrwnd 10
done
```

**Kernel Version Requirements:**

| Version | Feature |
|---------|---------|
| 2.6.38+ | initcwnd/initrwnd = 10 default |
| 3.2+ | PRR + initRTO = 1s |
| 3.5+ | TCP Early Retransmit |
| 4.9+ | BBR congestion control |

---

## Packet Processing Performance

### Dropping Performance by Layer (Single Core)

| Method | Throughput | Command/Config |
|--------|-----------|----------------|
| Application `recvmmsg()` | 175 kpps | Userspace discard |
| conntrack disabled | 333 kpps | `-t raw -j NOTRACK` |
| BPF socket filter | 512 kpps | `SO_ATTACH_FILTER` |
| iptables INPUT DROP | 608 kpps | `-I INPUT -j DROP` |
| iptables raw PREROUTING | 1.688 Mpps | `-t raw -I PREROUTING -j DROP` |
| nftables ingress | 1.53 Mpps | netdev table @ priority -500 |
| tc ingress | 1.8 Mpps | u32 match filters |
| **XDP** | **10 Mpps** | eBPF in driver context |

**XDP Example:**
```bash
ip link set dev eth0 xdp obj xdp-drop-ebpf.o
```

### Receiving Performance Optimizations

```bash
# SO_REUSEPORT for multi-process scaling
# Achieved 480k → 1.1M pps with 4 threads

# CPU pinning critical
taskset -c 0 ./receiver

# Batch syscalls
recvmmsg() with 1024 packets/call
sendmmsg() with 1024 packets/call
```

**NUMA Penalty:** 4x performance loss when receiver on wrong NUMA node.

---

## UDP/QUIC Optimizations

### GSO (Generic Segmentation Offload) - Linux 4.18+

```c
// UDP_SEGMENT socket option
int segment_size = 1200;
setsockopt(fd, IPPROTO_UDP, UDP_SEGMENT, &segment_size, sizeof(segment_size));
```

**Performance:**
- `sendmsg()`: 80-90 MB/s, ~900k syscalls
- `sendmmsg()`: batched, ~18k syscalls
- GSO: up to 64 segments per batch

### Pacing Options

```bash
# Kernel pacing via fq scheduler
SO_MAX_PACING_RATE  # rate-based pacing
SO_TXTIME / SCM_TXTIME  # scheduled transmission
```

---

## eBPF/XDP Architecture

### Cloudflare's 6-Layer eBPF Stack

1. **XDP Volumetric DoS**: L3 attack drops at NIC level
2. **XDP Load Balancing**: L4 distribution across servers
3. **Socket Rate Limiting**: `SO_ATTACH_BPF` with eBPF maps
4. **Application Helpers**: SOCKMAP for TCP splicing, TCP-BPF
5. **iptables Integration**: `xt_bpf` for payload matching
6. **Metrics**: `ebpf_exporter` for kernel metrics

### Socket Selection with eBPF

```c
// SO_ATTACH_REUSEPORT_EBPF for custom routing
bpf_sk_select_reuseport()  // select from SOCKMAP/SOCKHASH
```

### BPF Map Configuration

```go
// Example: hash map for TTL storage
bpfMapFd, err := ebpf.NewMap(ebpf.Hash, 4, 8, 5, 0)
// Requires: ulimit -l 10240 (locked memory)
```

---

## DDoS Mitigation Architecture

### Detection & Mitigation Layers

1. **Anycast Distribution**: Traffic spread across 180+ edge locations
2. **XDP Processing**: Wire-speed filtering at NIC
3. **Dynamic Signatures**: `dosd` daemon generates fingerprints
4. **Multi-Level Detection**: Server → Datacenter → Global
5. **Rule Gossip**: Mitigation rules multicast across infrastructure

**Attack Scale Handled:**
- 3.8 Tbps peak (65 seconds)
- 2.14 billion pps (60 seconds)

---

## SO_REUSEPORT Configuration

### Basic Setup

```c
int enable = 1;
setsockopt(fd, SOL_SOCKET, SO_REUSEPORT, &enable, sizeof(enable));
// Set after socket(), before bind()
```

### Distribution Methods

1. **Default**: Kernel hash on 4-tuple
2. **SO_INCOMING_CPU**: Same-CPU steering
3. **SO_ATTACH_REUSEPORT_EBPF**: Custom eBPF logic

### Socket Dispatch (tubular)

```bash
# eBPF at sk_lookup hook
# State in /sys/fs/bpf/
# Bindings: LPM trie (protocol/port/prefix)
# Sockets: Array indexed by destination ID
```

---

## Port Selection Optimization

### Problem: Even/Odd Port Latency Split

```
Even ports: ~0.025ms average
Odd ports:  ~4.59ms average (183x slower)
```

### Solutions

**Kernel 6.8+:**
```c
// Full system range randomization
setsockopt(fd, IPPROTO_IP, IP_LOCAL_PORT_RANGE, ...);
// Result: 0.029ms uniform latency
```

**Pre-6.8 Workaround:**
```c
// Random 500-1000 port window offset
IP_LOCAL_PORT_RANGE + IP_BIND_ADDRESS_NO_PORT
// Result: ~0.03ms with minimal errors
```

---

## io_uring Worker Configuration (Linux 5.15+)

### Worker Limits

```c
// IORING_REGISTER_IOWQ_MAX_WORKERS (0x13)
unsigned int workers[2] = {
  bounded_max,    // Index 0: file I/O workers
  unbounded_max   // Index 1: socket/char device workers
};
io_uring_register(ring_fd, 0x13, workers, 2);
```

**Scaling Behavior:**
- Workers scale per NUMA node
- Unbounded workers limited by `RLIMIT_NPROC`
- Bounded workers limited by SQ ring size

---

## Kernel Security Tunables

### Module & Execution Security

```bash
# Kernel config
CONFIG_MODULE_SIG_FORCE=y
CONFIG_KEXEC=n
CONFIG_KEXEC_FILE=y
CONFIG_KEXEC_SIG=y
CONFIG_KEXEC_SIG_FORCE=y
CONFIG_RANDOMIZE_BASE=y  # KASLR
CONFIG_SECURITY_DMESG_RESTRICT=y
CONFIG_LOCK_DOWN_KERNEL_FORCE_INTEGRITY=y
```

### Runtime Parameters

```bash
kernel.kptr_restrict=1    # Hide kernel pointers
kernel.dmesg_restrict=1   # Restrict kernel logs
```

---

## Kernel Bypass Techniques

### Performance Comparison (Single Core)

| Technique | Throughput | Trade-off |
|-----------|-----------|-----------|
| Vanilla Linux | 1M pps | Full stack features |
| PACKET_MMAP | 1.4M pps | Requires full NIC |
| PF_RING ZC | >1M pps | Non-mainline module |
| DPDK | >10M pps | Full NIC dedication |
| Netmap | >10M pps | Driver patching |
| **Partial Bypass** | 1.4M pps/queue | Kernel retains NIC |

**Cloudflare Approach:** Partial kernel bypass - kernel owns NIC, bypass on single RX queue.

---

## Quick Reference: Essential Sysctls

```bash
# === TCP Performance ===
net.ipv4.tcp_rmem = 8192 262144 536870912
net.ipv4.tcp_wmem = 4096 16384 536870912
net.ipv4.tcp_adv_win_scale = -2
net.ipv4.tcp_collapse_max_bytes = 6291456
net.ipv4.tcp_notsent_lowat = 131072
net.ipv4.tcp_slow_start_after_idle = 0

# === BBR (Linux 4.9+) ===
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr

# === Security ===
kernel.kptr_restrict = 1
kernel.dmesg_restrict = 1
```

---

## Sources

- [Optimizing TCP for high throughput and low latency](https://blog.cloudflare.com/optimizing-tcp-for-high-throughput-and-low-latency/)
- [HTTP/2 prioritization with BBR](https://blog.cloudflare.com/http-2-prioritization-with-nginx/)
- [How to drop 10 million packets](https://blog.cloudflare.com/how-to-drop-10-million-packets/)
- [How to receive a million packets](https://blog.cloudflare.com/how-to-receive-a-million-packets/)
- [Cloudflare Architecture and BPF](https://blog.cloudflare.com/cloudflare-architecture-and-how-bpf-eats-the-world/)
- [Tubular: eBPF Socket API](https://blog.cloudflare.com/tubular-fixing-the-socket-api-with-ebpf/)
- [connect() port selection](https://blog.cloudflare.com/linux-transport-protocol-port-selection-performance/)
- [io_uring worker pool](https://blog.cloudflare.com/missing-manuals-io_uring-worker-pool/)
- [Accelerating UDP for QUIC](https://blog.cloudflare.com/accelerating-udp-packet-transmission-for-quic/)
- [Linux kernel hardening](https://blog.cloudflare.com/linux-kernel-hardening/)
- [Kernel bypass](https://blog.cloudflare.com/kernel-bypass/)
- [Auto-mitigated 3.8 Tbps DDoS](https://blog.cloudflare.com/how-cloudflare-auto-mitigated-world-record-3-8-tbps-ddos-attack/)
- [Optimizing Linux for Mobile Web](https://blog.cloudflare.com/optimizing-the-linux-stack-for-mobile-web-per/)
