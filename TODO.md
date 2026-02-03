# Linux Performance Handbook - Improvement Plan

**Status:** 70% intermediate / 45% senior-grade
**Target:** Production-ready reference for senior systems engineers

---

## Research Requirements

**Every addition MUST be backed by:**
- Official tool documentation (man pages, GitHub READMEs)
- Kernel documentation (`Documentation/` in kernel tree)
- Academic papers (USENIX, OSDI, SOSP, EuroSys)
- Industry engineering blogs (verified, not Medium speculation)
- Benchmarks with reproducible methodology
- Version-specific changelog verification (kernel, JVM, tools)

**NO content based on:**
- Outdated Stack Overflow answers
- Unverified blog posts
- AI-generated summaries without source verification
- Personal opinions without data

---

## Priority 1: Critical Gaps

### 1.1 New Chapter: Observability & Metrics (Ch. 12)

**Research sources:**
- OpenTelemetry specification: https://opentelemetry.io/docs/specs/
- Prometheus exposition format spec
- Google SRE Book chapters on SLIs/SLOs
- Charity Majors' observability writing (honeycomb.io/blog)
- Cindy Sridharan's "Distributed Systems Observability" (O'Reilly)
- Paper: "So you want to trace your distributed system" (2014, Sigelman et al.)

**Content to write:**
- [ ] OpenTelemetry architecture (traces, metrics, logs, baggage)
- [ ] Instrumentation patterns (auto vs manual, SDK usage)
- [ ] SLI definition methodology (latency, availability, throughput, correctness)
- [ ] SLO design (error budgets, burn rates, alerting windows)
- [ ] Distributed tracing (Jaeger, Zipkin, Tempo) with correlation
- [ ] Metrics cardinality management (label explosion problem)
- [ ] Log aggregation performance (structured logging, sampling)
- [ ] Cost of observability (overhead quantification)
- [ ] eBPF-based observability (Beyla, Pixie, Parca) integration

**Estimated lines:** 500-600

---

### 1.2 New Chapter: Latency Analysis & Tail Performance (Ch. 13)

**Research sources:**
- Paper: "The Tail at Scale" (Dean & Barroso, 2013, Google)
- Paper: "Attack of the Killer Microseconds" (Barroso et al., 2017)
- Paper: "Amdahl's Law in the Multicore Era" (Hill & Marty, 2008)
- Gil Tene's talks on coordinated omission
- HdrHistogram documentation (hdrhistogram.org)
- Brendan Gregg's latency heat maps methodology

**Content to write:**
- [ ] Percentile fundamentals (p50, p95, p99, p99.9 interpretation)
- [ ] Tail latency amplification in distributed systems
- [ ] Coordinated omission problem and mitigation
- [ ] Latency heat maps generation and interpretation
- [ ] HdrHistogram usage for accurate percentile capture
- [ ] SLO vs SLA distinction with latency targets
- [ ] Queueing theory basics (Little's Law, M/M/1 implications)
- [ ] Latency breakdown methodology (network, compute, I/O, GC, locks)
- [ ] Hedged requests and backup request patterns
- [ ] Latency budgets across service boundaries

**Estimated lines:** 400-500

---

### 1.3 New Section: Troubleshooting Decision Framework

**Research sources:**
- Brendan Gregg's USE Method paper and blog posts
- Netflix's "Linux Performance Analysis in 60 Seconds"
- Google SRE Book troubleshooting chapter
- "Systems Performance" 2nd ed (Gregg, 2020) methodology chapters

**Content to write:**
- [ ] USE Method implementation (Utilization, Saturation, Errors)
- [ ] RED Method for services (Rate, Errors, Duration)
- [ ] Decision tree: "App is slow" → branching diagnosis
- [ ] Decision tree: "High CPU" → user vs system vs iowait
- [ ] Decision tree: "Memory pressure" → leak vs fragmentation vs OOM
- [ ] Decision tree: "Network latency" → DNS, TCP, app, serialization
- [ ] Tool selection matrix (symptom × tool × overhead)
- [ ] Anti-patterns: common misdiagnoses and their corrections
- [ ] "60-second checklist" with modern tool equivalents

**Location:** New file `00-troubleshooting-framework.md` or prefix to README
**Estimated lines:** 300-400

---

### 1.4 Rewrite: GPU & HPC Chapter (Ch. 11)

**Research sources:**
- NVIDIA Nsight Systems documentation
- NVIDIA DCGM API reference
- AMD ROCm documentation (rocm.docs.amd.com)
- Paper: "Dissecting the NVIDIA Volta GPU Architecture" (Jia et al., 2018)
- NCCL documentation for collective profiling
- MLPerf benchmark methodology
- CUDA occupancy calculator documentation

**Content to write:**
- [ ] Nsight Systems workflow (timeline analysis, kernel profiling)
- [ ] Nsight Compute for kernel-level optimization
- [ ] Multi-GPU profiling (NVLink, PCIe bottlenecks)
- [ ] NCCL collective communication profiling
- [ ] AMD ROCm profiler (rocprof) with examples
- [ ] AMD MI300X specific tuning (HBM3, Infinity Fabric)
- [ ] CPU-GPU synchronization bottleneck detection
- [ ] Memory transfer optimization (pinned memory, async copies)
- [ ] Mixed precision profiling (FP16, BF16, TF32)
- [ ] Power limiting and thermal throttle detection
- [ ] Container GPU profiling (NVIDIA MPS, MIG in containers)
- [ ] Triton Inference Server profiling integration

**Estimated lines:** 400-500 (up from current ~250)

---

## Priority 2: Outdated Content Updates

### 2.1 Java/JVM Modernization (Ch. 10)

**Research sources:**
- JDK 21 release notes (virtual threads, generational ZGC)
- JDK 22-25 release notes for recent features
- async-profiler 3.x documentation
- JEP specifications (ZGC, Loom, Panama)
- Netflix tech blog on JVM tuning (2023-2026 posts)
- Paper: "Scalable and Low-Latency Garbage Collection" (ZGC paper)

**Updates needed:**
- [ ] Virtual threads (Project Loom) performance characteristics
- [ ] Generational ZGC as primary recommendation (not G1GC)
- [ ] async-profiler 3.x new features (allocation profiling improvements)
- [ ] JFR streaming API for continuous profiling
- [ ] Foreign Function & Memory API (Panama) overhead
- [ ] Vector API (incubator → stable) profiling
- [ ] Update all examples from Java 11 → Java 21+ syntax
- [ ] Remove compressed oops caveats (ZGC handles differently)
- [ ] CRaC (Coordinated Restore at Checkpoint) for startup
- [ ] GraalVM Native Image profiling differences

---

### 2.2 Kernel Version Updates

**Research sources:**
- Kernel changelogs (6.8 through 6.12+)
- LWN.net kernel coverage
- kernel.org documentation updates
- Kernel mailing list archives for performance features

**Updates needed:**
- [ ] sched_ext merged status (6.12+) - update Ch. 06 examples
- [ ] netkit stabilization notes
- [ ] io_uring updates (multishot, provided buffers improvements)
- [ ] Memory tiering with CXL support
- [ ] BPF arena and kfuncs additions
- [ ] EEVDF scheduler (replaced CFS in 6.6)
- [ ] Per-VMA locks (6.4+) impact on mmap contention
- [ ] Lazy preemption model changes

---

### 2.3 Python Profiling Updates (Ch. 05)

**Research sources:**
- Python 3.12+ release notes (perf profiler support)
- Python 3.13 JIT compiler (experimental)
- py-spy, scalene, memray documentation
- PEP 669 (low-impact monitoring)

**Updates needed:**
- [ ] Python 3.12 sys.monitoring API
- [ ] Built-in perf support (-X perf)
- [ ] Scalene profiler (CPU + memory + GPU)
- [ ] Memray for memory profiling
- [ ] Python 3.13 JIT implications
- [ ] GIL-free Python (PEP 703) performance testing

---

## Priority 3: Missing Language Coverage

### 3.1 Go Profiling (extend Ch. 05)

**Research sources:**
- Go pprof documentation (pkg.go.dev/runtime/pprof)
- Go execution tracer documentation
- Felixge's "The Busy Developer's Guide to Go Profiling"
- Paper: "Understanding Real-World Concurrency Bugs in Go" (Tu et al., 2019)

**Content to write:**
- [ ] pprof CPU profiling with flame graph generation
- [ ] pprof heap profiling (inuse_space vs alloc_space)
- [ ] Goroutine profiling and leak detection
- [ ] Mutex contention profiling
- [ ] Block profiling for channel operations
- [ ] Execution tracer for latency analysis
- [ ] Continuous profiling with Parca/Pyroscope
- [ ] GODEBUG options for runtime insights
- [ ] Escape analysis and allocation reduction

**Estimated lines:** 250-300

---

### 3.2 Rust Profiling (extend Ch. 05)

**Research sources:**
- cargo-flamegraph documentation
- perf integration with Rust
- Rust Performance Book (nnethercote.github.io)
- DHAT documentation (Valgrind)
- Paper: "Fearless Concurrency? Understanding Concurrent Programming Safety in Real-World Rust" (2020)

**Content to write:**
- [ ] cargo-flamegraph setup and usage
- [ ] perf with Rust (debug symbols, frame pointers)
- [ ] DHAT for heap profiling
- [ ] criterion.rs for microbenchmarks
- [ ] async profiling (tokio-console)
- [ ] Memory layout optimization (cache alignment)
- [ ] Compile-time optimization flags (LTO, codegen-units)
- [ ] Profile-guided optimization workflow

**Estimated lines:** 200-250

---

### 3.3 C/C++ Profiling Deep-Dive (extend Ch. 05)

**Research sources:**
- Valgrind documentation (cachegrind, callgrind, massif)
- Intel VTune documentation
- AMD uProf documentation
- GCC/Clang optimization documentation
- Paper: "What Every Programmer Should Know About Memory" (Drepper, 2007)

**Content to write:**
- [ ] Valgrind cachegrind for cache analysis
- [ ] Valgrind callgrind + KCachegrind visualization
- [ ] Intel VTune hotspot analysis
- [ ] AMD uProf for AMD-specific insights
- [ ] AddressSanitizer/ThreadSanitizer performance overhead
- [ ] LTO and PGO automation with build systems
- [ ] AutoFDO workflow (Google's approach)
- [ ] Link-time dead code elimination

**Estimated lines:** 250-300

---

## Priority 4: Advanced Topics

### 4.1 Memory Efficiency Deep-Dive (extend Ch. 08)

**Research sources:**
- jemalloc documentation and tuning guide
- tcmalloc documentation (Google)
- mimalloc benchmarks (Microsoft)
- Paper: "Mesh: Compacting Memory Management for C/C++ Applications" (2019)
- Paper: "Scalable Memory Allocation Using jemalloc" (Evans)
- NUMA documentation in kernel tree

**Content to write:**
- [ ] Allocator comparison (glibc, jemalloc, tcmalloc, mimalloc)
- [ ] Allocator profiling (MALLOC_CONF, jemalloc stats)
- [ ] Fragmentation detection and mitigation
- [ ] NUMA-aware allocation patterns
- [ ] Huge page allocators (libhugetlbfs, THP interaction)
- [ ] Memory compaction triggers and monitoring
- [ ] Custom allocator integration points

**Estimated lines:** 300-350

---

### 4.2 NUMA Deep-Dive (extend Ch. 08)

**Research sources:**
- numactl documentation
- Kernel NUMA documentation
- Paper: "An Analysis of Linux Scalability to Many Cores" (Boyd-Wickizer et al., 2010)
- Intel/AMD NUMA topology documentation
- AutoNUMA kernel documentation

**Content to write:**
- [ ] NUMA topology discovery (numactl, lstopo, lscpu)
- [ ] Memory policy options (interleave, bind, preferred)
- [ ] CPU affinity with NUMA awareness
- [ ] AutoNUMA mechanics and tuning
- [ ] NUMA balancing metrics (numa_hint_faults)
- [ ] Cross-NUMA memory access penalties (quantified)
- [ ] NUMA-aware application design patterns
- [ ] CXL memory tiering interaction

**Estimated lines:** 250-300

---

### 4.3 HTTP/2 & HTTP/3 Performance (extend Ch. 03)

**Research sources:**
- RFC 9000 (QUIC), RFC 9114 (HTTP/3)
- Cloudflare HTTP/3 blog posts
- Paper: "Taking a Long Look at QUIC" (2017, IMC)
- curl HTTP/3 documentation
- h2load documentation

**Content to write:**
- [ ] HTTP/2 multiplexing and head-of-line blocking
- [ ] HTTP/2 flow control tuning
- [ ] HTTP/3 QUIC congestion control differences
- [ ] 0-RTT performance implications
- [ ] Connection coalescing impact
- [ ] h2load for HTTP/2 benchmarking
- [ ] curl --http3 testing
- [ ] TLS 1.3 handshake optimization
- [ ] ALPN negotiation overhead measurement

**Estimated lines:** 200-250

---

### 4.4 Database Profiling (new section or Ch. 14)

**Research sources:**
- PostgreSQL documentation (pg_stat_statements, EXPLAIN)
- MySQL Performance Schema documentation
- RocksDB tuning guide
- Paper: "The Design and Implementation of Modern Column-Oriented Database Systems" (2012)
- Use The Index, Luke (use-the-index-luke.com)

**Content to write:**
- [ ] PostgreSQL query profiling (EXPLAIN ANALYZE, pg_stat_statements)
- [ ] PostgreSQL connection pooling (PgBouncer profiling)
- [ ] MySQL slow query analysis
- [ ] MySQL Performance Schema usage
- [ ] Index efficiency analysis
- [ ] LSM tree compaction profiling (RocksDB, LevelDB)
- [ ] Write amplification measurement
- [ ] Buffer pool hit rate optimization
- [ ] Lock contention detection

**Estimated lines:** 350-400

---

### 4.5 Power & Energy Profiling (extend Ch. 05)

**Research sources:**
- Intel RAPL documentation
- turbostat source code documentation
- Paper: "Power Management in Cloud Computing" (survey)
- kernel powercap documentation
- Green500 methodology

**Content to write:**
- [ ] RAPL energy measurement (package, core, DRAM)
- [ ] turbostat for power state analysis
- [ ] powertop analysis workflow
- [ ] CPU frequency scaling governors impact
- [ ] Energy-delay product optimization
- [ ] Thermal throttling detection
- [ ] Container power accounting
- [ ] Carbon-aware workload scheduling

**Estimated lines:** 200-250

---

## Priority 5: Structural Improvements

### 5.1 Consolidate Redundancy

- [ ] Merge flame graph content from Ch.05 and Ch.06 into single authoritative section
- [ ] Consolidate sysctl references (Ch.02, Ch.08, Ch.09) into cheatsheet with cross-references
- [ ] Unify NIC configuration (Ch.03 analysis + Ch.09 tuning) with clear separation

### 5.2 Navigation Improvements

- [ ] Add curated sources to README navigation table
- [ ] Create tool index by problem type (symptom → tools mapping)
- [ ] Add "See also" cross-references between related chapters
- [ ] Create quick-reference cards for each chapter

### 5.3 Cheatsheet Integration

- [ ] Link `cheatsheets/one-liners.md` from relevant chapters
- [ ] Add chapter-specific one-liner sections
- [ ] Create printable PDF-ready cheatsheet format

### 5.4 Quantified Examples

- [ ] Add before/after metrics to tuning examples
- [ ] Include "expected improvement range" for optimizations
- [ ] Add "when this matters" context (workload types)
- [ ] Include failure mode warnings for aggressive tuning

---

## Priority 6: Production Wisdom

### 6.1 Anti-Patterns Documentation

**Per chapter, document:**
- [ ] Common misconfigurations and their symptoms
- [ ] Tuning parameters that backfire
- [ ] "Looks like improvement but isn't" scenarios
- [ ] Production constraints not obvious from docs

### 6.2 Safety Notes

- [ ] Mark tools unsafe for production (overhead warnings)
- [ ] Document rollback procedures for kernel tuning
- [ ] Add "test in staging first" warnings where appropriate
- [ ] Include incident response integration points

### 6.3 Cost-Benefit Analysis

- [ ] Tuning effort vs expected gain guidelines
- [ ] "Diminishing returns" thresholds
- [ ] When to stop optimizing (good enough criteria)
- [ ] Hardware upgrade vs software tuning decision framework

---

## Research Backlog

Papers to read and potentially extract insights:

### Systems Performance
- [ ] "Attack of the Killer Microseconds" (Barroso, 2017)
- [ ] "The Tail at Scale" (Dean & Barroso, 2013)
- [ ] "Amdahl's Law in the Multicore Era" (Hill & Marty, 2008)
- [ ] "What Every Programmer Should Know About Memory" (Drepper, 2007)

### eBPF & Tracing
- [ ] "BPF and XDP Reference Guide" (Cilium)
- [ ] "The BSD Packet Filter: A New Architecture for User-level Packet Capture" (original BPF)
- [ ] sched_ext design documents

### Databases
- [ ] "Architecture of a Database System" (Hellerstein, 2007)
- [ ] RocksDB wiki performance tuning

### Distributed Systems
- [ ] "Dapper, a Large-Scale Distributed Systems Tracing Infrastructure" (Google, 2010)
- [ ] "So you want to trace your distributed system" (Sigelman, 2014)

### Memory
- [ ] "Mesh: Compacting Memory Management" (2019)
- [ ] "Scalable Memory Allocation Using jemalloc" (Evans)
- [ ] "TCMalloc: Thread-Caching Malloc" (Google)

### Scheduling
- [ ] "The Linux Scheduler: A Decade of Wasted Cores" (Lozi et al., 2016)
- [ ] "ghOSt: Fast & Flexible User-Space Delegation of Linux Scheduling" (2021)
- [ ] EEVDF scheduler documentation

### GPU
- [ ] "Dissecting the NVIDIA Volta GPU Architecture" (Jia, 2018)
- [ ] NCCL design documentation
- [ ] CUDA Best Practices Guide

---

## Verification Checklist

Before merging any new content:

- [ ] All commands tested on Linux 6.6+ kernel
- [ ] Version requirements explicitly stated
- [ ] Sources cited with links
- [ ] No unverified claims
- [ ] Overhead/safety warnings included
- [ ] Cross-referenced with existing content
- [ ] Example output included where helpful
- [ ] Reviewed against anti-patterns list

---

## Completion Tracking

| Section | Status | Lines | Reviewer |
|---------|--------|-------|----------|
| Ch. 12 Observability | TODO | 0/500 | - |
| Ch. 13 Latency | TODO | 0/450 | - |
| Troubleshooting Framework | TODO | 0/350 | - |
| Ch. 11 GPU rewrite | TODO | 250/500 | - |
| Java modernization | TODO | - | - |
| Kernel updates | TODO | - | - |
| Go profiling | TODO | 0/275 | - |
| Rust profiling | TODO | 0/225 | - |
| C/C++ profiling | TODO | 0/275 | - |
| Memory deep-dive | TODO | 0/325 | - |
| NUMA deep-dive | TODO | 0/275 | - |
| HTTP/2-3 | TODO | 0/225 | - |
| Database profiling | TODO | 0/375 | - |
| Power profiling | TODO | 0/225 | - |
| Structural fixes | TODO | - | - |
| Anti-patterns | TODO | - | - |

**Total new content estimate:** ~4000 lines
**Total updates estimate:** ~1500 lines modified

---

## Notes

- Prioritize depth over breadth for senior audience
- Every tool example must be runnable
- Prefer battle-tested over bleeding-edge (unless explicitly marked experimental)
- Update this TODO as work progresses
