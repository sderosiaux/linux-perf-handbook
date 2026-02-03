# Dan Luu's Systems & Performance Insights

Extracted from danluu.com - rigorous, data-driven analysis of hardware/software performance.

---

## 1. Latency: The Hidden Tax

### End-to-End Input Latency (1977-2017)

| System | Year | Latency |
|--------|------|---------|
| Apple 2e | 1983 | 30ms |
| iPad Pro + Pencil | 2017 | 30ms |
| TI 99/4a | 1981 | 40ms |
| Kindle Paperwhite 3 | 2015 | 630ms |
| Kindle 4 | 2011 | 860ms |

**Key insight:** Despite 4000x faster CPUs and 500000x more transistors, modern systems often match or exceed 1980s latency. A 2017 PowerSpec G405 had keyboard-to-screen latency exceeding round-the-world packet time.

### Why Old Systems Feel Faster

1. **Higher input scanning**: Apple 2e scanned at 556Hz vs modern 100-200Hz
2. **Direct output**: CRTs added ~8.3ms; modern displays add 13.5ms minimum
3. **No pipeline complexity**: No process handoffs, IPC, triple-buffering, context switches

### Component Latency Breakdown

| Component | Fast | Slow |
|-----------|------|------|
| Keyboard | 15ms (Apple Magic USB) | 60ms (Logitech K360) |
| Terminal | ~5ms (emacs-eshell) | 44ms+ (hyper) |
| Display | 13.5ms (gaming) | 30ms+ (standard LCD) |

**Actionable:** Gaming keyboards aren't faster. The 45ms keyboard variance is perceivable (humans detect down to 2ms).

---

## 2. CPU Architecture: What Actually Matters

### Features Since the 80s

| Feature | Impact |
|---------|--------|
| Multi-level cache | 286: few cycles/access → Pentium 4: 400+ cycles to main memory |
| Prefetching | Recovers to 22 GB/s on 3GHz despite latency |
| Branch prediction | 14-cycle misprediction penalty (Haswell); 0.5-4% typical miss rate |
| SIMD (SSE/AVX) | 2-4x speedup for parallel operations |
| SMT/Hyperthreading | ~25% throughput gain, per-thread perf cost |

### Branch Prediction Evolution

| Strategy | Accuracy | CPI |
|----------|----------|-----|
| Always Taken | 70% | 2.14 |
| Two-bit saturating | 90% | 1.38 |
| Hybrid predictor | 96% | 1.15 |

**Without prediction:** 4.8 CPI. **With perfect prediction:** 1.0 CPI. Branch prediction is non-negotiable for modern performance.

---

## 3. Cache Behavior: Non-Obvious Effects

### LRU vs Random Eviction

- **LRU wins** when working set fits in cache
- **Random degrades gracefully** when working set exceeds cache
- **Two-choices algorithm** (LRU of 2 random lines) beats both for large caches (512KB+)

Mathematical basis: Load balancing theory. Choosing least-loaded of k random bins reduces max load from O(log n) to O(log log n / log k).

### Data Alignment Paradox

**Page-aligned data performs WORSE than misaligned.**

| Sandy Bridge L1 | Page-Aligned | Misaligned |
|-----------------|--------------|------------|
| Usable cache sets | 8 | 512 |

Power-of-2 alignment causes all accesses to hit same cache set. Prefetchers also won't cross page boundaries. **Deliberate slight misalignment reclaims cache capacity.**

### Cache Partitioning (Intel CAT)

- Enables 90% datacenter utilization (vs typical 10-50%)
- 64% average latency improvement in measured scenarios
- Most server workloads need only 4-6MB LLC despite 12-30MB available
- ROI: "$600M in free compute" for already-optimized deployments

---

## 4. Measurement Methodology

### The Three Deadly Sins

1. **Coordinated omission**: Closed-loop testing (wait for response before next request) underestimates latency during slowdowns. Use open-loop measurement.

2. **Aggregating percentiles wrong**: `avg(p99 per shard)` ≠ cluster p99. One example: server p99 ~16ms, client p99 ~240ms (15x difference).

3. **Insufficient resolution**: Minute-level metrics miss sub-minute spikes. Bursty events invisible without sub-second granularity.

### Sampling vs Tracing

| Approach | Use When | Blind Spots |
|----------|----------|-------------|
| Sampling (perf) | Hot code paths, sustained CPU | Tail latency, waiting time, rare spikes |
| Tracing | Tail latency, distributed queries, lock contention | ~1% CPU overhead |

**Fundamental limit:** 1kHz sampling cannot detect phenomena at higher frequencies (Nyquist). For tail latency investigation, tracing is required.

### Why p95 Isn't Enough

At p95, systems (and people) "constantly make basic game-losing mistakes." The gap to p99+ involves:
- Deliberate practice with feedback
- Self-observation (screen recording revealed 25% time debugging in isolation)
- Systematic improvement vs random grinding

For SLOs: p95 masks critical failure modes that compound in distributed systems.

---

## 5. Production Systems Gotchas

### Container Throttling Death Spiral

**Problem:** Linux CFS bandwidth quotas allow brief over-subscription then force sleep.

**Mechanism:**
1. Thread pools sized at 2x reserved cores (common heuristic)
2. Load spike exhausts quota quickly
3. Throttled shards queue more work
4. Wake with larger backlog → faster re-throttle

**Impact at Twitter:** Services degraded at 50% CPU despite theoretical capacity.

**Solutions:**
- Kernel patch preventing over-quota usage at any moment (50% cost reduction fleet-wide)
- Separate I/O from application threads (Finagle Offload Filter)
- Fewer, larger shards (0-20% CPU, 10-40% memory improvement)

### ECC Memory: The Silent Corruption Problem

- FIT rate: 0.057-0.071 faults/megabit → ~0.5 faults/year per 128GB server
- Firefox crash data suggests 10-20% of crashes from memory corruption
- Google's pre-ECC era: search index returned "seemingly random results"

**Rule:** Servers need ECC. Silent corruption replicates to backups.

---

## 6. Architecture Philosophy

### Simple Systems at Scale

| Company | Architecture | Scale | Outcome |
|---------|--------------|-------|---------|
| Wave | Python monolith + Postgres | $1.7B, 70 engineers | CRUD app serving fintech |
| Stack Overflow | Monolith | Top-100 internet traffic | $1.8B acquisition |

**Key principle:** "Computers are fast enough that high-traffic apps can be served with simple architectures."

### When Complexity is Justified

- Data residency laws requiring multi-region
- Vendor bugs with no fix path
- Telecom integrations lacking SaaS coverage
- Multi-region failover requirements

**Cost reality:** "Compensation cost of engineers dominates system cost." Engineering time is the constraint, not compute.

---

## 7. Mental Models

### Latency Numbers to Internalize

```
Keyboard scan → kernel:     15-60ms (varies 4x by keyboard)
Terminal processing:        5-44ms (varies 9x by terminal)
Display output:             13.5-30ms+
Total keypress-to-pixel:    30-860ms (varies 29x by system)

Branch misprediction:       14-20 cycles
L1 cache hit:               ~4 cycles
Main memory access:         400+ cycles (100x L1)
```

### Performance Intuition Checklist

- [ ] Am I measuring at the right percentile? (p95 hides tail problems)
- [ ] Am I using open-loop or closed-loop testing? (coordinated omission)
- [ ] Is my data aligned to power-of-2? (may hurt cache performance)
- [ ] Are my containers sized for peak or average? (throttling spiral)
- [ ] Do I need tracing or is sampling sufficient? (tail latency needs tracing)

### The Measurement Imperative

"Measurement is undervalued compared to building, yet produces outsized impact."

- Kyle Kingsbury's Jepsen found Redis losing 56% of writes during partition
- MongoDB "dropping a phenomenal amount of data"
- Consumer Reports testing changed Tesla braking, automotive ESC, tire siping

**Pattern:** Engineers want to improve but lack business justification until external measurements provide ammunition.

---

## Summary: Actionable Takeaways

1. **Latency compounds through the stack.** Measure end-to-end, not component-by-component.

2. **p95 is table stakes.** Monitor p99, p99.9 for real system health.

3. **Simple architectures scale further than expected.** Add complexity only when forced by regulatory, vendor, or operational requirements.

4. **Cache behavior is non-obvious.** Power-of-2 alignment hurts. Two-choices beats pure LRU. Most workloads need <6MB LLC.

5. **Container throttling is a death spiral.** Size for peak, not average. Separate I/O threads. Consider larger shards.

6. **Branch prediction is everything.** 96% accuracy = 1.15 CPI. 70% accuracy = 2.14 CPI. Structure code for predictable branches.

7. **Measurement creates change.** Publish benchmarks. External pressure fixes what internal advocacy cannot.

---

## Sources

- [Computer latency 1977-2017](https://danluu.com/input-lag/)
- [Branch prediction](https://danluu.com/branch-prediction/)
- [Cache replacement policies](https://danluu.com/2choices-hierarchical/)
- [3c-conflict](https://danluu.com/3c-conflict/)
- [Why ECC memory](https://danluu.com/why-ecc/)
- [CGroup throttling](https://danluu.com/cgroup-throttling/)
- [Simple architectures](https://danluu.com/simple-architectures/)
- [Sampling vs tracing](https://danluu.com/perf-tracing/)
- [P99 percentile problems](https://danluu.com/p99-hierarchical/)
