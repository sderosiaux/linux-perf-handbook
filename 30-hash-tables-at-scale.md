# Hash Tables at Scale: SwissTable, F14, Robin Hood, Cuckoo, Perfect Hashing

The default hash map shipped with `std::unordered_map`, `java.util.HashMap`, or Python's `dict` is not what production systems at scale use. LLM-actionable: pick the right hash table family from the decision matrix in §10 without reading anything else.

**Why this chapter exists:** hash tables are the universal indexing primitive. Pick the wrong one and pay 3-5x on lookup, 2x on memory, or eat a HashDoS incident. `std::unordered_map` → `absl::flat_hash_map` is typically 2-5x on throughput; a badly-sized map → a `reserve()`d one is 10x on bulk insert; `wyhash` vs `std::hash` on strings is +30-40%. Each is a one-line change.

Primary sources: Kulukundis (SwissTable, CppCon 2017), Bronson & Shi (F14, Meta 2019), Skarupke (Robin Hood), Fan & Andersen (MemC3/libcuckoo), Pibiri (PTHash), Esposito-Graf-Vigna (RecSplit).

See ch26 §4 for the HFT-focused intro; this chapter is the full reference.

---

## 1. Chaining vs Open Addressing

| Dimension | Chaining (`std::unordered_map`, old Java) | Open addressing (SwissTable, F14, hashbrown) |
|---|---|---|
| Allocations per insert | 1 (node) | 0 (bulk-allocated) |
| Cache lines per lookup | 2-3 (bucket + chain walk) | 1 (SIMD group) |
| Max load factor | ~1.0 (unbounded chains) | 0.75-0.875 then resize |
| Pointer stability | yes | no (rehash relocates) |
| Memory/entry overhead | 2-3 ptrs + node padding | 1 control byte |
| Typical throughput | 0.3-0.5x | 1.0x |

Open addressing wins because **memory latency dominates**: ~80-300 cycles to DRAM vs ~4 to L1. One saved indirection is the single biggest hash-table lever. For pointer stability, use `absl::node_hash_map` / `F14NodeMap` — same SIMD control array, nodes separate.

**Default:** open addressing. Chaining only if refs must survive inserts.

---

## 2. Pattern Classification

| Family | Representative | Probe | Max load | Delete | Cache |
|---|---|---|---|---|---|
| Chaining | `std::unordered_map` | chain walk | 1.0+ | easy | poor |
| Linear | `rigtorp::HashMap`, `emhash` | +1 | 0.5-0.9 | backshift | great |
| Quadratic | Java 8 `HashMap`, Python `dict` | i² | 0.75 | tombstone | good |
| Double hashing | F14 | h2(k) step | 0.857 | auto-recycled | good |
| **Robin Hood** | `ska::flat_hash_map`, `ankerl::unordered_dense` | linear + steal-from-rich | 0.9+ | backshift | great, low variance |
| Hopscotch | research / `tsl::hopscotch_map` | bounded neighborhood | 0.9 | swap | moderate |
| **SwissTable** | `absl::flat_hash_map`, Rust `hashbrown` | quadratic over 16-groups | 0.875 | tombstone | excellent (SIMD) |
| **F14** | `folly::F14FastMap` | double over 14-chunks | 0.857 | overflow counter | excellent (SIMD) |
| Cuckoo | libcuckoo, MemC3 | 2 tables, 2 hashes | 0.95+ (4-way) | trivial | moderate |
| **MPHF** (static) | PTHash, RecSplit, BBHash | none (perfect) | n/a | none | excellent |

Bold = usually what you want in that family.

---

## 3. SwissTable — `absl::flat_hash_map`, Rust `hashbrown`

### 3.1 Control byte layout

```
hash(k) = | H1 (57 bits → position) | H2 (7 bits → fingerprint) |

slots:    [k0 v0][k1 v1]...[kN vN]
control:  [ c0 ][ c1 ]...[ cN ]     1 byte per slot
  empty   = 0x80
  deleted = 0xFE
  full    = 0b0HHHHHHH (H2 fingerprint)
```

### 3.2 SIMD probe — 16 slots per instruction

```cpp
__m128i ctrl  = _mm_loadu_si128((__m128i*)(control + group));
__m128i match = _mm_cmpeq_epi8(ctrl, _mm_set1_epi8(h2));
uint32_t mask = _mm_movemask_epi8(match);
while (mask) {
    int i = __builtin_ctz(mask);
    if (slots[group + i].key == key) return &slots[group + i];
    mask &= mask - 1;
}
```

One `pcmpeqb` filters 16 slots. Key compare runs only on fingerprint match (~1/128 false-positive per slot). ARM NEON has the equivalent `vceqq_u8`.

### 3.3 Load factor × probe length (Abseil reported)

| Load | Avg groups (hit) | Avg groups (miss) | P99 groups |
|---|---|---|---|
| 0.50 | 1.00 | 1.00 | 1 |
| 0.75 | 1.03 | 1.15 | 2 |
| 0.875 (resize trigger) | 1.08 | 1.44 | 3 |

At max load: typically 1 cache line of control bytes + 1 of slots per lookup. Resize at 7/8 → post-resize load = 7/16.

### 3.4 Memory formula

```
bytes_per_entry = sizeof(K) + sizeof(V) + 1 (control byte)
total           = entries × (sizeof(K) + sizeof(V) + 1) / load_factor

eg: flat_hash_map<int64,int64>, 10M entries, load 0.7
    → 10e6 × 17 / 0.7 ≈ 243 MB
vs std::unordered_map<int64,int64>: ~48 B/entry → ~480 MB
```

### 3.5 Where it ships

- `absl::flat_hash_map` / `flat_hash_set` — Google internal default.
- Rust `hashbrown` is the same design. **Since Rust 1.36 (2019) `std::collections::HashMap` is hashbrown** (with SipHash-1-3 hasher).
- Boost 1.81+ ships `boost::unordered_flat_map` (SwissTable-family).

`flat_hash_map` = inline `(K,V)`, no ref stability. `node_hash_map` = table of heap pointers, ref stable, one extra indirection.

---

## 4. F14 — `folly::F14FastMap`

Same idea as SwissTable, different packing: **14 entries per chunk**, metadata inline.

### 4.1 Chunk layout

| Bytes | Contents |
|---|---|
| 0..13 | 7-bit tag per key (top bit set) |
| 14 | overflow_count (spilled collisions) |
| 15 | control byte |
| 16..N | aligned `(K,V)` slots |

`_mm_cmpeq_epi8` over the 14 tags. The overflow counter replaces tombstones — decremented on delete, so churn doesn't accumulate scars.

### 4.2 Probe length (F14 paper)

```
Load 12/14 (max):
  hit:  1.04 chunks avg
  miss: 1.275 chunks avg, P99 = 4
```

### 4.3 Variants — the reason to pick F14 over SwissTable

| Variant | Layout | Memory/entry | Iter | Ref stable |
|---|---|---|---|---|
| `F14ValueMap` | inline `(K,V)` in chunk | `sizeof(K)+sizeof(V)+~1.5B` | moderate | no |
| `F14NodeMap` | chunk → heap node ptrs | +16 B | slow | **yes** |
| `F14VectorMap` | chunk → `uint32_t` idx into contiguous value vector | meta + vector | **fastest** | no |
| `F14FastMap` | auto-picks Value (<24 B) or Vector (≥24 B) | adaptive | adaptive | no |

### 4.4 F14 vs SwissTable

| | SwissTable | F14 |
|---|---|---|
| Group size | 16 | 14 |
| Max load | 87.5% | 85.7% |
| Probing | quadratic | double hashing |
| Tombstones | yes | auto-recycled |
| Variants | flat, node | Fast / Value / Node / Vector |
| Wins on | broad ecosystem | churn-heavy, hot iteration |

Public benchmarks (Martinus, Tessil) put them within 10-20%. Pick on variant needs (F14 Vector for iteration; SwissTable for ecosystem). See §15.

---

## 5. Robin Hood — `ska::flat_hash_map`, `ankerl::unordered_dense`

Linear probing + one invariant: on insert, if the slot holds a key with **lower probe-sequence length (PSL)** than ours, swap — the richer resident gets pushed forward.

**Why it matters:**
- PSL variance collapses → P99 latency flat instead of spiky.
- **Backshift deletion** — no tombstones, no periodic cleanup rehash.
- Works well at **load 0.9+**, denser than SwissTable.

```cpp
for (size_t i = h(k) % capacity, psl = 0; ; ++i, ++psl) {
    if (slot[i].empty()) { slot[i] = {k,v,psl}; return; }
    if (slot[i].psl < psl) std::swap(slot[i], candidate); // rob the rich
    if (slot[i].key == k) { slot[i].v = v; return; }
}
```

Libs: `ska::flat_hash_map` (Skarupke reference), `ankerl::unordered_dense` (2022+, current Martinus leader on simple keys).

TRADEOFF: no SIMD parallel probe → loses to SwissTable/F14 when large keys make fingerprint filtering matter.

---

## 6. Cuckoo Hashing (the table, not the filter)

Pagh & Rodler (2001). Two hashes `h1, h2`, two tables. Every key lives in exactly one of 2 slots → **O(1) worst-case lookup**.

```
Lookup(k):  check slot[h1(k)] OR slot[h2(k)]  — always 2 reads
Insert(k):  place at h1; if occupied, evict resident, place k, reinsert evictee
            at its alternate slot; chain bounded → rehash on cycle
```

| Scheme | Max load | Lookup slots |
|---|---|---|
| 2-way, 1 slot/bucket | 0.50 | 2 |
| 2-way, 4 slots/bucket | **0.95** | 8 |
| 4-way, 1 slot/bucket | 0.97 | 4 |

Wins for concurrent KV: 2 independent reads, easy bucket-level locking. **libcuckoo** (CMU) ships optimistic cuckoo with fine-grained locks, used in MemC3 (Fan et al., NSDI 2013).

TRADEOFF: insert has occasional long eviction chains → tail-latency spike. Not great single-threaded.

**Cuckoo *filter* (membership-only) lives in ch27 §6.** Same idea, no value stored; 7 bits/key.

---

## 7. Perfect Hashing / MPHF — Static Sets

Build-time decision: if the keys never change, collapse lookup to one hash + one array access. MPHF = bijection `N keys → [0, N)`.

| Library | Bits/key | Build (1M keys) | Query | Notes |
|---|---|---|---|---|
| CHD (Belazzougui 2009) | ~2.6 | moderate | fast | classic |
| BDZ | ~3.0 | fast | fast | hypergraph |
| **RecSplit** (Esposito, Graf, Vigna 2020) | **1.8** | slow | fast | smallest |
| BBHash (Limasset 2017) | ~3.0 | fast, parallel | fast | bioinformatics scale |
| **PTHash** (Pibiri 2021) | 2.7-3.2 | fast, parallel | **fastest** | current SOTA query |
| gperf | many | slow | fastest (codegen) | compile-time tiny sets |
| Frozen | many | compile-time | fastest | C++ `constexpr` |

Tradeoff triangle: smallest mem = RecSplit; fastest query on millions = PTHash; compile-time tiny = Frozen/gperf; billions + parallel build = BBHash.

**Where MPHFs show up:** compilers (keyword tables), DNS zone loaders, search engine term dictionaries, Kafka partition tables when static, HFT instrument-code → ID (ch26 §4.3), language builtin symbol tables.

---

## 8. Hash Function Choice

### 8.1 HashDoS history

2011, Klink & Wälde (28c3): PHP, Python, Java, Ruby all had predictable string hashes → attacker-crafted POST pinned a core for minutes. Python shipped SipHash in 3.4 (2014). Java 8 tree-ifies any bucket with >8 collisions (`TREEIFY_THRESHOLD`). Rust `std::HashMap` ships SipHash-1-3 by default — this is why idiomatic Rust code swaps to `ahash` / `foldhash` for internal maps.

### 8.2 Comparison

| Hash | Speed (short keys) | DoS safe | Where default |
|---|---|---|---|
| `std::hash` (libstdc++) | ~2-3 GB/s | no | C++ std |
| MurmurHash3 | ~4 GB/s | no | older Guava, Kafka |
| CityHash / FarmHash | ~6 GB/s | no | Google internal (pre-Abseil) |
| XXH3 (Collet 2019) | **~30 GB/s** | no | LZ4, ClickHouse |
| wyhash (Wang 2020) | ~20 GB/s | no | `emhash`, `ankerl::unordered_dense` |
| aHash | ~15 GB/s | weakly (AES-NI) | Rust ecosystem (opt-in) |
| **foldhash** (2024) | ~20 GB/s | weakly | Rust, current fastest default |
| **SipHash-1-3** | ~1-2 GB/s | **yes** | Rust std, Python, Ruby, Perl |
| HighwayHash | ~10 GB/s | yes | adversarial + fast |

Throughput numbers are "typical" order of magnitude from Collet / Lemire microbenchmarks — treat as relative.

### 8.3 Decision

```
Input attacker-controlled (HTTP keys, public JSON, user IDs from internet)?
  YES → SipHash-1-3 or HighwayHash
  NO  → Hash cost in a hot loop?
          YES → foldhash (Rust), wyhash or XXH3 (C++)
          NO  → library default is fine
```

**Common mistake:** swapping Rust's default hasher for `ahash` in a user-facing service → HashDoS re-enabled. Keep SipHash on the adversarial boundary; use fast hashers only for internal, trusted-key maps.

---

## 9. Resize Strategies — the P99 Killer

A naive `insert()` that triggers rehash blocks for O(n). 100M entries = seconds of stall. Three patterns:

### 9.1 `reserve()` before bulk insert

```cpp
absl::flat_hash_map<K,V> m;
m.reserve(expected_n);  // one allocation, no mid-insert rehash
```
```rust
let mut m = HashMap::with_capacity(expected_n);
```

Sizing: `capacity = ceil(expected_n / target_load)`. SwissTable resizes at 7/8, so aim ≤ 0.7 for headroom.

### 9.2 Incremental / dual-table (Redis `rehashidx`)

```
dict.ht[0] = old, dict.ht[1] = new (2x, empty), dict.rehashidx = 0
on each op:
  migrate ht[0].buckets[rehashidx] → ht[1]; rehashidx++
  if rehashidx == ht[0].size: swap tables, rehashidx = -1
lookup: check both tables during migration
```

Zero stall, constant factor on lookups while migrating. Used in Redis, Aerospike, ScyllaDB cache layers.

### 9.3 When to care

Single-shot build-then-query → `reserve()` is enough. Latency-sensitive online growth → dual-table or pre-shard. Very large maps (>10 GB) → combine with **hugepages** (ch15): 2 MB pages cut TLB miss cost ~512x during full-table rehash scan. Allocate with `madvise(MADV_HUGEPAGE)`.

---

## 10. Concurrent Hash Maps

Single-threaded throughput is not the goal — the question is which contention profile you want.

| Library | Lang | Scheme | Wins |
|---|---|---|---|
| `java.util.concurrent.ConcurrentHashMap` | Java | CAS + bucket lock, tree-ify | JVM default, solid everywhere |
| `folly::ConcurrentHashMap` | C++ | SIMD chunk + fine-grained lock | Meta production |
| `folly::AtomicHashMap` | C++ | fixed-size, lock-free linear | bounded maps |
| `tbb::concurrent_hash_map` | C++ | per-bucket RW accessor | Intel TBB |
| **`DashMap`** | Rust | N shards × `RwLock<HashMap>` | Rust de-facto default |
| `scc::HashMap` | Rust | lock-free, async-aware | async-heavy contention |
| Cliff Click `NonBlockingHashMap` | Java | lock-free CAS state machine | legendary HFT Java |
| libcuckoo | C++ | optimistic cuckoo + bucket lock | read-mostly, O(1) worst-case |

### 10.1 DashMap trap

```rust
let m: DashMap<K, V> = DashMap::new();     // default: ncpus * 4 shards
if let Some(r) = m.get(&k) { /* r holds the shard's read lock */ }
```

Holding a `Ref` / `RefMut` / `entry()` guard across code that touches the same shard → deadlock or serialized writes on hot keys.

### 10.2 Scheduler pathologies (ch16)

- **Lock convoy** on hot shard → `perf sched latency` skewed, one core pegged. Fix: more shards, key re-hashing.
- **False sharing** on adjacent control bytes → `perf c2c` HITM. Fix: pad shards to 64 B cache line.
- **Priority inversion** under RT threads → `PTHREAD_PRIO_INHERIT` or keep RT threads out of the shared map path.

---

## 11. Decision Matrix

| Workload | Pick |
|---|---|
| C++ general-purpose | `absl::flat_hash_map` |
| C++ small values, hot iteration | `folly::F14VectorMap` |
| C++ pointer-stable refs needed | `absl::node_hash_map` / `F14NodeMap` |
| C++ churn-heavy (delete-heavy) | `rigtorp::HashMap` (backshift) or F14 |
| Rust general-purpose, trusted keys | `HashMap<_, _, foldhash::fast::RandomState>` |
| Rust adversarial input | std `HashMap` (SipHash-1-3 default) |
| Rust concurrent | `DashMap` (most cases), `scc::HashMap` (async-heavy) |
| Java | `ConcurrentHashMap` (also for single-threaded — Java 8 tree-ify is good enough) |
| Static key set, fast query | **PTHash** |
| Static key set, min memory | **RecSplit** |
| Static keys at C++ compile time | **Frozen** or `gperf` |
| Concurrent KV, read-mostly, tight P99 | **libcuckoo** |
| Huge map + P99 matters | Redis dual-table or pre-shard |

### Pick-by-workload

```
1. Static key set (never changes post-build)?
   YES → §7: PTHash (query) / RecSplit (mem) / Frozen (compile-time)
   NO  → 2.

2. Concurrent from many threads?
   YES → §10: DashMap (Rust), ConcurrentHashMap (Java),
              folly::ConcurrentHashMap or libcuckoo (C++)
   NO  → 3.

3. Attacker-controlled input?
   YES → DoS-safe hasher (SipHash/Highway) + any open-addressing table
   NO  → 4.

4. Language default:
   C++    → absl::flat_hash_map + wyhash (or F14FastMap if folly is linked)
   Rust   → HashMap<_, _, foldhash::fast::RandomState>
   Java   → HashMap (Java 8+ tree-ify handles collisions)
   Python → dict (SipHash built-in)
   Go     → map (AES-NI seeded, runtime DoS mitigation)
```

---

## 12. Integration With Other Chapters

- **ch14 Database Profiling** — Redis `dict` is §9.2 exactly. PostgreSQL `simplehash.h` = linear-probing template used by hash joins, aggregation, subplan hash. ClickHouse has a vectorized variant for `GROUP BY`.
- **ch15 Memory Subsystem** — hugepages for maps > a few GB (2 MB / 1 GB pages cut TLB miss cost). NUMA-pin the table to the socket of the owning threads; `numastat -m <pid>` shows remote-node hits rising with map size if not.
- **ch16 Scheduler / Interrupts** — concurrent map contention manifests as migration stalls and `perf c2c` HITM ping-pong. Fix: more shards, padding, per-thread shards + merges.
- **ch19 Storage Engine Patterns** — RocksDB `HashLinkList` memtable and PG/CH hash-join operators are the same decision surface at larger granularity.
- **ch26 C++ HFT** — §4 there is the prescription (SwissTable / F14 / Frozen). This chapter is the reference.
- **ch27 Compact Integer Sets** — Cuckoo *filter* in §6 there (membership only, ~7 bits/key). Same idea as Cuckoo *hashing* in §6 here (full key→value).

---

## 13. Libraries (2026)

### C++

| Library | Notes |
|---|---|
| `absl::flat_hash_map` / `node_hash_map` | [abseil-cpp](https://github.com/abseil/abseil-cpp), canonical SwissTable |
| `folly::F14FastMap` / Value / Node / Vector | [Folly](https://github.com/facebook/folly) |
| `boost::unordered_flat_map` | Boost 1.81+, SwissTable-family, std-compatible API |
| `ankerl::unordered_dense` | header-only, current Martinus-bench leader |
| `ska::flat_hash_map` | Skarupke's Robin Hood reference |
| `emhash` | many variants, benchmark-tuned |
| `rigtorp::HashMap` | backshift deletion, HFT-oriented |
| `tsl::robin_map`, `tsl::hopscotch_map` | Tessil benchmark reference impls |
| libcuckoo | [efficient/libcuckoo](https://github.com/efficient/libcuckoo) |
| Frozen | [serge-sans-paille/frozen](https://github.com/serge-sans-paille/frozen), constexpr PHF |
| gperf | GNU, system package |

### Rust

```bash
cargo add hashbrown    # SwissTable (what std uses internally)
cargo add dashmap      # sharded concurrent map
cargo add scc          # async-aware concurrent map
cargo add ahash        # AES-NI fast hasher (not safe for public keys)
cargo add foldhash     # 2024, current fastest Rust default-safe hasher
```

```rust
use foldhash::fast::RandomState;
let mut m: HashMap<K, V, RandomState> = HashMap::default();
```

### Perfect hashing

| Library | Notes |
|---|---|
| [PTHash](https://github.com/jermp/pthash) | C++, fastest query |
| [RecSplit](https://github.com/vigna/sux) (in `sux`) | C++, ~1.8 bits/key |
| [BBHash](https://github.com/rizkg/BBHash) | parallel build, bioinformatics |
| [CMPH](https://cmph.sourceforge.net/) | C, CHD/BDZ, many bindings |
| `boomphf` | Rust BBHash |

---

## 14. Diagnostic Patterns

### 14.1 "The hash table is slow"

```
1. perf stat -e cache-misses,cache-references <workload>.
   > 30% miss rate on hot path → already open addressing? check load next.

2. Load factor.
   unordered_map: bucket_count()/size() — often 0.5-1.0
   flat_hash_map: load_factor(), resizes at 0.875
   too high → reserve() more; too volatile → pre-size.

3. Hash quality.
   String keys: swap std::hash → wyhash/XXH3/foldhash → +30-40%.
   Custom structs: common bug is hashing only id_ while eq checks all fields
   → pathological collisions.

4. Value size.
   sizeof(V) ≫ 24 B and you iterate often → F14VectorMap or indirection.
   Large V splits cache lines in flat tables.

5. Insert P99 spikes = rehash. reserve() or dual-table (§9.2).
```

### 14.2 "HashDoS suspected"

Symptoms: one endpoint's CPU pegs on tiny request; profile = all time in bucket walks / tree-ified nodes. Check: was default hasher swapped to a non-keyed one (`ahash` unseeded, `wyhash`, XXH3) on user-facing input? Fix: SipHash-1-3 / HighwayHash at the adversarial boundary.

### 14.3 "Table memory too big"

```
Predict: entries × (sizeof(K)+sizeof(V)+metadata) / load_factor
Actual ≈ 2× predicted → std::unordered_map per-node malloc overhead.
Migrate to flat_hash_map / F14 → typically halves RSS.
Large duplicated V → intern via MPHF → canonical ptr, often 5-10x.
```

### 14.4 "Concurrent map doesn't scale past N cores"

```
perf c2c HITM on control array → false sharing. Pad shards to 64 B.
perf sched latency skewed → too few shards or skewed keys. Increase shard count.
Long-held DashMap guards / TBB accessors → tighten scope.
99% reads → libcuckoo or RCU with periodic snapshot rebuild.
```

---

## 15. Benchmark Caveats

Public benchmarks are famously misleading.

- **Martinus hashtable-bench** (reused by `ankerl`, `robin_hood`) uses uniform `u64→u64`. Real workloads have skewed keys, large values, strings, mixed ops. A +20% Martinus winner may lose on your workload.
- **Tessil's suite** tests more shapes (strings, churn, mixed) and generally ranks the top libraries within ~15% of each other.

**Practical rule:** in SwissTable / F14 / `ankerl::unordered_dense` / `emhash` territory, differences are workload-specific. Pick on *variant needs* (pointer stability, iteration, delete semantics), not benchmark rank. If you can't benchmark your own workload, pick `absl::flat_hash_map` (C++) or `hashbrown`/std (Rust) — you won't be > 20% off.

---

## 16. Recent Hash-Table Structures (2025–2026)

### 16.1 History-Independent Concurrent Robin Hood Table

- **Structure:** Robin Hood open-addressing table where **internal layout depends only on the set of keys, not the insertion order**. Uses LL/SC (load-linked / store-conditional) primitives — lock-free, linearizable. Stores 2 elements + 2 bits per memory cell.
- **Pick it when:** **multi-tenant or adversarial workloads** where insertion-order leaks matter — side-channel defense, deterministic testing, replicated state machines that must converge bit-for-bit.
- **How:** arXiv:2503.21016 reference impl. Not in mainline libraries yet (2026). If you don't need history-independence, **regular Robin Hood or SwissTable remains simpler and faster**.

### 16.2 PTHash-HEM — fast-build perfect hashing

- **Structure:** Minimal perfect hash function over a static key set. **HEM variant** (Hash-Evaluate-Map) uses ~2 bits/key, builds in sub-second time on 100M keys. ~10× faster build than the 2020 PTHash original.
- **Pick it when:** **static symbol tables** — compiler keywords, known protocol opcodes, DNS TLD list, Kafka partition assignment where partition count is fixed, URL routing tables, gperf replacements. Build offline, query at ~1 ns/lookup.
- **How:** [`jermp/pthash`](https://github.com/jermp/pthash) C++ header-only.
  ```cpp
  pthash::build_configuration config;
  config.c = 6.0;           // bucket size ~ c / log(n)
  config.alpha = 0.94;      // load factor
  pthash::single_phf<pthash::murmurhash2_64> f;
  f.build_in_internal_memory(keys.begin(), keys.size(), config);
  uint64_t idx = f(query_key);    // ~1 ns on modern CPU
  ```
- **When to beat SwissTable with PTHash:** static key sets > 10k elements. Below that, SwissTable's dynamism wins because build-time matters less.

### 16.3 `boost::unordered_flat_map` — the 2025 "portable SwissTable"

- **Structure:** Open-addressing 2-way chunked (15 slots per chunk + metadata byte), SIMD-friendly probing. Anti-drift mechanism: auto-rehash when repeated insert/erase degrades probe length. Max load factor 0.875 fixed.
- **Pick it when:** **new C++ code, STL-compatible interface desired, no Abseil dependency**. Boost ≥ 1.82 gives you Abseil-level perf without Google's build system.
- **How:** `#include <boost/unordered/unordered_flat_map.hpp>` (Boost ≥ 1.81). API is the `std::unordered_map` subset (no node extraction, no bucket API).
- **Vs `absl::flat_hash_map`:** roughly tied on microbenchmarks; `boost::` wins portability (vendored with Boost), `absl::` wins if you're in Google's ecosystem already.

### 16.4 Pick matrix — hash structures in 2026

| Workload | Structure | Lib / config |
|---|---|---|
| Default C++ hash map | **`boost::unordered_flat_map`** or **`absl::flat_hash_map`** | Boost 1.82+ / Abseil |
| Default Rust hash map | **`std::HashMap`** (already hashbrown/SwissTable) + `ahash` or `foldhash` | `use ahash::AHashMap;` |
| Static keys, 10k+ elements | **PTHash-HEM** | `jermp/pthash` — build offline, ~2 bits/key |
| Static keys, <10k or tiny | SwissTable-family | not worth MPHF build complexity |
| Multi-thread contended | **`folly::ConcurrentHashMap`** (C++) / **`DashMap`** (Rust) | shard-striped locking |
| Adversarial or multi-tenant | **SipHash-2-4** as hash function | Rust std default; C++ must opt-in |
| Insertion-order leak is a threat model | **History-Independent Robin Hood** (research) | arXiv:2503.21016 |
| Large values, reference stability needed | **`absl::node_hash_map`** / `F14NodeMap` | node_hash variants |

**What stayed true:** for 99% of cases, just use your language's default hash map. The 2025 research adds niche wins (PTHash for static sets, history-independence for adversarial settings), not a new default.

---

## 17. References

- Kulukundis, M. (2017). *Designing a Fast, Efficient, Cache-friendly Hash Table, Step by Step*. CppCon. [talk](https://www.youtube.com/watch?v=ncHmEUmJZf4)
- Abseil Team. [Swiss Tables Design Notes](https://abseil.io/about/design/swisstables).
- Bronson, N., & Shi, X. (2019). *Open-sourcing F14*. [Engineering at Meta](https://engineering.fb.com/2019/04/25/developer-tools/f14/).
- Skarupke, M. (2017). *I Wrote The Fastest Hashtable*. [probablydance.com](https://probablydance.com/2017/02/26/i-wrote-the-fastest-hashtable/).
- Pagh, R., Rodler, F. (2001). *Cuckoo Hashing*. ESA.
- Fan, B., Andersen, D., Kaminsky, M. (2013). *MemC3*. NSDI.
- Li, X. et al. (2014). *Algorithmic Improvements for Fast Concurrent Cuckoo Hashing* (libcuckoo). EuroSys.
- Belazzougui, D. et al. (2009). *Hash, displace, and compress* (CHD). ESA.
- Esposito, E., Graf, T. M., Vigna, S. (2020). *RecSplit*. ALENEX.
- Pibiri, G. E., Trani, R. (2021). *PTHash*. SIGIR.
- Limasset, A. et al. (2017). *Fast and Scalable MPHF for Massive Key Sets* (BBHash). SEA.
- Klink, A., Wälde, J. (2011). *Efficient DoS Attacks on Web Application Platforms*. 28c3.
- Aumasson, J.-P., Bernstein, D. J. (2012). *SipHash*. INDOCRYPT.
- Köppl, D. (2019). *Separate Chaining Meets Compact Hashing*. arXiv:1905.00163.
- Click, C. (2007). *A Lock-Free Hash Table*. JavaOne.
- Lemire, D. — [lemire.me](https://lemire.me/blog/) — hash function + table microbenchmarks.
