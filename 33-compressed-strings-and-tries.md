# Compressed Strings & Tries: FST, FSST, FM-index, Front-coding, LOUDS

Data structures for *sets and maps of strings* — terms dictionaries, autocomplete, dictionary compression, substring search in logs and DNA. The integer-set cousin is ch27 (Roaring, Elias-Fano, ART). This chapter answers: *which structure for which string workload, with how many bits per character, and at what query cost?*

**Why this chapter exists:** naive `HashMap<String, V>` or `std::set<std::string>` costs 40-80 B overhead per key and throws away prefix/suffix redundancy. On 10^8 URLs or log lines the difference between a plain trie and a front-coded FST is 10-50x memory, and between grep and FM-index is 10000x on substring queries. This decides whether a terms dict fits in RAM or thrashes SSD.

Primary sources: Lucene FST docs, Boncz/Neumann/Leis (FSST, PVLDB 2020), Navarro & Mäkinen (*Compressed Full-Text Indexes*, ACM CSUR 2007), Ferragina/Manzini (FM-index, JACM 2005), Askitis/Sinha (HAT-trie, 2007), Grossi/Gupta/Vitter (wavelet trees, SODA 2003), Aoe (double-array trie, 1989).

---

## 1. The Baseline Problem

| Structure | Memory | Point lookup | Prefix scan | Substring |
|---|---|---|---|---|
| `HashMap<String,V>` | ~40-80 B/key + string | O(\|k\|) avg | — | O(total) linear |
| Sorted `Vec<String>` | ~24 B/key + string | O(\|k\|·log n) | O(log n + k) | O(total) linear |
| Classic 256-ary trie | ~2 KB per internal node | O(\|k\|) | O(prefix + k) | — |

Classic trie dies on memory (256 ptrs × 8 B = 2 KB/node; 10^6 keys → GBs). Everything below buys the trie's query model at orders-of-magnitude less memory — or goes to compressed-text indexes where substring search is O(m) in *pattern* length, independent of corpus.

---

## 2. Pattern Classification

| Family | Representative | Mutable | bits/char (typical) | Key→V | Strengths |
|---|---|---|---|---|---|
| Path-compressed trie | **Patricia / radix** | yes | ~2 B/char | either | kernel indexes, IP routing |
| Adaptive node trie | **ART** (ch27) | yes | variable | K→V | OLTP main-mem |
| Array-backed trie | **Double-array** | rebuild | 2-8 B/char | either | static dict, tokenisation |
| Hybrid cache-aware | **HAT-trie** | yes | ~1-2 B/char | either | beats `std::map<string,V>` |
| Minimised DAG | **DAFSA** | no | < 1 B/char | key-only | spell-check |
| Minimised transducer | **FST** | no | < 1 B/char | K→V | Lucene terms dict, fuzzy/regex on compressed form |
| Succinct trie | **LOUDS / SuRF** | no | ~2 bits/node | either | range filter, memory-optimal |
| BWT-permuted trie | XBW | no | near H_k | either | genomics, research |
| Sorted delta | **Front-coding** | block-rebuild | LCP + suffix | either | Parquet dict, SSTable blocks |
| Learned symbol dict | **FSST** | static | ~0.5x raw | block codec | DuckDB/Velox string cols |
| Compressed text index | **BWT + FM-index** | no | H_k + o(n) | substring | bio, log search |
| Rank/select primitive | **Wavelet tree** | no | n·H_0 + o(n·log σ) | — | foundation for FM-index |
| Balanced rope | Rope / cord | yes | O(n) + tree | — | large-text editors |

Bold = usual default in each family.

---

## 3. Trie Refresh & Patricia / Radix Tree

Classic 256-ary trie: each node ~2 KB on 64-bit. Unusable past 10^5 keys. **Patricia / radix** (Morrison 1968) collapses runs of single-child nodes into one edge labelled by the skipped substring. Depth = number of *branching points* along key, not `|k|`.

```
keys: "roma", "romane", "romanus", "romulus"

   r
   └── "om"
        ├── "a"  ─── "ne"
        │       └─── "nus"
        └── "ulus"
```

Where it lives:
- **Linux kernel** `radix_tree` (page cache) → replaced by `xarray` in 4.20 (`include/linux/xarray.h`). Same idea, RCU-friendly API.
- **IP routing**: LC-trie / Patricia in BSD and `net/ipv4/fib_trie.c`.
- **Redis** `rax` (`src/rax.c`) for streams, cluster slots, module keys.

Pick it for mutable prefix indexes with long, sparsely-branching keys. Short high-fan-out keys → **ART (ch27 §7)** wins: four adaptive node sizes (4/16/48/256) + path compression, `log_256(N)` depth with cache-line nodes. Not repeated here.

---

## 4. Double-Array Trie (Aoe 1989)

Two `int32` arrays `base[]`, `check[]` encode a trie with O(1) transition and zero pointers.

```
transition(s, c):
    t = base[s] + c
    return (check[t] == s) ? t : FAIL
```

Memory: ~2-8 B/char of corpus after compaction. No allocator overhead, branch-predictable scans.

Lives in:
- **MeCab**, **Kuromoji** (Lucene Japanese tokenizer), **Lindera** — morphological analysis inner loop.
- `darts` / `darts-clone` (Yata), `cedar` (Yoshinaga, dynamic variant with cheaper rebuild).

Trade-off: O(|Σ|·n) construction, full rebuild on insert unless using `cedar`. Pick for static dictionaries where the O(1) transition matters (tokenisation).

---

## 5. HAT-trie (Askitis & Sinha 2007)

Burst-trie upper levels, **cache-sized array-hash buckets** at the leaves. Bucket bursts when it overflows L1/L2.

```
          [trie node]
         / | | | | \
     [bucket]   [bucket]      <- array hash of suffixes
     "erlang"   "ethernet"
     "erbium"   "ether"
```

Reported ~2x faster than `std::map<std::string,V>`, 30-50% less memory on word-frequency corpora (SIGMOD 2007). Production: `tsl::htrie_map` (header-only C++). Pick over `std::unordered_map<string,V>` when you also want ordered iteration and ART isn't available.

---

## 6. DAWG / DAFSA — static key-only dictionary

Trie + merging of equivalent subtrees → minimum deterministic acyclic automaton. **Prefix AND suffix sharing.**

```
"cat","cats","bat","bats"

trie               DAFSA
c─a─t(*)           c─┐
    └─s(*)           a─t(*)─s(*)
b─a─t(*)           b─┘
    └─s(*)
```

English wordlists (SCOWL, ~500k) land near **0.3-1 B/char of source dict**. Incremental O(n) construction on sorted input (Daciuk et al. 2000). Tools: `dawgdic`, `marisa-trie` (LOUDS-backed). Scrabble/crossword engines, Hunspell, Chrome spell-check. Key-only; for K→V go to FST.

---

## 7. FST — Finite State Transducer (Lucene, `fst` crate)

Generalises DAFSA with **outputs on transitions**, composed in a semiring (int-add, min, concat). Shared prefix *and* suffix, sum of outputs along path.

```
"cat"→3, "cats"→5, "bat"→7

               +0       +3
     start ──c──▶──a──▶──t──▶ (accept = 3)
             ▲              │
             │              └──s──▶ (accept = 5)
             │       +7
             └──b──▶──a──▶──t──▶ (accept = 7)
```

Lucene `o.a.l.util.fst.FST` and BurntSushi's Rust `fst` crate share the Mihov-Maurel minimisation. On real Lucene term dicts: **~2-4 bits per char** of terms; O(|k|) lookup returns an int (doc-freq, postings offset).

### 7.1 Why Lucene uses it

- `TermsEnum.seekExact/seekCeil` runs on the compressed bytes, no decompression.
- Intersection with any DFA — regex, wildcard, **Levenshtein** — walks the product automaton directly on FST bytes.
- 100M-term index fits in a few hundred MB, mmap-clean.

### 7.2 Levenshtein automaton intersection (Schulz & Mihov 2002)

For pattern `p` and edit distance `k`, build a DFA accepting all strings within `k` edits of `p` — O(|p|) states, constant fan-out (table-driven). Intersect with the FST → all matching terms in O(|output|) on the compressed structure.

```rust
use fst::{IntoStreamer, Streamer, Set};
use fst::automaton::Levenshtein;

let set = Set::new(mmap_bytes)?;                  // built offline
let lev = Levenshtein::new("perfomance", 2)?;     // ≤ 2 edits
let mut s = set.search(&lev).into_stream();
while let Some(term) = s.next() { /* ≤2 edits from "perfomance" */ }
```

Lucene equivalent: `FuzzyQuery` → `LevenshteinAutomata` → intersect terms FST. Slow fuzzy in ES/Solr/Tantivy is usually a *non-cached* Levenshtein DFA rebuilt per query; warm it.

Pick FST for: static K→V on string keys, memory-bound, read-heavy, need regex/fuzzy on compressed form. Skip if mutable → ART or HAT-trie.

---

## 8. LOUDS, SuRF & Succinct Tries

**LOUDS = Level-Order Unary Degree Sequence** (Jacobson 1989). Level-order traversal; per node emit `1^(#children) 0`. A trie of N nodes → **2N + o(N) bits** + N bytes of edge labels.

```
degrees 2,1,3,0,0,0,0 → bits: 11 0  1 0  111 0  0  0  0  0  0
```

Navigation uses `rank/select` on the bitvector, O(1) with `o(N)` auxiliary bits. `child(i,c)`, `parent(i)`, `first_child(i)` all in nanoseconds.

### 8.1 SuRF — Succinct Range Filter (Zhang et al., SIGMOD 2018)

LOUDS trie as LSM **range filter**: unlike Bloom (point-only), answers "any key in `[lo,hi]`?" with tunable FP, in ~**2 bits/key** dense layer + a few bits suffix. Research implementation in RocksDB forks; MyRocks explored it for range-heavy workloads. `marisa-trie` is LOUDS-based too.

Pick LOUDS/SuRF for the smallest possible static trie with `rank/select`. Only option beating Bloom on *range* predicates in an LSM.

---

## 9. XBW — eXtended Burrows-Wheeler

Ferragina/Luccio/Manzini/Muthukrishnan (2005/2009) generalise BWT to labelled trees. Rows sorted by reversed path-to-root; two arrays `(α, last)` + `rank/select` give subtree navigation. Size ~ k-th order entropy of labels + o(n). Implementations in `sdsl-lite`; used in labelled De Bruijn graphs in bio. Mostly research — pick only if LOUDS isn't compact enough.

---

## 10. Front-coding — workhorse for sorted string blocks

Sort strings, store `(LCP_with_previous, suffix)`. On ASCII URLs, LCP is typically 70-95%; block is 2-5x smaller than raw.

```
raw                   front-coded
example.com/a/1       (0, "example.com/a/1")
example.com/a/2       (14,"2")
example.com/a/3       (14,"3")
example.com/b/1       (12,"b/1")
                      → ~2-3 B/key on URL-like data
```

Access is O(block_size) forward decode → **block it**. Parquet dictionary pages (`DELTA_BYTE_ARRAY` is exactly front-coding), Cassandra/ScyllaDB SSTable index blocks, Bigtable SSTables all use 16-128-entry blocks + sparse per-block index. Combine with RePair / grammar compression for research-grade density (Martínez-Prieto et al.). Lucene BlockTree (`.tim`/`.tip`) used front-coding pre-FST and still uses it inside blocks.

---

## 11. FSST — Fast Static Symbol Table (Boncz/Neumann/Leis, PVLDB 2020)

Learns ≤255 symbols of 1-8 bytes each; encodes as 1-byte codes + escape-for-literal. Built for block-level columnar string compression where LZ4/Zstd are too slow to decompress.

Paper numbers (TPC-H, ClueWeb, real URL/email/JSON columns):
- **~2x compression** on raw strings, roughly matching Zstd default on columnar string data.
- **Decompress ~1-2 GB/s/core**, ~5-10x LZ4 on short strings.
- On top of dict encoding: **~1.2-1.5x overall vs `dict + Zstd`** but with dramatically faster scans.

```
symbol table (greedy substring counting):
  0x00 → "http://"
  0x01 → ".com"
  0x02 → "www."
  0xFF → escape (next byte literal)

"www.example.com"  → 0x02 'e''x''a''m''p''l''e' 0x01   (9 B vs 15)
```

Lives in:
- **DuckDB** default for `VARCHAR` since 0.6 (`storage/compression/fsst.cpp`).
- **Velox** (Meta/Presto-C++) column codec.
- **CedarDB/Umbra** — original home.

Reference C: `cwida/fsst` (~1500 LoC, no deps). Pick for long-tail string columns (URLs, UAs, log lines, JSON string values) when scans dominate. Skip for cold archives (Zstd wins on ratio) or high-entropy data (UUIDs, hashes — no symbols to learn).

**Tip:** stack it. `dict + FSST + Zstd` on sorted deduped values → net ~1.2x over `dict + Zstd` but 5-10x faster decompress.

---

## 12. BWT + FM-index — substring search in compressed text

BWT (Burrows-Wheeler 1994) permutes `T$` so equal contexts cluster. Ferragina-Manzini (JACM 2005): permutation + rank support answers substring queries in **O(m)** on the compressed form.

```
T = "abracadabra$"  →  BWT(T) = "ard$rcaaaabb"   (L column of sorted rotations)
```

Backward search given `rank_c(L, i)` and `C[c]` (count of symbols < c):

```
count(P):
    i = |P|-1; lo = 0; hi = n
    while i >= 0 and lo < hi:
        c  = P[i]
        lo = C[c] + rank_c(L, lo)
        hi = C[c] + rank_c(L, hi)
        i -= 1
    return hi - lo
```

One rank per step → **O(m)** total, independent of `|T|`.

Size: `|T|·H_k + o(n)` bits with run-length + wavelet-tree coding. Implementations:
- **`sdsl-lite`** (Gog/Beller/Moffat/Petri): `csa_wt<>`, `fm_index<>`, wavelet trees, rank/select.
- **BWA** (Heng Li), **Bowtie2** (Langmead) — full 3 Gbp human genome in ~4 GB RAM.
- Log-search research systems; ripgrep/Hyperscan already SIMD-scan fast enough for most page-cache-resident corpora, but FM-index wins asymptotically on TB+ log archives.

Pick for: static corpus, frequent substring/regex, pattern ≪ corpus, memory near `H_k`. Skip if grep on page cache already hits SLA.

---

## 13. Wavelet Trees (Grossi/Gupta/Vitter 2003)

Balanced binary tree over alphabet; each node stores a bitvector marking left/right at that level. Answers `rank_c(S,i)`, `select_c(S,k)`, `access(i)` in **O(log σ)** using `n·H_0 + o(n·log σ)` bits.

```
S = "abracadabra", σ = {a,b,c,d,r}
level 0 bitmap (split {a,b} vs {c,d,r}):
S    : a b r a c a d a b r a
left?: 1 1 0 1 0 1 0 1 1 0 1   → rank/select bitvector
```

The `rank_c(L, i)` call inside FM-index backward search *is* an O(log σ) wavelet-tree walk with O(1) rank on plain bitvectors. Same primitive backs Elias-Fano (ch27 §5), grammar-compressed text, succinct TSDB dicts. Libs: `sdsl-lite` (C++), `sucds`, `succinct` (Rust).

---

## 14. Ropes / cords — large-text editors

Balanced tree of string fragments. Insert/delete at `i` → O(log n); concat → O(1) amortised. SGI STL `rope`, GCC `__gnu_cxx::rope`, Rust `ropey` (Helix, Xi). **Niche**: only pick for interactive editing of multi-MB text. Any search/index problem below has a better answer above.

---

## 15. Bits-per-character Comparison

Order-of-magnitude. English wordlist (~500k words, ~5M chars) and URL list (~1M URLs, ~80M chars). Numbers are **typical** — always measure on your corpus.

| Structure | English wordlist | URL list | Notes |
|---|---|---|---|
| `Vec<String>` raw | 8 B/char (+ 24 B/key header) | 8+ B/char | baseline |
| HashMap keys | 10-12 B/char | similar | header + slack |
| Classic 256-ary trie | 100+ B/char | explodes | don't |
| Patricia / radix | 3-6 B/char | 2-4 B/char | mutable |
| Double-array | 2-4 B/char | 2-3 B/char | static |
| HAT-trie | 1-2 B/char | ~1 B/char | dynamic, cache-friendly |
| **DAFSA** (key-only) | **~0.3-0.8 B/char** | ~0.5 B/char | English suffix sharing |
| **FST** (key→int) | 0.5-1 B/char | ~0.7 B/char | Lucene terms dict |
| LOUDS trie | ~2 bits/node + labels | — | + rank/select overhead |
| **Front-coded blocks** | ~1 B/char | **0.3-0.8 B/char** | URLs share long prefixes |
| FSST alone | — | ~4 B/char (0.5x raw) | block codec |
| Dict + FSST + Zstd | — | 0.2-0.4 B/char | DuckDB/Velox default |
| FM-index | 1-2 B/char | 1-2 B/char | + substring search |

DAFSA wins wordlists via suffix sharing (`-ing`, `-ed`, `-s`). Front-coding wins URLs via long shared prefixes.

---

## 16. Decision Matrix

Stars are order-of-magnitude. Blank = not applicable.

| Structure | Mutable | Prefix scan | Range | Substring | Fuzzy/regex on compressed | bits/char |
|---|---|---|---|---|---|---|
| Patricia / radix | ★★★★ | ★★★★ | ★★ | | | ★★ |
| ART (ch27) | ★★★★★ | ★★★★★ | ★★★★ | | | ★★ |
| Double-array | rebuild | ★★★★ | ★★★ | | | ★★★ |
| HAT-trie | ★★★★ | ★★★ | ★★ | | | ★★★ |
| DAFSA | | ★★★★ | ★★★ | | weak | ★★★★★ |
| **FST** | | ★★★★★ | ★★★★ | | **★★★★★** | ★★★★★ |
| LOUDS / SuRF | | ★★★★ | ★★★★ | | | ★★★★ |
| Front-coding | block-rebuild | ★★★ | ★★★★ | | | ★★★★ |
| **FSST** | static | | | | | ★★★★ (blocks) |
| **FM-index** | | ★★ | ★★★ | ★★★★★ | ★★★★ | ★★★★ |
| Wavelet tree | | | | primitive | primitive | ★★★★ |
| Rope | ★★★★★ | | | slow | | — |

---

## 17. Pick-by-Workload Decision Tree

```
1. Substring / "find P anywhere in T"?
   YES → FM-index (sdsl-lite / BWA-style).
         Small corpus + page-cache-resident? SIMD grep (ripgrep/Hyperscan) wins absolute time.
   NO  → 2.

2. K→V or key-only, mostly static, need fuzzy/regex/prefix?
   YES → FST. Lucene/Tantivy for the full engine, Rust `fst` for the structure only.
   NO  → 3.

3. Static key-only dict (spell-check, wordlist)?
   YES → DAFSA (dawgdic) or marisa-trie (LOUDS).
   NO  → 4.

4. Dynamic ordered string map?
   YES → ART (ch27) for byte-keys. HAT-trie otherwise.
   NO  → 5.

5. Range filter in an LSM?
   YES → SuRF (LOUDS + suffix). Point-only → Bloom (ch27 §6).
   NO  → 6.

6. Compressing a columnar string block for fast scans?
   YES → Dict + FSST (DuckDB/Velox default).
         Sorted prefix-heavy (URLs, paths)? Front-coding.
         Cold archive, decompress speed irrelevant? Zstd.
   NO  → 7.

7. Editing large in-memory text?
   YES → Rope (ropey).
   NO  → `Vec<String>`/`HashMap` is fine until it isn't.
```

### Domain defaults

| Domain | Default | Reason |
|---|---|---|
| Lucene/Tantivy terms dict | FST | prefix + fuzzy + regex on compressed |
| LSM SSTable block index | Front-coding + sparse skip | compact, fast intra-block scan |
| LSM range filter | SuRF (LOUDS) or Bloom point-only | ch19, ch27 |
| Columnar string (Parquet) | Dict + front-coding (`DELTA_BYTE_ARRAY`) | canonical |
| Columnar string (DuckDB, Velox) | Dict + FSST | memcpy-speed decompress |
| CJK tokeniser (MeCab, Kuromoji, Lindera) | Double-array | O(1) transition in inner loop |
| Spell-check (Hunspell, Chrome) | DAFSA | suffix sharing |
| Linux kernel page cache | Radix → xarray | mutable, RCU |
| OLTP main-mem string map | ART (ch27) | mutable + ordered |
| Bioinformatics short-read align | FM-index (BWA, Bowtie2) | O(m) on compressed genome |
| Log pattern storage (ch12) | Drain clustering + FSST | high repetition, scan-heavy |

---

## 18. Integration with Other Chapters

- **ch12 Observability**: log pattern compression. Drain/Logstash clustering extracts templates; residual parameter columns compress brilliantly with FSST (10-20x on JSON/log archives, scans stay fast). ES `_source` stored raw → TB indexes; disable + push to columnar with FSST.
- **ch14 Database Profiling**: PostgreSQL `pg_trgm` (trigram GIN) for fuzzy; MySQL `ngram` for CJK; Oracle Text. Native slowness under load → port to a Lucene/Tantivy companion index (FST).
- **ch19 Storage Engine Patterns**: Lucene terms dict is an FST per segment; Parquet `DELTA_BYTE_ARRAY` = front-coding; DuckDB/Velox `VARCHAR` = FSST. Tuning: ch19 §5 (Parquet dict), ch19 §3 (LSM filters).
- **ch27 Compact Integer Sets**: ART detail there; `rank/select` behind Elias-Fano is the same succinct-structures family as wavelet trees here. Build one good rank/select lib, reuse everywhere (FM-index, LOUDS, PEF).
- **ch26 C++ HFT**: symbol→ID on hot paths. Closed static set of tickers → perfect hashing (ch26 §4) beats any trie. Open growing set → mmap'd small FST beats a hash map on cold cache since bytes fit in L2.

---

## 19. Libraries (production, 2026)

| Structure | Language | Library | Notes |
|---|---|---|---|
| Radix / xarray | C (kernel) | `include/linux/xarray.h` | RCU-safe, replaced `radix_tree` 4.20 |
| Radix (userspace) | C | Redis `src/rax.c`, `libradix` | embed-friendly |
| ART | multi | see ch27 §7 | DuckDB tree, `libart`, `art-tree` (Rust) |
| Double-array | C++ | `darts-clone` (Yata) | MeCab dep |
| Double-array dynamic | C++ | `cedar` (Yoshinaga) | incremental |
| HAT-trie | C++ | `tsl::htrie_map` (Tessil) | header-only |
| DAFSA | C++ | `dawgdic` (Yata) | static builder |
| DAFSA + LOUDS | C++/Py | `marisa-trie` (Yata) | Python/Rust bindings |
| FST | Java | Lucene `o.a.l.util.fst.FST` | canonical |
| FST | Rust | [`fst`](https://crates.io/crates/fst) (BurntSushi) | + `fst-levenshtein`, `fst-regex` |
| FST | Rust | `tantivy-fst` | Tantivy's vendored fork |
| LOUDS / succinct | C++ | [`sdsl-lite`](https://github.com/simongog/sdsl-lite) | wavelet tree, FM-index, LOUDS |
| LOUDS / succinct | Rust | `sucds`, `succinct` | |
| SuRF | C++ | `efficient/SuRF` | research ref, not prod-stable |
| FSST | C/C++ | [`cwida/fsst`](https://github.com/cwida/fsst) | ~1500 LoC, no deps |
| FSST | in-tree | DuckDB `src/storage/compression/fsst.cpp`; Velox `velox/dwio/.../fsst` | |
| BWT / FM-index | C++ | `sdsl-lite` `csa_wt<>`, `fm_index<>` | |
| BWT (bio) | C | BWA, Bowtie2 | genomics default |
| Rope | Rust | [`ropey`](https://crates.io/crates/ropey) | Helix descendants |

### Install / check

```bash
cargo add fst fst-levenshtein fst-regex                           # Rust FST
git clone https://github.com/simongog/sdsl-lite && cd sdsl-lite && ./install.sh ~/sdsl
git clone https://github.com/cwida/fsst && cd fsst && make        # FSST ref
pip install marisa-trie                                           # LOUDS (Python)
```

---

## 20. Diagnostic Patterns

### 20.1 "Lucene/ES terms dict too big"

```
Symptoms: heap pressure loading segments; slow TermsEnum.seekExact; multi-GB .tip/.tim.
Causes:
  - High-card keyword fields (UUIDs, IDs) → FST has no prefix to share
  - doc_values accidentally enabled alongside inverted
  - Many small segments → per-segment FST overhead
Fix:
  1. UUIDs as keyword: doc_values=false, index=true
  2. Force-merge off-peak (_forcemerge?max_num_segments=1)
  3. Consider ES `wildcard` field (7.9+) — n-gram + binary doc values,
     better for wildcard-heavy workloads than FST
```

### 20.2 "Fuzzy query slow"

```
Symptoms: P99 spikes on `match` with fuzziness; CPU on LevenshteinAutomata.
Causes: Levenshtein DFA rebuilt per query; fuzziness > 2 explodes DFA size.
Fix:
  1. Cap fuzziness at 2 (Damerau-Levenshtein DFA grows exponentially beyond).
  2. Cache DFA per term for hot queries.
  3. prefix_length ≥ 3 — anchors the intersection, cuts automaton size massively.
```

### 20.3 "Substring search slow on logs"

```
Symptoms: LIKE '%pat%' full-scan; grep-on-archives minutes/GB.
Causes: no index supports arbitrary substring (B-tree/hash/Bloom all fail).
Fix progression:
  1. ripgrep / Hyperscan on raw if page-cache-resident → often enough.
  2. Trigram index (pg_trgm GIN, ES ngram) → candidate + verification.
  3. FM-index (sdsl-lite / BWA-style build) → O(m) on compressed, TB corpora
     at a few GB RAM index.
```

### 20.4 "Columnar string column doesn't compress"

```
Symptoms: Parquet/DuckDB VARCHAR only 1.5x smaller than CSV.
Causes: high-entropy (hashes, encrypted) — compression floor; dict disabled
        above cardinality threshold; wrong codec for data shape.
Fix:
  1. Check cardinality; < 10^6 → force dict (Parquet use_dictionary=True;
     DuckDB auto).
  2. Sort by column + front-coding → 3-5x on URL-like.
  3. FSST block codec (DuckDB 0.6+, Velox; Parquet experimental) for
     memcpy-speed decompress.
  4. High-entropy: give up, spend the CPU elsewhere.
```

### 20.5 "Trie memory blew up"

```
Symptoms: OOM below theoretical key limit.
Causes: classic Σ-ary trie (Σ=256) ≈ 2 KB/internal node; or radix without
        path compression on long keys.
Fix:
  1. ART (ch27) — four node sizes, byte-granular.
  2. HAT-trie — cache-sized array-hash buckets under burst-trie.
  3. Static? Rebuild as FST/DAFSA offline, mmap at startup.
```

---

## 21. Succinct Data Structures — the shared foundation

Everything in §8, §11, §12, §13 leans on the same primitive:

```
rank_b(B, i)   = # of b-bits in B[0..i)         → O(1)
select_b(B, k) = position of k-th b-bit in B    → O(1)
```

on a bitvector of `n` bits using `o(n)` extra bits (Clark 1996, Jacobson 1989, Raman-Raman-Rao 2002). Production libs (`sdsl-lite`, `sucds`, Rust `succinct`) hit tens of nanoseconds per op with ~25% extra for rank, ~50% for select on fastest settings.

The space-time point is remarkable: `n + o(n)` bits instead of `n log n` for a tree, O(1) navigation. LOUDS, wavelet trees, FM-index, SuRF, Elias-Fano (ch27) are all different arrangements of bits such that rank/select answers the question you care about. Build one good rank/select layer; reuse everywhere.

---

## 22. Recent String-Index Structures (2025–2026)

### 22.1 4-ary Wavelet Tree with predictive prefetch

- **Structure:** wavelet tree with **4-ary internal nodes** (each node splits alphabet into 4, not 2) → tree depth cut by half, same space bound. Adds a small ML model that predicts upcoming `rank` query targets and issues `__builtin_prefetch` ahead of the actual query.
- **Pick it when:** you already run a **wavelet tree under FM-index** (substring search on logs, genomics, compressed full-text) and rank/select is the bottleneck. Drop-in replacement — no schema change.
- **Gain:** up to **2× faster rank/select** vs SDSL-lite binary wavelet tree at same space cost.
- **How:** reference impl from Kurpicz 2025 (SPE); not yet in SDSL-lite mainline. If you're building custom, the 4-ary reshape alone is a 1-afternoon engineering win.

### 22.2 r-index — repetition-aware compressed suffix array

- **Structure:** FM-index variant parameterized by **`r`** (number of runs in the Burrows-Wheeler transform), not `n`. For repetitive text, `r ≪ n` by orders of magnitude. Uses **run-length BWT** + compressed suffix array sampling at `O(r log(n/r))` positions.
- **Pick it when:** **highly repetitive corpora** — pangenomes (1000s of near-identical genomes), Git packfile indexing, versioned document stores, large log archives with templated entries. The more repetitive the data, the larger the win.
- **Gain:** typically **10-1000× smaller** than classical FM-index on human pangenome; substring search at similar speed.
- **How:** [`nicolaprezza/r-index`](https://github.com/nicolaprezza/r-index) C++. **Do not use** on non-repetitive text — you'll pay overhead without the compression.

### 22.3 Updated pick matrix — string structures in 2026

| Workload | Structure | Lib |
|---|---|---|
| Static key→value, want fuzzy queries | **FST** | [`burntsushi/fst`](https://github.com/BurntSushi/fst) (Rust), Lucene FST (Java) |
| String column compression (OLAP) | **FSST** | DuckDB, Velox native; [`cwida/fsst`](https://github.com/cwida/fsst) |
| Sorted dict, prefix-shared | **Front-coding** | Parquet page dicts (native), manual for custom formats |
| Dynamic prefix tree, integer keys | **ART** (see ch27) | `armon/libart`, DuckDB |
| Dynamic prefix tree, string keys | **HAT-trie** or **ART with byte keys** | `Tessil/hat-trie` |
| Substring search, non-repetitive text | **FM-index + wavelet tree** | `sdsl-lite` (consider 4-ary variant) |
| Substring search, **repetitive** text | **r-index** | `nicolaprezza/r-index` |
| Log pattern compression | **FSST** + Zstd-dict fallback | FSST first, Zstd on residual if worth it |

### 22.4 What stayed true

- FST is still the default for immutable `string → payload` maps with fuzzy/regex query needs. The Lucene/Tantivy design from ~2013 remains unmatched in its niche.
- FSST before Zstd, not after. Applies in 2026 same as 2020.
- ART is the right mutable trie. Libart / DuckDB integration is stable.
- SuRF (Succinct Range Filter) still useful for LSM range-lookup filtering.

### 22.5 What not to bother with in 2026

- Rebuilding a DAWG by hand when you have FST in a library — FST strictly dominates on updateable-ish workloads.
- Classical FM-index for versioned / repetitive corpora — r-index is 100× smaller.
- Custom Patricia trie — ART is the Patricia trie you want, plus adaptive node sizes.

---

## 23. References

- Ferragina, Manzini. *Indexing Compressed Text*. JACM 52(4), 2005.
- Burrows, Wheeler. *A Block-Sorting Lossless Data Compression Algorithm*. DEC SRC TR 124, 1994.
- Boncz, Neumann, Leis. *FSST: Fast Random Access String Compression*. PVLDB 13(11), 2020.
- Grossi, Gupta, Vitter. *High-Order Entropy-Compressed Text Indexes*. SODA 2003.
- Jacobson. *Space-efficient static trees and graphs*. FOCS 1989.
- Zhang et al. *SuRF: Practical Range Query Filtering with Fast Succinct Tries*. SIGMOD 2018.
- Askitis, Sinha. *HAT-trie*. SIGMOD 2007.
- Aoe. *An Efficient Digital Search Algorithm by Using a Double-Array Structure*. IEEE TSE, 1989.
- Daciuk, Mihov, Watson, Watson. *Incremental Construction of Minimal Acyclic FSA*. Comp. Linguistics, 2000.
- Schulz, Mihov. *Fast String Correction with Levenshtein Automata*. IJDAR, 2002.
- Navarro, Mäkinen. *Compressed Full-Text Indexes*. ACM CSUR, 2007.
- Morrison. *PATRICIA*. JACM, 1968.
- Gog, Beller, Moffat, Petri. *From Theory to Practice: Plug and Play with Succinct Data Structures*. SEA 2014.
- Gallant. [*Index 1,600,000,000 Keys with Automata and Rust*](https://burntsushi.net/transducers/). 2015.
- McCandless, Hatcher, Gospodnetić. *Lucene in Action*, 2nd ed.
