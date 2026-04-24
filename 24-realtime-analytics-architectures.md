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
| RisingWave | Kappa (SQL MVs) | Rust/Tokio async | Hummock LSM on S3 (shared) | No | ~1s (checkpoint floor) |

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

## Netflix Keystone — Stream Processing at 1 Trillion Events/Day

**Scale:** 130M subscribers, 190+ countries, trillions of events/day, thousands of concurrent streaming jobs.

### Architecture

```
One isolated Flink cluster per job (not shared)
Container runtime: Netflix Titus (CPU/memory/network isolation)
Deployment: Spinnaker → Titus
State: ZooKeeper (consensus) + S3 (checkpoint/savepoint storage)
Source of truth: AWS RDS (desired goal states) — entire system reconciles against it
```

**Declarative reconciliation model:** Users declare goal state via UI/API; platform drives current → goal autonomously. All operations are idempotent — transient failures self-resolve.

**Self-healing monitor (per-job, inside each Flink cluster):**
- Detects Task Manager drift vs. container runtime view
- Stalled Job Manager leader election
- Unstable / periodically-restarting task managers
- Network partition on any container
- Blast radius isolation: each job = independent Flink cluster

**Job diversity / configuration knobs:**
```
State size:   stateless → 10s of TB local state
Window sizes: seconds → hours-long custom session windows
Failure recovery: seconds target (harder with large state + shuffling stages)

Stateless redeployment: low-latency (may duplicate) vs. low-duplicate (higher latency)
Stateful redeployment: resume from checkpoint/savepoint or start fresh
Backfill: dynamic source switching (stateless only) or rewind to checkpoint
Connectors: Kafka, Elasticsearch, Hive — dynamic source/sink switching without rebuild
```

**ZooKeeper state corruption recovery:** full cluster rebuild from RDS source of truth.

---

## Netflix Delta — CDC + Data Synchronization

**Problem:** Multi-store sync (MySQL → Elasticsearch + Memcached) via dual writes or distributed transactions both fail:
- Dual writes: second-write failure diverges stores permanently
- XA transactions: block on process crash, no deadlock detection, ES doesn't support XA

### Kafka Cluster Config (High-Durability vs Standard)

| Config | Standard Keystone Kafka | Delta's special cluster |
|---|---|---|
| `unclean.leader.election.enable` | `true` | **`false`** |
| Replication factor | 2 | **3** |
| Min in-sync replicas | 1 | **2** |
| Producer acks | 1 | **`all`** |
| Broker storage | Local disks | **EBS volumes** |

**EBS impact on broker replacement:** new instance attaches prior broker's EBS volume → catch-up time from **hours → minutes**. Decoupled storage/broker lifecycles drastically reduce recovery blast radius.

### Delta-Connector (CDC Service)

```
Sources: MySQL binlog, Postgres WAL, Cassandra (multi-master), DynamoDB streams
Full dump: manual trigger (entire DB, specific table, or specific PKs)
Chunked dumps: failure mid-dump resumes from chunk, not from scratch
No table locks during dumps (never blocks write traffic)
HA: standby instances across AWS AZs
```

**Data flow (Movie Search example):**
```
MySQL mutation → Delta-Connector → Kafka → Delta Flink app
    → enrichment calls (Deal Service, Talent Service, Vendor Service)
    → Elasticsearch index (near real-time)
```

**Delta Stream Processing (on SPaaS/Flink):**
- Annotation-driven DSL — users write enrichment logic; Flink details abstracted
- Built-in: deduplication, schematization, fault tolerance
- Self-service UI for Flink job deployment + dynamic config changes without recompile

---

## RisingWave — Streaming SQL with Decentralized State

**Source:** RisingWave Labs OSS codebase (`github.com/risingwavelabs/risingwave`, verified against `src/stream/`, `src/storage/hummock`, `src/meta/` at commit `aaa8dbd3b4`).

Postgres-compatible streaming DB. Same problem class as Flink + Materialize (incremental materialized views over unbounded streams) but with a distinctive operational posture.

### Architectural Pipeline

```
SQL
 └─ frontend (Calcite-style planner)        src/frontend/
      └─ streaming physical plan (protobuf)
           └─ meta scheduler                src/meta/src/stream/
                │  fragments the DAG, assigns actors to compute nodes
                ▼
           compute nodes                    src/stream/
             from_proto/ → Executor tree
             Actor::run() loop driven by barriers
                │
                ├─ state → Hummock (shared LSM on S3)    src/storage/hummock/
                └─ output → Dispatcher → downstream fragment or Sink
```

**Key distinction vs Flink:** state lives in **Hummock**, a shared LSM backed by S3, not in per-task RocksDB. Scaling = redistributing vnodes, new actors load state from object storage. No stateful rebalancing cost.

### Execution Model — Pull-Based Async Streams

The operator contract (`src/stream/src/executor/mod.rs:240`):

```rust
pub trait Execute: Send + 'static {
    fn execute(self: Box<Self>) -> BoxedMessageStream;
}
```

An operator = `self → Stream<Message>`. Composed with `#[try_stream]` (futures-async-stream). No event loop — Tokio scheduler drives the pipeline; backpressure is `await`-native.

`Message` variants flowing between operators:

| Variant | Payload | Role |
|---|---|---|
| `Chunk(StreamChunk)` | Columnar batch with `Op` per row: `Insert` / `Delete` / `UpdateInsert` / `UpdateDelete` | The **delta** — retractions enable correct incremental joins/aggs |
| `Barrier(Barrier)` | `EpochPair{prev, curr}` + optional `Mutation` | Dual-purpose: checkpoint marker **and** reconfiguration command |
| `Watermark(Watermark)` | Lower bound on event-time for a column | Window close, late-data filter, TTL |

### Barriers = Chandy-Lamport + Control Plane

Injected ~1Hz by the meta node. Propagation rule at every operator: receive `Barrier(N)` on all upstreams → align → flush state to epoch `N` → forward downstream → `StateTable.commit(N)` to Hummock.

Reconfiguration piggybacks on barriers via `Mutation` (`src/stream/src/executor/mod.rs:362`):

```
Mutation::Add / Update / Stop / Pause / Resume
         / Throttle / SourceChangeSplit / StartFragmentBackfill / ...
```

Epoch N-1 runs with old topology, N with new — atomically. Scale up/down, add MV, pause a source: all zero-downtime.

### Sharding — vnodes

Default **256 vnodes** per fragment (`VirtualNode::COUNT_FOR_COMPAT = 1 << 8`, `src/common/src/hash/consistent_hash/vnode.rs:62`). An actor owns a vnode bitmap. Rebalancing = redistribute vnodes across actors; Hummock keys are vnode-prefixed so state "moves" implicitly.

### Backfill — The Non-Obvious Part

`CREATE MV AS SELECT ...` over an existing table must snapshot historical data **while** new mutations arrive (`src/stream/src/executor/backfill/`, `chain.rs`, `rearranged_chain.rs`). Interleaves:
- Batch scan cursor over base table
- Live CDC stream from upstream

Upstream updates are filtered against the scanned-prefix boundary. When cursor reaches end → switch to pure streaming.

### vs Flink / Materialize — Tradeoff Matrix

| Axis | RisingWave | Flink | Materialize |
|---|---|---|---|
| State backend | Hummock LSM on S3 (shared) | RocksDB local per task | Timely in-memory + persist layer |
| Rescale cost | Seconds (vnode reassignment, state on S3) | Minutes (restore from checkpoint) | Seconds–minutes |
| Theoretical model | Retraction streams, no cycles | Retraction + append, barriers | Timely dataflow + differential |
| Watermark model | Scalar per column, `min` on merge | Scalar per column, `min` on merge | Multi-dim pointstamps (frontiers) |
| Primary interface | SQL only | DataStream API + SQL | SQL only |
| Runtime | Tokio async, pull-based | JVM threads, push-based | Rust timely, push-based |

**When RisingWave wins:** cloud-native deployments where S3-based state means you pay object-storage prices for large state instead of provisioning local NVMe; frequent topology changes (adding MVs, rescaling).

**When Flink wins:** ultra-low-latency (sub-10ms), complex DataStream logic that SQL can't express, mature ecosystem of connectors.

**When Materialize wins:** correctness-critical workloads where differential dataflow's formal guarantees matter; iterative/recursive queries.

### Operational Observations

- **Checkpoint interval ~1s** → p99 latency floor is the checkpoint period for consistent reads.
- **Hummock compaction** is the LSM-level concern — see `35-lsm-compaction-strategies.md`. Compactor is a separate service, can be scaled independently.
- **Barrier alignment** is the typical stall point: one slow upstream delays the whole epoch. Monitor `actor_current_epoch` metric to find laggards.
- **Backpressure** surfaces as growing gap between `prev_epoch` and wall clock at a source operator.

---

## Distributed Time in Stream Engines — Lineage

The mechanism every stream engine uses to reason about progress descends from the same theoretical root. Naming it explicitly helps cross-reference across systems.

### The Academic Chain

```
Lamport 1978 ("Time, Clocks, and the Ordering of Events")
  └─ scalar logical clock, C(a) < C(b) if a → b causally
       │
       ▼
Mattern/Fidge 1988 (vector clocks)
  └─ V[i] per process, partial order detects true concurrency
       │
       ├─ Chandy-Lamport 1985 (distributed snapshot)
       │    └─ "marker" messages sweep the graph → consistent cut
       │         │
       │         ▼
       │    Carbone et al. 2015 (Flink ABS)
       │         └─ barriers = markers specialized for dataflow
       │
       ▼
Naiad / Murray-McSherry 2013 ("Timely Dataflow")
  └─ pointstamp = (epoch, loop_1, loop_2, ...) — vector clock for dataflow
  └─ frontier = min of in-flight pointstamps — progress guarantee
       │
       ├─ Differential Dataflow / Materialize
       │    └─ keeps full pointstamp lattice
       │
       └─ Flink / RisingWave
            └─ simplified: scalar epoch + scalar watermark per column
            └─ loses: cycles, nested iteration scopes
            └─ gains: simpler implementation, adequate for SQL MVs
```

### What Each System Actually Implements

| System | Progress primitive | Math object | Detects concurrency? |
|---|---|---|---|
| Kafka consumer offsets | Per-partition offset | Vector of scalars, no causality | No (append-only log) |
| Flink barriers | Global epoch counter | Lamport scalar | No (total order by design) |
| Flink watermarks | Per-column event-time bound | Lamport scalar | No |
| RisingWave `EpochPair` | Global checkpoint epoch | Lamport scalar | No |
| RisingWave watermark | Per-column event-time bound | Lamport scalar | No |
| Naiad pointstamp | `(epoch, *loop_counters)` | Vector clock, partial order | **Yes** (cycles + event-time) |
| Materialize frontier | Antichain of pointstamps | Semi-lattice | **Yes** |
| CRDT version vector | `[node_id → counter]` | Vector clock | **Yes** (concurrent writes) |
| HLC | Physical time + logical counter | Lamport + wall clock | No (total order) |

See also: [`crdt-lock-free-distributed-state.md`](crdt-lock-free-distributed-state.md) for vector clocks applied to replicated state (same math, different problem).

### The Merge Rule — Same Pattern Everywhere

When data from multiple streams/nodes meets:

- **Lamport on receive:** `C := max(C_local, C_received) + 1`
- **Vector clock on receive:** `V[i] := max(V_local[i], V_received[i])` for all i
- **Watermark at merge/join:** `W_out := min(W_in_1, W_in_2, ...)`
- **Frontier advancement:** `F := min over all in-flight pointstamps`

`max` when reconstructing "what has happened" (causality), `min` when computing "what is guaranteed complete" (frontier). Same algebra, dual operations on the same lattice.

### Why Simplification Works for SQL MVs

SQL materialized views don't express cycles (no tail-recursive CTE in streaming). Therefore:
- No need for loop counters in timestamps → epoch scalar is sufficient
- No need for full lattice frontier → per-column scalar watermark is sufficient
- Retractions handle "update" semantics → no need for pointstamp diffs

TRADEOFF: can't express iterative computations (graph algorithms, fixed-point queries) without external orchestration. If you need those → Materialize or raw Timely.

### Practical Debugging Implications

When a stream pipeline stalls, the question is always: **which progress primitive is stuck?**

| Symptom | Which clock is stuck | Where to look |
|---|---|---|
| No output, epoch advancing | Watermark not advancing | Upstream source idle detection, event-time extractor |
| Epoch not advancing | Barrier blocked | Slowest upstream actor, state flush latency |
| Output has "old" data only | Backfill not done | `BackfillExecutor` progress, base-table scan cursor |
| Random ordering of updates | Retractions reordered | Join-key partitioning, shuffle correctness |

---

## Inaccessible Articles (403 / ECONNREFUSED)

- Netflix Keystone (medium.com, 403)
- Netflix Delta (medium.com, 403)
- Walmart Lambda Architecture (medium.com, 403)
- Teads 100B events/day (medium.com, 403)
- King RBEA (techblog.king.com, ECONNREFUSED)

Medium's paywall blocks automated fetches. King's tech blog appears offline.
