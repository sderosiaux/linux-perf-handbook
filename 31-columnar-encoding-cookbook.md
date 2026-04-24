# Columnar Encoding Cookbook: Parquet, ORC, Arrow, ClickHouse

Per-column encoding choices in columnar formats. ch19 §5 is the high-level map; this is the cookbook: given a schema + workload, pick the right encoding per column and the right compressor on top. Primary sources: Parquet spec, ClickHouse docs, Gorilla paper (Pelkonen VLDB 2015), FSST paper (Boncz VLDB 2020), ALP paper (Afroozeh/Kuschewski/Boncz SIGMOD 2023), Dremel (Melnik VLDB 2010).

Columnar wins come mostly from *encoding* (dict/RLE/delta/FOR/bit-pack), not *compression* (Zstd/LZ4). Wrong encoding → Zstd rescues the ratio at 5–20× the decode CPU. Right encoding → SIMD decodes at near-memcpy, general compressor has almost nothing left.

---

## 1. Two-layer model

```
raw values → LOGICAL ENCODING → GENERAL COMPRESSION → bytes
             dict/RLE/bitpack   Zstd/LZ4/Snappy
             delta/FOR/Gorilla  (Parquet "compression",
             FSST/ALP           CH final codec)
             (Parquet "encoding",
              CH codec chain)
```

Why this order:
- Encoding removes *structural* redundancy Zstd can't see — a run of `(42, 42, 42, …)` → one RLE token vs full LZ77 over every byte.
- Decoders vectorize. Bit-packed ≥10 GB/s/core; Zstd-only 1–2 GB/s.
- Predicate pushdown (min/max, bloom) works on encoded pages, no decompress.

TRADEOFF: double-compressing well-encoded columns with Zstd-9 adds 2–5% size at 5–10× decode cost (§9.5).

---

## 2. Pattern classification

| Encoding | Best for | Type | Ratio (typical) | Decode | Random access |
|---|---|---|---|---|---|
| PLAIN | fallback, high-entropy | any | 1× | memcpy | O(1) |
| Dictionary | low-card strings/enums | any | 5–100× | SIMD gather | O(1) via idx |
| RLE | constant runs | int/bool | 10–1000× | O(runs) | O(log runs) |
| RLE+Bitpack hybrid | Parquet V2 default | int | 3–20× | SIMD | O(1) in run |
| Bit-packing | narrow-range ints | int | 2–8× | SIMD BP128 | O(1) |
| FOR / PFOR | clustered ints | int | 2–10× | SIMD + patch | O(1) |
| Delta / DoD | timestamps, monotonic IDs | int | 10–50× | sequential | block-level |
| Gorilla XOR | TSDB floats | f64 | 1.4–4.5× | bit-by-bit | block |
| Chimp / Chimp128 | TSDB floats | f64 | 1.5–5× | semi-SIMD | block |
| ALP | general floats | f64/f32 | 2–4× | SIMD | block |
| FSST | short strings | varchar | 2–4× | ~memcpy | O(1) |
| Front-coding | sorted dict | str | 2–8× | sequential | prefix seek |
| BYTE_STREAM_SPLIT | float columns | f32/f64 | boosts LZ4/Zstd | shuffle | — |

Numbers are order-of-magnitude, shape-dependent.

---

## 3. Dictionary encoding — 90% default for strings

```
Column chunk
 ├─ Dictionary page    PLAIN unique values, written once
 └─ Data pages         RLE_DICTIONARY: per-row indices (bit-packed + RLE-hybrid)
```

256 uniques → 8-bit indices → 32× shrink vs 32-byte strings *before* any Zstd.

**Fallback threshold.** Parquet bails to PLAIN when the dict grows past `parquet.dictionary.page.size` (default **1 MB**). UUIDs / free text / hashes hit the fallback → ratio collapses.

```python
pq.write_table(table, "data.parquet",
    use_dictionary=["status", "country", "category"],
    dictionary_pagesize_limit=2 << 20,   # bump from 1 MB
    data_page_version="2.0")             # RLE_DICTIONARY fast path
```

**Sorted dict → range pushdown.** Sorted dict turns `WHERE country BETWEEN 'A' AND 'D'` into a contiguous index range → bitmap scan. arrow-rs + DuckDB sort by default; Parquet spec doesn't mandate it.

Wins: `enum`, `status`, `country_code`, `log_level`, low-card categoricals (<~64K).
Loses: UUIDs, emails, free text → PLAIN fallback + Zstd.

---

## 4. RLE and RLE-Bitpack hybrid

Classic RLE: `(value, run_length)` pairs. Collapses on sorted / sparse / boolean flag columns. Parquet's `RLE` is a **hybrid** — per run, a 1-bit header picks bit-packed-run (multiples of 8) or rle-run (fixed-width repeated value). Byte-aligned, SIMD-decodable. Used for: definition/repetition levels (every nested column, Dremel encoding), dict indices (`RLE_DICTIONARY`), booleans in V2.

Collapses when avg run < 4. Quick check: `runs = 1 + np.sum(arr[1:] != arr[:-1]); avg_run = len(arr)/runs`.

---

## 5. Bit-packing (BP128)

Pack N values into `N × bit_width` bits.

| Range | Bits | vs int32 |
|---|---|---|
| [0,1] | 1 | 32× |
| [0,255] | 8 | 4× |
| [0,65535] | 16 | 2× |
| [0,16M] | 24 | 1.33× |

Lemire **SIMD-BP128**: 128 values at a time with AVX2/AVX-512, 3–5 Gints/s/core. Used by FastPFor, Arrow, Parquet decoders, Lucene postings. Parquet bit-packs def/rep levels + dict indices when card < 2^bit_width. Bit-pack + LZ4 is a sweet spot; bit-pack + Zstd-9 wastes CPU (already dense).

---

## 6. Frame-of-Reference (FOR) and PFOR-Delta

Per-block: subtract min, bit-pack residuals.

```
block = [1_000_000, 1_000_007, 1_000_003, 1_000_015, ...]
ref   = 1_000_000; residuals = [0, 7, 3, 15, ...]       → 4–5 bits each
```

Vertica (C-Store), parts of ClickHouse, Arrow use FOR. Clustered ints land in 2–6 bits/value.

**PFOR / PFOR-Delta** (Zukowski et al. ICDE 2006): bit-width fits 90%+, outliers to a patch list. Handles skew. **Delta-FOR** = delta first, FOR second. Parquet `DELTA_BINARY_PACKED` emits block-of-mini-blocks delta with per-mini-block FOR — variable-variance safe.

---

## 7. Delta and Delta-of-Delta (Gorilla)

`d_i = v_i - v_{i-1}`, `dd_i = d_i - d_{i-1}`. Constant-interval series → DoD is 0 → 1 bit/sample "same stride". Gorilla (Pelkonen VLDB 2015): **~1.37 B per `(ts, value)` pair** on Facebook TSDB — ~12× vs raw 16B pairs. Timestamp code: 0 → 1 bit; [-63,64] → 2+7 bits; [-255,256] → 3+9; [-2047,2048] → 4+12; else → 4+32.

ClickHouse exposes this as first-class codecs:

```sql
CREATE TABLE metrics (
    ts    DateTime64(9) CODEC(DoubleDelta, ZSTD(1)),
    value Float64       CODEC(Gorilla,     ZSTD(1))
) ENGINE = MergeTree ORDER BY ts;
```

`DoubleDelta` targets monotonic near-constant-stride; compose with a general codec for residual entropy.

---

## 8. Float encoding — Gorilla, Chimp, ALP

**Gorilla XOR (2015).** XOR with previous. Slowly drifting → leading+trailing zeros → encode `(leading_zeros, meaningful_bits)`. Repeated → 1 bit. Paper: ~0.6–1.2 B/float for slow metrics, ~8 B/val for random floats.

**Chimp / Chimp128 (Liakos VLDB 2022).** XOR against the best of last 128 values, Huffman on leading-zero counts. Paper: 5–50% smaller than Gorilla, 2–3× faster decode. DuckDB shipped Chimp/Patas before ALP.

**ALP (Afroozeh/Kuschewski/Boncz SIGMOD 2023) — current SOTA.** Adaptive Lossless floating-Point. Most real-world doubles are decimals with bounded significant digits (`1.234`, `99.95`). ALP finds per-block `(e, f)` s.t. `value ≈ (integer × 10^e) × 10^(-f)`, bit-packs integers with FOR, patches exceptions. Paper: beats Chimp128, matches/beats Zstd-on-raw-floats on IoT / scientific / financial data, decode 2–5× faster than Chimp. **Used in DuckDB 1.0+.**

NOTE: Gorilla fine for pure TSDB with slowly-changing values. **ALP supersedes Gorilla for general float columns in 2026** — pick ALP if your system offers it.

**BYTE_STREAM_SPLIT (Parquet).** Preprocessor, not compressor. Split N floats into 4 byte-streams (byte 0 of each, then byte 1, …) → lower per-stream entropy → Zstd/LZ4 compresses 10–30% better. Trivial decode. Default in `parquet-cpp` for float columns with Zstd.

### Float decision table

| Shape | Pick |
|---|---|
| TSDB timestamps | `DELTA_BINARY_PACKED` / `DoubleDelta` |
| TSDB values, slow drift | Gorilla or ALP |
| Decimals (prices, measures) | ALP (big win) |
| Scientific mixed | ALP or Chimp128 |
| Noisy sensor | `BYTE_STREAM_SPLIT` + Zstd |
| Already-compressed blobs | none / Zstd only |

---

## 9. String encoding and general compression

### 9.1 FSST (Boncz/Neumann/Leis VLDB 2020)

Fast Static Symbol Table. Trained dict of ≤255 byte codes for frequent 1–8 byte substrings; encoded strings swap substrings for 1-byte codes. Decode = single-byte table-lookup loop, **~2–3 GB/s/core (near memcpy)**, vs Zstd dict mode ~500 MB/s. Paper: 2–3× on short strings (URLs, log lines, names, JSON keys) where dict fails on cardinality. **DuckDB default for long strings.** Parquet has an open proposal; CH uses LZ4/Zstd.

Use when: high card (dict fails), short strings (<128 B), random-access decode matters. Skip for large blobs / already-compressed.

### 9.2 Front-coding + Parquet `DELTA_BYTE_ARRAY`

`(shared_prefix_len, suffix)` per value. Sorted URL/path/domain dicts → **2–8× vs PLAIN**. Random access needs a skip list. Used by LevelDB/RocksDB SST blocks, Tantivy term dict, Lucene FSTEnum. Parquet's `DELTA_BYTE_ARRAY` is front-coding inside the spec (all major writers support it).

### 9.3 General compression layer

| Codec | Decode | Ratio | Use |
|---|---|---|---|
| Snappy | ~2 GB/s | baseline | legacy Hadoop |
| LZ4 | 3–5 GB/s | ~Snappy +5% | hot analytics; CH default |
| LZ4HC | 3–5 GB/s decode, slow encode | ~Zstd-1 | write-once |
| Zstd(1) | 1–2 GB/s | Snappy +15–25% | general default |
| Zstd(3) | ~900 MB/s | +25–35% | balanced |
| Zstd(9) | ~500 MB/s | +35–50% | cold |
| Zstd(22) | ~200 MB/s | +50%+ | archival |
| Brotli | ~400 MB/s | text-heavy | rare in columnar |

Order-of-magnitude on modern x86; verify.

### 9.4 Trained Zstd dictionaries

`zstd --train samples -o dict` builds a ~100 KB corpus-tuned dict. Small payloads (<1 KB) sharing structure — JSON logs, protobufs, URLs — compress **2–5× better with a trained dict** per Zstd docs. Matters for: small row groups (<1 MB) where in-frame history is thin; per-message compression (Kafka, Redis — ch22/ch21); many columns with similar JSON/XML structure. Doesn't help for large RGs (>64 MB). CH/Parquet writers don't use trained dicts by default.

### 9.5 Don't double-compress

- **Dict + Zstd-9**: indices are near-uniform 8-bit ints post-dict. Zstd-9 saves ~3% at 5× decode cost. Use `Zstd(1)` or `LZ4`.
- **FSST + Zstd**: FSST output ≈ random byte codes. Adds 2–5% at big CPU cost.
- **Gorilla / ALP + Zstd-9**: already bit-dense. `Zstd(1)` picks up maybe 5%. Skip `Zstd(9+)`.

Rule: **cap Zstd at 3 after good encoding.** Move to 9+ only for cold/archival (ch19, ch25).

---

## 10. Parquet / ORC / Arrow layout

```
File
 ├─ Row Group 0                   ← parallelism & IO unit
 │   └─ Column Chunk (col A)      ← encoding/compression per column
 │       ├─ Dictionary Page
 │       └─ Data Pages (encoded + compressed + stats)
 ├─ ...
 └─ Footer: schema + RG metadata + Column Index (V2) + Offset Index (V2)
```

**Row group sizing (ch04).** Worker gets one RG at a time → lower bound on parallelism. `target_rg ≈ total_data / (target_workers × 2)`, cap 512 MB–1 GB.

| Size | Use |
|---|---|
| 32 MB | very selective queries, small files |
| **128 MB (default)** | balanced |
| 512 MB | Spark/Presto scans |
| 1 GB | archival, infrequent scan |

**Page size.** Default **1 MB** (`parquet.page.size`). Random-access granularity: predicate pushdown skips per-page via Column Index; a single-row match still materializes a 1 MB page. Don't go below ~64 KB (header overhead).

**Column Index + Offset Index (V2).** Per-page min/max in the footer, separate from RG stats → page-level pruning. DuckDB, DataFusion, arrow-rs, parquet-java use it. **Enable it** — older writers default to RG-only.

```python
pq.write_table(table, "data.parquet",
    write_statistics=True,
    write_page_index=True,          # pyarrow >= 13
    data_page_version="2.0")
```

**Bloom filters (ch27).** Parquet V2 per-column blooms for high-card equality predicates (user/trace IDs) where min/max can't help. Sizing per ch27 §6.2: 10 bits/key ≈ 1% FPR.

```python
pq.write_table(table, "f.parquet",
    bloom_filters=["trace_id", "user_id"], bloom_filter_fpp=0.01)
```

**ORC.** ≈ Parquet + built-in stripe indexes every 10 000 rows + native column blooms (pre-dated Parquet's) + RLE-V2. Often better compression at comparable CPU — Parquet won the ecosystem.

**Arrow IPC / Feather.** Uncompressed by default — zero-copy in-memory is the point. Optional LZ4/Zstd per-buffer (Feather V2). Typical: Arrow IPC for hot data + Parquet for cold.

---

## 11. ClickHouse codecs — composition model

`CODEC(a, b, c)` chains left-to-right.

| Codec | Purpose |
|---|---|
| `Delta(N)` | delta, N-byte stride — monotonic ints |
| `DoubleDelta(N)` | DoD — near-constant stride |
| `Gorilla(N)` | XOR float — slowly changing |
| `FPC` | float compression — scientific |
| `T64` | 64-row bit-packing — narrow-range ints |
| `GCD` | divide by GCD per block |
| `ZSTD(1..22)` / `LZ4` / `LZ4HC(l)` | general |

Real ClickHouse-docs examples:

```sql
-- OTel traces (ClickHouse docs)
CREATE TABLE otel_traces (
    Timestamp  DateTime64(9) CODEC(Delta(8), ZSTD(1)),
    TraceId    String        CODEC(ZSTD(1)),
    Duration   Int64         CODEC(ZSTD(1)),
    ServiceName LowCardinality(String) CODEC(ZSTD(1))
) ENGINE = MergeTree ORDER BY (ServiceName, Timestamp);

-- Stack Overflow posts (ClickHouse docs)
CREATE TABLE posts (
    Id          Int32  CODEC(Delta(4), ZSTD(1)),
    ViewCount   UInt32 CODEC(Delta(4), ZSTD(1)),
    AnswerCount UInt16 CODEC(Delta(2), ZSTD(1)),
    LastEditDate DateTime64(3, 'UTC') CODEC(Delta(8), ZSTD(1))
) ENGINE = MergeTree ORDER BY (PostTypeId, toDate(CreationDate));
```

Patterns: correlated with sort key → `Delta, ZSTD(1)` (clustering). Near-constant-stride ts → `DoubleDelta, ZSTD(1)`. Float metrics → `Gorilla, ZSTD(1)` (or `FPC` for scientific). Low-card strings → `LowCardinality(String) CODEC(ZSTD(1))` (LowCardinality = CH dict encoding). Hashes / trace IDs → `ZSTD(1)` only. Cluster default if unspecified: `LZ4`.

---

## 12. Vectorized decoding

Modern engines (DuckDB, Velox, DataFusion, ClickHouse) decode into vectorized batches (1024/2048 rows), never row-at-a-time:

- Bit-packed decode: one AVX2 instruction unpacks 8× 8-bit-packed ints, 4+ GB/s/core.
- `RLE_DICTIONARY` fast path: whole page = single run of same dict index → emit constant vector, predicate short-circuits ("value in [min, max]?" → 1 compare per page).
- SIMD gather: dict decode = `indices → gather(dict)` → string_view batch.

Velox and DuckDB publish: **encoded columns often decode faster than PLAIN** on selective queries — engine skips pages via stats, and bit-packed decode is per-byte faster than `memcpy` of filtered 8-byte values.

---

## 13. Decision tree — per-column encoding

```
INTEGER
 ├─ low cardinality (<64K)     → DICTIONARY (sorted if possible)
 ├─ boolean/flag               → RLE (or bit-pack hybrid)
 ├─ narrow range (<24 bits)    → BIT_PACKING (Parquet hybrid handles it)
 ├─ clustered around base      → FRAME-OF-REFERENCE
 ├─ monotonic / time-ordered
 │   ├─ near-constant stride   → DELTA-OF-DELTA (DoubleDelta / DELTA_BINARY_PACKED)
 │   └─ variable stride        → DELTA          (DELTA_BINARY_PACKED)
 └─ high entropy (hashes/IDs)  → PLAIN + ZSTD(1)

FLOAT / DOUBLE
 ├─ TSDB slow drift            → GORILLA (or ALP)
 ├─ decimals (prices, measures)→ ALP                    [DuckDB SOTA]
 ├─ scientific                 → ALP or CHIMP128
 └─ noisy / random             → BYTE_STREAM_SPLIT + ZSTD

STRING
 ├─ low card (<64K)            → DICTIONARY
 ├─ sorted + shared prefixes   → DELTA_BYTE_ARRAY / front-coding
 ├─ short high-card            → FSST                   [DuckDB default]
 ├─ long / free text           → PLAIN + ZSTD(3-9)
 └─ already-compressed         → PLAIN + NONE

BOOLEAN                        → RLE or BIT_PACKED (hybrid)

General compression layer:
  hot dashboards               → LZ4 (or ZSTD(1))
  mixed R/W                    → ZSTD(1-3)              ← default 2026
  cold                         → ZSTD(9)
  archival                     → ZSTD(22)
  after FSST/Gorilla/ALP       → LZ4 or ZSTD(1) only
```

---

## 14. Diagnostic patterns

**"Parquet scan is slow"** — check in order:
1. Pushdown firing? `parquet-tools meta f.parquet | grep statistics`. Missing → rewrite with `write_statistics=True`.
2. Column Index? Missing → rewrite with `data_page_version=2.0 + write_page_index=True`.
3. Filter column sorted? If not, per-page stats span whole domain → no pruning. Fix: `ORDER BY filter_col` on write.
4. Dict undersized? PLAIN fallback visible in parquet-tools → raise `dictionary_pagesize_limit`.
5. RG too large for parallelism? Target `num_row_groups ≈ 2 × concurrent_workers`.

**"ClickHouse column is huge despite ZSTD"**

```sql
SELECT name, compression_codec,
       formatReadableSize(data_compressed_bytes)   AS comp,
       formatReadableSize(data_uncompressed_bytes) AS raw,
       data_uncompressed_bytes / data_compressed_bytes AS ratio
FROM system.columns WHERE table = 'my_table' ORDER BY data_compressed_bytes DESC;
```

Low ratio on monotonic int → missing `Delta`. Timestamp → missing `DoubleDelta`. Low-card string → should be `LowCardinality(String)`. TSDB float → missing `Gorilla`.

**"Small Parquet files, poor compression"** — RGs <16 MB starve Zstd. Compact (Iceberg / Delta OPTIMIZE), use a trained Zstd dict (§9.4), or switch to encoding-heavy path (Delta-FOR, FSST).

**"Decode CPU is 80% of query time"** — over-compressed. Drop `ZSTD(9)` → `ZSTD(1)` / `LZ4`. Check double-compression (§9.5). Verify vectorized decode (DuckDB `EXPLAIN ANALYZE` — look for per-page scan).

---

## 15. Libraries

| Scope | Library |
|---|---|
| Parquet C++/Python | `apache/arrow` (parquet-cpp, pyarrow) |
| Parquet Java | `apache/parquet-java` |
| Parquet Rust | `apache/arrow-rs` (`parquet` crate) |
| Parquet Go | `apache/arrow-go` |
| ORC | `apache/orc` |
| ClickHouse | `clickhouse/clickhouse` |
| DuckDB (native + Parquet + ALP) | `duckdb/duckdb` |
| FSST reference | `cwida/fsst` |
| Zstd + dict trainer | `facebook/zstd` |
| Velox / DataFusion | `facebookincubator/velox`, `apache/datafusion` |

```bash
# Inspect Parquet
python -c "import pyarrow.parquet as pq; m=pq.read_metadata('f.parquet'); print(m.row_group(0).column(0).statistics)"
cargo install parquet --features cli && parquet-schema f.parquet
duckdb -c "SELECT * FROM parquet_metadata('f.parquet');"

# Train a Zstd dict
zstd --train samples/*.json -o log.dict --maxdict=100K
zstd -D log.dict -19 big.json -o big.json.zst

# ClickHouse: codec per column
clickhouse-client --query "SELECT name, type, compression_codec FROM system.columns WHERE table='otel_traces'"
```

---

## 16. Integration

- **ch04 Disk & Storage** — row group ↔ IO read unit. 128 MB RG aligns with 128 MB readahead; 1 GB RG needs bigger readahead or bottlenecks on seeks.
- **ch14 Database Profiling** — CH codec tuning, DuckDB `EXPLAIN ANALYZE`, Postgres TOAST (LZ4/pglz). Use this chapter to pick `CODEC(...)`.
- **ch19 Storage Engine Patterns** — §5 stays high-level; this is the deep reference.
- **ch22 Kafka** — trained Zstd dicts (§9.4) beat producer compression tuning for small-message topics.
- **ch25 Big Data ML** — Parquet is the interchange; writer settings determine every downstream reader's cost.
- **ch27 Compact Integer Sets** — Parquet/ORC blooms = ch27 Bloom family per column chunk. Column Index + min/max = sparse immutable random-access pattern.

---

## 17. Next-Gen Columnar Encodings & Formats (2025–2026)

### 17.1 ALP — the new default for float columns

- **Structure:** decomposes each IEEE-754 double as `value = decimal_int × 10^exponent`, stores `decimal_int` (integer, highly compressible via FOR/BP) + a per-block exponent `e`. Falls back to patched bit-packing when `e` doesn't exist.
- **Pick it when:** columns of **real-world floats** with decimal representation (prices, measurements, sensor readings, IEEE-754 doubles from ML). Classical Gorilla/Chimp miss these.
- **Measured gain vs Gorilla/Chimp:** typical 2-5× better compression ratio **and** 2-3× faster decode. Matches/beats Zstd-on-raw-floats in ratio while decoding at SIMD speed.
- **How:**
  - DuckDB ≥1.0: native ALP for `DOUBLE`/`FLOAT` columns (automatic).
  - Parquet: **arriving late 2026** (community evaluation Oct 2025, Iceberg dev thread Jan 2026). Track `parquet-java` and `arrow-rs/parquet` release notes.
  - Reference impl: [`cwida/ALP`](https://github.com/cwida/ALP) C++ header-only.

### 17.2 Falcon — GPU-native ALP

- **Structure:** ALP reimplemented for GPU execution — block-parallel exponent search, DMA-friendly memory layout, lossless decimal recovery via SIMD integer math.
- **Pick it when:** **GPU query path** (RAPIDS, cuDF, Velox-GPU, Theseus). CPU ALP ceiling on decode throughput is ~2-3 GB/s/core; Falcon pushes far beyond.
- **How:** research-stage (arXiv:2511.04140); precursor of the GPU-ALP that will land in RAPIDS' cuDF pipeline.

### 17.3 G-ALP writer configuration for GPU readers

- **What it is:** not a new encoding — a **Parquet writer config profile** optimized for GPU reads. Parquet's CPU-centric defaults (128 MB row group, 1 MB page, dict fallback at 1 MB) waste GPU parallelism.
- **Pick it when:** writing Parquet **specifically to be read on GPU** (ML training datasets, cuDF-backed analytics).
- **How (concrete writer knobs):**
  ```python
  # pyarrow GPU-oriented Parquet profile
  pq.write_table(
      table,
      'gpu.parquet',
      row_group_size=32 * 1024 * 1024,      # 32 MB (vs default 128 MB) — more parallelism per file
      data_page_size=256 * 1024,            # 256 KB (vs 1 MB) — finer grained random access
      dictionary_pagesize_limit=4 * 1024 * 1024,   # 4 MB (vs 1 MB) — avoid fallback
      compression='zstd', compression_level=1,
      use_dictionary=True,
      write_statistics=True,
  )
  ```
  Effective read bandwidth reported: **up to 125 GB/s** on A100 with matching cuDF reader.

### 17.4 Lance — random-access columnar format

- **Structure:** column chunks with **per-column adaptive encoding choice** (FOR, delta, dict, bit-packing) + Lance-specific directory for O(1) row-level access. No forced row-group boundary; pages are individually addressable.
- **Pick it when:** you need **both** analytic scan speed **and** row-level point access on the same data. ML feature stores, vector-search result materialization, training/eval dataset sharing.
- **Why not Parquet for this:** Parquet row-level access costs 1+ IOPS *per column* + large read amp because page boundaries don't align with row access patterns.
- **How:** `pip install pylance` → `lance.write_dataset(df, 'path.lance')`. LanceDB layers ANN on top.

### 17.5 FastLanes — SIMD/GPU-first file format

- **Structure:** format designed around **cascading light encodings** (FSST → ALP → FOR → bit-pack) optimized for SIMD. Explicitly supports **multi-column compression** (MCC) exploiting cross-column correlations (e.g., `timestamp` + `sequence_id` as one compressed unit).
- **Pick it when:** **new pipeline, free to pick format**, SIMD/GPU compute, very large dataset where Parquet's Thrift metadata overhead matters. Not for interoperability-critical use cases (Parquet/Arrow rules there).
- **How:** [cwida/FastLanes](https://github.com/cwida/FastLanes). Research-stage but Boncz/CWI is the source that gave us FSST and ALP — track it.

### 17.6 Decision update — pick by data type + reader

| Column type | CPU reader 2026 | GPU reader 2026 | Immutable index / cold storage |
|---|---|---|---|
| Float64 measurements | **ALP** (DuckDB today; Parquet late-26) → fallback Gorilla/Chimp | **Falcon / cuDF ALP** | ALP + Zstd(1) |
| Float32 embeddings | ALP if decimals structured, else **bit-pack + Zstd** or **halfvec quantize** (see ch29) | Falcon, or raw bit-pack | dict if common values |
| Monotonic int (timestamps, IDs) | **Delta / Delta-of-delta + BP** (Parquet V2 native) | same, SIMD BP128 | same + Zstd |
| Low-cardinality string | **Dict + RLE-BP** (Parquet `RLE_DICTIONARY`) | Dict + SIMD scatter | Dict + front-coding |
| Long text / URLs / JSON | **FSST** (DuckDB/Velox); Zstd-dict for cold tiers | FSST | FSST + Zstd-dict |
| Mixed row-level + scan access | — | — | **Lance format** |

### 17.7 What stayed the same

- **FSST for strings** still dominates — use it *before* Zstd, not after (FSST decode ~2-3 GB/s/core; Zstd on FSST output usually hurts, not helps).
- **Delta-of-delta for timestamps** is still ~1 bit/sample on constant-interval series.
- **Row group sizing rule:** CPU readers → 128 MB default is fine. GPU readers → drop to 16-32 MB per §17.3.
- **Zstd dict training** (§9.4) still gives 2-5× on small-message topics — no 2025 paper replaced this technique.

---

## 18. References

- Melnik et al. (2010). *Dremel*. VLDB.
- Pelkonen et al. (2015). *Gorilla*. VLDB. (~1.37 B/ts)
- Zukowski, Heman, Nes, Boncz (2006). *Super-Scalar RAM-CPU Cache Compression* (PFOR). ICDE.
- Lemire, Boytsov (2015). *Decoding billions of integers per second through vectorization*. SPE.
- Boncz, Neumann, Leis (2020). *FSST: fast random access string compression*. VLDB.
- Liakos et al. (2022). *Chimp: Efficient Lossless FP Compression for Time Series*. VLDB.
- Afroozeh, Kuschewski, Boncz (2023). *ALP: Adaptive Lossless FP Compression*. SIGMOD.
- [Parquet encodings spec](https://parquet.apache.org/docs/file-format/data-pages/encodings/), [implementation status](https://parquet.apache.org/docs/file-format/implementationstatus/)
- [ClickHouse column compression codecs](https://clickhouse.com/docs/sql-reference/statements/create/table#column_compression_codec)
- [DuckDB storage internals](https://duckdb.org/docs/internals/storage.html)
- [Zstd dictionary compression](https://github.com/facebook/zstd#dictionary-compression)
- [Apache ORC spec](https://orc.apache.org/specification/)

---

## See Also

- [Storage Engine Patterns](19-storage-engine-patterns.md)
- [Disk & Storage](04-disk-storage.md)
- [Database Profiling](14-database-profiling.md)
- [Big Data ML Platforms](25-big-data-ml-platforms.md)
- [Compact Integer Sets](27-compact-integer-sets.md)
