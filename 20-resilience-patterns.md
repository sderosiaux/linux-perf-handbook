# Resilience Patterns

Production-proven patterns for building systems that degrade gracefully under failure.

Sources: Martin Fowler, Netflix/Hystrix, SoundCloud, Shopify, DoorDash, Stripe, Cloudflare, Figma, Yahoo, Allegro

---

## Circuit Breakers

### State Machine

```
CLOSED ──(threshold exceeded)──► OPEN ──(sleep_window elapsed)──► HALF-OPEN
  ▲                                                                     │
  └────────────────(probe succeeds)───────────────────────────────────┘
  OPEN ◄──────────────────────────(probe fails)────────────────────────
```

**States:**
- **CLOSED**: normal operation, failures counted
- **OPEN**: requests fail-fast, no upstream calls
- **HALF-OPEN**: one probe request allowed; success → CLOSED, failure → OPEN

### Threshold Models

**Count-based (avoid):** consecutive failure count. A single request in a quiet period determines state — breaks at low traffic.

**Rate-based in rolling window (preferred):**
```
# Hystrix defaults:
rolling_window     = 10,000ms / 10 buckets (1s resolution)
min_requests       = 20          # must see ≥20 req in window before opening
error_threshold    = 50%         # open when error rate ≥50%
health_snapshot_interval = 500ms # not real-time
```

**Duration-based window** (Finagle/SoundCloud): tracks success % over a time duration, not request count. Handles variable-traffic services correctly.

### Production Configuration (Shopify)

> Most circuit breakers are misconfigured. The math matters.

```
# Key parameters:
half_open_resource_timeout  = p99 healthy latency  (NOT the full service timeout)
error_timeout (sleep window) = 30s+                (NOT 2s — short windows cause thrashing)
success_threshold            = 3                   (NOT 1 — prevents flip-flop)
```

**Why `half_open_resource_timeout` = p99 healthy latency:**
```
# With 42 Redis instances, error_timeout=2s, half_open_resource_timeout=0.25s:
additional_utilization = (42 × 0.25 / 2.0) × 2 = 263%  → total outage

# Fixed: error_timeout=30s, half_open_resource_timeout=0.05s (= p99):
additional_utilization = (42 × 0.05 / 30.0) × 2 = 4%   → barely noticeable
```

**Why `success_threshold=3`:**
- At 90% error rate, threshold=1: P(flip-flop) ≈ 90% per probe cycle → circuit constantly thrashes
- At 90% error rate, threshold=3: P(flip-flop) ≈ 0.1% per cycle → stable

**Why `error_timeout` must be long (30s+):**
- Short sleep window → frequent probing → sustained load on already-failing service → prevents recovery

### Bulkhead Isolation (Thread Pools)

```
# One thread pool per downstream dependency (Hystrix model)
# Exhausting pool A cannot block pool B

thread_pool_size   = 10 (Hystrix default)
queue_size         = -1 (SynchronousQueue — immediate reject on overflow, no queuing)
rejection_threshold = 5 (secondary limit when queue > 0)

# Semaphore isolation (alternative): no timeout enforcement possible
# Thread pool isolation: full timeout + rejection control
```

**Sizing bulkheads** (from production at scale):
```
# The stateful component (DB, cache) determines cluster size
# Everything else is scaled relative to it
cluster = stateful_replicas + app_servers + burst_nodes
# Each cluster is independent — Class A shared services are the cascade vector
```

**Capacity formula (LinkedIn Voyager):**
```
cluster_size = ceil((vertical_max_qps × 1.20) / host_max_qps)
# 20% headroom to absorb downstream anomalies
# Measure actual per-host QPS by shedding real traffic — estimates are off by 30–300%
```

### Graceful Degradation Taxonomy (LinkedIn)

1. **Trade with time** — serve from cache / alternate PoP
2. **Trade with complexity** — use simpler/cached data source (ranked → generic results)
3. **Trade with feature** — return default/partial response instead of error

---

## Rate Limiting Algorithms

### Algorithm Comparison

| Algorithm | Storage | Atomicity | Burst | Accuracy | Use case |
|---|---|---|---|---|---|
| Fixed window counter | O(1) | Redis INCR (atomic) | 2× at boundary | Good | Simple, most cases |
| Sliding window log | O(n) per user | Redis ZSET (atomic) | Exact | Exact | Low-volume, strict |
| Sliding window approx | O(1) | INCR × 2 | Smooth | ~6% error | High-scale (Cloudflare) |
| Token bucket | O(1) | Needs Lua/lock | Yes | Good | Burst allowance |
| Leaky bucket | O(1) | Non-atomic in memcached | No | Good | Smooth output |

### Sliding Window Approximation (Cloudflare)

Stores only 2 integers per key. 0.003% incorrect decisions at billions of req/day.

```python
# rate = prior_count × ((period - elapsed) / period) + current_count
def is_allowed(key, limit, period):
    now = time.time()
    current_window = int(now / period)
    elapsed = now % period

    prev_count = redis.get(f"{key}:{current_window - 1}") or 0
    curr_count = redis.get(f"{key}:{current_window}") or 0

    rate = prev_count * ((period - elapsed) / period) + curr_count
    if rate >= limit:
        return False

    redis.incr(f"{key}:{current_window}")
    redis.expire(f"{key}:{current_window}", period * 2)
    return True
```

**Why not token bucket in memcached:** no compare-and-swap → race condition. Two threads both read 1 remaining token, both proceed. Fix requires Redis Lua (adds complexity) or distributed lock (adds latency).

### Sliding Window Log — Exact (Figma/Redis)

```python
# Redis sorted set per user: score = value = timestamp
def is_allowed(key, limit, window_seconds):
    now = time.time()
    pipe = redis.pipeline()
    pipe.zremrangebyscore(key, 0, now - window_seconds)  # evict stale
    pipe.zcard(key)                                        # count remaining
    pipe.zadd(key, {str(now): now})                        # log this request
    pipe.expire(key, window_seconds)
    _, count, _, _ = pipe.execute()

    return count < limit
# O(n) memory — one entry per request. Exact but expensive at scale.
# Deliberately keeps counting post-limit: extends ban for malicious actors.
```

### Layered Rate Limiting (Stripe's 4-tier model)

```
Tier 1: Request rate limiter    — N req/s per user (token bucket, Redis)
         Primary limiter. Allows brief bursts for real-world event spikes.

Tier 2: Concurrent request limiter — max N in-flight requests simultaneously
         Combats retry storms on CPU-bound endpoints. Better signal than time-based
         rate for resource consumption.

Tier 3: Fleet load shedder      — reject non-critical traffic when fleet > 80%
         Reserve 20% worker capacity for critical requests. HTTP 503.

Tier 4: Worker utilization shedder — last resort, 4-priority traffic tiers
         critical_methods > POSTs > GETs > test_mode
         Sheds gradually to prevent oscillation/flapping.
```

**Operational safeguards:**
- Middleware failures must NOT propagate to API (exception isolation mandatory)
- Feature flags for per-limiter kill switches
- Dark-launch validation before enforcement
- HTTP 429 (rate limited) vs. 503 (capacity) — different semantics, different client behavior

### Distributed Rate Limiting (Yahoo Cloud Bouncer)

```
Architecture: MySQL policies → per-node Controller (cached) → in-process check (~microseconds overhead)

Gossip protocol (UDP):
- Each packet: last-second traffic data only (minimal size)
- Convergence: O(log N) — 1,000+ nodes in practice
- Tradeoff: cross-DC gossip latency compromises accuracy

Token bucket config:
  refill_rate    = N tokens/second
  max_capacity   = burst ceiling
  refill_interval = seconds
```

---

## Load Balancing Algorithms

### Power-of-Two-Choices (P2C) — Least Request

```
pick 2 random backends → route to the one with fewer active requests
```

- Eliminates herd behavior of pure round-robin under variable latency
- O(1) decision, no global state
- Envoy: `lb_policy: LEAST_REQUEST` with `active_request_bias` (default 1.0)

```yaml
# Envoy config
load_balancing_policy:
  policies:
    - typed_extension_config:
        name: envoy.load_balancing_policies.least_request
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.load_balancing_policies.least_request.v3.LeastRequest
          active_request_bias:
            default_value: 1.0   # 0.0 = round-robin, higher = more aggressive least-req
```

### Ring Hash vs. Maglev (Consistent Hashing)

| Property | Ring Hash (Ketama) | Maglev |
|---|---|---|
| Lookup speed | O(log N) | O(1) fixed 65,537-entry table |
| Key remapping on host change | ~1/N keys | ~2/N keys (2× less stable) |
| Distribution quality | Depends on ring size | Uniform by construction |
| Small cluster behavior | Poor with low `min_ring_size` | Good |
| Weight support | Yes (proportional entries) | Yes (min 1 entry/host) |

```yaml
# Envoy: ring hash — tune ring size for distribution quality
common_lb_config:
  consistent_hashing_lb_config:
    minimum_ring_size: 1024    # default too small for < 100 hosts
    maximum_ring_size: 8388608
```

### Katran: XDP/eBPF L4 Load Balancer (Meta)

> Runs on every backend server — no dedicated L4LB fleet.

```
NIC DMA → XDP hook (eBPF program) → IP-in-IP encapsulation → backend
         ↑ before kernel network stack, per-CPU-queue parallel
```

**Architecture properties:**
- Fully lockless: per-CPU BPF maps, zero synchronization on hot path
- Linear throughput scaling with NIC queue count
- Zero CPU waste when idle (vs. DPDK busy-poll loops)
- DSR-only: response traffic bypasses L4LB entirely
- VIP management: ExaBGP + ECMP at switch layer

**Consistent hashing:** Extended Maglev — adds unequal weight support (critical during hardware refresh). Hash computation fits in L1 cache.

**Constraints:**
```
- DSR-only (no SNAT)
- L3 topology required
- No IP fragmentation, no IP options
- Max 3.5 KB packet size
- Single-interface deployment
```

### Stateless L4 Director + Rendezvous Hashing (GitHub GLB)

```
Router (ECMP) → L4 Director (stateless, rendezvous hash) → L7 Proxy (HAProxy, TCP termination)
                                                              ↓
                                                        Backend servers (DSR response)
```

- Rendezvous (HRW) hashing: constant-time, stateless, per-flow consistent
- FoU (Foo-over-UDP) encapsulation: director never terminates TCP
- HAProxy: graceful connection draining — long-running Git clones survive maintenance
- ECMP at router ensures within-flow director consistency

---

## DNS Production Failures

### AWS VPC Resolver Limit (Stripe)

> **Hard limit: 1,024 DNS packets/second per ENI.** Hadoop PTR reverse-lookups hit this silently.

**Cascade pattern:**
```
1 job bursts PTR lookups → VPC resolver 1024 pps limit hit
→ Unbound request list fills (check: unbound-control dump_requestlist 0)
→ client retries × 5 + resolver retries + upstream retries = ~7× amplification
→ SERVFAIL storms across all DNS clients on affected hosts
```

**Diagnosis:**
```bash
# Check Unbound request list depth
unbound-control dump_requestlist 0   # thread 0; look for "iterator wait" entries

# Count DNS packets to VPC resolver per second
iptables -L OUTPUT -v -n -x | grep <vpc_resolver_ip>
# Or add a counter rule:
iptables -I OUTPUT -d 169.254.169.253 -p udp --dport 53 -j LOG --log-prefix "DNS: "

# Rotating DNS packet capture (30-minute window)
tcpdump -n -tt -i any -W 30 -G 60 -w '%FT%T.pcap' port 53

# Unbound stats
unbound-control stats_noreset | grep -E "total\.|num\."
```

**Fix: local Unbound per node with zone forwarding**
```
# /etc/unbound/unbound.conf
server:
  interface: 127.0.0.1
  do-not-query-localhost: no

forward-zone:
  name: "in-addr.arpa."                    # PTR queries → local resolution
  forward-addr: 127.0.0.1@5353

forward-zone:
  name: "consul."
  forward-addr: 127.0.0.1@8600

forward-zone:
  name: "."
  forward-addr: 169.254.169.253            # VPC resolver for everything else

# Each node has its own ENI → 1,024 pps budget per node, not shared
```

**Rule:** PTR (reverse DNS) lookups are a hidden load multiplier. Rate-budget them separately. Never perform PTR lookups synchronously in hot paths.

### DNS-Based Load Balancing (Dropbox)

Geo-DNS fails when ISP peering agreements create topological distance not reflected in geography.

**Better approach: latency-measured routing**
```
1. Client measures latency to all edge PoPs (background)
2. Map client subnet → resolver IP (test queries to random subdomains)
3. Aggregate: subnet → resolver → per-PoP p75 latency
4. Route each resolver subnet to its lowest-p75-latency PoP
5. Push as JSON to DNS platform (NS1/Route53) using EDNS Client Subnet (ECS)
```

**Results vs. geo-baseline:**
- 10–15% improvement p75/p95 overall
- Vladivostok: 300–400ms → ~150ms (was routing via Frankfurt, now Tokyo)
- Egypt: 10% gain (Milan → Paris, following cable topology)

```bash
# Verify EDNS Client Subnet is being sent
dig +subnet=0/0 example.com @resolver
dig +subnet=1.2.3.4/24 example.com @8.8.8.8  # test specific subnet routing

# Check what PoP a resolver resolves to
dig +short example.com @<isp_resolver>
```

---

## See Also

- [Network Tuning](09-network-tuning.md) — TCP parameters, SNAT/PAWS gotcha
- [Latency Analysis](13-latency-analysis.md) — coordinated omission, percentile measurement
- [Observability & Metrics](12-observability-metrics.md) — alerting, SLOs
- [Container & K8s](07-containers-k8s.md) — cgroup-level isolation, CPU throttling
