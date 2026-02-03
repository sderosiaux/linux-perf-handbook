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
