# LSM Compaction Strategies: Leveled, Tiered, Hybrid, Lazy, Monkey, Wacky

Compaction is the biggest knob on an LSM engine. Wrong strategy → write amp 10×, P99 read 100×, disk at 2× dataset. Right strategy → same hardware, 3-5× the throughput. LSM basics are in [ch19](19-storage-engine-patterns.md#3-lsm-tree-write-optimized-engines); this chapter goes deep on compaction only.

Primary sources: RocksDB wiki, Cassandra/Scylla docs, Dayan et al. (Monkey SIGMOD 2017, Dostoevsky SIGMOD 2018, Wacky lineage 2022), Athanassoulis et al. (RUM Conjecture CIDR 2016).

---

## 1. RUM Conjecture — Compaction is a 2-of-3 Choice

Athanassoulis (CIDR 2016): pick **two** of Read-overhead, Update-overhead, Memory/space. The third pays.

| Access method | Picks | Pays |
|---|---|---|
| B+Tree | R, M | U (random writes) |
| LSM leveled | R, M | U (WA) |
| LSM tiered / STCS | U, M (bounded) | R (many runs) |
| FIFO / log only | U, M | R (scan) |
| Bloom / filter overlays | R, U | M |

Leveled pushes toward (R, M); tiered toward U. Everything below is a point on that curve.

---

## 2. Strategy Comparison

`T` = size ratio per level (default 10), `L` = #levels, `N` = entries. Formulas from Dostoevsky (Dayan & Idreos 2018).

Two dimensions: **generic family** (what happens at each level) and **system-specific name** (what a given DB calls its implementation).

| Generic family | RocksDB/LevelDB | Cassandra | ScyllaDB | Write amp | Read I/O (no filter) | Space amp | Best for |
|---|---|---|---|---|---|---|---|
| **Leveled** | Leveled (default) | **LCS** | LCS | `O(L·T)` ~20-30× | `O(L)` | `~1 + 1/T` (≈1.11) | Read-heavy, space-tight, SSD |
| **Size-Tiered** | (via Universal) | **STCS** (default) | **STCS** (OSS default) | `O(L)` ~log_T(N) | `O(T·L)` | up to 2× at major | Write-heavy bursts |
| **Universal (bounded-tiered)** | **Universal** | — | — | `O(L)` bounded | `O(L)` runs | bounded via `max_size_amplification_percent` | Write-heavy with space cap |
| **Time-Window** | — | **TWCS** | TWCS | `~1` per window | `O(#active windows)` | `~1` | TTL'd time series, append-mostly |
| **FIFO drop-oldest** | **FIFO** | — | — | `~1` | `O(#segments)` | `~1` | TTL metrics, drop-oldest |
| **Incremental (STCS + runs)** | — | — | **ICS** (Enterprise default ≥ 2020.1) | ≈STCS | ≈STCS | bounded via SSTable runs | STCS space fix, Enterprise |
| **Unified (tunable Dostoevsky)** | — | **UCS** (5.0+) | UCS (experimental) | f(W): WA↔RA slider | f(W) | f(W) | Cluster where workload is mixed or unknown; runtime-reconfigurable |
| **Lazy leveling (Dostoevsky)** | via `dynamic_level_bytes` | via UCS mixed `L/T` tokens | via UCS | `O(L+T)` | `O(T+L)` | `~1 + 1/T` | Pareto-optimal across broad parameter space |
| **WiscKey / KV sep** | **BlobDB** | — | — | `O(L)` on keys only | `O(L) + 1 rand` | depends on GC | Large values (>1 KB) |

**Take-away:** leveled trades WA for low read/space; tiered inverts. All others refine that axis.

---

## 3. Leveled — RocksDB/LevelDB Default

```
memtable → flush → L0 (overlapping, 4-8 files)
L0 → compact → L1 (non-overlapping, size = max_bytes_for_level_base)
L1 → L2 → … Ln      each level ~T× previous
```

L1+ invariant: a key lives in one SSTable per level. Compaction: pick one L_i SSTable + overlapping L_{i+1} files, merge.

**WA ≈ `1 + L·T/2`.** With T=10, L≈5 for 100 GB → WA ~26×. RocksDB wiki: 10-30× in production. SSD endurance pays.

```cpp
Options opts;
opts.compaction_style = kCompactionStyleLevel;
opts.write_buffer_size                 = 64 << 20;
opts.max_write_buffer_number           = 4;
opts.level0_file_num_compaction_trigger = 4;
opts.level0_slowdown_writes_trigger     = 20;
opts.level0_stop_writes_trigger         = 36;
opts.max_bytes_for_level_base           = 256 << 20;
opts.max_bytes_for_level_multiplier     = 10;
opts.target_file_size_base              = 64 << 20;
opts.level_compaction_dynamic_level_bytes = true;   // see §3.1
opts.max_background_jobs = 8;
opts.max_subcompactions  = 4;
```

### 3.1 `level_compaction_dynamic_level_bytes=true`

Without it, partially-filled DBs keep upper levels empty while still paying full WA to the bottom. Turn it on → RocksDB sizes bottom-up: bottommost = actual data, each level above = 1/T of it. Rockset/Facebook: 20-40% WA reduction on partially-filled DBs, space amp bounded ≈1.111. **Almost always on in 2026.**

### 3.2 When it wins

Read-heavy, P99 matters, SSD endurance OK, space tight. OLTP KV, metadata, CockroachDB/TiKV on NVMe.

---

## 4. Size-Tiered (STCS) — Cassandra Default

Bucket SSTables by size; when `min_threshold` (4) similar-size SSTables exist, merge into one. No levels.

```yaml
compaction = {
  'class': 'SizeTieredCompactionStrategy',
  'min_threshold': 4, 'max_threshold': 32,
  'bucket_low': 0.5, 'bucket_high': 1.5,
  'min_sstable_size': 50,
  'tombstone_threshold': 0.2,
  'tombstone_compaction_interval': 86400
}
```

**WA ≈ `log_T(N)`** (much lower). **Read amp = killer**: a point lookup probes every run, bloom filters carry everything.

**Temporary 2× space:** major compaction keeps old + new on disk. Plan free space accordingly — has killed many clusters.

**Rough tombstone problem:** tombstones in small tiers never meet shadowed data in the biggest tier → they never drop. Signal: `Droppable tombstone ratio > 0.5` with compactions idle → force `nodetool compact` or switch strategy.

Wins: write-heavy, bursty, disk headroom available, reads concentrate on recent data.

---

## 5. Universal — RocksDB Tiered-with-Guardrails

```cpp
opts.compaction_style = kCompactionStyleUniversal;
auto& u = opts.compaction_options_universal;
u.size_ratio = 1;
u.min_merge_width = 2;
u.max_size_amplification_percent = 200;   // total/bottom ≤ 2×
```

Single level, newest→oldest sorted runs. Three merge triggers: space-amp cap, size-ratio, width. WA 2-5× (RocksDB wiki) vs leveled's 10-30×. Peak disk ~2× during largest merge. Read amp `O(#runs)` — filters mandatory.

Where it shines: Kafka Streams / Flink RocksDB state (ch22), write-heavy with durability priority over read latency.

---

## 6. FIFO — Drop-Oldest, No Merge

```cpp
opts.compaction_style = kCompactionStyleFIFO;
opts.compaction_options_fifo.max_table_files_size = 100ULL << 30;
opts.compaction_options_fifo.allow_compaction     = false;
opts.ttl = 7 * 24 * 3600;
```

No merging. Delete oldest SSTable when size cap or TTL hits. WA=1. Read amp = #L0 files — only viable when reads are sequential by time. Classic: metrics, short-retention logs. Never mix with point lookups.

---

## 7. TWCS — Time-Window Compaction

```yaml
compaction = {
  'class': 'TimeWindowCompactionStrategy',
  'compaction_window_unit': 'HOURS',
  'compaction_window_size': 1
}
```

Time buckets. Inside current window: STCS-like. Rolled-over windows compact once, then frozen. No cross-window compaction.

**Why it wins for TTL:** all data in a window shares TTL → when window expires, the entire SSTable drops (Cassandra's "fully expired SSTable" fast path). No merge, no tombstones, no read cost.

Traps:
- **Out-of-order writes** land in old windows → force recompact → cliff. Reject late writes at app.
- **Mixed TTL in one table** → can't drop wholesale → tombstones pile. Split tables by TTL class.
- Window sizing: aim `retention / window ∈ [20, 50]`.

---

## 8. Hybrid / Lazy Leveling (Dostoevsky)

Dostoevsky (Dayan VLDB 2018): tiered on all levels except the last; leveled only on the last.

**Insight:** ≥1-1/T of the dataset lives at the last level. Leveling the non-last levels pays `T×` WA for little space-amp win. Lazy leveling saves ~T× WA on 1-1/T of the work, matches leveling's space amp within ~1%. **Pareto-optimal across a wide parameter space** (Dostoevsky Fig. 4-6).

Approximations available today:
- RocksDB leveled + `level_compaction_dynamic_level_bytes=true` + large base (hybrid at L0+L1, leveled deeper).
- ScyllaDB **ICS** (Incremental Compaction Strategy, **Enterprise-only**) — splits SSTables into sorted runs of 1 GB fragments (default `sstable_size_in_mb=1000`), compacts one fragment at a time. Same RA/WA as STCS but temp-space overhead drops from ~100% (STCS, forcing ~50% usable disk) to ~5% on major compactions (up to ~80% usable disk). **Default in ScyllaDB Enterprise ≥ 2020.1**; not available in ScyllaDB open-source.
- Cassandra 5.0 **UCS** (Unified Compaction Strategy, CEP-26): see §8.1.

### 8.1 UCS — Cassandra 5.0's tunable Dostoevsky

UCS unifies STCS + LCS + lazy leveling behind a **single integer scaling parameter `W`** (alias `scaling_parameters`). Groups SSTables in levels by `log_F(size)`, triggers compaction per level at `T` SSTables. `W` sets both `F` and `T`:

| `W` | Mode | Fan factor `F` | Trigger `T` | Behaves like |
|---|---|---|---|---|
| `W < 0` | Leveled | `2 − W` | `2` | LCS with fan `\|W\|+2` |
| `W = 0` | Degenerate | `2` | `2` | Neutral — leveled ≡ tiered |
| `W > 0` | Tiered | `2 + W` | `2 + W` | STCS with threshold `F` |
| `W = 2` (default) | Tiered | `4` | `4` | ≈ STCS default |
| `W = −8` | Leveled | `10` | `2` | ≈ LCS default |

**Per-level mix syntax (5.0+):** pass a string of `T/L/N` tokens + optional integers — e.g. `'scaling_parameters': 'T8, T4, N, L4'` → tiered-8 at L0, tiered-4 at L1, neutral at L2, leveled-4 beyond. Tokens shorter than depth repeat the last. **Explicit lazy leveling:** `'T4, T4, L4'` = tiered upper + leveled last.

```cql
ALTER TABLE t WITH compaction = {
  'class': 'UnifiedCompactionStrategy',
  'scaling_parameters': 'T4, T4, L4',    -- lazy-leveled Dostoevsky
  'target_sstable_size': '1GiB',
  'base_shard_count': '4'
};
```

**Runtime reconfig:** changing `W` only triggers the compactions needed to reach the new state; already-done work is incorporated. **→ UCS is the only production strategy you can re-tune without downtime.**

Start with `W=2` (default) on new clusters. Shift negative on read-heavy, positive on write-heavy. Use the mix syntax when workload is clearly lazy-leveled (time-stable large dataset).

---

## 9. Monkey — Per-Level Bloom Sizing

Classical RocksDB gives every level the same `bits_per_key` (default 10 → ~1% FPR; see [ch27 §6.2](27-compact-integer-sets.md#62-bloom-sizing-formula-memorize)). Dayan (SIGMOD 2017) proved this wrong.

**Insight:** a point lookup probes levels top-down; FP at level `i` = one SSTable I/O. Upper levels are probed on every lookup; lower levels only after all upper BFs miss. Optimal (Monkey, simplified):

```
bits_per_key_at_level_i  ∝  ln(T) + L - i
```

Shallower levels get **more** bits. Same total filter memory, expected FP collapses.

Monkey paper: on uniform point lookups, ~2-3× fewer zero-result I/Os at constant filter memory → ~40-60% P99 drop on miss-heavy workloads (existence checks, dedup).

RocksDB approximation (no first-class Monkey):
```cpp
opts.optimize_filters_for_hits = true;    // drop filter on bottommost (crude Monkey)
table_options.partition_filters = true;
table_options.cache_index_and_filter_blocks       = true;
table_options.pin_l0_filter_and_index_blocks_in_cache = true;
```

`optimize_filters_for_hits=true`: bottom level has no filter (~90% filter memory saved). Good for "key usually exists"; bad for existence-check workloads — keep filters on all levels there.

---

## 10. Wacky — Joint Auto-Tuner (Dayan 2022)

Monkey (filters) + Dostoevsky (lazy leveling) + memtable tuning as a **joint** optimization. Cost = weighted WA + read I/O + space amp with workload-derived weights; solved analytically, not grid search.

No off-the-shelf flag. Take-aways:
1. Tune **jointly**, not one knob at a time.
2. Feed workload stats (R/W ratio, zero-result query rate, key Zipf) into the tuner.
3. Re-tune on drift — static configs rot.

Approximating systems: RocksDB-Cloud tuning service, TiKV auto-adjustments, CockroachDB heuristics.

---

## 11. Compaction Internals

### 11.1 Subcompactions

```cpp
opts.max_background_jobs = 8;   // flush + compact threads
opts.max_subcompactions  = 4;   // parallelism within a single job
```

Splits one compaction's key range across threads. Scales near-linearly to SSD bandwidth. `max_background_jobs ≈ #cores/2` on dedicated nodes.

### 11.2 Trivial moves

When L_i SSTable's key range doesn't overlap L_{i+1}, RocksDB **renames** the file — zero data rewrite. Monitor `rocksdb.num-trivial-move`.

### 11.3 Intra-L0 compaction

If L0→L1 is blocked, RocksDB merges L0 files among themselves to bound read amp. Automatic at `level0_file_num_compaction_trigger`.

### 11.4 Compaction filters

Row-level callback during compaction. Use for TTL-without-tombstones, schema migration, GDPR purge.

```cpp
class PurgeFilter : public rocksdb::CompactionFilter {
  Decision FilterV2(int, const Slice& k, ValueType, const Slice& v,
                    std::string*, std::string*) const override {
    return IsExpired(v) ? Decision::kRemove : Decision::kKeep;
  }
  const char* Name() const override { return "PurgeFilter"; }
};
opts.compaction_filter = new PurgeFilter();
```

**Gotcha:** filter sees one SSTable at a time → resurrection risk if dropping a delete while deeper put survives.

### 11.5 Periodic / TTL compactions

```cpp
opts.ttl = 30 * 24 * 3600;
opts.periodic_compaction_seconds = 7 * 24 * 3600;
```

Forces long-untouched SSTables to rewrite → drops expired tombstones, re-applies filters, updates compression. **Enable on any long-retention DB** — unflushed tombstones are the #1 cause of creeping read amp.

---

## 12. Tombstones — The Silent Killer

### 12.1 Survival rules

Tombstone drops only when:
- Reached the **bottommost level** (RocksDB) or the SSTable with the shadowed put (Cassandra STCS), **AND**
- Older than grace period (`gc_grace_seconds`, default 10 days in Cassandra) so hinted handoff / repair propagate it.

Until then: space + read cost on every scan.

### 12.2 Tombstone resurrection

```
T0: PUT k=1 → L3
T1: DELETE k → tombstone in L0
T2: tombstone compacts down, meets PUT at L3, both drop.
    gc_grace NOT elapsed.
T3: node was down T1..T3, missed the tombstone.
T4: repair: sees surviving PUT on other replicas, tombstone gone → PUT resurrects.
```

Dead record reappears. Guardrails:
- Never `gc_grace_seconds = 0` on replicated tables with MVs.
- `CompactionFilter` dropping deletes must honor wall-clock grace.
- Bulk deletes + long compaction lag + short grace = resurrection storm.

### 12.3 Range deletes vs point deletes

Range delete: one tombstone for `[start, end)`. Cheaper to store/scan, but RocksDB range tombstones live in a separate block (extra lookup check); compacting range + overlapping point tombstones is O(n log n) — a flood can stall compaction. Rule: **one range delete over 100k point deletes**; partition-level TTL better still.

### 12.4 Diagnostic

```bash
# Cassandra
nodetool tablestats ks.tbl | grep -E 'tombstone|droppable'
# "Droppable tombstone ratio: 0.6" + idle compactions → wrong strategy

# RocksDB
ldb --db=/var/lib/rocks/db dump_live_files   # num_deletions per SST
# rocksdb.compaction-pending + high num-files-at-levelN → backlog
```

---

## 13. Stall Diagnosis

RocksDB write stalls — symptom to knob:

| Symptom | Cause | Knob |
|---|---|---|
| `Stalling: N immutable memtables` | Flush too slow | ↑ `max_background_flushes`, ↑ `max_write_buffer_number`, check disk BW |
| `Stalling: pending compaction bytes` | Deep compaction lag | ↑ `max_background_compactions`, `max_subcompactions`; check IOPS |
| `Stalling: N level-0 files` | L0→L1 lag | ↑ subcompactions, cautiously ↑ `level0_slowdown_writes_trigger` |
| `Stopping: hard limit` | Hit `level0_stop_writes_trigger` | Emergency ↑ trigger; fix root cause |
| `write-amp-above-threshold` | WA > SSD sustainable | Switch to universal/tiered or slow writes |
| P99 read 10× P50 | Bloom FPR drift, cold SSTable | ↑ `bits_per_key`, warm block cache |
| Space amp > 2× | Missing major compaction, stuck tombstones | `periodic_compaction_seconds`, `CompactRange`, check grace |
| Compaction CPU 100%, disk idle | Compression CPU-bound | LZ4 shallower, Zstd bottom only; ↓ Zstd level |
| Compaction disk 100%, CPU idle | I/O saturated | Rate limiter; ↑ `compaction_readahead_size` (ch04) |
| Read IOPS spikes during compaction | No readahead on compaction | `opts.compaction_readahead_size = 2 << 20;` |

### 13.1 Rate limiter

```cpp
opts.rate_limiter.reset(rocksdb::NewGenericRateLimiter(
    /*bytes_per_sec=*/ 100 << 20, 100000, 10,
    rocksdb::RateLimiter::Mode::kWritesOnly));
```

Essential on GP3/Azure Premium where compaction starves foreground reads. Budget ≤50% of disk IOPS ([ch04](04-disk-storage.md)).

---

## 14. Compression per Level

Heat gradient: L0 hot → bottom cold.

```cpp
opts.compression_per_level = {
    kNoCompression, kNoCompression,        // L0-L1 hot
    kLZ4Compression, kLZ4Compression,      // mid
    kZSTD, kZSTD, kZSTD,                   // cold
};
opts.bottommost_compression = kZSTD;
opts.bottommost_compression_opts.level = 6;
```

Meta/RocksDB wiki: Zstd-6 on bottom cuts space 30-50% vs LZ4 at ~2× compaction CPU, zero read-latency impact (decompression is fast). LZ4 mid-levels = free win. No-compression on L0/L1 avoids double-compressing data that'll be rewritten in seconds.

---

## 15. Decision Tree

```
Append-only with TTL?
├─ Single TTL class, reads by time range only  → FIFO
├─ Single TTL class, point/mixed reads          → TWCS (Cassandra/Scylla) or universal+ttl (RocksDB)
│
Mutable keys:
├─ Reads dominant (>70%), P99 matters
│    → Leveled + level_compaction_dynamic_level_bytes=true + per-level filters (Monkey-ish)
│
├─ Writes dominant
│    disk ≥ 2× dataset        → STCS / Universal
│    disk ~1.5× dataset       → Universal, max_size_amplification_percent=150
│    space amp must be ~1.1   → Leveled (accept 10-30× WA)
│
├─ Mixed, unclear
│    → Lazy leveling (Dostoevsky) via RocksDB leveled+dynamic+partitioned filters,
│      or Scylla ICS, or Cassandra 5.0 UCS with tuned W
│
└─ Large values (>1 KB avg)
     → WiscKey: BadgerDB, or RocksDB BlobDB
         opts.enable_blob_files = true;
         opts.min_blob_size = 1024;
         opts.blob_file_size = 256 << 20;
         opts.blob_compression_type = kZSTD;
       WiscKey (Lu et al. FAST 2016): WA 3-10× lower when values dominate.
```

### 15.1 Quick matrix

| Workload | Strategy | Key flags |
|---|---|---|
| OLTP KV (CockroachDB, TiKV) | Leveled + dynamic | `level_compaction_dynamic_level_bytes=true`, partitioned filters |
| Kafka Streams / Flink state (ch22) | Universal | `kCompactionStyleUniversal` for write bursts |
| Metrics / logs (TTL) | FIFO or TWCS | `ttl`, `periodic_compaction_seconds` |
| Cassandra write-heavy | UCS (5.0+) or STCS | `tombstone_threshold=0.1`; ICS needs Scylla Enterprise |
| Cassandra time series | TWCS | `compaction_window_size` ≈ retention/30 |
| Large values (>1 KB) | BadgerDB / BlobDB | key-value separation |
| Read-mostly, existence checks | Leveled + 14-16 bits/key | `optimize_filters_for_hits=false` |
| Dedup / zero-result lookups | Leveled + Monkey filters | per-level bits if possible |

---

## 16. OPTIONS File Sanity Check

RocksDB persists `OPTIONS-xxxxx` at each flush/compaction. Read the **live** one, not your template.

```bash
ls -t /var/lib/rocks/db/OPTIONS-* | head -1
ldb --db=/var/lib/rocks/db dump_options
```

Checklist:
1. `compaction_style` correct?
2. `level_compaction_dynamic_level_bytes=true` unless reason.
3. `max_background_jobs`, `max_subcompactions` parallel enough?
4. `write_buffer_size × max_write_buffer_number` = total memtable.
5. `compression_per_level`: none on L0/L1, Zstd on bottom.
6. `ttl`, `periodic_compaction_seconds` set on long-retention DBs.
7. Bloom config matches workload (see §9).
8. `rate_limiter` set on cloud disks.
9. `compaction_readahead_size` ≥ 2 MB on HDD / network block.

Cross-check live:
```bash
ldb --db=/var/lib/rocks/db stats
# look for: W-Amp per level, Flush(GB), Read(GB), Stall(sec) — should be ~0
```

---

## 17. Diagnostic Patterns

### 17.1 Write stalls every few minutes
1. `rocksdb.cfstats` → Stall(sec) per level.
2. L0 stalls → flush slow (↑ flushes, ↑ buffer) or L0→L1 slow (↑ subcompactions).
3. Pending compaction bytes → deep levels lagging; ↑ bg compactions, check IOPS.

### 17.2 Read P99 spikes over time
1. Bloom FPR drift (useful vs full-positive) → filter undersized for grown N (ch27 §6.2).
2. L grew? More levels = more probes. ↑ `max_bytes_for_level_base` or dynamic.
3. Tombstones per SSTable rising → missing `periodic_compaction_seconds`.
4. Block cache hit falling → working set grew, ↑ `block_cache`.

### 17.3 Space amp too high
1. Major compaction running → 2× is transient.
2. Stuck compactions (pending high, disk idle) → rate limiter too tight or all threads flushing.
3. gc_grace / TTL too long → tombstones hoarded.
4. `level_compaction_dynamic_level_bytes=false` on half-full DB → turn it on.

### 17.4 Tombstone backlog
1. `Droppable tombstone ratio > 0.3` + idle compactions → wrong strategy.
2. `gc_grace_seconds < 2× repair interval` → resurrection risk.
3. Prefer one range delete over 100k point deletes.

---

## 18. Systems

| System | Default | Notes |
|---|---|---|
| [RocksDB](https://github.com/facebook/rocksdb) | Leveled | most configurable; Universal, FIFO, BlobDB |
| [LevelDB](https://github.com/google/leveldb) | Leveled | original, minimal knobs |
| [Pebble](https://github.com/cockroachdb/pebble) | Leveled | CockroachDB Go clone; dynamic levels default |
| [BadgerDB](https://github.com/hypermodeinc/badger) | Leveled | WiscKey, large values |
| [Cassandra](https://cassandra.apache.org/doc/stable/cassandra/operating/compaction/index.html) | STCS | LCS, TWCS, **UCS** (5.0+) |
| [ScyllaDB OSS](https://opensource.docs.scylladb.com/stable/kb/compaction.html) | STCS | LCS, TWCS; **ICS Enterprise-only** |
| [ScyllaDB Enterprise](https://docs.scylladb.com/) | **ICS** ≥ 2020.1 | SAG for space-amp goal |
| [HBase](https://hbase.apache.org/book.html#regions.arch.compactions) | Minor+Major | stripe compactions ≈ leveled |
| [TiKV](https://docs.pingcap.com/tidb/stable/tune-tikv-memory-performance) | RocksDB leveled | `defaultcf` vs `writecf` tuned separately |
| [Sled](https://github.com/spacejam/sled) | log-structured | not classical LSM |

**ScyllaDB Enterprise: ICS default since 2020.1 — don't override blindly.** **ScyllaDB OSS: still STCS default; no ICS.** **Cassandra 5.0 UCS** is tunable Dostoevsky; start with UCS on new clusters.

---

## 19. Integration with Other Chapters

- **[ch04 Disk & Storage](04-disk-storage.md)** — compaction is an I/O amplifier. Budget ≤50% of disk IOPS; set `compaction_readahead_size` ≥ 2 MB on rotating/cloud block; `fio` from ch04 tells the ceiling.
- **[ch14 Database Profiling](14-database-profiling.md)** — Cassandra/Scylla/TiKV diagnosis; this chapter is the prescriptive companion.
- **[ch19 Storage Engine Patterns](19-storage-engine-patterns.md)** — LSM fundamentals.
- **[ch22 Kafka](22-kafka-messaging.md)** — Streams/Flink RocksDB state stores; universal compaction + short retention usually right.
- **[ch27 Compact Integer Sets](27-compact-integer-sets.md)** — bloom/ribbon sizing math. Monkey (§9) is that math per-level. Ribbon filters: ~30% memory savings at same FPR, drop-in for RocksDB.

---

## 20. Recent Structures & Tuning (2025–2026)

### 20.1 The metric shift: WA → throughput under CPU contention

**What changed:** 2025 SIGMOD work (Tsinghua) measured that pure-leveled RocksDB achieves only **64% of the query throughput** of lazy-leveling because **compaction burns 62% of CPU** under steady writes. The RUM-conjecture "minimize WA" target is misleading when your bottleneck is CPU, not IOPS.

**Pick it when:** CPU-bound LSM workload (Flink/Kafka Streams state, high QPS KV stores). Symptom: `iostat` shows disk <50% util while `mpstat` shows compaction threads at 100%.

**How to apply:**
```
# Cassandra 5.0+ — explicit lazy-leveling via UCS
ALTER TABLE t WITH compaction = {
  'class': 'UnifiedCompactionStrategy',
  'scaling_parameters': 'T4, T4, L4',       -- tiered L0/L1, leveled L2+
  'target_sstable_size': '1GiB'
};

# RocksDB — approximate lazy-leveling
opts.level_compaction_dynamic_level_bytes = true;   // auto-size bottom-up
opts.max_bytes_for_level_base = <data_size> / 10;   // target ≈ 1/T of expected data
opts.max_background_jobs = min(num_cores / 2, 8);   // cap CPU stolen from queries
```

### 20.2 New / refined structures & strategies

| Structure / strategy | What it is | Pick it when | Lib / config |
|---|---|---|---|
| **Horizontal-growth LSM** (SIGMOD 2025) | More SSTables per level, fewer levels | Your data is small-medium (<1 TB/node) and you were scaling `multiplier=10` blindly → consider `multiplier=4` + larger base | RocksDB: `max_bytes_for_level_multiplier=4`, `max_bytes_for_level_base` ~10× memtable |
| **UCS `T4, T4, L4` mix** (Cassandra 5.0) | Tiered upper, leveled last — explicit Dostoevsky | Mixed read/write Cassandra cluster, or unknown workload shape | Cassandra ≥5.0 CQL above |
| **UCS `W=−8`** (LCS-equivalent) | Pure leveled via UCS | Read-heavy Cassandra where you want runtime-tunable LCS (unlike static LCS) | `scaling_parameters=-8` |
| **UCS `W=+4`** (aggressive tiered) | STCS-like but bounded, tunable | Write-heavy burst workload, Cassandra | `scaling_parameters=4`, `target_sstable_size='1GiB'` |
| **ICS + SAG (Space Amp Goal)** | Scylla Enterprise: ICS with leveled-like last tier | Scylla Enterprise write-heavy + space-tight | `compaction_strategy_options={space_amplification_goal: 1.5}` |
| **Workload-adaptive (ArceKV-style)** | Runtime switch leveling↔tiered based on observed r/w ratio | No single workload phase dominates; daily cycle (write-heavy ingest night, read-heavy day) | Research stage; approximate today with **periodic online reconfiguration** of UCS `W` via cron |
| **Compact SST index (Disco-style)** | Shrinks partition-filter/index memory footprint | Bloom + partition-index eats >10% of RAM budget | Not yet mainline; monitor `rocksdb.estimate-table-readers-mem` — if >5 GB, pre-reserve via `cache_index_and_filter_blocks=true` + pinned top level |

### 20.3 Decision update for 2026

```
Workload signature                        → Strategy (concrete config)

Write-heavy, CPU-bound                    → UCS 'T4, T4, L4'  or  RocksDB leveled + dynamic_level_bytes
                                              + cap background_jobs at ~num_cores/2
                                              (SIGMOD 2025 throughput-optimal)

Write-heavy, IO-bound                     → STCS / UCS W>0  (classical WA-min)
                                              + cap compaction bandwidth via rate_limiter

Read-heavy, bloom-enabled                 → LCS / UCS W=−8 + Monkey bloom sizing
                                              (ch27 §6.2 for bloom/elem per level)

TTL metrics                               → FIFO (RocksDB) or TWCS (Cassandra/Scylla)
                                              — not UCS; bucketing matters more than W

Mixed / unknown / will evolve             → UCS 'T4, T4, L4' default + plan to re-tune W
                                              online as you learn the workload
```

**Key operational change from 2018→2025 thinking:** **throttle compaction CPU** (`max_background_jobs`, rate limiter) is now as important as picking the right strategy. A leveled LSM with compaction-threads = all-cores gives the worst of both worlds: high WA cost *and* low query throughput.

---

## 21. References

- Athanassoulis et al. (2016). *Designing Access Methods: The RUM Conjecture*. CIDR.
- Dayan, Athanassoulis, Idreos (2017). *Monkey: Optimal Navigable Key-Value Store*. SIGMOD.
- Dayan, Idreos (2018). *Dostoevsky: Better Space-Time Trade-Offs for LSM-Tree KV Stores via Adaptive Removal of Superfluous Merging*. SIGMOD.
- Dayan et al. (2022). *Spooky* / Wacky-lineage auto-tuner. SIGMOD.
- Lu, Pillai, Arpaci-Dusseau (2016). *WiscKey: Separating Keys from Values in SSD-Conscious Storage*. FAST.
- [RocksDB Wiki — Compaction](https://github.com/facebook/rocksdb/wiki/Compaction), [Leveled](https://github.com/facebook/rocksdb/wiki/Leveled-Compaction), [Universal](https://github.com/facebook/rocksdb/wiki/Universal-Compaction), [Write Stalls](https://github.com/facebook/rocksdb/wiki/Write-Stalls), [BlobDB](https://github.com/facebook/rocksdb/wiki/BlobDB)
- [Cassandra Compaction Strategies](https://cassandra.apache.org/doc/stable/cassandra/operating/compaction/index.html), [UCS 5.0](https://cassandra.apache.org/doc/latest/cassandra/managing/operating/compaction/ucs.html)
- [ScyllaDB — Incremental Compaction](https://www.scylladb.com/2019/05/08/incremental-compaction/)
- [Rockset — Remote Compactions in RocksDB-Cloud](https://rockset.com/blog/remote-compactions-in-rocksdb-cloud/)
- [BadgerDB — WiscKey design](https://dgraph.io/blog/post/badger/)
- [Pebble design](https://github.com/cockroachdb/pebble/blob/master/docs/rocksdb.md)

---

## See Also

- [Storage Engine Patterns](19-storage-engine-patterns.md) — LSM fundamentals
- [Database Profiling](14-database-profiling.md) — Cassandra/Scylla/TiKV diagnosis
- [Disk & Storage](04-disk-storage.md) — IOPS budgets, fio, readahead
- [Kafka Messaging](22-kafka-messaging.md) — RocksDB in Streams/Flink state stores
- [Compact Integer Sets](27-compact-integer-sets.md) — bloom/ribbon sizing (Monkey's building block)
