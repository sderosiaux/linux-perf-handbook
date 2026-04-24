# Vector / ANN Indexes: HNSW, IVF, PQ, DiskANN, ScaNN

Approximate Nearest Neighbor indexes for high-dimensional vectors (embeddings, image descriptors, audio fingerprints). LLM-actionable: decision matrix, pick-by-workload heuristics, parameter cheat-sheets, memory sizing formulas, integration with pgvector, NUMA, columnar stores, semantic caches.

**Why this chapter exists:** RAG pipelines, agent memory, semantic search and recsys all collapse to the same primitive — *given query vector `q`, return the `k` nearest points from a corpus of `N`*. Naive `O(N·d)` dies at `N > 10^5`. Wrong index loses 10-40 recall points or burns 100× more RAM than needed. Matching recall target + latency budget + mutation pattern + corpus size to the structure is the entire game.

Primary sources: Malkov & Yashunin (HNSW), Jégou et al. (PQ, IVF-PQ), Jayaram Subramanya et al. (DiskANN), Chen et al. (SPANN), Guo et al. (ScaNN), Johnson/Douze/Jégou (FAISS), ann-benchmarks.com.

---

## 1. The Baseline Problem

Given `N` vectors `x_i ∈ R^d` and a query `q`, find the `k` nearest.

| Approach | Index size | Query cost | Recall@k |
|---|---|---|---|
| Brute force (`FlatL2`) | `4·N·d` bytes | `O(N·d)` | 1.0 (exact) |
| LSH | many hash tables | O(1) per table | poor above d=50 |

Flat dies on latency at `N·d > 10^8`. LSH dies on recall at real embedding dims (384, 768, 1536, 3072). Curse of dimensionality: in `d ≥ 20` all pairwise distances concentrate; kd-trees degrade to full scan. Modern ANN exploits **graph navigability** (HNSW), **partitioning + quantization** (IVF-PQ), or **learned quantization** (ScaNN).

```
recall@10
  1.00 ┤                              * brute force
  0.99 ┤                        * HNSW (ef=400)
  0.95 ┤            * HNSW (ef=100)    * IVF-PQ (nprobe=64)
  0.90 ┤      * IVF-Flat (nprobe=8)
  0.80 ┼─────────────────────────────────── QPS (log)
        10       100      1k       10k
```

Every index exposes **one or two knobs** that slide along its own curve. ann-benchmarks.com is the canonical source for per-library, per-dataset curves.

---

## 2. Pattern Classification

| Family | Representative | Recall@10 @ 95% | Build | Mutable | Mem/vec (d=768) | SSD |
|---|---|---|---|---|---|---|
| Graph | **HNSW** | 0.95-0.99 | medium | yes | ~3.4 KB (M=16) | no |
| Graph (disk) | **DiskANN/Vamana** | 0.95-0.98 | high | append | ~200 B + SSD | yes |
| Partition | IVF-Flat | 0.90-0.95 | low | trained+add | ~3 KB | partial |
| Partition+PQ | **IVF-PQ** | 0.80-0.92 | medium | trained+add | ~32-128 B | yes |
| Hybrid | **SPANN** | 0.90-0.95 | high | append | ~200 B + SSD | yes (B-scale) |
| Learned quant | **ScaNN** (AH) | 0.90-0.96 | medium | batch | ~16-64 B | no |
| Hash | LSH | 0.70-0.85 | low | yes | variable | no |

Bold = the default in that family.

---

## 3. HNSW — the 80% default for in-memory ANN

Malkov & Yashunin, 2016. Multi-layer proximity graph; each node has short + long-range edges. New point gets max level `L ~ Exp(1/ln M)`; at each level it connects to `M` nearest already-inserted neighbors with diversity pruning. Level 0 holds every point; higher layers exponentially sparser (log N). Search: greedy-descend from top entry toward `q`, then ef-sized beam search at layer 0.

### 3.1 Parameters

| Param | Default | Range | Controls |
|---|---|---|---|
| `M` | 16 | 8-64 | max edges/node (↑ recall, ↑ RAM, ↓ build speed) |
| `ef_construction` | 200 | 100-800 | build-time beam |
| `ef_search` | 50 | 10-800 | **runtime recall knob**, ≥ k |

hnswlib rule: `M ∈ {12, 16, 32, 48, 64}`; 16 fits most 768-d workloads.

### 3.2 Memory sizing (memorize)

```
bytes/vec ≈ 1.1 · (M · 2 · 8 + vec_bytes)
                  └── ~2·M int64 neighbor IDs  └── 4·d (f32) or 2·d (halfvec)

d=768,  M=16, f32 → 3.6 KB/vec        d=1536, M=16, f32 → 6.8 KB/vec
d=768,  M=16, f16 → 2.0 KB/vec        10M · d=1536 · M=16 ≈ 68 GB RAM
```

Above ~50M vecs of d=1536, HNSW in RAM stops being viable on a single node → DiskANN/SPANN or shard.

### 3.3 Query cost & fit

QPS ∝ `1 / ef_search` · `1 / log N` · `1 / d` (doubling ef ~halves QPS). Typical (ann-benchmarks, sift-128, single thread): hnswlib ~10-20k QPS at recall 0.95, ~2-5k QPS at 0.99.

**Pick HNSW when:** working set fits RAM, streaming inserts, recall ≥ 0.95 tight latency, filtered search (Qdrant, Weaviate, pgvector). **Skip if:** corpus > 100M/node (DiskANN/SPANN), memory-bound (IVF-PQ), or need 0.999 with hard latency (GPU brute-force).

### 3.4 Runnable — hnswlib

```python
import hnswlib, numpy as np
idx = hnswlib.Index(space='cosine', dim=768)
idx.init_index(max_elements=100_000, ef_construction=200, M=16)
idx.add_items(np.random.randn(100_000, 768).astype(np.float32))
idx.set_ef(100)                              # ef_search = recall knob
labels, dists = idx.knn_query(query, k=10)
```

---

## 4. IVF — partition first, scan a shortlist

Offline: k-means on a training sample → `nlist` centroids. Each vector goes in its nearest centroid's posting list. Online: compute distance from `q` to every centroid, probe the `nprobe` closest lists, scan.

| Param | Typical | Controls |
|---|---|---|
| `nlist` | `4·√N` to `16·√N` | partitions (FAISS wiki) |
| `nprobe` | 1-128 | recall knob |
| `training_size` | ≥ 30·nlist | k-means stability |

Query cost ≈ `d · (nlist + nprobe·N/nlist)`. Optimal `nlist ≈ √(nprobe·N)` → classic `√N` rule. Pure IVF-Flat is rare — almost always composed with PQ.

---

## 5. Quantization — trading recall for memory

Raw float32 (1536·4 = 6 KB/vec) forces quantization.

| Technique | Bits/vec (d=768) | Recall hit | Distance compute |
|---|---|---|---|
| `float16` / halfvec | `16·d` | ~0 | same as f32 |
| SQ8 (scalar) | `8·d` | < 1 pt | int8 SIMD (AVX-VNNI) |
| SQ4 | `4·d` | 1-3 pts | 4-bit LUT |
| PQ m=96, nbits=8 | 768 | 3-8 pts | 8× compression, LUT |
| PQ m=32, nbits=8 | 256 | 5-15 pts | 24× compression |
| OPQ + PQ | same as PQ | 1-3 pts better | rotation pre-step |
| Binary (Hamming) | `d` | 10-20 pts | `popcount(xor)` |

**PQ**: split `d` into `m` sub-vectors of dim `d/m`. Train separate k-means with `2^nbits` centroids per sub-vector. Encode as `m` codes of `nbits`. Distance to `q` uses a precomputed LUT (`m × 2^nbits` floats): sum `m` LUT reads per candidate. Defaults: `nbits=8`, `m ∈ {d/4, d/8, d/16}`.

**OPQ** (Ge et al., 2014): learn orthogonal rotation `R` that redistributes variance across sub-vectors before PQ. Recovers 1-3 recall points at same compression. FAISS: `OPQm,IVFnlist,PQm`.

### 5.1 IVF-PQ — classic billion-scale combo

```
FAISS factory: "IVF4096,PQ64"       → 4096 centroids, 64 sub-quantizers·8b = 64 B/vec
100M vecs, d=768 → 6.4 GB           (vs 307 GB for f32 flat)
```

Well-tuned IVF-PQ at `nprobe=32` on 100M vectors: ~1-5k QPS/core at recall@10 ≈ 0.85-0.92 (FAISS wiki). **Residual VQ** and **additive quantization** chain codebooks for more precision at same bit budget (diminishing returns past 3-4 levels).

```python
import faiss, numpy as np
xb = np.random.randn(1_000_000, 128).astype('float32')
index = faiss.IndexIVFPQ(faiss.IndexFlatL2(128), 128, 4096, 16, 8)
index.train(xb[:200_000]); index.add(xb); index.nprobe = 32
D, I = index.search(xb[:5], 10)
```

---

## 6. DiskANN & SPANN — billion-scale on one box

**DiskANN / Vamana** (Microsoft, NeurIPS 2019): single-level graph (not multi-layer like HNSW), built with Vamana — greedy neighbor search + robust pruning with diversity `α`. Graph on SSD; small node cache + PQ-compressed vectors in RAM. Paper headline: **1B-point graph, 32 GB RAM, single box, sub-5ms p95, recall@10 > 0.95** — vs HNSW's ~500 GB RAM at that scale. Trick: one SSD page per hop, prefetch next candidates, re-rank with full-precision vectors from disk.

**SPANN** (Chen et al., NeurIPS 2021): hybrid IVF + DiskANN. Centroids + nav graph in RAM; posting lists on SSD. 2× latency reduction vs DiskANN at equal recall on SIFT1B / SPACEV1B. Shipped in Milvus.

Pick when: corpus > 50-100M/node, NVMe (ideally io_uring, ch19 §8), p95 ~1-5ms acceptable, bulk/append inserts. **Skip if** corpus fits RAM (HNSW wins everywhere) or need < 500 µs p99.

---

## 7. ScaNN — anisotropic quantization

Guo et al., ICML 2020. Standard PQ minimizes reconstruction error, but for MIPS what matters is error **parallel to the query direction**. ScaNN's Anisotropic Vector Quantization reweights the loss. Top performer on ann-benchmarks glove-100-angular and similar since 2020. TensorFlow-backed; integrated in Vertex AI Matching Engine. Pick when: SOTA recall at compact memory, TF stack OK, no streaming inserts.

---

## 8. Filtered ANN — the hard problem

Real queries carry metadata: `tenant_id=42 AND created_at > now()-30d`. Three strategies, each wins in a different selectivity regime.

| Strategy | How | Wins when filter is… |
|---|---|---|
| **Pre-filter** | materialize matching IDs → brute-force subset | very selective (< 1-5%) |
| **Post-filter** | ANN full index → drop non-matches → refill if short | not selective (> 30%) |
| **In-graph filter** | navigate HNSW, skip non-matching neighbors | middle (5-30%) |

**Trap**: 1% selective + post-filter → ANN returns `k=10` of which 0-1 match; refill with `k=1000` blows latency and may undercount. 50% selective + pre-filter → brute-force 500k vectors, slower than plain HNSW.

Library support: **Qdrant** in-graph with payload index (configurable `full_scan_threshold`); **Weaviate** planner picks from selectivity estimates; **Milvus** bitmap filter in-graph for HNSW, pre-filter for IVF; **pgvector** 0.5.x post-filter only, 0.7.0 adds iterative scans; **Vespa** filter intersection first-class.

---

## 9. Hybrid Search — dense + sparse

Pure vector loses on exact matches ("error code E123", SKUs). Pure BM25 loses on paraphrase. Production RAG runs both and fuses with **Reciprocal Rank Fusion** (Cormack et al., SIGIR 2009):

```
rrf_score(doc) = Σ   1 / (k + rank_i(doc))       k = 60 default
              i ∈ rankers
```

No score calibration needed: BM25 returns 0-30, cosine returns 0-1, RRF doesn't care.

Native: **Elasticsearch/OpenSearch** `rrf` retriever (ES 8.8+, OS 2.14+); **Vespa** full rank pipelines; **Qdrant** Query API `prefetch` + fusion; **pgvector** manual SQL (`tsvector` + `ts_rank` joined with cosine); **Weaviate** hybrid with alpha.

---

## 10. Decision Matrix

| Structure | Recall@95 QPS | Build | Mem/vec | B-scale | Mutable | Filtered |
|---|---|---|---|---|---|---|
| Flat | ★ | instant | ★★★★★ | no | yes | trivial |
| **HNSW** | ★★★★ | ★★★ | ★★ | shard | yes | ★★★★ |
| IVF-Flat | ★★★ | ★★ | ★★ | shard | add | ★★ |
| **IVF-PQ** | ★★★ | ★★★ | ★★★★★ | yes | add | ★★ |
| OPQ+IVF-PQ | ★★★ | ★★★★ | ★★★★★ | yes | add | ★★ |
| **DiskANN** | ★★★★ | ★★★★ | ★★★★ (SSD) | yes | append | ★★ |
| **SPANN** | ★★★★ | ★★★★ | ★★★★ (SSD) | yes | append | ★★ |
| **ScaNN** | ★★★★★ | ★★★ | ★★★★ | yes | batch | ★ |

---

## 11. Pick-by-Workload Heuristics

```
1. Corpus size?
   ≤ 10^5       → Flat. No index. Done.
   10^5–10^8    → go to 2.
   > 10^8       → DiskANN/SPANN, or shard HNSW.

2. Memory budget?
   Generous (vec_bytes·N fits 2×)   → HNSW.
   Tight (< 256 B/vec)              → IVF-PQ or OPQ+IVF-PQ.

3. Mutation?
   Streaming per-request       → HNSW (Qdrant, pgvector).
   Batch ingest, rare retrain  → IVF-PQ or ScaNN.
   Append-only billion-scale   → DiskANN / SPANN.

4. Recall target?
   ≥ 0.99 → HNSW ef_search≥400, or GPU brute (FAISS-GPU).
   0.95   → HNSW default, or ScaNN.
   0.90   → IVF-PQ nprobe 16-32.
   0.80   → tight IVF-PQ, or binary codes.

5. Filter selectivity?
   < 5%   → pre-filter.
   5-30%  → in-graph (Qdrant, Weaviate, Milvus HNSW).
   > 30%  → post-filter; HNSW wins.
```

### Domain defaults

| Domain | Default | Why |
|---|---|---|
| RAG 10k-1M docs | pgvector HNSW or Qdrant | simple ops, filters, PG-native |
| RAG 1M-100M chunks | Qdrant / Milvus / Weaviate | dedicated, sharding |
| Image dedup 1B+ | IVF-PQ (FAISS) or DiskANN | binary / compact PQ |
| Recsys candidate gen | ScaNN or FAISS IVF-PQ | high QPS, batch train |
| Semantic cache (ch21) | pgvector or in-proc hnswlib | small corpus, low latency |
| Multi-tenant SaaS | Qdrant collections / Turbopuffer namespaces | isolation + cost |
| Serverless / S3 | Turbopuffer, LanceDB | object-store native |
| Hybrid BM25+vector | Elastic / OpenSearch / Vespa | native RRF |

---

## 12. Integration with Other Chapters

- **ch14 Database Profiling**: pgvector. 0.5.0 added `hnsw`, 0.7.0 added `halfvec` (f16) and sparsevec. `maintenance_work_mem` is critical during HNSW build — spills cripple build time. `SET maintenance_work_mem='2GB'; SET max_parallel_maintenance_workers=7;` before `CREATE INDEX`. Query: `SET hnsw.ef_search=100;`.
- **ch15 Memory Subsystem**: HNSW > 32 GB on NUMA — `numactl --membind=0` or the graph walk bounces between nodes and p99 doubles. THP madvise hugepages help dTLB on large indexes (ch26 §2).
- **ch19 Storage Engines**: LanceDB stores vectors in columnar Lance format (Arrow), builds IVF-PQ alongside. Parquet + FAISS external-index for offline recsys. DiskANN direct I/O; pair with io_uring (ch19 §8) at billion-scale.
- **ch21 Caching**: semantic cache = vector lookup on recent (query, response) pairs. Tiny in-proc HNSW (10k-100k). Hit threshold typically `cosine > 0.92`. Trap: 0.92 is not a paraphrase guarantee; always log for offline eval.
- **ch27 Compact Integer Sets**: PQ codes are dense `uint8[m]`. IVF posting lists are Roaring when sparse per centroid. Filter results are bitmap intersections before the ANN phase (Qdrant, Milvus).

---

## 13. Libraries (production, 2026)

| Library | Lang | Indexes | Notes |
|---|---|---|---|
| [FAISS](https://github.com/facebookresearch/faiss) | C++/Py | Flat, IVF, PQ, IVF-PQ, HNSW, GPU | Meta, reference, factory strings |
| [hnswlib](https://github.com/nmslib/hnswlib) | C++/Py | HNSW | reference HNSW, header-only |
| [Qdrant](https://qdrant.tech/) | Rust | HNSW + payload | strong filtered search |
| [Milvus](https://milvus.io/) | C++/Go | HNSW, IVF, DiskANN, SCANN | multi-index, Knowhere |
| [Weaviate](https://weaviate.io/) | Go | HNSW + modules | hybrid search native |
| [pgvector](https://github.com/pgvector/pgvector) | C (PG ext) | IVF-Flat, HNSW, halfvec | 0.5.0 HNSW, 0.7.0 halfvec/sparse |
| [LanceDB](https://lancedb.com/) | Rust | IVF-PQ, HNSW | columnar + vector, object-store |
| [Turbopuffer](https://turbopuffer.com/) | closed | managed | serverless on object storage (2024) |
| [ScaNN](https://github.com/google-research/google-research/tree/master/scann) | C++/TF | AH quantization | top on ann-benchmarks |
| [Vespa](https://vespa.ai/) | Java/C++ | HNSW + ranking | full IR + ANN, hybrid native |
| [DiskANN](https://github.com/microsoft/DiskANN) | C++ | Vamana | Microsoft reference |
| [USearch](https://github.com/unum-cloud/usearch) | multi | HNSW | SIMD-heavy, ultra-small binary |
| [Voyager](https://github.com/spotify/voyager) | multi | HNSW | Spotify's hnswlib fork |

```bash
pip install faiss-cpu hnswlib usearch lancedb          # or faiss-gpu-cu12
docker run -p 6333:6333 qdrant/qdrant
docker run -p 8080:8080 cr.weaviate.io/semitechnologies/weaviate:latest
curl -sfL https://raw.githubusercontent.com/milvus-io/milvus/master/scripts/standalone_embed.sh | bash

# pgvector
CREATE EXTENSION vector;
CREATE INDEX ON docs USING hnsw (embedding vector_cosine_ops) WITH (m=16, ef_construction=64);
```

---

## 14. Parameter Cheat-Sheet

```
HNSW (hnswlib, Qdrant, pgvector, Milvus)
  Build:  M=16 (d≤1000), 32-48 for higher recall
          ef_construction=200 (400-800 for static corpora)
  Query:  ef_search=100  (k ≤ ef_search ≤ 800)
  Size:   bytes/vec ≈ 1.1·(M·2·8 + 4·d)   f32  |  +2·d for halfvec

IVF / IVF-PQ (FAISS, Milvus)
  Build:  nlist = 4·√N..16·√N, train ≥ 30·nlist samples
          m = d/4, d/8, d/16 → 64/32/16 B/vec at nbits=8
  Query:  nprobe = 8-64  (~1% of nlist starting point)

DiskANN
  Build:  R (max degree)=64-128, L (search-list)=100-200
          α (diversity)=1.2-1.4, PQ bytes=32-64 in-RAM
  Search: Ls (beam) = k · 2-10
```

---

## 15. Diagnostic Patterns

**"Recall dropped after insert burst"** — HNSW: ef_construction too low on streaming inserts; IVF/IVF-PQ: centroids drifted (retrain on new distribution); deletes tombstone without rebalance (full rebuild when >10% deleted).

**"P99 spikes on specific queries"** — measure filter selectivity. Post-filter + selective → re-query expands k → blows budget. Pre-filter + unselective → brute-force huge subset. Fix: filter-aware index (Qdrant, Weaviate) or shard by dominant filter field.

**"Index build OOM"** — pgvector: raise `maintenance_work_mem` to 2-4 GB, parallel workers to 7. FAISS: train on sample ≤ 1M. HNSW M=64 on 100M ≈ 100 GB — drop M or shard.

**"QPS too low, recall is fine"** — drop `ef_search` until recall starts dropping, stop one step before. Batch queries. halfvec cuts bandwidth 2× → 1.3-1.7× QPS. Try SQ8 (recall delta often < 1 pt). Strip `IndexIDMap` if unneeded.

**"Can't fit in RAM"** — ladder, apply top-down:

```
1. halfvec (f16)        2× smaller, ~0 recall loss
2. SQ8 scalar           4× smaller, < 1 pt loss
3. IVF-PQ (m=d/8, 8b)   24× smaller, 3-8 pts loss
4. OPQ + IVF-PQ         same mem, recover 1-3 pts
5. DiskANN / SPANN      graph on SSD, 32 GB RAM → 1B vecs
6. Shard across nodes   last resort
```

---

## 16. Distance Metrics

| Metric | Formula | When |
|---|---|---|
| L2 | `‖a−b‖₂` | raw pixel / descriptor |
| Inner product | `a·b` | most LLM embeddings (normalized) |
| Cosine | `(a·b) / (‖a‖·‖b‖)` | semantic embeddings, not unit-norm |
| Hamming | `popcount(a XOR b)` | binary codes (ITQ, 1-bit quant) |
| L1 (Manhattan) | `Σ|a_i−b_i|` | rare, some anomaly detection |

OpenAI `text-embedding-3-*` are L2-normalized → IP ≡ cosine ≡ (2−L2²)/2. Pick IP for speed. Nomic, Jina, Cohere vary — check before choosing the metric.

---

## 17. ann-benchmarks Reference

ann-benchmarks.com runs reproducible recall-vs-QPS curves across ~30 libraries on standard datasets (glove-100-angular, sift-128-euclidean, deep-image-96-angular, gist-960-euclidean, nytimes-256-angular, mnist-784-euclidean). Only place for apples-to-apples numbers. Rules: cite recall@10 at specific QPS on a specific dataset; don't cross datasets (different geometry); hardware matters (usually m5.8xlarge).

Top by family (2024-25): graph = hnswlib, glass, voyager, ngt-panng. Quantization = scann, faiss-ivfpq. Disk = diskann. GPU = faiss-gpu, cuVS (RAPIDS).

---

## 18. Filtered-ANN Structures (2025–2026)

Real RAG/e-commerce/multi-tenant workloads need *ANN restricted to a subset* — the §7 pre/post-filter dichotomy breaks between 1% and 50% selectivity. These are the five structures that fill the gap, with explicit pick rules.

### 18.1 FilteredVamana — label-aware Vamana graph

- **Structure:** DiskANN's Vamana graph extended so each edge is tagged with the intersection of endpoint labels. Search starts from per-label entry points and only traverses edges valid for the query's labels.
- **Pick it when:** filter predicate = small set of **categorical labels** (< ~100 distinct values total), labels known at build time, build runs offline.
- **How:** `microsoft/DiskANN` (C++/Python). Key params: `--filtered_index 1`, provide per-vector label file at build, query with `--query_filters_file`.

### 18.2 StitchedVamana — per-label subgraph + stitch

- **Structure:** Build a full Vamana subgraph **per label**, then stitch them via vectors carrying multiple labels (bridge nodes). Disconnected subgraphs fused at multi-label points.
- **Pick it when:** **high-cardinality labels** (1000s) but each vector only carries **few labels** (1–5). Build time can be long (offline).
- **How:** Same DiskANN repo, build mode `stitched`. Expect build time 2-5× FilteredVamana; query-time recall on filtered subsets markedly higher.

### 18.3 ACORN — HNSW with predicate pushdown

- **Structure:** HNSW graph where search routing scores candidates by `distance × filter_match_score`. Maintains connectivity at low selectivity by not pruning graph edges based on filters; enforces filter only at candidate-emit.
- **Pick it when:** **selectivity unknown or mixed** across queries (one tenant 1% selective, another 50%), multiple filter schemas, need a single index. Default choice if you can't pre-characterize workload.
- **How:** reference impl from [ACORN paper](https://arxiv.org/abs/2403.04871); incorporated in modern Qdrant/Weaviate as "in-graph filtering" modes.

### 18.4 SeRF — range-filter specialist

- **Structure:** Segment tree over a vector graph, indexed by the ordered attribute. Query: `[low, high]` on the attribute + ANN on the vectors in that segment.
- **Pick it when:** filter is a **single range on an ordered attribute** — timestamps, price, score, version. Dominates all alternatives here.
- **How:** [SeRF GitHub](https://github.com/microsoft/SeRF). Directly maps to time-bucket RAG queries ("last 7 days", "price < $100").

### 18.5 VecFlow — GPU-native filtered ANN

- **Structure:** Label-centric indexing where each label owns a sub-index on GPU. Query: dispatch filter labels in parallel across SMs.
- **Pick it when:** you have **GPU budget** (A100/H100 in inventory) and query volume is high (>100k QPS filtered). Reports **5M QPS at recall 0.9 on single A100** — 10-100× CPU Filtered-DiskANN.
- **How:** [VecFlow repo](https://github.com/Sepalpuppy/VecFlow) (research-stage); Milvus GPU also supports filtered search as a prod alternative.

### 18.6 Pick-by-filter-type decision tree

```
Query pattern                              → Structure       → Config hint

Single label, small cardinality (<100)    → FilteredVamana  → DiskANN build_memory_index --filtered_index
                                                                M=48, L=100, α=1.2

Multiple labels per vec, high-card        → StitchedVamana  → DiskANN stitched mode
                                                                tolerate 2-5× build time

Range on ordered attr (time, price)       → SeRF            → Segment-tree over attr, L2 on vec
                                                                window queries native

Selectivity varies widely, mixed schemas  → ACORN           → Qdrant payload-filter "selectivity=auto"
                                             (or pgvector)   → or pgvector HNSW + WHERE clause

High-QPS + GPU available                  → VecFlow / Milvus GPU
                                                              → A100+: 5M QPS recall 0.9

No label info in advance / ad-hoc         → Qdrant in-graph filter (§7)
```

### 18.7 Update to the §7 pre/post/in-graph rule

Old rule said "in-graph for 10-20% selectivity." The 2025 benchmarks (arXiv:2509.07789, arXiv:2508.16263) sharpen it:

| Selectivity | Old recommendation | 2026 recommendation |
|---|---|---|
| <1% | Pre-filter | Pre-filter + small sub-index per label (FilteredVamana mini-index if label fits) |
| 1–10% | In-graph | **FilteredVamana** or **StitchedVamana** if labels are categorical |
| 10–50% | In-graph or post | **ACORN** if mixed schema; Filtered-DiskANN if single schema |
| >50% | Post-filter | Post-filter still fine; SeRF if range attribute |
| Any + range | (not addressed) | **SeRF** wins consistently on ordered attrs |

---

## 19. References

- Malkov, Yashunin (2018). *HNSW: Efficient and robust approximate nearest neighbor search using Hierarchical Navigable Small World graphs*. IEEE TPAMI.
- Jégou, Douze, Schmid (2011). *Product quantization for nearest neighbor search*. IEEE TPAMI.
- Ge, He, Ke, Sun (2014). *Optimized Product Quantization*. IEEE TPAMI.
- Johnson, Douze, Jégou (2019). *Billion-scale similarity search with GPUs* (FAISS).
- Jayaram Subramanya et al. (2019). *DiskANN*. NeurIPS.
- Chen et al. (2021). *SPANN*. NeurIPS.
- Guo et al. (2020). *Accelerating Large-Scale Inference with Anisotropic Vector Quantization* (ScaNN). ICML.
- Cormack, Clarke, Büttcher (2009). *Reciprocal Rank Fusion*. SIGIR.
- Babenko, Lempitsky (2014). *Additive Quantization for Extreme Vector Compression*. CVPR.
- [ann-benchmarks.com](https://ann-benchmarks.com/), [FAISS wiki](https://github.com/facebookresearch/faiss/wiki), [pgvector](https://github.com/pgvector/pgvector).

---

## See Also

- [Database Profiling](14-database-profiling.md) — pgvector, work_mem, Postgres extensions
- [Memory Subsystem](15-memory-subsystem.md) — NUMA pinning for large in-RAM indexes
- [Storage Engine Patterns](19-storage-engine-patterns.md) — Lance, Parquet, io_uring for DiskANN
- [Caching Patterns](21-caching-patterns.md) — semantic cache with vector lookup
- [Compact Integer Sets](27-compact-integer-sets.md) — Roaring for filter bitmaps and posting lists
