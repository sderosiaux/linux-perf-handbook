# Big Data Platforms, ML Infrastructure & Vector Search

Production insights on large-scale data processing, distributed training, and similarity search.

Sources: Uber, LinkedIn, Spotify, eBay, Dropbox, Lyft

---

## Big Data Platform Evolution

### Uber — Four Generations of Ingestion (100+ PB, 10K vCores)

**Scale:** 100K Presto queries/day, 10K Spark jobs/day, 20K Hive queries/day, 100K concurrent batch jobs.

#### The Small Files Problem

HDFS NameNode becomes the bottleneck past 50–100 PB due to namespace pressure from small files:

```
Gen 2 (snapshot): 1,000+ Spark executors/job to recreate full tables
                  20+ hours per ingestion cycle for 100+ TB tables → 100 GB delta
                  Multiple near-identical copies of data → storage blowout

Gen 3 (Marmaray + Apache Hudi):
  Mini-batch every 10–15 min
  30-min raw data latency
  ETL wall-time: 20+ hours → < 30 min
  Hudi Merge-On-Read eliminates copy-on-write overhead

Gen 4 (target): 5-min raw latency, 10-min modeled latency, >1 GB target file size
```

**Parquet adoption:** JSON → Parquet columnar format reduces I/O significantly. Target file size 128 MB (Gen 3) → >1 GB (Gen 4) to eliminate NameNode pressure.

**Hudi Merge-On-Read pattern:** Write deltas to log files; merge-on-read combines base Parquet + delta logs at query time. Avoids rewriting entire partitions per upsert.

---

### LinkedIn HDFS at Exabyte Scale

**Scale:** 1 exabyte total, largest cluster 500 PB / 10,000 nodes, 1.1 billion namespace objects.

#### NameNode JVM Tuning

```
JVM heap for 1.1B objects: 380 GB
Young:Tenured heap ratio: 1:4
Workload: 95% reads, 5% writes

Non-fair JVM locks vs fair mode: ~10x throughput improvement in benchmarks
```

#### Observer NameNode Pattern (3x Throughput)

```
Primary NameNode: handles writes + metadata changes
Observer NameNode(s): serve read-only ops (getFileInfo, getListing, getBlockLocations)
Journal Nodes: replicate edits (3 journals for HA)

Results:
  Avg throughput:  150K ops/sec
  Peak throughput: 250K ops/sec
  Primary RPC latency: <10 ms
  Observer read latency: 1–2 ms
  Journal tailing lag: 2–8 minutes (legacy) → <6 ms (Fast Path)
  Fast Path end-to-end: 25–30 ms
```

#### Small Files: Satellite Cluster

```
Problem: system directories = 50% of all files, 0.07% of cluster capacity
Initial block-to-file ratio: 1.1  (90% of files were small)
After satellite cluster:     1.4

Satellite cluster: 32 DataNodes (vs thousands in primary)
  - 100 million files migrated, 60 TB total
  - Per-node density: 9M blocks/node (satellite) vs 200K blocks/node (primary)
  - Virtual volume partitioning: each physical drive split into 10 virtual volumes
```

#### Historical Growth (2015–2020)

| Metric | 2015 | 2020 | Growth |
|---|---|---|---|
| Storage | 20 PB | 500 PB | 25× |
| Namespace objects | 145M | 1.6B | 11× |

#### Selective Wire Encryption Impact

Adding TLS to HDFS data transfers (WebHDFS over HTTPS):
```
Read/write latency reduction:     36–46%
Read/write throughput improvement: 56–85%
```
Port-based selective encryption allows hot paths to skip TLS while securing bulk transfers.

---

### LinkedIn YARN at 10,000+ Nodes

**Cluster:** ~10K nodes primary cluster, target sub-cluster ~5K nodes, 2× YoY growth.

#### Scheduler Throughput (Container Allocations/sec)

```
Pre-merge:            500 containers/sec (primary), 250 containers/sec (secondary)
Post-merge baseline:  ~600 containers/sec average
Post-merge worst-case: 50 containers/sec for multi-hour periods
Target SLA:           p95 application delay ≤ 10 minutes
```

**Root cause of worst-case:** DNS synchronization contention between merged partitions.

**Fixes:**
- DNS synchronization fix: ~10% throughput improvement
- Partition-aware scheduling: **9× improvement in worst-case scenarios**

#### DynoYARN Simulation — Capacity Cliff

| Scale multiplier | Node Managers | Apps/Day | p95 Delay |
|---|---|---|---|
| 1.0× | 7,152 | 237,472 | 4.6 min |
| 1.6× | 11,443 | 377,962 | 10.3 min ← SLA limit |
| 1.8× | 12,873 | 424,540 | **22.8 min** ← cliff |

**The SLA cliff is steep:** 1.8× load doubles the p95 delay. Plan cluster expansion well before 1.6× multiplier.

#### Robin Multi-Cluster Federation

```
Stateless REST load balancer routes jobs before submission to YARN RM
Best scheduling policy: Most Absolute DRF (Dominant Resource Fairness) across sub-clusters
Data locality: 50% locality loss after federation → mitigated by rack-striping allocation
Self-healing: runs on Kubernetes, independent of YARN RM failures
```

---

### Spotify — Scio on Google Dataflow

**Scale:** 80K Dataflow jobs/month, 1,300+ deployed pipelines, 200+ internal users.

**Largest single batch job:**
```
800 × n1-highmem-32 workers = 25,600 CPUs, 166.4 TB RAM
Input: 325 billion rows from 240 TB Bigtable
```

**Join optimization:**
- Side-input hash joins for small right-hand sides (broadcast join, avoids shuffle)
- Skewed and sparse-outer join strategies for extreme data imbalance

**Serialization:** Standard Java coders replaced by Scala reflection + Chill (Twitter's Kryo-based library) — eliminates explicit coder configuration overhead.

**Orchestration migration (Luigi/Flo → Flyte):**

| Pain point | Root cause |
|---|---|
| Wasted GKE compute | Dependency resolution inside launched containers (poll at runtime) |
| No pre-execution visibility | Workflows opaque until execution completes |
| Fragmented ecosystem | Python Luigi + Java Flo — dual versioning, users stuck on old versions |
| No early failure detection | No pre-execution dependency validation |

Flyte: orchestration logic in managed backend (thin SDK), static workflow definition enables pre-execution lineage + validation.

---

### eBay — The Accelerator: 1B rows/sec on a Single Machine

**Hardware:** 72-core machine with fast disk.
**Dataset:** 1.1 TB raw (280 GB gzip-compressed), 6.3 billion rows, 14 columns.

**Throughput numbers:**

| Operation | Throughput |
|---|---|
| CSV import (gzip) | 182 MB/s, 1.0M rows/s |
| Type conversion | 560 MB/s, 3.3M rows/s |
| Hashing (average) | 230 MB/s, 1.3M rows/s |
| Read 3 cols, write 1 col | 77M rows/s |
| Sum aggregation | **>1,000M rows/s** |

Full column sum over 6.3 billion rows: **~6 seconds in Python**.

**Architecture decisions:**
- **Columnar row-column format** — each column stored independently, entropy-coded columns separate
- **Sequential disk reads (streaming-first)** — no random access, maximizes HDD/SSD sequential bandwidth
- **Min/max values auto-cached per column** — range query pruning without full scan
- **Dataset chaining is lightweight** — no data copying between pipeline stages
- **Job checksumming** — identical parameters skip recomputation (memoization)
- Job startup: **fraction of a second** vs JVM/Spark overhead of tens of seconds

---

## Distributed ML Training

### Uber Horovod — Ring-AllReduce vs Parameter Server

**Problem with parameter servers at scale (128 GPUs):**
```
Standard distributed TensorFlow: ~50% GPU utilization
Cause: O(N) communication fan-in to dedicated aggregation servers → bottleneck
```

**Ring-AllReduce (Horovod):**
```
Each of N nodes communicates with exactly 2 peers
2×(N-1) communication rounds — no dedicated server bottleneck
Launch: mpirun -np 16 -H server1:4,server2:4,server3:4,server4:4 python train.py
         ↑ 4 nodes × 4 GPUs = 16 total GPUs
```

**Results at 128 GPUs (Inception V3, ResNet-101):**
```
GPU utilization: 50% → 88% scaling efficiency
Training speed: ~2× faster than standard distributed TF
```

**Tensor fusion — mandatory optimization:**
```
Small tensors batched before ring-allreduce
Throughput improvement: up to 65% on models with many layers
(especially models with unoptimized TCP stacks)
```

**RDMA vs TCP:**
```
Most models:  +3–4% improvement with RDMA (marginal)
VGG-16:       +30% speedup (communication-bottlenecked architecture)
Rule: RDMA only pays for communication-bound models
```

---

## ML-Driven Cost Optimization (Dropbox Cannes)

**Problem:** Pre-warming previews for all uploaded files wastes compute — most files are never previewed.

**Solution:** Gradient-boosted classifier predicts whether a file will be previewed within 60 days.

```
Infrastructure cost before: $1.7M/year (pre-warm fleet)
ML system operating cost:    $9K/year (Suggest Backend + Predict Service)
ROI: ~189×

Scale: ~500M requests/day
Rejection rate: ~40% of pre-warm requests (avoided unnecessary work)
Model accuracy: >70% on holdout
```

**Features:** file extension, account type, 30-day account activity (no heavy signal).

**Serving pipeline:**
```
Riviera pre-warm request
    → Suggest Backend (live signals from Edgestore metadata + User Profile Service)
    → Predict Service (feature encoding + GBM inference)
    → pre-warm decision (go / skip)
```

**Online feature store:** User Profile Service backed by **RocksDB** — aggregated behavioral signals, sub-millisecond p99 reads.

**Deployment:**
```
Gradual rollout: 1% (via Stormcrow feature flag) → 25% → 100%
3% holdout maintained permanently for continuous A/B baseline
Hourly model metrics in Hive; Superset dashboards with drift alerts
```

**Pattern applicable to any pre-computation problem:**
> Train a lightweight classifier to predict whether expensive downstream work (pre-warming, rendering, indexing, prefetch) will actually be consumed. At 500M req/day, even 40% rejection rate is transformative.

---

## Inference Throughput: Disable TF Multicore (Dropbox OCR)

**Scale:** 20+ billion image and PDF files, 10–20% are document photos.

**Throughput optimizations measured:**

| Optimization | Gain |
|---|---|
| Disable TensorFlow multicore | **3× throughput** |
| DenseNet-121 vs Inception-ResNet-v2 | **2× speedup**, negligible accuracy loss |
| AVX2 vectorization + XLA compilation | Additional significant gain |
| Exponential backoff retry | **88% reduction in PDF metadata extraction failures** |

**Why multicore hurts inference throughput:**
> Thread contention on inference workloads. Single-threaded-per-core process model (one process per CPU core, each running single-threaded TF) maximizes throughput by eliminating synchronization overhead.

```python
# Disable TF multicore explicitly
import tensorflow as tf
config = tf.ConfigProto(
    inter_op_parallelism_threads=1,
    intra_op_parallelism_threads=1
)
```

**Pipeline architecture:**
- Async event-stream via Cape framework
- PDF rendering: PDFium (Chromium) at 2048×2048px per page
- Security: software jails, one per CPU core, single-threaded per jail
- Processing capped at 10 pages/document (covers ~90% of PDFs)

---

## Vector Similarity Search at Billion Scale (eBay)

**Scale:** 1.7 billion live listings, 134 million users, 13 production indices.

### Algorithm Comparison: HNSW vs ScaNN

| Property | HNSW (HNSWLib) | ScaNN (Google) |
|---|---|---|
| Algorithm | Graph-based (proximity graph) | Product quantization + partitioning |
| Index build time | Slower | Faster for most index sizes |
| Recall on high-dim vectors | Better | Degrades faster under filtering |
| Dynamic updates | Requires periodic maintenance | Max size set at build time |
| Memory on update | Proportional to M (edges/node) | Expensive reallocation if over max |
| Runtime accuracy tradeoff | `efSearch` at query time | Partition-based |

**Key parameters:**
```
HNSW:
  M = edges per node (higher M = better recall, more memory)
  efConstruction = search width during build (accuracy vs build time)
  efSearch = search width at query time (tunable per query)

ScaNN:
  max_index_size: must be set before build to avoid reallocation
  Partitioning: ANN vs brute-force tradeoff based on partition count
```

### Production Numbers

```
Corpus:        1.7 billion vectors
Indices:       13 production indices
Latency p95:   <25ms end-to-end
k=200 lookup:  12ms at p95
QPS:           thousands to tens of thousands
Revenue:       "millions of dollars per year" across 13 indices
```

**8-bit quantization:** 4× memory reduction vs 32-bit full precision. Memory and build time scale linearly with vector dimensionality.

### Post-Filtering Failure Mode

> Multi-condition post-filtering on ANN results can return **zero results** at high filter selectivity.

**Solutions:**
1. Pre-filter candidate set before ANN search (reduce corpus first)
2. Use filter-aware index structures that respect constraints during graph traversal
3. Increase k (retrieve more candidates) and post-filter — degrades recall predictably

### Infrastructure: Sharding + Replication

```
Each production index: sharded across query nodes + replicated for fault tolerance
Sharding by ID range: each node owns a slice of the 1.7B vector corpus
Query fan-out: coordinator distributes query → merges top-k from each shard
```

---

## Data Discovery & Metadata at Scale (Uber Databook)

**Scale:** hundreds of PB across HDFS, trillions of daily Kafka messages, peak search 1,500 QPS.

**Why Cassandra over MySQL:**
```
MySQL single-master: ~70ms latency per cross-datacenter write + hours of manual failover
Cassandra XDC: no latency penalty for multi-datacenter writes + automatic failover
→ Cassandra chosen for metadata storage
```

**Architecture:**
```
Collection layer:
  Quartz-scheduled crawlers (Hive, Vertica, MySQL, Postgres, Cassandra sources)
  Kafka event-based pipeline (near-real-time lineage/freshness updates)
  Distributed execution: Cherami task queue on Docker containers

Storage layer: Cassandra (XDC replication)

Serving layer:
  Dropwizard REST APIs
  Elasticsearch (multi-dimensional search: name, owner, column, nested column)
  React UI
```

**Read-time linking pattern:**
> Cluster-agnostic metadata (descriptions, ownership) linked to cluster-specific metadata (latency, usage stats) at read time — avoids timing race conditions from write-time approaches.

---

## See Also

- [Real-Time Analytics Architectures](24-realtime-analytics-architectures.md) — AresDB, AthenaX, Pinot, Lambda vs Kappa
- [Kafka & Messaging](22-kafka-messaging.md) — Kafka production ops, Flink state backends
- [Database Scaling](23-database-scaling.md) — Presto/Trino, Alluxio cache, sharding
- [Storage Engine Patterns](19-storage-engine-patterns.md) — RocksDB, LSM-tree, columnar formats
