# Compact Integer Sets: Roaring, Elias-Fano, Bloom Family, ART

Data structures for sets of integers (IDs, offsets, doc IDs, series IDs, tombstones) where size and set-op throughput both matter. LLM-actionable: explicit decision matrix, pick-by-workload heuristics, library pointers, integration with the rest of the perf handbook (TSDB, LSM, Kafka, search, OLAP).

**Why this chapter exists:** ~every performance-critical system eventually needs to answer *"is X in this set?"* or *"A ∩ B"* over millions to billions of integers. Picking the wrong structure bloats memory 100× or turns a 100 ns lookup into a 10 ms scan. The families below cover ~99% of real workloads; the trick is matching workload to structure, not picking a favorite.

Primary sources: Chambi/Lemire (Roaring), Vigna/Ottaviano (Elias-Fano), Lemire et al. (XOR/Binary Fuse), Leis et al. (ART).

---

## 1. The Baseline Problem

You have a universe `U = 2^32` (or `2^64`) and a set `S ⊂ U`. Two naive options collapse at the extremes:

| Structure | Memory | Point lookup | `A ∩ B` |
|---|---|---|---|
| Sorted `int32[]` | 4·\|S\| bytes | O(log n) binary search | O(\|A\|+\|B\|) merge |
| Flat bitmap (2^U bits) | 512 MB constant (U=2^32) | O(1) | SIMD AND, bounded by U |

Sorted array dies on intersection cost when `|S|` is large. Flat bitmap dies on memory when `|S|` is small. **Everything below is a way to get the best of both, probabilistically or exactly.**

---

## 2. Pattern Classification

| Family | Representative | Mutable | Exact | Point lookup | Intersection | Memory/elem (typical) |
|--------|-------|---------|-------|--------------|--------------|------------------------|
| Hybrid bitmap | **Roaring** | yes | yes | O(log k) | O(\|A\|+\|B\|) on keys | 1–32 bits |
| RLE bitmap | WAH, EWAH, Concise | rebuild | yes | scan | AND on runs | 2–8 bits |
| Succinct | **Elias-Fano**, PEF | immutable | yes | O(1) via `select` | O(\|A\|+\|B\|) | ~log₂(U/n)+2 bits |
| Probabilistic | **Bloom**, Cuckoo, XOR, Binary Fuse | partial | FP allowed | O(k) hashes | — | 7–12 bits |
| Trie | **ART**, Judy | yes | yes | O(key_len) | weak | variable, key-shape dependent |

Bold = the one you almost always want in that family.

---

## 3. Roaring Bitmaps — the 90% default

### 3.1 The core trick: 16+16 split

Every 32-bit integer `v` decomposes:
- `key = v >> 16` → identifies a **bin** (one of 65 536)
- `val = v & 0xFFFF` → position inside the bin

Only non-empty bins are allocated. Sparse and dense both stay compact.

```
Level 1: sorted array of (key, container_ptr)   → binary search O(log k)
Level 2: container, chosen per-bin based on content
```

### 3.2 Three container formats, auto-selected

| Container | Trigger | Layout | Size | Set ops |
|---|---|---|---|---|
| Array | ≤ 4 096 values | sorted `uint16[n]` | 2·n bytes | sort-merge / galloping |
| Bitmap | > 4 096 values | 2^16 bits fixed | 8 KB constant | SIMD AND (word-by-word) |
| Run | long consecutive ranges | `(start, length)[]` | 4·R bytes | interval merge |

Cross-over at 4 096 elements: below this, `2·n < 8 KB`. Above, bitmap wins. Runtime switches format after every insert/delete if needed.

### 3.3 Why intersection is fast

Two-level mirror:

```
Step 1: merge the level-1 key arrays (two-pointer scan).
        Cost: O(|keyA| + |keyB|). Typical: tens to thousands of keys.
Step 2: for matching keys only, intersect containers pairwise.
        Array×Array = sort-merge. Bitmap×Bitmap = 1024 ANDs (8 KB / 64).
        Mixed = membership test per element.
Step 3: if result drops below 4 096, demote bitmap→array.
```

A flat 2^32 bitmap AND = 8M 64-bit ops. Roaring on sparse data = 10⁴–10⁶× faster.

### 3.4 Sizing rule of thumb

```
sparse set of n IDs uniformly over 2^32:
  flat bitmap: 512 MB
  int32[]:     4·n bytes      (n=10 000 → 40 KB)
  roaring:     ~2·n bytes     (n=10 000 → 20 KB)
                               (n=10 000 000 → ~32 MB)

dense set (>50% of some bin filled):
  bitmap containers dominate: 8 KB per touched bin
```

### 3.5 When to pick Roaring

- You need mutation (add/remove) **and** fast set ops (AND/OR/ANDNOT/cardinality).
- Workload unknown a priori or mixed sparse+dense.
- Production libs exist in every major language (see §8).
- You want exact answers (no FP rate to tune).

**Skip Roaring if:** the set is immutable + extremely dense (Elias-Fano beats it 2×) **or** you only need membership with tolerable FP (Bloom family is 5–10× smaller).

---

## 4. RLE Bitmaps (WAH / EWAH / Concise) — mostly legacy

Chronology of the same idea: compress runs of 0s and 1s word-aligned.

| Year | Name | Status | Where you still see it |
|---|---|---|---|
| 2001 | WAH | superseded | research code |
| 2010 | EWAH | alive | **Git packfile bitmap index** |
| 2010 | Concise | superseded | old Druid versions |
| 2014 | Roaring | current | Druid (modern), ClickHouse, Elasticsearch, Lucene, Pilosa |

The problem: point lookup requires a **sequential scan** of the compressed stream — no per-key index. Roaring's 2-level keying killed the RLE family for random-access workloads. EWAH survives specifically because Git's pack bitmap index is write-once and read-sequentially.

**Decision rule:** don't pick WAH/EWAH/Concise for new code unless you have a specific ecosystem reason (Git).

---

## 5. Elias-Fano & Partitioned Elias-Fano (PEF) — when immutable beats everything

### 5.1 The core encoding

Given a sorted sequence `v₀ < v₁ < … < vₙ₋₁` in `[0, U)`:

```
L = ⌊log₂(U/n)⌋                     # bits per low part

Low part (fixed-width):              n · L bits
  Store the L low bits of each vᵢ verbatim.

High part (unary bucket histogram):  n + 2^H bits   where H = ⌈log₂ U⌉ - L
  Buckets of size 2^L.
  For each bucket: write a '1' per element inside, then '0' as separator.

Total: n · (log₂(U/n) + 2)   bits
```

This is within 2 bits of the information-theoretic minimum `log₂ C(U, n)`.

### 5.2 What you get for free

- `access(i)`: O(1) with `select₁` on high bits + low-bit fetch.
- `rank`, `select`, `predecessor`, `successor`: all O(1) with standard succinct support structures.
- Random access **without** decompressing.

### 5.3 Partitioned Elias-Fano

Real posting lists are not uniform. Chunk the sequence, pick EF parameters per chunk, add a skip index. **State of the art for search engine posting lists** (PISA, Tantivy, Terrier, Folly F14 posting lists).

### 5.4 When to pick Elias-Fano / PEF

- **Immutable indices**: build-once, query-many. Append-only TSDB blocks, search posting lists, static analytics.
- Dense-ish sorted data where roaring's bitmap containers would waste space.
- You need `rank/select` natively (navigation queries, cumulative stats).
- 2× memory vs. Roaring matters more than the ability to mutate.

**Skip if:** you need `.add()`/`.remove()`. Rebuild cost is linear.

### 5.5 TRADEOFF vs. Roaring

```
Roaring  : mutable,   ~2·n bytes  on sparse,  ~1 bit/elem on very dense
EF/PEF   : immutable, ~n·log₂(U/n)+2n bits  → typically ½ of Roaring on sorted dense data
```

Rule: if you'd otherwise use Roaring and the set is immutable and big, benchmark PEF — 2× memory reduction is common.

---

## 6. Bloom Family — "does X exist?" only

Philosophy: trade exactness for ~10 bits/element. Never false negatives; tunable false-positive rate.

### 6.1 Comparison

| Filter | Year | Bits/elem (1% FPR) | FPR floor | Delete | Immutable build | Typical use |
|---|---|---|---|---|---|---|
| **Bloom** | 1970 | 9.6 | tunable | no | no | LSM SST filters (Cassandra, classic RocksDB) |
| Counting Bloom | — | ~38 (4× Bloom) | tunable | yes | no | avoided in practice |
| **Cuckoo** | 2014 | ~7 | ~0.5% | yes | no | RocksDB modern filters, Scylla |
| **XOR** | 2019 | ~9.8 | 0.39% | no | yes | CDN negative caches |
| **Binary Fuse** | 2022 | ~9.0 | 0.39% | no | yes | **current best immutable filter** |
| Ribbon | 2021 | ~7–8 | tunable | no | yes | RocksDB optional filter |

### 6.2 Bloom sizing formula (memorize)

```
bits per element:   m/n = -1.44 · log₂(FPR)
optimal k hashes:   k   = (m/n) · ln 2

FPR 1%   →  9.6 bits/elem,  k=7
FPR 0.1% →  14.4 bits/elem, k=10
FPR 0.01%→  19.2 bits/elem, k=13
```

### 6.3 Decision tree for probabilistic filters

```
Need delete?
 ├─ yes → Cuckoo
 └─ no  → Can rebuild from source on schema change?
          ├─ yes → Binary Fuse (smaller + faster than Bloom/Cuckoo, immutable)
          └─ no  → Bloom (simplest, ubiquitous)
```

### 6.4 Where they show up in this handbook

- **LSM-tree SST filters** (ch19, ch14): Bloom → Cuckoo/Ribbon reduces read amplification for missing keys.
- **Redis pre-check** (ch21): Bloom filter at 1-in-10⁶ FPR → 80 MB for 10M IDs vs. 630 MB raw keys — 90% reduction.
- **Kafka dedup** (ch22): negative lookup on a rolling bloom filter.
- **CDN cache negative lookup**: "is this path definitely not in origin?" → Binary Fuse.

### 6.5 Trap: FPR drift under load

Bloom FPR is a probability over **random** inputs. If attackers or naturally correlated keys hash into the same positions, effective FPR for a specific query pattern can be 10–100× higher than the theoretical rate. Monitor false-positive *rate observed*, not *rate assumed*.

---

## 7. Adaptive Radix Tree (ART) — sorted sets with mutability

### 7.1 Concept

Trie over bytes of the key (256-ary), but with **four node sizes** to keep memory proportional to fan-out:

| Node | Capacity | Layout | Scan cost |
|---|---|---|---|
| Node4 | ≤ 4 | `uint8[4]` keys + `ptr[4]` | linear, fits in a cache line |
| Node16 | ≤ 16 | `uint8[16]` keys + `ptr[16]` | SIMD (SSE2 `pcmpeqb`) |
| Node48 | ≤ 48 | `uint8[256]` index + `ptr[48]` | O(1) index lookup |
| Node256 | ≤ 256 | `ptr[256]` direct | O(1) |

Nodes upgrade/downgrade automatically. **Path compression** stores shared prefixes inline, so depth ~ `log256(N)` for random keys.

### 7.2 What ART gives you that Roaring doesn't

- **Ordered iteration + range queries** — native, fast.
- **Ordered map**, not just a set (values attached to keys).
- Mutation at full speed, no rebuild.
- Keys are byte-strings, not just integers — use it for strings too.

### 7.3 Where it shows up

- **DuckDB**: primary index structure.
- **HyPer** (the paper): main-memory OLTP indexing.
- **RedisJSON**: path resolution.
- **libart** / **rART** (Rust): drop-in libs.

### 7.4 When to pick ART

- You need `(key → value)` ordered, mutable, range-queryable.
- Keys vary in length (strings, composite keys).
- You'd otherwise use a `std::map` / `BTreeMap` but want cache-friendly nodes.

**Skip if:** you only need set-ops (Roaring wins) or only point lookup on integers (hash table wins).

---

## 8. Decision Matrix

Stars are order-of-magnitude, not rigorous benchmarks.

| Structure | Sparse | Dense | Point lookup | Set ops (AND/OR) | Ordered iter | Range query | Insert/Delete | Exact |
|---|---|---|---|---|---|---|---|---|
| **Roaring** | ★★★★ | ★★★★ | ★★★ | ★★★★ | ★★★ | ★★ | ★★★ | yes |
| WAH/EWAH | ★★ | ★★★ | ★ | ★★★ | ★★★ | ★★ | rebuild | yes |
| **PEF** | ★★★★ | ★ | ★★★ | ★★ | ★★★★ | ★★★★ | immutable | yes |
| Bloom | ★★★★ | ★★★★ | ★★★ | — | — | — | add only | FP |
| Cuckoo | ★★★★ | ★★★★ | ★★★ | — | — | — | ★★★ | FP |
| **Binary Fuse** | ★★★★★ | ★★★★★ | ★★★ | — | — | — | immutable | FP |
| **ART** | ★★ | ★★ | ★★★★ | ★ | ★★★★ | ★★★★ | ★★★★ | yes |

---

## 9. Pick-by-Workload Heuristics (actionable)

```
Q: You have sets of integers. Which structure?

1. Do you need AND/OR/ANDNOT across sets?
   YES → Roaring (if mutable) or PEF (if immutable + dense)
   NO  → go to 2.

2. Do you only need "does X exist?" and FP is acceptable?
   YES → Bloom family:
           mutable + delete: Cuckoo
           immutable:        Binary Fuse (smaller + faster than Bloom in 2026)
           simple/portable:  Bloom
   NO  → go to 3.

3. Do you need ordered iteration, range queries, or key→value?
   YES → ART
   NO  → hash table suffices (out of scope here).
```

### Domain-specific defaults

| Domain | Default | Reason |
|---|---|---|
| TSDB inverted index (label → series_id) | Roaring | mutable, intersection-heavy (ch12) |
| Search engine posting lists | PEF | build-once, compact, native `rank/select` |
| LSM SST filters | Bloom (legacy) or Cuckoo/Ribbon (modern) | negative lookups only (ch14, ch19) |
| Kafka offset sets / tombstones | Roaring | mutable, range-dense (ch22) |
| CDN negative cache | Binary Fuse | immutable fleet-wide |
| Order book price index | ART or sorted vector | ordered + mutable + small (ch26) |
| OLAP predicate bitmaps (ClickHouse, Pinot) | Roaring | pushdown into vectorized scans (ch19) |
| Git pack bitmap index | EWAH | historical, ecosystem-locked |

---

## 10. Integration with Other Chapters

- **ch12 Observability**: `grep -r roaring` in Prometheus/Mimir source → `postings.go`. High-cardinality label explosions are roaring intersections going bad. Symptom: query latency scaling with `|series|` instead of `|result|`; cause: predicate didn't narrow via roaring early.
- **ch14 Database Profiling**: PostgreSQL BRIN indexes, ClickHouse `bloom_filter` / `tokenbf_v1` indexes, RocksDB `filter_policy`. Tuning these = tuning the structures in this chapter.
- **ch19 Storage Engine Patterns**: LSM bloom/ribbon filters at §2. Columnar engines (ClickHouse, DuckDB, Parquet) push predicates into roaring/bloom bitmap indices — that's how they scan 10 GB in 100 ms.
- **ch21 Caching Patterns**: bloom pre-filter in front of Redis/cache for existence checks. Sizing formula in §6.2.
- **ch22 Kafka**: tombstone sets, exactly-once dedup window → roaring or rolling bloom.
- **ch26 HFT**: perfect hashing (gperf, Frozen) is a cousin of bloom — compile-time exact filter for static symbol sets.

---

## 11. Libraries (production-grade, 2026)

| Structure | Language | Library | Notes |
|---|---|---|---|
| Roaring | C | [CRoaring](https://github.com/RoaringBitmap/CRoaring) | canonical, has Go/Rust/Python/JS bindings |
| Roaring | Java | [RoaringBitmap](https://github.com/RoaringBitmap/RoaringBitmap) | used by Lucene, Druid, Pinot |
| Roaring | Rust | `roaring` crate | pure Rust |
| Roaring | Go | `github.com/RoaringBitmap/roaring` | used by Prometheus-adjacent projects |
| Elias-Fano / PEF | C++ | [PISA](https://github.com/pisa-engine/pisa), [sux](https://github.com/vigna/sux) | research-grade, fast |
| Elias-Fano | Rust | `sucds` | succinct data structures |
| Bloom / Cuckoo / XOR / Binary Fuse | multi | [FastFilter](https://github.com/FastFilter) (Lemire et al.) | reference implementations |
| Bloom | Java | Guava `BloomFilter` | good enough, standard |
| ART | C | [libart](https://github.com/armon/libart) | embedded in DuckDB, HyPer |
| ART | Rust | `art-tree` | Rust port |
| ART | C++ | DuckDB source tree | extract if needed |

### One-line install / check

```bash
# C (Debian/Ubuntu)
sudo apt install libroaring-dev

# Rust
cargo add roaring

# Go
go get github.com/RoaringBitmap/roaring/v2

# Python (CRoaring binding)
pip install pyroaring

# Benchmark cardinality of existing bitmap dumps
python3 -c "import pyroaring; b=pyroaring.BitMap.deserialize(open('bm.roaring','rb').read()); print(len(b))"
```

---

## 12. Diagnostic Patterns

### 12.1 "Our query is slow at high cardinality"

```
Check:
  1. Is the predicate narrowing via bitmap intersection?
     → yes: roaring key-array scan bounded by |matching keys|
     → no:  loop over every series/row → O(N), not O(result)
  2. Is the bitmap dense enough to benefit from SIMD AND?
     → if arrays dominate, consider forcing bitmap format at load time
  3. Are the bitmaps actually in memory?
     → disk-resident roaring = you pay IO on every query (ch04)
```

### 12.2 "Bloom filter hit rate dropping"

```
Symptoms: LSM read amplification increasing, latency P99 creeping up.
Causes:
  - Key distribution shifted → effective FPR higher than designed
  - Filter too small for current N → m/n ratio dropped below target
  - Correlated keys → cluster into the same bits
Fix:
  1. Recompute m/n for current N, target FPR (see §6.2)
  2. Consider switching to Cuckoo/Ribbon (same FPR, fewer bits, or better locality)
  3. Monitor *observed* FPR, not configured FPR
```

### 12.3 "Immutable index is too big"

```
Currently using roaring? → try PEF, expect ~2× reduction on sorted dense data.
Currently using int32[]? → roaring or PEF.
Currently using a trie? → if keys are integers and you don't need range-by-prefix, roaring.
```

---

## 13. Recent Refinements (2025–2026)

### 13.1 Multi-dimensional range intersection (Berens-Teubner)

- **Structure:** k parallel roaring-like indices, one per dimension; intersection proceeds in **cardinality-estimated order** (smallest-first) with an early-exit when the running AND's cardinality falls below a threshold.
- **Pick it when:** you currently emulate a k-dimensional range by **AND-ing k separate roaring bitmaps**, with k ≥ 3, and the selectivity of each dimension varies widely across queries. Classical fixed-order AND wastes work on dimensions that don't narrow much.
- **How:** VLDB Dec 2025 paper (Berens, Teubner). No mainline lib yet; approximate in code by sorting your AND operands by cardinality ascending before the merge loop. For `CRoaring`, pre-compute `roaring_bitmap_get_cardinality(bm_i)` and sort.

### 13.2 Contextual Arithmetic Trits (CAT) posting lists

- **Structure:** sorted-int posting list compressed via **arithmetic coding with trits (base-3 symbols)** over a learned context model. Targets < 1 bit/element on postings with strong context structure (e.g., timestamps in a bounded range, IDs from a known distribution).
- **Pick it when:** **immutable search index** where PEF already feels suboptimal — you know the data has context structure (monotonic, clustered) that PEF's uniform model doesn't exploit.
- **How:** Barsamian-Chailloux ACM TOIS Sept 2025. Reference impl in paper; not a drop-in — requires an offline context model. For PISA users: worth a benchmark; for Lucene users: out of ecosystem.

### 13.3 Binary Fuse filter — still the 2026 immutable default

- **Structure:** §6.1 entry stays current. **No 2025 replacement.** Ribbon filter in RocksDB trades locality for smaller space; Binary Fuse still wins on raw speed + smallness for static sets.

### 13.4 Updated pick matrix

| Workload | Structure | Change from §5 |
|---|---|---|
| Mutable int-set, set ops | **Roaring** | unchanged |
| Immutable, sorted dense | **PEF** | unchanged |
| Immutable, context-rich posting list (time, ID) | **CAT** (research) | optional micro-optimization over PEF |
| Multi-D range (k≥3) intersection | **Roaring** + cardinality-sorted AND | operational tweak, not new structure |
| FP-tolerant immutable membership | **Binary Fuse** | unchanged |
| FP-tolerant mutable membership | **Cuckoo filter** | unchanged |
| Ordered map, range queries | **ART** | unchanged |

**What hasn't happened:** no Roaring-killer for mutable int sets. Despite ~10 alternative proposals per year, Roaring's mutability + prod-library ecosystem (CRoaring, Java, Rust, Go) keeps it the default. The §5 heuristics remain valid through 2026.

---

## 14. References

- Chambi, Lemire, Kaser, Godin (2016). *Better bitmap performance with Roaring bitmaps*. Software: Practice and Experience.
- Lemire et al. (2018). *Roaring bitmaps: Implementation of an optimized software library*. SPE.
- Vigna (2013). *Quasi-succinct indices*. WSDM.
- Ottaviano, Venturini (2014). *Partitioned Elias-Fano indexes*. SIGIR.
- Leis, Kemper, Neumann (2013). *The Adaptive Radix Tree: ARTful indexing for main-memory databases*. ICDE.
- Graf, Lemire (2020). *Xor Filters: Faster and Smaller Than Bloom and Cuckoo Filters*. ACM JEA.
- Dillinger et al. (2022). *Ribbon filter: practically smaller than Bloom and Xor*. (Meta/RocksDB).
- Fan et al. (2014). *Cuckoo Filter: Practically Better Than Bloom*. CoNEXT.
