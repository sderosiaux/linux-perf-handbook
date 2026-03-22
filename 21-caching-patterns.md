# Caching Patterns at Scale

Production insights on distributed caching, Redis encoding, and cache architecture.

Sources: Twitter, Deliveroo, GitHub, Slack, Instagram, ClickHouse, Netflix

---

## Redis Memory Encoding Optimization

### Ziplist Encoding Thresholds (Deliveroo)

Redis uses two encoding modes per data structure: a compact **ziplist** (sequential memory) and a full hash table. The threshold is config-controlled.

**Memory overhead per element by structure:**

| Structure | Overhead/element | Notes |
|---|---|---|
| Flat key | 63 bytes | Per Redis key overhead |
| Hash | 70 bytes | Full hashtable encoding |
| Set | 194 bytes | Full encoding (no scores) |
| ZSET | 122 bytes | Includes 8-byte score |
| Hash (ziplist) | ~37 bytes | When < 512 elements |
| ZSET (ziplist) | ~37 bytes | When < 128 elements |

**Ziplist config (redis.conf):**
```
hash-max-ziplist-entries 512   # default; safe for most use cases
hash-max-ziplist-size -2       # 8KB per ziplist node

zset-max-ziplist-entries 128   # default; performance cliff beyond this
zset-max-ziplist-size -2
```

**Performance tradeoff vs flat keys:**
- Set: 5% slower hits, 1–3% slower misses
- Hash: 10% slower hits, **15% slower near ziplist limit** (128–512 entries)
- ZSET: 10% slower with ziplist, **20% slower without**; 25% lower write throughput on large sets

**Memory savings at scale (10M IDs, 32 bytes each):**
- Flat keys: ~630 MB
- ZSET ziplist with ~256 partitions: **320 MB (65% reduction)**
- Bloom filter (1-in-million false positive): **80 MB (90% reduction)**

```
# Sweet spot: ~256 partitions balances savings and operational simplicity
# Beyond 512 entries/partition: ziplist disabled, savings vanish
```

### Hybrid List Encoding (Twitter — 105TB, 39M QPS)

Twitter's Timeline Service uses a **ziplist-of-ziplists** ("Hybrid List"):

```
┌──────────┐    ┌──────────┐    ┌──────────┐
│ ziplist  │───▶│ ziplist  │───▶│ ziplist  │
│ (≤ N KB) │    │ (≤ N KB) │    │ (≤ N KB) │
└──────────┘    └──────────┘    └──────────┘
```

- Ziplist threshold = maximum Timeline size in bytes
- When a single list exceeds the threshold, ziplists are linked
- Ziplists retained until completely empty (prevents fragmentation)
- Each ziplist fits in CPU cache lines

**Scale context:**
- Timeline Service: ~30M QPS, ~40TB heap, 6,000+ instances
- BTree deployment: ~65TB heap, ~9M QPS, 4,000+ instances
- Redis server latency: **< 0.5ms**; JVM-via-Redis: **~10ms**

**Network bottleneck rule:**
```
On 1 Gbps link at 100K+ RW/s:
  avg object > 1 KB → network becomes the bottleneck before CPU
  avg object < 1 KB → CPU bound first
```

---

## When to Replace Redis with MySQL (GitHub)

GitHub moved timeline/organization data from Redis to a MySQL-backed `GitHub::KV` store. Decision thresholds:

**MySQL viable when:**
```
Peak write rate        < 2,100 keys/second  (p98 < 1,500 keys/s)
Replication delay      < 180ms at peak
mset store writes      < 270/second
mget store reads       < 460/second
```

**What was saved (GitHub organization timelines):**
- 350M+ daily Redis writes reduced **65%** by switching write-time → read-time filtering
- Additional **30% reduction** (~11% total) via deduplication of timeline writes

**Migration pattern:**
```
1. Dual-write to MySQL + Redis simultaneously (feature flag)
2. Reads still from Redis (no change to latency budget)
3. Prioritize migration by write volume, NOT read volume
4. Flag flip for reads only after write parity confirmed for 7+ days
```

**`GitHub::KV` design (InnoDB-backed):**
- Key expiration support built-in
- Batching + throttling library (Redis pipeline semantics)
- Dropped: update timestamps, expiration metadata (reduced schema surface)

---

## Migrating from Redis Queues to Kafka (Slack — 1.4B jobs/day)

**Scale before migration:** 50 Redis clusters serving job queue.

**Redis failure mode that drove migration:**
```
Memory exhaustion → dequeue impossible
→ queue buildup accelerates exhaustion
→ runaway feedback loop → full outage
```

**Kafka configuration (post-migration):**
```properties
# 16 brokers on i3.2xlarge EC2
# Per topic: 32 partitions, RF=3, 2-day retention
unclean.leader.election.enable=true   # availability over consistency
acks=1                                 # leader-only, minimize latency

# 50 topics × 32 partitions = 1,600 monitored partitions
```

**Operational safeguards:**
```
JQRelay: Consul locks — exactly one relay process per Kafka topic
Heartbeat canaries: enqueued every minute per partition for lag detection
Rate limits: configurable via Consul watch API (live, no redeploy)
```

**Resilience tested:**
- Single broker kill: ✓
- 2 brokers in one AZ simultaneously: ✓
- All 3 brokers killed (forced unclean election): ✓
- Full cluster restart: ✓

---

## Distributed S3 Cache (ClickHouse)

**Problem:** S3 latency (tens of ms) dwarfs intra-zone network latency (100–250 µs) — remote cache economically viable.

**Architecture:**
```
Query → Worker → Local disk cache (LRU) → S3
                     ↑
              Read-through + write-through
```

**Performance:**
| Query type | Cold | Hot |
|---|---|---|
| Full scan 32GB compressed | 29.7s | 3.8s |
| Scattered reads | 0.21s | 59ms |
| 6 parallel nodes | 4.4s | 0.7s |

**Design decisions:**
- **LRU only** — immutable column files need no invalidation logic
- **Write-through** — every insert/merge is immediately cached (no separate warming job)
- **No TTL** — content-addressed data never expires; eviction is capacity-driven
- **Consistent hashing** across nodes — horizontal scale without remap

**When LRU is sufficient:** If cached objects are immutable (write-once, content-addressed, append-only), LRU without invalidation is architecturally correct and eliminates an entire class of consistency bugs.

**Remote cache justification formula:**
```
cache_overhead_ms << origin_latency_ms
100–250 µs << 30,000–100,000 µs (S3)  → 2 orders of magnitude margin
```

---

## Cache Warming Strategies

### Write-Through as Warming (ClickHouse)
Wire the write path through the cache so every new insert is immediately warm. Eliminates cold-start for recent data without a separate warming job.

### Cold-Start on Process Restart
- Keep cache external to process (Redis, Memcached, S3-backed) — process restarts don't evict
- For in-process caches: warm from DB on startup before accepting traffic
- Canary pods that warm before the rest of the fleet (prevents thundering herd on cache miss)

---

## See Also

- [Database Profiling](14-database-profiling.md) — Redis active expiration, LZ4 compression
- [Resilience Patterns](20-resilience-patterns.md) — Rate limiting algorithms, cache invalidation patterns
- [Kafka & Messaging](22-kafka-messaging.md) — Queue patterns, DLQ design
