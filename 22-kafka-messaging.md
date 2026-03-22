# Kafka & Distributed Messaging at Scale

Production operations, failure modes, and queue design patterns.

Sources: LinkedIn, Dropbox, Yelp, Confluent, Uber, Segment, Indeed, Slack

---

## Kafka at Scale

### Scale Reference Points

| Company | Scale | Notes |
|---|---|---|
| LinkedIn | 7T messages/day, 4,000+ brokers, 100+ clusters, 7M partitions | Largest single clusters: 140+ brokers, 1M replicas each |
| Dropbox | 60 MB/s per broker throughput limit | Fully randomized messages (no compression benefit) |
| Slack | 1.4B jobs/day, 1,600 partitions | 50 topics × 32 partitions, 16 i3.2xlarge brokers |

### Dropbox — Per-Broker Throughput Limits

```
Max throughput: 60 MB/s per broker (conservative, randomized payloads)
Safe operating capacity: ~50 MB/s per broker (20% failure tolerance headroom)
Recommended partitions per broker: 10 (avoid write contention)
Max consumer groups before network bottleneck: 5
```

**Health thresholds:**
```
IO thread idle % < 20%  → worker thread saturation, reduce load
ISR changes > 50%       → broker replication lag, investigate immediately
```

**Important:** The 60 MB/s figure used fully randomized messages. Real production payloads compress significantly — test with production-representative data.

### LinkedIn — Multi-Tier Topology

```
Local cluster (per datacenter)
      ↓ MirrorMaker cross-DC replication
Aggregate cluster
      ↓
Consumer services
```

**Kafka Audit:** compares producer counts vs consumer counts at each tier boundary — messages can silently disappear at tier boundaries without this infrastructure.

**Custom operational features LinkedIn built:**
- "Maintenance mode" for brokers: prevents new partition assignment during removal
- Minimum `replication.factor` enforcement at topic creation
- Billing/accounting on produce/consume usage per team

---

## Kafka Configuration Reference

### Broker Tuning

```properties
# Replication
num.replica.fetchers=4              # increase on high-partition brokers
replica.fetch.max.bytes=1048576     # match producer batch expectations

# Network + IO threads
num.network.threads=9               # ~3× CPU cores for network-heavy workloads
num.io.threads=16                   # scale with disk count

# Log segments
log.segment.bytes=1073741824        # 1 GB (reduces small file overhead)
log.retention.hours=168             # 7 days default; tune per topic
```

### Producer Tuning

```properties
batch.size=65536                    # 64 KB (default 16 KB too small at scale)
linger.ms=5                         # allow batching; 0 = latency-optimized
compression.type=lz4                # best throughput/CPU tradeoff at scale
acks=all                            # required for EOS; acks=1 for latency-priority
```

### Exactly-Once Semantics (EOS)

```properties
# Producer
enable.idempotence=true
acks=all
max.in.flight.requests.per.connection=5   # max safe with idempotence; >5 breaks ordering on retry
retries=2147483647                         # effectively infinite
transactional.id=<unique-per-producer-instance>   # MUST be unique per instance
transaction.timeout.ms=60000              # default; tune for long batches

# Broker
min.insync.replicas=2                     # requires acks=all to enforce

# Consumer
isolation.level=read_committed            # skip aborted transaction messages
enable.auto.commit=false                  # mandatory for EOS
```

**EOS failure modes:**
```
max.in.flight > 5 + idempotence → reordering on retry
non-unique transactional.id    → fencing fails, duplicate transactions
transaction.timeout too short  → batch aborted mid-way, consumer sees no data
isolation.level=read_uncommitted (default) → reads aborted data
```

---

## Linux/OS Tuning for Kafka Brokers

```bash
# File descriptors (/etc/security/limits.conf)
kafka  soft  nofile  128000
kafka  hard  nofile  128000

# VM — Kafka is page-cache-dependent
sysctl -w vm.swappiness=1              # never swap; 0 can prevent OOM killer
sysctl -w vm.dirty_background_ratio=5
sysctl -w vm.dirty_ratio=80           # allow large dirty pages before blocking writes

# Disk scheduler (HDD only — use none/noop for NVMe)
echo deadline > /sys/block/sdX/queue/scheduler

# Network buffers
sysctl -w net.core.rmem_max=134217728
sysctl -w net.core.wmem_max=134217728
sysctl -w net.ipv4.tcp_rmem='4096 65536 134217728'
sysctl -w net.ipv4.tcp_wmem='4096 65536 134217728'
sysctl -w net.ipv4.tcp_window_scaling=1
```

---

## ZooKeeper Migration Without Downtime (Yelp — "ZooKeeper Mitosis")

### Phase 1: DNA Replication

```
Source ZK cluster: .1, .2, .3
Target ZK cluster: .4, .5, .6 (start with .6 dormant)
```

```bash
# 1. Add 2 destination nodes to source zoo.cfg → 5-node joint cluster
#    Keep .6 dormant to preserve source quorum

# 2. Rolling restart with 10-second intervals between nodes
#    Start with destination nodes first

# 3. Update zookeeper.connect in Kafka config:
zookeeper.connect=192.168.1.4,192.168.1.5,192.168.1.6/kafka

# 4. Rolling restart Kafka (no cluster-wide downtime)
```

**Speed up data migration:**
```bash
# zkcopy with transaction batching: 10x speed improvement
# mtime filtering: reduces downtime window from 25 minutes → <2 minutes
zkcopy --src zk://source-host:2181/kafka --dst zk://dest-host:2181/kafka \
       --batch 1000 --since <epoch_ms>
```

### Phase 2: Mitosis (Cluster Separation)

```bash
# CRITICAL: all these steps must be simultaneous across target nodes
# Staggered restarts break quorum

# 1. Restore original zoo.cfg on all nodes (do NOT restart yet)

# 2. Apply iptables simultaneously across all target nodes:
iptables -A INPUT  -p tcp -d <source_nodes> -j REJECT
iptables -A OUTPUT -p tcp -d <source_nodes> -j REJECT

# 3. Simultaneously restart all target nodes (including dormant .6)
#    → leader election completes in isolation

# 4. Restart source cluster with original config
# 5. Remove firewall rules
```

**Gotchas:**
- Target cluster must start completely empty
- Firewall separation must be simultaneous — staggered breaks quorum
- Never share ZooKeeper between Kafka and other services (hard-to-debug perf issues)

---

## Distributed Queue & DLQ Patterns

### Multi-Level Retry Topology (Uber Reliable Reprocessing)

```
Main topic → retry-1 → retry-2 → … → DLQ
```

**Why not inline retry:**
> Inline retry blocks the entire batch while retrying. Decoupled retry topics preserve main-topic throughput.

**Error triage at dequeue point:**
```python
try:
    process(message)
except TransientError:
    publish(retry_topic_1, message)    # network blip, lock timeout, etc.
except DeterministicError:
    publish(dlq, message)             # NPE, schema mismatch, code bug
    # NEVER retry deterministic failures — wastes capacity, delays visibility
```

**DLQ operations (three primitives):**
```
list   → inspect contents (preserve)
purge  → discard (acknowledge permanent failure)
merge  → republish to retry-1 (NOT main topic — keep live traffic isolated)
```

### Exactly-Once via Consumer-Side Deduplication (Segment — 200B messages)

**Scale:** 60B deduplication keys, 1.5TB on-disk seen-set, 0.6% duplicate rate, 4-week window.

**Architecture:**
```
Kafka (at-least-once) → Dedup Worker (RocksDB) → Output topic
                                ↑
                        Partition by messageId (UUID4)
                        → same ID always hits same worker
                        → no cross-instance coordination
```

```python
def process(msg):
    if rocksdb.has_seen(msg.id):
        return  # discard — bloom filter O(1) negative lookup
    publish_to_output(msg)
    rocksdb.mark_seen(msg.id)  # WAL ensures no loss on crash
```

**Why RocksDB over Memcached:**
- Memcached: expensive, RAM-limited, requires failover replicas
- RocksDB: EBS-backed, **~100x cheaper per key**, bloom filters for fast negative lookups, WAL for crash recovery

**Deduplication window sizing:**
- 4 weeks chosen based on observed duplicate distribution (nearly all appear within hours)
- Under traffic spikes: window shrinks gracefully (size-bounded, not time-bounded) — deliberate degradation

**Kafka offset as source of truth:** After crash, reconcile RocksDB against successfully published output topic — resolves the "mark-seen vs publish" ordering race.

### RocksDB as Queue Backend (Uber Cherami)

```
Producer → Input Host → WebSocket fan-out → N Storage Hosts (RocksDB)
                                                    ↓
                              Full synchronous replication (not majority quorum)
                              Ack to producer only when ALL storage hosts confirm
```

**Extent (immutable log segment):** one RocksDB instance per extent, shared LRU block cache. Keys: `sequence_number → payload` — optimized for compaction, avoids write amplification.

**Failure on storage host crash:**
```
Extent → sealed (all inflight acks lost)
→ unacked messages redelivered on next open (at-least-once)
→ consumers MUST be idempotent
```

**DLQ behavior:** Messages exceeding max retry count → poison pill → per-consumer-group DLQ. DLQ supports purge (discard) and merge (republish to retry chain). This unblocks the consumer group without losing the message.

### Head-of-Line Blocking in RabbitMQ (Indeed — 1.5M job applications/day)

**Production incident:**
```
09:38: Service A goes down
09:38-09:41: Failed messages requeued to HEAD of queue (RabbitMQ default)
09:41: 90,000 messages in 3 minutes pile up at head
Result: all consumer threads blocked, valid messages starved
```

**Root cause:** RabbitMQ requeues failed messages to the **head**, not the tail.

**Three-tier solution:**

| Tier | Trigger | Mechanism |
|---|---|---|
| Immediate requeue | Transient (network blip, lock timeout) | Standard AMQP requeue |
| Delay queue | Recoverable (replication lag, eventual consistency) | TTL expiry → dead-letter exchange → main queue tail |
| DLQ | Exhausted retries | Manual inspection |

**Delay queue implementation:**
```
# Delay queue has NO consumers — purely TTL-based holding area
# On TTL expiry: dead-letter exchange routes to main queue TAIL
# Escalation: n immediate requeues → delay queue → m delay cycles → DLQ
```

**The structural fix is universal:** Any broker that requeues to head (Redis/Sidekiq, Resque, SQS visibility timeout race) has this failure mode. Solution: remove failed messages from main queue, park in separate structure with time-based re-entry.

---

## See Also

- [Caching Patterns](21-caching-patterns.md) — Redis encoding, Slack Redis→Kafka migration
- [Resilience Patterns](20-resilience-patterns.md) — Rate limiting, circuit breakers
- [Database Scaling](23-database-scaling.md) — Sharding, consistent hashing
