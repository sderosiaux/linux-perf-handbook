# Performance Profiling

CPU profiling, flame graphs, memory profilers, and language-specific tools.

## CPU Profiling (perf)

### Basic Statistics
```bash
perf stat command              # Basic stats
perf stat -d command           # Detailed stats
perf stat -d -d -d command     # Extra detailed (IPC, branch-misses, cache-misses)
perf stat -p PID sleep 10      # Profile running process
perf stat -a sleep 10          # System-wide

# Specific events
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

# Call graph with DWARF (for native images)
perf record -g --call-graph dwarf ./program
```

### Analyzing Profiles
```bash
perf report                    # Interactive report
perf report --stdio            # Text report
perf report -n                 # Show sample counts
perf report --sort comm,dso    # Sort by command, DSO
perf report -g graph           # Call graph as graph
perf report --no-children      # Self time only
perf annotate                  # Source/assembly annotation
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

### Brendan Gregg's FlameGraph Tools
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

## Java Profiling

### async-profiler
```bash
# Download from https://github.com/async-profiler/async-profiler
# Sets -XX:+DebugNonSafepoints by default for accurate line numbers

# CPU profiling - samples proportional to CPU activity
./asprof -e cpu -d 30 -f cpu.html PID

# Allocation profiling
./asprof -e alloc -d 30 -f alloc.html PID

# Wall clock (includes blocked/waiting time)
./asprof -e wall -d 30 -f wall.html PID

# Lock profiling
./asprof -e lock -d 30 -f lock.html PID

# Mixed stack traces (Java + native frames)
./asprof --cstack dwarf -d 30 -f mixed.html PID

# Start/stop manually
./asprof start PID
./asprof stop PID
./asprof -o flamegraph PID > out.svg

# JFR output
./asprof -f output.jfr PID
```

**Key insight:** JFR sample counts don't correlate with CPU time; async-profiler samples do because it uses kernel scheduler signals (ITIMER_PROF).

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

### Safepoint Bias and Debug Info

JFR execution sampling is not safepoint-biased (sampler lands at any point), but **symbolication** is biased to safepoints because debug info is only inserted at safepoint locations by the JIT compiler.

```bash
# Enable precise debug info for accurate line-level profiling
java -XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints YourApp
```

**Key insight:** Without `-XX:+DebugNonSafepoints`, your flame graph line numbers may point to loop back-edges rather than actual hot code.

### Context Labels for Flame Graph Filtering

Add dimensions beyond stack traces to profiling data (API endpoints, customer IDs, trace IDs) to filter flame graphs.

```java
// Thread coloring / context labeling
// Attach metadata to profiling samples
// Filter flame graphs by: endpoint, customer, trace_id
```

**Key insight:** Context labels transform profiling from "what code is slow" to "which requests/customers cause slowness."

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

### OpenTelemetry eBPF Profiler

eBPF-based profiling runs in kernel space using VMStruct to walk JVM stacks - provides system-wide observability with lower overhead than userspace profilers.

**Key insight:** eBPF profilers avoid signal handler overhead by running in kernel space, but still rely on unofficial JVM internals (VMStruct).

## Memory Profiling

### heaptrack
```bash
heaptrack command              # Record allocations
heaptrack --record-only cmd    # No analysis
heaptrack_gui heaptrack.*.gz   # GUI analysis
heaptrack_print heaptrack.*.gz # Text analysis
```

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

### massif (heap profiler)
```bash
valgrind --tool=massif command
ms_print massif.out.*          # View results
massif-visualizer massif.out.* # GUI
```

### Native Memory Tracking (NMT)

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

## Browser Profiling (Firefox Profiler)

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

## Hardware Counters (PMU)

### Modern CPU Architecture

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

### Lightweight XDP/eBPF Profiling

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

**Key insight:** perf adds ~627 instructions overhead to a 2-instruction XDP drop program; Inxpect (using kfunc to call RDPMC directly) adds ~40.

## Profile-Guided Optimization (PGO)

### LLVM: Instrumentation vs Sampling

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

### GraalVM Native Image with JMH

JMH can benchmark GraalVM native images directly, enabling AOT vs JIT comparison with identical workloads.

```bash
# Build native image with JMH
native-image --no-fallback -jar benchmarks.jar

# Profile native image with perf + dwarf call graphs
perf record -g --call-graph dwarf ./target/benchmarks

# Analyze assembly
perf annotate

# Increase trivial method inlining budget (default 20 nodes)
native-image -H:MaxNodesInTrivialMethod=40 -jar app.jar
```

**Key insight:** PGO with GraalVM can outperform HotSpot by unrolling non-counted loops - something HotSpot JIT cannot do. Performance gaps between AOT and JIT often come from inlining decisions.

## Continuous Benchmarking

### Change Point Detection

Change point detection algorithms outperform threshold-based alerting for regression detection.

```bash
# Apache DataSketches change point detection
# Catches 5% regressions in noisy data
# vs threshold alerting: catches only 100%+ regressions
```

### Cloud Benchmark Tuning

```bash
# Key tuning for reproducible cloud benchmarks:
# 1. Use C-family instances (consistent CPU generation)
# 2. Use EBS with provisioned IOPS (not local SSDs)
# 3. Disable CPU frequency scaling
sudo cpupower frequency-set -g performance

# Disable turbo boost (Intel)
echo 1 | sudo tee /sys/devices/system/cpu/intel_pstate/no_turbo

# Disable turbo boost (AMD)
echo 0 | sudo tee /sys/devices/system/cpu/cpufreq/boost

# Pin CPU frequency
sudo cpupower frequency-set -d 2.4GHz -u 2.4GHz
```

**Key insight:** Local SSDs in cloud have noisy neighbor problems; EBS with provisioned IOPS is more stable for benchmarking. Cloud CPU virtualization is stable; I/O causes most benchmark variance.

## Cross-Platform/Unified Profiling

### ADAPTEST

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

## Syscall Tracing

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

## Quick Reference

| Task | Command |
|------|---------|
| Quick CPU stats | `perf stat command` |
| Detailed CPU stats | `perf stat -d -d -d command` |
| Record profile | `perf record -g command` |
| View profile | `perf report` |
| Live profiling | `perf top -g` |
| Flame graph | `perf script \| stackcollapse-perf.pl \| flamegraph.pl > out.svg` |
| Trace syscalls | `strace -c command` |
| Java CPU | `./asprof -d 30 -f cpu.html PID` |
| Java wall clock | `./asprof -e wall -d 30 -f wall.html PID` |
| Python flame | `py-spy record -o out.svg --pid PID` |
| Memory check | `valgrind --leak-check=full command` |
| Heap tracking | `heaptrack command` |
| CPU frequency | `turbostat -i 1` |
| NMT summary | `jcmd <pid> VM.native_memory summary` |

## Go Profiling

Go has first-class profiling support via `runtime/pprof` and `net/http/pprof`. Profiles are protocol buffer format, analyzed with `go tool pprof`.

### CPU Profiling with runtime/pprof

```go
import (
    "os"
    "runtime/pprof"
)

func main() {
    f, _ := os.Create("cpu.prof")
    pprof.StartCPUProfile(f)
    defer pprof.StopCPUProfile()

    // ... your code ...
}
```

```bash
# Run and analyze
go run main.go
go tool pprof cpu.prof

# Interactive commands
(pprof) top10          # Top 10 functions by CPU
(pprof) top -cum       # By cumulative time
(pprof) list funcName  # Source with hotspots
(pprof) web            # Open flame graph in browser
(pprof) svg            # Generate SVG
(pprof) disasm funcName # Disassembly view
```

Sample output:
```
(pprof) top10
Showing nodes accounting for 4.2s, 84% of 5s total
      flat  flat%   sum%        cum   cum%
     1.5s   30%    30%      1.5s    30%  runtime.mapaccess1_fast64
     0.8s   16%    46%      0.8s    16%  runtime.memmove
     0.6s   12%    58%      3.2s    64%  main.processData
     0.5s   10%    68%      0.5s    10%  runtime.mallocgc
     0.4s    8%    76%      0.4s     8%  runtime.scanobject
     0.4s    8%    84%      4.0s    80%  main.handler
```

### HTTP Profiling with net/http/pprof

```go
import (
    "net/http"
    _ "net/http/pprof"  // Side-effect import
)

func main() {
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()
    // ... your code ...
}
```

```bash
# Collect 30-second CPU profile
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Heap profile
go tool pprof http://localhost:6060/debug/pprof/heap

# Goroutine profile
go tool pprof http://localhost:6060/debug/pprof/goroutine

# Block profile
go tool pprof http://localhost:6060/debug/pprof/block

# Mutex profile
go tool pprof http://localhost:6060/debug/pprof/mutex

# Execution trace (opens in browser)
curl -o trace.out http://localhost:6060/debug/pprof/trace?seconds=5
go tool trace trace.out
```

Available endpoints:
- `/debug/pprof/profile` - CPU profile (takes `seconds` param)
- `/debug/pprof/heap` - Heap allocations
- `/debug/pprof/goroutine` - All goroutine stacks
- `/debug/pprof/block` - Stack traces leading to blocking
- `/debug/pprof/mutex` - Mutex contention
- `/debug/pprof/threadcreate` - Stack traces that led to thread creation
- `/debug/pprof/trace` - Execution trace

### Heap Profiling: inuse_space vs alloc_space

Two heap profile modes serve different purposes:

```bash
# Current memory usage (what's holding memory NOW)
go tool pprof -inuse_space http://localhost:6060/debug/pprof/heap

# Total allocations (what's causing GC pressure)
go tool pprof -alloc_space http://localhost:6060/debug/pprof/heap

# Show allocation counts instead of bytes
go tool pprof -alloc_objects http://localhost:6060/debug/pprof/heap
go tool pprof -inuse_objects http://localhost:6060/debug/pprof/heap
```

**Key insight:**
- `inuse_space`: Debug memory leaks and high RSS
- `alloc_space`: Debug GC pressure and allocation hotspots
- High `alloc_space` but low `inuse_space` = short-lived allocations causing GC work

Sample heap analysis:
```
(pprof) top -inuse_space
Showing nodes accounting for 512MB, 89% of 576MB total
      flat  flat%   sum%        cum   cum%
    256MB   44%    44%     256MB    44%  bytes.makeSlice
    128MB   22%    67%     128MB    22%  encoding/json.(*decodeState).literalStore
     64MB   11%    78%      64MB    11%  main.cacheEntry
     64MB   11%    89%     320MB    56%  main.processRequest
```

### Goroutine Profiling and Leak Detection

```bash
# Get goroutine profile
go tool pprof http://localhost:6060/debug/pprof/goroutine

# Debug=2 gives full stack traces (not for pprof)
curl http://localhost:6060/debug/pprof/goroutine?debug=2

# Compare goroutine counts over time
watch -n 5 'curl -s http://localhost:6060/debug/pprof/goroutine?debug=1 | head -1'
```

Detecting leaks:
```bash
# Take baseline
curl -o goroutine1.prof http://localhost:6060/debug/pprof/goroutine

# After some time...
curl -o goroutine2.prof http://localhost:6060/debug/pprof/goroutine

# Compare (positive diff = leaked goroutines)
go tool pprof -base goroutine1.prof goroutine2.prof
(pprof) top
```

Common leak patterns:
```
goroutine 1 [chan receive]:       # Blocked on channel nobody sends to
goroutine 2 [select]:             # select with no default, all channels dead
goroutine 3 [semacquire]:         # Waiting on mutex held by dead goroutine
goroutine 4 [IO wait]:            # Connection never closed
```

### Mutex Contention Profiling

Enable mutex profiling (disabled by default due to overhead):

```go
import "runtime"

func main() {
    runtime.SetMutexProfileFraction(5)  // Sample 1/5 of mutex events
    // ... your code ...
}
```

```bash
# Collect mutex profile
go tool pprof http://localhost:6060/debug/pprof/mutex

(pprof) top
      flat  flat%   sum%        cum   cum%
    1.20s   60%    60%     1.20s    60%  sync.(*Mutex).Lock
    0.80s   40%   100%     0.80s    40%  sync.(*RWMutex).RLock
```

The profile shows **time spent waiting** to acquire locks, not time holding them.

### Block Profiling

Block profiling captures time goroutines spend blocked on synchronization primitives.

```go
import "runtime"

func main() {
    runtime.SetBlockProfileRate(1)  // Capture all blocking events
    // ... your code ...
}
```

```bash
go tool pprof http://localhost:6060/debug/pprof/block

(pprof) top
      flat  flat%   sum%        cum   cum%
    2.5s    50%    50%      2.5s    50%  runtime.chanrecv1
    1.5s    30%    80%      1.5s    30%  runtime.selectgo
    1.0s    20%   100%      1.0s    20%  sync.(*Cond).Wait
```

**Key insight:** High block time on `chanrecv`/`chansend` indicates producer-consumer imbalance. High `selectgo` time may indicate inefficient select statements.

### Execution Tracer

The execution tracer captures scheduling events, syscalls, GC, and more.

```go
import (
    "os"
    "runtime/trace"
)

func main() {
    f, _ := os.Create("trace.out")
    trace.Start(f)
    defer trace.Stop()

    // ... your code ...
}
```

```bash
# Generate trace
go run main.go

# Open trace viewer (browser-based)
go tool trace trace.out
```

Trace viewer provides:
- **Goroutine analysis**: See scheduling latency per goroutine
- **Network blocking**: Time waiting on network I/O
- **Sync blocking**: Time waiting on mutexes/channels
- **Syscall blocking**: Time in syscalls
- **Scheduler latency**: Time runnable but not running
- **GC events**: Stop-the-world pauses, concurrent GC phases
- **User annotations**: Custom regions/tasks

User annotations for custom regions:
```go
import "runtime/trace"

func processRequest(ctx context.Context, req *Request) {
    ctx, task := trace.NewTask(ctx, "processRequest")
    defer task.End()

    trace.WithRegion(ctx, "validate", func() {
        validate(req)
    })

    trace.WithRegion(ctx, "transform", func() {
        transform(req)
    })
}
```

### GODEBUG Options

Runtime debugging via environment variables:

```bash
# GC tracing
GODEBUG=gctrace=1 ./myapp
# Output: gc 1 @0.012s 2%: 0.026+0.45+0.003 ms clock, 0.10+0.36/0.84/0.27+0.014 ms cpu, 4->4->0 MB, 5 MB goal, 4 P

# Schedule tracing
GODEBUG=schedtrace=1000 ./myapp    # Print every 1000ms
GODEBUG=scheddetail=1,schedtrace=1000 ./myapp  # Detailed

# Allocation tracing (very verbose)
GODEBUG=allocfreetrace=1 ./myapp
```

GC trace format explained:
```
gc 1 @0.012s 2%: 0.026+0.45+0.003 ms clock, 0.10+0.36/0.84/0.27+0.014 ms cpu, 4->4->0 MB, 5 MB goal, 4 P
|  |  |      |   |                    |                                      |           |         |
|  |  |      |   |                    |                                      |           |         +- GOMAXPROCS
|  |  |      |   |                    |                                      |           +- heap goal
|  |  |      |   |                    |                                      +- heap before->after->live
|  |  |      |   |                    +- CPU time (assist/background/idle)
|  |  |      |   +- Wall clock time (STW1+concurrent+STW2)
|  |  |      +- % time in GC since start
|  |  +- Time since start
|  +- GC number
+- gc marker
```

### Escape Analysis

Understand what allocates on heap vs stack:

```bash
# Show escape analysis decisions
go build -gcflags='-m' ./...

# More verbose
go build -gcflags='-m -m' ./...

# Even more verbose
go build -gcflags='-m -m -m' ./...
```

Sample output:
```
./main.go:15:6: can inline processItem
./main.go:20:6: cannot inline handleRequest: function too complex
./main.go:23:12: leaking param: data to result ~r1 level=0
./main.go:25:17: &Result{...} escapes to heap
./main.go:30:9: make([]byte, n) escapes to heap
```

**Key insight:** "escapes to heap" = allocation. Reduce by:
- Passing pointers down instead of returning them up
- Using fixed-size arrays instead of slices when size is known
- Avoiding interface{} when concrete types work

### Flame Graphs with pprof

```bash
# Generate SVG flame graph
go tool pprof -svg cpu.prof > cpu.svg

# Interactive web UI with flame graph
go tool pprof -http=:8080 cpu.prof
# Navigate to /ui/flamegraph

# Compare two profiles
go tool pprof -diff_base baseline.prof current.prof
(pprof) top  # Shows regressions
```

### Continuous Profiling (Parca/Pyroscope)

For production continuous profiling:

```go
// Pyroscope integration
import "github.com/grafana/pyroscope-go"

func main() {
    pyroscope.Start(pyroscope.Config{
        ApplicationName: "my-app",
        ServerAddress:   "http://pyroscope:4040",
        ProfileTypes: []pyroscope.ProfileType{
            pyroscope.ProfileCPU,
            pyroscope.ProfileAllocObjects,
            pyroscope.ProfileAllocSpace,
            pyroscope.ProfileInuseObjects,
            pyroscope.ProfileInuseSpace,
            pyroscope.ProfileGoroutines,
            pyroscope.ProfileMutexCount,
            pyroscope.ProfileMutexDuration,
            pyroscope.ProfileBlockCount,
            pyroscope.ProfileBlockDuration,
        },
    })
    // ... your code ...
}
```

```bash
# Parca agent (eBPF-based, no code changes)
parca-agent --node=my-node --remote-store-address=parca:7070
```

**Key insight:** Parca uses eBPF for zero-instrumentation profiling. Pyroscope requires SDK integration but offers more control.

### Go Profiling Quick Reference

| Profile Type | Enable | Collect |
|-------------|--------|---------|
| CPU | automatic | `?seconds=30` on `/debug/pprof/profile` |
| Heap | automatic | `/debug/pprof/heap` |
| Goroutine | automatic | `/debug/pprof/goroutine` |
| Block | `runtime.SetBlockProfileRate(1)` | `/debug/pprof/block` |
| Mutex | `runtime.SetMutexProfileFraction(5)` | `/debug/pprof/mutex` |
| Trace | `trace.Start(f)` | `/debug/pprof/trace?seconds=5` |

## Rust Profiling

Rust binaries work well with standard Linux profiling tools. Key requirements: debug symbols and frame pointers.

### Building for Profiling

```toml
# Cargo.toml - profile for profiling
[profile.release]
debug = true  # Include debug symbols

# For accurate stack traces
[profile.release]
debug = true
force-frame-pointers = true  # Rust 1.79+
```

```bash
# Pre-1.79: Use RUSTFLAGS
RUSTFLAGS="-C force-frame-pointers=yes" cargo build --release

# Verify symbols are present
nm target/release/myapp | head
readelf -S target/release/myapp | grep debug
```

### cargo-flamegraph

The easiest way to generate flame graphs for Rust:

```bash
# Install
cargo install flamegraph

# Generate flame graph (runs perf under the hood)
cargo flamegraph --bin myapp

# With arguments
cargo flamegraph --bin myapp -- --config prod.toml

# Profile tests
cargo flamegraph --test integration_tests

# Profile benchmarks
cargo flamegraph --bench my_benchmark

# Profile release build
cargo flamegraph --release --bin myapp

# Output to custom file
cargo flamegraph -o profile.svg --bin myapp

# Include kernel stacks
cargo flamegraph --root --bin myapp
```

Output: `flamegraph.svg` in current directory.

### perf with Rust Binaries

```bash
# Record with call graphs (DWARF for accuracy)
perf record -g --call-graph dwarf target/release/myapp

# Record with frame pointers (lower overhead)
perf record -g --call-graph fp target/release/myapp

# Analyze
perf report
perf report --no-children  # Self time only

# Generate flame graph
perf script | stackcollapse-perf.pl | flamegraph.pl > rust.svg

# Annotate source (requires debug symbols)
perf annotate --source
```

Common issues and fixes:
```bash
# Unknown symbols? Ensure debug=true in Cargo.toml
# Verify: nm target/release/myapp | grep main

# Inlined functions missing? Use DWARF unwinding
perf record --call-graph dwarf ./myapp

# perf complains about permissions?
sudo sysctl kernel.perf_event_paranoid=1
# Or run with sudo
```

### DHAT for Heap Profiling

DHAT (Dynamic Heap Allocation Tool) profiles heap usage without Valgrind overhead.

```toml
# Cargo.toml
[dependencies]
dhat = "0.3"

[features]
dhat-heap = []  # Feature-gate to disable in production
```

```rust
#[cfg(feature = "dhat-heap")]
#[global_allocator]
static ALLOC: dhat::Alloc = dhat::Alloc;

fn main() {
    #[cfg(feature = "dhat-heap")]
    let _profiler = dhat::Profiler::new_heap();

    // ... your code ...
}
```

```bash
# Run with DHAT enabled
cargo run --release --features dhat-heap

# Output: dhat-heap.json
# View at https://nnethercote.github.io/dh_view/dh_view.html
```

DHAT output shows:
- Total allocations/deallocations
- Maximum heap usage
- Allocation sites with counts and sizes
- Access patterns (reads/writes per allocation)

Sample DHAT output:
```
dhat: Total:     1,234,567 bytes in 45,678 blocks
dhat: At t-gmax: 567,890 bytes in 12,345 blocks
dhat: At t-end:  0 bytes in 0 blocks (no leaks)

PP 1/45 (56.78% of total bytes)
  Total:     700,000 bytes (100%) in 7,000 blocks (100%)
  Max:       100 bytes
  At t-gmax: 500,000 bytes (88.01%) in 5,000 blocks (40.49%)
  Allocated at {
    main::process_items (src/main.rs:45)
    main::handler (src/main.rs:23)
  }
```

### criterion.rs Benchmarks

Statistically rigorous benchmarking:

```toml
# Cargo.toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "my_benchmark"
harness = false
```

```rust
// benches/my_benchmark.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn fibonacci(n: u64) -> u64 {
    match n {
        0 => 1,
        1 => 1,
        n => fibonacci(n-1) + fibonacci(n-2),
    }
}

fn criterion_benchmark(c: &mut Criterion) {
    c.bench_function("fib 20", |b| b.iter(|| fibonacci(black_box(20))));
}

criterion_group!(benches, criterion_benchmark);
criterion_main!(benches);
```

```bash
# Run benchmarks
cargo bench

# Run specific benchmark
cargo bench -- fib

# Compare against baseline
cargo bench -- --save-baseline main
# ... make changes ...
cargo bench -- --baseline main

# Generate profile-friendly output
cargo bench -- --profile-time 10
```

Sample output:
```
fib 20                  time:   [24.891 us 25.034 us 25.195 us]
                        change: [-2.1234% -0.9876% +0.1543%] (p = 0.12 > 0.05)
                        No change in performance detected.
Found 3 outliers among 100 measurements (3.00%)
  2 (2.00%) high mild
  1 (1.00%) high severe
```

### tokio-console for Async Debugging

Debug async runtime behavior:

```toml
# Cargo.toml
[dependencies]
console-subscriber = "0.2"
tokio = { version = "1", features = ["full", "tracing"] }
```

```rust
#[tokio::main]
async fn main() {
    console_subscriber::init();

    // ... your async code ...
}
```

```bash
# Build with tokio_unstable (required)
RUSTFLAGS="--cfg tokio_unstable" cargo build

# Run your app, then in another terminal:
cargo install tokio-console
tokio-console

# Connect to remote
tokio-console http://remote-host:6669
```

tokio-console shows:
- Active tasks and their states
- Task poll times and waker counts
- Resource utilization (channels, semaphores)
- Warnings for slow tasks or busy polling

### Compile-Time Optimization: LTO

Link-Time Optimization for smaller, faster binaries:

```toml
# Cargo.toml
[profile.release]
lto = true          # Full LTO (slow compile, best perf)
# lto = "thin"      # Thin LTO (faster compile, good perf)

codegen-units = 1   # Single codegen unit (required for full LTO benefit)
```

Compile time vs runtime tradeoff:
| Setting | Compile Time | Binary Size | Runtime |
|---------|-------------|-------------|---------|
| `lto = false` | Fast | Largest | Baseline |
| `lto = "thin"` | Medium | Medium | ~5-10% better |
| `lto = true, codegen-units = 1` | Slow | Smallest | ~10-20% better |

### Profile-Guided Optimization (PGO)

Two-phase compilation using runtime profiles:

```bash
# Phase 1: Build instrumented binary
RUSTFLAGS="-Cprofile-generate=/tmp/pgo-data" cargo build --release

# Phase 2: Run training workload
./target/release/myapp --typical-workload

# Phase 3: Merge profile data
llvm-profdata merge -o /tmp/pgo-data/merged.profdata /tmp/pgo-data

# Phase 4: Build optimized binary
RUSTFLAGS="-Cprofile-use=/tmp/pgo-data/merged.profdata" cargo build --release
```

PGO improvements vary by workload:
- CPU-bound: 5-20% improvement typical
- Branch-heavy code: 10-30% improvement
- Cache-sensitive code: Variable, sometimes 40%+

### Sampling Profiler: samply

Modern sampling profiler with Firefox Profiler integration:

```bash
# Install
cargo install samply

# Profile
samply record target/release/myapp

# Opens Firefox Profiler automatically
# Supports:
# - Call trees with timing
# - Flame graphs
# - Source annotation
# - Marker tracks
```

### Rust Profiling Quick Reference

| Task | Command |
|------|---------|
| Flame graph | `cargo flamegraph --release` |
| perf record | `perf record -g --call-graph dwarf ./target/release/app` |
| perf report | `perf report --no-children` |
| DHAT heap | `cargo run --features dhat-heap` |
| Benchmarks | `cargo bench` |
| Async debug | `RUSTFLAGS="--cfg tokio_unstable" cargo run` + `tokio-console` |
| Build with symbols | `[profile.release] debug = true` |
| Enable frame pointers | `[profile.release] force-frame-pointers = true` |
| Full LTO | `[profile.release] lto = true, codegen-units = 1` |
| PGO instrumented | `RUSTFLAGS="-Cprofile-generate=dir"` |
| PGO optimized | `RUSTFLAGS="-Cprofile-use=merged.profdata"` |

## C/C++ Profiling

### Valgrind Cachegrind (Cache Analysis)

Simulates CPU cache hierarchy to find cache-unfriendly code:

```bash
# Run cache simulation
valgrind --tool=cachegrind ./program

# Output: cachegrind.out.<pid>
cg_annotate cachegrind.out.12345

# Annotate specific source files
cg_annotate cachegrind.out.12345 src/hot_path.c

# Diff two runs
cg_diff cachegrind.out.baseline cachegrind.out.new

# Show annotation on diff
cg_annotate diff.out
```

Sample output:
```
--------------------------------------------------------------------------------
I   refs:      4,567,890,123
I1  misses:           45,678
LLi misses:           12,345
I1  miss rate:          0.01%
LLi miss rate:          0.00%

D   refs:      2,345,678,901  (1,567,890,123 rd   + 777,788,778 wr)
D1  misses:        5,678,901  (    4,567,890 rd   +   1,111,011 wr)
LLd misses:          345,678  (      234,567 rd   +     111,111 wr)
D1  miss rate:           0.2% (          0.3%     +         0.1%  )
LLd miss rate:           0.0% (          0.0%     +         0.0%  )

--------------------------------------------------------------------------------
         Ir          I1mr         ILmr          Dr         D1mr         DLmr          Dw        D1mw        DLmw  file:function
--------------------------------------------------------------------------------
 45,678,901         1,234          123  23,456,789      345,678       12,345  12,345,678     111,111      11,111  src/matrix.c:multiply
 12,345,678           567           56   6,789,012      123,456        5,678   3,456,789      56,789       5,678  src/sort.c:quicksort
```

**Key insight:** High D1/LL miss rates indicate cache-unfriendly access patterns. Look for:
- Strided access in arrays (column-major vs row-major)
- Poor data locality in linked structures
- False sharing in multithreaded code

### Valgrind Callgrind + KCachegrind

Call graph profiler with detailed cost attribution:

```bash
# Basic profiling
valgrind --tool=callgrind ./program

# With cache simulation
valgrind --tool=callgrind --cache-sim=yes ./program

# With branch prediction simulation
valgrind --tool=callgrind --branch-sim=yes ./program

# Profile only part of execution
valgrind --tool=callgrind --instr-atstart=no ./program
# Then in another terminal: callgrind_control -i on
# Later: callgrind_control -i off

# Dump intermediate results
callgrind_control -d

# Output: callgrind.out.<pid>
```

Analysis:
```bash
# CLI annotation
callgrind_annotate callgrind.out.12345
callgrind_annotate --auto=yes callgrind.out.12345  # Include all source

# GUI analysis (highly recommended)
kcachegrind callgrind.out.12345
# or
qcachegrind callgrind.out.12345
```

KCachegrind provides:
- Flat profile with inclusive/exclusive costs
- Call graph visualization
- Source annotation with cost per line
- Callee/caller maps
- Treemap view for size comparison

Sample callgrind output:
```
--------------------------------------------------------------------------------
Profile data file 'callgrind.out.12345' (creator: callgrind-3.18.1)
--------------------------------------------------------------------------------
I1 cache:         32768 B, 64 B, 8-way associative
D1 cache:         32768 B, 64 B, 8-way associative
LL cache:         262144 B, 64 B, 8-way associative
Timerange: Basic block 0 - 123456789
Trigger: Program termination

--------------------------------------------------------------------------------
            Ir
--------------------------------------------------------------------------------
 1,234,567,890  PROGRAM TOTALS

--------------------------------------------------------------------------------
            Ir  file:function
--------------------------------------------------------------------------------
   456,789,012  src/engine.c:process_frame [/path/to/program]
   234,567,890  src/parser.c:parse_token [/path/to/program]
   123,456,789  src/allocator.c:alloc_block [/path/to/program]
```

### Valgrind Massif (Heap Profiler)

Track heap usage over time:

```bash
# Basic heap profiling
valgrind --tool=massif ./program

# Include stack in measurements
valgrind --tool=massif --stacks=yes ./program

# Time-based snapshots (vs instruction-based)
valgrind --tool=massif --time-unit=ms ./program

# Detailed snapshots more frequently
valgrind --tool=massif --detailed-freq=1 ./program

# Track specific allocation size threshold
valgrind --tool=massif --threshold=0.1 ./program

# Output: massif.out.<pid>
```

Analysis:
```bash
# CLI visualization
ms_print massif.out.12345

# GUI visualization
massif-visualizer massif.out.12345
```

Sample ms_print output:
```
    MB
 64.0^                                                          #
     |                                                     @@@@##
     |                                                @@@@@@@@@@#
     |                                           @@@@@@@@@@@@@@@#
     |                                      @@@@@@@@@@@@@@@@@@@@#
     |                                 @@@@@@@@@@@@@@@@@@@@@@@@@#
     |                            @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@#
     |                       @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@#
     |                  @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@#
     |             @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@#
     |        @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@#
     |   @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@#
     | @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
   0 +---------------------------------------------------------->
     0                                                         10s

Number of snapshots: 50
 Detailed snapshots: [5, 15, 25, 35, 45 (peak)]

--------------------------------------------------------------------------------
  n        time(i)         total(B)   useful-heap(B) extra-heap(B)    stacks(B)
--------------------------------------------------------------------------------
 45     10,234,567       67,108,864       64,000,000     3,108,864            0
--------------------------------------------------------------------------------

->64.00% (64,000,000B) 0x4012AB: process_data (processor.c:45)
| ->48.00% (48,000,000B) 0x4013CD: allocate_buffer (buffer.c:123)
| | ->48.00% (48,000,000B) 0x401234: malloc (vg_replace_malloc.c:381)
| ->16.00% (16,000,000B) 0x4014EF: create_cache (cache.c:67)
|   ->16.00% (16,000,000B) 0x401234: malloc (vg_replace_malloc.c:381)
```

### Intel VTune

Comprehensive profiler for Intel CPUs (free for open source):

```bash
# Install (from Intel oneAPI)
# https://www.intel.com/content/www/us/en/developer/tools/oneapi/vtune-profiler.html

# Hotspots analysis (sampling)
vtune -collect hotspots ./program

# Microarchitecture analysis (top-down)
vtune -collect uarch-exploration ./program

# Memory access analysis
vtune -collect memory-access ./program

# Threading analysis
vtune -collect threading ./program

# HPC analysis
vtune -collect hpc-performance ./program

# Custom analysis
vtune -collect-with runsa -knob enable-stack-collection=true ./program

# Analyze specific function
vtune -collect hotspots -inline-mode on ./program

# Report generation
vtune -report hotspots -r r001hs

# CLI summary
vtune -report summary -r r001hs
```

Sample VTune hotspots output:
```
Elapsed Time: 10.234s
CPU Time: 9.876s

Top Hotspots
Function                     CPU Time  %
------------------------------------------
process_frame                   3.2s   32.4%
parse_token                     2.1s   21.3%
malloc                          1.5s   15.2%
hash_lookup                     0.9s    9.1%
memcpy                          0.7s    7.1%
[Others]                        1.5s   14.9%

Top Tasks
Loop at processor.c:45-67       2.8s   28.4%
Loop at parser.c:123-145        1.9s   19.2%
```

### AMD uProf

AMD's profiler for AMD CPUs:

```bash
# Install from AMD
# https://developer.amd.com/amd-uprof/

# CPU profiling
AMDuProfCLI collect --config assess ./program

# IBS (Instruction-Based Sampling)
AMDuProfCLI collect --config ibs ./program

# Memory analysis
AMDuProfCLI collect --config mem ./program

# Report
AMDuProfCLI report -i session_dir

# GUI
AMDuProfUI
```

### Sanitizers

Compiler-based instrumentation for runtime bug detection:

#### AddressSanitizer (ASan)
Detects memory errors (buffer overflow, use-after-free, etc.):

```bash
# Compile with ASan
gcc -fsanitize=address -g program.c -o program
clang -fsanitize=address -g program.c -o program

# Run (2-3x slowdown typical)
./program

# Configure via environment
ASAN_OPTIONS=detect_leaks=1:detect_stack_use_after_return=1 ./program

# Common options
ASAN_OPTIONS=\
  detect_leaks=1:\
  detect_stack_use_after_return=1:\
  check_initialization_order=1:\
  strict_init_order=1:\
  detect_odr_violation=2:\
  halt_on_error=0
```

Sample ASan output:
```
=================================================================
==12345==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x...
READ of size 4 at 0x... thread T0
    #0 0x4012ab in process_data src/process.c:45
    #1 0x4013cd in main src/main.c:23
    #2 0x7f... in __libc_start_main

0x... is located 4 bytes to the right of 100-byte region [0x...,0x...)
allocated by thread T0 here:
    #0 0x... in malloc
    #1 0x4011ef in allocate_buffer src/buffer.c:12
    #2 0x4012a0 in process_data src/process.c:40
```

#### ThreadSanitizer (TSan)
Detects data races:

```bash
# Compile with TSan
gcc -fsanitize=thread -g program.c -o program -lpthread
clang -fsanitize=thread -g program.c -o program -lpthread

# Run (5-15x slowdown, 5-10x memory overhead)
./program

TSAN_OPTIONS=second_deadlock_stack=1:history_size=4 ./program
```

Sample TSan output:
```
==================
WARNING: ThreadSanitizer: data race (pid=12345)
  Write of size 8 at 0x... by thread T1:
    #0 increment_counter src/counter.c:15
    #1 worker_thread src/worker.c:45

  Previous read of size 8 at 0x... by thread T2:
    #0 read_counter src/counter.c:20
    #1 monitor_thread src/monitor.c:30

  Location is global 'counter' of size 8 at 0x... (program+0x...)
==================
```

#### MemorySanitizer (MSan)
Detects uninitialized memory reads (Clang only):

```bash
# Compile with MSan (requires all dependencies also compiled with MSan)
clang -fsanitize=memory -g program.c -o program

# Track origins of uninitialized values
clang -fsanitize=memory -fsanitize-memory-track-origins=2 -g program.c -o program

./program
```

#### UndefinedBehaviorSanitizer (UBSan)
Detects undefined behavior:

```bash
# Compile with UBSan
gcc -fsanitize=undefined -g program.c -o program
clang -fsanitize=undefined -g program.c -o program

# Specific checks
gcc -fsanitize=signed-integer-overflow,null,bounds program.c -o program

# All checks
clang -fsanitize=undefined,float-divide-by-zero,unsigned-integer-overflow program.c
```

#### Sanitizer Overhead Summary

| Sanitizer | Slowdown | Memory | Use Case |
|-----------|----------|--------|----------|
| ASan | 2-3x | 3-4x | Memory safety, use in CI |
| TSan | 5-15x | 5-10x | Data races, concurrent code |
| MSan | 3x | 2x | Uninitialized reads |
| UBSan | <1.5x | Minimal | Undefined behavior, always-on in debug |

### LTO (Link-Time Optimization)

Cross-module optimization at link time:

```bash
# GCC LTO
gcc -flto -O2 *.c -o program

# With parallel linking
gcc -flto=auto -O2 *.c -o program  # Auto-detect cores
gcc -flto=4 -O2 *.c -o program     # 4 parallel jobs

# Clang LTO
clang -flto *.c -o program         # Full LTO
clang -flto=thin *.c -o program    # ThinLTO (faster compile)

# CMake
cmake -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_INTERPROCEDURAL_OPTIMIZATION=ON ..
```

**Key insight:** LTO enables cross-file inlining and dead code elimination. Thin LTO offers ~80% of full LTO benefit with much faster compile times.

### PGO (Profile-Guided Optimization)

```bash
# GCC PGO workflow
# Step 1: Build instrumented binary
gcc -fprofile-generate -O2 program.c -o program

# Step 2: Run representative workload
./program < training_data.txt

# Step 3: Build optimized binary
gcc -fprofile-use -O2 program.c -o program_opt

# Clang PGO workflow
# Step 1: Instrumented build
clang -fprofile-instr-generate -O2 program.c -o program

# Step 2: Run workload (generates default.profraw)
./program < training_data.txt

# Step 3: Merge profiles
llvm-profdata merge -output=default.profdata *.profraw

# Step 4: Optimized build
clang -fprofile-instr-use=default.profdata -O2 program.c -o program_opt
```

PGO improvements:
- Branch prediction hints: 5-15%
- Function layout: 2-5%
- Inlining decisions: 5-10%
- Total: 10-30% typical for branch-heavy code

### AutoFDO (Automatic Feedback-Directed Optimization)

Use production sampling profiles instead of instrumented builds:

```bash
# Step 1: Build with debug info
gcc -g -O2 program.c -o program

# Step 2: Collect perf profile in production
perf record -b -e br_inst_retired.near_taken:pp ./program

# Step 3: Convert to AutoFDO format
create_gcov --binary=./program --profile=perf.data \
            --gcov=program.afdo --gcov_version=1

# Step 4: Build with AutoFDO profile
gcc -fauto-profile=program.afdo -O2 program.c -o program_opt

# LLVM AutoFDO (requires llvm-profgen)
perf record -b -e cycles:u ./program
llvm-profgen --binary=./program --output=program.prof --perfdata=perf.data
clang -fprofile-sample-use=program.prof -O2 program.c -o program_opt
```

**Key insight:** AutoFDO uses sampling (no instrumentation overhead) so you can profile in production. Quality depends on LBR (Last Branch Record) support.

### BOLT (Binary Optimization and Layout Tool)

Post-link optimization using profiles:

```bash
# Build with relocations preserved
clang -Wl,--emit-relocs -O2 program.c -o program

# Collect perf data with LBR
perf record -e cycles:u -j any,u -o perf.data ./program

# Convert perf data
perf2bolt -p perf.data -o perf.fdata ./program

# Optimize binary layout
llvm-bolt program -o program.bolt -data=perf.fdata \
    -reorder-blocks=ext-tsp \
    -reorder-functions=hfsort+ \
    -split-functions \
    -split-all-cold \
    -dyno-stats

# 10-30% speedup typical for large binaries
```

### CMake Integration for Optimized Builds

```cmake
# CMakeLists.txt

# LTO
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)

# PGO - generate phase
option(PGO_GENERATE "Build instrumented binary for PGO" OFF)
if(PGO_GENERATE)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fprofile-generate")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fprofile-generate")
endif()

# PGO - use phase
option(PGO_USE "Use PGO profile data" OFF)
if(PGO_USE)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fprofile-use")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fprofile-use")
endif()

# Sanitizers (debug builds)
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    add_compile_options(-fsanitize=address,undefined)
    add_link_options(-fsanitize=address,undefined)
endif()
```

```bash
# Build workflow
mkdir build && cd build

# Debug with sanitizers
cmake -DCMAKE_BUILD_TYPE=Debug ..
make

# Release with LTO
cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_INTERPROCEDURAL_OPTIMIZATION=ON ..
make

# PGO training build
cmake -DCMAKE_BUILD_TYPE=Release -DPGO_GENERATE=ON ..
make
./program < training.txt

# PGO optimized build
cmake -DCMAKE_BUILD_TYPE=Release -DPGO_USE=ON ..
make
```

### C/C++ Profiling Quick Reference

| Task | Command |
|------|---------|
| Cache analysis | `valgrind --tool=cachegrind ./program && cg_annotate cachegrind.out.*` |
| Call graph | `valgrind --tool=callgrind ./program && kcachegrind callgrind.out.*` |
| Heap over time | `valgrind --tool=massif ./program && ms_print massif.out.*` |
| Memory errors | `gcc -fsanitize=address -g && ./program` |
| Data races | `gcc -fsanitize=thread -g && ./program` |
| Undefined behavior | `gcc -fsanitize=undefined -g && ./program` |
| VTune hotspots | `vtune -collect hotspots ./program` |
| AMD uProf | `AMDuProfCLI collect --config assess ./program` |
| LTO build | `gcc -flto -O2 *.c -o program` |
| PGO generate | `gcc -fprofile-generate -O2 *.c -o program && ./program` |
| PGO use | `gcc -fprofile-use -O2 *.c -o program_opt` |
| AutoFDO | `perf record -b ./program && create_gcov && gcc -fauto-profile` |

## Cross-Language Comparison

| Capability | Go | Rust | C/C++ |
|------------|----|----- |-------|
| CPU profiling | `runtime/pprof` | `perf`, `cargo-flamegraph` | `perf`, VTune, uProf |
| Heap profiling | `heap` profile | DHAT, heaptrack | Massif, heaptrack |
| Flame graphs | `go tool pprof -http` | `cargo flamegraph` | FlameGraph scripts |
| Race detection | `go build -race` | `cargo +nightly miri`, TSan | TSan |
| Memory safety | GC, no unsafe | Borrow checker, ASan | ASan, MSan, Valgrind |
| Benchmarking | `testing.B` | `criterion.rs` | Google Benchmark |
| PGO | Limited | `RUSTFLAGS=-Cprofile-*` | Full GCC/Clang support |
| LTO | Default | `lto = true` | `-flto` |
| Async debugging | N/A (goroutines) | `tokio-console` | N/A |
| Execution tracing | `go tool trace` | `tracing` crate | LTTng, perf |

## Profiling Workflow Summary

### Quick Performance Check
```bash
# Any language
perf stat ./program

# Go
go tool pprof -http=:8080 http://localhost:6060/debug/pprof/profile?seconds=30

# Rust
cargo flamegraph --release

# C/C++
perf record -g ./program && perf report
```

### Memory Investigation
```bash
# Go
go tool pprof -http=:8080 http://localhost:6060/debug/pprof/heap

# Rust (build with dhat feature)
cargo run --features dhat-heap

# C/C++
valgrind --tool=massif ./program && ms_print massif.out.*
heaptrack ./program && heaptrack_gui heaptrack.*.gz
```

### Concurrency Issues
```bash
# Go
go build -race && ./program
# Check goroutine profile for leaks
curl http://localhost:6060/debug/pprof/goroutine?debug=2

# Rust
RUSTFLAGS="--cfg tokio_unstable" cargo run  # then tokio-console

# C/C++
gcc -fsanitize=thread -g program.c && ./program
```

### Production Profiling
```bash
# Go: Continuous profiling
# Add pyroscope.Start() or use Parca agent

# Rust/C/C++: Sampling profiler
perf record -g -p PID sleep 60
# Or use eBPF-based profiler (bcc, Parca agent)
```

## See Also

- [eBPF & Tracing](06-ebpf-tracing.md) - BCC tools, bpftrace, kernel tracing
- [Java/JVM](10-java-jvm.md) - JVM-specific profiling and tuning
- [GPU & HPC](11-gpu-hpc.md) - GPU profiling, CUDA optimization
- [Kernel Tuning](08-kernel-tuning.md) - CPU governors, scheduler tuning
