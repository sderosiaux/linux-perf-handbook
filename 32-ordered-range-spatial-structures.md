# Ordered / Range / Spatial Structures: Skip Lists, B-tree Variants, Fenwick, BKD, R-tree, S2/H3

Data structures for keys that need **order** (iteration, predecessor/successor), **ranges** (prefix scans, interval overlap, `[lo, hi]` filters), or **geometry** (k-NN, bounding boxes, geospatial joins). Companion to ch27 (sets of integers). LLM-actionable: comparison tables, pick-by-workload tree, library pointers, diagnostics.

**Why:** once keys are not flat integers — composite, ordered, or spatial — hash tables and Roaring stop helping. Picking wrong = O(n²) spatial joins, or a B-tree where linear scan would win.

Primary sources: Pugh (skip lists), Levandoski/Lomet/Sengupta (Bw-tree), Bender et al. (Be-tree), Mao/Kohler (Masstree), Leis et al. (OLC), Fenwick, Bentley (kd-tree), Guttman (R-tree), Beckmann (R*-tree), Lucene BKD, Uber H3, Google S2, PostGIS, SQLite R*Tree docs.

---

## 1. Pattern Classification

| Family | Representative | Ops | Mutability | Cache |
|---|---|---|---|---|
| Probabilistic ordered | **Skip list** | point, range, order | mutable, lock-free | poor (pointer chase) |
| B-tree | **B+tree, Bw-, Be-, Masstree, OLC** | point, range, order | mutable | excellent (block) |
| Ordered trie | **ART** (ch27) | point, range, prefix | mutable | good |
| Prefix sum | **Fenwick (BIT), segment tree** | range-sum, range-update | mutable | good |
| Interval | **Interval tree / augmented BST** | stabbing, overlap | mutable | moderate |
| Low-D k-NN | **kd-tree, ball tree** | k-NN (D<20) | rebuild | degrades > D=20 |
| Multi-dim range | **BKD** (Lucene) | range, geo, k-NN | append-only, LSM | excellent |
| 2D/3D bbox | **R-tree, R*-tree, R+-tree** | overlap, contains, k-NN | mutable / bulk-loaded | moderate |
| N-D → 1-D | **Z-order, Hilbert, GeoHash** | range via prefix | rebuild | excellent |
| Geospatial cell | **S2, H3** | covering, neighbors | append cell IDs | excellent |
| Theoretical | Van Emde Boas, y-fast trie | predecessor O(log log U) | academic | poor |

Bold = default in that family.

---

## 2. Skip Lists

**Pugh 1990.** Each node appears at level `i` with probability `p^i` (`p=1/2` or `1/4`). Forward pointers at level `i` skip `~2^i` nodes. Point/pred/succ in expected O(log n). No rotations. Concurrent insert = CAS on forward pointers; deletes use marked pointers (Harris/Fraser).

**Production:**
- **RocksDB memtable** — `InlineSkipList<Cmp>`, lock-free writers, arena-allocated, inline keys, height cap ~12. See `memtable/inlineskiplist.h`.
- **LevelDB memtable** — classic skip list, single-writer.
- **Redis ZSET** — skip list + hashmap (O(1) member + ordered rank). Switches to `listpack` under `zset-max-listpack-entries` (128) / `zset-max-listpack-value` (64 B).
- **`java.util.concurrent.ConcurrentSkipListMap`** — JVM reference.

### Skip list vs B-tree

| | Skip list | B+tree |
|---|---|---|
| In-memory, concurrent writes | fine-grained CAS, no rotations | lock coupling nontrivial |
| Disk, page-aligned | pointer chase = miss/level | fan-out 100-1000, depth 3-4 at billions |
| Range scan | adequate (L0 list) | better (sequential leaves) |
| Memory overhead | ~2x node from forward arrays | ~1.3x slotted pages |
| Cache | poor (heap-scattered) | excellent (block) |

**Rule:** in-memory write-heavy concurrent ordered map → skip list. Disk-backed or range-heavy → B+tree.

---

## 3. B-tree Variants

### 3.1 B+tree refresher

Keys in internals, values in linked leaves. Fan-out = `page_size / (key+ptr)`. 4 KB + 16 B keys → fan-out ~200, depth 4 covers 1.6 B entries. Watch: split/merge under writes; crab latching serializes the root.

### 3.2 Bw-tree (Microsoft, VLDB 2013)

Lock-free via **delta chains**. Updates prepend a delta; CAS flips `mapping_table[pid]`. Background consolidation merges chain → new base page. Used in Hekaton (SQL Server in-memory), Peloton, CMU papers. Tradeoff: read cost grows with chain length → latency tails if consolidation lags.

### 3.3 Bε-tree / Be-tree (Bender et al.)

**Write-optimized.** Each internal node has a message buffer of size `ε·B`. Inserts append; batched flush downward trades some read fan-out (factor `ε`) for `(1/ε·log n)` fewer write IOs.

| Workload | vs B+tree |
|---|---|
| Sequential insert | same |
| Random insert (billions) | 10-50x fewer IOs |
| Point lookup | same order (traverse + buffer check) |
| Range scan | slightly slower (merge buffers + leaves) |

Used in TokuDB, TokuMX, Percona ft-index, BetrFS. Internal nodes ~4 MB vs 16 KB B+tree.

### 3.4 Masstree (MIT, EuroSys 2012)

Trie of B+trees indexed on 8-byte key slices. Variable-length keys, optimistic concurrency (version counters), per-core caches, software prefetch. ~6M queries/s/core on Silo benchmarks.

### 3.5 OLC B-tree (Leis, DaMoN 2019)

**Optimistic Lock Coupling.** Readers: read `v_before`, read contents, read `v_after`; if unchanged and unlocked → success, else retry. Writers bump version on release. Zero cache-line ping-pong on reads. Used in Umbra, LeanStore.

### 3.6 Pick

```
Disk-backed OLTP, mixed R/W       → B+tree (PostgreSQL, InnoDB, WiredTiger)
In-memory OLTP, variable-length   → Masstree or ART (ch27)
In-memory OLTP, fixed keys        → OLC B-tree
Write-amplification-sensitive     → Be-tree (TokuDB/ft-index)
Lock-free hard requirement        → Bw-tree (accept read chain cost)
Primary index, in-memory analytic → ART (DuckDB)
```

---

## 4. Prefix / Order-Preserving Hashing

Hash the key with an order-preserving function, bucket contents sorted. Rare in practice. Shows up where a **sorted primary key** does the clustering and a secondary hash narrows further (ClickHouse, some Iceberg partition transforms). Tradeoff: skew → hot buckets. Don't reach for this unless the access pattern is measured.

---

## 5. Fenwick Tree (BIT)

`update(i, Δ)` and `prefix_sum(i)` both O(log n), in-place, 20 lines.

```cpp
struct Fenwick {
    std::vector<int64_t> t;                       // 1-indexed
    explicit Fenwick(int n) : t(n + 1) {}
    void update(int i, int64_t d) {
        for (; i < (int)t.size(); i += i & -i) t[i] += d;
    }
    int64_t prefix(int i) const {
        int64_t s = 0;
        for (; i > 0; i -= i & -i) s += t[i];
        return s;
    }
    int64_t range(int l, int r) const { return prefix(r) - prefix(l - 1); }
};
```

`i & -i` isolates the lowest set bit (BMI1 `BLSI`). No branches, no pointers, cache-resident for `n < 10^7`.

**Extensions:** range-update+point-query (one BIT over diffs), range-update+range-query (two BITs), 2D for image integrals with sparse updates.

**Production:** streaming percentiles over integer histograms (BIT over bucket counts → `prefix_sum` gives rank → quantile); real-time leaderboards with bounded integer scores; time-series count-by-bucket with O(log n) range count. See ch12.

**Rule:** prefix/range aggregate over an integer domain with point updates → Fenwick is the smallest structure that answers it.

---

## 6. Segment Tree

Each node holds an associative aggregate (sum, min, max, gcd, xor, custom monoid). `point_update`, `range_update` with lazy propagation, `range_query`, all O(log n). Memory 2-4n.

| | BIT | Segment tree |
|---|---|---|
| Prefix / range sum | yes | yes |
| Range min/max | hack | native |
| Range update + range query | 2 BITs | lazy, general |
| Arbitrary monoid | no | yes |
| Code | ~20 lines | 60-150 |

**Persistent segment tree:** update returns a new root sharing unchanged subtrees → O(log n) per version; used for offline range queries (k-th smallest in `[l,r]`), time-travel. Real use: network flow analysis, sweep-line geometry, range aggregates in streaming pipelines.

---

## 7. Interval Tree / Augmented BST

Stabbing ("intervals covering point `p`") or overlap ("intersect `[L,R]`") in O(log n + k). Balanced BST keyed by `l_i`, augmented with `max_r` over the subtree. Prune descent when `L > node.max_r`.

**Used by:** calendar/scheduling, genomics (UCSC bigWig, tabix, bedtools), firewall/ACL rule matching (nftables sets), Linux kernel VMA lookup (`find_vma_intersection`).

**Tradeoff:** for static sets, a sorted array filtered by `end` is often faster (Heng Li's **cgranges**). Interval tree only when the set is mutable.

---

## 8. kd-tree / Ball tree — low-D k-NN

Recursive axis-aligned splits (kd) or ball splits. The curse of dimensionality:

| D | kd-tree speedup vs linear |
|---|---|
| 2-3 | 100-1000x |
| 8-12 | 10-50x |
| 16-20 | 2-5x |
| > 20 | ≤ 1x (often worse) |

In high D, every bounding region touches the query ball → no pruning. **Rule:** D > 20 (SIFT-128, text 384/768/1536/3072) → HNSW/IVF-PQ/DiskANN, **see ch29**. kd-tree is for physical space (2-3), small feature vectors, tabular spatial joins.

Libs: scikit-learn `KDTree`/`BallTree`, SciPy `cKDTree`, FLANN, nanoflann.

---

## 9. BKD Tree — Lucene's multi-dim index

**Procopiuc/Agarwal/Arge/Vitter, SSTD 2003.** Lucene 6 (2016) replaced hierarchical trie-ranged numeric terms with BKD. Powers Elasticsearch/OpenSearch geo (`geo_point`), numeric ranges (`LongPoint`, `DoublePoint`), dates, IP ranges.

- Points serialized into **blocks** (default 512 docs, `DEFAULT_MAX_POINTS_IN_LEAF_NODE`).
- Bottom-up build, split on widest-range axis.
- Immutable per segment; LSM-style merges like the rest of Lucene.
- Per-block min/max stored for pruning.

Vs old trie-ranged terms: ~50% smaller numeric/geo index, native geo k-NN (`LatLonPoint.nearest`), faster selective range queries.

```java
new LatLonPoint("loc", lat, lon);        // 2-D BKD
LatLonPoint.newBoxQuery("loc", minLat, maxLat, minLon, maxLon);
LatLonPoint.newDistanceQuery("loc", lat, lon, radiusMeters);
```

In Elasticsearch `geo_point`: BKD pre-filters by bbox, exact distance refinement follows. Cross-ref: ch19 §4-5.

---

## 10. R-tree Family

### 10.1 R-tree (Guttman 1984)

B-tree whose keys are **minimum bounding rectangles (MBRs)**. Sibling MBRs overlap → queries descend multiple subtrees. Overlap grows with skew.

### 10.2 R*-tree (Beckmann 1990)

Refined insert/split heuristics (minimize overlap, area, margin) + **forced reinsertion** on overflow. ~30-50% fewer disk accesses vs vanilla R-tree. Used by **PostGIS** (via GiST), **SQLite** (module is named `rtree`; internally uses R*-tree heuristics), Boost.Geometry, `rstar` (Rust).

### 10.3 R+-tree

Disallow sibling MBR overlap by **duplicating** entries that straddle. Static or rarely-updated only; clean point-in-polygon pruning.

### 10.4 STR (Sort-Tile-Recursive) bulk load

Leutenegger 1997. Sort by x into `√n` slices, sort each by y, pack. O(n log n), near-optimal packed R-tree. Used for immutable geo datasets (map tiles, offline processing).

### 10.5 Pick

| Scenario | Pick |
|---|---|
| Mutable R/W OLTP geo | R*-tree (PostGIS GiST, SQLite rtree) |
| Static bulk-load, read-only | R+-tree or STR-packed R-tree |
| Append-heavy (log-structured) | BKD (Lucene) — R-trees dislike append without rebuild |
| PIP over many polygons | R*-tree candidates → exact PIP (GEOS/JTS) |

**Integrations:** PostGIS `CREATE INDEX ... USING GIST (geom)` covers `&&`, `<->`, `@>`; SQLite `CREATE VIRTUAL TABLE t USING rtree(id, minX, maxX, minY, maxY)`; MySQL InnoDB `SPATIAL INDEX` on `GEOMETRY`; Elasticsearch `geo_shape`. See ch14.

---

## 11. Space-Filling Curves

Locality-preserving N-D → 1-D bijection. Index the 1-D key in a B+tree/LSM, get spatial clustering for free.

| Curve | Locality | Encode | Human-readable | Anisotropy | Use |
|---|---|---|---|---|---|
| **Z-order / Morton** | good | cheapest (`PDEP`) | no | mild | Parquet/Iceberg Z-order clustering (Delta/Iceberg `OPTIMIZE`) |
| **Hilbert** | best | moderate (5-10x Morton) | no | none | Google S2 cell IDs, high-quality spatial sort |
| **GeoHash** | moderate | cheap | yes (base-32) | strong (lat-dependent) | Redis GEO (52-bit geohash in ZSET), Elasticsearch `geohash_grid` |

```cpp
uint64_t morton2d(uint32_t x, uint32_t y) {
    return _pdep_u64(x, 0x5555555555555555ULL) |
           _pdep_u64(y, 0xAAAAAAAAAAAAAAAAULL);
}
```

**Rule:** Hilbert when encode cost is fine and cluster quality matters. Z-order when cheap suffices. GeoHash when human-readable strings matter (Redis GEO).

---

## 12. S2 (Google) — spherical quad-tree + Hilbert

Earth projected on 6 cube faces; each face a 30-level quad-tree. Cells ordered along a **Hilbert curve** inside each face. **64-bit cell ID** = face (3 bits) + Hilbert path + terminator. `level 30` ≈ 0.7 cm²; `level 0` ≈ 85 M km².

Cell IDs totally ordered → **range queries = B+tree range scans** on cell ID columns. Covering a region = a few sorted ID ranges.

```cpp
S2RegionCoverer::Options opts;
opts.set_max_cells(8);
S2RegionCoverer coverer(opts);
S2CellUnion covering = coverer.GetCovering(region);  // few uint64_t ranges
```

**Used by:** Google Maps/S2 library (C++, Go, Java, Python, Rust `s2`), Foursquare, MongoDB `2dsphere`, CockroachDB spatial (inverted index over cell IDs).

---

## 13. H3 (Uber) — hexagonal hierarchical

Earth tessellated with hexagons. 16 resolutions: `res 0` = 122 cells (~4.3 M km² each), `res 15` ~ 1 m². 64-bit cell ID = resolution + path from base cell. Aperture-7 hierarchy: a parent hex approximates 7 children (imperfect nesting).

**Why hexagons:** isotropy. All 6 neighbors equidistant. Squares (S2) have 4 near + 4 diagonal-far → anisotropy. Matters for ride matching, supply/demand heatmaps, dispatch — uniform neighbor sampling is the whole game.

### S2 vs H3

| | S2 (square, quad-tree) | H3 (hexagon) |
|---|---|---|
| Neighbor distance | 4 near + 4 diagonal-far | 6 uniform |
| Hierarchy | exact 4:1 | imperfect 7:1 |
| Covering | excellent (Hilbert ranges) | harder (no total order) |
| Cross-resolution | cheap (prefix) | needs compact/uncompact |
| Typical win | range queries, B+tree index | grid math, k-ring neighborhood |

**Production H3:** Uber dispatch/surge, Databricks (`h3_longlatash3`, `h3_kring`, `h3_polyfillash3`), Apache Sedona (H3+S2+GeoHash+R-tree+KDB for spatial Spark).

```python
import h3
cell = h3.latlng_to_cell(37.775, -122.418, 9)   # res 9 ~ 100m
ring = h3.grid_disk(cell, k=2)
parent = h3.cell_to_parent(cell, 7)
cells = h3.h3shape_to_cells(polygon, res=9)
```

---

## 14. Van Emde Boas, y-fast trie

O(log log U) predecessor on an integer universe. Beautiful theory, ugly constants, massive memory. **Not used in production.** Recognize the name; reach for B+tree, ART, or sorted array instead.

---

## 15. Decision Matrix

| Structure | Point | Range | Order iter | k-NN | Concurrent W | Disk | Multi-dim |
|---|---|---|---|---|---|---|---|
| Skip list | ★★★ | ★★★ | ★★★ | — | ★★★★ | ★ | — |
| B+tree | ★★★★ | ★★★★ | ★★★★ | — | ★★★ | ★★★★ | — |
| Bw-tree | ★★★ | ★★★ | ★★★ | — | ★★★★★ | ★★★ | — |
| Be-tree | ★★★ | ★★★ | ★★★ | — | ★★★ | ★★★★ | — |
| Masstree | ★★★★ | ★★★ | ★★★ | — | ★★★★ | ★ | — |
| ART (ch27) | ★★★★ | ★★★★ | ★★★★ | — | ★★★ | ★★ | — |
| BIT | ★★★★ | ★★★★ (sum) | — | — | ★★ | ★★★★ | 2D |
| Segment tree | ★★★ | ★★★★ (monoid) | — | — | ★★ | ★★★ | limited |
| Interval tree | — | ★★★★ (overlap) | ★★★ | — | ★★ | ★★★ | 1D |
| kd-tree | — | ★★★ (low D) | — | ★★★★ (D<20) | rebuild | ★ | ★★★ |
| BKD | — | ★★★★ | — | ★★★ | append | ★★★★★ | ★★★★ |
| R*-tree | — | ★★★★ | — | ★★★ | ★★★ | ★★★ | ★★★★ |
| S2 | ★★★★ | ★★★★ | ★★★★ | ★★★ | ★★★★ | ★★★★★ | ★★★★ (sphere) |
| H3 | ★★★★ | ★★ | ★★ | ★★★★ (k-ring) | ★★★★ | ★★★★ | ★★★★ (hex) |

---

## 16. Pick-by-Workload

```
1. Geospatial (lat/lon or x/y/z)?
   YES → 2.    NO → 4.

2. Need uniform neighbor distance (ride match, heatmap)?
   YES → H3.   NO → 3.

3. Need range scans / B+tree indexing of cell IDs?
   YES → S2 (Hilbert-ordered uint64).
   Mutable polygons, R/W OLTP       → R*-tree (PostGIS GiST, SQLite rtree).
   Append-heavy points (logs)       → BKD (Lucene/Elasticsearch).
   Clustering in Parquet/Iceberg    → Z-order (Morton).

4. Multi-dim but not geospatial (numeric ranges, time+value)?
   YES → BKD (append-only) or R*-tree (mutable).  NO → 5.

5. Interval overlap / stabbing?
   YES → interval tree (mutable) or sorted array (static, cgranges).  NO → 6.

6. Range aggregates with point/range updates?
   sum only          → Fenwick (20 lines).
   any monoid        → segment tree (lazy).
   NO → 7.

7. In-memory ordered map, concurrent writes?
   YES → skip list (RocksDB memtable, Redis ZSET);
         variable-length/composite keys → ART (ch27).
   NO  → B+tree on disk (Postgres, InnoDB, WiredTiger);
         in-memory fixed keys → OLC B-tree or Masstree;
         write-amp sensitive → Be-tree.

8. k-NN in D < 20?
   YES → kd-tree / ball tree.
   D > 20 (embeddings) → HNSW / IVF-PQ / DiskANN → ch29.
```

### Domain defaults

| Domain | Default |
|---|---|
| LSM memtable (RocksDB, LevelDB) | skip list |
| Redis sorted set | skip list + dict (listpack small) |
| PostgreSQL PK, InnoDB, WiredTiger | B+tree |
| PostgreSQL/SQLite spatial | R*-tree (PostGIS GiST, SQLite rtree) |
| Elasticsearch geo_point / numeric range | BKD |
| MongoDB `2dsphere`, CockroachDB spatial | S2 |
| Uber dispatch, ride match | H3 |
| Time-series rank/percentile over buckets | Fenwick |
| Calendar / events overlap | interval tree |
| Genomics region overlap | cgranges (static), interval tree (mutable) |
| Parquet/Iceberg spatial clustering | Z-order |

---

## 17. Libraries (2026)

| Structure | Lang | Library |
|---|---|---|
| Skip list | C++/Java/Go | RocksDB `InlineSkipList` (embedded); `ConcurrentSkipListMap`; `huandu/skiplist` |
| B+tree | C | LMDB / libmdbx (mmap + CoW, ch19) |
| Be-tree | C | Percona ft-index (Tokutek) |
| Masstree | C++ | `kohler/masstree-beta` |
| ART | C/Rust/C++ | libart; `art-tree` crate; DuckDB source (ch27 §7) |
| BIT / Segment tree | any | roll your own (20-150 LoC) |
| Interval tree | C/C++/Py | IITree/cgranges (Heng Li); `intervaltree`, `ncls` |
| kd-tree | C++/Py | nanoflann; FLANN; scikit-learn `KDTree`; SciPy `cKDTree` |
| BKD | Java | Lucene `org.apache.lucene.util.bkd` (in ES/OpenSearch/Solr) |
| R-tree | C/C++/Rust | libspatialindex; Boost.Geometry `rtree`; `rstar` |
| R-tree (DB) | SQL | SQLite `rtree` module; PostGIS GiST/SP-GiST |
| S2 | C++/Go/Java/Py/Rust | `s2geometry` (Google); `s2` crate |
| H3 | C/Py/Go/Rust/Java | `uber/h3`; `h3`; `uber/h3-go`; `h3o` |
| Spatial Spark | Scala/Py | Apache Sedona (H3, S2, GeoHash, R-tree, KDB) |
| Spatial predicates | C++/Java | GEOS / JTS (exact polygon ops) |

```bash
pip install h3 shapely rtree pygeos                    # Python stack
cargo add h3o s2 rstar                                 # Rust
sudo apt install libh3-dev libspatialindex-dev libgeos-dev postgis  # Debian
sqlite3 :memory: "CREATE VIRTUAL TABLE t USING rtree(id, minX, maxX, minY, maxY);"
```

---

## 18. Integration

- **ch12 Observability** — Fenwick/segment tree for streaming rank/percentile over bounded integer histograms; smaller and faster than t-digest when the value domain is naturally bucketed.
- **ch14 Database Profiling** — PostGIS `EXPLAIN ANALYZE` shows GiST R*-tree scans; `&&` bbox pre-filter + `ST_Intersects` refinement is the standard pattern. MySQL `SPATIAL INDEX`, SQLite `rtree`. Watch bbox false positives.
- **ch19 Storage Engine Patterns** — RocksDB memtable = `InlineSkipList`; Lucene BKD = the numeric/geo index behind Elasticsearch segments.
- **ch27 Compact Integer Sets** — ART handles ordered integer/string keys; this chapter covers the rest.
- **ch29 Vector ANN** — kd-tree degrades > D≈20; HNSW/IVF-PQ/DiskANN beyond.

---

## 19. Diagnostic Patterns

### 19.1 "Range query slow on large table"

```
1. Is the index clustered on the range column?
     clustered (BRIN, ClickHouse ORDER BY, InnoDB PK): range = contiguous pages.
     non-clustered: row hit = random IO → switch to index-only scan or cluster.
2. Is the structure actually supporting range?
     hash on integer → rebuild as B+tree.
     BKD with wrong dim ordering → put most-selective dim first.
3. Tree depth?
     B+tree > 6 levels on SSD rare; > 3 on NVMe unusual. Deeper → pages too small or keys too wide.
```

### 19.2 "Spatial join is O(n²)"

```
Symptom: PostGIS / Sedona geo join takes forever.
Usually: missing spatial index or predicate not pushed down.
1. GIST/BKD on both sides (PostGIS) or Sedona spatial partitioner set.
2. bbox && first, exact ST_Intersects/ST_Contains after — planner does this
   unless a UDF hides the geometry.
3. Spatial partitioning (Sedona KDBTree/QuadTree/Hilbert).
4. Covering with S2/H3 cells + equi-join on cell ID when containment is fully
   inside/outside.
```

### 19.3 "R-tree not being used"

```
- Mixed SRIDs/types in geometry column → planner gives up.
- Operator doesn't match opclass (custom function instead of &&).
- Table too small — seq scan genuinely wins. EXPLAIN first, don't force.
```

### 19.4 "BKD query touches too many blocks"

```
- Leaf block size too small → raise DEFAULT_MAX_POINTS_IN_LEAF_NODE.
- Points unsorted at build → BKD bulk-build works best on sorted input per segment.
- Many small segments, each with its own BKD → force_merge to reduce count.
```

### 19.5 When NOT to use a spatial index

| Condition | Why |
|---|---|
| Cardinality < ~1000 | full scan beats index descent |
| Low selectivity (> ~20% match) | planner should ignore; if not, hint off |
| Column rewritten in bulk often | rebuild cost > query savings; consider BRIN or offline STR-pack |
| Heavy skew into one hotspot | R-tree MBR overlap collapses; bucket by H3/S2 instead |

---

## 20. References

- Pugh (1990). *Skip Lists: A Probabilistic Alternative to Balanced Trees*. CACM.
- Levandoski, Lomet, Sengupta (2013). *The Bw-Tree*. ICDE.
- Bender et al. (2015). *Introduction to Bε-trees and Write-Optimization*. USENIX ;login.
- Mao, Kohler, Morris (2012). *Masstree: Cache Craftiness for Fast Multicore Key-Value Storage*. EuroSys.
- Leis, Haubenschild, Neumann (2019). *Optimistic Lock Coupling*. DaMoN.
- Fenwick (1994). *A new data structure for cumulative frequency tables*. SPE.
- Bentley (1975). *Multidimensional binary search trees*. CACM.
- Procopiuc, Agarwal, Arge, Vitter (2003). *Bkd-Tree: A Dynamic Scalable kd-Tree*. SSTD.
- Guttman (1984). *R-Trees*. SIGMOD.
- Beckmann, Kriegel, Schneider, Seeger (1990). *The R*-tree*. SIGMOD.
- Leutenegger, Lopez, Edgington (1997). *STR: Simple and Efficient R-Tree Packing*. ICDE.
- Uber Engineering (2018). *H3: Uber's Hexagonal Hierarchical Spatial Index*. h3geo.org.
- Google S2 Geometry Library — s2geometry.io.
- Lucene docs on BKD, `LatLonPoint`, `LongPoint`.
- PostGIS docs on GiST indexing and spatial operators.
- SQLite R*Tree module — sqlite.org/rtree.html.
- Redis docs on ZSET internals (skip list + listpack) and GEO (GeoHash-encoded ZSET).
- RocksDB wiki, `memtable/inlineskiplist.h`.
- Apache Sedona docs on spatial partitioning.

---

## See Also

- [ch14 Database Profiling](14-database-profiling.md)
- [ch19 Storage Engine Patterns](19-storage-engine-patterns.md)
- [ch12 Observability Metrics](12-observability-metrics.md)
- [ch27 Compact Integer Sets](27-compact-integer-sets.md)
- [ch29 Vector ANN](29-vector-ann.md)
- [ch26 C++ HFT Optimization Patterns](26-cpp-hft-optimization-patterns.md) — order-book linear-scan result as the limiting case of "when not to use a tree".
