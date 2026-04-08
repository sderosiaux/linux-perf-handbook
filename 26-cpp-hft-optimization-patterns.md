# C++ HFT Optimization Patterns

Production-grade low-latency C++ patterns from the trading industry. Curated for engineers tuning hot paths down to the nanosecond, and designed to be consumable by LLM-driven autoresearch loops.

**Why this chapter exists:** most performance literature stops at "use a profiler". HFT firms (Optiver, Hudson River Trading, Citadel, IMC, Jump, Jane Street) have collectively spent billions to push C++ wire-to-wire latency under 1 microsecond. The patterns below come from talks, blog posts and papers their engineers have made public — primary sources only.

---

## 1. The Latency Hierarchy

The whole game is choosing which slowdown to fight. Numbers below are realistic order-of-magnitude on a tuned, colocated server (2024-2026).

```
                 latency       what dominates                  who uses it
FPGA hot path    ~750  ns      gate propagation                Optiver, HRT (FPGA)
ef_vi / DPDK     ~700  ns      DMA + userspace poll            HFT software gateway
Onload TCPDirect ~1    us      userspace TCP stack             order entry
Onload pipes RTT ~940  ns      userspace IPC                   inter-process
SHM queue write  ~30   ns      atomic store + memcpy           strategy fan-out
Order book ins   ~22   ns      linear scan + insert            tick handling
L1 hit            4    cyc     register reload                 hot data
L2 hit           ~10   cyc                                     warm data
L3 hit           ~40   cyc                                     LLC
DRAM fetch       ~200  cyc     ~80-100 ns                      cold data
Mispredict       ~15-20 cyc    pipeline flush                  bad branch
Linux syscall    ~100-300 ns   ring transition                 avoid in hot path
TLB miss + walk  ~30-100 ns    page table walk                 fix with hugepages
Context switch   ~3    us      TLB + cache trash               avoid via isolcpus
Kernel UDP RTT   ~3    us      socket+stack                    bypass
```

Source: Gross CppCon 2024 (Optiver), Rigtorp benchmarks, AMD/Solarflare published numbers, NVG Associates FPGA reports.

---

## 2. Pattern Classification

| Family | Goal | Headline win | Watch out for |
|--------|------|--------------|---------------|
| Closed hash maps (SwissTable/F14) | Cache-friendly probing | 2-5x vs `std::unordered_map` | Reference stability |
| Perfect hashing (Frozen, gperf) | Compile-time symbol → ID | 2-3x vs hash maps for static sets | Build time, immutability |
| Vector + linear scan | Order books, small ordered sets | Beats `std::map` and binary search at small N | Insert cost = O(n) |
| Branchless binary search | Mid-size sorted arrays | -42% branch misses | No early exit |
| Cache warming / shadow exec | I-cache + D-cache primed | ~90% latency reduction | I/O side effects |
| Hot/cold splitting + IIFE | Keep I-cache dense | Eliminates jumps to cold code | Manual placement |
| `[[likely]]` / `[[unlikely]]` | Force compiler layout | Hot path inlined sequentially | Verify with disasm |
| `alignas(64)` for sync | Kill false sharing | 5-8x on contended atomics | Wastes cache for non-shared data |
| `alignas(8)` for hot data | Maximize spatial locality | Tighter cache lines | Don't 64-align everything |
| SoA splits | Vectorizable scans | Compiler auto-AVX | Worse for narrow access |
| Hugepages 2M / 1G | Reduce TLB pressure | -23% exec time, dTLB misses 19M→6K | Defrag stalls if THP=always |
| Allocator preallocation | Zero-runtime alloc | Removes malloc tail | Memory upfront |
| SPSC ring (Rigtorp) | Producer/consumer wait-free | 5.5M → 112M items/s | Single producer only |
| Seqlock | Lock-free read-mostly | 0-cost reads when no contention | Race per spec, undefined behavior |
| FastQueue (Gross) | SHM fan-out 1→N | Beats Aeron+Disruptor 2-3 readers | Variable-length only |
| Disruptor | LMAX-style fan-out | 6M events/s single-threaded | More complex |
| Kernel bypass (Onload/ef_vi) | Skip syscall+stack | ~3 us → 700 ns UDP | NIC dependency |
| `isolcpus`+`nohz_full` | Quiet CPU for spin loops | Eliminates jitter source | Wastes a core |
| `__builtin_prefetch` | Cover DRAM latency | -23.5% on irregular walks | Useless on linear scans |

---

## 3. The Order Book Journey (David Gross, Optiver, CppCon 2024)

The single most instructive case study in modern HFT C++. Same problem, five iterations, each measured on a real week of NASDAQ data over Microsoft, Nvidia, Tesla.

### 3.1 Stage 1 — `std::map<Price, Volume>`

```cpp
std::map<Price, Volume, std::greater<>> bids;
auto [it, ok] = bids.try_emplace(price, volume);
if (!ok) it->second += volume;
```

**Verdict:** complexity is great (`O(log n)` add, amortized `O(1)` modify via cached iterator), but it is a node container. Latency distribution is bimodal and the average drifts upward as the heap fragments. **Reject.**

### 3.2 Stage 2 — `std::vector<std::pair<Price, Volume>>` + `std::lower_bound`

```cpp
auto it = std::lower_bound(bids.begin(), bids.end(), price, comp);
if (it != bids.end() && it->first == price) it->second += volume;
else bids.insert(it, {price, volume});
```

**Verdict:** cache locality is excellent, but the latency distribution grows a giant fat tail. Why? Order book updates are exponentially distributed toward the top of book. With best bid at index 0, every update shifts the rest of the vector. Reject as-is.

### 3.3 Stage 3 — Reverse the vector

Best bid at the **end** of the vector. Inserts/erases at the top of book now move zero or one elements. Tail collapses.

```
Update density (log scale):
index 0    *
index 1    **
index 2    ****
index 3    ********
...                       <- exponential decay outward
```

**Principle #2 (Gross):** *understand your problem by looking at data*. The vector's natural ordering was wrong; reversing it required no algorithm change.

### 3.4 Stage 4 — Branchless `lower_bound`

`perf record` showed 30% of CPU time on two `jle`/`jge` instructions inside `std::lower_bound`. Top-down breakdown showed **25% bad speculation**. The CPU branch predictor cannot guess where the market is going.

```cpp
template <class ForwardIt, class T, class Compare>
ForwardIt branchless_lower_bound(ForwardIt first, ForwardIt last,
                                 const T& value, Compare comp) {
    auto length = last - first;
    while (length > 0) {
        auto half = length / 2;
        // multiplication by 1 forces GCC to emit CMOV (clang sometimes resists)
        first += comp(first[half], value) * (length - half);
        length = half;
    }
    return first;
}
```

| Counter | `std::lower_bound` | branchless | delta |
|---------|--------------------|------------|-------|
| Median latency | 32.0 ns | 29.0 ns | -9.4% |
| Branch misses | 48.24 M | 27.83 M | -42% |
| Cycles | 3.92 B | 3.53 B | -10% |
| Instructions | 5.49 B | 5.55 B | +1% |
| IPC | 1.40 | 1.57 | +12% |

**Trade-off:** branchless removes the early-exit, so it always touches `floor(log2 n) + 1` cache lines. On Apple Clang use `-mllvm -x86-cmov-converter=false` to keep CMOV. See Probably Dance "Beautiful Branchless Binary Search" and mhdm.dev `sb_lower_bound`.

### 3.5 Stage 5 — Linear search (winner)

For book sizes typical of single-stock matching (~1000 levels per side, action concentrated at ≤10), the simplest possible algorithm wins.

| Strategy | Median |
|----------|--------|
| `std::map` | bimodal, ~80 ns |
| `std::vector` + `lower_bound` | 32 ns |
| reversed vector + branchless lower_bound | 29 ns |
| **reversed vector + linear search** | **22 ns** |

**Principle #4 (Gross):** *simplicity is the ultimate sophistication*. If your final solution is tiny and fast, you have done your job.

**Principle #5 (Gross):** *mechanical sympathy*. Linear scans are perfect for hardware prefetchers, branch predictors and AVX auto-vectorization (especially after SoA split into a price-only `uint64_t` array). Eytzinger layouts and SIMD k-ary trees only win at much larger N.

```cpp
// Idiomatic Gross-style add: hot path is the [[likely]] branch
template <class T, class Compare>
void AddOrder(T& levels, Price price, Volume vol, Compare comp) {
    auto it = std::ranges::find_if(levels.begin(), levels.end(),
        [comp, price](const auto& p) { return comp(p.first, price); });
    if (it != levels.end() && it->first == price) [[likely]] {
        it->second += vol;
    } else {
        levels.insert(it, {price, vol});
    }
}
```

---

## 4. Hash Map Strategies

### 4.1 SwissTable (`absl::flat_hash_map`, Matt Kulukundis CppCon 2017)

Open addressing with a segregated metadata array.

```
hash = | H1 (57 bits, position) | H2 (7 bits, fingerprint) |

slots:    [k0][k1][k2][k3][k4][k5][k6][k7] ...
metadata: [m0 m1 m2 m3 m4 m5 m6 m7 ...] (1 byte each)

m_i layout:
  empty   = 0b1000_0000  (0x80)
  deleted = 0b1111_1110  (0xfe)
  full    = 0b0HHH_HHHH  where HHHHHHH = H2(key)
```

The lookup is one SSE2 trick:

```cpp
__m128i ctrl = _mm_loadu_si128(metadata + group);
__m128i match = _mm_cmpeq_epi8(ctrl, _mm_set1_epi8(h2));
uint32_t mask = _mm_movemask_epi8(match);
// 'mask' has one bit per matching slot in this group of 16
while (mask) {
    int i = __builtin_ctz(mask);
    if (slots[group + i] == key) return &slots[group + i];
    mask &= mask - 1;
}
```

**Why it wins:**
- 16-way parallel filter on 16 bytes of metadata, no key dereference until a fingerprint matches
- Probe sequences are 1.0-1.5 groups on average even at **load factor 14/16 = 87.5%**
- Segregated metadata stays cache-resident across many keys
- Falls back portably on ARM NEON; on plain SSE2 it works on every x86_64 since 2003

**When it loses:** large cold payloads (then F14 wins) and any environment without SIMD (kernel code).

### 4.2 F14 (`folly::F14FastMap`, Bronson & Shi, Meta 2019)

Filter 14 keys per chunk using double hashing for collision resolution.

| Chunk byte | Contents |
|------------|----------|
| 0..13 | 7-bit tag with top bit set, one per key |
| 14..15 | overflow_count + control |
| 16..N | aligned key/value slots |

Lookup compares the needle's tag against all 14 tags in parallel via `_mm_cmpeq_epi8`.

```
Expected probes at max load factor 12/14:
  hit:  1.04
  miss: 1.275 (P99 = 4)
Fewer than 1% of keys live beyond the first 3 chunks.
```

**F14 vs SwissTable cheat sheet:**

| | SwissTable | F14 |
|---|---|---|
| Group size | 16 | 14 |
| Max load | 14/16 = 87.5% | 12/14 = 85.7% |
| Probing | quadratic | double hashing |
| Tombstones | yes | replaced by `overflow_count` (auto-recycled) |
| ARM NEON | optional fallback | mandatory (built-in) |
| Variants | one | Fast / Node / Value / Vector |
| When it wins | large cold payloads | most workloads, esp. churn-heavy |

**Variant selection:**
- `F14FastMap` — default; auto-picks Value for entries <24 bytes, Vector otherwise
- `F14NodeMap` — only variant with reference stability; needed when iterators must survive insert
- `F14ValueMap` — inline storage, half the memory of `dense_hash_map`
- `F14VectorMap` — contiguous value array, saves ~16 bytes/entry, best iteration speed

### 4.3 Perfect hashing for symbols / instrument IDs

When the keyset is fixed at compile time (instrument list, exchange tags, FIX field IDs), perfect hashing collapses lookup to one comparison.

| Library | Style | Typical lookup vs `std::unordered_set` |
|---------|-------|----------------------------------------|
| GNU `gperf` | Codegen | 2-3x faster |
| `serge-sans-paille/frozen` | C++14 constexpr | 2x faster, integrates with `constexpr`/`constinit` |
| `Kronuz/constexpr-phf` | C++17 constexpr | 605 ms vs 1821 ms (stop-word bench) |
| `static_maps` (arXiv 2602.22506) | C++23 consteval | Lower variance, accepts `string_view` |

```cpp
// Frozen example for instrument code → enum
constexpr auto instruments = frozen::make_unordered_map<frozen::string, Instrument>({
    {"AAPL", Instrument::AAPL},
    {"NVDA", Instrument::NVDA},
    {"TSLA", Instrument::TSLA},
});
auto it = instruments.find("NVDA"); // O(1), no hash collision possible
```

Frozen guarantees a perfect hash function; the build cost is paid once per project.

### 4.4 String key tricks

- **Specialize the hasher.** `std::hash<std::string>` is slow. CityHash, MurmurHash3, FarmHash, or `wyhash` give 30-40% lookup speedups. `emhash` and `ankerl::unordered_dense` ship with `wyhash` by default.
- **Use `std::string_view`** as the key, never `std::string`, to avoid allocations on lookup. Both Abseil and Folly support transparent (heterogeneous) lookups.
- **Length bucketing.** When ticker symbols span 1-8 chars, store small keys inline (`std::array<char, 8>`) and skip strcmp branches entirely. SSO already does this for `std::string` ≤ 16 chars on libstdc++/libc++.
- **Open addressing + backshift deletion.** `rigtorp::HashMap` is tuned for delete-heavy workloads (orders rotating in/out). Backshift avoids tombstones entirely; references the Köppl 2019 paper.

### 4.5 Hash map don'ts

Quoting Gross's principle list and Sean Parent's Photoshop rule:

> Most of the time you do not want node containers. The entire family `std::map` / `std::set` / `std::unordered_map` / `std::unordered_set` / `std::list` should be set aside. Use vectors and closed hash maps backed by arrays.

---

## 5. Cache Lines, False Sharing and the 8-vs-64-Byte Trap

Cache lines are 64 bytes on x86_64 and most ARM, **128 bytes on Apple Silicon M-series**. Use `std::hardware_destructive_interference_size` (C++17) to query at compile time.

### 5.1 The classic false sharing fix

```cpp
struct PaddedCounters {
    alignas(std::hardware_destructive_interference_size) std::atomic<uint64_t> producer{0};
    alignas(std::hardware_destructive_interference_size) std::atomic<uint64_t> consumer{0};
};
```

Speedup vs unaligned: 5-8x on 2-thread contention, sometimes 50x on 32+ cores.

### 5.2 The over-alignment trap (Gross)

> On x86 the best alignment for your data is 8 bytes. You actually do not want to align that on a cache line because it affects the locality of your elements.

`alignas(64)` belongs on **synchronization boundaries** (atomic counters, ring head/tail, sequence numbers). Hot data should pack tightly:

```
Bad — 4 byte int padded to 64 bytes, 16x cache waste
[int][........60B padding........]
[int][........60B padding........]

Good — pack 16 ints per cache line
[int][int][int][int][int][int][int][int][int][int][int][int][int][int][int][int]
```

A 32 KB L1 holds 8000 packed ints versus 500 aligned ones — effective L1 shrinks 16x if you over-align.

### 5.3 SoA vs AoS — pragmatic, not dogmatic

Gross's answer to "do you use SoA at Optiver?":

> We are definitely using it where it's needed — not as something followed on every structure. In some cases it does help. In the specific case of a vector of pairs, you can split it and get some performance gain out of it.

Splitting `std::vector<std::pair<Price, Volume>>` into `std::vector<Price>` + `std::vector<Volume>` enables AVX auto-vectorization on the price scan when prices are `uint64_t`. The catch: AVX startup latency makes it slower on thin books (a few levels). Measure both.

### 5.4 Cache warming

**Cache warming = run the hot path on synthetic data so I-cache + D-cache stay primed.** HFT firms send simulated packets all the way to the NIC and let the NIC drop them; modern NICs have hardware support for this loopback.

| Source | Reported gain |
|--------|---------------|
| Bilokon & Gunduz arXiv 2309.04259 | ~90% latency reduction (267 ms → 25 ms) |
| Cook CppCon 2017 (Optiver) | "Most effective single optimization" |

Implementation: a separate thread fires `IsInteresting(synthetic_msg) → SendOrder()` calls on a fake market data feed. The order send goes only as far as the NIC buffer.

### 5.5 Cache-line alignment helpers

```cpp
constexpr size_t kCacheLine = std::hardware_destructive_interference_size;

template <typename T>
struct alignas(kCacheLine) CacheAligned { T value; char pad[kCacheLine - sizeof(T) % kCacheLine]; };
```

---

## 6. Branchless and Hot/Cold Splitting

### 6.1 `[[likely]]` / `[[unlikely]]` (C++20)

The compiler cannot guess your data. Hint it. Without `[[likely]]` GCC and Clang both move the rare-case `add` instruction to the very end of the function — far from your hot path, evicting it from I-cache.

```cpp
if (it->first == price) [[likely]] { it->second += volume; }
else                    { levels.insert(it, {price, volume}); }
```

After `[[likely]]`, GCC/Clang emit:

```asm
.L2:
    cmp  rbx, r15
    je   .L4
    cmp  DWORD PTR [rbx], ebp
    jne  .L5
    add  DWORD PTR [rbx+4], edx   ; <-- hot add packed adjacent
.L1:
    add  rsp, 24
    pop  rbx
    pop  rbp
```

### 6.2 IIFE (Immediately-Invoked Function Expression) for cold code

The classic "assert that prints diagnostics" pattern bloats the hot function with cold code. Wrap it in an inline lambda annotated `noinline, cold`:

```cpp
#define EXPECT(cond) \
    if (!(cond)) [[unlikely]] { \
        [&]() __attribute__((noinline, cold)) { HandleError(); }(); \
    }

void DeleteOrder(auto& levels, Price price, Volume vol, auto comp) {
    auto it = std::ranges::find_if(levels.begin(), levels.end(), ...);
    EXPECT(it != levels.end() && it->first == price);
    it->second -= vol;
    if (it->second <= 0) levels.erase(it);
}
```

The cold function is emitted in a `.text.cold` section by the linker. The hot function stays in I-cache; only the call instruction remains in the hot path, jumping out only on the cold branch.

### 6.3 Branchless `lower_bound` (production)

See section 3.4. Two key invariants:

1. **No early exit.** Iterations are constant (`floor(log2 n)` or `+1`). The CPU predicts the loop trip count perfectly.
2. **Multiplication by `(length - half)`.** Without the multiplication GCC sometimes emits a conditional jump instead of `cmovne`. Clang occasionally cannot be persuaded; use `-mllvm -x86-cmov-converter=false`.

### 6.4 `constexpr` / `consteval` / `if consteval` (C++17 → C++23)

| Year | Feature | What you get |
|------|---------|--------------|
| C++11 | `constexpr` | Compile-time eval if all args are constant |
| C++17 | `if constexpr` | Discard untaken branches at compile time |
| C++20 | `consteval` | Function MUST run at compile time |
| C++23 | `if consteval` | Detect *during evaluation* whether you are in a constant context |

The arXiv paper measured the simplest constexpr case (factorial(10)): 2.69 ns runtime → 0.245 ns when the compiler folded the call (~90% reduction).

`if consteval` lets one function expose two implementations:

```cpp
constexpr int popcount(uint64_t x) {
    if consteval {
        // portable: works in constant evaluation
        int n = 0; while (x) { n += x & 1; x >>= 1; } return n;
    } else {
        return __builtin_popcountll(x); // POPCNT in run time
    }
}
```

### 6.5 Lambdas vs `std::function`

`std::function` uses type erasure and indirect dispatch. Replacing a lambda with `std::function` in a hot data structure can slow it down by 5-10x because the compiler can no longer inline the call. Templates and `auto` parameters preserve the type. Quote (Gross):

> If you were to use `std::function` here for some reason — sometimes a matter of style, people comment "this lambda could be refactored" — the consequence would actually be huge for the performance of this data structure.

---

## 7. Memory Prefetching

### 7.1 The intrinsic

```cpp
__builtin_prefetch(addr,
                   /*rw*/      0,   // 0 = read, 1 = write
                   /*locality*/3);  // 3=keep in L1, 2=L2, 1=L3, 0=non-temporal
```

x86 also provides `_mm_prefetch(addr, _MM_HINT_T0|T1|T2|NTA)`.

### 7.2 When prefetching helps

| Access pattern | Hardware prefetch handles it? | Manual prefetch ROI |
|----------------|-------------------------------|---------------------|
| Linear scan | Yes (Intel L2 stream prefetcher) | None |
| Strided (constant) | Yes | None |
| Pointer chasing (linked list) | No | High |
| Hash probe | Partially | Medium |
| Tree traversal | No | High |
| Random | No | Medium |

Rule of thumb: prefetch ~100 ns of work ahead, or roughly 8-16 loop iterations on integer-bound code. Daniel Lemire's blog has tuned distances for hash join.

### 7.3 Staging through L2 → L1

For large structures, prefetching directly to L1 is wasteful: there are only 10-20 outstanding L1 prefetches available. Stage in two hops:

```cpp
for (size_t i = 0; i < n; ++i) {
    if (i + 64 < n) __builtin_prefetch(&data[i + 64], 0, 1); // L2
    if (i + 8  < n) __builtin_prefetch(&data[i + 8],  0, 3); // L1
    process(data[i]);
}
```

A published cache study reports a 35% bandwidth gain from tuning L2 distance to 64 and L1 to 8.

### 7.4 Production caveats

- **Useless prefetches cost.** Each one consumes a decode slot.
- **Prefetching the wrong line evicts a useful one.** Profile before deploying.
- **Disable compiler auto-prefetch** when you hand-tune (`-fno-prefetch-loop-arrays` on GCC). Otherwise you compete for outstanding-miss slots.
- **Cache warming** (section 5.4) is the moral equivalent of prefetching the entire hot path; for once-per-event code it dominates per-call prefetches.

---

## 8. Hugepages

Translation Lookaside Buffer entries are scarce — typically 64 + 1024 (L1 + L2 dTLB) on Intel. Touching a 1 GB working set with 4 KB pages needs ~262k TLB entries, so the dTLB miss rate climbs into the double digits. Hugepages multiply each entry's coverage by 512 (2 MB) or 262144 (1 GB).

### 8.1 Numbers (Rigtorp benchmark, mimalloc-backed allocation)

| Pages | dTLB load misses | Miss rate | Wall time |
|-------|------------------|-----------|-----------|
| 4 KB baseline | 19,436,707 | 93.43% | 0.709 s |
| 2 MB THP | 6,320 | 0.07% | **0.544 s (-23%)** |
| 1 GB | 7,869 | 1.23% | 0.598 s |

Once your working set fits in the L2 dTLB with 2 MB pages, 1 GB delivers little extra. Google's TCMalloc Temeraire paper reports +7% RPS fleet-wide from making the allocator hugepage-aware (USENIX 2021).

### 8.2 Choose your mechanism — and disable THP for HFT

| Mechanism | API | When to use |
|-----------|-----|-------------|
| `MAP_HUGETLB | MAP_HUGE_2MB` | `mmap()` direct from a pre-reserved pool | Production HFT (deterministic) |
| Hugetlbfs file mount | `mmap()` of `/dev/hugepages/...` | Multi-process sharing |
| `madvise(MADV_HUGEPAGE)` (THP) | After regular `mmap()` | Best-effort, non-critical |
| THP `always` mode | None — kernel decides | **Avoid** for HFT |

> THP is great until the kernel decides *now* is the time to defragment your zone synchronously. I have experienced tens-of-millisecond stalls from this on JVMs.
> — Gil Tene, paraphrased from the Mechanical Sympathy mailing list

**Hugetlbfs gotcha:** if a process exhausts the pool *during* a copy-on-write fault (e.g. after `fork()`), the kernel sends `SIGBUS`. Avoid `fork()` in hugetlbfs-backed processes, or allocate before forking.

### 8.3 Allocator integration

```cpp
template <typename T>
struct huge_page_allocator {
    using value_type = T;
    static constexpr std::size_t huge_page_size = 1ULL << 21; // 2 MiB

    T* allocate(std::size_t n) {
        std::size_t bytes = round_to(huge_page_size, n * sizeof(T));
        void* p = mmap(nullptr, bytes, PROT_READ | PROT_WRITE,
                       MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB, -1, 0);
        if (p == MAP_FAILED) throw std::bad_alloc{};
        return static_cast<T*>(p);
    }
    void deallocate(T* p, std::size_t n) noexcept {
        munmap(p, round_to(huge_page_size, n * sizeof(T)));
    }
};

// Plug into containers that allocate big and rarely
absl::flat_hash_map<Key, Value, std::hash<Key>, std::equal_to<>,
                    huge_page_allocator<std::pair<const Key, Value>>> map;
```

For general-purpose use, **mimalloc** ships with hugepage support out of the box:

```bash
env MIMALLOC_LARGE_OS_PAGES=1 \
    MIMALLOC_EAGER_COMMIT_DELAY=0 \
    LD_PRELOAD=./libmimalloc.so \
    ./trading_engine
```

### 8.4 Reservation & verification

```bash
# Reserve at boot via cmdline (do this; runtime allocation gets fragmented)
GRUB_CMDLINE_LINUX="... hugepages=1024 hugepagesz=2M default_hugepagesz=2M ..."

# Or runtime, as close to boot as possible
echo 1024 > /proc/sys/vm/nr_hugepages

# Verify CPU support
grep -E 'pse|pdpe1gb' /proc/cpuinfo | uniq

# Pool status
cat /proc/meminfo | grep -i huge

# Per-process residency
grep -E 'AnonHugePages|HugePages' /proc/<pid>/smaps
```

### 8.5 Pre-fault and lock

After `mmap`, **touch every page** before the latency-critical section to fault them in upfront, then `mlockall` to prevent swap:

```cpp
mlockall(MCL_CURRENT | MCL_FUTURE);
std::fill_n(buf, n, 0); // touch every byte → resident
```

---

## 9. Linux Tuning Checklist for HFT

Composite of Rigtorp's [Low Latency Tuning Guide](https://rigtorp.se/low-latency-guide/), SUSE's nohz_full series and Red Hat KCS 3720611. Apply only on dedicated trading hardware.

### 9.1 BIOS

- Energy profile = "maximum performance"
- Disable Hyper-Threading / SMT (doubles effective L1/L2 per thread)
- Enable Turbo (with adequate cooling)
- Disable C-states beyond C1 — every wakeup is ~100 us of jitter

### 9.2 Kernel cmdline (`/etc/default/grub`)

```
isolcpus=2-15
nohz_full=2-15
rcu_nocbs=2-15
nosmt
transparent_hugepage=never
mitigations=off
nmi_watchdog=0
nowatchdog
nosoftlockup
audit=0
intel_pstate=disable
processor.max_cstate=1
intel_idle.max_cstate=0
skew_tick=1
```

`nohz_full` automatically implies `rcu_nocbs` on modern kernels (≥5.x). The tick is only suppressed when **only one runnable thread** is on the core, so a single busy-spin trader thread is the perfect tenant.

### 9.3 Runtime

```bash
# Performance governor everywhere
for f in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
    echo performance > "$f"
done

# SMT off (if not done at boot)
echo off > /sys/devices/system/cpu/smt/control

# Drain workqueues from isolated cores (cpus 0-1 keep them)
for q in /sys/devices/virtual/workqueue/*/cpumask; do
    echo 3 > "$q"  # 0b11 = cores 0,1
done

# Drain IRQs from isolated cores
for irq in /proc/irq/*/smp_affinity_list; do
    [ -w "$irq" ] && echo 0-1 > "$irq" 2>/dev/null
done

# Slow vmstat updates (reduces vmstat_update kthread wakeups)
sysctl vm.stat_interval=120

# Disable swap and KSM
swapoff -a
echo 0 > /sys/kernel/mm/ksm/run

# Disable NUMA balancing
echo 0 > /proc/sys/kernel/numa_balancing
```

### 9.4 Verify

```bash
# Should be ≈0 over 10 s on isolated cores
perf stat -e 'sched:sched_switch' -a -A --timeout 10000

# Should show 1-2 timer interrupts per second per isolated core
perf stat -e 'irq_vectors:local_timer_entry' -a -A --timeout 10000

# TLB shootdowns
grep TLB /proc/interrupts

# Jitter histogram
hiccups | column -t -R 1,2,3,4,5,6
```

### 9.5 Application-side

```cpp
// At startup
mlockall(MCL_CURRENT | MCL_FUTURE);

cpu_set_t cpuset; CPU_ZERO(&cpuset); CPU_SET(3, &cpuset);
pthread_setaffinity_np(pthread_self(), sizeof(cpuset), &cpuset);

// SCHED_OTHER + busy-spin gives best latency on isolated cores
// SCHED_FIFO is OK but contends with kernel housekeeping if any leaks through

while (!stop) {
    if (poll_nic()) handle();
    else _mm_pause(); // 35-100 cycle hint to the scheduler / SMT sibling
}
```

`_mm_pause()` (the `PAUSE` instruction) avoids memory-order machine clears on x86 and reduces power on hyperthreaded siblings — even though SMT is off here, it's still the right primitive.

---

## 10. Memory Allocators

### 10.1 The HFT principle

**No allocations in the hot path.** Every malloc is unbounded latency, possibly a syscall, possibly a TLB shootdown. Pre-allocate everything, reuse via free lists.

### 10.2 Patterns

| Pattern | When | Library |
|---------|------|---------|
| Pre-sized `std::vector::reserve()` | Bounded growth | Stdlib |
| Object pool / free list | Order objects, market data structs | Hand-rolled |
| Arena / bump allocator | Per-event scratch space | `std::pmr::monotonic_buffer_resource` |
| Slab allocator | Fixed-size objects | `boost::pool` |
| Hugepage arena | Large containers | `huge_page_allocator` (sec 8) |
| General-purpose | Non-hot paths | mimalloc, jemalloc, tcmalloc |

### 10.3 `std::pmr::monotonic_buffer_resource`

Bump-pointer arena with no `deallocate()`. Rewinds on destruction.

```cpp
#include <memory_resource>

void process_event() {
    std::array<std::byte, 64 * 1024> stack_buffer;
    std::pmr::monotonic_buffer_resource arena{
        stack_buffer.data(), stack_buffer.size(), std::pmr::null_memory_resource()
    };
    std::pmr::vector<Order> hot_orders{&arena};
    hot_orders.reserve(256);
    // ... no malloc, no free, all on the stack
}
```

`std::pmr::null_memory_resource()` upstream prevents fallback to the heap — if the arena overflows you get `std::bad_alloc` instead of a hidden `mmap` syscall. cppreference benchmark on `std::pmr::list<int>` of 200k nodes: ~3x faster than `std::list` with default allocator.

### 10.4 Free list for object reuse

```cpp
template <typename T, size_t N>
class Pool {
    alignas(T) std::byte storage_[N * sizeof(T)];
    T* free_head_ = nullptr;
public:
    Pool() { for (size_t i = 0; i < N; ++i) deallocate(reinterpret_cast<T*>(storage_) + i); }
    T* allocate() {
        if (!free_head_) return nullptr;
        T* p = free_head_;
        free_head_ = *reinterpret_cast<T**>(p);
        return p;
    }
    void deallocate(T* p) {
        *reinterpret_cast<T**>(p) = free_head_;
        free_head_ = p;
    }
};
```

Allocation is two loads + one store. Deallocation is two stores. Both are branch-free.

### 10.5 General-purpose allocators ranked

| Allocator | Strength | HFT use |
|-----------|----------|---------|
| **mimalloc** (Microsoft) | Hugepage-aware, low fragmentation | Default LD_PRELOAD for non-hot paths |
| **jemalloc** (Facebook) | Best for high concurrency | Web servers, less for HFT |
| **tcmalloc** (Google, Temeraire) | Hugepage-aware since 2021 | +7% fleet RPS at Google |
| **glibc malloc** | Bundled | Avoid in production HFT |

---

## 11. Lock-Free Queues

### 11.1 SPSC ring buffer (Erik Rigtorp)

The classic "head + tail + array" ring with two nuclear optimizations.

```cpp
template <typename T, size_t Capacity>
struct SPSCQueue {
    alignas(64) std::atomic<size_t> head_{0};
    alignas(64) size_t cached_tail_ = 0;
    alignas(64) std::atomic<size_t> tail_{0};
    alignas(64) size_t cached_head_ = 0;
    alignas(64) T data_[Capacity];
};
```

**Optimization 1: align head and tail to separate cache lines.** Producer touches `tail_`, consumer touches `head_`. Without padding, MESI coherency traffic kills you.

**Optimization 2: cached opposite index.** When the producer reads `head_`, it observes how much room is available, caches that locally, and skips re-reads until it has filled the cached window. Same for the consumer with `tail_`.

| Implementation | Throughput |
|----------------|-----------|
| Naive ring (boost::lockfree::spsc) | ~5.5 M items/s |
| `folly::ProducerConsumerQueue` | ~95 M items/s |
| `rigtorp::SPSCQueue` | **~112 M items/s** |

Cache misses dropped from ~300 M to ~15 M for 100 M operations.

Source: https://rigtorp.se/ringbuffer/, https://github.com/rigtorp/SPSCQueue.

### 11.2 FastQueue — single writer, many readers, SHM (Gross 2024)

A bounded **fan-out** SHM queue. One writer, N readers, variable-length messages. Designed for the `MarketData → 50 strategies` topology that Optiver uses.

**Layout:**

```
ProtocolHeader (versioned, magic, queue sizes)
+----------------------------------------------+
|                                              |
|   FastQueue 0  (one cache line of header)    |
|     mReadCounter  (atomic uint64_t, alignas) |
|     mWriteCounter (atomic uint64_t, alignas) |
|     mBuffer       (variable-length messages) |
|                                              |
|   FastQueue 1                                |
|   ...                                        |
+----------------------------------------------+
```

**Two atomic counters, both written by the producer:**

```
mReadCounter == mWriteCounter            -> nothing in flight
mReadCounter <  mWriteCounter            -> ongoing write
                                            (readers see [.. mReadCounter])
After write:    mReadCounter == mWriteCounter == new value
```

**Writer (simplified):**

```cpp
void QProducer::Write(std::span<std::byte> buf) {
    const int32_t payload = sizeof(int32_t) + buf.size();
    mLocalCounter += payload;
    mQ->mWriteCounter.store(mLocalCounter, std::memory_order_release);
    std::memcpy(mNext, &payload, sizeof(int32_t));
    std::memcpy(mNext + sizeof(int32_t), buf.data(), buf.size());
    mQ->mReadCounter.store(mLocalCounter, std::memory_order_release);
    mNext += payload;
}
```

**Reader:**

```cpp
int32_t QConsumer::TryRead(std::span<std::byte> out) {
    if (mLocalRead == mQ->mReadCounter.load(std::memory_order_acquire)) return 0;
    int32_t size; std::memcpy(&size, mNext, sizeof(int32_t)); // see "data race" note
    auto wc = mQ->mWriteCounter.load(std::memory_order_acquire);
    EXPECT(wc - mLocalRead <= QUEUE_SIZE, "queue overflow");
    EXPECT(size <= out.size(),            "buffer too small");
    std::memcpy(out.data(), mNext + sizeof(size), size);
    mLocalRead += sizeof(size) + size;
    mNext      += sizeof(size) + size;
    wc = mQ->mWriteCounter.load(std::memory_order_acquire);
    EXPECT(wc - mLocalRead <= QUEUE_SIZE, "queue overflow");
    return size;
}
```

**The data race:** the `memcpy` of size and payload happens while the producer may be writing the same bytes. By the C++ standard this is undefined. Detected by re-checking `mWriteCounter`; on overflow detection the process dies. ISO C++ proposal **P1478R5 (Byte-wise atomic memcpy)** would fix this formally; until then, "fast and correct in practice, illegal in theory".

**Critical optimizations (each measured):**

1. **Cache the write counter, advance in big chunks.** Reserve N bytes (e.g. 100 KB on an 8 MB queue) per atomic store. Touching the contended cache line every 1000 messages instead of every message is the single biggest win.

   ```cpp
   if (mCachedWriteCounter < mLocalCounter) {
       mCachedWriteCounter = Align<Q_WRITE_BLOCK_BYTES>(mLocalCounter);
       mQ->mWriteCounter.store(mCachedWriteCounter, std::memory_order_release);
   }
   ```

2. **Align messages to 8 bytes, not 64.** Cache-line alignment hurts spatial locality across consecutive messages.

3. **Cache the read counter on the reader.** Skip atomic loads when the previously-read value already shows enough data.

4. **Zero-copy serialization API.** Hand the caller a writable span into the ring instead of taking a `std::span<byte>`. Saves a `memcpy`. Effective speedup: ~2x for typical SBE/CapnProto/FlatBuffers/Protobuf payloads.

   ```cpp
   template <class C>
   void Write(int32_t size, C serialize) {
       auto buf = GetBuffer(size); // inside the ring
       serialize(buf);
       Flush();
   }
   ```

5. **Bulk writing.** Batch multiple messages into one atomic store.

6. **NUMA-aware queue header duplication** when consumers live on different NUMA nodes.

**Final results, AMD EPYC 9474F, 73-byte messages, 8 MB queue:**

| Readers | Aeron (M msg/s) | Disruptor | SeqLockQueue | **FastQueue v3** |
|---------|-----------------|-----------|--------------|------------------|
| 2       | 11.7            | 10.0      | 24.4         | **31.3**         |
| 3       | 9.0             | 9.5       | 19.6         | **23.0**         |
| 5       | 3.8             | 7.0       | 6.3          | 6.0              |
| 10      | 2.7             | 3.5       | 3.6          | 3.9              |
| 15      | 2.4             | 2.6       | 3.2          | 3.4              |

FastQueue is the fastest for low fan-out (1-3 readers) and competitive everywhere. SeqLockQueue wins for the 4-10 reader sweet spot.

### 11.3 Seqlock for read-mostly market data

A seqlock has one rule: the writer increments an even counter to odd before writing, and back to even after. Readers retry if the counter is odd or has changed.

```cpp
template <typename T>
class Seqlock {
    alignas(64) std::atomic<uint64_t> seq_{0};
    T value_{};
public:
    T load() const noexcept {
        T tmp;
        for (;;) {
            uint64_t s1 = seq_.load(std::memory_order_acquire);
            if (s1 & 1) continue; // write in progress
            tmp = value_;          // <-- formally a data race; same caveat as FastQueue
            std::atomic_thread_fence(std::memory_order_acquire);
            if (seq_.load(std::memory_order_acquire) == s1) return tmp;
        }
    }
    void store(const T& t) noexcept {
        uint64_t s = seq_.load(std::memory_order_relaxed);
        seq_.store(s + 1, std::memory_order_release); // odd
        value_ = t;
        std::atomic_thread_fence(std::memory_order_release);
        seq_.store(s + 2, std::memory_order_release); // even
    }
};
```

- Reads cost two atomic loads + a fence — no contention with other readers.
- Used by Aeron, Bitwyre, Citadel for fanning market snapshots from a single feed handler to many strategies.
- Same UB problem as FastQueue's `memcpy`. P1478R5 is the formal fix.

Reference: `rigtorp::Seqlock`, https://github.com/rigtorp/Seqlock.

### 11.4 LMAX Disruptor in C++

Same principle as the seqlock + ring: pre-allocated bounded buffer, sequence numbers, no allocation, no locks. The LMAX exchange achieved 6 million events/s on a single thread with this pattern.

C++ port: `Abc-Arbitrage/Disruptor-cpp` (full Java v3.3.7 feature parity, depends on Boost).

```cpp
auto disruptor = std::make_shared<Disruptor<Order>>(
    eventFactory, /*ringSize=*/ 1024, taskScheduler);
auto buffer = disruptor->ringBuffer();
auto seq = buffer->next();
buffer->get(seq) = Order{...};
buffer->publish(seq);
```

Mandatory invariants: ring size is a power of two (so modulo is `& (size - 1)`), no dynamic allocation after startup, sequence counters are 64-bit and never wrap in practice.

---

## 12. Kernel Bypass

### 12.1 The latency ladder

| Layer | Typical UDP RTT |
|-------|----------------|
| Linux kernel (sockets) | ~3 µs |
| `Onload` LD_PRELOAD on BSD sockets | ~1.5-2 µs |
| `TCPDirect` (Solarflare/AMD) | ~1 µs |
| **`ef_vi` / DPDK (L2 API)** | **~700 ns** |
| FPGA Application Onload Engine | ~750 ns |

### 12.2 OpenOnload — drop-in transparent

`onload` is an LD_PRELOAD library that intercepts BSD socket calls and runs them in userspace against Solarflare/AMD NICs. Zero source code changes.

```bash
onload --profile=latency ./trading_engine
```

Modern Onload (2025-08-11 build) achieves **939.935 ns RTT** through pipes when both processes are co-located. AMD owns Solarflare today; the GitHub repo is `Xilinx-CNS/onload`. Onload also supports AF_XDP NICs from other vendors.

### 12.3 ef_vi — Layer 2 raw API

```cpp
ef_driver_handle dh;
ef_pd pd;
ef_vi vi;
ef_driver_open(&dh);
ef_pd_alloc(&pd, dh, ifindex, EF_PD_DEFAULT);
ef_vi_alloc_from_pd(&vi, dh, &pd, dh, -1, -1, -1, NULL, -1, EF_VI_FLAGS_DEFAULT);
// Then poll vi for descriptors, manage your own ring buffers
```

You manage everything: ring buffers, descriptors, free pools. The reward is the lowest software latency available on x86, plus hardware timestamps.

### 12.4 DPDK alternatives

DPDK gives the same model on commodity NICs. Architecture:

- **EAL** abstracts hugepages, CPU pinning, NUMA
- **PMD** (Poll Mode Drivers) replace interrupt-driven kernel drivers
- **Mempool/Mbuf** are pre-allocated packet buffers in hugepages, NIC DMAs straight in
- **Rings** are lockless MPMC queues for inter-thread packet handoff

Same trick: huge page-backed packet pools, busy-spin polling, zero syscalls.

### 12.5 In-process: shared memory, not sockets

Once on a server, **never use sockets**. Use the FastQueue / Seqlock / Disruptor patterns above on a `shm_open` + `mmap` segment. Quote (Gross):

> The general pattern is to bypass the kernel for low latency connections, and once on a server box locally, to use shared memory to fan out information to all the different processors.

---

## 13. Profiling for Nanoseconds

### 13.1 The methodology pyramid

```
Top-Down (Intel TMAM)              <- start here, 4 buckets, no overlap
   |
   v
perf record -g (sampling)          <- find the hot function
   |
   v
perf annotate / objdump -d         <- find the hot instruction
   |
   v
libpapi HW counters around code    <- validate the fix
   |
   v
ScopedTrace + Clang XRay           <- production timing
   |
   v
HdrHistogram + alerts              <- regression detection
```

### 13.2 Top-Down — your first measurement is never specific

Intel's Top-Down Microarchitectural Analysis Method partitions every pipeline slot into one of four buckets:

```
Pipeline Slots
+------------------+--------------------+
|   Not Stalled    |       Stalled      |
+--------+---------+----------+---------+
|Retiring|Bad Spec |Front End |Back End |
+--------+---------+----------+---------+
                    Fetch L/B  Core/Mem
```

```bash
perf stat -I 10000 -M Frontend_Bound,Backend_Bound,Bad_Speculation,Retiring -p $PID
```

Typical reading on Optiver's order book before optimization (Gross):

```
25.0 % bad_speculation     <- branch predictor losing the binary search
26.4 % retiring            <- only 26% of slots are productive
17.9 % backend_bound       <- DRAM and dependency stalls
30.6 % frontend_bound      <- I-cache misses on cold paths
```

25% bad speculation is enormous — that is what motivated the branchless and then linear search rewrites.

### 13.3 perf record at function and instruction granularity

```bash
perf record -g -p $PID sleep 30
perf annotate
# colored disasm with per-line %; the hot conditional jumps stand out
```

Fork-and-exec pattern when your benchmark has a long warmup you do not want to profile:

```cpp
void RunPerf() {
    if (fork() == 0) {
        auto parentPid = std::to_string(getppid());
        execlp("perf", "perf", "record", "-g", "-p", parentPid.c_str(), nullptr);
    }
}
void InitAndRun() {
    InitBenchmark(); // long, ignored by perf
    RunPerf();       // start sampler
    RunBenchmark();  // hot section measured
}
```

### 13.4 libpapi for surgical counters

`papipp` (David Gross's C++ wrapper, https://github.com/david-grs/papipp) wraps PAPI nicely:

```cpp
#include "papipp.h"
papi::event_set<PAPI_TOT_INS, PAPI_TOT_CYC, PAPI_BR_MSP, PAPI_L1_DCM> ev;
ev.start_counters();
RunBenchmark();
ev.stop_counters();
double ipc = double(ev.get<PAPI_TOT_INS>().counter()) / ev.get<PAPI_TOT_CYC>().counter();
```

Use this to validate a specific micro-optimization, not for first-pass exploration.

### 13.5 Intrusive `ScopedTrace` with `__rdtsc`

```cpp
struct ScopedTrace {
    uint64_t start_;
    const char* name_;
    ScopedTrace(const char* name) : start_(__rdtsc()), name_(name) {}
    ~ScopedTrace() { trace_queue.push({name_, __rdtsc() - start_}); }
};

void Executor::SendOrder() {
    ScopedTrace t{__FUNCTION__};
    ...
}
```

A separate thread drains `trace_queue` to disk or to a metrics endpoint. Cost is two `RDTSC` instructions (~20-25 cycles each) plus one queue push. Cheap enough to leave on for the actual hot functions, expensive enough that you do not put it everywhere.

### 13.6 Clang XRay — runtime-patched, recompile-free

Compile with `-fxray-instrument`. The compiler emits NOP sleds at every function entry/exit. By default they execute as no-ops; at runtime you patch them to call your handler:

```cpp
__xray_set_handler(MyProfile);
__xray_patch_function(function_id);

[[clang::xray_never_instrument]]
void MyProfile(int32_t fid, XRayEntryType type) {
    if (type == ENTRY) Profiling::StartTrace(fid);
    else if (type == EXIT) Profiling::StopTrace();
}
```

You can patch in production without re-deploying. The unpatched cost is one branch over a NOP — sub-nanosecond overhead.

### 13.7 HdrHistogram for the distribution

Always look at percentiles, never at averages. Use HdrHistogram's `Histogram` (3 sig figs, 1-hour range, ~3 ns recording cost). See chapter 13 (Latency Analysis) and the Coordinated Omission guide.

### 13.8 Run on isolated cores *with the system load*

Gross's "you're not alone" principle: a single benchmark on an idle box lies about your production behaviour. The L3 cache is shared across cores, and other tenants on the box (logging daemons, monitoring agents) compete for it. Run benchmarks while the rest of the workload is also running. Single vs 6-worker scaling on Gross's measurements stays at ≈1.0× until the working set hits L3, then collapses.

---

## 14. Anti-Patterns (the consensus "don'ts")

| Don't | Why |
|-------|-----|
| `std::map`, `std::set`, `std::list`, `std::unordered_map`, `std::unordered_set` in hot paths | Node-based, pointer chasing, malloc storms |
| `std::function` in inner loops | Type erasure prevents inlining; 5-10x slower |
| `dynamic_cast` on the hot path | RTTI walk; replace with templates or `std::variant + visit` |
| Throwing exceptions for control flow | Exceptions are zero-cost only on the not-thrown path |
| `alignas(64)` on every struct member | Wastes L1 capacity 16x; use only on sync primitives |
| `transparent_hugepage=always` | Synchronous defrag stalls; use `madvise` or off |
| Unbounded `malloc`/`new` per event | Tail latency from allocator |
| Closed-loop benchmarks (wait-then-send) | Coordinated omission, see ch.13 |
| `std::shared_ptr` as a bus between threads | Atomic refcount on every copy |
| Polling at low frequency | Microsecond gap between sleep and wake; busy-spin instead on isolated cores |
| Single-thread benchmarks on idle hardware | Lies about production cache behaviour (sec 13.8) |
| `std::this_thread::sleep_for(1ns)` | Best case, you yield to the OS for 50+ µs |
| `printf` / `iostream` in hot path | Locks, allocations, syscalls |
| Mixing `float` and `double` in arithmetic | Implicit conversion costs ~50% (arXiv 2309.04259) |
| Building per-message error strings | std::string allocs; defer to cold IIFE |

---

## 15. Quantified Pattern Catalog

From Bilokon & Gunduz, *C++ Design Patterns for Low-latency Applications Including High-Frequency Trading* (arXiv:2309.04259, 2023). Microbenchmarks on gcc, x86_64, single-core.

| Pattern | Cold | Warm | Improvement |
|---------|------|------|-------------|
| Cache warming | 267,685,006 ns | 25,635,035 ns | ~90% |
| `constexpr` (factorial) | 2.69 ns | 0.245 ns | ~91% |
| Loop unrolling | 4,539 ns | 1,260 ns | ~72% |
| Lock-free atomic vs mutex | 175,904 ns | 65,369 ns | ~63% |
| Short-circuit boolean | varies | varies | ~50% |
| `float`-only vs mixed | 21.6 ns | 14.2 ns | ~52% |
| SSE2 SIMD | 21,447 ns | 10,929 ns | ~49% |
| Branch reduction | 7.35 ns | 4.68 ns | ~36% |
| Compile-time dispatch | 2.60 ns | 1.92 ns | ~26% |
| `__builtin_prefetch` | 8,235,924 ns | 6,301,400 ns | ~24% |
| Force-inline | 2.39 ns | 1.90 ns | ~21% |
| Slowpath removal | 28,074 ns | 24,755 ns | ~12% |
| Signed→unsigned | 0.321 ns | 0.282 ns | ~12% |

These are individual microbenchmarks on isolated patterns; stacking them in a real system is non-additive but each is a genuine independent lever.

---

## 16. The Nine Principles (David Gross, CppCon 2024)

1. **No node containers.** Vectors and closed hash maps. Always.
2. **Understand your problem by looking at data.** Profile distributions, not means. The order book lives at the top of book.
3. **Hand-tailored algorithms beat clever generic ones.** Linear search beat Eytzinger and 4 binary-search variants on a 1000-level book.
4. **Simplicity is the ultimate sophistication.** The fastest order book is also the simplest.
5. **Mechanical sympathy.** Algorithms in harmony with hardware: linear access, predictable branches, constant-loop trip counts.
6. **Be mindful of what you use.** The Linux kernel networking stack is beautiful but unnecessary; bypass it.
7. **Right tool for the right task.** Many queues exist; pick the one matching your fan-out, message size, and producer count.
8. **Being fast is good — staying fast is better.** Alerts on regressions, not just dashboards.
9. **You're not alone.** Measure with the rest of the system running. L3 contention is real. Scaling factor stays ~1.0× until the working set hits L3.

---

## 17. Tips for LLM Autoresearch Optimization Loops

The point of writing this chapter is to make these patterns mechanically applicable by an LLM-driven optimization loop. Each item below is a single change with a measurable effect, a way to revert, and known caveats. They are ordered by typical ROI.

| # | Lever | Validation signal | Caveat |
|---|-------|-------------------|--------|
| 1 | Replace `std::unordered_map` → `absl::flat_hash_map` (or `folly::F14FastMap`) | Lookup median ↓; cache misses ↓ | Reference stability differs |
| 2 | Replace `std::map` → reversed `std::vector` + linear scan | Insert-end latency ↓; tail collapses | Insert-middle is O(n) |
| 3 | Add `[[likely]]` / `[[unlikely]]` to the dominant branch | I-cache miss rate ↓; check disasm | Wrong hint hurts |
| 4 | `std::function` → template/lambda inlining | Inlined call site in disasm; instructions/op ↓ | Compile time ↑ |
| 5 | Pre-`reserve()` every container | Heap allocations counter ↓ | Memory upfront |
| 6 | Replace dynamic alloc → `std::pmr::monotonic_buffer_resource` (with `null_memory_resource` upstream) | malloc count → 0 in hot path | Bound the arena |
| 7 | Cache-line align contended atomics with `alignas(std::hardware_destructive_interference_size)` | False sharing counter ↓; 5-8x throughput | 60 bytes wasted per atomic |
| 8 | Reduce `alignas(64)` to `alignas(8)` on hot data fields | L1 occupancy ↑; cache misses ↓ | Only after sync primitives are isolated |
| 9 | `MAP_HUGETLB | MAP_HUGE_2MB` for large allocations + `mlockall` | dTLB load misses ↓ orders of magnitude | Pool reservation, no fork() |
| 10 | `mlockall(MCL_CURRENT|MCL_FUTURE)` at startup | Page faults ↓ to 0 | Memory residency ↑ |
| 11 | Rewrite branch in `lower_bound` → branchless with `*` trick | branch_misses ↓ ~40%; check CMOV in disasm | Loses early exit |
| 12 | Move error handling to `[&]() __attribute__((noinline,cold)) { ... }();` | I-cache misses ↓; check `.text.cold` symbol | Stack frames in cold path |
| 13 | Add Frozen / `gperf` perfect hash for static symbol tables | Lookup ↓ to one comparison | Build time ↑ |
| 14 | Insert `__builtin_prefetch(addr, 0, 1)` 8-16 iterations ahead in pointer-chasing loops | DRAM stall counter ↓ | Linear scans gain nothing |
| 15 | Split `std::vector<std::pair<A,B>>` into `vector<A>` + `vector<B>` (SoA) | AVX vectorization in disasm; throughput ↑ on scans | Hurts narrow access |
| 16 | Switch general allocator to `mimalloc` with `MIMALLOC_LARGE_OS_PAGES=1` | RSS in hugepages; non-hot allocations faster | Side effects on fork() |
| 17 | Replace string keys with `std::string_view` + transparent hash | Allocations on lookup ↓ to 0 | Lifetime hazards |
| 18 | Specialize hash to `wyhash` for string keys | Lookup throughput +30-40% | Different distribution |
| 19 | Cache the contended counter locally and write back in blocks (FastQueue trick) | Atomic store rate ↓ ~1000x | Window of stale visibility |
| 20 | Add `isolcpus`+`nohz_full`+`rcu_nocbs` to GRUB on dedicated cores | `irq_vectors:local_timer_entry` → 1-2 Hz | Single tenant per core |
| 21 | Pin hot threads with `pthread_setaffinity_np` to isolated cores | `sched:sched_switch` → 0 on those cores | Crash-binds to one CPU |
| 22 | Move IRQs and workqueues off isolated cores | `/proc/interrupts` shows 0 on isolated | Verify per IRQ |
| 23 | Replace `std::list` traversal with vector indices | Cache misses ↓ | Iterator semantics differ |
| 24 | Insert `_mm_pause()` in busy-spin polling loops | Power ↓; SMT-sibling throughput ↑ | x86-only, ~30-100 cycles |
| 25 | Compile with `-fno-prefetch-loop-arrays` when hand-prefetching | Outstanding-miss buffer not contended | Apply only after measurement |
| 26 | LTO + PGO + AutoFDO build | Code layout ↑; I-cache miss ↓ | Build pipeline complexity |
| 27 | `consteval` lookup tables | Runtime cost = 0 | Heavy build time |
| 28 | Replace shared `std::shared_ptr` traffic with raw pointer + lifetime owner | Atomic refcount ops ↓ | Lifetime audit needed |
| 29 | Top-down `perf stat -M` to redirect optimization effort | Identify bound (front/back/spec/retire) | Per-architecture metrics |
| 30 | Generational test with libpapi delta around the change | IPC, branch_misses, L1_DCM, cycles | Run on isolated CPU |

**Loop architecture sketch:**

```
1. Build & profile baseline (perf stat -M topdown + libpapi snapshot)
2. Pick lever from table 17 by *bound* (e.g. high bad_speculation -> #11, #3)
3. Apply patch
4. Re-run benchmark with the same seed and dataset
5. Compare:
   - p50 / p99 latency
   - branch_misses, l1_dcm, dtlb_load_misses
   - Top-down %
   - IPC
6. Accept if improvement > noise floor and counters move in expected direction
7. Bisect if improvement contradicts expected direction (often: cache eviction)
8. Commit + tag with the counter delta in the commit message
```

Each lever has an expected counter signal. Reject changes that move counters in the wrong direction even if median latency improves — the gain is noise and will not survive a workload change.

---

## 18. Quick Reference

```bash
# Top-down 4-bucket measurement
perf stat -I 10000 -M Frontend_Bound,Backend_Bound,Bad_Speculation,Retiring -p $PID

# Hot function annotation
perf record -g -p $PID -- sleep 30 && perf annotate

# Verify CPU isolation
perf stat -e 'sched:sched_switch,irq_vectors:local_timer_entry' -a -A --timeout 10000

# Verify no TLB pressure
perf stat -e 'dTLB-loads,dTLB-load-misses,iTLB-loads,iTLB-load-misses' -a --timeout 10000

# Reserve hugepages
echo 1024 > /proc/sys/vm/nr_hugepages
cat /proc/meminfo | grep -i huge

# Low-latency runtime tuning (one-shot)
for f in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do echo performance > "$f"; done
echo off  > /sys/devices/system/cpu/smt/control
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo 0    > /proc/sys/kernel/numa_balancing
swapoff -a
sysctl vm.stat_interval=120
```

```cpp
// Branchless lower_bound (GCC-friendly)
template <class It, class T, class Cmp>
It branchless_lower_bound(It first, It last, const T& v, Cmp c) {
    auto n = last - first;
    while (n > 0) { auto h = n / 2; first += c(first[h], v) * (n - h); n = h; }
    return first;
}

// Hot/cold IIFE
#define EXPECT(x) if (!(x)) [[unlikely]] { [&]() __attribute__((noinline,cold)) { abort(); }(); }

// SHM hugepage allocator pattern
void* p = mmap(nullptr, sz, PROT_READ|PROT_WRITE,
               MAP_PRIVATE|MAP_ANONYMOUS|MAP_HUGETLB|MAP_HUGE_2MB, -1, 0);
mlockall(MCL_CURRENT|MCL_FUTURE);

// False-sharing-free atomic pair
struct alignas(std::hardware_destructive_interference_size) Counter {
    std::atomic<uint64_t> v{0};
};
```

---

## 19. References

### Talks

- David Gross, *When Nanoseconds Matter: Ultrafast Trading Systems in C++*, CppCon 2024 keynote ([cppcon.org](https://cppcon.org/2024-keynote-david-gross/), slides in [CppCon2024 GitHub](https://github.com/CppCon/CppCon2024/blob/main/Presentations/When_Nanoseconds_Matter.pdf))
- Carl Cook, *When a Microsecond Is an Eternity: High Performance Trading Systems in C++*, CppCon 2017 ([slides](https://github.com/CppCon/CppCon2017/blob/master/Presentations/When%20a%20Microsecond%20Is%20an%20Eternity/When%20a%20Microsecond%20Is%20an%20Eternity%20-%20Carl%20Cook%20-%20CppCon%202017.pdf))
- Matt Kulukundis, *Designing a Fast, Efficient, Cache-friendly Hash Table, Step by Step*, CppCon 2017 — the SwissTable origin talk
- Sergey Slotin, *SIMD-friendly K-ary trees*, CppCon 2022 — Eytzinger and beyond
- Gil Tene, *How NOT to Measure Latency*, Strange Loop — coordinated omission

### Papers

- Bilokon, P. & Gunduz, B., *C++ Design Patterns for Low-Latency Applications Including High-Frequency Trading*, [arXiv:2309.04259](https://arxiv.org/abs/2309.04259), 2023
- Hunter et al., *Beyond malloc efficiency to fleet efficiency: a hugepage-aware memory allocator (Temeraire)*, USENIX OSDI 2021 — Google's TCMalloc hugepage paper
- Köppl, D., *Separate Chaining Meets Compact Hashing*, [arXiv:1905.00163](https://arxiv.org/abs/1905.00163), 2019 — backshift deletion behind `rigtorp::HashMap`
- *static_maps: consteval std::map and std::unordered_map Implementations in C++23*, [arXiv:2602.22506](https://arxiv.org/html/2602.22506)

### Engineering blogs

- Erik Rigtorp, [Low Latency Tuning Guide](https://rigtorp.se/low-latency-guide/)
- Erik Rigtorp, [Using Huge Pages on Linux](https://rigtorp.se/hugepages/)
- Erik Rigtorp, [Optimizing a Ring Buffer for Throughput](https://rigtorp.se/ringbuffer/)
- Hudson River Trading, [Low Latency Optimization: Using Huge Pages on Linux](https://www.hudsonrivertrading.com/hrtbeat/low-latency-optimization-part-2/) (and the `hudson-trading/hrtbeat` GitHub repo)
- Nathan Bronson & Xiao Shi, [Open-sourcing F14 for memory-efficient hash tables](https://engineering.fb.com/2019/04/25/developer-tools/f14/), Engineering at Meta
- Abseil team, [Swiss Tables Design](https://abseil.io/about/design/swisstables)
- Daniel Lemire, [Is software prefetching useful for performance?](https://lemire.me/blog/2018/04/30/is-software-prefetching-__builtin_prefetch-useful-for-performance/)
- Probably Dance, [Beautiful Branchless Binary Search](https://probablydance.com/2023/04/27/beautiful-branchless-binary-search/)
- Algorithmica, [Binary Search](https://en.algorithmica.org/hpc/data-structures/binary-search/) — Eytzinger, branch-free, prefetching

### Open-source code

- [`abseil/abseil-cpp`](https://github.com/abseil/abseil-cpp) — `flat_hash_map`, `flat_hash_set`
- [`facebook/folly`](https://github.com/facebook/folly/blob/main/folly/container/F14.md) — F14 family
- [`serge-sans-paille/frozen`](https://github.com/serge-sans-paille/frozen) — constexpr perfect hashing
- [`rigtorp/SPSCQueue`](https://github.com/rigtorp/SPSCQueue), [`rigtorp/MPMCQueue`](https://github.com/rigtorp/MPMCQueue), [`rigtorp/Seqlock`](https://github.com/rigtorp/Seqlock), [`rigtorp/HashMap`](https://github.com/rigtorp/HashMap), [`rigtorp/spartan`](https://github.com/rigtorp/spartan)
- [`Abc-Arbitrage/Disruptor-cpp`](https://github.com/Abc-Arbitrage/Disruptor-cpp) — LMAX Disruptor C++ port
- [`Xilinx-CNS/onload`](https://github.com/Xilinx-CNS/onload) — OpenOnload + ef_vi
- [`microsoft/mimalloc`](https://github.com/microsoft/mimalloc) — hugepage-aware general allocator
- [`david-grs/papipp`](https://github.com/david-grs/papipp) — C++ wrapper around libpapi (Gross's tool)
- [`hudson-trading/hrtbeat`](https://github.com/hudson-trading/hrtbeat) — HRT benchmark sources, including `huge_memory_bench.cpp`
- [`CppCon/CppCon2024`](https://github.com/CppCon/CppCon2024) — Gross slides + many adjacent talks (`So_You_Think_You_Can_Hash`, `Multi_Producer_Multi_Consumer_Lock_Free_Atomic_Queue`, `When_Lock-Free_Still_Isn't_Enough`)

### Cross-references in this handbook

- [Chapter 13 — Latency Analysis & Tail Performance](13-latency-analysis.md) — percentiles, coordinated omission, HdrHistogram
- [Chapter 15 — Memory Subsystem](15-memory-subsystem.md) — NUMA, hugepages from the kernel side
- [Chapter 16 — Scheduler & Interrupts](16-scheduler-interrupts.md) — EEVDF, isolation
- [Chapter 19 — Storage Engine Patterns](19-storage-engine-patterns.md) — adjacent low-latency data structures
- [CRDTs: Lock-Free Distributed State](crdt-lock-free-distributed-state.md) — when to skip coordination entirely
- [Coordinated Omission Guide](coordinated-omission-guide.md) — measure latency correctly before tuning it
