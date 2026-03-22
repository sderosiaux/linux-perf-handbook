# Real-Time Analytics Architectures: Production Patterns

Extracted from production systems at Uber, Netflix, Walmart, and Uber's mobile crash analytics platform. Focused on infrastructure decisions, concrete numbers, and failure modes that map to Linux/systems tuning.

---

## Uber: AresDB — GPU-Accelerated OLAP

**Source**: https://www.uber.com/blog/aresdb/

### Numbers

- Query latency: sub-second to low seconds for typical OLAP queries
- Data freshness: seconds (upsert path through redo log → live store)
- Scale target: millions to billions of records over a few days of data
- Kafka infrastructure feeding AresDB: **>1 trillion messages/day** (Uber-wide)

### Storage Architecture: Two-Layer Design

```
Live Store (host RAM)
  - Uncompressed columnar vectors
  - Partitioned into configurable batches
  - Primary key index for deduplication/upsert
  - Holds recent, mutable data

Archive Store (disk + selective GPU load)
  - Sorted, compressed columnar data (run-length encoding)
  - Organized by UTC day
  - Loaded into GPU memory on demand, evicted by priority/recency

Redo Log (disk)
  - Write-ahead log for crash recovery
  - Backfill queue for late-arriving records (event_time < archive cutoff)
```

### GPU Usage Pattern (CUDA)

- **Transfer model**: CPU → GPU at query time, batch-by-batch via CUDA streams
- **Execution model**: One-Operator-Per-Kernel (OOPK) using Thrust library
- **Operations on GPU**: filtering, sorting, reduction, aggregation on dimension + measure vectors
- **No persistent GPU cache**: data transferred per query, not pinned between queries (acknowledged limitation)
- **Justification**: GPU wins on compute-to-storage-bandwidth ratio vs CPU for columnar scan workloads

Query execution flow:
```
1. Identify relevant archive batches (column pruning, time range)
2. Transfer batches CPU → GPU via CUDA streams (pipelined)
3. OOPK kernels: filter → sort → reduce → aggregate
4. Return result to host
```

### Data Model

- **Fact tables**: infinite time-series streams with primary keys (for upsert)
- **Dimension tables**: bounded entity tables
- **Supported types**: Bool, Int{8,16,32}, Uint{8,16,32}, Float32, UUID, GeoPoint, GeoShape, SmallEnum (≤256), BigEnum (≤65k)
- Strings are **converted to enums pre-ingestion** — avoids dictionary encoding overhead at query time

### Lambda-Inspired Split (not full Lambda)

| Layer | Data | Mutability | Compression |
|-------|------|-----------|-------------|
| Live store | Recent (hours/days) | Mutable (upsert) | None |
| Archive store | Historical | Immutable | RLE + sorting |

- Queries span both layers transparently
- Avoids full Lambda complexity (no separate batch recomputation path)
- Archiving is a scheduled merge process, not continuous

### Failure Modes

| Failure | Mechanism | Mitigation |
|---------|-----------|------------|
| Process crash | Redo log | Replay from last checkpoint |
| Massive backfill | Backfill queue fills | Queue blocks ingestion (pre-configured size) |
| GPU OOM | Per-query transfer, no persistence | Device manager tracks memory per query |
| No replication | Not implemented | Listed as future work at publication time |

### Operational Knobs

- `archive_delay`: how long before live data becomes eligible for archiving
- `archive_interval`: how frequently archiving runs
- Memory budget: single shared pool across live vectors, archive vectors, indexes, backfill queue, temp storage
- `preloading_days` + column priority: controls which archive columns stay resident in GPU memory

---

## Uber: AthenaX — Streaming SQL at Scale

**Source**: https://www.uber.com/blog/athenax/

### Numbers

- **220+ streaming applications** in production across multiple DCs
- **Billions of messages/day** processed by AthenaX jobs
- **Up to several million messages/second** using only 8 YARN containers
- Input: Kafka infrastructure carrying >1 trillion messages/day (Uber-wide)
- Availability SLA: **99.99% uptime** for critical jobs

### Architecture

**Stack**: SQL → Flink physical plan → YARN

```
Kafka Topics
    |
    v
AthenaX Compiler
  [1] Logical plan (DAG of query semantics)
  [2] Optimized logical plan (push projections/filters down, minimize shuffle)
  [3] Physical plan (parallelism, locality, Flink operators)
    |
    v
Flink on YARN (auto-allocated containers)
    |
    v
Kafka / External Sinks
```

### Architecture Pattern: Pure Kappa

- **No batch layer** — continuous SQL on unbounded streams
- No Lambda reprocessing path
- State managed by Flink checkpointing (at-most-once / at-least-once / exactly-once configurable)
- Trade-off accepted: correctness for late data handled by watermarks, not batch recomputation

### Operational Features

- **Auto-scaling**: monitors Flink watermarks + GC stats; restarts jobs between peak/off-peak
- **Failure recovery**: automatic restart on node failure, network partition, DC failover
- **Exactly-once**: via Flink checkpointing + Kafka transactional producers (where configured)
- Resource allocation: YARN-managed, no manual container sizing per job

### Why Flink over Spark Streaming

- True record-by-record processing (not micro-batch)
- Stateful operators with managed state backends (RocksDB)
- Lower latency: milliseconds vs seconds for Spark micro-batch
- Native watermark/event-time support for out-of-order streams

---

## Uber: Maze — Funnel Visualization

**Source**: https://www.uber.com/blog/maze/

### Architecture

**Purpose**: Funnel/conversion analysis (e.g., driver sign-up flow with 50%+ conversion improvement attributed)

```
Hive (daily batch)         AthenaX (real-time stream)
       \                        /
        -> Pre-aggregation layer
               |
         OLAP DB (MemSQL explored)
               |
            Redis (aggregation result cache)
               |
         Node RPC proxy
               |
     React 16 + D3 frontend
```

Three-tier client-side cache (unusual pattern):
1. React in-state cache + Selectors (memoization)
2. Browser Web Workers + SharedWorkers (instant recalculation without re-render)
3. Node-side memory cache for common aggregations

### Data Model

- Session-based: thousands of sessions → single aggregated tree per funnel step
- Pre-aggregation before DB write — OLAP DB sees summary rows, not raw events
- Scale claim: "millions of nodes and hundreds of MB of data" per funnel request

### Gaps / Limitations Disclosed

- No explicit latency SLAs published
- MemSQL explored but final storage backend not confirmed
- No failure recovery procedures disclosed

---

## Uber: Mobile App Crash Analytics (Healthline)

**Source**: https://www.uber.com/en-SG/blog/real-time-analytics-for-mobile-app-crashes/

### Numbers

- **1.5M backend errors/second** (peak)
- **1,500 mobile crashes/second** (peak)
- Crash payload: **10–200 KB per event**
- Daily volume: **~36 TB/day**
- Retention: **45 days**

### Architecture: Hybrid Lambda

```
Mobile SDK / Backend
        |
   Enrichment pipeline
        |
      Kafka
        |
      Flink  <-- stream transformation: flatten, compress, sample
     /     \
 Kafka      Kafka
(metadata) (compressed dumps)
     |           |
  Pinot RT    Pinot RT
  (live)      (compressed)
     |           |
  Spark       Spark
  (offline    (offline
   segments)   segments)
     |           |
  Pinot        Pinot
  Offline      Offline
```

Three Pinot tables:
| Table | Content | Purpose |
|-------|---------|---------|
| `healthline_crash_event_prod` | Flattened crash metadata | Aggregation queries |
| `healthline_crash_event_denormalized_prod` | Exploded array fields | Per-field aggregation |
| `healthline_crash_event_compressed_prod` | GZipped raw crash dumps | Drill-down / raw access |

### Storage: Apache Pinot

- **Real-time segments**: ingested directly from Kafka (no broker latency)
- **Offline segments**: batch-loaded from Hive via Spark (scheduled)
- **Intentional overlap** between RT and offline segments — hedge against offline job failure
- Index types used: inverted (exact match), text (regex/partial match), range (numerical comparisons)
- Columnar storage, segment-level pruning

### Flink Sampling Strategy (Critical for Cost Control)

Problem: Compressed crash dumps caused Flink task manager OOM + hotspotting on spiky issue IDs.

Solution:
- **10-second tumbling windows** on compressed data stream only
- Bucket key includes `issue_id + time_window` (not just `issue_id`) — prevents hot partition per issue spike
- Index fields stored for **ALL crashes** (not sampled) — preserves aggregation accuracy
- Compressed payload achieves ~50–60% reduction via GZip

### vs. Elasticsearch

Explicit comparison: Pinot shows significantly lower performance degradation as time window increases (Elasticsearch degrades non-linearly). Specific numbers not published.

### Failure Modes

| Failure | Impact | Mitigation |
|---------|--------|------------|
| Offline Spark job failure | Historical gap | RT/offline overlap fills the window |
| Flink task manager OOM | Pipeline stall | Time-windowed sampling + composite bucket key |
| Pinot segment bloat | 40 GB per 6-hour table, avg 1.1 GB/segment (30 segments) | Sampling on compressed table |
| Cross-DC replication | No native Pinot support | Aggregated Kafka results replicated per region |

### End-to-End Pipeline Latency

Not explicitly quantified. Flow: `SDK → enrichment → Kafka → Flink → Kafka → Pinot RT ingestion`. Implicit sub-minute latency (used for release canary dashboards).

---

## Architecture Comparison Matrix

| System | Pattern | Stream Engine | Storage | GPU | Latency SLA |
|--------|---------|--------------|---------|-----|-------------|
| AresDB | Lambda-lite | Kafka ingest | Custom columnar (host+GPU RAM) | Yes (CUDA/Thrust) | Sub-second |
| AthenaX | Kappa | Flink on YARN | Kafka output | No | Milliseconds |
| Maze | Hybrid | AthenaX + Spark | MemSQL + Redis | No | Not disclosed |
| Healthline (Pinot) | Hybrid Lambda | Flink + Kafka | Apache Pinot (RT+offline) | No | Sub-minute |

---

## Cross-Cutting Infrastructure Patterns

### Lambda vs Kappa Decision Framework

**Choose Lambda when:**
- Historical reprocessing is a hard requirement (regulatory, ML training)
- Batch accuracy > streaming approximation (e.g., exact dedup over large windows)
- Different teams own batch vs streaming paths

**Choose Kappa when:**
- Single code path is operationally cheaper (AthenaX: one job = one SQL query)
- Flink/Spark Streaming state can handle the window size
- Late-data tolerance via watermarks is acceptable

**Hybrid (Uber Healthline pattern):**
- Real-time for recency, batch for history + cost
- Intentional segment overlap as availability hedge
- Best for: immutable event logs + OLAP query layer that supports both segment types (Pinot, Druid)

### GPU for Analytics: When It Pays

AresDB conditions where GPU wins:
1. Columnar scan on large datasets where memory bandwidth is the bottleneck
2. Aggregation workloads (sum, count, percentile) over filtered subsets
3. Data fits in GPU VRAM or can be streamed in predictable batches
4. Query parallelism >> CUDA core count (many independent aggregations)

GPU loses when:
- Data is too small (PCIe transfer overhead dominates)
- Queries are highly selective (index lookup faster than full scan)
- Workload is write-heavy (GPU not involved in ingestion path)

### Kafka as Universal Backbone

All systems use Kafka as the durable event bus. Key design decisions:
- **Topic-per-table-type** (Healthline: separate topics for metadata vs compressed dumps)
- **Flink as the stateful transformation layer** between raw Kafka and storage
- **YARN or K8s** for Flink job lifecycle, not bare-metal Flink clusters

### Pinot vs Elasticsearch for OLAP

Pinot advantages at Uber's crash analytics scale:
- Columnar storage with segment pruning → linear degradation with window size
- Native Kafka RT ingestion without separate connector complexity
- Offline segment support for cost-efficient historical storage
- Explicit index type selection (inverted/text/range) vs Elasticsearch's unified inverted index

Elasticsearch breaks down when:
- Time windows grow (inverted index degrades on large date ranges)
- High-cardinality aggregations (memory pressure on field-data cache)
- Mixed RT + historical query patterns

### Sampling as a First-Class Architectural Concern

Healthline pattern: sample at the **payload level** (compressed dumps), not at the **event level** (metadata). This preserves:
- Exact aggregation counts (all events indexed)
- Drill-down capability (sampled compressed dumps sufficient for crash analysis)
- Pipeline throughput (reduces Flink output volume by 50–60%)

Lesson: Design sampling boundaries to align with query patterns. Sampling the wrong field causes either inaccurate aggregations or inability to drill down.

---

## Linux/Systems Tuning Implications

### For GPU Analytics (AresDB-style)

```bash
# PCIe bandwidth is the bottleneck for CPU→GPU transfers
# Check PCIe topology
lstopo --of txt | grep -i pcie
nvidia-smi topo -m  # GPU-GPU and CPU-GPU interconnect matrix

# Pin CUDA streams to NUMA node local to GPU
numactl --cpunodebind=0 --membind=0 ./analytics-server

# Huge pages for GPU-accessible host memory (pinned memory)
echo 1024 > /proc/sys/vm/nr_hugepages
# Allocate via cudaMallocHost() with cudaHostAllocPortable
```

### For High-Throughput Kafka Consumers (AthenaX/Healthline)

```bash
# Flink TaskManager memory: off-heap for network buffers
# taskmanager.memory.network.fraction: 0.1 (default)
# taskmanager.memory.network.min: 64mb
# taskmanager.memory.network.max: 1gb

# OS-level: increase socket receive buffer for Kafka consumers
sysctl -w net.core.rmem_max=134217728
sysctl -w net.core.rmem_default=67108864

# Reduce GC pressure (Flink's auto-scaler watches GC stats)
# Use G1GC with short pause targets for streaming jobs
-XX:+UseG1GC -XX:MaxGCPauseMillis=100 -XX:G1HeapRegionSize=32m
```

### For Columnar Storage (Pinot/AresDB)

```bash
# Mmap for offline segment access (avoid double-buffering)
# Pinot uses mmap by default; ensure vm.swappiness is low
sysctl -w vm.swappiness=1

# Segment files benefit from readahead tuning
# For sequential scan (full segment scans):
blockdev --setra 4096 /dev/nvme0n1  # 2MB readahead

# For random access (inverted index lookups):
blockdev --setra 128 /dev/nvme0n1   # 64KB readahead

# Page cache pressure: analytics servers benefit from higher dirty ratio
sysctl -w vm.dirty_ratio=40
sysctl -w vm.dirty_background_ratio=10
```

### For Flink Sampling / Windowing (Healthline pattern)

```bash
# Flink TaskManager heap: size for state backend
# RocksDB state backend: off-heap, tune block cache
# state.backend: rocksdb
# state.backend.rocksdb.block.cache-size: 256mb
# state.backend.rocksdb.writebuffer.size: 64mb
# state.backend.rocksdb.writebuffer.count: 4

# For tumbling window operators: ensure adequate off-heap for window buffers
# taskmanager.memory.managed.fraction: 0.4
```

---

## Inaccessible Articles (403 / ECONNREFUSED)

- Netflix Keystone (medium.com, 403)
- Netflix Delta (medium.com, 403)
- Walmart Lambda Architecture (medium.com, 403)
- Teads 100B events/day (medium.com, 403)
- King RBEA (techblog.king.com, ECONNREFUSED)

Medium's paywall blocks automated fetches. King's tech blog appears offline.
