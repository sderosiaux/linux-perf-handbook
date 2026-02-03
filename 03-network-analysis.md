# Network Analysis

Traffic analysis, DNS tools, HTTP testing, and network debugging.

## Connection Monitoring

### ss (socket statistics)
```bash
ss -tuln                       # TCP/UDP listening numeric
ss -tunap                      # All with process
ss -s                          # Summary stats
ss -o state established        # Established only
ss -o state time-wait          # TIME_WAIT sockets
ss dst 10.0.0.1                # By destination
ss sport = :443                # By source port
ss dport = :80                 # By dest port
ss -ti                         # TCP internal info
ss -m                          # Memory usage per socket
ss -n4lt '(sport = :5000)'     # Filter expression
```

### netstat
```bash
netstat -tulpn                 # Listening + process
netstat -an                    # All numeric
netstat -rn                    # Routing table
netstat -i                     # Interface stats
netstat -s                     # Protocol stats
netstat -c                     # Continuous
```

### lsof (network)
```bash
lsof -i                        # All network
lsof -i :80                    # By port
lsof -i TCP                    # TCP only
lsof -i @192.168.1.1           # By host
lsof -i TCP:1-1024             # Port range
lsof -iTCP -sTCP:LISTEN        # Listening
lsof -n -P -i                  # Numeric (faster)
```

### conntrack
```bash
conntrack -L                   # List connections
conntrack -C                   # Count
conntrack -E                   # Event mode
conntrack -D -s 10.0.0.1       # Delete by source
cat /proc/sys/net/nf_conntrack_max  # Max limit
```

## Traffic Monitoring

### iftop
```bash
iftop                          # Default interface
iftop -i eth0                  # Specific interface
iftop -n                       # Numeric (no DNS)
iftop -P                       # Show ports
iftop -B                       # Bytes (not bits)
iftop -f "port 80"             # Filter
```

### nethogs
```bash
nethogs                        # Per-process bandwidth
nethogs eth0                   # Specific interface
nethogs -d 1                   # 1 second refresh
nethogs -t                     # Tracemode (text)
nethogs -v 2                   # Show upload/download
```

### bandwhich
```bash
bandwhich                      # Per-process + connection
bandwhich -i eth0              # Specific interface
bandwhich -n                   # No DNS resolution
bandwhich -r                   # Show raw mode
```

### bmon
```bash
bmon                           # Bandwidth monitor
bmon -p eth0                   # Specific interface
bmon -o ascii                  # ASCII output
```

### nload
```bash
nload                          # Real-time graphs
nload eth0                     # Specific interface
nload -u M                     # MBit/s
nload -a 300                   # 5 min average
```

### ifstat
```bash
ifstat                         # Interface stats
ifstat -t                      # Add timestamp
ifstat 1                       # 1 second interval
ifstat -i eth0                 # Specific interface
```

## Packet Analysis

### tcpdump
```bash
tcpdump -i eth0                # Capture on interface
tcpdump -i any                 # All interfaces
tcpdump -n                     # Numeric
tcpdump -A                     # ASCII (HTTP)
tcpdump -X                     # Hex + ASCII
tcpdump -w capture.pcap        # Write to file
tcpdump -r capture.pcap        # Read file
tcpdump -c 100                 # Capture 100 packets
tcpdump host 10.0.0.1          # Filter by host
tcpdump port 80                # Filter by port
tcpdump 'tcp[tcpflags] & tcp-syn != 0'  # SYN packets
tcpdump -s0 -w - | wireshark -k -i -    # Pipe to wireshark
```

### termshark (TUI wireshark)
```bash
termshark -i eth0              # Live capture
termshark -r capture.pcap      # Read file
termshark -Y "http"            # Display filter
termshark -f "port 80"         # Capture filter
```

### tshark
```bash
tshark -i eth0                 # Capture
tshark -r file.pcap            # Read file
tshark -Y "http.request"       # Display filter
tshark -f "port 443"           # Capture filter
tshark -T fields -e ip.src -e ip.dst  # Extract fields
tshark -q -z io,stat,1         # I/O stats
tshark -q -z conv,tcp          # Conversations
```

### ngrep
```bash
ngrep -d any "pattern"         # Search all interfaces
ngrep -q -W byline port 80     # HTTP friendly
ngrep '' multicast             # Monitor multicast
ngrep -d any "GET|POST" port 80  # HTTP methods
```

### tcpflow
```bash
tcpflow -i eth0                # Reconstruct streams
tcpflow -c                     # Output to console
tcpflow -r capture.pcap        # From pcap
```

## DNS Tools

### dig
```bash
dig example.com                # A record
dig example.com MX             # MX records
dig +short example.com         # Short output
dig @8.8.8.8 example.com       # Specific DNS
dig +trace example.com         # Trace resolution
dig +nocmd example.com any +multiline +noall +answer
dig -x 8.8.8.8                 # Reverse lookup
```

### host
```bash
host example.com               # Basic lookup
host -t MX example.com         # MX records
host -t ANY example.com        # All records
host -a example.com            # Verbose
```

### nslookup
```bash
nslookup example.com           # Basic lookup
nslookup -type=mx example.com  # MX records
nslookup example.com 8.8.8.8   # Specific DNS
```

### whois
```bash
whois example.com              # Domain info
whois 8.8.8.8                  # IP info
```

## Route Analysis

### mtr (better traceroute)
```bash
mtr example.com                # Interactive
mtr -r example.com             # Report mode
mtr -rw -c 10 example.com      # Wide report, 10 cycles
mtr -T example.com             # TCP mode
mtr -u example.com             # UDP mode
mtr --json example.com         # JSON output
mtr -P 443 example.com         # Specific port
```

### traceroute
```bash
traceroute example.com         # Basic trace
traceroute -n example.com      # Numeric
traceroute -T example.com      # TCP SYN
traceroute -I example.com      # ICMP
traceroute -p 443 example.com  # Specific port
```

### tracepath
```bash
tracepath example.com          # Path + MTU discovery
tracepath -n example.com       # Numeric
```

### ping / gping
```bash
ping -c 4 example.com          # 4 pings
ping -i 0.2 example.com        # 200ms interval
ping -s 1472 example.com       # Packet size
ping -f example.com            # Flood (root)
ping -D example.com            # Timestamps
gping example.com google.com   # Multiple with graph
```

## Port Scanning

### nmap
```bash
nmap host                      # Top 1000 ports
nmap -p- host                  # All 65535 ports
nmap -sV host                  # Version detection
nmap -O host                   # OS detection
nmap -A host                   # Aggressive scan
nmap -sU host                  # UDP scan
nmap -sS host                  # SYN scan (stealth)
nmap -Pn host                  # Skip host discovery
nmap -oN output.txt host       # Normal output
nmap -oX output.xml host       # XML output
nmap --script vuln host        # Vulnerability scripts
nmap -sn 192.168.1.0/24        # Ping sweep
```

### masscan (fast scanner)
```bash
masscan -p80,443 10.0.0.0/8    # Specific ports
masscan -p1-65535 host         # All ports
masscan --rate 10000 ...       # Packets/sec
masscan -oJ output.json ...    # JSON output
```

### rustscan (faster nmap wrapper)
```bash
rustscan -a host               # Scan + pipe to nmap
rustscan -a host -- -sV        # With nmap options
rustscan -a host -r 1-65535    # Port range
```

## HTTP Testing

### curl
```bash
curl -I url                    # Headers only
curl -v url                    # Verbose
curl -o file url               # Download
curl -X POST -d "data" url     # POST
curl -H "Header: value" url    # Custom header
curl -u user:pass url          # Basic auth
curl -k url                    # Skip TLS verify
curl -w "%{time_total}\n" url  # Timing
curl --compressed url          # Accept encoding
curl -L url                    # Follow redirects
curl -x proxy:port url         # Via proxy
```

### wget
```bash
wget url                       # Download
wget -c url                    # Continue partial
wget -r -l 2 url               # Recursive, 2 levels
wget -m url                    # Mirror site
wget --spider url              # Check link
wget -q -O - url               # Quiet, stdout
```

### hey (HTTP load generator)
```bash
hey url                        # Default 200 requests
hey -n 1000 url                # 1000 requests
hey -c 50 url                  # 50 concurrent
hey -q 100 url                 # Rate limit 100 QPS
hey -m POST -d "body" url      # POST with body
hey -H "Auth: token" url       # Custom header
```

### wrk (HTTP benchmark)
```bash
wrk url                        # Default benchmark
wrk -t 4 -c 100 url            # 4 threads, 100 conns
wrk -d 30s url                 # 30 second duration
wrk -s script.lua url          # Lua script
wrk -H "Header: value" url     # Custom header
```

### vegeta (HTTP attack)
```bash
echo "GET http://url" | vegeta attack -duration=5s | vegeta report
echo "GET http://url" | vegeta attack -rate=100 | vegeta report
vegeta encode | vegeta attack | vegeta encode > results.gob
vegeta report -type=json < results.gob
vegeta plot < results.gob > plot.html
```

### k6 (load testing)
```bash
k6 run script.js               # Run script
k6 run --vus 10 --duration 30s script.js
k6 run --out json=out.json script.js
```

### ab (Apache Bench)
```bash
ab -n 1000 -c 10 url           # 1000 requests, 10 concurrent
ab -t 60 -c 10 url             # 60 seconds, 10 concurrent
ab -k url                      # Keep-alive
ab -H "Header: val" url        # Custom header
```

## SSL/TLS Testing

### openssl
```bash
openssl s_client -connect host:443
openssl s_client -connect host:443 -servername host  # SNI
openssl s_client -connect host:443 < /dev/null | openssl x509 -text
openssl s_client -connect host:443 -showcerts
```

### testssl.sh
```bash
testssl.sh host                # Full test
testssl.sh --fast host         # Quick test
testssl.sh -e host             # Cipher test
testssl.sh -p host             # Protocol test
testssl.sh -U host             # Vulnerability test
```

## Low-Level Tools

### hping3
```bash
hping3 -S host -p 80           # SYN to port 80
hping3 -F host                 # FIN
hping3 --flood host            # Flood (caution!)
hping3 -1 host                 # ICMP
hping3 -2 host -p 53           # UDP
hping3 --traceroute -S host    # Traceroute with SYN
```

### socat
```bash
socat TCP-LISTEN:8080,fork TCP:remote:80  # Port forward
socat - TCP:host:port          # Connect
socat TCP-LISTEN:8080,fork EXEC:/bin/bash  # Shell (unsafe!)
socat OPENSSL-LISTEN:443,cert=cert.pem,key=key.pem,fork TCP:backend:80
```

### nc (netcat)
```bash
nc -l 8080                     # Listen
nc host 80                     # Connect
nc -z host 1-1000              # Port scan
nc -u host 53                  # UDP
nc -l 8080 < file              # Serve file
nc host 8080 > file            # Receive file
```

## Interface Configuration

### ip
```bash
ip addr                        # Show addresses
ip link                        # Show links
ip route                       # Routing table
ip -s link                     # Statistics
ip neigh                       # ARP table
ip addr add 10.0.0.1/24 dev eth0  # Add address
ip link set eth0 up/down       # Up/down interface
ip route add default via 10.0.0.1  # Add route
```

### ethtool
```bash
ethtool eth0                   # Settings
ethtool -i eth0                # Driver info
ethtool -S eth0                # Statistics
ethtool -k eth0                # Offload features
ethtool -g eth0                # Ring buffer
ethtool -s eth0 speed 1000 duplex full  # Set speed
ethtool -K eth0 tso off        # Disable TSO
```

### lnstat
```bash
lnstat                         # Kernel network stats
lnstat -j                      # JSON output
lnstat -k arp_cache            # ARP cache stats
```

## Quick Reference

| Task | Command |
|------|---------|
| Listening ports | `ss -tulpn` |
| Connections | `ss -tunap` |
| Traffic by process | `nethogs` |
| Packet capture | `tcpdump -i eth0 -w out.pcap` |
| DNS lookup | `dig +short example.com` |
| Route analysis | `mtr -r example.com` |
| Port scan | `nmap -sV host` |
| HTTP benchmark | `hey -n 1000 -c 50 url` |
| SSL test | `testssl.sh host` |
| Interface stats | `ip -s link` |

## HTTP/2 Performance Analysis

HTTP/2 introduces multiplexing, header compression, and server push over a single TCP connection.

### Multiplexing and Streams

HTTP/2 multiplexes requests as streams over one TCP connection. Each stream has independent flow control.

```bash
# Check HTTP/2 support
curl -I --http2 -s https://example.com | grep -i "^http/"

# Debug HTTP/2 connection details
curl -v --http2 https://example.com 2>&1 | grep -E "(ALPN|h2|stream)"

# Show HTTP/2 frame details with nghttp
nghttp -nv https://example.com
```

**Stream prioritization:**
```bash
# nghttp with priority tree visualization
nghttp -v --stat https://example.com

# Output shows frame types: HEADERS, DATA, SETTINGS, WINDOW_UPDATE
# Weight ranges 1-256, dependencies form priority tree
```

**Head-of-line blocking:**

HTTP/2 solves HTTP/1.1 HOL blocking at application layer but introduces TCP-level HOL blocking. A single lost packet blocks all streams until retransmitted.

```bash
# Measure TCP retransmissions affecting HTTP/2
ss -ti dst example.com | grep retrans

# tcpdump for packet loss analysis
tcpdump -i eth0 -n "tcp port 443" -w h2.pcap
# Analyze with: tshark -r h2.pcap -q -z io,stat,1,tcp.analysis.retransmission
```

### Flow Control Tuning

HTTP/2 has connection-level and stream-level flow control windows (default 65535 bytes).

```bash
# Check current window sizes with nghttp
nghttp -nv https://example.com 2>&1 | grep WINDOW_UPDATE

# Server-side (nginx) tuning
# http2_recv_buffer_size 256k;       # Initial recv buffer
# http2_max_concurrent_streams 128;  # Default 128
# http2_max_field_size 4k;           # Max header field size
```

```bash
# Monitor HTTP/2 with tshark
tshark -i eth0 -Y "http2" -T fields -e http2.streamid -e http2.type -e http2.length

# Frame types: 0=DATA, 1=HEADERS, 3=RST_STREAM, 4=SETTINGS, 8=WINDOW_UPDATE
```

### HTTP/2 Benchmarking

#### h2load

The primary HTTP/2 benchmark tool from nghttp2 project.

```bash
# Basic benchmark
h2load -n 10000 -c 100 -m 10 https://example.com
# -n: total requests
# -c: concurrent clients (TCP connections)
# -m: max concurrent streams per connection

# Sustained load with timing
h2load -n 100000 -c 50 -m 32 --duration=60 https://example.com

# With request body
h2load -n 1000 -c 10 -m 10 -d request_body.txt https://example.com/api

# POST with headers
h2load -n 1000 -c 10 -H "Content-Type: application/json" \
       -d '{"key":"value"}' https://example.com/api

# Output columns:
# req/s: requests per second
# time for req: min/max/mean/sd latency
# time for connect: TCP+TLS handshake time
```

**Interpreting h2load output:**
```
finished in 5.23s, 19120.46 req/s, 2.34MB/s
requests: 100000 total, 100000 started, 100000 done, 100000 succeeded
status codes: 100000 2xx, 0 3xx, 0 4xx, 0 5xx
traffic: 12.24MB total, 1.91MB headers, 9.54MB data
                     min         max         mean        sd
time for request:    1.23ms     89.45ms     5.21ms      3.12ms
time for connect:   45.12ms    156.78ms    78.34ms     12.45ms
time for 1st byte:  46.89ms    198.23ms    85.67ms     15.23ms
```

#### curl HTTP/2 debugging

```bash
# Force HTTP/2
curl --http2 -v https://example.com

# HTTP/2 prior knowledge (h2c - cleartext)
curl --http2-prior-knowledge http://localhost:8080

# Timing with HTTP/2
curl --http2 -w "\nDNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTLS: %{time_appconnect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" -o /dev/null -s https://example.com

# Trace HTTP/2 frames (requires curl 7.61+)
curl --http2 -v --trace-ascii /dev/stdout https://example.com 2>&1 | head -100
```

#### nghttp inspection

```bash
# Verbose request with timing
nghttp -nv --stat https://example.com

# Multiple requests (tests multiplexing)
nghttp -nv https://example.com/path1 https://example.com/path2 https://example.com/path3

# Show dependency tree
nghttp -v --dep-idle https://example.com

# Upgrade from HTTP/1.1
nghttp -nvu http://example.com
```

## HTTP/3 and QUIC

HTTP/3 uses QUIC (UDP-based) eliminating TCP HOL blocking. Each stream has independent loss recovery.

### QUIC vs TCP

| Aspect | TCP | QUIC |
|--------|-----|------|
| HOL blocking | Connection-level | Stream-level only |
| Handshake | TCP + TLS (2-3 RTT) | Combined (1 RTT, 0-RTT resume) |
| Connection migration | Breaks on IP change | Survives via Connection ID |
| Congestion control | Kernel-level | User-space, pluggable |

### 0-RTT Performance

QUIC supports 0-RTT resumption for repeated connections. Reduces latency by ~1 RTT for returning clients.

```bash
# Test 0-RTT with curl (requires HTTP/3 build)
curl --http3 -v https://example.com 2>&1 | grep -i "0-rtt\|early"

# Check server 0-RTT support
# Look for "early_data" in TLS extension during handshake
```

**Security consideration:** 0-RTT data can be replayed. Servers should only allow idempotent operations in early data.

### Connection Migration

QUIC connections survive network changes via Connection ID rather than IP:port tuple.

```bash
# Simulate migration impact with mtr during request
mtr -r -c 10 example.com &
curl --http3 -o /dev/null -w "Total: %{time_total}s\n" https://example.com
```

### QUIC Congestion Control

QUIC supports pluggable congestion control in user space:
- **Cubic:** Default, similar to TCP Cubic
- **BBR:** Google's bandwidth-based algorithm
- **Reno:** Conservative, fair sharing

```bash
# Check server congestion control (server-side config)
# nginx: quic_congestion_control_algorithm bbr;
# caddy: experimental_http3 on; (uses quic-go defaults)
```

### HTTP/3 Testing

#### curl --http3

Requires curl built with HTTP/3 support (quiche, ngtcp2, or msh3).

```bash
# Check curl HTTP/3 support
curl --version | grep -i "http3\|quic"

# Force HTTP/3
curl --http3 -v https://example.com

# HTTP/3 only (fail if unavailable)
curl --http3-only https://example.com

# Timing comparison
echo "HTTP/2:"
curl --http2 -o /dev/null -s -w "Total: %{time_total}s TTFB: %{time_starttransfer}s\n" https://cloudflare.com
echo "HTTP/3:"
curl --http3 -o /dev/null -s -w "Total: %{time_total}s TTFB: %{time_starttransfer}s\n" https://cloudflare.com
```

#### quiche tools

Cloudflare's QUIC implementation includes testing tools.

```bash
# quiche-client basic request
quiche-client https://example.com

# With verbose QUIC debugging
RUST_LOG=debug quiche-client https://example.com

# h3i - HTTP/3 inspection tool
h3i https://example.com
```

#### Measuring QUIC Benefits

```bash
# Compare protocols under packet loss
# Requires tc for traffic shaping

# Add 1% packet loss
sudo tc qdisc add dev eth0 root netem loss 1%

# Test both protocols
for proto in "--http2" "--http3"; do
    echo "Testing $proto with 1% loss:"
    for i in {1..10}; do
        curl $proto -o /dev/null -s -w "%{time_total}\n" https://cloudflare.com
    done | awk '{sum+=$1} END {print "Avg:", sum/NR "s"}'
done

# Remove traffic shaping
sudo tc qdisc del dev eth0 root
```

**Expected results:** HTTP/3 shows significantly better performance under packet loss due to stream-level loss recovery. Difference is most pronounced at >1% loss rates.

### QUIC Debugging

```bash
# Wireshark QUIC decryption (requires SSLKEYLOGFILE)
SSLKEYLOGFILE=/tmp/keys.log curl --http3 https://example.com
# Import /tmp/keys.log in Wireshark: Edit > Preferences > TLS > (Pre)-Master-Secret

# qlog format for QUIC tracing (if supported by client/server)
# Output compatible with qvis.quictools.info visualization

# Check UDP port 443 traffic
ss -ulnp | grep 443
tcpdump -i eth0 -n "udp port 443" -c 100
```

## TLS Performance

TLS handshake overhead significantly impacts connection latency. TLS 1.3 reduces round trips.

### TLS 1.3 Handshake

TLS 1.3 completes in 1-RTT (vs 2-RTT for TLS 1.2). Key exchange and encryption parameters in single flight.

```bash
# Check TLS version negotiated
curl -v https://example.com 2>&1 | grep "SSL connection using"
openssl s_client -connect example.com:443 2>/dev/null | grep "Protocol"

# Force TLS 1.3
curl --tlsv1.3 -v https://example.com 2>&1 | grep -E "TLSv1.3|SSL"
openssl s_client -connect example.com:443 -tls1_3

# Handshake timing breakdown
curl -w "DNS: %{time_namelookup}s\nTCP: %{time_connect}s\nTLS: %{time_appconnect}s\nTotal: %{time_total}s\n" \
     -o /dev/null -s https://example.com
```

### Session Resumption

Session tickets allow 0-RTT resumption, skipping full handshake.

```bash
# Test session resumption with openssl
openssl s_client -connect example.com:443 -sess_out /tmp/sess.pem </dev/null
openssl s_client -connect example.com:443 -sess_in /tmp/sess.pem </dev/null 2>&1 | grep "Reused"

# Check if server supports session tickets
openssl s_client -connect example.com:443 2>/dev/null | grep -i "ticket"
```

**TLS 1.3 0-RTT (Early Data):**
```bash
# Test early data support
openssl s_client -connect example.com:443 -tls1_3 -early_data request.txt

# Server must explicitly enable:
# nginx: ssl_early_data on;
# Risk: Early data is replayable - only use for idempotent requests
```

### Certificate Chain Impact

Large certificate chains add latency. Each certificate ~1-2KB.

```bash
# Check certificate chain size
echo | openssl s_client -connect example.com:443 -showcerts 2>/dev/null | \
    awk '/BEGIN CERT/,/END CERT/' | wc -c

# Count certificates in chain
echo | openssl s_client -connect example.com:443 2>/dev/null | \
    grep -c "Certificate chain"

# Detailed chain analysis
openssl s_client -connect example.com:443 -showcerts 2>/dev/null | \
    openssl x509 -text -noout | grep -E "Subject:|Issuer:|Not After"
```

**Optimization strategies:**
- Remove unnecessary intermediate certificates
- Use ECDSA certificates (smaller than RSA)
- Enable OCSP stapling (avoids extra round trip)

```bash
# Check OCSP stapling
echo | openssl s_client -connect example.com:443 -status 2>/dev/null | \
    grep -A 5 "OCSP Response"

# Verify certificate with OCSP
openssl ocsp -issuer chain.pem -cert server.pem -url http://ocsp.example.com
```

### TLS Cipher Performance

ECDHE+AESGCM offers best performance/security balance on modern CPUs with AES-NI.

```bash
# List supported ciphers
openssl ciphers -v 'HIGH:!aNULL' | head -20

# Test specific cipher
openssl s_client -connect example.com:443 -cipher ECDHE-RSA-AES128-GCM-SHA256

# Benchmark cipher performance (local)
openssl speed -evp aes-128-gcm
openssl speed -evp chacha20-poly1305

# Compare ECDSA vs RSA signing
openssl speed ecdsap256 rsa2048
```

**Cipher recommendations for performance:**
| Priority | Cipher Suite | Notes |
|----------|--------------|-------|
| 1 | TLS_AES_128_GCM_SHA256 | TLS 1.3, AES-NI accelerated |
| 2 | TLS_AES_256_GCM_SHA384 | TLS 1.3, stronger |
| 3 | TLS_CHACHA20_POLY1305_SHA256 | TLS 1.3, faster without AES-NI |
| 4 | ECDHE-ECDSA-AES128-GCM-SHA256 | TLS 1.2, requires ECDSA cert |

### TLS Testing Tools

```bash
# Full TLS analysis
testssl.sh --fast example.com

# Protocol and cipher check
testssl.sh -p -e example.com

# Specific vulnerability checks
testssl.sh -U example.com

# sslyze for detailed analysis
sslyze example.com --regular

# SSL Labs API (for comprehensive public test)
curl "https://api.ssllabs.com/api/v3/analyze?host=example.com&startNew=on"
```

## Protocol Comparison Quick Reference

| Metric | HTTP/1.1 | HTTP/2 | HTTP/3 |
|--------|----------|--------|--------|
| Connections | Multiple | Single multiplexed | Single multiplexed |
| HOL blocking | Per-connection | TCP-level | None (stream-level) |
| Header compression | None | HPACK | QPACK |
| Handshake RTT | 2-3 (TCP+TLS) | 2-3 (TCP+TLS) | 1 (0 with resumption) |
| Transport | TCP | TCP | QUIC (UDP) |
| Connection migration | No | No | Yes |

**When to use each:**
- **HTTP/1.1:** Legacy compatibility, very simple requests
- **HTTP/2:** Default for modern web, good multiplexing
- **HTTP/3:** High latency networks, mobile users, lossy connections
