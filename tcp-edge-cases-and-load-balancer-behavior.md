# TCP Edge Cases & Load Balancer Behavior

Production-ready guide for diagnosing and resolving TCP and load balancer issues affecting latency and reliability. Focused on LLM-actionable troubleshooting.

## 1. TCP SYN Retry Timeout

### The Default 5-Second Problem

**Default behavior:** Initial SYN timeout is 1 second, with exponential backoff doubling on each retry.

```bash
# Check current SYN retry count
sysctl net.ipv4.tcp_syn_retries
# Default: 6 retries = ~127 seconds total (pre-3.7 kernel: 5 retries = ~180s)

# Retry timeline with default settings
# Retry 0: 1s
# Retry 1: 3s (waited 2s)
# Retry 2: 7s (waited 4s)
# Retry 3: 15s (waited 8s)
# Retry 4: 31s (waited 16s)
# Retry 5: 63s (waited 32s)
# Retry 6: 127s (waited 64s)
```

**Production impact:** A single dropped SYN packet = 1-3 second delay before first retry. At scale, even 0.1% packet loss creates noticeable tail latency.

### Detection Commands

```bash
# Monitor SYN retransmissions in real-time
ss -tan state syn-sent | wc -l

# Count SYN retransmits
nstat -az | grep TcpRetransSegs

# Track failed connection attempts
netstat -s | grep "failed connection attempts"

# Detailed per-connection view
ss -ti dst <target-ip> | grep retrans

# Capture SYN packets with tcpdump
tcpdump -i any 'tcp[tcpflags] & tcp-syn != 0 and tcp[tcpflags] & tcp-ack == 0' -nn

# eBPF-based SYN tracking
bpftrace -e '
tracepoint:tcp:tcp_retransmit_skb /args->skb->sk->__sk_common.skc_state == 2/ {
  printf("SYN retransmit to %s\n", ntop(args->daddr));
  @syn_retrans[ntop(args->daddr)] = count();
}'
```

### Configuration Best Practices

```bash
# Reduce SYN retries for faster failure detection
# Trade-off: Less resilience to transient network issues
sysctl -w net.ipv4.tcp_syn_retries=2  # ~7 seconds total (1s + 3s + 7s)

# Per-socket override (application-level)
# In C: setsockopt(fd, IPPROTO_TCP, TCP_SYNCNT, &retries, sizeof(retries));
```

**When to reduce:**
- Internal microservices with fast failure requirements
- When circuit breakers handle retries at application layer
- Low-latency requirements where 127s is unacceptable

**When NOT to reduce:**
- WAN connections with variable latency
- Mobile clients with unreliable networks
- When you lack application-level retry logic

### Connection Pool Warm Connection Strategies

**Problem:** Cold connections pay full SYN + TLS handshake cost. With default 6 retries, a failed SYN blocks pool acquisition for seconds.

#### Strategy 1: Pre-Warming with Health Checks

```bash
# HAProxy example - health checks keep backend connections warm
backend app_servers
  option httpchk GET /health
  http-check expect status 200
  server app1 10.0.1.10:8080 check inter 2000 rise 2 fall 3

# Benefit: Dead backends detected before request arrives
# Cost: Extra load from health checks
```

#### Strategy 2: Connection TTL Management

```go
// Go example: Limit connection lifetime to prevent stale connections
transport := &http.Transport{
    MaxIdleConns:        100,
    MaxIdleConnsPerHost: 10,
    IdleConnTimeout:     30 * time.Second,  // Close idle after 30s
    MaxConnsPerHost:     100,
}

// Key insight: Balance between:
// - Too short TTL = connection churn, handshake overhead
// - Too long TTL = stale connections, blocked by firewalls/LBs
```

#### Strategy 3: Fail-Fast with Reduced SYN Retries

```bash
# Application-level configuration
# Java example (Netty)
bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000);

# Python example (requests)
requests.get(url, timeout=(3.0, 10.0))  # (connect, read)

# Key: Set connect timeout LOWER than tcp_syn_retries allows
# Forces application-level retry/circuit breaking
```

#### Strategy 4: TCP Fast Open (with caveats - see section 5)

```bash
# Enable TFO for warm connections
sysctl -w net.ipv4.tcp_fastopen=3  # Client + Server

# Application must use sendto() or MSG_FASTOPEN flag
# Saves 1 RTT on subsequent connections to same server
```

### Decision Tree: SYN Retry Configuration

```
Is this an internal microservice mesh?
├─> YES: Do you have application-level retries?
│   ├─> YES: Reduce to 2-3 retries (fail fast, circuit breaker handles)
│   └─> NO:  Keep default 6 (avoid cascading failures)
└─> NO: Internet-facing or WAN?
    └─> Keep default 6 (unreliable networks need resilience)

Are connection pools hitting timeout?
├─> Check pool sizing: L = λ × W (Little's Law)
├─> Check warm-up: Pre-establish connections during startup
└─> Check TTL: Balance freshness vs handshake cost
```

---

## 2. Load Balancer Buffering Issues

### AWS ELB Buffering Behavior

**Problem:** ELBs buffer megabytes of data. Client uploads complete, ELB still streaming to backend. ELB closes connection before backend processes all data.

```
Timeline:
T+0ms:   Client starts upload (10 MB file)
T+500ms: Client finishes upload to ELB
T+501ms: Client sees HTTP 200 OK from ELB
T+600ms: ELB still streaming to backend (slow network)
T+700ms: Backend closes connection (idle timeout)
T+701ms: ELB realizes backend is gone
T+702ms: Client already disconnected - never sees error
```

**Result:** Client thinks success, backend never processed request.

### Detection Commands

```bash
# Check ELB idle timeout (default 60s)
aws elbv2 describe-load-balancer-attributes \
  --load-balancer-arn arn:aws:... \
  --query 'Attributes[?Key==`idle_timeout.timeout_seconds`].Value'

# Monitor backend connection errors (indicates buffering issues)
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name TargetConnectionErrorCount \
  --dimensions Name=LoadBalancer,Value=app/my-lb/... \
  --start-time 2026-02-03T00:00:00Z \
  --end-time 2026-02-03T23:59:59Z \
  --period 300 \
  --statistics Sum

# Check for request ID reuse (see next section)
grep "X-Amzn-Trace-Id" /var/log/application.log | sort | uniq -d

# Monitor connection timing from backend perspective
ss -ti | grep -E "timer:|retrans:"

# Check for incomplete requests reaching backend
tcpdump -i any port 8080 -A | grep -E "Content-Length:|^$" -A 5
```

### Configuration Examples

#### AWS ELB Timeout Configuration

```bash
# Increase idle timeout to match backend processing time
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn arn:aws:... \
  --attributes Key=idle_timeout.timeout_seconds,Value=300

# Rule: idle_timeout > backend_keep_alive + processing_time + buffer
# Example: 300s idle > 120s keep-alive + 60s processing + 120s buffer
```

#### Backend Keep-Alive Must Be SHORTER Than ELB

```bash
# WRONG: Backend keep-alive > ELB idle timeout
# nginx.conf (BAD)
keepalive_timeout 120s;  # ELB idle = 60s → 502 errors

# CORRECT: Backend keep-alive < ELB idle timeout
# nginx.conf (GOOD)
keepalive_timeout 50s;   # ELB idle = 60s → Backend closes first
```

**Key insight:** The backend MUST close idle connections before the load balancer does. Otherwise, ELB sends new request to dead backend socket = 502 error.

#### Application-Level Solutions

```python
# Python Flask example: Stream responses to prevent buffering
from flask import Response, stream_with_context

@app.route('/upload', methods=['POST'])
def upload():
    def generate():
        # Process upload incrementally
        for chunk in request.stream:
            process_chunk(chunk)
            yield f"Processed {len(chunk)} bytes\n"

    # ELB sees streaming response, doesn't buffer entire upload
    return Response(stream_with_context(generate()), mimetype='text/plain')
```

```go
// Go example: Flush writer to prevent buffering
func uploadHandler(w http.ResponseWriter, r *http.Request) {
    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "Streaming unsupported", http.StatusInternalServerError)
        return
    }

    // Process body incrementally
    buffer := make([]byte, 32*1024)
    for {
        n, err := r.Body.Read(buffer)
        if n > 0 {
            processChunk(buffer[:n])
            fmt.Fprintf(w, "Processed %d bytes\n", n)
            flusher.Flush()  // Prevent ELB buffering
        }
        if err == io.EOF {
            break
        }
    }
}
```

### Request ID Reuse Bug Pattern

**Problem:** ELB reuses request IDs when connections are recycled. Backend sees same X-Amzn-Trace-Id for different requests.

**Real-world case:** Cassandra Python driver auth bypass (CVE-like behavior)
1. Request A arrives with trace-id-123
2. Backend authenticates, stores session for trace-id-123
3. ELB recycles connection, sends Request B with SAME trace-id-123
4. Backend sees cached session, bypasses auth
5. Request B executes with Request A's privileges

**Detection:**

```bash
# Check for duplicate request IDs in logs
awk -F'X-Amzn-Trace-Id: ' '{print $2}' access.log | sort | uniq -d

# Alert on duplicate IDs within time window
awk -F'X-Amzn-Trace-Id: ' '
{
  id = $2;
  timestamp = $1;
  if (id in seen && (timestamp - seen[id]) < 60) {
    print "DUPLICATE ID:", id, "at", timestamp, "(previous:", seen[id], ")";
  }
  seen[id] = timestamp;
}' access.log
```

**Mitigation:**

```python
# NEVER use request ID as sole session identifier
# BAD:
session = cache.get(request.headers['X-Amzn-Trace-Id'])

# GOOD: Combine with client-unique data
session_key = f"{request.headers['X-Amzn-Trace-Id']}:{request.remote_addr}:{request.headers['User-Agent']}"
session = cache.get(session_key)

# BEST: Use cryptographically random session tokens
session_token = secrets.token_urlsafe(32)
```

---

## 3. Server vs Client Timeout Anti-Patterns

### The Golden Rule

**ALWAYS set server-side timeout SHORTER than client-side timeout + network latency.**

```
Client Timeout:  30s
Network RTT:     +2s (P99)
Server Timeout:  25s (MUST be less than 32s)
```

**Why:** If server times out AFTER client, client may retry with new request ID while server still processing old request.

### Common Anti-Patterns

#### Anti-Pattern 1: Server Timeout > Client Timeout

```python
# BAD: Server timeout exceeds client
# client.py
response = requests.get(url, timeout=30)

# server.py (Django)
TIMEOUT = 60  # WRONG! Server outlives client

# Result: Client retries, server still processing
# Load doubles, duplicate work, inconsistent state
```

#### Anti-Pattern 2: No Timeout Propagation

```java
// BAD: Downstream call uses fixed timeout
// Frontend timeout: 5s
// Backend makes DB call with 30s timeout → frontend already gave up

// GOOD: Propagate remaining time budget
public Response handleRequest(long deadlineMs) {
    long remainingMs = deadlineMs - System.currentTimeMillis();
    if (remainingMs < 100) {
        return Response.timeout();
    }
    // Pass remaining budget to DB
    return db.query(query, Math.max(100, remainingMs - 500));
}
```

#### Anti-Pattern 3: Ignoring Network Latency

```bash
# Calculate realistic timeout
# Client → LB: 50ms P99
# LB → Backend: 50ms P99
# Backend processing: 1000ms P95
# Backend → LB: 50ms P99
# LB → Client: 50ms P99

# Total P99: 50+50+1000+50+50 = 1200ms
# Add buffer: 1200ms * 1.5 = 1800ms
# Set client timeout: 2000ms
# Set server timeout: 1500ms (< 2000ms - 200ms network)
```

### Detection Commands

```bash
# Check for timeout misconfigurations
# 1. Identify requests timing out at client but succeeding at server

# Client-side (check logs for timeouts)
grep -E "timeout|timed out|connection reset" client.log | wc -l

# Server-side (check for requests that completed after timeout)
awk '
{
  if ($7 ~ /timeout/) { timeout[$10] = $1; }  # Track client timeouts by request ID
  if ($7 ~ /200|201|204/) {
    if ($10 in timeout && $1 - timeout[$10] > 30) {
      print "Late success:", $0;
    }
  }
}' server.log

# 2. Monitor duplicate request IDs (indicates retry after timeout)
grep "request_id" server.log | awk '{print $NF}' | sort | uniq -d

# 3. Check timeout settings
# Node.js
grep -r "timeout" --include="*.js" | grep -E "http|request"

# Python
grep -r "timeout=" --include="*.py"

# Java
grep -r "setTimeout" --include="*.java"
```

### Configuration Best Practices

#### Multi-Tier Timeout Strategy

```yaml
# Example: 3-tier architecture (Frontend → Service → Database)

frontend:
  client_timeout: 5000ms
  request_timeout: 4500ms  # Leave 500ms for network

service:
  server_timeout: 4000ms   # Fail before frontend times out
  db_timeout: 3000ms       # Fail with time to return error

database:
  query_timeout: 2500ms    # Fail with time for service to handle
  connection_timeout: 500ms

# Rule: Each tier = previous - (processing_time + 2 × network_latency)
```

#### Context-Based Timeout Propagation (Go)

```go
// Propagate deadlines through request chain
func handleRequest(w http.ResponseWriter, r *http.Request) {
    // Extract or set deadline
    ctx := r.Context()
    deadline, ok := ctx.Deadline()
    if !ok {
        var cancel context.CancelFunc
        ctx, cancel = context.WithTimeout(ctx, 5*time.Second)
        defer cancel()
        deadline, _ = ctx.Deadline()
    }

    // Calculate budget for downstream call
    remaining := time.Until(deadline)
    if remaining < 100*time.Millisecond {
        http.Error(w, "Insufficient time budget", http.StatusRequestTimeout)
        return
    }

    // Reserve 200ms for local processing + network
    downstreamCtx, cancel := context.WithTimeout(ctx, remaining-200*time.Millisecond)
    defer cancel()

    result, err := downstreamService.Call(downstreamCtx, request)
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            http.Error(w, "Downstream timeout", http.StatusGatewayTimeout)
            return
        }
    }
    json.NewEncoder(w).Encode(result)
}
```

#### gRPC Deadline Propagation

```go
// gRPC automatically propagates deadlines via context
func (s *server) GetUser(ctx context.Context, req *pb.UserRequest) (*pb.User, error) {
    // Check remaining time
    deadline, ok := ctx.Deadline()
    if ok {
        remaining := time.Until(deadline)
        if remaining < 50*time.Millisecond {
            return nil, status.Error(codes.DeadlineExceeded, "insufficient time")
        }
    }

    // Downstream call inherits deadline from ctx
    posts, err := s.postService.GetPosts(ctx, req.UserId)
    // ...
}
```

---

## 4. Connection Pool Warm Connection Strategies

### Pool Sizing: Little's Law Application

```
Required connections = (request_rate × avg_latency) / 1000

Example:
- 1000 req/s
- 100ms average latency
- Required: 1000 × 0.1 = 100 connections

Add 20-30% buffer for bursts: 120-130 connections
```

### Detection Commands

```bash
# Check current pool usage
# Java (via JMX)
jconsole  # Navigate to MBeans → HikariCP → Pool

# Or via command line
jcmd <pid> GC.heap_info | grep HikariPool

# Python (Django)
python manage.py shell
>>> from django.db import connection
>>> connection.queries  # Active queries
>>> len(connections._connections)  # Pool size

# Check for pool exhaustion
grep -E "pool.*exhaust|waiting for connection|connection timeout" application.log

# Monitor connection lifetimes
ss -tan | awk '$1=="ESTAB" {print $5}' | sort | uniq -c | sort -rn

# Track connection churn (high = poor pooling)
netstat -s | grep "passive connections opening"
```

### Strategy 1: Lazy Initialization with Warm-Up

```python
# Python (psycopg2) - Warm up pool during startup
from psycopg2 import pool

connection_pool = pool.ThreadedConnectionPool(
    minconn=10,   # Minimum connections (created immediately)
    maxconn=100,  # Maximum connections
    host="db.example.com",
    database="mydb",
    user="user",
    password="pass"
)

# Warm-up function
def warm_up_pool():
    connections = []
    for _ in range(connection_pool.minconn):
        conn = connection_pool.getconn()
        conn.cursor().execute("SELECT 1")  # Prime the connection
        connections.append(conn)

    # Return connections to pool
    for conn in connections:
        connection_pool.putconn(conn)

# Call during app startup
warm_up_pool()
```

### Strategy 2: Health Check Background Thread

```java
// Java (HikariCP) - Built-in health checks
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://db.example.com/mydb");
config.setUsername("user");
config.setPassword("pass");

// Pool settings
config.setMinimumIdle(10);
config.setMaximumPoolSize(100);

// Health check settings
config.setConnectionTestQuery("SELECT 1");
config.setKeepaliveTime(30000);  // 30s - ping idle connections
config.setMaxLifetime(600000);   // 10min - recycle old connections
config.setIdleTimeout(300000);   // 5min - close idle connections

// Validate on borrow (optional, adds latency)
config.setConnectionTimeout(3000);  // 3s timeout for new connections

HikariDataSource ds = new HikariDataSource(config);
```

### Strategy 3: Predictive Scaling

```go
// Go - Adjust pool size based on load
type AdaptivePool struct {
    minConns int
    maxConns int
    current  int
    mu       sync.Mutex
}

func (p *AdaptivePool) scale() {
    p.mu.Lock()
    defer p.mu.Unlock()

    // Monitor metrics
    activeConns := getCurrentActiveConns()
    requestRate := getRequestRate()
    avgLatency := getAvgLatency()

    // Calculate required connections (Little's Law)
    required := int(requestRate * avgLatency / 1000)
    required = int(float64(required) * 1.3)  // 30% buffer

    // Scale within bounds
    target := clamp(required, p.minConns, p.maxConns)

    if target > p.current {
        // Scale up gradually
        p.scaleUp(min(target-p.current, 10))
    } else if target < p.current-20 {  // Hysteresis
        // Scale down gradually
        p.scaleDown(min(p.current-target, 5))
    }
}

// Run scaler in background
go func() {
    ticker := time.NewTicker(10 * time.Second)
    for range ticker.C {
        pool.scale()
    }
}()
```

### Strategy 4: Connection Pre-Binding to NUMA Nodes

```bash
# For NUMA systems: Bind connections to specific NUMA nodes
# Reduces cross-NUMA memory access latency

# Check NUMA topology
numactl --hardware

# Bind process to NUMA node 0
numactl --cpunodebind=0 --membind=0 java -jar app.jar

# Or in code (C)
#include <numa.h>
numa_set_preferred(0);  // Allocate on node 0
```

---

## 5. TCP Fast Open (TFO) Deployment Issues

### What TFO Does

Saves 1 RTT by including application data in SYN packet. Requires cookie from previous connection.

```
Standard TCP:
Client: SYN →
Server:     ← SYN-ACK
Client: ACK →
Client: DATA →
Total: 2 RTT before data

TCP Fast Open:
Client: SYN + DATA + Cookie →
Server:                       ← SYN-ACK + DATA
Total: 1 RTT before data
```

### Deployment Issues in Production

#### Issue 1: Middlebox Interference

**Problem:** Firewalls, load balancers, and NAT devices drop SYN packets with data.

```bash
# Test TFO support
# Client-side
sysctl -w net.ipv4.tcp_fastopen=1  # Enable client TFO

curl --tcp-fastopen https://example.com

# If TFO fails silently → Middlebox interference
# Check with tcpdump
tcpdump -i any 'tcp[tcpflags] & tcp-syn != 0' -vvv -X | grep -A 10 "TFO"

# Common failure: No TFO cookie in SYN-ACK
# = Server or middlebox stripped TFO option
```

**Mitigation:**

```bash
# Fallback to standard TCP when TFO fails
# Most implementations do this automatically

# Check TFO statistics
nstat -az | grep -i "tfo\|fastopen"

# TcpFastOpenActive: Client TFO attempts
# TcpFastOpenActiveFail: Client TFO failures
# TcpFastOpenPassive: Server TFO accepted
# TcpFastOpenPassiveFail: Server TFO rejected

# If fail rate > 10%, disable TFO
```

#### Issue 2: Security - Replay Attacks

**Problem:** Data in SYN can be replayed by network duplicates.

```
Attack scenario:
T+0ms:  Client sends: SYN + "DELETE /account/123"
T+1ms:  Server receives, processes, responds
T+2ms:  Attacker duplicates SYN packet
T+3ms:  Server receives duplicate, processes AGAIN
Result: Account deleted twice
```

**RFC 7413 recommendation:** Only use TFO for idempotent requests (GET, HEAD, PUT, DELETE with idempotency keys).

**Detection:**

```bash
# Monitor for replayed TFO requests
bpftrace -e '
tracepoint:tcp:tcp_receive_reset {
  @resets[args->sport] = count();
}
tracepoint:tcp:tcp_fastopen {
  printf("TFO from %s:%d\n", ntop(args->saddr), args->sport);
}'

# Check for duplicate requests within narrow time window
awk '{print $1, $7}' access.log | sort | uniq -c | awk '$1 > 1 {print}'
```

**Mitigation:**

```python
# Application-level: Only enable TFO for safe methods
# Django example
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')

# Nginx config
location /api {
    # Disable TFO for non-idempotent endpoints
    tcp_fastopen off;
}

location /api/read-only {
    # Enable TFO for read-only
    tcp_fastopen on;
}
```

#### Issue 3: Cookie Management

**Problem:** TFO requires server to store cookies per-client. Memory overhead + cookie rotation complexity.

```bash
# Server-side: Enable with cookie cache
sysctl -w net.ipv4.tcp_fastopen=2  # Server only
sysctl -w net.ipv4.tcp_fastopen_key=<hex-key>  # 16-byte key

# Check cookie cache size
cat /proc/sys/net/ipv4/tcp_fastopen_blackhole_timeout_sec
# Default: 3600s (1 hour)

# Monitor cookie usage
ss -ti | grep "ts " | wc -l  # Connections with timestamps (TFO uses timestamps)
```

**Mitigation:**

```bash
# Rotate keys periodically (prevent cookie forgery)
# Generate new key
openssl rand -hex 16 > /etc/tfo_key

# Apply atomically
sysctl -w net.ipv4.tcp_fastopen_key=$(cat /etc/tfo_key)

# Automate rotation (weekly)
echo "0 0 * * 0 openssl rand -hex 16 | xargs -I {} sysctl -w net.ipv4.tcp_fastopen_key={}" | crontab -
```

#### Issue 4: Browser Support Abandonment

**Reality check:** Major browsers DISABLED TFO by default.
- Firefox: Removed in v87 (2021) due to compatibility issues with TLS 1.3
- Chrome: Disabled by default, enabled only for specific domains via QUIC
- Safari: Limited support

**Key insight:** TFO is effectively dead for HTTP/HTTPS traffic. Use HTTP/3 (QUIC) instead, which includes 0-RTT resumption WITHOUT TFO's security issues.

### When to Use TFO (Rare Cases)

```
Use TFO IF:
✓ Internal datacenter traffic (no middleboxes)
✓ Custom protocols (not HTTP)
✓ Idempotent operations ONLY
✓ Measured RTT savings justify complexity

Do NOT use TFO IF:
✗ Internet-facing services
✗ Non-idempotent operations
✗ HTTP/HTTPS traffic (use HTTP/3 instead)
✗ Passing through firewalls/load balancers
```

---

## 6. Timeout Configuration Best Practices

### Comprehensive Timeout Hierarchy

```yaml
# Infrastructure Layer
network:
  tcp_syn_retries: 2        # 7s total (fast fail)
  tcp_keepalive_time: 60    # Detect dead connections
  tcp_keepalive_intvl: 10
  tcp_keepalive_probes: 3

load_balancer:
  idle_timeout: 60s         # Close idle connections
  connection_timeout: 5s    # Fail fast on backend unavailable

# Application Layer
frontend:
  request_timeout: 30s      # User-facing requests
  connect_timeout: 3s       # Connection establishment
  read_timeout: 27s         # Leave 3s for processing

backend:
  server_timeout: 25s       # Fail before frontend
  handler_timeout: 20s      # Reserve 5s for cleanup
  db_timeout: 15s           # Reserve 5s for result processing
  cache_timeout: 1s         # Fast fail on cache miss

# Database Layer
database:
  connection_timeout: 5s    # Fast fail on connection
  query_timeout: 10s        # Prevent long-running queries
  idle_timeout: 300s        # Recycle idle connections
  max_lifetime: 3600s       # Prevent stale connections
```

### Timeout Anti-Pattern Detection

```bash
# Script: Detect timeout misconfigurations
#!/bin/bash

# 1. Find all timeout settings
find . -type f \( -name "*.py" -o -name "*.js" -o -name "*.go" -o -name "*.java" \) \
  -exec grep -Hn -E "(timeout|Timeout|TIMEOUT)[^a-zA-Z]*([:=]|=)" {} \;

# 2. Check for hardcoded timeouts > 30s
find . -type f \( -name "*.py" -o -name "*.js" -o -name "*.go" -o -name "*.java" \) \
  -exec grep -Hn -E "(timeout.*[3-9][0-9]{2}|timeout.*[1-9][0-9]{3})" {} \; \
  | grep -v "^\s*//"  # Exclude comments

# 3. Verify server timeout < client timeout
# Extract timeout values and compare
# (Implementation depends on codebase structure)

# 4. Check for missing timeouts on external calls
grep -rn "http.Get\|requests.get\|fetch" --include="*.go" --include="*.py" --include="*.js" \
  | grep -v "timeout"
```

### Production Timeout Decision Matrix

| Service Type | Connect TO | Read TO | Server TO | Justification |
|--------------|-----------|---------|-----------|---------------|
| User-facing API | 3s | 27s | 25s | Users abandon > 30s |
| Internal RPC | 1s | 9s | 8s | Fast fail for retries |
| Database query | 5s | 10s | - | Prevent long queries |
| Cache lookup | 100ms | 900ms | - | Cache should be fast |
| External API | 5s | 25s | - | Account for WAN latency |
| Health check | 1s | 2s | - | Fast failure detection |
| Batch job | 30s | 3600s | 3570s | Long operations allowed |

### Monitoring and Alerting

```promql
# Prometheus queries for timeout issues

# 1. High timeout rate
rate(http_request_timeouts_total[5m]) > 0.01

# 2. Requests exceeding 95% of timeout
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
  > (http_request_timeout_seconds * 0.95)

# 3. Client timing out before server
rate(client_timeout_total[5m]) > 0
  and rate(server_processing_total[5m]) > rate(server_timeout_total[5m])

# 4. Connection pool exhaustion (proxy for timeout issues)
connection_pool_wait_duration_seconds > 5
```

---

## Decision Trees for LLM Troubleshooting

### Tree 1: Connection Establishment Failures

```
Is connection establishment failing?
├─> Check: ss -tan state syn-sent | wc -l
├─> If count > 0: SYN packets stuck
│   ├─> Check: nstat -az | grep TcpRetransSegs
│   ├─> If high retransmits:
│   │   ├─> Network issue OR
│   │   ├─> Backend unresponsive OR
│   │   └─> Firewall dropping SYNs
│   └─> Fix:
│       ├─> Reduce tcp_syn_retries for fast fail
│       ├─> Implement circuit breaker
│       └─> Check backend health
└─> If count = 0: Issue is elsewhere (not SYN)
```

### Tree 2: Load Balancer Timeout Errors

```
Are you seeing 502/504 errors from load balancer?
├─> 502 Bad Gateway:
│   ├─> Check: Backend keep-alive < LB idle timeout?
│   │   ├─> If NO: Backend closes after LB → 502
│   │   └─> Fix: Reduce backend keep-alive to 80% of LB idle
│   └─> Check: Backend health check passing?
│       └─> If NO: LB sending to dead backend
│
└─> 504 Gateway Timeout:
    ├─> Check: Backend response time > LB timeout?
    │   ├─> If YES: Backend too slow
    │   └─> Fix: Increase LB timeout OR optimize backend
    └─> Check: Backend timeout > LB timeout?
        └─> If YES: Backend outlives LB → wasted work
            └─> Fix: Backend timeout = LB timeout - 5s
```

### Tree 3: Connection Pool Exhaustion

```
Are requests waiting for connections?
├─> Check: Connection pool metrics (active/idle/waiting)
├─> If waiting > 0:
│   ├─> Pool undersized?
│   │   ├─> Calculate: L = λ × W (Little's Law)
│   │   └─> Fix: Increase max connections
│   ├─> Connections leaking?
│   │   ├─> Check: Are connections returned to pool?
│   │   └─> Fix: Add connection try-finally blocks
│   └─> Connections stale?
│       ├─> Check: Long idle connections timing out?
│       └─> Fix: Enable keep-alive, reduce max_lifetime
└─> If waiting = 0: Pool healthy, issue elsewhere
```

### Tree 4: Timeout Misconfiguration

```
Seeing duplicate requests/work?
├─> Check: Client timeout < Server timeout?
│   ├─> If YES: Client retries while server still working
│   └─> Fix: Server timeout = Client timeout - network latency - buffer
│
└─> Seeing partial results?
    ├─> Check: Timeout propagation through layers?
    ├─> If NO: Downstream calls have independent timeouts
    └─> Fix: Implement context-based deadline propagation
```

---

## Quick Reference Card

### Detection Command Cheat Sheet

```bash
# TCP SYN Issues
ss -tan state syn-sent | wc -l                      # Stuck SYNs
nstat -az | grep TcpRetransSegs                     # Retransmits
tcpdump 'tcp[tcpflags] & tcp-syn != 0' -nn          # Capture SYNs

# Load Balancer
aws elbv2 describe-load-balancer-attributes ...     # Check timeouts
grep "X-Amzn-Trace-Id" log | sort | uniq -d         # Duplicate IDs
ss -ti | grep "timer:\|retrans:"                    # Connection state

# Timeouts
grep -E "timeout|timed out" app.log | wc -l         # Client timeouts
awk '...' server.log                                # Late successes
grep "request_id" log | awk '{print $NF}' | sort | uniq -d  # Retries

# Connection Pools
jcmd <pid> GC.heap_info | grep HikariPool           # Java pools
ss -tan | awk '$1=="ESTAB"' | wc -l                 # Active connections
netstat -s | grep "passive connections"             # Connection churn

# TCP Fast Open
nstat -az | grep -i "tfo\|fastopen"                 # TFO statistics
ss -ti | grep "ts " | wc -l                         # TFO-capable connections
```

### Configuration Quick Reference

```bash
# TCP Tuning
sysctl -w net.ipv4.tcp_syn_retries=2                # Fast SYN fail (7s)
sysctl -w net.ipv4.tcp_fastopen=3                   # Enable TFO (risky)
sysctl -w net.ipv4.tcp_keepalive_time=60            # Detect dead conns

# Timeouts (General Rule)
# Server timeout = Client timeout - 5s - (2 × network_latency)
# Example: 30s client → 23s server (assuming 1s RTT)

# Connection Pools
# Size = (requests/sec × avg_latency_sec) × 1.3
# Example: 1000 rps × 0.1s × 1.3 = 130 connections
```

---

## Sources

- [TCP Syn Retries](https://medium.com/@avocadi/tcp-syn-retries-f30756ec7c55)
- [Understanding TCP SYN Timeouts](http://www.justsomestuff.co.uk/wiki/doku.php/linux/syn_tcp_timeout)
- [tcp(7) - Linux manual page](https://www.man7.org/linux/man-pages/man7/tcp.7.html)
- [Network timeouts](https://mechpen.github.io/posts/2019-11-17-network-timeouts/)
- [Linux TCP_RTO_MIN, TCP_RTO_MAX and the tcp_retries2 sysctl](https://pracucci.com/linux-tcp-rto-min-max-and-tcp-retries2.html)
- [When TCP sockets refuse to die](https://blog.cloudflare.com/when-tcp-sockets-refuse-to-die/)
- [Elastic Load Balancing Connection Timeout Management](https://aws.amazon.com/blogs/aws/elb-idle-timeout-control/)
- [Troubleshoot your Application Load Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-troubleshooting.html)
- [Why Your AWS Load Balancer Connections Keep Dropping at 60 Seconds](https://medium.com/@shiprakhare04/why-your-aws-load-balancer-connections-keep-dropping-at-60-seconds-and-how-to-fix-it-0265ab329b9c)
- [Cassandra tuning for Request timed out](https://groups.google.com/a/lists.datastax.com/g/java-driver-user/c/i8bFAiOU50U)
- [Anti-patterns which all Cassandra users must know](https://medium.com/analytics-vidhya/anti-patterns-which-all-cassandra-users-must-know-1e54c60ff1fa)
- [Cassandra driver retry policies](https://docs.datastax.com/en/datastax-drivers/connecting/retry-policies.html)
- [TCP Fast Open Wikipedia](https://en.wikipedia.org/wiki/TCP_Fast_Open)
- [RFC 7413 - TCP Fast Open](https://datatracker.ietf.org/doc/html/rfc7413)
- [TCP Fast Open? Not so fast!](https://blog.apnic.net/2021/07/05/tcp-fast-open-not-so-fast/)
- [The Art of HTTP Connection Pooling](https://devblogs.microsoft.com/premier-developer/the-art-of-http-connection-pooling-how-to-optimize-your-connections-for-peak-performance/)
- [HTTP keep-alive, pipelining, multiplexing & connection pooling](https://www.haproxy.com/blog/http-keep-alive-pipelining-multiplexing-and-connection-pooling)
- [Apache HttpClient Connection Management](https://www.baeldung.com/httpclient-connection-management)
- [Timeout in a distributed system (Microservices)](https://harish-bhattbhatt.medium.com/timeout-in-a-distributed-system-microservices-4fa36c611850)
- [Timeout Strategies in Microservices Architecture](https://www.geeksforgeeks.org/system-design/timeout-strategies-in-microservices-architecture/)
- [Mastering Connection Timeouts in Microservices](https://medium.com/tuanhdotnet/mastering-connection-timeouts-in-microservices-techniques-and-best-practices-3ce96a115539)
