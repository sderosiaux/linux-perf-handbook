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

---

## SpinalTap — CDC at Airbnb

**Guarantees:** at-least-once, zero data loss, sub-second propagation, per-row commit-order enforcement.

### Architecture: 3-Layer Abstraction

```
Source: reads DB changelog (MySQL binlog, DynamoDB streams)
        → filters/transforms to typed Mutation objects (insert/update/delete,
          before/after values, transaction info, schema)

Pipe:   unit of parallelism; coordinates source↔destination,
        periodic checkpointing, lifecycle management, keep-alive/auto-recovery

Destination: publishes to Kafka (Thrift wire format);
             tracks last-published mutation for checkpoint derivation
```

**Cluster management:** Apache Helix — deterministic load balancing, horizontal scaling, Leader-Standby state model per source.

**Split-brain mitigation:** global leader epoch per source, atomically incremented on leader transition. Clients filter mutations with stale epochs → discarded.

**Throughput optimizations:**
```
Buffered Destination: in-memory bounded queue decouples source streaming from publish latency
Destination Pool:     application-level partitioning across thread pool of buffered destinations
                      handles spiky traffic without sacrificing ordering or consistency
(Kafka strong consistency settings created a publish bottleneck — both patterns resolve it)
```

**Use cases:** cache invalidation (Memcached/Redis), real-time Elasticsearch indexing (review/inbox/support search), Hive/HBase snapshot pipeline (daily backup → hourly granularity), inter-service signaling (Availability Service subscribes to Reservation Service changes).

---

## Spark Partitioning Strategies (Airbnb)

**Incident:** single job wrote 1.1 million files to HDFS → NameNode outage.
- Root cause: 2000–3000 Spark partitions × 365 Hive date partitions = up to 1.1M files
- HDFS cost: 150 bytes NameNode memory per file

### Strategy Comparison

| Strategy | When to use | Pitfall |
|---|---|---|
| `coalesce(N)` | Single hPartition, can cache | Pushdown to early stage unless cached |
| `repartition(N)` | Single hPartition, round-robin | Useless for multi-hPartition |
| `repartition(N, col)` | Multiple small hPartitions | HashPartitioner → 1 file/hPartition always |
| `repartition(N, col, rand%k)` | Equal-size known hPartitions | Birthday Problem: 5 files×365 partitions → 63% executor utilization |
| **`repartitionByRange(N, hash(cols), rand())`** | **Default for multi-hPartition** | Requires cache; driver needs 4–6 GB extra |

**Recommended pattern:**
```scala
// Compute synthetic sort keys (infinite hash space → no Birthday Problem collisions)
df.withColumn("_hash", hash(partitionColumns))
  .withColumn("_rand", rand())
  .repartitionByRange(fileCount, col("_hash"), col("_rand"))
  .drop("_hash", "_rand")
```

File count upper bound: `fileCount + count(distinct hash)`.

**File size heuristic:** target multiple of HDFS block size (128 MB). Use row-count estimation (count ÷ target rows/file) — compressed on-disk size ≠ in-heap size.

---

## TensorFlowOnSpark — Distributed TF on Spark/Hadoop Clusters (Yahoo)

**Key design:** direct process-to-process tensor communication — unlike SparkNet/TensorFrame which route through the Spark driver (bottleneck).

```
Roles: TF workers + TF parameter servers each occupy one Spark executor
Data: TensorFlow QueueRunners reading directly from HDFS (Spark = launcher only)
     OR Spark RDD piped via feed_dict into TF computation graph
Migration cost: fewer than 10 lines of Python changes for existing TF programs
```

**Scaling (Inception V3 to 0.730 top-5 accuracy):**

| Workers | Time | Speedup |
|---|---|---|
| 1 | 46 hours | 1× |
| 2 | 22.5 hours | 2.0× |
| 4 | 13 hours | 3.5× |
| 8 | 7.5 hours | **6.1× (near-linear)** |

**RDMA optimization:**
```
Custom gRPC_RDMA protocol for Infiniband clusters
Tensors allocated once at job start, reused across steps (zero-copy)
VGG-19: significant speedup over gRPC (direct memory access bypasses Ethernet ceiling)
```

---

## Netflix Media ML Platform

**Problem:** multiple teams independently computing identical embeddings against the same assets → wasted compute, inconsistent results.

### Amber Feature Store

```
Memoized feature store tied to media entities (videos, artwork)
Each algorithm = "Amber Feature" with own scope for computation, storage, triggering
Amber Features compose via dependency semantics → complex DAG of interrelated algorithms
Recursive dependency resolution: algorithm pipelines triggered automatically when upstream changes
Immutability + versioning + auditing + metrics on feature values
Amber Compute: data replication to different backends per access pattern
```

**Match Cutting case study (cross-title video similarity):**
```
Single title:  ~2K shots → ~2M pair comparisons
Series (10ep): 200M comparisons
Cross-1000 files: ~200 trillion computations (naive pre-compute: 2^1000 — impossible)

Solution: decompose into independent Amber Features
  - Shot segmentation: computed once, memoized, shared as canonical dependency
  - Embeddings: tied to shot deduplication, auto-triggered on standardized encode
  - Pair computation: on-demand over pre-stored embeddings (avoids 2^N pre-computation)
```

### Training Performance

```
GPU cluster: Ray (multi-GPU, multi-node)
Data loading: FSx (high-performance NFS)
Preprocessing: offloaded to CPU instances (separate from GPU workers)
Result: 3–5x increase in overall training system throughput
```

**Marken (serving + vector search):** annotations (versioned, strongly typed, associated with media entities). Supports temporal/spatial queries and vector search at catalog scale.

---

## Lyft ML Infrastructure

### Feature Store

**Scale:** several 1,000s of features, millions of requests/minute, single-digit ms latency, 99.99%+ availability.

```
Definition language: SQL (one column = entity ID; rest = features)
Feature versions: per-feature versioning (not group-level) for faster iteration

Ingestion:
  Batch:     Flyte-scheduled SQL jobs against Hive warehouse
  Streaming: custom Flink jobs with SQL-over-stream-windows

Write path: validate → DynamoDB (latest value per feature per entity)
            → replicate to Hive + Elasticsearch
Read path:  Redis cache → DynamoDB on miss (no locking on read path)
            Bulk batch-get: Redis → DynamoDB fallback
Training:   replicated Hive tables queried directly (point-in-time reads, zero impact on online latency)
```

**Online-offline parity:** same SQL definition used for training and serving — same validations, same transformations. Eliminates training-serving skew.

**Failure modes:**
- Optimistic locking via DynamoDB conditional checks prevents stale writes from concurrent callers
- Feature metadata includes per-feature alerting config (owning team paged on generation failure)
- Hive replication is eventually consistent — accepted inconsistency window for ML workloads

### Flyte — Cloud-Native ML Platform

**Scale:** 7,000+ unique workflows, 100,000+ executions/month, 1 million tasks/month, 10 million containers/month.

```
Execution model: DAG-based, Kubernetes-native
Task granularity: every task = its own container image (heterogeneous steps first-class)
Strong typing on all task inputs/outputs → enables data lineage, parameterization, caching
Every entity is immutable; changes = new explicit version (enables rollback + reproducibility)

Task-level caching: if identical strongly-typed inputs already computed → output reused
                    Eliminates redundant data prep across hyperparameter sweeps
Backend plugins: extend task types via K8s CRDs (Spark-on-k8s, SageMaker, Qubole, BigQuery)
Multi-tenant: each team has isolated repo, deploys independently
```

### LyftLearn — ML Training on Kubernetes

**Scale:** hundreds of models/week, dozens of teams.

```
Notebook spin-up: seconds (K8s pod scheduling + image pull optimized)
Auto-terminate idle notebooks: few hours (fast spin-up = no UX penalty)
Training jobs: Kubernetes Jobs with configurable hardware (CPU cores, memory, GPU count)
Parallelization: multiple jobs with different hyperparameters / configs run concurrently
Container-per-model-version: diverse library versions (sklearn, XGBoost, PyTorch, TF) coexist
```

**Architecture:**
```
User selects: hardware config + base image → K8s pod spins up in seconds
"Save Model": snapshots code + deps as new Docker layer → ECR
Storage: AWS EFS mounted as K8s volume (survives ephemeral container loss)
Metadata: AWS Aurora RDS (ownership, run history, metrics)
Training data: Hive, Presto, or Spark queries
```

**Layered API principle:** REST API → CLI → GUI, each built on the layer below — programmatic access is always the canonical interface.

### Amundsen — Data Discovery at Lyft

**Result:** time to discover a data artifact reduced to **5% of baseline** (20x improvement). Data grew 40x over ~10 years.

```
Search: Google-like full-text search (table name, column name, descriptions)
Ranking: PageRank-style from query audit logs (highly queried = higher rank) — auto-generated, no human curation
Column stats: count, null, zero, min, max, avg — computed over last day's partition
Graph DB backend: people + table nodes, usage edges → PageRank popularity scoring
Security model: existence + fundamental metadata visible to all; richer stats gated on data permissions
  ("security by obscurity is wrong — fix the data model instead of hiding existence")
```

**GDPR use case:** tag PII fields in metadata graph → auto-fulfill data-subject requests.

---

## Uber AI/ML Scaling (Michelangelo)

**Infrastructure:** Michelangelo Job Controller — unified federation layer over multiple Kubernetes clusters across AZs/regions; abstracts cluster topology from ML engineers.

```
Workload routing by SKU: A10 GPUs → serving (lower latency); H100/A100 → training
Tiered memory: GPU VRAM → CPU system memory → NVMe offload (LLM training)
Full-mesh NVLink within nodes; inter-node network: 25 GB/s → 100 GB/s (4x upgrade)
QoS/isolation: dedicated rack + network topologies (training elephant flows isolated from serving)
```

**Concrete improvements:**
```
Network upgrade:          2x training throughput
Memory offloading:        2x MFU (Model Flops Utilization), 34% GPU usage reduction
H100 vs A100:             4x TFlops, 2x memory bandwidth
LLM serving framework B vs TRT-LLM on H100: 2x latency, 6x throughput
Reactive scheduling:      opportunistic training during serving off-peak windows
DIMM upgrade (16→32 GB):  unlocked GPU allocation rates on legacy racks
```

**GPU SKU selection** (17 variants benchmarked): tree-based models → small CPU instances; DL → A10/A100; LLM training → H100 + NVMe offload.

---

## Distributed ML Pipelines

### Gradient Boosted Trees at Scale (Yelp — CTR Prediction)

**Infrastructure:** 50 machines, 36 cores + 60 GB RAM each, 5 XGBoost workers/node, AWS EMR + YARN.

```
Dataset: hundreds of millions of training samples
Training time: was ~2 days for 1/10 dataset → now < 3 hours at full scale
Accuracy: 4% MXE improvement on test set
Cost: 1/3 of original
```

**Critical YARN config:**
```
DominantResourceCalculator (CPU+memory) instead of DefaultResourceCalculator (memory-only)
→ accounts for CPU in allocation — mandatory for CPU-bound XGBoost workloads
Without it: memory allocation succeeds but cores are starved; workers underutilize
```

**XGBoost tuning:**
```
method: "approx" (bins continuous features → faster tree construction)
max_delta_step: prevents gradient explosion on imbalanced click data
early_stopping: balances accuracy vs. scoring latency
Incremental retraining: "refresh" updater for cheap updates on new data
Checkpointing: automated retry on failure
Network saturation: horizontal scaling has diminishing returns past cluster optimum — identify empirically
```

### ML for Payments Retry Optimization (Dropbox)

**Problem:** monolithic payment platform embedded prediction logic — 2-minute average p99 per prediction.

**Solution:** extracted dedicated gRPC Predict Service.

```
Before: ~2 minutes average p99
After:  < 300ms p99 (400x improvement)

Model: gradient boosted ranking model
Input: usage patterns, payment history, failure details
Output: ranked 1-hour time chunks over 4/6/8-day retry window (192 chunks for 8-day)
Feature store: Edgestore (KV store) populated daily by Airflow jobs
```

**Key lesson:** extracting ML serving into a dedicated service decouples deployment cycles, enables independent scaling, and eliminates latency from monolith dependency chains.

### ML Workflows (Twitter)

**Platform:** Apache Airflow on Apache Aurora, Celery with MySQL as message queue (no separate Redis/RabbitMQ).

```
Timelines Quality team: model retraining 4 weeks → 1 week → measurable improvement in ranking quality
Custom operators: Aurora, DeepBird (train/serve/load-test), hyperparameter tuning, HDFS/CI utilities
XCom type checking: validates data types between operators at DAG definition time (not runtime)
Hyperparameter tuning: DAG-of-subdags pattern, random search + grid search; Bayesian planned
```

**Stateless container survival:** DAG instances persisted to MySQL → auto-recreated on pod restart.

### Privacy-Preserving Analytics (LinkedIn PriPeARL)

**Architecture:** Kafka events → batch/intra-day pipeline → Pinot → Privacy Mechanism → noisy response.

**Noise approach: deterministic pseudorandom (hash-based), not true DP randomness:**
```
Prevents averaging attacks while maintaining latency SLA
Hash function + double (vs BigDecimal) floating-point: "reduced latency significantly"
Hierarchical time-range partitioning: atomic boundaries bound member event frequency across query ranges
Post-processing: negative noisy counts capped at zero; high-relative-noise results suppressed
```

---

## Flickr Similarity Search — LOPQ at 1 Billion Photos

**Algorithm:** LOPQ (Locally Optimized Product Quantization) for ANN at billion scale.

```
Feature: deep neural network scene recognition; internal pre-softmax vector used
Compression: 8 bytes/image (vs ~1 TB raw for 256-dim float vectors at 1B scale)
Partitioning: k=1000 × k=1000 = 1 million partitions (1000x fewer distance computations)
```

**Multi-index construction:**
```
Split feature vector in 2 halves → cluster each independently (k=1000 per half)
→ k² = 1 million partitions (multi-index)
Residual quantization on approximation errors → compressed code storage
Local PCA rotation per cluster: fits local data distribution, reduces quantization error
Variance balancing: PCA dimension permutation distributes information evenly across splits
```

**Performance:**
- 1000x reduction in distance computations vs. brute-force
- Distance caching: precomputed squared differences per quantization split (avoid per-query arithmetic)
- Product quantization independence assumption violated in practice → mitigated by per-cluster PCA rotations

---

## Experimentation Platform (Spotify)

**Architecture:** domain-based bucket pool — experiments consume slices, released buckets unusable until carryover period expires.

```
Assignment: HASH(user_id, SALT) % bucket_count → each experiment gets its own salt
Carryover prevention: new salts on release shuffle users into new buckets
Holdback groups: quarterly exemptions from all experiments → measure cumulative treatment effect
Compensation factor: 1 / traffic_fraction (e.g. 50% traffic → 2x overallocation needed)
Hard stop: compensation factor > 5 → stop starting new experiments (too many wasted users)
```

**Validity checks:**
- Sample ratio mismatch (actual vs. targeted exposure)
- Pre-exposure activity parity between groups
- Client crash monitoring
- Remote config property collision warnings

**Statistical modes:** sequential (real-time with multiple comparison corrections) or fixed-horizon (post-experiment). Superiority tests for success metrics; non-inferiority tests for guardrails.

**Weekday bias:** gradual ramp-up assignment over time to smooth day-of-week effects.

---

## See Also

- [Real-Time Analytics Architectures](24-realtime-analytics-architectures.md) — AresDB, AthenaX, Pinot, Lambda vs Kappa
- [Kafka & Messaging](22-kafka-messaging.md) — Kafka production ops, Flink state backends
- [Database Scaling](23-database-scaling.md) — Presto/Trino, Alluxio cache, sharding
- [Storage Engine Patterns](19-storage-engine-patterns.md) — RocksDB, LSM-tree, columnar formats
