# Probabilistic Sketches: HLL, CMS, t-digest, DDSketch, MinHash, Theta

Sub-linear memory structures trading exactness for KB-scale sketches on cardinality, frequency, quantile, set-op and similarity queries over streams of billions. LLM-actionable: sketch-per-workload matrix, error formulas, mergeability table, diagnostics, integration with ch12/13/14/19/21/22/27.

**Why this chapter exists:** "COUNT DISTINCT over 10B events", "P99 across 1000 shards", "top-K hot keys at 10M qps", "similar docs across 1B items" all break exact algorithms. Pick wrong and P99 dashboards silently lie (coordinated omission, non-mergeable t-digest across shards), Redis HLL memory blows up (sparse mode disabled), or top-K lists invent heavy hitters. Sketches are cheap only if the error formula is respected.

Primary sources: Flajolet (HLL), Heule (HLL++), Ertl (UltraLogLog 2024), Cormode-Muthukrishnan (CMS), Metwally (Space-Saving), Dunning (t-digest), Masson/Rim/Lee (DDSketch), Karnin-Lang-Liberty (KLL), Broder (MinHash), Charikar (SimHash), Indyk-Motwani (LSH).

Complementary to **ch27 (Compact Integer Sets)**: Bloom/Cuckoo answer *"is X in S?"*; HLL/Theta answer *"how many distinct / |A ∩ B|?"*. Roaring stores the set exactly; Theta approximates set ops without the set.

---

## 1. The Baseline Problem

| Question | Exact cost | Sketch cost | Family |
|---|---|---|---|
| distinct count | `O(d)` hash set | 2–32 KB fixed | HLL, UltraLogLog |
| freq of key `k` | `O(d)` counter map | `O(ε⁻¹ log δ⁻¹)` bits | CMS, CountSketch |
| top-K | `O(d)` + sort | `O(K/ε)` counters | Space-Saving, HeavyKeeper |
| P99 | `O(n)` sorted | `O(log(εn)/ε)` | DDSketch, t-digest, KLL |
| `\|A ∩ B\|` | materialize both | `O(k)` hashes | Theta, MinHash |
| `sim(A, B)` | full shingles | 64–256 bits | MinHash, SimHash, LSH |

Rule: if you can't write the error bound on a napkin, you're using the sketch wrong.

---

## 2. Pattern Classification

| Family | Pick | Memory (typical) | Error | Mergeable | Deletable |
|---|---|---|---|---|---|
| Cardinality | **HLL++** | 12 KB → ±0.81% | relative std err `1.04/√m` | ★★★★★ (max regs) | no |
| Cardinality 2024 | UltraLogLog | ~24-28% less vs HLL | relative | ★★★★★ | no |
| Frequency | **CMS-CU** | `w·d` counters | one-sided `εN` | ★★★★★ (sum tables) | CU only |
| Heavy Hitters | **Space-Saving** | `1/ε` counters | `εN` | ★★★ (Agarwal 2013) | no |
| Quantile rank | **KLL** | `O((1/ε)·√log(1/δ))` | rank `ε` | ★★★★★ | no |
| Quantile value | **DDSketch** | `O(log(nmax/nmin)/log(1+α))` | relative `α` | ★★★★★ (exact) | no |
| Quantile hybrid | t-digest | ~100–1000 centroids | rank, tail-biased | ★★ (lossy) | no |
| Set-op | **Theta** | `k` hashes (~32 KB @ k=4096) | `1/√k` | ★★★★★ | no |
| Jaccard | **MinHash** | `k·64` bits | `1/√k` | ★★★★★ (min) | no |
| Cosine | SimHash | 64–256 bits | Hamming ~ θ/π | ★★★ (sums) | no |
| ANN | LSH | `L·k·bucket` | recall/precision curve | ★★★★ per-band | no |

Bold = default pick in that family.

---

## 3. Cardinality — HLL, HLL++, UltraLogLog

### 3.1 The trick (Flajolet 2007)

```
For each x in stream:
  h = hash64(x)
  j = top p bits of h           # register index, p ∈ [4, 18]
  ρ = leading-1 position in rest
  M[j] = max(M[j], ρ)

estimate = α_m · m² / Σ 2^(-M[j])    where m = 2^p
```

Intuition: if the longest run of leading zeros is `ρ`, you've ~seen `2^ρ` distinct values. `m` registers average variance out.

### 3.2 Sizing (memorize)

Std err `σ = 1.04/√m`, 6 bits/register:

| p | m | Memory | σ |
|---|---|---|---|
| 10 | 1 024 | 768 B | ±3.25% |
| 12 | 4 096 | 3 KB | ±1.63% |
| **14** | **16 384** | **12 KB** | **±0.81%** |
| 16 | 65 536 | 48 KB | ±0.41% |
| 18 | 262 144 | 192 KB | ±0.20% |

p=14 = Redis / Druid / ClickHouse `uniqHLL12`-ish territory. Double `m` → halve error at 2× memory.

### 3.3 HLL++ (Heule, Google 2013)

- **64-bit hashes** — original HLL saturates at ~10⁹ cardinality (hash collisions). HLL++ pushes to 2⁶⁴.
- **Sparse representation**: store `(idx, ρ)` delta-compressed when `n << m`. ~10× less RAM for small `n`.
- **Bias correction** via empirical lookup table (replaces LinearCounting handoff).

Same ±0.81% at p=14, far less memory on small `n`.

### 3.4 UltraLogLog (Ertl 2024) and LogLog-Beta (Qin 2016)

UltraLogLog (Ertl, VLDB 2024) uses the FGRA estimator on 8-bit registers with extra sub-NLZ history bits. Paper reports **28% less memory** at same std err using ML estimation, **24% with a simpler/faster estimator**, **17% with martingale in non-distributed use**. Commutative, idempotent, mergeable, O(1) insert. Reference Java impl: Hash4j library. LogLog-Beta (Qin 2016) is a predecessor simplifying small-range bias without switching estimators.

Rule: HLL++ is still the industry default (Redis, BigQuery, ClickHouse, Druid). UltraLogLog for new code where memory matters — 24-28% smaller HLL, preserved merge semantics.

### 3.5 Merge — the real reason HLL won

```
HLL(A ∪ B) = elementwise_max(HLL(A), HLL(B))
```

Associative, commutative, register-wise. Distributed COUNT DISTINCT = each shard emits 12 KB sketch, coordinator max-merges. BigQuery `HLL_COUNT.MERGE`, Druid `hyperUnique`, ClickHouse `uniqHLL12` all do exactly this.

### 3.6 Trap: intersection via inclusion-exclusion

`|A ∩ B| = |A| + |B| − |A ∪ B|`. Math correct, error explodes: ±1% of each term is 10–100× the intersection size when intersection is small. **Use Theta sketches (§7) or MinHash (§8).**

---

## 4. Frequency — CMS, CountSketch

### 4.1 Count-Min Sketch (Cormode-Muthukrishnan 2005)

```c
uint64_t C[d][w];                                 // d rows, w cols

void update(K x, uint64_t c) {
  for (int i = 0; i < d; i++) C[i][h_i(x) % w] += c;
}
uint64_t estimate(K x) {
  uint64_t m = UINT64_MAX;
  for (int i = 0; i < d; i++) m = min(m, C[i][h_i(x) % w]);
  return m;
}
```

Error: w.p. `1-δ`, estimate within `εN` of true count.

```
w = ⌈e/ε⌉    d = ⌈ln(1/δ)⌉    memory = w·d counters

ε=10⁻³, δ=10⁻²  → w=2 719, d=5  ≈ 54 KB
ε=10⁻⁴, δ=10⁻³  → w=27 183, d=7 ≈ 760 KB
```

CMS always **overestimates** (every increment lands in every row).

### 4.2 Conservative Update (Estan-Varghese 2002)

Only increment the minimum cells:

```c
uint64_t m = estimate(x);
for (int i = 0; i < d; i++)
  if (C[i][h_i(x) % w] == m) C[i][h_i(x) % w] += c;
```

Reduces overestimation 2–10× on Zipfian streams. Default CMS variant for heavy-hitter work.

### 4.3 CountSketch (Charikar-Chen-Farach-Colton 2002)

Adds sign hash `s_i : U → {−1,+1}`, uses median of signed estimates. Two-sided unbiased error vs CMS one-sided. Pick CMS when monotonicity matters (billing, rate limits), CountSketch when unbiasedness matters (stream statistics, sketching moments).

### 4.4 Merge

Elementwise sum of tables, provided identical hash functions + dimensions. Fleet-wide: seed hashes deterministically (e.g. `seed = hash("svc-name-v2")`) so replicas produce compatible tables.

---

## 5. Heavy Hitters / Top-K

| Algo | Year | Memory | Guarantee |
|---|---|---|---|
| Misra-Gries | 1982 | `k` counters | freq > N/(k+1) |
| Lossy Counting | 2002 | `O((1/ε)·log(εN))` | all ≥ εN, no FN |
| **Space-Saving** | 2005 | `1/ε` counters | all ≥ εN, no FN |
| HeavyKeeper | 2018 | `k·d` counters | probabilistic, decay |
| CMS + bounded heap | — | CMS + K-heap | composed |

### 5.1 Space-Saving (Metwally 2005)

```
m = ⌈1/ε⌉ slots (counter, key).
on x:
  if x tracked:       counter++
  elif slot free:     add (1, x)
  else:               evict min-counter slot, replace with (min+1, x)
```

Returns every true heavy hitter (freq > εN). Merge exists (Agarwal et al. 2013).

### 5.2 HeavyKeeper (Gong et al. 2018)

On collision, decrement existing counter with prob `b^(-count)` (`b≈1.08`). Favors heavy flows, evicts light fast. Used in DDoS detection, network telemetry, scalable rate limiting.

### 5.3 Real-world: Twitter stream-lib (2011)

Open-sourced CMS + Space-Saving for trending topics at firehose scale. `CMS + bounded min-heap of K` is the most common industry composition — what Kafka Streams windowed top-K and Flink `TopN` are effectively doing.

---

## 6. Quantiles — t-digest, DDSketch, KLL

Most consequential section for **ch13 (Latency Analysis)**. Wrong quantile sketch = P99 dashboards silently lie.

### 6.1 Error type

```
Rank error ε: |est_rank(q) − true_rank(q)| ≤ ε·n
              P99, n=10⁶, ε=0.01 → rank ±10k → value slop unbounded at tail

Value error α: |est_value(q) − true_value(q)| ≤ α · true_value(q)
              P99=1000 ms, α=0.01 → result ∈ [990, 1010] ms
```

For latency P99, **value error** is what ops expect. Rank error is surprisingly bad at tails: 1% rank error at P99.9 can report P98.9 or P99.99.

### 6.2 Comparison

| Sketch | Error | Bucketing | Memory | Merge lossless |
|---|---|---|---|---|
| GK (Greenwald-Khanna 2001) | rank `ε` | deterministic | `O((1/ε)log(εn))` | no (not assoc.) |
| **KLL** 2016 | rank `ε` | compactors | `O((1/ε)√log(1/δ))` | **yes** |
| t-digest (Dunning 2019) | rank, tail-biased | variable centroids | ~1–10 KB | **no** (re-cluster) |
| **DDSketch** (Datadog 2019) | **value `α`** | log-linear buckets | `O(log(nmax/nmin)/log(1+α))` | **yes** (bucket sum) |
| HDR Histogram (Tene) | value, pre-sized | linear-in-mantissa | fixed | yes, fixed range |
| Moments | value | raw moments + Chebyshev | tiny | yes, coarse |

### 6.3 DDSketch — why Datadog built it

t-digest gives rank error, not value error, and is not **strictly** mergeable — merging two t-digests ≠ feeding all points into one. Unacceptable for SLO aggregation.

DDSketch (Masson-Rim-Lee 2019) uses log-linear buckets:

```
bucket(v) = ⌈log(v) / log(1+α)⌉
```

- **Relative error**: `|q̂ − q| ≤ α · q` for every quantile. What ops expect.
- **Exact merge**: bucket-wise sum, zero loss.
- **O(1) insert** (one log + array increment).

At `α=0.01`, ~2 KB covers 9 orders of magnitude of latency. Datadog aggregates trillions of points/day this way.

**This is why OpenTelemetry Exponential Histograms (2022 spec) and Prometheus Native Histograms (2023) adopted log-linear bucketing** — same math.

### 6.4 t-digest — per-node, tail-biased

Scale function `k(q) = δ · arcsin(2q−1)/π` concentrates centroids at extremes. ~100 centroids → sub-ms error at P99/P99.9. Cross-shard merge is lossy; use for per-node quantiles, aggregate with DDSketch/KLL. Used in Cassandra, Elasticsearch, ClickHouse `quantileTDigest`, Druid.

### 6.5 KLL — right answer for rank error

Hierarchy of compactors: capacity `k`, randomly drops half on full, promotes remainder to next level. Asymptotically optimal for ε-approximate quantiles in `O((1/ε)√log(1/δ))` space. **Fully mergeable with zero extra error.** Apache DataSketches' default `KllDoublesSketch`.

### 6.6 Coordinated omission — the P99 killer (ch13 crossover)

```
Symptom: load test reports P99 = 50 ms, prod P99 = 2 s.
Cause:   Sampling on response. If service stalls 1 s, no samples recorded
         during stall → stall never appears in the quantile sketch.
```

Gil Tene (HDR). Fixes:

- **`recordValueWithExpectedInterval`** (HDR): backfill synthetic samples at expected spacing.
- **Client-side**: record `should_have_completed_at`, not `now() - sent_at`.
- DDSketch/OTel does NOT fix coordinated omission — it's an input discipline problem, not a bucketing problem.

Audit sampling before arguing about sketch choice.

---

## 7. Set-op Sketches — Theta

HLL via inclusion-exclusion fails for small intersections (§3.6). Theta sketches solve this.

### 7.1 Theta (DataSketches, Yahoo 2014)

```
Keep the k smallest hash values seen (k=4096 typical).
θ = k-th smallest / 2⁶⁴

|S| ≈ k/θ               std err ≈ 1/√k    (~1.56% @ k=4096)

Union(A, B)      = merge-min k smallest across A∪B
Intersection     = hashes in both, track retained θ
A-not-B          = symmetric variant
```

Result of any op is another theta sketch → composable. Fixed ~32 KB at k=4096.

**Metamarkets/Druid** built the `thetaSketch` aggregator for ad-tech cohort math — canonical published use case.

### 7.2 Tuple sketches

Theta + summary per retained hash (sum of impressions, revenue). Lets you compute `sum(revenue WHERE user ∈ A ∩ B)` losslessly. Standard in audience segmentation / attribution.

### 7.3 Decision

```
Distinct count + unions only               → HLL++
Distinct count + intersections/differences → Theta
Per-element aggregation alongside set math → Tuple
Jaccard only (not cardinality)             → MinHash
```

---

## 8. Similarity — MinHash, SimHash, LSH

### 8.1 MinHash (Broder 1997)

Estimate Jaccard `J(A,B) = |A∩B|/|A∪B|`:

```
For k independent hashes h_1…h_k:
  sig_i(A) = min_{x∈A} h_i(x)

J_est = |{i : sig_i(A) = sig_i(B)}| / k     std err ≈ 1/√k
```

k=128 → ~9% err, k=256 → ~6%, k=1024 → ~3%. Memory `k·64` bits. Merge (union) = elementwise min.

### 8.2 b-bit MinHash (Li-König 2010)

Keep only `b` lowest bits of each sig (b=1 or 4). Shrinks 8–64× at mild variance cost. `b=1, k=1024` ≈ 128 B/set — still useful for web-scale dedup.

### 8.3 HyperMinHash (Yu-Weber 2017)

Combines HLL register idea + MinHash. Single compact sketch supports both cardinality AND Jaccard. Research-relevant; limited prod adoption.

### 8.4 SimHash (Charikar 2002)

Cosine similarity on high-dim vectors (TF-IDF, embeddings):

```
for each bit j in {0…k-1}:
  s_j = Σ_{f∈v} w_f · (hash(f, j)==1 ? +1 : -1)
  bit_j = sign(s_j)

Hamming dist between sim-hashes ≈ (θ/π)·k       θ = angle
```

Google Manku-Jain-Das Sarma 2007: 64-bit SimHash for near-duplicate web pages at crawl scale.

### 8.5 LSH

Framework, not a sketch. Partition signatures into `L` bands of `r` rows; same-bucket in ≥1 band ⇔ similar enough.

```
P(collision) = 1 − (1 − s^r)^L           s = Jaccard
```

Tune `(L, r)` for sharp S-curve around target threshold. Modern ANN (HNSW, FAISS IVF-PQ) beats LSH on recall-throughput but LSH wins when signatures must merge across streams.

### 8.6 Quick picker

| Have | Want | Use |
|---|---|---|
| sets/shingles | Jaccard | MinHash |
| real-valued vecs | cosine | SimHash |
| top-K similar out of N | sub-linear search | LSH on either |
| streams merging across shards | mergeable sim | MinHash + LSH |

---

## 9. Mergeability — the operational filter

Distributed observability requires lossless merge. Non-mergeable sketch = must centralize raw data.

| Sketch | Merge op | Lossless |
|---|---|---|
| HLL / HLL++ / UltraLogLog | register-wise max | yes |
| Bloom (ch27) | bitwise OR | yes |
| CMS / CountSketch | elementwise sum (same hashes) | yes |
| Space-Saving | merge + prune | bounded err |
| t-digest | centroid re-cluster | **no** |
| **DDSketch / OTel Exponential** | bucket sum | **yes** |
| **KLL** | hierarchy merge | yes |
| HDR Histogram | bucket sum | yes (fixed range) |
| Theta / Tuple | min-k merge | yes |
| MinHash | elementwise min | yes |
| SimHash | sum then re-sign (needs raw sums) | yes |

Rule: distributed P99 dashboards compose losslessly only with **DDSketch, KLL, HDR**. t-digest is best-effort across shards.

---

## 10. Decision Matrix

| Sketch | Mem | Merge | Insert | Tail accuracy | Intersect |
|---|---|---|---|---|---|
| **HLL++** | ★★★★★ | ★★★★★ | ★★★★ | — | ★ (bad via IE) |
| UltraLogLog | ★★★★★ | ★★★★★ | ★★★★ | — | ★ |
| **CMS-CU** | ★★★ | ★★★★★ | ★★★★★ | — | — |
| **Space-Saving** | ★★★★ | ★★★ | ★★★★ | — | — |
| HeavyKeeper | ★★★★ | ★★★ | ★★★★ | — | — |
| **DDSketch** | ★★★★ | ★★★★★ | ★★★★ | ★★★★★ | — |
| t-digest | ★★★★ | ★★ | ★★★★ | ★★★★★ | — |
| KLL | ★★★★ | ★★★★★ | ★★★★ | ★★★★ | — |
| **Theta** | ★★★ | ★★★★★ | ★★★★ | — | ★★★★★ |
| **MinHash** | ★★★ | ★★★★★ | ★★★★ | — | ★★★★ |
| SimHash | ★★★★★ | ★★★ | ★★★★ | — | — |

---

## 11. Pick-by-Workload Heuristics

```
1. distinct count                       → HLL++ (or UltraLogLog for 20% less mem)
2. freq of specific key                 → CMS-CU
3. top-K frequent                       → Space-Saving (+ bounded heap)
4. P99/P99.9 distributed or shards      → DDSketch  (relative err + exact merge)
5. quantile rank-err OK single node     → KLL or t-digest
6. quantile fixed range (1us–1h)        → HDR Histogram
7. |A ∩ B| cardinality                  → Theta sketch, NOT HLL IE
8. Jaccard similarity                   → MinHash + LSH
9. near-duplicate docs                  → SimHash
10. ANN over vectors                    → LSH / HNSW (out of scope, see FAISS)
```

### Domain defaults

| Domain | Pick | Why |
|---|---|---|
| Prometheus/Mimir cardinality | HLL on series IDs | early label-explosion alarm (ch12) |
| OTel / Prom Native Histograms | Exponential Histogram | spec = DDSketch family |
| Datadog P99 | DDSketch | relative err, trillions/day |
| ClickHouse distinct | `uniqHLL12`, `uniqCombined` | in-engine HLL |
| ClickHouse quantile | `quantileTDigest` / `quantileDDSketch` | pick by error type |
| Redis distinct | `PFADD/PFCOUNT/PFMERGE` | HLL built-in |
| Druid cohort math | Theta | canonical use case |
| Kafka Streams dedup window | Bloom + HLL | ch22, ch27 combo |
| Twitter trending | CMS + Space-Saving | stream-lib |
| Google near-dup | SimHash 64-bit | Manku 2007 |
| PostgreSQL distinct | `postgresql-hll` | mergeable aggs |

---

## 12. Integration with Other Chapters

- **ch12 Observability**: Prom Native Histograms and OTel Exponential Histograms are DDSketch-family — the only sane choice for multi-service P99. Pre-filter high-cardinality label explosions via HLL-of-series-IDs before they hit TSDB. Mimir/Thanos query fanout merges across shards: only DDSketch/KLL/HDR give correct global quantiles.
- **ch13 Latency Analysis**: DDSketch is default for distributed P99/P99.9. HDR is default for fixed-range microbenchmarks with coordinated-omission discipline. t-digest acceptable per-node; beware merge lossiness. **Coordinated omission dominates tail error** (§6.6) — fix sampling path first.
- **ch14 Database Profiling**: Redis `PFADD/PFCOUNT/PFMERGE` (built-in since 2.8.9); `postgresql-hll` (Citus/Aiven); ClickHouse `uniqHLL12 / uniqCombined / quantileTDigest / quantileDDSketch`; BigQuery `HLL_COUNT.*`; Snowflake `APPROX_COUNT_DISTINCT` (HLL++). Know the sketch behind the SQL function = know the error budget.
- **ch19 Storage Engine Patterns**: columnar engines can embed per-column HLL / theta in Parquet/ORC footer for instant approximate COUNT DISTINCT. LSM compaction co-merges per-SST HLL for free range cardinalities.
- **ch21 Caching**: Bloom (ch27, existence) + HLL (ch28, distinct count) on same Redis. `BF.ADD` + `PFADD` per write = admission filter + cardinality dashboard.
- **ch22 Kafka**: rolling Bloom + per-window HLL for exactly-once dedup; Space-Saving + CMS for trending-topic; top-K heavy consumers for lag anomaly.
- **ch27 Compact Integer Sets**: probabilistic companion. Bloom/Cuckoo/Binary Fuse = existence. HLL/Theta = cardinality. Roaring/PEF = exact storage. Four together cover all "set of things" problems.

---

## 13. Libraries (production-grade, 2026)

| Family | Lang | Library | Notes |
|---|---|---|---|
| All (HLL, Theta, KLL, CMS, Frequent-Items) | Java/C++/Python/Go | [Apache DataSketches](https://datasketches.apache.org) | reference, Yahoo/Apache |
| HLL/CMS/top-K | Java | [stream-lib](https://github.com/addthis/stream-lib) | Twitter legacy, still used |
| HLL | Redis | `PFADD/PFCOUNT/PFMERGE` | built-in, 12 KB dense |
| HLL | PostgreSQL | [postgresql-hll](https://github.com/citusdata/postgresql-hll) | Citus/Aiven |
| HLL / UltraLogLog | Java | [hash4j](https://github.com/dynatrace-oss/hash4j) | Dynatrace, ULL ref impl |
| HLL | Rust | `hyperloglogplus` crate | pure Rust |
| HLL | Go | `axiomhq/hyperloglog` | HLL++ |
| MinHash/LSH/HLL | Python | [datasketch](https://github.com/ekzhu/datasketch) | MinHash, LSH, weighted |
| t-digest | Java/Rust | [tdunning/t-digest](https://github.com/tdunning/t-digest), `tdigest` crate | reference |
| DDSketch | Java/Go/Python | [sketches-java](https://github.com/DataDog/sketches-java), `sketches-go`, `sketches-py` | Datadog ref |
| HDR Histogram | Java/C/Py/Rust | [HdrHistogram](https://github.com/HdrHistogram) | Tene, CO-aware |
| Exponential Histogram | All | [OTel SDK](https://opentelemetry.io/docs/specs/otel/metrics/data-model/#exponentialhistogram) | spec-mandated |
| ClickHouse | SQL | `uniqHLL12`, `quantileTDigest`, `quantileDDSketch` | in-engine |

### One-line install

```bash
pip install datasketches              # HLL, Theta, KLL, CMS, Frequent
pip install datasketch                # MinHash, LSH, HLL
pip install ddsketch                  # Datadog DDSketch
pip install hdrhistogram              # HDR, CO-aware

cargo add hyperloglogplus
cargo add tdigest

go get github.com/axiomhq/hyperloglog
go get github.com/DataDog/sketches-go/ddsketch

# Redis (built-in)
redis-cli PFADD my_hll a b c d
redis-cli PFCOUNT my_hll                     # → 4
redis-cli PFMERGE dst src1 src2

# ClickHouse
#   SELECT uniqHLL12(user_id),
#          quantileDDSketch(0.01)(latency_ms) FROM events

# Postgres
#   CREATE EXTENSION hll;
#   SELECT hll_cardinality(hll_add_agg(hll_hash_bigint(id))) FROM t;
```

---

## 14. Diagnostic Patterns

### 14.1 "HLL COUNT DISTINCT is off"

```
σ = ±0.81% @ p=14. 20%+ deviation → NOT sketch noise.
Check:
  1. Same hash seed / same p / same impl across shards?
     HLL merges only compose when m=2^p matches.
  2. Sparse→dense downgrade lossy in your client lib?
  3. Using inclusion-exclusion for intersection? → §3.6, use Theta.
  4. 32-bit hash at high cardinality? → collisions saturate; use 64-bit.
```

### 14.2 "P99 sketch diverges across shards"

```
Shards: P99=100 ms each. Global merged: P99=300 ms.
Root causes (in priority):
  1. t-digest merge is lossy → swap to DDSketch or KLL.
  2. Coordinated omission → HDR recordValueWithExpectedInterval,
     or instrument at client / load-gen (§6.6, ch13).
  3. Fixed-bucket histograms with different boundaries per shard
     → swap to Exponential/DDSketch (universal log-linear).
  4. Clock skew → tag at ingress, not per-shard.
```

### 14.3 "CMS frequency overcounting"

```
CMS is one-sided. Worst case +εN over true count.
N=10⁹, ε=10⁻³ → up to 10⁶ overcount worst-case.
Fix:
  1. Switch to Conservative Update (§4.2) — 2–10× reduction on skew.
  2. Double w (= e/ε) to halve error budget.
  3. For top-K, use Space-Saving (bounded err, no FN).
  4. Need unbiased? Swap to CountSketch.
```

### 14.4 "Top-K returns wrong elements"

```
  1. m too small. Rule: m ≥ 10K for moderately skewed, 100K for uniform.
  2. Stream not heavy-tailed → no element > εN → Space-Saving returns
     arbitrary survivors. Not a bug; ill-posed question.
```

### 14.5 "Theta sketch memory grows unexpectedly"

```
  - k (nominal entries) caps memory. Default 4096 → ~32 KB.
  - Updatable mode ~2× compact mode. Serialize compact at rest.
  - Intersection of very small + very large sketches degrades precision.
    Rescale both to same θ before comparing.
```

---

## 15. Next-Gen Cardinality Sketches (2024–2026)

### 15.1 UltraLogLog — the 2024 HLL successor

- **Structure:** 8-bit registers (vs HLL's 6-bit) storing 2 extra "sub-NLZ history bits" per register. Estimator: FGRA (Fisherian Generalized Remaining Area).
- **Pick it when:** new code, any cardinality workload where you'd otherwise reach for HLL/HLL++. **Smaller at same error, merge-compatible, constant-time insert.**
- **How:** Java → [Hash4j `UltraLogLog`](https://github.com/dynatrace-oss/hash4j) (Dynatrace, prod-grade). C++ → `hanyifansuda/UltraLogLog`. **24-28% less memory** than HLL at same std err (depending on ML vs fast estimator).
- **Sizing:** `p=10` (1024 bytes) → ±1.63% RSE; `p=14` (16 KB) → ±0.40% RSE. Roughly 25% smaller than HLL for the same target error.

### 15.2 DynamicLogLog (DLL) — fixes HLL's "error bulge"

- **Structure:** HLL extended with a secondary **dynamic sub-estimator** for the transition region `n ∈ [2.5B, 5B]` where HLL's switch from LinearCounting to harmonic-mean produces a **34% peak error spike**.
- **Pick it when:** your cardinality routinely sits in **1B–10B distinct range** and you can't afford the HLL error spike. Examples: global ad-tech uniques, CDN IP counting, anti-abuse fingerprint dedup.
- **How:** reference impl from arXiv:2603.27405. Target error in the spike region drops from 34.1% (HLL @1536 B) → 1.83% (DLL @1024 B). Same merge semantics as HLL.
- **Note:** DLL fused with UltraLogLog → **UDLL6**: ULL-level accuracy at 75% the memory. Emerging research impl, not yet in Hash4j.

### 15.3 HyperLogLogLog — storage-compressed HLL

- **Structure:** A post-compression of an existing HLL sketch from `O(m log log n)` bits down to `m log₂log₂log₂ m + O(m + log log n)` bits. **Drop-in wire format**: preserves merge, amortized O(1) update.
- **Pick it when:** you already have HLL deployed (Redis, Druid, ClickHouse `uniqHLL12`) and the problem is **storage at rest**, not RAM. Use it as a serialization format for checkpointing / cold-tier offload of sketches.
- **How:** arXiv:2205.11327 reference impl (Python/C). Typically 30-50% smaller serialized than HLL wire format.

### 15.4 Pick matrix — cardinality sketches in 2026

| Workload | Pick | Lib |
|---|---|---|
| New code, general cardinality | **UltraLogLog** | Hash4j (Java), `hanyifansuda/UltraLogLog` (C++) |
| New code, want theoretical optimum | UltraLogLog + ML estimator (slow insert) | Hash4j `UltraLogLog.getDistinctCountEstimate()` with `mlEstimator=true` |
| Legacy HLL ecosystem, can't migrate | **HLL++ + HyperLogLogLog serialization** | standard HLL + compression layer on persist |
| Very large cardinality (1B-10B) with error cliff | **DynamicLogLog / UDLL6** | research impl; wait for Hash4j integration |
| Redis-backed counters, simple API | **Redis `PFADD`/`PFCOUNT`** (classic HLL, 12 KB/key) | Redis ≥ 2.8.9; don't over-engineer |
| BigQuery / ClickHouse existing SQL | `APPROX_COUNT_DISTINCT` / `uniqHLL12` | stays HLL++, per vendor |

### 15.5 Quantile sketches — no 2025 paradigm shift

DDSketch + KLL + t-digest remain the state of the art. OpenTelemetry **Exponential Histogram** (DDSketch-family) is consolidating as the distributed-tracing standard.

- For P99 on merge-heavy distributed workloads → **DDSketch** (relative-error guarantee, exact merge). ClickHouse `quantileDDSketch` since 24.1 (Jan 2024).
- For quantile-value-accurate streaming → **KLL** (rank error is `O(1/ε)`-optimal).
- For compute-cheap approximate → **t-digest** (still fine; slight merge loss).
- **Avoid Prometheus classical histograms** for P99 across shards — use native histograms (exponential, DDSketch-compatible) ≥ Prom 2.40.

---

## 16. References

- Flajolet, Fusy, Gandouet, Meunier (2007). *HyperLogLog*. AOFA.
- Heule, Nunkesser, Hall (2013). *HyperLogLog in Practice*. EDBT (Google).
- Ertl (2024). *UltraLogLog*. VLDB.
- Qin, Kim, Tung (2016). *LogLog-Beta*. arXiv:1612.02284.
- Cormode, Muthukrishnan (2005). *Count-Min Sketch*. JoA.
- Estan, Varghese (2002). *Conservative Update*. SIGCOMM.
- Charikar, Chen, Farach-Colton (2002). *CountSketch*. ICALP.
- Metwally, Agrawal, El Abbadi (2005). *Space-Saving*. ICDT.
- Manku, Motwani (2002). *Lossy Counting*. VLDB.
- Gong et al. (2018). *HeavyKeeper*. USENIX ATC.
- Agarwal, Cormode, Huang, Phillips, Wei, Yi (2013). *Mergeable Summaries*. ACM TODS.
- Greenwald, Khanna (2001). *GK Quantile Summaries*. SIGMOD.
- Karnin, Lang, Liberty (2016). *KLL — Optimal Quantile Approximation in Streams*. FOCS.
- Dunning (2019). *Computing Extremely Accurate Quantiles Using t-Digests*. arXiv:1902.04023.
- Masson, Rim, Lee (2019). *DDSketch*. PVLDB.
- Tene. *How NOT to Measure Latency*. HDR Histogram talks.
- Broder (1997). *MinHash — Resemblance and Containment*. SEQUENCES.
- Charikar (2002). *SimHash*. STOC.
- Li, König (2010). *b-Bit Minwise Hashing*. WWW.
- Yu, Weber (2017). *HyperMinHash*. arXiv:1710.08436.
- Indyk, Motwani (1998). *LSH*. STOC.
- Manku, Jain, Das Sarma (2007). *Near-Duplicates for Web Crawling*. WWW.
- Apache DataSketches project (2015–). https://datasketches.apache.org
- OpenTelemetry Exponential Histogram Data Model spec (2022+).
