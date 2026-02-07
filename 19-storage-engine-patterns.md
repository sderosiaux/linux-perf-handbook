# Storage Engine Patterns for Performance Analysis

Reference for diagnosing, tuning, and optimizing storage engines. Actionable patterns for LLM-assisted performance work.

---

## 1. Pattern Classification

Storage engines fall into distinct architectural patterns, each with specific performance characteristics and optimization vectors.

| Pattern | Data Structure | Optimal For | Watch For |
|---------|----------------|-------------|-----------|
| Memory-Mapped B+Tree | B+Tree + mmap | Read-heavy, ACID | Map size, CoW overhead |
| LSM Tree | Memtable + SSTables | Write-heavy | Write amplification, compaction |
| Log-Structured | Append-only + sparse index | Time-series, streams | Segment sizing, retention |
| Columnar | Column chunks + compression | OLAP, scans | Row group sizing, encoding |
| Vectorized In-Memory | Arrow-compatible batches | Analytics, IPC | Batch size, SIMD alignment |

---

## 2. Memory-Mapped + Copy-on-Write Engines

**Pattern**: Kernel manages memory via mmap; B+Tree with CoW pages.

### Engines

| Engine | Language | Notes |
|--------|----------|-------|
| LMDB | C | Single writer, zero-copy reads |
| BoltDB/bbolt | Go | etcd storage, single writer |
| MDBX | C | LMDB fork, more features |

### Key Metrics

```bash
# Page fault rate (indicates mmap pressure)
awk '/pgfault|pgmajfault/ {print $1, $2}' /proc/vmstat

# Per-process page faults
ps -o pid,min_flt,maj_flt,cmd -p <PID>

# Memory map size
cat /proc/<PID>/maps | grep -E "\.mdb|\.db" | awk '{print $1}'

# Check if DB file in page cache
vmtouch <db_file>  # Shows resident pages
```

### LMDB Tuning

**Map Size Configuration**:
```c
// Set max size - LMDB mmaps this much virtual memory
mdb_env_set_mapsize(env, 256ULL * 1024 * 1024 * 1024);  // 256GB
```

Rule: Set map size 5x expected DB size for MVCC headroom. On 64-bit systems, overprovisioning is cheap (virtual memory only).

**Page Size**:
- LMDB uses OS page size (typically 4KB)
- Match filesystem block size for aligned I/O
- Larger pages: fewer TLB misses, more read amplification

**Key Layout for Locality**:
```
# Good: Related keys cluster
user:1001:profile
user:1001:settings
user:1001:activity

# Bad: Random access pattern
profile:user:1001
settings:user:1001
activity:user:1001
```

**MDB_APPEND for Bulk Loads**:
```c
// Keys must be pre-sorted; avoids B+tree rebalancing
mdb_put(txn, dbi, &key, &data, MDB_APPEND);
```

Performance gain: 2-10x for sorted bulk inserts.

### MDBX Improvements Over LMDB

| Feature | Performance Impact |
|---------|-------------------|
| LIFO GC recycling | Up to several times write perf on systems with disk cache |
| Auto size management | No manual resize needed |
| Continuous compaction | Automatic space reclamation |
| Write-before-failure | Multiple reduction in syscalls for large txns |

**MDBX Tuning**:
```c
// Enable write-through for small transactions
mdbx_env_set_option(env, MDBX_opt_writethrough_threshold, 1024);

// Configure geometry (auto-grow)
MDBX_envinfo info;
mdbx_env_set_geometry(env,
    0,           // size_lower
    -1,          // size_now (unchanged)
    256ULL<<30,  // size_upper (256GB)
    64<<20,      // growth_step (64MB)
    16<<20,      // shrink_threshold
    -1);         // pagesize (unchanged)
```

### BoltDB/bbolt Tuning

```go
db, _ := bolt.Open("my.db", 0600, &bolt.Options{
    // Disable fsync for bulk loads (data loss risk on crash)
    NoSync: true,

    // Skip truncate on non-ext3/ext4 (faster grow)
    NoGrowSync: true,

    // Pre-populate page cache for sequential scans
    MmapFlags: syscall.MAP_POPULATE,
})
```

**Sharding for Large DBs**:
```go
// Hash key to shard (N shards)
shardID := hash(key) % N
db := shards[shardID]
```

Benefit: Export/scan time goes from O(n²) to O(n/N).

### Diagnosing mmap Issues

```bash
# High major faults = DB larger than RAM
perf stat -e major-faults -p <PID> -- sleep 10

# TLB pressure from large mmap
perf stat -e dTLB-load-misses -p <PID> -- sleep 10

# Hot page detection
perf mem record -p <PID> -- sleep 30
perf mem report --sort=mem
```

**Decision Tree: mmap Performance**

```
Major faults > 100/sec?
├─ YES → DB exceeds RAM
│   ├─ Can add RAM? → Add RAM
│   └─ No → Consider LSM engine (RocksDB) or shard
└─ NO → Check TLB misses
    ├─ > 5% miss rate → Enable huge pages
    └─ < 5% → mmap performing well
```

---

## 3. LSM Tree (Write-Optimized) Engines

**Pattern**: Memtable → SSTable flush → Background compaction.

### Engines

| Engine | Language | Notes |
|--------|----------|-------|
| RocksDB | C++ | Facebook, most configurable |
| LevelDB | C++ | Google, original LSM |
| Pebble | Go | CockroachDB, RocksDB-compatible |
| BadgerDB | Go | WiscKey (separate value log) |

### The Three Amplifications

| Type | Definition | Trade-off |
|------|------------|-----------|
| Write Amplification | Bytes written to disk / bytes written by app | SSD wear, throughput |
| Read Amplification | Disk reads per logical read | Latency |
| Space Amplification | Disk size / logical data size | Storage cost |

**Measuring Write Amplification**:
```bash
# RocksDB statistics
rocksdb_ldb --db=/path stats | grep "compaction.bytes"

# Or via iostat
# Write amp ≈ disk_write_rate / app_write_rate
iostat -x 1 | awk '/sda/ {print $4}'  # wMB/s
```

Typical values:
- Leveled compaction: 10-30x
- Universal compaction: 2-5x

### Compaction Strategies

**Leveled Compaction** (default):
```
L0: [SST][SST][SST]     # May overlap
L1: [SST][SST][SST]     # Non-overlapping, 10x L0
L2: [SST][SST]...[SST]  # Non-overlapping, 10x L1
...
```

Best for: Read-heavy, space-constrained, SSD.

**Universal/Tiered Compaction**:
```
[Tier1: recent SSTables]
[Tier2: older SSTables, compacted together]
[Tier3: oldest, largest]
```

Best for: Write-heavy, throughput-focused, when 2x space overhead acceptable.

### RocksDB Tuning

**Quick Start Configuration** (SSD, general workload):
```cpp
Options options;

// Memtable
options.write_buffer_size = 64 << 20;           // 64MB per memtable
options.max_write_buffer_number = 3;            // 3 concurrent memtables
options.min_write_buffer_number_to_merge = 1;

// Block cache
BlockBasedTableOptions table_options;
table_options.block_cache = NewLRUCache(512 << 20);  // 512MB cache
table_options.block_size = 4096;                      // 4KB blocks

// Compression (none at L0-L1, LZ4 at L2+, Zstd at bottom)
options.compression_per_level = {
    kNoCompression, kNoCompression,
    kLZ4Compression, kLZ4Compression,
    kZSTD, kZSTD, kZSTD
};

// Parallelism
options.max_background_compactions = 4;
options.max_background_flushes = 2;

// Bloom filters
table_options.filter_policy.reset(NewBloomFilterPolicy(10));  // 10 bits/key = ~1% FP
```

**Bloom Filter Sizing**:

| bits/key | False Positive Rate | Memory |
|----------|---------------------|--------|
| 6 | ~5.7% | Low |
| 10 | ~1% (default) | Moderate |
| 16 | ~0.09% | Higher |
| 24 | ~0.006% | High |

For point lookups, 10-16 bits/key is typical. Increase if FP causing excessive disk reads.

**Ribbon Filters** (RocksDB 6.24+):
```cpp
// 30% space savings vs Bloom, 3-4x CPU
options.filter_policy.reset(NewRibbonFilterPolicy(10, 3));
// Use Bloom for L0-L2, Ribbon for deeper levels
```

**Compaction Tuning**:
```cpp
// Leveled compaction trigger
options.level0_file_num_compaction_trigger = 8;

// L1 size (match L0 to avoid bottleneck)
options.max_bytes_for_level_base = 512 << 20;  // 512MB

// Level multiplier
options.max_bytes_for_level_multiplier = 10;

// Target file size
options.target_file_size_base = 64 << 20;  // 64MB
```

### Pebble (Go) Specifics

```go
opts := &pebble.Options{
    // Faster commit pipeline than RocksDB
    MaxConcurrentCompactions: func() int { return 4 },

    // Backwards skiplist links for faster reverse iteration
    Comparer: pebble.DefaultComparer,

    // Memtable
    MemTableSize: 64 << 20,
    MemTableStopWritesThreshold: 4,
}
```

### BadgerDB (Value Log Pattern)

BadgerDB separates keys (LSM) from values (append-only log).

```go
opts := badger.DefaultOptions("/path/to/db")

// Value threshold: values > this go to value log
opts.ValueThreshold = 1024  // 1KB

// Value log file size
opts.ValueLogFileSize = 1 << 30  // 1GB

// Compaction
opts.NumCompactors = 4
opts.NumLevelZeroTables = 5
opts.NumLevelZeroTablesStall = 15
```

**When to use**: Large values (>1KB), write-heavy, can tolerate higher read latency.

### Diagnosing LSM Issues

```bash
# RocksDB internal stats
rocksdb_ldb --db=/path stats

# Key metrics to watch:
# - Compaction pending bytes
# - Write stall count
# - Block cache hit rate
# - Bloom filter useful rate

# Write stall detection via logs
grep -E "stall|compaction" /var/log/rocksdb.log

# Compaction I/O impact
iostat -x 1 | grep -A1 Device
```

**Decision Tree: LSM Performance**

```
Write latency spikes?
├─ YES → Check write stalls
│   ├─ L0 files high → Increase L0 trigger or compaction threads
│   ├─ Memtable full → Increase write_buffer_size
│   └─ Compaction behind → Increase max_background_compactions
└─ NO → Check read performance
    ├─ High read latency → Check bloom filter FP rate, cache hit rate
    └─ Acceptable → LSM healthy
```

---

## 4. Log-Structured + Sparse Index

**Pattern**: Append-only segments with minimal indexing.

### Engines

| Engine | Use Case | Notes |
|--------|----------|-------|
| Kafka | Event streaming | Partition-based, O(1) append |
| ClickHouse MergeTree | OLAP | Sorted, merged |
| Chronicle Queue | Low-latency messaging | mmap-based |

### Kafka Segment Tuning

**Segment Size** (`log.segment.bytes`):
```properties
# Default: 1GB
log.segment.bytes=1073741824

# Smaller segments: finer retention granularity, more file handles
# Larger segments: fewer files, coarser retention
```

Rule: `log.segment.bytes` < `log.retention.bytes / 10` for reasonable granularity.

**Index Granularity** (`log.index.interval.bytes`):
```properties
# Default: 4096 bytes
log.index.interval.bytes=4096

# Lower = more index entries = faster seeks, larger index
# Higher = fewer entries = slower seeks, smaller index
```

With 4096 bytes, a 1GB segment creates ~262K index entries (2MB index).

**Retention Settings**:
```properties
# Time-based (segments deleted after 7 days)
log.retention.hours=168

# Size-based (per-partition)
log.retention.bytes=10737418240  # 10GB

# Segment roll time (force new segment)
log.roll.hours=168  # 7 days
```

**Critical**: Kafka only deletes *closed* segments. If retention is 7 days but segments are 10GB and you write 1GB/day, segments won't close for 10 days.

### ClickHouse MergeTree Optimization

**Primary Key Design**:
```sql
-- Low cardinality first, high cardinality last
CREATE TABLE events (
    event_date Date,          -- Low cardinality (365 values/year)
    event_type String,        -- Medium cardinality
    user_id UInt64,           -- High cardinality
    event_time DateTime,
    payload String
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, event_type, user_id)  -- Key order matters!
SETTINGS index_granularity = 8192;
```

**Why order matters**: With 8192-row granules, ClickHouse can skip entire granules when filtering. Low-cardinality columns first means more granule exclusion.

**Index Granularity**:
```sql
-- Default: 8192 rows per granule
-- Smaller = more index entries, better pruning, more memory
-- Larger = fewer entries, coarser pruning, less memory
SETTINGS index_granularity = 8192
```

**Skipping Indices**:
```sql
-- MinMax (best for slowly-changing columns)
ALTER TABLE events ADD INDEX idx_time (event_time) TYPE minmax GRANULARITY 1;

-- Bloom filter (best for equality predicates)
ALTER TABLE events ADD INDEX idx_user (user_id) TYPE bloom_filter GRANULARITY 1;
```

---

## 5. Columnar On-Disk (OLAP)

**Pattern**: Column chunks + compression + predicate pushdown.

### Formats

| Format | Ecosystem | Notes |
|--------|-----------|-------|
| Parquet | Spark, Hive, DuckDB | De facto standard |
| ORC | Hive, Presto | Better compression |
| Iceberg | Multi-engine | Table format over Parquet |
| Delta Lake | Databricks/Spark | ACID over Parquet |

### Parquet Tuning

**Row Group Size**:
```python
# PyArrow
import pyarrow.parquet as pq

pq.write_table(
    table,
    'data.parquet',
    row_group_size=1_000_000,  # 1M rows or
    # row_group_size=128 << 20  # 128MB
)
```

Rule: 512MB-1GB row groups for analytics. Smaller groups for selective queries.

**Predicate Pushdown Requirements**:
1. Data must be sorted on filter column
2. Row group statistics must exist
3. Query filter must match column type

```python
# Good: sorted data enables min/max pruning
df = df.sort_values('event_date')

# Check statistics
import pyarrow.parquet as pq
meta = pq.read_metadata('data.parquet')
for i in range(meta.num_row_groups):
    print(f"RG {i}: {meta.row_group(i).column(0).statistics}")
```

**Page-Level Statistics** (Parquet 2.0+):
```python
# Enable page-level stats for finer pruning
pq.write_table(
    table,
    'data.parquet',
    write_statistics=True,
    data_page_version='2.0'
)
```

**Dictionary Encoding**:
```python
# Automatic for low-cardinality columns
# Manual control
pq.write_table(
    table,
    'data.parquet',
    use_dictionary=['category', 'status'],  # Explicit columns
    dictionary_pagesize_limit=1 << 20  # 1MB per page
)
```

### Table Format Comparison

| Feature | Delta Lake | Iceberg | Hudi |
|---------|------------|---------|------|
| Write Performance | Fast | Moderate | Fast (upsert) |
| Read Performance | Fast | Fast (partitions) | Moderate |
| Time Travel | Yes | Yes | Yes |
| Schema Evolution | Good | Best | Good |
| Multi-Engine | Spark-centric | Best | Good |

---

## 6. Vectorized In-Memory / IPC

**Pattern**: CPU-friendly columnar batches for SIMD operations.

### Apache Arrow

**Batch Size Selection**:
```python
# Target: L1 cache (32-64KB typical)
# Batch of 1024-2048 rows typical for DuckDB
# Batch of 64KB-1MB for record batches (bounded at 2^16 records)
```

| Batch Size | Trade-off |
|------------|-----------|
| Small (1K rows) | Better cache locality, more overhead |
| Medium (8K-64K rows) | Balanced |
| Large (>64K rows) | Less overhead, may exceed L1 cache |

**SIMD Alignment**:
```cpp
// Arrow automatically aligns to 64-byte boundaries
// Manual buffer allocation for custom code:
auto buffer = arrow::AllocateBuffer(size, arrow::default_memory_pool());
// Buffer is 64-byte aligned for AVX-512
```

**Dictionary Encoding for SIMD**:
```python
# Dictionary-encoded columns enable SIMD predicate evaluation
import pyarrow as pa

# Create dictionary-encoded array
arr = pa.DictionaryArray.from_arrays(
    indices=pa.array([0, 1, 0, 2]),
    dictionary=pa.array(['a', 'b', 'c'])
)
# SIMD can process indices in parallel
```

### DuckDB Optimization

```sql
-- Memory configuration
SET memory_limit='4GB';
SET temp_directory='/fast/ssd/tmp';

-- Thread configuration
SET threads=8;
-- For remote files, increase beyond CPU count
SET threads=32;

-- Compression for in-memory tables
ATTACH ':memory:' AS db (COMPRESS);
```

**Query Profiling**:
```sql
-- View execution plan
EXPLAIN SELECT * FROM t WHERE x > 100;

-- Measure actual execution
EXPLAIN ANALYZE SELECT * FROM t WHERE x > 100;
```

---

## 7. Lock-Free / Single-Writer Designs

**Pattern**: Concurrency through architecture, not locks.

### LMAX Disruptor (Java)

```java
// Ring buffer configuration
Disruptor<Event> disruptor = new Disruptor<>(
    Event::new,
    1024 * 1024,  // Buffer size (power of 2)
    DaemonThreadFactory.INSTANCE,
    ProducerType.SINGLE,  // Single producer = no CAS
    new BusySpinWaitStrategy()  // Lowest latency
);
```

**Wait Strategies**:

| Strategy | Latency | CPU Usage |
|----------|---------|-----------|
| BusySpinWaitStrategy | Lowest | 100% core |
| YieldingWaitStrategy | Low | High |
| SleepingWaitStrategy | Moderate | Low |
| BlockingWaitStrategy | Higher | Lowest |

**Performance**: 6M+ TPS on single thread, 3 orders of magnitude lower latency than queues.

### Seastar/ScyllaDB Shard-per-Core

```cpp
// Each core owns its resources
seastar::sharded<my_service> service;

// Message passing between cores
return seastar::smp::submit_to(target_core, [&] {
    return service.local().process(data);
});
```

**Key Principle**: No shared state between cores. Data is partitioned, not shared.

### Kafka Partition Model

```properties
# Partition count = parallelism unit
num.partitions=12

# Rule: partitions >= max(producer threads, consumer threads)
# Too few: throughput bottleneck
# Too many: overhead, rebalancing time
```

---

## 8. Zero-Copy & Kernel-Bypass

**Pattern**: Avoid memcpy and syscalls.

### sendfile/splice

```c
// Zero-copy file to socket
sendfile(socket_fd, file_fd, &offset, count);

// Zero-copy via pipe
splice(file_fd, NULL, pipe_write, NULL, count, SPLICE_F_MOVE);
splice(pipe_read, NULL, socket_fd, NULL, count, SPLICE_F_MOVE);
```

Performance: 81% speedup with mmap, 91% with sendfile vs read/write.

### io_uring

```c
// Initialize ring
struct io_uring ring;
io_uring_queue_init(256, &ring, 0);  // 256 entries

// Prepare read
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buf, len, offset);

// Submit batch
io_uring_submit(&ring);

// Polling mode (kernel thread polls SQ)
io_uring_queue_init(256, &ring, IORING_SETUP_SQPOLL);
```

**Tuning**:

| Parameter | Default | Tuning Guidance |
|-----------|---------|-----------------|
| Ring entries | 32 | 128-1024 for high throughput |
| IO depth | 1 | 32-128 for NVMe |
| SQPOLL | Off | On for ultra-low latency |

### SPDK/DPDK

```bash
# SPDK setup
./scripts/setup.sh  # Unbind NVMe from kernel

# Reserve huge pages (2MB)
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

# CPU isolation
isolcpus=2,3,4,5  # Add to kernel cmdline
```

**SPDK Performance Tips**:
- Disable hyperthreading for dedicated cores
- Use CPU affinity for polling threads
- 3.37x speedup vs kernel io_uring for NVMe (non-cached reads)

---

## 9. Time-Series & Immutable Blocks

**Pattern**: Time-ordered, append-only, block-based.

### Prometheus TSDB

**Block Configuration**:
```bash
prometheus \
  --storage.tsdb.retention.time=30d \
  --storage.tsdb.retention.size=400GB \
  --storage.tsdb.wal-compression=true \
  --storage.tsdb.max-block-duration=2h
```

| Setting | Default | Impact |
|---------|---------|--------|
| retention.time | 15d | Data lifespan |
| retention.size | 0 (unlimited) | Disk cap |
| wal-compression | true (2.20+) | CPU vs disk |
| max-block-duration | 2h | Query efficiency |

**Memory Spikes**: Compaction (every ~3 days) can spike memory 2-3x. Set memory limit with 2x headroom.

### InfluxDB TSM

```toml
[data]
# Cache size (memory before flush)
cache-max-memory-size = "1g"

# WAL fsync interval
wal-fsync-delay = "0s"  # 0 = immediate durability

# Compaction throughput
compact-throughput = "48m"
compact-throughput-burst = "48m"
```

**Schema Design for Performance**:
```
# Good: Tags for filtering, fields for values
temperature,location=us-east,sensor=temp1 value=25.5

# Bad: High cardinality tags
temperature,request_id=abc123 value=25.5  # request_id creates series explosion
```

---

## 10. Filesystem-Level Optimization

### ZFS ARC Tuning

```bash
# Check current ARC size
arc_summary

# Set max ARC (default: 50% of RAM on Linux)
echo "options zfs zfs_arc_max=8589934592" >> /etc/modprobe.d/zfs.conf  # 8GB

# L2ARC tuning (SSD cache)
echo "options zfs l2arc_write_max=67108864" >> /etc/modprobe.d/zfs.conf  # 64MB/s
echo "options zfs l2arc_noprefetch=0" >> /etc/modprobe.d/zfs.conf  # Cache prefetched blocks
```

**ARC Monitoring**:
```bash
arcstat 1  # Live hit ratios

# Key metrics:
# hits/s, misses/s, hit% (target > 90%)
# l2hits/s, l2misses/s (if L2ARC enabled)
```

### O_DIRECT vs Page Cache

**When to use O_DIRECT**:
- Database manages its own cache (MySQL InnoDB, RocksDB)
- Working set >> RAM
- Need predictable latency

**When to use Page Cache**:
- Working set fits in RAM
- Read-heavy with good locality
- Simplicity preferred

```c
// O_DIRECT requirements
// - Buffer must be aligned to logical sector size (typically 512 or 4096)
// - Offset must be aligned
// - Length must be aligned

int fd = open("file", O_RDWR | O_DIRECT);
void *buf;
posix_memalign(&buf, 4096, 4096);  // Align to 4KB
```

**Check alignment requirements** (Linux 6.1+):
```bash
# Use statx to query alignment requirements
stat --format='%B' /path/to/file  # Block size
```

---

## 11. Diagnostic Cheat Sheet

### One-Liners by Symptom

```bash
# High write latency (LSM)
# Check memtable flush rate
iostat -x 1 | awk '/sda/ && NR>2 {print $4, $5}'  # w/s, wMB/s

# mmap page fault rate
awk '/pgmajfault/ {print "major:", $2}' /proc/vmstat

# RocksDB compaction stall
grep -c "Stalling" /var/log/rocksdb/*.LOG

# Kafka segment lag
kafka-consumer-groups.sh --describe --group <group> | awk '{sum+=$5} END {print sum}'

# Parquet row group stats
parquet-tools meta file.parquet | grep "row group"

# Arrow batch memory usage
import pyarrow as pa; print(pa.total_allocated_bytes())
```

### Key Metrics by Engine Type

| Engine Type | Watch | Tool |
|-------------|-------|------|
| mmap (LMDB) | major faults, TLB misses | perf stat, vmtouch |
| LSM (RocksDB) | write stalls, compaction pending | db stats, iostat |
| Log (Kafka) | segment count, lag | kafka-tools |
| Columnar (Parquet) | row groups read vs skipped | query explain |
| Vectorized (Arrow) | batch size, memory | pa.total_allocated_bytes() |
| TSDB (Prometheus) | WAL size, compaction memory | prometheus metrics |

---

## 12. In-Memory Ring Buffer Message Stores

**Pattern**: Fixed-size circular buffer with power-of-2 masking for O(1) append and lookup. Used as a hot cache in front of durable storage (PostgreSQL, disk segments).

### When to Use

| Use Case | Why Ring Buffer |
|----------|----------------|
| Message broker hot cache | O(1) append + lookup, zero GC after init |
| Recent-window aggregation | Fixed memory, automatic eviction of old data |
| Event replay buffer | Sequential offsets, consumers trail behind producers |
| Deduplication tracker | Last N batches per producer, fixed-size per-key |

### Core Algorithm

```
Capacity = power of 2 (e.g., 1024)
Mask = capacity - 1 (e.g., 0x3FF)

store(item):
    offset = atomicSequence.getAndIncrement()   // 1 CAS
    buffer[offset & mask] = item                // 1 array write (bitwise AND = 1 CPU cycle)
    // old slot silently overwritten on wrap-around

get(offset):
    head = sequence.get() - capacity
    if offset < head or offset >= sequence.get(): return null   // out of window
    item = buffer[offset & mask]                                // 1 array read
    if item.offset != offset: return null                       // ABA protection
    return item
```

**Critical properties:**
- **No eviction loop.** Unlike `ConcurrentSkipListMap` + `while(size > limit) removeFirst()`, wrap-around is automatic.
- **No GC pressure.** Fixed `Object[]` allocated once. No node objects, no rebalancing, no resizing.
- **Cache-friendly.** Sequential array access vs pointer chasing in tree/list structures.
- **Lock-free writes.** Single `AtomicLong.getAndIncrement()` (`LOCK XADD` on x86) — concurrent writers get unique slots without contention.

### Performance Comparison

| Operation | ConcurrentSkipListMap | Ring Buffer | Speedup |
|-----------|----------------------|-------------|---------|
| Store (single) | O(log n) + node alloc | O(1), no alloc | 5-15x |
| Store (batch of 100) | 100 × O(log n) | 1 CAS + 100 writes | 10-20x |
| Get (single) | O(log n) | O(1) | 10-20x |
| Fetch (range of 10) | O(log n + 10) | O(10) | 3-5x |
| Eviction | O(k) explicit loop | Free (wrap) | ∞ |
| GC pressure | High (nodes) | Zero | N/A |

Measured on a high-throughput messaging workload: **35.8M get/s** on ring buffer vs **~2M ops/s** on `ConcurrentSkipListMap` (1024 entries, random access).

### Sizing

```
Buffer memory = capacity × entry_size
Entry = offset (8B) + timestamp (8B) + key ref (8B) + payload ref (8B) = ~32B metadata + payload

Example: 65,536 slots × 1KB avg message ≈ 64MB per partition
         100 partitions × 64MB = 6.4GB total
```

Rule: Size to hold 5-10 seconds of peak throughput. Messages older than the buffer window are fetched from durable storage (PostgreSQL, S3, segment files).

### ABA Problem and Protection

When the buffer wraps, slot N now holds message at offset `N + capacity` instead of offset `N`. A slow consumer reading offset `N` would get the wrong message.

**Protection**: Store the logical offset alongside the value. On read, verify `stored.offset == requested.offset`. If mismatch, the slot was overwritten — return null and fall back to durable storage.

```
Time T1: buffer[5] = Message(offset=5, payload=A)    // written by producer
Time T2: buffer[5] = Message(offset=1029, payload=B) // overwritten after wrap

Consumer reads offset 5:
  item = buffer[5 & mask]           // gets Message(offset=1029, payload=B)
  item.offset == 5?                 // NO → ABA detected, fall back to DB
```

### Concurrent Variants

| Variant | Thread Safety | Use Case |
|---------|---------------|----------|
| `AtomicLong` sequence + plain array | Multi-writer, single read path | Produce-heavy, single flush thread |
| `AtomicReferenceArray` slots + `VarHandle` | Per-slot volatile visibility | Multi-writer, multi-reader |
| `synchronized drain()` on inner list | Atomic batch extract | Batched flush with linger timer |

The `AtomicReferenceArray` variant gives per-slot atomic visibility without locking the entire buffer. Two threads writing to different slots have zero contention. Use `VarHandle` for the sequence counter to avoid `AtomicInteger` wrapper overhead on the hot path.

### Integration with Durable Storage

Ring buffer as L1 cache in a tiered message store:

```
Producer → Ring Buffer (L1, O(1), 64MB, ~10s window)
               │
               ├── Cache HIT → return directly (sub-microsecond)
               │
               └── Cache MISS → fetch from PostgreSQL/S3 (milliseconds)
                                  └── Backfill cache on fetch

Flush path:  Ring Buffer → Batch Executor → PostgreSQL (batch INSERT)
             Triggered by: count ≥ 1000 | size ≥ 1MB | linger ≥ 5ms
```

The flush triggers balance latency (linger timer) vs throughput (count/size batching). A linger timer of 5ms means: if no flush trigger fires within 5ms of the first write, flush anyway. This bounds worst-case latency.

### Diagnosing Ring Buffer Issues

```bash
# Symptom: consumers always miss the cache (high DB read rate)
# Cause: buffer too small or consumer too slow
# Fix: increase capacity or add consumer-side batching

# Symptom: producer throughput drops periodically
# Cause: flush executor can't keep up → backpressure
# Fix: increase flush batch size, add writer threads, or use MPMC queue

# Monitor wrap-around rate
# If lowWatermark advances faster than consumers read → buffer is too small
# Rule: consumer lag (in offsets) should be < 50% of buffer capacity
```

### Linux Kernel Tuning for Ring Buffer Workloads

```bash
# Huge pages reduce TLB misses for large buffers (>2MB)
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled
# Application calls madvise(MADV_HUGEPAGE) on the buffer allocation

# Lock pages in memory (prevent swap-out of hot buffer)
ulimit -l unlimited
# Application calls mlock() or uses -XX:+AlwaysPreTouch in JVM

# NUMA: pin buffer allocation to local node
numactl --membind=0 java -jar broker.jar
# Or use libnuma/JNA to allocate on specific NUMA node
```

---

## 13. Quick Reference: Tuning Parameters

### RocksDB

| Parameter | Default | Tuning Range | Impact |
|-----------|---------|--------------|--------|
| write_buffer_size | 64MB | 32MB-256MB | Memory, flush freq |
| max_write_buffer_number | 2 | 3-6 | Memory, stall prevention |
| level0_file_num_compaction_trigger | 4 | 4-10 | Write stalls |
| max_background_compactions | 1 | CPU cores | Compaction speed |
| bloom_filter_bits | 10 | 10-20 | Memory, FP rate |

### Kafka

| Parameter | Default | Tuning Range | Impact |
|-----------|---------|--------------|--------|
| log.segment.bytes | 1GB | 100MB-1GB | File count, retention granularity |
| log.index.interval.bytes | 4096 | 1024-8192 | Index size, seek speed |
| num.partitions | 1 | topic-dependent | Parallelism |
| log.retention.hours | 168 | workload | Storage |

### Prometheus

| Parameter | Default | Tuning Range | Impact |
|-----------|---------|--------------|--------|
| storage.tsdb.retention.time | 15d | 1d-∞ | Disk usage |
| storage.tsdb.wal-compression | true | true/false | CPU vs disk |
| storage.tsdb.max-block-duration | 2h | 2h-24h | Query efficiency |

---

## Sources

This chapter synthesizes information from:

- [LMDB Documentation](http://www.lmdb.tech/doc/)
- [RocksDB Tuning Guide](https://github.com/facebook/rocksdb/wiki/RocksDB-Tuning-Guide)
- [RocksDB Compaction Wiki](https://github.com/facebook/rocksdb/wiki/Compaction)
- [RocksDB Bloom Filter](https://github.com/facebook/rocksdb/wiki/RocksDB-Bloom-Filter)
- [Pebble - CockroachDB](https://github.com/cockroachdb/pebble)
- [BadgerDB](https://github.com/dgraph-io/badger)
- [libmdbx GitHub](https://github.com/erthink/libmdbx)
- [bbolt GitHub](https://github.com/etcd-io/bbolt)
- [ClickHouse MergeTree Documentation](https://clickhouse.com/docs/engines/table-engines/mergetree-family/mergetree)
- [Kafka Segment Retention](https://strimzi.io/blog/2021/12/17/kafka-segment-retention/)
- [Confluent Kafka Topic Configuration](https://docs.confluent.io/platform/current/installation/configuration/topic-configs.html)
- [Apache Arrow Format](https://arrow.apache.org/overview/)
- [DuckDB Performance Guide](https://duckdb.org/docs/stable/guides/performance/overview)
- [LMAX Disruptor User Guide](https://lmax-exchange.github.io/disruptor/user-guide/index.html)
- [Seastar Tutorial](https://github.com/scylladb/seastar/blob/master/doc/tutorial.md)
- [ScyllaDB Shard-per-Core](https://www.scylladb.com/product/technology/shard-per-core-architecture/)
- [io_uring Documentation](https://kernel.dk/io_uring.pdf)
- [SPDK NVMe Driver](https://spdk.io/doc/nvme.html)
- [Prometheus Storage](https://prometheus.io/docs/prometheus/latest/storage/)
- [InfluxDB TSM Engine](https://docs.influxdata.com/influxdb/v1/concepts/storage_engine/)
- [ZFS ARC Tuning](https://klarasystems.com/articles/performance-tuning-arc-l2arc-slog/)
- [OpenZFS Module Parameters](https://openzfs.github.io/openzfs-docs/Performance%20and%20Tuning/Module%20Parameters.html)
- [Direct I/O Semantics](https://ext4.wiki.kernel.org/index.php/Clarifying_Direct_IO's_Semantics)
- [ScyllaDB I/O Access Methods](https://www.scylladb.com/2017/10/05/io-access-methods-scylla/)
- [JCTools Lock-Free Queues](https://github.com/JCTools/JCTools) — Java concurrent queues (MPSC, MPMC, SPSC)
- [LMAX Disruptor Ring Buffer](https://lmax-exchange.github.io/disruptor/disruptor.html) — Original ring buffer pattern for JVM
- [Pinterest LMDB Optimization](https://medium.com/pinterest-engineering/how-optimizing-memory-management-with-lmdb-boosted-performance-on-our-api-service-f85fa7d1626d)
- [Parquet Performance Tuning](https://iceberglakehouse.com/posts/2024-10-all-about-parquet-part-10/)
- [DataFusion Parquet Pruning](https://datafusion.apache.org/blog/2025/03/20/parquet-pruning/)

---

## See Also

- [Disk & Storage](04-disk-storage.md) - I/O benchmarking, filesystem tools
- [Database Profiling](14-database-profiling.md) - PostgreSQL, MySQL, query optimization
- [Database Production Debugging](database-production-debugging.md) - Hot partitions, cache pollution
- [Memory Subsystem](15-memory-subsystem.md) - Page faults, THP, mmap behavior
- [Kernel Tuning](08-kernel-tuning.md) - vm.dirty_*, page cache, direct I/O settings
