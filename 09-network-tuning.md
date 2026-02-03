# Network Tuning

TCP optimization, congestion control, and NUMA-aware networking.

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

### iperf3 (bandwidth testing)

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
