# Database Scaling Patterns

Sharding strategies, ID generation, connection pooling, and distributed query engines.

Sources: Instagram, Notion, Pinterest, Square, PayPal, Uber, Lyft, Netflix, Quora

---

## ID Generation for Sharded Systems

### Instagram — 64-bit k-Sortable IDs via PostgreSQL

**Bit layout:**
```
[ 41 bits: ms timestamp ] [ 13 bits: logical shard ID ] [ 10 bits: per-shard sequence ]
  ↑ 69 years from epoch    ↑ 8,192 shards max           ↑ 1,024 IDs/ms/shard
```

- **Epoch offset:** `2011-01-01T00:00:00Z` (custom, not Unix 0) → smaller timestamps, more bits for future
- **Max throughput:** 1,024,000 IDs/second per shard
- **No central coordinator** — each shard generates IDs independently via PostgreSQL sequence

**PostgreSQL stored procedure per schema:**
```sql
CREATE OR REPLACE FUNCTION insta5.next_id(OUT result bigint) AS $$
DECLARE
  our_epoch bigint := 1314220021721;  -- ms since 2011-01-01 UTC
  seq_id bigint;
  now_ms bigint;
  shard_id int := 5;                  -- hardcoded per schema at creation
BEGIN
  SELECT nextval('insta5.table_id_seq') % 1024 INTO seq_id;
  SELECT FLOOR(EXTRACT(EPOCH FROM clock_timestamp()) * 1000) INTO now_ms;
  result := (now_ms - our_epoch) << 23;
  result := result | (shard_id << 10);
  result := result | (seq_id);
END;
$$ LANGUAGE PLPGSQL;
```

**Architecture:** thousands of logical shards → each gets its own PostgreSQL schema (`insta5`, `insta6`, ...) with its own sequence. Physical hosts are added under logical shards — no resharding needed.

### Pinterest — Type-Encoded IDs

**Bit layout:**
```
[ 18 bits: shard ID ] [ 10 bits: type ID ] [ 36 bits: local auto-increment ]
  ↑ 262,144 shards     ↑ 1,024 obj types   ↑ per-shard MySQL AUTO_INCREMENT
```

**Advantages:**
- Routing requires only a bitmask — zero lookup table overhead
- Object type encoding enables per-type sharding strategies without schema changes
- 4,096 virtual shards (later expanded) → adding hosts = remapping virtual shards

### ID Generation Decision Tree

```
Need distributed IDs?
├── Single sequence per shard → PostgreSQL function (Instagram)
│   Bit layout: [41 time][13 shard][10 seq]
│   k-sortable, no coordinator, 1M IDs/sec/shard
│
├── Type-aware routing needed → encode type in ID (Pinterest)
│   [18 shard][10 type][36 local]
│   Zero-lookup routing by bitmask
│
└── App-layer generation → Snowflake/Twitter model
    Same bit layout, application generates, no DB dependency
```

---

## Shard Count Selection

### Notion — 480 Logical Shards / 32 Physical Hosts

**Why 480:**
```
480 is divisible by: 32, 40, 48, 60, 80
→ Incremental host addition: 32 → 40 → 48 (no full remap)

Powers of 2 force doubling: 32 → 64 hosts
Highly composite numbers allow: 32 → 40 → 48 → 60 → 80 → ...
```

| Dimension | Value |
|---|---|
| Logical shards | 480 |
| Physical RDS instances | 32 |
| Shards per host | 15 |
| Target IOPS budget | 60,000 total |
| Max table size | 500 GB |
| Max per-host data | 10 TB |

**Implementation:** 480 PostgreSQL schemas per cluster (`schema001.block`, `schema002.block`, ...) rather than native PG partitioning — simpler routing logic.

**Partition key:** Workspace UUID — entire workspace on one shard. All tables reachable from `block` are co-located → eliminates cross-shard joins for dominant access pattern.

### Shard Count Rules

| Rule | Rationale |
|---|---|
| Logical >> physical (480/32) | Rebalance by remapping, not resharding |
| Highly composite number (480, 720) | Incremental host addition without full remap |
| Avoid powers-of-2 at small counts | Forces doubling |
| Partition key = highest-cardinality natural key | User ID > timestamp > random UUID |
| Never use timestamp as shard key | Creates write hotspot on "current" shard |

---

## Zero-Downtime Reshard Playbook

**Notion lesson:** Catch-up script must drain in <30 seconds for zero-downtime cutover. If backfill rate < write rate, a maintenance window is required.

```
1. Audit log / CDC capture on source tables
   └─ captures all writes during migration

2. Provision new shard topology (keep old live)

3. Backfill new shards from source
   └─ version-stamped, idempotent (skip stale records)
   └─ Notion: ran m5.24xlarge (96 vCPUs), ~3 days for production dataset

4. Dark reads: compare N% of reads against both stores
   └─ separate team writes independent verification logic

5. Gradual traffic shift by user cohort:
   1% → 5% → 25% → 50% → 100% over days

6. Monitor: error rate, p99 latency, replication lag

7. Keep old shards read-only for rollback window (≥ 7 days)

8. Drop old shards only after full validation
```

**Regrets (Notion, post-mortem):**
- Should have used composite `(id, space_id)` key from the start
- Shard before you're under load — migration under traffic pressure is painful

---

## Connection Pool Sizing

### Formula

```
pool_size = (core_count × 2) + effective_disk_spindle_count

effective_disk_spindle_count:
  NVMe SSD  → 1
  HDD RAID  → number of spindles
  EBS       → 1 (network-bound I/O)

Minimum: 10 connections
Add: +10% headroom for monitoring/admin connections
```

Source: HikariCP "About Pool Sizing" — validated by Square (Cash App) and PayPal (Hera) patterns.

**Saturation signal:**
```
Queue depth > 0 for > 100ms → backpressure to application tier
```

### PayPal Hera — Sidecar Connection Multiplexer

Scale: **100s of billions of DB queries/day** across MySQL, Oracle, PostgreSQL.

```
App Process → Hera (localhost:11111) → DB Connection Pool → DB
```

- Multiplexes many goroutines over a fixed pool of persistent DB connections
- Eliminates TCP handshake + auth overhead per query
- Per-shard connection pools — no cross-shard sharing

**Connection states monitored (`state-log`):**
```
init | acpt | wait | busy | schd | fnsh | quce | asgn | idle | bklg | strd
                                                               ↑
                                                        bklg = backlog (pool saturated)
                                                        → triggers query eviction
```

On saturation: poorly performing queries are **evicted** before DB is overwhelmed. Transparent failover to replica on primary failure.

---

## Distributed Query Engines (Presto/Trino)

### Uber Presto at Scale

| Metric | Value |
|---|---|
| Workers per cluster | 300+ |
| Accessible data | 5+ PB |
| Query completion (< 60s) | > 90% |
| Daily data processed (Alluxio) | 50 PB |
| Daily queries | 500K |
| Daily active users | 9,000 |

**Custom Parquet reader (2–10x speedup):**
1. **Nested column pruning** — skip unneeded columns in nested schemas (5+ levels deep)
2. **Columnar reads** — avoid row-to-column conversion overhead
3. **Predicate pushdown** — filter at scan time, not post-read
4. **Dictionary pushdown** — skip row groups using dictionary pages

### Presto Worker Config Baseline

```properties
# config.properties (worker)
query.max-memory-per-node=8GB
query.max-total-memory-per-node=10GB
task.concurrency=16                          # = vCPU count
task.writer-count=4

# Spill (enable for batch/ETL only — avoid on interactive clusters)
experimental.spill-enabled=true
experimental.spiller-spill-path=/mnt/spill
experimental.spiller-max-used-space-threshold=0.7

# S3/HDFS connector
hive.s3.max-connections=500                  # avoid S3 throttling at scale
hive.orc.use-column-names=true
hive.parquet.use-column-names=true
```

```ini
# jvm.config
-Xmx<85%_of_node_RAM>
-XX:+UseG1GC
-XX:G1HeapRegionSize=32M
-XX:+ExplicitGCInvokesConcurrent
```

**Join strategy by data size:**
| Build-side size | Strategy |
|---|---|
| < 1 GB | Broadcast join (default) |
| Large-large | Partitioned (repartition) join |
| Skewed data | `TABLESAMPLE` or pre-aggregate |

### Cluster Isolation Strategy (Lyft)

```
Dedicated cluster per use case:
  ad-hoc cluster    → interactive queries, SLA: sub-minute
  ETL cluster       → batch queries, SLA: hours
  dashboard cluster → fixed queries, SLA: seconds

Why: runaway ETL queries starve interactive users when sharing
Implementation: separate coordinator endpoints, not query tagging
```

### Alluxio Local NVMe Cache (Uber — 65% → 90% hit rate)

```
Worker node: 500 GB NVMe SSD cache per node
Cache key format: hdfs://<path><mod_time>
```

**Cache hit rate:**
- Baseline: ~65%
- With selective caching filter: **> 90%**

**Selective caching filter** (cache only if):
- Hot tables (high access frequency)
- Stable partitions (low update frequency)
- Common partition access patterns

**Consistency:**
```
Stale cache on update → append Hive mod_time to cache key (auto-invalidates)
Node join/leave       → consistent hashing on virtual ring (not modulo)
Cross-restart         → file-level metadata persisted to disk
```

**S3 best practices for Presto:**
```
Use ORC with predicate pushdown — Presto skips row groups natively
Always filter on partition columns (date, region) — avoids full S3 list
Compact small files (< 128 MB) nightly via Spark before Presto reads
S3 Select enabled for simple filters — reduces data transfer
```

---

## MyRocks Migration (HBase → MySQL/RocksDB at Quora)

**Motivation:**
| Metric | HBase | MyRocks |
|---|---|---|
| Storage per TB | Baseline | ~50% reduction (factoring HDFS 3x replication) |
| Operational complexity | High (HDFS, ZooKeeper, RegionServers) | Lower (MySQL-compatible) |
| Read latency p99 | High (RPC hops) | Lower (local disk) |

**MyRocks config:**
```ini
rocksdb_block_cache_size=32G
rocksdb_max_background_jobs=8
rocksdb_compaction_style=level
rocksdb_block_size=16384                      # 16 KB blocks

# Tiered compression: speed for hot levels, ratio for cold
rocksdb_compression_per_level=lz4,lz4,lz4,zstd,zstd,zstd,zstd
```

**Migration pattern:**
```
1. Shadow write: new writes → HBase + MyRocks simultaneously
2. Backfill: scan HBase → write MyRocks (deduplicate by timestamp)
3. Dark reads: random-sample comparison (10% traffic)
4. Gradual shift: 10% → 25% → 50% → 100%
5. HBase decommission: after 30+ days with no anomalies
```

**Compression tiers:** LZ4 for L0-L2 (speed, hot data), Zstd for L3+ (ratio, cold data). Net: ~50% storage reduction vs HBase with HDFS 3x replication.

---

## MySQL Replication at Scale

### GitHub freno — Replication Throttling

**Architecture:**
```
Percona pt-heartbeat (100ms inserts on primary)
     ↓
Replica calculates lag:
  SELECT unix_timestamp(now(6)) - unix_timestamp(ts) AS replication_delay
     ↓
freno aggregates: MAX across all monitored replicas (never average)
     ↓
Client-side memcached cache (TTL=20ms): ~800 rps → ~50 rps to freno
```

**Key thresholds:**
```
Remove replica from pool: ~5s lag (old) → target sub-second
p95 write-wait: < 600ms
Batch write discipline: 50–100 rows per segment between lag checks
```

**Throttling control:**
```bash
.freno throttle gh-ost ttl 120 ratio 0.5   # restrict to 50% throughput for 120s
```

**Impact:** 30% of reads shifted from primary to replicas.

**Read-after-write consistency trick (GitHub):**
```
Store write timestamp in job payload
→ Block execution until freno confirms lag < elapsed time since write
→ Strong consistency without querying primary
```

### Meta MySQL Raft (FlexiRaft)

**Architecture:** Raft consensus embedded in MySQL via `MyRaft` plugin (kuduraft fork).

```
Ring topology: 12 members = 3 regions × (1 primary-capable + 2 logtailers + 3 read replicas)
Data quorum: 2/3 in-region (small)
Leader election quorum: crosses regions (large) — FlexiRaft asymmetric
```

**Key timing:**
```
Heartbeat interval: 500ms
Election timeout:   3 missed heartbeats = ~1.5s
Graceful promotion: ~300ms
Total failover:     ~2s (vs 20–40s with semisync → 10x improvement)
```

**Write path:**
```
1. Engine prepare → in-memory binlog payload
2. Group commit assigns GTID; Raft assigns OpId (term:index)
3. Raft compresses + writes to binlog; user thread blocks
4. 2/3 in-region votes → engine commit → client response
5. Commit marker sent async to out-of-region followers
```

**Crash recovery scenarios:**

| State at crash | Recovery |
|---|---|
| Transaction unprepared | Lost in-memory; engine rolls back on restart |
| Binlog flushed, not replicated | Prepared txn rolled back; new leader truncates via No-Op |
| In binlog + majority, primary crashed before engine commit | New leader reapplies on commit marker receipt |

**Quorum Shatter** (2+ bad entities in one region): manual fix via `Quorum Fixer` Python tool — identifies longest-log entity, overrides quorum, promotes, resets.

### ProxySQL Monotonic Read Consistency (Shopify)

**Problem:** Replica lag (seconds to minutes) causes inconsistent reads when successive queries hit different replicas at different lag points.

**Solution — UUID hash pinning:**
```sql
/* consistent_read_id:550e8400-e29b-41d4-a716-446655440000 */ SELECT ...
```

ProxySQL hashes UUID → integer → modulo against weight-adjusted server count → deterministic replica selection.

**Critical gotcha:** ProxySQL's default post-selection rebalancing breaks consistency — **disable rebalancing** for `consistent_read_id` requests.

**Rule:** Never average replica lag — always use **max across all monitored replicas** (GitHub/freno principle).

---

## See Also

- [Database Profiling](14-database-profiling.md) — Query profiling, slow query analysis, Cassandra ops
- [Resilience Patterns](20-resilience-patterns.md) — Consistent hashing (Ring Hash, Maglev)
- [Kafka & Messaging](22-kafka-messaging.md) — Connection pooling, queue design
- [Storage Engine Patterns](19-storage-engine-patterns.md) — LSM-tree, RocksDB internals
