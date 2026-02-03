# Performance Profiling

CPU profiling, flame graphs, and language-specific profilers.

## perf (Linux profiler)

### Basic Profiling
```bash
perf stat command              # Basic stats
perf stat -d command           # Detailed stats
perf stat -p PID sleep 10      # Profile running process
perf stat -a sleep 10          # System-wide

# CPU cycles, cache misses, branch misses
perf stat -e cycles,cache-misses,branch-misses command
```

### Recording Profiles
```bash
perf record command            # Record profile
perf record -g command         # With call graphs
perf record -p PID sleep 30    # Profile running process
perf record -a sleep 30        # System-wide
perf record -F 99 command      # 99 Hz sampling
perf record -e cpu-cycles command  # Specific event
perf record -e 'syscalls:*' command  # Trace syscalls
```

### Analyzing Profiles
```bash
perf report                    # Interactive report
perf report --stdio            # Text report
perf report -n                 # Show sample counts
perf report --sort comm,dso    # Sort by command, DSO
perf report -g graph           # Call graph as graph
perf report --no-children      # Self time only
perf annotate                  # Source annotation
```

### Live Profiling
```bash
perf top                       # Live function profiling
perf top -p PID                # Specific process
perf top -g                    # With call graphs
perf top -e cache-misses       # Specific event
```

### Kernel Tracing
```bash
perf trace                     # Like strace, but faster
perf trace -p PID              # Trace process
perf trace -e open,read,write  # Specific syscalls
perf trace -s                  # Summary mode
```

## Flame Graphs

### Generate with perf
```bash
# Record profile with call stacks
perf record -F 99 -g command
# or for running process
perf record -F 99 -g -p PID sleep 30

# Generate flame graph
perf script | stackcollapse-perf.pl | flamegraph.pl > out.svg
```

### Brendan Gregg's FlameGraph tools
```bash
git clone https://github.com/brendangregg/FlameGraph

# CPU flame graph
perf record -F 99 -g -a sleep 30
perf script | ./FlameGraph/stackcollapse-perf.pl | ./FlameGraph/flamegraph.pl > cpu.svg

# Off-CPU flame graph (requires eBPF tools)
offcputime -df -p PID 30 | ./FlameGraph/flamegraph.pl --color=io --title="Off-CPU" > offcpu.svg
```

### Options
```bash
flamegraph.pl --title "Title" < collapsed > out.svg
flamegraph.pl --width 1200 < collapsed > out.svg
flamegraph.pl --colors java < collapsed > out.svg  # Color palette
flamegraph.pl --reverse < collapsed > icicle.svg   # Icicle graph
flamegraph.pl --inverted < collapsed > out.svg     # Inverted
```

## strace / ltrace

### strace
```bash
strace command                 # Trace syscalls
strace -p PID                  # Attach to process
strace -f command              # Follow forks
strace -e open,read,write cmd  # Specific syscalls
strace -e trace=file command   # File operations
strace -e trace=network cmd    # Network operations
strace -c command              # Summary statistics
strace -T command              # Time spent in syscall
strace -t command              # Timestamp each line
strace -o output.txt command   # Output to file
strace -s 1024 command         # String length
```

### ltrace
```bash
ltrace command                 # Library calls
ltrace -p PID                  # Attach to process
ltrace -e malloc+free command  # Specific functions
ltrace -c command              # Summary
ltrace -S command              # Include syscalls
```

## Java Profilers

### async-profiler
```bash
# Download from https://github.com/async-profiler/async-profiler

# CPU profiling
./asprof -d 30 -f cpu.html PID

# Allocation profiling
./asprof -e alloc -d 30 -f alloc.html PID

# Wall clock (includes wait time)
./asprof -e wall -d 30 -f wall.html PID

# Lock profiling
./asprof -e lock -d 30 -f lock.html PID

# Start/stop manually
./asprof start PID
./asprof stop PID
./asprof -o flamegraph PID > out.svg

# JFR output
./asprof -f output.jfr PID
```

### JFR (Java Flight Recorder)
```bash
# Start recording
jcmd PID JFR.start duration=60s filename=recording.jfr

# Dump recording
jcmd PID JFR.dump filename=dump.jfr

# Stop recording
jcmd PID JFR.stop

# Via JVM args
java -XX:StartFlightRecording=duration=60s,filename=out.jfr App
```

### jstack / jmap
```bash
jstack PID                     # Thread dump
jstack -l PID                  # With locks
jmap -heap PID                 # Heap summary
jmap -histo PID                # Histogram
jmap -dump:format=b,file=heap.bin PID  # Heap dump
```

### jstat
```bash
jstat -gc PID 1000             # GC stats every 1s
jstat -gcutil PID 1000         # GC utilization %
jstat -class PID               # Class loading
jstat -compiler PID            # JIT compilation
```

## Python Profilers

### py-spy
```bash
# Install: pip install py-spy

py-spy top --pid PID           # Live top view
py-spy record -o out.svg --pid PID  # Flame graph
py-spy record -o out.svg -- python script.py  # New process
py-spy record -r 100 --pid PID # 100 Hz sampling
py-spy record --native --pid PID  # Include native frames
py-spy dump --pid PID          # Thread dump
```

### cProfile
```bash
python -m cProfile script.py
python -m cProfile -o out.prof script.py
python -m cProfile -s cumtime script.py  # Sort by cumulative time

# Visualize with snakeviz
pip install snakeviz
snakeviz out.prof
```

### memory_profiler
```bash
pip install memory_profiler
python -m memory_profiler script.py

# In code: @profile decorator
```

### line_profiler
```bash
pip install line_profiler
kernprof -l -v script.py  # Functions with @profile decorator
```

## Ruby Profilers

### rbspy
```bash
# Install: cargo install rbspy

rbspy snapshot --pid PID       # Single snapshot
rbspy record --pid PID         # Record profile
rbspy record --format flamegraph --file out.svg --pid PID
rbspy record -- ruby script.rb # New process
rbspy top --pid PID            # Live top
```

## Node.js Profilers

### Node.js built-in
```bash
node --prof script.js          # V8 profiler
node --prof-process isolate-*.log > processed.txt

# Inspect profiling
node --inspect script.js       # Chrome DevTools
node --inspect-brk script.js   # Break on start
```

### 0x (flame graph)
```bash
npm install -g 0x
0x script.js                   # Generate flame graph
0x -o node script.js           # Open in browser
```

### clinic.js
```bash
npm install -g clinic
clinic doctor -- node script.js
clinic flame -- node script.js
clinic bubbleprof -- node script.js
```

## Memory Profilers

### Valgrind (memcheck)
```bash
valgrind command               # Memory errors
valgrind --leak-check=full cmd # Full leak check
valgrind --track-origins=yes cmd  # Track uninitialized
valgrind --show-leak-kinds=all cmd
valgrind --log-file=val.log cmd
```

### Valgrind (cachegrind)
```bash
valgrind --tool=cachegrind cmd # Cache simulation
cg_annotate cachegrind.out.*   # View results
```

### Valgrind (callgrind)
```bash
valgrind --tool=callgrind cmd  # Call graph profiler
callgrind_annotate callgrind.out.*
kcachegrind callgrind.out.*    # GUI viewer
```

### heaptrack
```bash
heaptrack command              # Record allocations
heaptrack --record-only cmd    # No analysis
heaptrack_gui heaptrack.*.gz   # GUI analysis
heaptrack_print heaptrack.*.gz # Text analysis
```

### massif (heap profiler)
```bash
valgrind --tool=massif command
ms_print massif.out.*          # View results
massif-visualizer massif.out.* # GUI
```

## CPU Analysis

### turbostat
```bash
turbostat                      # CPU frequency, power
turbostat -i 1                 # 1 second interval
turbostat -c 0-3               # Specific cores
turbostat --show Busy%,Bzy_MHz,PkgWatt
```

### cpupower
```bash
cpupower frequency-info        # Current policy
cpupower frequency-set -g performance  # Set governor
cpupower frequency-set -u 3.0GHz  # Set max freq
cpupower frequency-set -d 2.0GHz  # Set min freq
cpupower idle-info             # C-state info
```

### x86_energy_perf_policy
```bash
x86_energy_perf_policy         # Show current
x86_energy_perf_policy performance  # Set performance
x86_energy_perf_policy powersave    # Set powersave
```

## Quick Reference

| Task | Command |
|------|---------|
| Quick CPU stats | `perf stat command` |
| Record profile | `perf record -g command` |
| View profile | `perf report` |
| Live profiling | `perf top -g` |
| Flame graph | `perf script \| stackcollapse-perf.pl \| flamegraph.pl > out.svg` |
| Trace syscalls | `strace -c command` |
| Java CPU | `./asprof -d 30 -f cpu.html PID` |
| Python flame | `py-spy record -o out.svg --pid PID` |
| Memory check | `valgrind --leak-check=full command` |
| CPU frequency | `turbostat -i 1` |

## FOSDEM Insights

### JFR Safepoint Bias and Debug Info
Source: FOSDEM 2025 - Advancing Java Profiling: Achieving Precision and Stability with JFR, eBPF, and User Context

JFR execution sampling is not safepoint-biased (sampler lands at any point), but **symbolication** is biased to safepoints because debug info is only inserted at safepoint locations by the JIT compiler.

```bash
# Enable precise debug info for accurate line-level profiling
java -XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints YourApp

# async-profiler sets this flag by default
./asprof -d 30 -f cpu.html PID
```

**Key insight:** Without `-XX:+DebugNonSafepoints`, your flame graph line numbers may point to loop back-edges rather than actual hot code.

### async-profiler: CPU Time vs Wall Clock
Source: FOSDEM 2025 - Advancing Java Profiling

async-profiler uses kernel scheduler signals (ITIMER_PROF) to sample threads proportionally to their CPU consumption - unlike JFR which uses a fixed sampling period across all threads.

```bash
# CPU profiling - samples proportional to CPU activity
./asprof -e cpu -d 30 -f cpu.html PID

# Wall clock - includes blocked/waiting time
./asprof -e wall -d 30 -f wall.html PID

# Mixed stack traces (Java + native frames)
./asprof --cstack dwarf -d 30 -f mixed.html PID
```

**Key insight:** JFR sample counts don't correlate with CPU time; async-profiler samples do.

### OpenTelemetry eBPF Profiler for JVM
Source: FOSDEM 2025 - Advancing Java Profiling

The OpenTelemetry eBPF profiler runs in kernel space using VMStruct to walk JVM stacks - provides system-wide observability with lower overhead than userspace profilers.

```bash
# eBPF-based profiling runs in kernel space
# Uses perf_event_open for CPU timer signals
# Symbolication via VMStruct (same as async-profiler)
```

**Key insight:** eBPF profilers avoid signal handler overhead by running in kernel space, but still rely on unofficial JVM internals (VMStruct).

### Context Labels for Flame Graph Filtering
Source: FOSDEM 2025 - Advancing Java Profiling

Add dimensions beyond stack traces to profiling data (API endpoints, customer IDs, trace IDs) to filter flame graphs.

```java
// Thread coloring / context labeling
// Attach metadata to profiling samples
// Filter flame graphs by: endpoint, customer, trace_id
```

**Key insight:** Context labels transform profiling from "what code is slow" to "which requests/customers cause slowness."

### Modern CPU Architecture Reality
Source: FOSDEM 2025 - Almost Everything I Knew About Java Performance Was Wrong

Modern CPUs execute 10+ instructions per cycle using massive out-of-order execution with ~900 physical registers (vs 32-64 architectural). Branch prediction and memory load prediction are critical.

```bash
# Check CPU microarchitecture details
perf stat -d -d -d ./your_program

# Key metrics to watch:
# - instructions per cycle (IPC)
# - branch-misses
# - cache-misses (L1, LLC)
```

**Key insight:** If prediction works, `invokeInterface` costs nothing; if prediction fails, code runs 6x slower as CPU speculatively executes wrong paths.

### GraalVM Native Image Benchmarking with JMH
Source: FOSDEM 2025 - Unpick Performance Mysteries: Benchmarking GraalVM Native Executables

JMH can benchmark GraalVM native images directly, enabling AOT vs JIT comparison with identical workloads.

```bash
# Build native image with JMH
native-image --no-fallback -jar benchmarks.jar

# Profile native image with perf + dwarf call graphs
perf record -g --call-graph dwarf ./target/benchmarks

# Analyze assembly
perf annotate
```

**Key insight:** PGO (Profile Guided Optimization) with GraalVM can outperform HotSpot by unrolling non-counted loops - something HotSpot JIT cannot do.

### JVM Inlining Tuning for Native Image
Source: FOSDEM 2025 - Unpick Performance Mysteries

```bash
# Increase trivial method inlining budget
native-image -H:MaxNodesInTrivialMethod=40 -jar app.jar

# Default is 20 nodes - increase for more aggressive inlining
```

**Key insight:** Performance gaps between AOT and JIT often come from inlining decisions; tuning thresholds can close the gap.

### Native Memory Tracking (NMT) Extended API
Source: FOSDEM 2025 - Native Memory Tracking for All: Extending NMT Beyond HotSpot

JDK 25+ brings dynamic mem tags for tracking native allocations in core libraries and FFM.

```bash
# Enable NMT
java -XX:NativeMemoryTracking=summary MyApp

# Get NMT report
jcmd <pid> VM.native_memory summary
jcmd <pid> VM.native_memory detail

# Categories: Java Heap, Class, Thread, Code, GC, Compiler, Internal, etc.
```

**Key insight:** Current NMT only tracks HotSpot allocations; future versions will cover JNI bindings and FFM allocations via dynamic mem tags.

### Firefox Profiler for Cross-Platform Analysis
Source: FOSDEM 2025 - Collaborate Using the Firefox Profiler

Firefox Profiler works with Chrome, Node.js, and Linux perf data - shareable URLs preserve exact view state.

```bash
# Profile Node.js and view in Firefox Profiler
node --prof app.js
# Drag profile into profiler.firefox.com

# Profile with perf, convert to Firefox format
perf record -g ./program
perf script > profile.txt
# Import at profiler.firefox.com
```

**Key insight:** Shareable permalinks encode zoom level, filters, selected tab - collaborators see exactly what you see.

### PGO in LLVM: Instrumentation vs Sampling
Source: FOSDEM 2025 - Profile Guided Optimization (PGO) in LLVM

Two PGO approaches: instrumentation (precise, high overhead) vs sampling (production-safe, requires LBR hardware).

```bash
# Instrumentation PGO (not for production)
clang -fprofile-instr-generate program.c -o program
./program  # training run
llvm-profdata merge -output=default.profdata *.profraw
clang -fprofile-instr-use=default.profdata program.c -o program_opt

# Sampling PGO (production safe, requires LBR)
perf record -b ./program  # collect LBR data
# Use AutoFDO or llvm-profgen to convert
```

**Key insight:** AMD CPUs only support LBR from Zen 3+ (with caveats) and Zen 4+ fully - Intel has had it for years.

### Continuous Benchmarking Infrastructure
Source: FOSDEM 2026 - Continuous Performance Engineering

Change point detection algorithms outperform threshold-based alerting for regression detection.

```bash
# Apache DataSketches change point detection
# Catches 5% regressions in noisy data
# vs threshold alerting: catches only 100%+ regressions

# Key tuning for reproducible cloud benchmarks:
# 1. Use C-family instances (consistent CPU generation)
# 2. Use EBS with provisioned IOPS (not local SSDs)
# 3. Disable CPU frequency scaling
sudo cpupower frequency-set -g performance
```

**Key insight:** Local SSDs in cloud have noisy neighbor problems; EBS with provisioned IOPS is more stable for benchmarking.

### CPU Frequency and Power Tuning for Benchmarks
Source: FOSDEM 2026 - Continuous Performance Engineering

```bash
# Disable CPU frequency scaling
sudo cpupower frequency-set -g performance

# Disable turbo boost (Intel)
echo 1 | sudo tee /sys/devices/system/cpu/intel_pstate/no_turbo

# Disable turbo boost (AMD)
echo 0 | sudo tee /sys/devices/system/cpu/cpufreq/boost

# Pin CPU frequency
sudo cpupower frequency-set -d 2.4GHz -u 2.4GHz
```

**Key insight:** Cloud CPU virtualization is stable; I/O (especially "local" SSDs) causes most benchmark variance.

### Inxpect: Lightweight XDP/eBPF Profiling
Source: FOSDEM 2026 - Inxpect Profiling

Standard perf profiling of XDP programs drops throughput from 15M to 4M pps due to fentry/fexit overhead.

```c
// Inxpect approach: embed profiling macros directly
#include "inxpect.h"

SEC("xdp")
int xdp_prog(struct xdp_md *ctx) {
    START_TRACE();
    // ... packet processing ...
    END_TRACE();
    return XDP_PASS;
}
```

```bash
# Inxpect uses kfunc to call RDPMC directly
# 71% faster than perf without sampling
# 122% faster with sampling (every 64 packets)
```

**Key insight:** perf adds ~627 instructions overhead to a 2-instruction XDP drop program; Inxpect adds ~40.

### ADAPTEST: Unified Performance Analysis
Source: FOSDEM 2026 - Towards Unified Full-Stack Performance Analysis

CERN's ADAPTEST tool provides architecture-agnostic profiling with on/off-CPU analysis.

```bash
# Define system graph in YAML
# Run adaptest with Linux-perf module
adaptest -s system_requirements.yaml -d -- ./program

# Analyze results
adaptest-analyzer result_dir/
# Opens interactive web UI with flame graphs
```

**Key insight:** Different profilers (Intel VTune, AMD uProf, NVIDIA Nsight) fragment the ecosystem; ADAPTEST aims to unify with modular architecture.
