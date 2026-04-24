# Learned Indexes: RMI, PGM, ALEX, LIPP, Learned Bloom

Index = model that maps `key → position`. B-tree does it with splits; a learned model does it with arithmetic. Since Kraska et al. (SIGMOD 2018) the field has oscillated between "replaces B-trees" hype and sober benchmarks. This chapter gives the honest 2026 picture: where learned indexes actually win, where they lose, and how to recognise a workload that fits.

Primary sources: Kraska et al. *The Case for Learned Index Structures* (SIGMOD 2018), Ferragina & Vinciguerra *PGM-index* (VLDB 2020), Ding et al. *ALEX* (SIGMOD 2020), Wu et al. *LIPP* (VLDB 2021), Galakatos et al. *FITing-tree* (SIGMOD 2019), Marcus et al. *SOSD* (VLDB 2020).

---

## 1. The Core Idea

Sorted array `A[0..n-1]` induces `F(key) = position of key in A` = CDF scaled by `n`. B-tree encodes `F` as splits, `O(log n)` to locate. Learned index encodes `F` as a parametric model `f(k) ≈ F(k)`.

```
predict:    pos_hat = f(key)                            # O(1)
guarantee:  |f(key) - F(key)| ≤ ε  ∀ key ∈ A            # bound at build
last-mile:  binary search A[pos_hat-ε .. pos_hat+ε]     # O(log ε)
```

Total: `O(model) + O(log ε)`. If data has exploitable structure, `f` is a piecewise linear function of tiny size and `ε` is small (tens to hundreds). Model replaces upper B-tree levels; last-mile replaces leaves.

**Three things this never does well:**
1. Random/adversarial keys → ε explodes, last-mile dominates.
2. Updates that violate the fitted CDF → error bound breaks or retrain.
3. Worst-case latency guarantees — ε is average-ish; pathological keys live at the boundary.

---

## 2. Family Classification

| Approach | Build | Query | Memory | Mutability | Dataset assumption |
|---|---|---|---|---|---|
| B+tree (reference) | O(n log n) | O(log n), 3-5 cache misses | ~O(n), 30-60% fill | full | none |
| ART (ch27, ref.) | O(n·key_len) | O(key_len) | key-shape dep. | full | shared prefixes help |
| **RMI** (Kraska 2018) | O(n) 1-pass | O(1) + O(log ε) | **< 1 MB** typical | static, rebuild | sortable, learnable CDF |
| **PGM** (2020) | O(n) streaming | O(log_c n) + O(log ε) | O(n/ε) segments | static (PGM-Dyn mutable) | sortable numeric |
| **ALEX** (2020) | O(n) + gaps | O(log ε) amortised | 1.5-2× B-tree | **mutable** | near-stationary CDF |
| **LIPP** (2021) | O(n) | **no last-mile**, O(1)/level | ~B-tree | mutable | exact position |
| FITing-tree (2019) | O(n) | O(log ε) | tiny, tunable | static | superseded by PGM |
| Learned Bloom | O(n) + train | O(model) + backup | smaller **if** fits | static | stable key/non-key |

---

## 3. RMI — Recursive Model Index (Kraska 2018)

Staged hierarchy. Top model routes to submodels; submodels predict position with local ε.

```
 [root f₀] → [f₁,₀ f₁,₁ … f₁,k₁] → … → leaf f_L,i  (per-model εᵢ)
                                       ↓
                binary search A[pos_hat-εᵢ .. pos_hat+εᵢ]
```

Paper shapes: 2-stage. Stage 0 = small NN (2×32) or linear. Stage 1 = 10⁴–2·10⁵ linear regressors. ε stored per submodel.

Reported (Kraska et al., *Weblogs* / *Maps* / *Lognormal*, 200 M integer keys): 1.5–3× faster than cache-optimised B-tree, index 10–100× smaller, lookups in tens of ns.

Python fitting intuition:

```python
# Stage 0 global linear; Stage 1 per-leaf linear + ε = max|pred-pos|.
from sklearn.linear_model import LinearRegression; import numpy as np
def fit_rmi(keys, K=1000):
    keys=np.sort(keys); pos=np.arange(len(keys))
    root=LinearRegression().fit(keys.reshape(-1,1),pos)
    leaf=np.clip((root.predict(keys.reshape(-1,1))/len(keys)*K).astype(int),0,K-1)
    leaves={}
    for lid in range(K):
        m=leaf==lid
        if m.sum()<2: continue
        lm=LinearRegression().fit(keys[m].reshape(-1,1),pos[m])
        eps=int(np.abs(lm.predict(keys[m].reshape(-1,1))-pos[m]).max())
        leaves[lid]=(lm,eps)
    return root, leaves
```

Caveats: **static** (mutation invalidates ε); non-trivial training time for NN root; ε is a training max → drifted keys land outside; **the 2018 paper overclaimed** — SOSD shows the B-tree gap closes to <2× once both are equally cache-optimised and integer-only, and on strings/skewed/multi-dim B-tree regains the lead.

---

## 4. PGM-index — Piecewise Geometric Model (Ferragina, Vinciguerra 2020)

Piecewise linear approximation of the CDF with **strict** max-error `ε`:

```
given sorted k₀..k_{n-1}, target ε:
  greedily extend segment [kᵢ..kⱼ] while
    ∃ a,b: ∀ p∈[i..j]: | a·k_p + b - p | ≤ ε
  emit (k_i, slope, intercept); restart at k_{j+1}
```

`m ≪ n` segments. Recurse on segment-start keys → fixed branching `c ≈ 2ε`. Space O(n/c), query O(log_c n) + bounded binary search. Cleaner than RMI: hard deterministic bound, streaming O(n) build, single knob ε.

Reported (200 M integer keys): ~83× smaller than FAST/B-tree at comparable latency; competitive/faster than RMI; sub-µs lookups. SOSD ranks PGM top-2 with RMI — PGM wins on size, RMI on raw lookup when the CDF is NN-learnable.

C++ usage (`gvinciguerra/PGM-index`):

```cpp
#include "pgm/pgm_index.hpp"
constexpr int Epsilon = 64, EpsilonR = 4;
pgm::PGMIndex<uint64_t, Epsilon, EpsilonR> idx(sorted_keys);
auto r  = idx.search(target);    // {lo, pos, hi}
auto it = std::lower_bound(sorted_keys.begin()+r.lo,
                           sorted_keys.begin()+r.hi, target);
// r.hi - r.lo ≤ 2·Epsilon + 1 → last-mile is 1-2 cache misses when hot
```

**Default PGM over RMI:** hard bound, header-only C++17, no tuning. Use RMI only if a staged NN root clearly wins on *your* CDF — rare; verify with SOSD first.

---

## 5. ALEX — Mutable Learned Index (Microsoft, SIGMOD 2020)

Point: **inserts without full rebuild**, keeping most of the learned-index win.

Tricks: (a) gapped arrays at leaves — model predicts into an array with empty slots, insert amortised O(1); (b) model-based inserts, local shift or split if target gap full; (c) adaptive nodes with model-based directories, split/merge on insert pressure; (d) per-node cost models decide split/expand/rebuild.

Reported numbers (Ding et al., 200 M keys, YCSB-like): read-only matches/beats B-tree, within ~2× of static RMI; 95/5 read-heavy faster than B+tree and smaller; 50/50 write-heavy parity with B+tree, still smaller. **Adversarial updates (CDF-breaking): B+tree catches up; ALEX splits frequently.**

```cpp
#include "alex.h"                        // microsoft/ALEX, header-only
alex::Alex<uint64_t, uint64_t> idx;
for (auto& [k,v] : kv) idx.insert(k, v);
auto it = idx.find(key);
auto lo = idx.lower_bound(k_lo), hi = idx.upper_bound(k_hi);
for (; lo != hi; ++lo) scan(lo.key(), lo.payload());
```

Caveats: error bounds are soft (guarantees come from split/expand, not fixed ε); worst-case latency spikes during reorganisation; adversarial update patterns trigger excessive splits.

---

## 6. LIPP and FITing-tree

**LIPP** (Wu et al. VLDB 2021). Removes last-mile entirely: each internal node directly addresses child positions, keys placed at exact predicted positions (collisions push down a level). Mutation via per-node local rebuild. Tradeoff: memory overhead vs PGM; hotspots deepen the tree. Pick LIPP over ALEX when P99 matters more than size.

**FITing-tree** (Galakatos et al. 2019). Simpler PLA cousin of PGM; superseded in 2026 — same idea, tighter bounds in PGM.

---

## 8. Learned Bloom Filters

Kraska 2018's second proposal. Replace/augment Bloom with learned classifier `f(key) → [0,1]`, plus a backup Bloom for false negatives of the model.

```
query(k):
   if f(k) ≥ τ: return "probably in"     # model "yes"
   else:        return backup.query(k)   # Bloom over keys with f(k)<τ
```

Every true member goes through either the model (score ≥ τ) or the backup → no false negatives. FPs = model above τ for non-keys + backup FPR on the rest.

**Can win when:** key/non-key distributions well-separated in a feature space the model learns cheaply; distributions are stable; `model_bytes + backup_bloom_bytes < plain_bloom_bytes` at equal FPR.

**Rarely ships because:** drift → observed FPR climbs; needs representative non-member distribution at training time (ML problem); Bloom/Cuckoo/XOR/Binary Fuse already do ~9 bits/elem at 0.4-1% FPR (ch27), learned must beat that *including* the model; Mitzenmacher (2018), Dai & Shrivastava (2020, Ada-BF) show original analysis under-counted model size.

### 8.1 Comparison vs ch27

| Filter | Bits/elem @ 1% FPR | FPR floor | Drift-sensitive | Production |
|---|---|---|---|---|
| Bloom | ~9.6 | tunable | no | ubiquitous |
| Cuckoo | ~7 | ~0.5% | no | RocksDB modern |
| XOR / Binary Fuse | ~9–9.8 | 0.39% | no | CDN neg caches |
| Ribbon | ~7–8 | tunable | no | RocksDB optional |
| **Learned Bloom** | workload-dependent | depends on τ | **yes** | rare, research |

**2026 default:** Binary Fuse (immutable), Cuckoo (mutable). Learned Bloom only when (a) you own training data, (b) distribution is stable, (c) retraining + monitoring budgeted, (d) benchmarking confirms the saving.

---

## 9. Error Bound ε — Dominant Tuning Knob

For PGM / FITing-tree / ALEX-like:

```
Build:     O(n) regardless of ε (streaming PLA)
Segments:  m ~ O(n/ε), decreases ~linearly in ε
Size:      ~m · (key + 2 coeffs)
Last-mile: O(log 2ε), 1-2 cache lines at small ε, cold DRAM at large
```

- `ε = 8–32`: tiny index, L1 last-mile, many segments → deeper recursion.
- `ε = 64–128`: SOSD sweet spot for dense integer CDFs. L1/L2 last-mile.
- `ε = 256+`: smallest index, last-mile touches cold cache — only for tight memory budgets.

Sweep on a representative workload; measure P50 **and** P99. The ε that minimises mean often inflates P99 on skewed CDFs. Pick for P99 when it matters.

---

## 10. SOSD — Search On Sorted Data Benchmark

Marcus et al. VLDB 2020, <https://github.com/learnedsystems/SOSD>. The only way to make honest claims.

- Datasets: `books`, `wiki_ts`, `osm_cellids`, `fb`, `lognormal`, `uniform`, `normal` — 200 M keys.
- Structures: B-tree (STX), binary/interpolation search, RadixSpline, RMI, PGM, ART.
- Metrics: point-lookup ns, build ns, index size bytes.

- **Uniform/linear**: RMI/PGM beat B-tree 1.5-3×; smallest indexes.
- **Realistic skewed** (`books`, `osm`): gap narrows; B-tree within 1.2× of best learned.
- **Adversarial / heavy-tailed**: ε explodes → learned loses to a tuned B-tree or RadixSpline.
- **RadixSpline** (non-ML PLA-ish) competitive across the board and simpler — consider before RMI/PGM.

Run SOSD on your dataset before deploying.

---

## 11. Where Learned Indexes Fit in This Handbook

**Database profiling (ch14).**
- OLTP (PostgreSQL, MySQL): heavy updates, unpredictable CDFs, strong worst-case expectations. Stay with B-tree — learned indexes aren't faster after WAL, MVCC, planner stats.
- OLAP (ClickHouse, DuckDB, Snowflake internals): sorted columnar segments are immutable after write. Candidate fit for PGM / RadixSpline over dictionaries or position arrays. Research-grade, not default.
- Don't conflate "learned index" (lookup accelerator) with "learned cardinality estimator" (planner).

**Storage engines (ch19).**
- LSM SSTables — each flushed SST is sorted and immutable. Build PGM over the SST sparse index at flush time, replace block-index binary search. Bourbon (Dai et al. OSDI 2020) did this on LevelDB; ~1.5× read speedup on cache-hot workloads.
- Columnar segments (Parquet row groups, ClickHouse parts): same argument — build once at seal, query many.
- B-tree engines (LMDB, BoltDB): no fit. Mutability is the point.

**Compact sets (ch27).**
- Learned Bloom vs Bloom/Cuckoo/XOR/Binary Fuse: §8.3. Default Binary Fuse; learned only when distribution known, stable, monitored.
- Posting-list position arrays: PEF is incumbent. PGM over positions is an alternative — smaller on smooth lists, larger on bursty. PISA has experimental learned posting lists; not default.

**HFT (ch26).** NOT a fit. HFT wants deterministic worst-case sub-µs lookups. Learned indexes trade average wins for tail risk at the ε boundary. Perfect hashing (Frozen, gperf), small sorted vectors with branchless binary search, or ART are the right tools (ch26 §2, §3.2).

---

## 12. Decision Matrix

| Workload signal | Pick |
|---|---|
| Sorted, immutable, integer, exploitable CDF, memory-tight | **PGM** (RMI if NN root helps) |
| Sorted, mutable, moderate writes, stable CDF | **ALEX** or **LIPP** |
| Point-lookup P99 critical, mutable | **LIPP** |
| Strings / byte keys / composite | **ART** (ch27) — not learned |
| OLTP, unpredictable updates, worst-case | **B+tree** |
| LSM SST block index (immutable per SST) | **PGM over SST index** (experimental) |
| Set-membership, stable dist., retrain budget | **Learned Bloom** — only after benchmark vs Binary Fuse |
| Set-membership, default | **Binary Fuse** (immut.) / **Cuckoo** (mut.) |
| HFT / deterministic latency | **Perfect hash / sorted vec / ART** (ch26) |

```
Data mutable?
 ├─ NO  → learnable CDF? YES → PGM | RMI | RadixSpline
 │                        NO → B+tree | ART | sorted vector
 └─ YES → learnable AND update-stable? YES → ALEX or LIPP
                                        NO → B+tree (stay safe)
```

---

## 13. Performance Comparison — 2026

Order-of-magnitude, integer keys, 200 M entries, SOSD-style bench. Exact numbers depend on dataset.

| Structure | Build | P50 | P99 | Bytes/elem | Update | Drift tolerance |
|---|---|---|---|---|---|---|
| B+tree (STX, cache-tuned) | 1× ref | ~200-400 ns | ~500-800 ns | ~6-10 B | full | excellent |
| ART | 1-2× | ~150-300 ns | ~500 ns | variable | full | excellent |
| RadixSpline | 0.5-1× | ~80-150 ns | ~300 ns | ~1-2 B | rebuild | good (non-ML) |
| **RMI** (linear) | 0.5-1× | ~80-120 ns | ~200-500 ns | **< 1 B** | rebuild | **poor** |
| **PGM** (ε=64) | 0.5-1× | ~100-150 ns | ~250-400 ns | ~0.5-1 B | rebuild | **poor** |
| **ALEX** | 1-2× | ~150-250 ns | ~400-800 ns | ~3-5 B | full (soft ε) | **fragile** |
| **LIPP** | 1-2× | ~100-200 ns | ~300-500 ns | ~5-10 B | full | fragile |

Ranges consistent with Marcus et al. (SOSD), Ferragina & Vinciguerra, Ding et al. Not guarantees.

**Verdict.** In 2026, B-tree/ART still win in most production scenarios because of robustness to update and distribution drift. Learned indexes are worth it only for: (1) sorted static columns with smooth exploitable CDFs, (2) memory-dominant constraints, (3) build pipeline owns the data, retraining cheap, drift monitored. Outside those the 2× average speedup evaporates against higher P99 and operational cost.

---

## 14. Diagnostic Patterns

**"Queries getting slower over time."** P50 stable, P99 creeping. Cause: distribution drift. New keys fall outside the fitted ε band → last-mile becomes a full binary search; ALEX splits more. Check: (1) observed max `|pos_hat - pos|` over time, should stay ≤ ε; (2) ALEX/LIPP split/expand rate spikes = CDF shift; (3) training-time CDF quantiles vs current. Fix: rebuild (PGM/RMI) or force node reorg (ALEX) on a schedule triggered by drift metrics.

**"Learned Bloom FPR observed >> designed."** Effective FPR climbs from 1% at deploy to 5-10% later. Cause: non-member distribution changed, τ no longer separates. Check: shadow-log non-member queries, compare to training sample. Fix: retrain with fresh sample, or switch to Binary Fuse / Cuckoo if retraining is expensive.

**"Build time dominates ingest."** Rebuilding too often, or NN root in RMI. Fix: PGM (streaming O(n)); batch rebuilds by layer, not per-flush; RadixSpline — same query speed, simpler.

**"Build fine but P99 terrible."** ε too large (cold-cache last-mile) or CDF discontinuity a PLA segment spans badly. Check: distribution of `|pos_hat - pos|` per query — bimodal = discontinuity → split dataset, fit per partition. Fix: lower ε; partition keyspace; LIPP (no last-mile).

---

## 15. Libraries (2026)

| Project | Language | License | Status |
|---|---|---|---|
| [PGM-index](https://github.com/gvinciguerra/PGM-index) | C++17 header-only + Python | MIT | reference, maintained |
| [ALEX](https://github.com/microsoft/ALEX) | C++17 header-only | MIT | Microsoft Research reference |
| [LIPP](https://github.com/Jiacheng-WU/lipp) | C++ | MIT | research |
| [RMI](https://github.com/learnedsystems/RMI) | Rust compiler → C++ | MIT | code-gen, research |
| [RadixSpline](https://github.com/learnedsystems/RadixSpline) | C++ header-only | MIT | simpler PLA-ish baseline |
| [SOSD](https://github.com/learnedsystems/SOSD) | C++ | MIT | **the benchmark** |
| FITing-tree | C++ | — | superseded by PGM |

```bash
# PGM C++
git clone --depth=1 https://github.com/gvinciguerra/PGM-index.git  # -I PGM-index/include (C++17)
pip install pgm-index                                              # Python binding
# ALEX C++
git clone --depth=1 https://github.com/microsoft/ALEX.git          # -I ALEX/src/core
# SOSD — benchmark YOUR dataset before deploying
git clone --recursive https://github.com/learnedsystems/SOSD.git && cd SOSD
./scripts/download.sh && ./scripts/build.sh && ./build/benchmark <dataset>
```

---

## 16. Updatable Learned Index Structures (2025–2026)

### 16.1 LMG Index — multi-group learned index with robustness

- **Structure:** Keys are partitioned into fixed-size **"groups"** (windows of sorted keys). Each group has its own lightweight model + gapped array. A top-level **routing model** (not a full RMI) maps key → group. Within-group search = model-predict + bounded local scan.
- **Why it beats ALEX:** ALEX uses one big gapped-array per node; when predictions drift (skewed inserts), search cost explodes. LMG bounds the damage to one group; only that group retrains.
- **Pick it when:** updatable learned index in memory, **mixed read/write, unknown or shifting distribution**, need resilience to skewed insert patterns. Reports **1.41× faster than ALEX, 1.22× faster than B+Tree, 1.7× faster than Dynamic PGM after insert bursts**.
- **How:** research impl from arXiv:2512.24824. Key params: group size (typical 1024–4096 keys), error threshold ε (16–64). No mainstream prod library yet — treat as "evaluate before committing to ALEX for new 2026 code."

### 16.2 Dynamic PGM (DPGM) with sampling — external-memory winner

- **Structure:** PGM-index (piecewise linear approximation, §4) + a **sampling layer** that learns from a subset of keys (typical 1/64 or 1/256), reducing build memory footprint proportionally. Queries use the sampled PGM for narrow prediction, then binary search locally.
- **Pick it when:** **sorted keys on disk** (external-memory joins, cold-tier OLAP indexing, LSM SST position arrays). Order-of-magnitude smaller than ALEX; ALEX OOMs on 100M+ key datasets during build.
- **How:** [`gvinciguerra/PGM-index`](https://github.com/gvinciguerra/PGM-index). Use `pgm::MappedPGMIndex` for disk-resident variant:
  ```cpp
  #include "pgm/pgm_index.hpp"
  #include "pgm/pgm_index_variants.hpp"
  // ε = 16 typical, sampling rate 1/64
  pgm::PGMIndex<uint64_t, 16> idx(keys.begin(), keys.end());
  auto range = idx.search(query);            // returns predicted [lo, hi]
  auto it = std::lower_bound(&keys[range.lo], &keys[range.hi+1], query);
  ```
- **Memory:** typically 10-100× smaller than ALEX for 100M+ sorted keys.

### 16.3 UpLIF — hybrid placeholder+buffer for continuous inserts

- **Structure:** Per-model **delta buffer** for incoming inserts that don't fit the learned slot, + **placeholder slots** reserved in the key domain for future inserts. Periodic buffer-prune merges deltas into the main model; placeholder-filling triggers a smaller local retrain rather than global rebuild.
- **Pick it when:** **high insert rate on a learned index** — workload where ALEX's "filled-slot → explosive search cost" pathology would bite. E.g., continuously-ingested time series with monotonic timestamps + occasional backfill.
- **How:** arXiv:2408.04113 reference impl. Conceptually: treat any learned index + delta buffer as a mini-LSM on top of the learned structure.

### 16.4 Workload-drift-aware deployment (NeurBench lessons)

- **What it is:** not a structure — an **operational pattern**. NeurBench 2025 confirmed that XIndex/PGM/LIPP all degrade substantially under data and query drift; static deployment without monitoring is a ticking bomb.
- **Pick it when:** any learned-index production rollout.
- **How (concrete monitoring):**
  - Track **prediction error distribution** (ε_observed vs ε_configured). If p99 ε drifts >2× configured → trigger retrain.
  - Track **buffer/delta size** (for UpLIF-style). If growing monotonically → the model is no longer the right fit for incoming data.
  - Track **tail-latency ratio** (p99 / p50). Learned indexes usually have p99/p50 around 3-5×. A jump to >10× means distribution drift.

### 16.5 Decision update — when to pick learned vs classical in 2026

| Workload | Structure | Why |
|---|---|---|
| OLTP mutable in-memory, mixed r/w | **ART or B+Tree** (NOT learned) | learned cannot recover from drift fast enough; ART's Node4→Node256 auto-adaptation is already doing the right thing |
| Static sorted, large, disk-backed | **DPGM with sampling** | dominates ALEX by 10-100× memory; binary search is fast on SSD |
| Mutable but stable distribution (time-series append, monotonic IDs) | **LMG Index** (evaluate) or **ALEX** | LMG better on insert bursts; ALEX well-tested |
| High insert rate, model may break | **UpLIF-style hybrid** (learned + delta buffer) | tolerates workload drift via buffering |
| Immutable LSM SST position array | **PGM-index** or **RadixSpline** | sorted segments, static, memory-bound |
| Cold-tier columnar sorted segment | **PGM-index with sampling** | lowest memory footprint per sorted run |
| Anything with hard latency SLA (HFT, realtime) | **NOT learned** — ART/B+Tree | learned has tail-risk from retrain / drift |

**Honest verdict unchanged:** in 2026, **B-tree / ART remain the default for OLTP**. Learned indexes apply only to specific niches: static sorted data (PGM), slow-drift append-only (ALEX/LMG/UpLIF), or memory-bound external-memory scenarios (DPGM with sampling). Always ship retraining + drift monitoring with any learned-index deployment.

---

## 17. References

- Kraska et al. (2018). *The Case for Learned Index Structures*. SIGMOD.
- Mitzenmacher (2018). *A Model for Learned Bloom Filters and Optimizing by Sandwiching*. NeurIPS.
- Galakatos et al. (2019). *FITing-tree*. SIGMOD.
- Ferragina, Vinciguerra (2020). *The PGM-index: a fully-dynamic compressed learned index*. VLDB.
- Ding et al. (2020). *ALEX: An Updatable Adaptive Learned Index*. SIGMOD.
- Marcus et al. (2020). *Benchmarking Learned Indexes* (SOSD). VLDB.
- Kipf et al. (2020). *RadixSpline: A Single-Pass Learned Index*. aiDM @ SIGMOD.
- Wu et al. (2021). *Updatable Learned Index with Precise Positions (LIPP)*. VLDB.
- Dai, Shrivastava (2020). *Adaptive Learned Bloom Filter (Ada-BF)*. NeurIPS.
- Dai et al. (2020). *From WiscKey to Bourbon: a Learned Index for LSM*. OSDI.

---

## See Also

- [Compact Integer Sets](27-compact-integer-sets.md) — Bloom family, ART, Roaring; Learned Bloom comparison.
- [Storage Engine Patterns](19-storage-engine-patterns.md) — LSM SST structure, columnar segments; PGM-over-SST.
- [Database Profiling](14-database-profiling.md) — OLTP vs OLAP tradeoffs for index choice.
- [C++ HFT Optimization Patterns](26-cpp-hft-optimization-patterns.md) — why learned indexes are wrong for deterministic-latency paths.
