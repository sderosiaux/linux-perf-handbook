# Network Tuning

TCP optimization, congestion control, NUMA-aware networking, and protocol performance.

## TCP Congestion Control

### BBR vs CUBIC

```bash
# Check current
sysctl net.ipv4.tcp_congestion_control

# Available algorithms
sysctl net.ipv4.tcp_available_congestion_control

# Enable BBR
sysctl -w net.ipv4.tcp_congestion_control=bbr
sysctl -w net.core.default_qdisc=fq

# Enable CUBIC (default)
sysctl -w net.ipv4.tcp_congestion_control=cubic
```

| Algorithm | Best For |
|-----------|----------|
| BBR | High latency, lossy networks, WAN |
| CUBIC | Default, LAN, low latency |
| DCTCP | Datacenter (requires ECN) |

### TCP Fast Open

```bash
# Enable TFO (client + server)
sysctl -w net.ipv4.tcp_fastopen=3

# Values:
# 0 = disabled
# 1 = client only
# 2 = server only
# 3 = client + server
```

## Buffer Sizes

### Core Buffers

```bash
# Socket buffer defaults (bytes)
sysctl -w net.core.rmem_default=262144
sysctl -w net.core.wmem_default=262144

# Socket buffer max
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216

# Optmem (ancillary buffer)
sysctl -w net.core.optmem_max=65536
```

### TCP Buffers

```bash
# TCP memory autotuning (pages)
# min, default, max
sysctl -w net.ipv4.tcp_mem="8388608 12582912 16777216"

# TCP receive buffer (bytes)
sysctl -w net.ipv4.tcp_rmem="4096 87380 16777216"

# TCP send buffer (bytes)
sysctl -w net.ipv4.tcp_wmem="4096 65536 16777216"

# Enable autotuning (default)
sysctl -w net.ipv4.tcp_moderate_rcvbuf=1
```

### UDP Buffers

```bash
sysctl -w net.ipv4.udp_mem="8388608 12582912 16777216"
sysctl -w net.ipv4.udp_rmem_min=16384
sysctl -w net.ipv4.udp_wmem_min=16384
```

## Connection Handling

### Backlog & Queues

```bash
# Listen backlog
sysctl -w net.core.somaxconn=65535

# SYN backlog
sysctl -w net.ipv4.tcp_max_syn_backlog=65535

# Network device backlog
sysctl -w net.core.netdev_max_backlog=65535
sysctl -w net.core.netdev_budget=600
sysctl -w net.core.netdev_budget_usecs=8000
```

### Connection Tracking

```bash
# Max tracked connections
sysctl -w net.nf_conntrack_max=1048576
sysctl -w net.netfilter.nf_conntrack_max=1048576

# Conntrack timeout
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=86400
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30

# Check current count
sysctl net.netfilter.nf_conntrack_count
```

### TIME_WAIT

```bash
# Reduce TIME_WAIT duration
sysctl -w net.ipv4.tcp_fin_timeout=15

# Reuse TIME_WAIT sockets
sysctl -w net.ipv4.tcp_tw_reuse=1

# Max TIME_WAIT buckets
sysctl -w net.ipv4.tcp_max_tw_buckets=1440000
```

### Port Range

```bash
# Local port range
sysctl -w net.ipv4.ip_local_port_range="1024 65535"
```

## TCP Options

### Keep-Alive

```bash
# Keep-alive settings
sysctl -w net.ipv4.tcp_keepalive_time=600    # 10 min before probe
sysctl -w net.ipv4.tcp_keepalive_intvl=60    # 1 min between probes
sysctl -w net.ipv4.tcp_keepalive_probes=3    # 3 probes
```

### Retries

```bash
# SYN retries
sysctl -w net.ipv4.tcp_syn_retries=3
sysctl -w net.ipv4.tcp_synack_retries=3

# Connection retries
sysctl -w net.ipv4.tcp_retries1=3
sysctl -w net.ipv4.tcp_retries2=8
```

### Features

```bash
# SACK (Selective ACK)
sysctl -w net.ipv4.tcp_sack=1

# Window scaling
sysctl -w net.ipv4.tcp_window_scaling=1

# Timestamps (PAWS, RTT)
sysctl -w net.ipv4.tcp_timestamps=1

# No slow start after idle
sysctl -w net.ipv4.tcp_slow_start_after_idle=0

# No metrics save
sysctl -w net.ipv4.tcp_no_metrics_save=1

# MTU probing
sysctl -w net.ipv4.tcp_mtu_probing=1
```

## NUMA-Aware Networking

### RSS (Receive Side Scaling)

Distributes packets across CPUs using hardware hash.

```bash
# Check RSS queues
ethtool -l eth0

# Set RSS queues
ethtool -L eth0 combined 8

# Check RSS indirection table
ethtool -x eth0

# Set RSS hash key
ethtool -X eth0 equal 8
```

### RPS (Receive Packet Steering)

Software-based RSS for NICs without hardware support.

```bash
# Enable RPS (hex CPU mask)
echo f > /sys/class/net/eth0/queues/rx-0/rps_cpus  # CPUs 0-3
echo ff > /sys/class/net/eth0/queues/rx-0/rps_cpus # CPUs 0-7

# Flow limit
echo 4096 > /sys/class/net/eth0/queues/rx-0/rps_flow_cnt
sysctl -w net.core.rps_sock_flow_entries=32768
```

### RFS (Receive Flow Steering)

Steers flows to CPUs where application runs.

```bash
# Enable RFS
sysctl -w net.core.rps_sock_flow_entries=32768
echo 4096 > /sys/class/net/eth0/queues/rx-0/rps_flow_cnt
```

### XPS (Transmit Packet Steering)

Maps TX queues to CPUs.

```bash
# Set XPS (hex CPU mask per queue)
echo 1 > /sys/class/net/eth0/queues/tx-0/xps_cpus
echo 2 > /sys/class/net/eth0/queues/tx-1/xps_cpus
echo 4 > /sys/class/net/eth0/queues/tx-2/xps_cpus
```

### Busy Polling

Reduces latency by polling instead of interrupt.

```bash
# Enable busy polling
sysctl -w net.core.busy_poll=50          # 50 μs
sysctl -w net.core.busy_read=50          # 50 μs

# Per-socket (application code)
setsockopt(fd, SOL_SOCKET, SO_BUSY_POLL, &timeout, sizeof(timeout));
```

### IRQ Affinity

```bash
# View IRQ affinity
cat /proc/interrupts | grep eth0
cat /proc/irq/IRQ_NUM/smp_affinity

# Set IRQ affinity (hex CPU mask)
echo 1 > /proc/irq/IRQ_NUM/smp_affinity    # CPU 0
echo 2 > /proc/irq/IRQ_NUM/smp_affinity    # CPU 1

# Or use irqbalance
systemctl stop irqbalance
```

## NIC Tuning

### Ring Buffers

```bash
# Check current
ethtool -g eth0

# Set ring buffer size
ethtool -G eth0 rx 4096 tx 4096
```

### Offloading

```bash
# View offload settings
ethtool -k eth0

# Enable/disable features
ethtool -K eth0 tso on       # TCP Segmentation Offload
ethtool -K eth0 gro on       # Generic Receive Offload
ethtool -K eth0 gso on       # Generic Segmentation Offload
ethtool -K eth0 lro off      # Large Receive Offload (often bad)
ethtool -K eth0 rx-checksum on
ethtool -K eth0 tx-checksum on
```

### Coalescing

```bash
# View coalesce settings
ethtool -c eth0

# Set coalescing (reduce interrupts, increase latency)
ethtool -C eth0 rx-usecs 50 tx-usecs 50

# Adaptive coalescing
ethtool -C eth0 adaptive-rx on adaptive-tx on
```

## Traffic Shaping

### tc (Traffic Control)

```bash
# Add latency
tc qdisc add dev eth0 root netem delay 100ms

# Add latency with jitter
tc qdisc add dev eth0 root netem delay 100ms 20ms distribution normal

# Add packet loss
tc qdisc add dev eth0 root netem loss 1%

# Rate limit
tc qdisc add dev eth0 root tbf rate 100mbit burst 32kbit latency 400ms

# Remove
tc qdisc del dev eth0 root
```

### iperf3 (Bandwidth Testing)

```bash
# Server
iperf3 -s

# Client
iperf3 -c server              # TCP test
iperf3 -c server -u           # UDP test
iperf3 -c server -P 10        # 10 parallel streams
iperf3 -c server -R           # Reverse (server sends)
iperf3 -c server -t 60        # 60 second test
iperf3 -c server -b 1G        # 1 Gbps bandwidth
```

## gRPC Optimization

Default gRPC-Go uses reflection for marshalling, adding ~12% overhead. Replace with vtprotobuf for code-generated serialization.

```bash
# Install vtprotobuf plugin
go install github.com/planetscale/vtprotobuf/cmd/protoc-gen-go-vtproto@latest

# Generate with vtprotobuf
protoc --go_out=. --go-grpc_out=. --go-vtproto_out=. myservice.proto
```

```go
// Register faster codec (grpc-go)
import "google.golang.org/grpc/encoding"

type vtCodec struct{}
func (vtCodec) Marshal(v interface{}) ([]byte, error) {
    return v.(interface{ MarshalVT() ([]byte, error) }).MarshalVT()
}
func (vtCodec) Unmarshal(data []byte, v interface{}) error {
    return v.(interface{ UnmarshalVT([]byte) error }).UnmarshalVT(data)
}
func init() { encoding.RegisterCodec(vtCodec{}) }
```

For streams, use `RecvMsg()` with pooled messages to avoid heap allocation per message. gRPC-go 1.66+ enables receive buffer pooling by default. For older versions:

```go
grpc.NewServer(grpc.ReadBufferSize(0)) // Enables experimental receive buffer pool
```

**Key insight:** Reducing allocations from 11 GB/s to 20 MB/s cut CPU usage by 50% for 5 Gbps throughput.

## TLS/Crypto Acceleration

VPP (Vector Packet Processing) TLS plugin can offload crypto to hardware using OpenSSL async jobs and DPDK crypto PMD.

```bash
# VPP config for async TLS
tls {
    async
    engine openssl
}

# DPDK crypto device binding
dpdk-devbind.py -b vfio-pci 0000:06:00.0  # crypto accelerator
```

Performance comparison (RSA-4K handshakes):
- Software only: ~6-7 Gbps
- Hardware accelerated with async: 20% higher throughput, 50% lower CPU

**Key insight:** Async crypto jobs let the CPU continue application work while hardware processes RSA/AES, critical for TLS handshake-heavy workloads.

## HTTP/2, HTTP/3, QUIC

Real-world protocol performance from browser telemetry:

| Protocol | Time to Request Start (75th percentile, Android) |
|----------|--------------------------------------------------|
| HTTP/1.1 | ~320ms |
| HTTP/2   | ~260ms |
| HTTP/3   | ~130ms |

DNS over HTTPS (DoH) optimization:
- Long-lived connections to DoH server are critical
- Server-side: Set `TCP_NODELAY` on DoH endpoints
- With QUIC 0-RTT, DoH latency matches legacy UDP DNS

Current adoption (browser telemetry):
- HTTPS: 98% of traffic
- HTTP/3: 20-30% of connections
- ECH (Encrypted Client Hello): 0.3% (low server support)
- DoH: 12% of DNS queries

**Key insight:** QUIC 0-RTT resumption eliminates handshake overhead, making encrypted DNS viable without latency penalty.

## Link Aggregation/Bonding

Linux kernel supports OVS-style balance-slb (source load balancing) without switch cooperation.

```bash
# New kernel transmit hash policy (kernel 6.x+)
nmcli connection add type bond con-name bond0 ifname bond0 \
    bond.options "mode=balance-xor,xmit_hash_policy=vlan+srcmac"

# Enable balance-slb in NetworkManager
nmcli connection modify bond0 bond.options \
    "mode=balance-xor,xmit_hash_policy=vlan+srcmac,balance-slb=1"
```

nftables rules to prevent duplicate broadcast/multicast packets:
```bash
nft add table netdev bond_slb
nft add set netdev bond_slb seen { type ether_addr . vlan_id \; flags dynamic,timeout \; timeout 30s \; }
nft add chain netdev bond_slb ingress { type filter hook ingress device bond0 priority 0 \; }
nft add rule netdev bond_slb ingress ether type vlan vlan id . ether saddr @seen drop
nft add rule netdev bond_slb ingress ether type vlan vlan id . ether saddr update @seen
```

nftables 1.0.6+ added `destroy` command for idempotent cleanup:
```bash
nft destroy table netdev bond_slb  # No error if missing
```

**Key insight:** balance-slb enables multi-path without LACP, but requires nftables rules to deduplicate broadcast traffic arriving on multiple ports.

## Latency Analysis Tools

Tools for tracing packet latency through hypervisor to VM:

```bash
# pwru (Packet, Where Are You) - trace kernel functions per packet
pwru --filter-dst-port 102 --output-tuple

# bpftrace for latency measurement between kernel functions
bpftrace -e '
kprobe:skb_recv_done { @start[arg0] = nsecs; }
kprobe:consume_skb {
    $lat = nsecs - @start[arg0];
    @latency = hist($lat / 1000);  // microseconds
    delete(@start[arg0]);
}'
```

SVTrace wrapper combines bpftrace with real-time visualization:
```bash
svtrace --live --interface eth0 --filter "ether proto 0x88ba"
```

Key optimizations for virtualized networks:
- Isolate KVM vCPUs: `isolcpus=4-7` + `taskset -c 4-7 qemu-system-x86_64`
- Set vhost thread priority higher than QEMU main thread
- Result: Reduced latency from 200us to 40us for packet transit

**Key insight:** OVS upcalls to userspace cause latency spikes. For real-time, use kernel datapath or avoid OVS entirely.

## Traffic Fingerprinting

nDPI library fingerprints OS/application from packet metadata without payload inspection.

```bash
# TCP fingerprint components
# TTL range + Window size + TCP options + MSS
# Example: Windows lacks TCP timestamps, Apple devices have distinct option ordering

# View JA4 TLS fingerprint in Wireshark
wireshark -Y "tls.handshake.type == 1" -T fields -e ja4.hash
```

TCP fingerprint detection:
```bash
# Detect high-speed scanners (masscan, zmap) - no TCP options
tcpdump -n 'tcp[tcpflags] == tcp-syn and tcp[20:4] == 0'
```

JA4 TLS fingerprint (replaces JA3):
- Sorts cipher suites/extensions to counter randomization (GREASE)
- Format: `protocol_version + cipher_count + extension_count + sorted_ciphers_hash + sorted_extensions_hash`

**Key insight:** Encrypted traffic still leaks client identity through TLS ClientHello structure, TCP options, and connection patterns.

## Real-Time Video/Media

RTCP feedback mechanisms for adaptive bitrate over unstable networks:

| Feedback Type | Purpose | Latency Impact |
|--------------|---------|----------------|
| REMB (Receiver Estimated Max Bitrate) | Bandwidth estimation from inter-packet arrival | Proactive |
| NACK | Request retransmission of lost packets | 1-2 frames |
| PLI (Picture Loss Indication) | Request full keyframe (10x size) | 100-500ms |

REMB bandwidth estimation:
```
# Detects buffer buildup before packet loss
# If inter-packet arrival time grows -> network congestion imminent
# Reduce bitrate proactively
```

Test results (race car at 160 kph, 12 cell handovers per lap):
- Without RTCP adaptation: Frame rate drops to 0 frequently
- With RTCP REMB + NACK: Constant frame rate, 50x reduction in frozen frames

L4S (Low Latency Low Loss Scalable throughput):
- RFC 8888 adds RTCP feedback for ECN congestion bits
- Network equipment marks packets approaching queue limits
- Enables earlier congestion detection than packet loss

**Key insight:** Varying bitrate based on RTCP feedback maintains constant frame rate over unstable 5G; fixed bitrate causes frequent freezes.

## High-Performance Config

### Server Profile

**/etc/sysctl.d/99-network-performance.conf:**
```bash
# Congestion control
net.ipv4.tcp_congestion_control = bbr
net.core.default_qdisc = fq

# Buffers
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.rmem_default = 262144
net.core.wmem_default = 262144
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# Connections
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.core.netdev_max_backlog = 65535

# Fast open
net.ipv4.tcp_fastopen = 3

# TIME_WAIT
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1

# Keep-alive
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 60
net.ipv4.tcp_keepalive_probes = 3

# Disable slow start after idle
net.ipv4.tcp_slow_start_after_idle = 0

# Port range
net.ipv4.ip_local_port_range = 1024 65535
```

## Kernel Bypass Technologies

When the standard Linux network stack becomes the bottleneck.

### Overview

```
Traditional Stack:        XDP:                    AF_XDP:                 DPDK:
┌─────────────────┐      ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Application   │      │   Application   │     │   Application   │     │   Application   │
├─────────────────┤      ├─────────────────┤     ├─────────────────┤     ├─────────────────┤
│    Socket API   │      │    Socket API   │     │   AF_XDP Socket │     │    DPDK PMD     │
├─────────────────┤      ├─────────────────┤     │   (zero-copy)   │     │   (poll mode)   │
│   TCP/IP Stack  │      │   TCP/IP Stack  │     ├─────────────────┤     ├─────────────────┤
├─────────────────┤      ├─────────────────┤     │   XDP Program   │     │                 │
│     Driver      │      │  XDP │ Driver   │     ├─────────────────┤     │    Userspace    │
├─────────────────┤      ├─────────────────┤     │     Driver      │     │     Driver      │
│       NIC       │      │       NIC       │     ├─────────────────┤     ├─────────────────┤
└─────────────────┘      └─────────────────┘     │       NIC       │     │       NIC       │
                                                 └─────────────────┘     └─────────────────┘
  ~1M pps/core           ~5-10M pps/core         ~20-40M pps/core        ~40-70M pps/core
```

### XDP (eXpress Data Path)

In-kernel fast path for packet processing. Processes packets at driver level before sk_buff allocation.

**Modes:**

| Mode | Location | Performance | Requirements |
|------|----------|-------------|--------------|
| Generic | After sk_buff | Lowest | Any NIC |
| Native | Driver hook | High | Driver support |
| Offload | NIC hardware | Highest | SmartNIC (Netronome) |

**Use cases:**
- DDoS mitigation (drop malicious packets early)
- Load balancing (redirect to backends)
- Packet filtering (firewall acceleration)
- Traffic sampling

**Quick start:**
```bash
# Attach XDP program
ip link set dev eth0 xdp obj xdp_prog.o sec xdp

# Check XDP status
ip link show eth0 | grep xdp

# Remove XDP program
ip link set dev eth0 xdp off

# Native vs generic mode
ip link set dev eth0 xdpgeneric obj xdp_prog.o  # Generic (slower)
ip link set dev eth0 xdpdrv obj xdp_prog.o      # Native (faster)
```

**Simple drop program:**
```c
SEC("xdp")
int xdp_drop_all(struct xdp_md *ctx) {
    return XDP_DROP;  // Drop all packets
}

// Return values:
// XDP_DROP    - Drop packet
// XDP_PASS    - Pass to normal stack
// XDP_TX      - Bounce back out same interface
// XDP_REDIRECT - Send to another interface/CPU
// XDP_ABORTED - Error, drop + trace
```

**Driver support check:**
```bash
# Check if NIC supports native XDP
ethtool -i eth0 | grep driver
# Common drivers with XDP: i40e, ice, ixgbe, mlx5, virtio_net, veth

# Verify XDP program loaded in native mode
bpftool net show dev eth0
```

**Performance:** 5-10M pps/core in native mode, limited by BPF program complexity.

### AF_XDP (Zero-Copy Sockets)

Kernel-supported fast path to userspace. Combines XDP filtering with zero-copy delivery to application.

**How it works:**
1. XDP program filters packets, redirects selected ones to AF_XDP socket
2. Packets placed in shared UMEM (user-mapped memory)
3. Application reads directly from ring buffers - no copies

**Architecture:**
```
                     ┌─────────────────┐
                     │   Application   │
                     │  (poll rings)   │
                     └────────┬────────┘
                              │ mmap
              ┌───────────────┴───────────────┐
              │            UMEM               │
              │  ┌──────┐ ┌──────┐ ┌──────┐  │
              │  │ Fill │ │Compl.│ │ RX   │  │
              │  │ Ring │ │ Ring │ │ Ring │  │
              │  └──────┘ └──────┘ └──────┘  │
              └───────────────┬───────────────┘
                              │
              ┌───────────────┴───────────────┐
              │         XDP Program           │
              │   (bpf_redirect_map to xsk)   │
              └───────────────┬───────────────┘
                              │
              ┌───────────────┴───────────────┐
              │            Driver             │
              └───────────────────────────────┘
```

**Setup:**
```bash
# Requirements
# - Kernel 4.18+ (basic), 5.4+ (recommended)
# - Driver with AF_XDP support
# - libxdp >= 1.2.2, libbpf

# Check AF_XDP support
grep -r XDP /proc/config.gz  # CONFIG_XDP_SOCKETS=y

# Load XDP program that redirects to AF_XDP socket
# (typically done in application using libxdp)
```

**Performance:**
- Copy mode: ~15-20M pps/core
- Zero-copy mode: ~30-40M pps/core
- With DPDK AF_XDP PMD: ~20-35M pps/core

**Key insight:** AF_XDP keeps Linux networking tools working (ip, tcpdump on other traffic) while providing near-DPDK speeds for targeted flows.

### DPDK (Data Plane Development Kit)

Full userspace networking. Kernel completely bypassed.

**When DPDK:**
- Need absolute maximum performance (line rate at 100Gbps+)
- Dedicated NIC for application (no sharing with kernel)
- Can handle all protocol processing in userspace
- Have team expertise for DPDK development

**When NOT DPDK:**
- Need standard Linux tools (ip, iptables, tcpdump)
- Mixed traffic (some kernel, some fast-path)
- Don't want to maintain separate network stack
- Development velocity matters more than last 10% performance

**Setup complexity:**
```bash
# 1. Bind NIC to DPDK-compatible driver
modprobe vfio-pci
dpdk-devbind.py --status
dpdk-devbind.py -b vfio-pci 0000:03:00.0

# 2. Allocate hugepages (required for DPDK)
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
mkdir -p /mnt/huge
mount -t hugetlbfs nodev /mnt/huge

# 3. NIC is now invisible to kernel
ip link show  # eth0 gone
```

**DPDK vs AF_XDP poll mode driver:**
```bash
# DPDK can use AF_XDP as backend (hybrid approach)
# Simpler setup, kernel still sees NIC
dpdk-testpmd -l 0-3 -n 4 \
    --vdev="net_af_xdp0,iface=eth0,start_queue=0,queue_count=1" \
    -- -i

# Pure DPDK (full bypass)
dpdk-testpmd -l 0-3 -n 4 -a 0000:03:00.0 -- -i
```

**Performance:** 40-70M pps/core, line rate at 100Gbps with proper NICs.

### Decision Tree

```
START: Is kernel networking your bottleneck?
│
├─> No: Packet rate < 1M pps/core
│   └─> Use standard stack with tuning (buffers, RSS, busy polling)
│
├─> Yes: Need > 1M pps/core
│   │
│   ├─> Only need filtering/redirect (no userspace processing)?
│   │   └─> Use XDP native mode
│   │       - DDoS mitigation, load balancing, firewall
│   │       - 5-10M pps/core
│   │
│   ├─> Need userspace processing but also standard tools?
│   │   └─> Use AF_XDP
│   │       - Selected flows fast-pathed, rest through kernel
│   │       - 20-40M pps/core
│   │
│   └─> Need maximum performance, dedicated NIC acceptable?
│       └─> Use DPDK
│           - 40-70M pps/core, line rate at 100Gbps
│           - No kernel networking on that NIC
```

### Performance Comparison

| Technology | Packets/sec (64B) | Latency | Kernel Tools | Setup Complexity |
|------------|-------------------|---------|--------------|------------------|
| Kernel stack | 0.5-1M pps/core | ~50us | Full | Low |
| XDP (generic) | 1-2M pps/core | ~30us | Full | Low |
| XDP (native) | 5-10M pps/core | ~5us | Full | Medium |
| AF_XDP (copy) | 15-20M pps/core | ~3us | Partial | Medium |
| AF_XDP (zero-copy) | 30-40M pps/core | ~2us | Partial | High |
| DPDK | 40-70M pps/core | <1us | None | High |

### Setup Complexity Comparison

| Technology | Kernel Version | Driver Changes | Hugepages | Application Changes |
|------------|---------------|----------------|-----------|---------------------|
| XDP | 4.8+ | None (generic) | No | BPF program |
| XDP native | 4.8+ | Need driver support | No | BPF program |
| AF_XDP | 4.18+ | Need driver support | No | Ring buffer polling |
| DPDK | Any | Bind to DPDK driver | Yes | Full rewrite |

### Hybrid Approaches

**XDP + normal stack:**
```c
// Filter at XDP level, pass interesting traffic up
SEC("xdp")
int xdp_filter(struct xdp_md *ctx) {
    // Parse headers...
    if (is_attack_traffic())
        return XDP_DROP;
    if (needs_fast_path())
        return bpf_redirect_map(&xsks_map, index, 0);
    return XDP_PASS;  // Regular stack for everything else
}
```

**AF_XDP with kernel fallback:**
```bash
# Bind AF_XDP to specific queue
# Other queues handled by kernel normally
ethtool -L eth0 combined 4
# Queue 0 -> AF_XDP
# Queues 1-3 -> kernel stack
```

**DPDK AF_XDP PMD:**
```bash
# DPDK application using kernel driver via AF_XDP
# NIC stays visible to kernel, DPDK gets fast path
# Best of both worlds, some performance cost
```

### Troubleshooting

```bash
# XDP program not loading?
ip link set dev eth0 xdpgeneric obj prog.o  # Try generic first
dmesg | grep -i xdp

# AF_XDP zero-copy not working?
# Check driver support
ethtool -i eth0
# Enable zero-copy explicitly in application

# DPDK not seeing NIC?
dpdk-devbind.py --status
# Check IOMMU enabled for vfio-pci
dmesg | grep -i iommu

# Performance lower than expected?
# Check mode (native vs generic for XDP)
bpftool net show
# Check zero-copy mode for AF_XDP
# Check hugepage allocation for DPDK
cat /proc/meminfo | grep Huge
```

### Production Recommendations

1. **Start with XDP** - Test if in-kernel processing is sufficient
2. **Graduate to AF_XDP** - When userspace processing needed
3. **DPDK as last resort** - Only for maximum performance requirements
4. **Always benchmark** - Real workloads, not synthetic tests
5. **Monitor bypass paths** - Lost visibility requires explicit instrumentation

**Key insight:** XDP/AF_XDP provide 80% of DPDK performance with 20% of the complexity. Most applications don't need full bypass.

## io_uring for Networking (5.19+)

io_uring provides significant performance improvements for network I/O, with major enhancements in 6.x kernels.

### Key Features by Kernel Version

| Version | Feature | Impact |
|---------|---------|--------|
| 5.19+ | Multishot accept | Single SQE handles multiple accepts |
| 6.0+ | Multishot receive | Single SQE handles multiple recvs |
| 6.1+ | Provided buffers v2 | More efficient buffer management |
| 6.4+ | Zero-copy send | Eliminates copy on transmit |
| 6.7+ | Registered ring fds | Reduce fd lookup overhead |

### Multishot Accept Pattern

```c
// Single SQE handles unlimited accepts
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_multishot_accept(sqe, listen_fd, NULL, NULL, 0);

// Each completion (CQE) delivers one connection
// CQE_F_MORE flag indicates more completions coming
```

### Provided Buffers (Zero-Copy Receive)

```c
// Register buffer ring with kernel
struct io_uring_buf_ring *br;
io_uring_register_buf_ring(&ring, &br_params, 0);

// Prep multishot recv with buffer selection
io_uring_prep_recv_multishot(sqe, fd, NULL, 0, 0);
sqe->flags |= IOSQE_BUFFER_SELECT;
sqe->buf_group = buf_group_id;
```

### Performance Comparison

| Approach | Connections/sec | Syscalls |
|----------|-----------------|----------|
| epoll + accept | 80,000 | High |
| io_uring accept | 120,000 | Medium |
| io_uring multishot | 200,000+ | Minimal |

**Key insight:** Multishot operations eliminate the syscall-per-event overhead that limits epoll. A single SQE submission can handle thousands of connections.

## Quick Reference

| Task | Setting |
|------|---------|
| Enable BBR | `net.ipv4.tcp_congestion_control=bbr` |
| Increase buffers | `net.core.rmem_max=16777216` |
| More connections | `net.core.somaxconn=65535` |
| Reduce TIME_WAIT | `net.ipv4.tcp_fin_timeout=15` |
| Enable TFO | `net.ipv4.tcp_fastopen=3` |
| No slow start idle | `net.ipv4.tcp_slow_start_after_idle=0` |
| Enable busy poll | `net.core.busy_poll=50` |
| Set RSS queues | `ethtool -L eth0 combined 8` |
| Set ring buffer | `ethtool -G eth0 rx 4096` |
| Enable TSO | `ethtool -K eth0 tso on` |

## See Also

- [eBPF & Tracing](06-ebpf-tracing.md) - XDP, netkit, BPF networking
- [Kernel Tuning](08-kernel-tuning.md) - NUMA, CPU isolation for network workloads
- [Network Analysis](03-network-analysis.md) - Diagnostic tools, traffic capture
- [Containers & K8s](07-containers-k8s.md) - Container networking, CNI tuning
