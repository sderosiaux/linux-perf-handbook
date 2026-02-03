# Java/JVM

JVM monitoring, profiling, and tuning for Java 21+ (LTS).

## JVM Monitoring

### jps (Java Process Status)
```bash
jps                            # List Java processes
jps -l                         # Full main class
jps -v                         # JVM arguments
jps -m                         # Main method arguments
```

### jstat (JVM Statistics)
```bash
jstat -gc PID 1000             # GC stats every 1s
jstat -gcutil PID 1000         # GC utilization %
jstat -gccause PID 1000        # GC cause
jstat -class PID               # Class loading
jstat -compiler PID            # JIT compilation
jstat -printcompilation PID 1000  # Method compilation
```

**GC columns (generational collectors):**
- `S0C/S1C` - Survivor space capacity
- `S0U/S1U` - Survivor space used
- `EC/EU` - Eden capacity/used
- `OC/OU` - Old gen capacity/used
- `MC/MU` - Metaspace capacity/used
- `YGC/YGCT` - Young GC count/time
- `FGC/FGCT` - Full GC count/time

### jinfo
```bash
jinfo PID                      # All info
jinfo -flags PID               # JVM flags
jinfo -sysprops PID            # System properties
jinfo -flag MaxHeapSize PID    # Specific flag
```

### jcmd (JVM Command)
```bash
jcmd PID help                  # Available commands
jcmd PID VM.version            # JVM version
jcmd PID VM.flags              # JVM flags
jcmd PID VM.system_properties  # System properties
jcmd PID VM.uptime             # Uptime
jcmd PID VM.native_memory      # Native memory (requires -XX:NativeMemoryTracking=summary)
jcmd PID GC.heap_info          # Heap info
jcmd PID GC.run                # Trigger GC
jcmd PID Thread.print          # Thread dump
jcmd PID Thread.dump_to_file -format=json threads.json  # JSON thread dump (JDK 21+)
jcmd PID Compiler.queue        # JIT queue
```

## Thread Analysis

### jstack
```bash
jstack PID                     # Thread dump
jstack -l PID                  # With locks
jstack -e PID                  # Extended info (JDK 21+)
```

### Thread States
- `RUNNABLE` - Executing or ready
- `BLOCKED` - Waiting for monitor
- `WAITING` - Object.wait() without timeout
- `TIMED_WAITING` - sleep() or wait() with timeout
- `NEW` - Not yet started
- `TERMINATED` - Completed

### Detect Deadlocks
```bash
jstack PID | grep -A 50 "Found.*deadlock"
jcmd PID Thread.print
```

## Virtual Threads (Project Loom)

Virtual threads are lightweight threads managed by the JVM, not the OS. Production-ready in JDK 21.

### Performance Characteristics
| Aspect | Platform Threads | Virtual Threads |
|--------|-----------------|-----------------|
| Memory | ~2MB stack | ~few KB (grows as needed) |
| Creation | ~100-300us (kernel syscall) | ~1us (JVM managed) |
| Context switch | OS scheduler overhead | JVM continuation swap |
| Max practical count | 1K-10K | Millions |
| Best for | CPU-bound, short tasks | I/O-bound, blocking ops |

### When to Use Virtual Threads
```java
// IDEAL: I/O-bound workloads with blocking operations
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<Response>> futures = urls.stream()
        .map(url -> executor.submit(() -> httpClient.send(request, handler)))
        .toList();
    // Each blocking HTTP call releases carrier thread
}

// AVOID: CPU-intensive computation
// Virtual threads provide no benefit - use platform thread pool sized to CPU cores
ExecutorService cpuPool = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());
```

### Creating Virtual Threads
```java
// Direct creation
Thread.startVirtualThread(() -> task());

// Builder pattern
Thread vt = Thread.ofVirtual()
    .name("worker-", 0)  // worker-0, worker-1, ...
    .start(() -> task());

// Executor (recommended for most use cases)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> blockingOperation());
}
```

### Thread Pinning (JDK 21-23)

**Critical limitation in JDK 21-23:** Virtual threads pin to carrier threads during:
1. `synchronized` blocks with blocking operations inside
2. Native method calls (JNI)

```java
// PROBLEMATIC in JDK 21-23: pins carrier thread
synchronized (lock) {
    socket.read();  // Blocking I/O inside synchronized = pinned
}

// SOLUTION for JDK 21-23: use ReentrantLock
private final ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    socket.read();  // No pinning with ReentrantLock
} finally {
    lock.unlock();
}
```

**JDK 24+ (JEP 491):** Synchronized no longer causes pinning. The JVM unmounts virtual threads even inside synchronized blocks.

### Detecting Pinned Threads

```bash
# Runtime detection via system property
java -Djdk.tracePinnedThreads=full MyApp    # Full stack trace
java -Djdk.tracePinnedThreads=short MyApp   # Problematic frames only

# JFR event (enabled by default, threshold 20ms)
jfr print --events jdk.VirtualThreadPinned recording.jfr
```

**JFR events for virtual threads:**
- `jdk.VirtualThreadStart` / `jdk.VirtualThreadEnd` - lifecycle (disabled by default)
- `jdk.VirtualThreadPinned` - blocking while pinned (threshold 20ms)
- `jdk.VirtualThreadSubmitFailed` - resource issues

### JFR Streaming for Pinned Thread Detection
```java
// Continuous monitoring with JFR Event Streaming
try (var stream = new RecordingStream()) {
    stream.enable("jdk.VirtualThreadPinned").withStackTrace();
    stream.onEvent("jdk.VirtualThreadPinned", event -> {
        log.warn("Virtual thread pinned for {}ms: {}",
            event.getDuration().toMillis(),
            event.getStackTrace());
    });
    stream.startAsync();
}
```

### Structured Concurrency (Preview)

Structured concurrency treats groups of related tasks as a unit. Preview in JDK 21-24, revised API in JDK 25.

```java
// JDK 25+ syntax (--enable-preview required)
try (var scope = StructuredTaskScope.open()) {
    Subtask<User> userTask = scope.fork(() -> fetchUser(id));
    Subtask<Order> orderTask = scope.fork(() -> fetchOrder(id));

    scope.join();  // Wait for all, fail if any fails

    return new UserOrder(userTask.get(), orderTask.get());
}

// JDK 21-24 syntax
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Subtask<User> userTask = scope.fork(() -> fetchUser(id));
    Subtask<Order> orderTask = scope.fork(() -> fetchOrder(id));

    scope.join().throwIfFailed();

    return new UserOrder(userTask.get(), orderTask.get());
}
```

**JDK 25 Joiners** (completion policies):
```java
// Race: first success wins, cancel others
var scope = StructuredTaskScope.open(Joiner.anySuccessfulResultOrThrow());

// All results gathered
var scope = StructuredTaskScope.open(Joiner.allSuccessfulOrThrow());
```

## Garbage Collection

### GC Selection for Java 21+

**ZGC (Generational) - Recommended for latency-sensitive workloads:**
```bash
-XX:+UseZGC -XX:+ZGenerational    # JDK 21-22
-XX:+UseZGC                        # JDK 23+ (generational is default)
```

**G1GC - Default, good general-purpose:**
```bash
-XX:+UseG1GC                       # Default since JDK 9
```

**Shenandoah - Alternative low-latency:**
```bash
-XX:+UseShenandoahGC
-XX:+ShenandoahGenerational        # JDK 25+ (generational mode)
```

### When to Choose Which GC

| Criteria | ZGC | G1GC | Shenandoah |
|----------|-----|------|------------|
| Latency target | <1ms pauses | <200ms pauses | <10ms pauses |
| Heap size | Large (8GB+) | Any | Large (4GB+) |
| CPU overhead | Higher | Lower | Moderate |
| Throughput | Good | Best | Good |
| Tuning needed | Minimal | Some | Minimal |

**Netflix recommendation:** ZGC as default for JDK 21+ services. Their testing showed:
- Non-generational ZGC: 36% more CPU than G1 (worst case)
- Generational ZGC: ~10% better CPU than G1
- Pause times consistently <1ms vs G1's variable pauses

### Generational ZGC Tuning (JDK 21+)
```bash
-XX:+UseZGC
-XX:+ZGenerational                 # Enable generational mode (default in JDK 23+)
-Xmx16g                            # Max heap - often the only tuning needed
-XX:SoftMaxHeapSize=12g            # Soft limit ZGC tries to stay under
-XX:ZCollectionInterval=0          # Disable proactive GC (default)
-XX:ZUncommitDelay=300             # Seconds before uncommitting unused memory
```

**ZGC diagnostics:**
```bash
-Xlog:gc*:file=gc.log:time,uptime:filecount=5,filesize=10m
-Xlog:gc+phases=debug              # Detailed phase timing
```

### G1GC Tuning
```bash
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200           # Target pause time
-XX:G1HeapRegionSize=16m           # Region size (1-32MB)
-XX:InitiatingHeapOccupancyPercent=45  # Start mixed GC
-XX:G1ReservePercent=10            # Reserve for promotion
-XX:G1NewSizePercent=5             # Min young gen %
-XX:G1MaxNewSizePercent=60         # Max young gen %
```

### GC Logging (Unified Logging)
```bash
# Standard GC logging
-Xlog:gc*:file=gc.log:time,uptime:filecount=5,filesize=10m

# Verbose with phases
-Xlog:gc*=debug:file=gc.log

# Specific components
-Xlog:gc+heap=debug:file=heap.log
-Xlog:safepoint:file=safepoint.log
-Xlog:gc+ergo=debug                # Ergonomic decisions
```

### GC-Friendly Coding Patterns

Real-world data shows 25-50% CPU time spent on GC in high-throughput systems.

```java
// AVOID: Returning new allocations escapes to heap
byte[] process() { return new byte[1024]; }

// PREFER: Accept pre-allocated buffer (may stay on stack in caller)
void process(byte[] buffer) { /* fill buffer */ }

// AVOID: Boxed types in hot paths
Map<Long, Boolean> flags;  // Each entry = object allocations

// PREFER: Primitive collections or packed representations
long[] flagBits;  // Bitset pattern

// AVOID: Hidden allocations
time.format(...)  // Creates intermediate strings
String.format(...)

// PREFER: Pre-sized collections
new ArrayList<>(expectedSize);
new HashMap<>(expectedSize, 1.0f);
```

**Key insight:** Copying 64 bytes costs roughly the same as dereferencing a pointer due to cache line granularity - prefer value semantics over pointer indirection when objects are small.

### Struct-of-Arrays for Memory Efficiency

Data-oriented design reduces per-object overhead. JVM object headers are 12-16 bytes with compressed oops (8 bytes with Compact Object Headers in JDK 24+).

```java
// AVOID: Array of objects (N * (header + fields + padding))
record Component(int type, boolean enabled, int parent, int[] children) {}
Component[] components;  // Each Component has object header overhead

// PREFER: Struct of arrays (single header per array, packed data)
class ComponentTable {
    byte[] types;           // 1 byte per component vs 4 for int field
    BitSet enabled;         // 1 bit per component vs 1+ bytes for boolean
    short[] parents;        // Use smallest sufficient type
    // Children stored separately if sparse
    Map<Integer, short[]> childrenByIndex;
}
```

### Object Pooling

Object pools reduce allocation pressure between GC cycles.

```java
// ThreadLocal pool pattern - CAUTION with virtual threads
// Virtual threads: millions of threads = millions of ThreadLocal instances
private static final ThreadLocal<byte[]> BUFFER_POOL =
    ThreadLocal.withInitial(() -> new byte[8192]);

// For virtual threads, prefer scoped allocation or arena patterns
void process() {
    byte[] buf = new byte[8192];  // Stack allocation candidate
    // use buffer
}

// For heavy reuse with virtual threads, consider external pool
// Apache Commons Pool2 or custom bounded pool with synchronization
```

## Memory Management

### Heap Sizing
```bash
-Xms4g                             # Initial heap
-Xmx4g                             # Max heap (set equal to -Xms for predictability)
-XX:MetaspaceSize=256m             # Initial metaspace
-XX:MaxMetaspaceSize=512m          # Max metaspace
```

### Compact Object Headers (JDK 24+)
```bash
# Reduces object header from 12-16 bytes to 8 bytes
-XX:+UseCompactObjectHeaders       # Experimental in JDK 24
```

### Heap Analysis with jmap
```bash
jmap -heap PID                     # Heap summary
jmap -histo PID                    # Object histogram
jmap -histo:live PID               # Live objects only (triggers GC)
jmap -dump:format=b,file=heap.hprof PID       # Heap dump
jmap -dump:live,format=b,file=heap.hprof PID  # Live only
```

### Heap Dump Analysis
```bash
# Eclipse MAT (Memory Analyzer Tool)
mat heap.hprof

# JDK Mission Control
jmc

# Command line histogram from dump
jcmd PID GC.class_histogram
```

### OOM Handling
```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/java/
-XX:+ExitOnOutOfMemoryError        # Exit on OOM (recommended for containers)
-XX:OnOutOfMemoryError="kill -9 %p"  # Custom action
```

### Native Memory Tracking
```bash
# Enable (adds 5-10% overhead)
-XX:NativeMemoryTracking=summary   # or detail

# View
jcmd PID VM.native_memory summary
jcmd PID VM.native_memory detail
jcmd PID VM.native_memory baseline
jcmd PID VM.native_memory summary.diff  # Compare to baseline
```

**Native memory categories:**
- `Java Heap` - Heap memory
- `Class` - Loaded class metadata
- `Thread` - Thread stacks
- `Code` - JIT compiled code
- `GC` - GC data structures
- `Internal` - Command line parser, JVMTI
- `Symbol` - Symbols and symbol tables
- `Native Memory Tracking` - NMT overhead

### Direct Memory
```bash
-XX:MaxDirectMemorySize=1g         # Off-heap direct buffers
```

### Huge Pages
```bash
# Requires OS setup: echo 1024 > /proc/sys/vm/nr_hugepages
-XX:+UseLargePages
-XX:+UseTransparentHugePages       # Or explicit huge pages
-XX:LargePageSizeInBytes=2m
```

## JIT Tuning

```bash
# Print compilation
-XX:+PrintCompilation

# Tiered compilation (default)
-XX:+TieredCompilation

# Reserved code cache
-XX:ReservedCodeCacheSize=512m     # Increase for large apps
-XX:InitialCodeCacheSize=64m

# Print inlining decisions
-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining

# Compilation thresholds (rarely need changing)
-XX:CompileThreshold=10000         # Invocations before compile

# Graal JIT (alternative compiler)
-XX:+UnlockExperimentalVMOptions -XX:+UseJVMCICompiler
```

## JFR (Java Flight Recorder)

### Start Recording
```bash
# Via jcmd
jcmd PID JFR.start duration=60s filename=recording.jfr
jcmd PID JFR.start maxage=1h maxsize=100m disk=true name=continuous

# Via JVM args
-XX:StartFlightRecording=duration=60s,filename=out.jfr
-XX:StartFlightRecording=maxage=1h,maxsize=100m,disk=true,name=continuous
```

### Control Recording
```bash
jcmd PID JFR.check              # List recordings
jcmd PID JFR.stop name=NAME     # Stop recording
jcmd PID JFR.dump name=NAME filename=dump.jfr
```

### Analyze Recording
```bash
# JDK Mission Control
jmc

# Command line
jfr print recording.jfr
jfr print --events jdk.GCPhaseParallel recording.jfr
jfr print --events jdk.VirtualThreadPinned recording.jfr
jfr summary recording.jfr
jfr metadata recording.jfr      # Available event types
```

### JFR Streaming API (JDK 14+)

Continuous profiling with real-time event consumption.

```java
// In-process streaming
try (var rs = new RecordingStream()) {
    rs.enable("jdk.CPULoad").withPeriod(Duration.ofSeconds(1));
    rs.enable("jdk.GCPhasePause");
    rs.enable("jdk.VirtualThreadPinned").withStackTrace();

    rs.onEvent("jdk.CPULoad", event -> {
        double machineTotal = event.getFloat("machineTotal");
        if (machineTotal > 0.8) {
            alerting.notify("High CPU: " + machineTotal);
        }
    });

    rs.startAsync();  // Non-blocking
}

// Remote streaming (out-of-process)
try (var es = new EventStream(Path.of("/path/to/repository"))) {
    es.onEvent("jdk.GCPhasePause", System.out::println);
    es.start();
}
```

**Integration with observability tools:**
```java
// Export to Prometheus/OpenTelemetry
rs.onEvent("jdk.GCPhasePause", event -> {
    meterRegistry.timer("jvm.gc.pause")
        .record(event.getDuration());
});

rs.onEvent("jdk.ObjectAllocationOutsideTLAB", event -> {
    meterRegistry.counter("jvm.alloc.outside_tlab")
        .increment(event.getLong("allocationSize"));
});
```

## async-profiler

Low-overhead profiler. Version 3.x+ recommended for JDK 21+.

### Basic Usage
```bash
# CPU profiling (flame graph output)
./asprof -d 60 -f cpu.html PID

# Allocation profiling
./asprof -e alloc -d 60 -f alloc.html PID

# Wall clock (includes wait time - good for I/O-bound apps)
./asprof -e wall -d 60 -f wall.html PID

# Lock contention
./asprof -e lock -d 60 -f lock.html PID
```

### async-profiler 3.x+ Features

```bash
# Profile multiple events simultaneously (JFR output only)
./asprof -e cpu,alloc,lock -d 60 -f profile.jfr PID

# The --all flag: cpu, wall, alloc, live, lock, nativemem
./asprof --all -d 60 -f profile.jfr PID

# Native memory profiling (malloc/free tracking)
./asprof -e nativemem -d 60 -f native.jfr PID

# Filter long-running transactions
./jfrconv --latency 100ms profile.jfr output.html
```

### Java Agent Mode (3.x+)
```bash
# Load as agent at startup
java -javaagent:async-profiler.jar=start,event=cpu,alloc,file=profile.jfr MyApp

# Attach to running JVM
java -jar async-profiler.jar start -e cpu,alloc -f profile.jfr PID
```

### Native + Java Mixed Stacks
```bash
# Include native frames with DWARF unwinding
./asprof -d 60 --cstack dwarf -f out.html PID

# Frame pointer based (faster, less accurate)
./asprof -d 60 --cstack fp -f out.html PID
```

### JFR Converter
```bash
# Convert JFR to flame graph
./jfrconv profile.jfr profile.html

# Filter by method name
./jfrconv --include 'com/myapp/.*' profile.jfr filtered.html

# Allocation profile to pprof
./jfrconv --alloc profile.jfr profile.pprof

# Time range filtering
./jfrconv --from 10s --to 60s profile.jfr range.html
```

### Virtual Thread Profiling
```bash
# Wall clock profiling captures virtual thread activity
./asprof -e wall -t -d 60 -f vt.html PID

# Per-thread breakdown
./asprof -e wall -t -d 60 -o collapsed PID | ./flamegraph.pl --title "Virtual Threads" > vt.svg
```

## Foreign Function & Memory API (Panama)

FFM API (finalized JDK 22) replaces JNI with safer, faster native interop.

### Performance Characteristics

| Aspect | JNI | FFM API |
|--------|-----|---------|
| Development effort | High (header files, native code) | Low (pure Java) |
| Type safety | Manual | Compile-time checked |
| Memory safety | Manual | Arena-scoped lifetime |
| Call overhead | Baseline | Comparable (4-5x faster claimed, parity in practice) |
| JIT optimization | Limited | Full (method handle inlining) |

### Downcall Performance Options
```java
var linker = Linker.nativeLinker();

// Standard downcall - includes thread state transition
var handle = linker.downcallHandle(funcAddr,
    FunctionDescriptor.of(JAVA_LONG, ADDRESS, JAVA_INT));

// Critical downcall - no state transition, blocks GC
// Use for high-frequency calls to functions that never block
var criticalHandle = linker.downcallHandle(funcAddr, descriptor,
    Linker.Option.critical(true));  // true = allow heap access

// Capture errno after native call
var captureHandle = linker.downcallHandle(funcAddr, descriptor,
    Linker.Option.captureCallState("errno"));
```

### Memory Management
```java
// Arena-scoped allocation (automatic cleanup)
try (Arena arena = Arena.ofConfined()) {
    MemorySegment buffer = arena.allocate(1024);
    nativeCall(buffer);
}  // Memory freed here

// Global arena (manual or never freed)
Arena global = Arena.global();

// Shared arena (thread-safe)
try (Arena shared = Arena.ofShared()) {
    // Multiple threads can use this arena
}
```

### Profiling FFM
```bash
# Debug FFI code generation (debug build)
java -Xlog:foreign+downcall=debug ...

# Dump generated binding bytecode
-Djdk.internal.foreign.SPEC_DUMP=true
```

**Key insight:** The memory barrier (`lock add` on x86) after native calls is the main FFM overhead. Use `Linker.Option.critical()` for high-frequency calls to functions that never block.

## CRaC (Coordinated Restore at Checkpoint)

Checkpoint/restore for instant startup. Requires CRaC-enabled JDK (Azul Zulu, Liberica, Amazon Corretto).

### Performance Impact
| Metric | Cold Start | CRaC Restore |
|--------|------------|--------------|
| Spring Boot startup | 4-5 seconds | 40-50ms |
| Time to first request | seconds | milliseconds |
| Memory footprint | Growing | Pre-warmed |
| JIT compilation | From scratch | Already compiled |

### Basic Workflow
```bash
# 1. Start application with CRaC agent
java -XX:CRaCCheckpointTo=/checkpoint -jar app.jar

# 2. Warm up application (process requests, let JIT compile)

# 3. Trigger checkpoint
jcmd PID JDK.checkpoint

# 4. Restore from checkpoint
java -XX:CRaCRestoreFrom=/checkpoint -jar app.jar
```

### Application Requirements

CRaC requires explicit handling of external resources:

```java
public class MyResource implements Resource {
    private Socket socket;

    @Override
    public void beforeCheckpoint(Context<? extends Resource> context) {
        socket.close();  // Must close before checkpoint
    }

    @Override
    public void afterRestore(Context<? extends Resource> context) {
        socket = new Socket(host, port);  // Reopen after restore
    }
}

// Register with CRaC
Core.getGlobalContext().register(myResource);
```

**Framework support:** Spring Boot 3.2+, Micronaut, Quarkus have built-in CRaC support.

### Profiling CRaC Applications
```bash
# Profile cold start
./asprof -e cpu -d 30 -f cold-start.html PID

# Checkpoint, then profile restore
java -XX:CRaCRestoreFrom=/checkpoint -XX:StartFlightRecording=filename=restore.jfr -jar app.jar

# Compare JFR recordings
jfr print --events jdk.ExecutionSample cold.jfr > cold-methods.txt
jfr print --events jdk.ExecutionSample restore.jfr > restore-methods.txt
```

## GraalVM Native Image

AOT compilation to native binary. Different profiling approach than JVM.

### Build Options
```bash
# Basic build
native-image -jar app.jar

# With monitoring features
native-image --enable-monitoring=heapdump,jfr,jvmstat,nmt app

# Optimization levels
native-image -O2 app           # Default: balanced
native-image -O3 app           # Maximum optimization (longer build)

# With G1 GC (Oracle GraalVM only)
native-image --gc=G1 app
```

### Profile-Guided Optimization (PGO)
```bash
# 1. Build instrumented binary
native-image --pgo-instrument -jar app.jar -o app-instrumented

# 2. Run with representative workload
./app-instrumented
# ... exercise typical code paths ...
# Produces default.iprof

# 3. Build optimized binary using profile
native-image --pgo=default.iprof -jar app.jar -o app-optimized
```

**PGO results:** ~33% faster, ~15% smaller binaries vs non-PGO.

### ML-Based Static Profiling (JDK 24+)
```bash
# -O3 enables GraalNN neural network profiler
native-image -O3 -jar app.jar

# ~6% speedup without manual profile collection
```

### Native Memory Tracking (GraalVM 23+)
```bash
# Build with NMT support
native-image --enable-monitoring=nmt app

# View memory usage at runtime
./app -XX:+PrintNMTStatistics
```

### JFR with Native Image
```bash
# Build with JFR support
native-image --enable-monitoring=jfr app

# Record at runtime
./app -XX:StartFlightRecording=filename=native.jfr
```

### Linux perf with Native Image
```bash
# Build with debug info
native-image -g app

# Profile with perf
perf record -g ./app
perf report
```

### Profiling Comparison: JVM vs Native Image

| Aspect | JVM | Native Image |
|--------|-----|--------------|
| Startup time | Seconds | Milliseconds |
| Peak throughput | Higher (JIT adapts) | Fixed (AOT) |
| Memory footprint | Higher | Lower |
| Profiling tools | Full JFR, async-profiler | Limited JFR, perf |
| Warmup needed | Yes | No |
| Profile data | Runtime collected | Build-time PGO |

## jlink / Custom Runtimes

JDK 24+ enables jlink without jmods folder (JEP 493).

```bash
# Check linking capability
java -XshowSettings 2>&1 | grep "Linking from runtime image"

# Create minimal custom runtime
jlink --add-modules java.base,java.logging,jdk.jfr \
      --output custom-runtime

# With compression
jlink --add-modules java.base,java.logging \
      --compress=2 \
      --output minimal-runtime

# Verify modules
./custom-runtime/bin/java --list-modules
```

**Size reduction:** Custom runtimes can be 50%+ smaller than full JDK.

## Container-Specific Tuning

Modern JVMs (10+) detect container limits automatically.

```bash
# Verify container awareness
java -XshowSettings:system

# Memory configuration
-XX:+UseContainerSupport          # Default on (Java 10+)
-XX:MaxRAMPercentage=75.0         # Use % of container memory
-XX:InitialRAMPercentage=50.0
-XX:MinRAMPercentage=25.0         # For small heaps

# CPU limits
-XX:ActiveProcessorCount=4        # Override detected CPUs

# Recommended for containers
-XX:+ExitOnOutOfMemoryError       # Let orchestrator restart
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/            # Writable location
```

### Container GC Recommendations
```bash
# Small containers (<2GB): Serial or G1
-XX:+UseSerialGC                  # Minimal overhead

# Medium containers (2-8GB): G1
-XX:+UseG1GC -XX:MaxGCPauseMillis=100

# Large containers (>8GB) with latency requirements: ZGC
-XX:+UseZGC -XX:+ZGenerational
```

## Quick Reference

| Task | Command |
|------|---------|
| List Java processes | `jps -lv` |
| GC stats | `jstat -gcutil PID 1000` |
| Thread dump | `jstack -l PID` |
| Thread dump (JSON) | `jcmd PID Thread.dump_to_file -format=json threads.json` |
| Heap histogram | `jmap -histo PID` |
| Heap dump | `jmap -dump:format=b,file=heap.hprof PID` |
| JVM flags | `jcmd PID VM.flags` |
| Start JFR | `jcmd PID JFR.start duration=60s filename=out.jfr` |
| CPU profile | `./asprof -d 60 -f cpu.html PID` |
| Wall clock profile | `./asprof -e wall -d 60 -f wall.html PID` |
| Native memory | `jcmd PID VM.native_memory summary` |
| Trigger GC | `jcmd PID GC.run` |
| Virtual thread events | `jfr print --events jdk.VirtualThreadPinned recording.jfr` |
| Detect pinning | `java -Djdk.tracePinnedThreads=full MyApp` |

## JVM Flag Quick Reference (Java 21+)

```bash
# Essential flags for production
-Xms4g -Xmx4g                          # Heap (set equal for predictability)
-XX:+UseZGC -XX:+ZGenerational         # Low-latency GC (JDK 21-22)
-XX:+UseZGC                            # Low-latency GC (JDK 23+)
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/java/
-XX:+ExitOnOutOfMemoryError
-Xlog:gc*:file=gc.log:time,uptime:filecount=5,filesize=10m
-XX:StartFlightRecording=maxage=1h,maxsize=100m,disk=true

# Virtual threads (JDK 21-23 debugging)
-Djdk.tracePinnedThreads=short

# Performance diagnostics
-XX:NativeMemoryTracking=summary
-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining

# Container environment
-XX:+UseContainerSupport
-XX:MaxRAMPercentage=75.0
-XX:+ExitOnOutOfMemoryError
```
