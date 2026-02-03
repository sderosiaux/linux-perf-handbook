# Java/JVM

JVM monitoring, profiling, and tuning for Java 21+ (LTS).

## Emergency Triage: First 5 Minutes

When paged at 3am, run this decision tree:

```bash
# 1. Is the JVM even the problem?
top -p $(pgrep -f java) -H      # CPU? Which threads?
iostat -x 1 3                    # Disk I/O starving you?
ss -s                            # Socket exhaustion?
dmesg | tail -20                 # OOM killer? cgroup limits?

# 2. Is it OOM killed?
dmesg | grep -i "kill.*java\|oom"
cat /sys/fs/cgroup/memory.events 2>/dev/null | grep oom  # cgroup v2

# 3. Is it a deadlock?
jstack PID | grep -i "deadlock"

# 4. Is it a GC death spiral?
jstat -gcutil PID 1000 5
# RED FLAGS: FGC incrementing fast, OU >90% after FGC, GCT/uptime >10%

# 5. Is it stuck/hung?
jstack PID > /tmp/stack1.txt; sleep 10; jstack PID > /tmp/stack2.txt
diff /tmp/stack1.txt /tmp/stack2.txt  # Same stacks = stuck

# 6. Is it container CPU throttled?
cat /sys/fs/cgroup/cpu.stat 2>/dev/null | grep throttled
# nr_throttled > 0 = you're being throttled
```

### GC Death Spiral Detection

```bash
# Quick death spiral check
jstat -gcutil PID 1000 10 | awk '
  NR>1 && $4 > 90 && $8 > 10 {
    print "WARNING: Old gen " $4 "% full, GC time " $8 "%"
  }
  NR>1 && prev_fgc && $8 > prev_fgc {
    print "CRITICAL: Full GC rate increasing"
  }
  { prev_fgc = $8 }
'
```

**Death spiral indicators:**
- `FGC` count increasing faster than `YGC`
- `OU` (Old Used) consistently >90% after Full GC
- `GCT` / uptime > 10-20%
- Back-to-back Full GCs with no memory reclaimed

### Thread Dump Triage Patterns

```bash
# Find bottleneck: 100 threads BLOCKED on same monitor
jstack PID | grep -E "BLOCKED|waiting to lock" | sort | uniq -c | sort -rn

# Find what everyone is waiting for
jstack PID | grep -B5 "BLOCKED" | grep "waiting on"

# Connection pool starvation (common silent killer)
jstack PID | grep -A2 "pool-" | grep -c WAITING

# High CPU thread identification
top -H -p PID -n 1 | head -20  # Get thread ID (decimal)
printf '%x\n' THREAD_ID        # Convert to hex
jstack PID | grep -A 30 "nid=0xHEX"
```

### When jstack Won't Work

```bash
# JVM frozen? Force thread dump to stdout/stderr
kill -3 PID  # SIGQUIT - works when jstack hangs

# Hardcore: gdb when JVM is completely stuck
gdb -p PID -batch -ex "thread apply all bt"

# Container: enter namespace first
docker exec -it CONTAINER jcmd 1 Thread.print
nsenter -t $(docker inspect -f '{{.State.Pid}}' CONTAINER) -a jstack 1
```

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

### TLAB (Thread-Local Allocation Buffers)

TLABs allow threads to allocate objects without synchronization - critical for allocation-heavy workloads.

```bash
# TLAB configuration
-XX:TLABSize=512k                  # Initial TLAB size (auto-sized by default)
-XX:MinTLABSize=2k                 # Minimum TLAB size
-XX:TLABWasteTargetPercent=1       # Max wasted space in TLAB (default: 1%)
-XX:+ResizeTLAB                    # Dynamic TLAB sizing (default: on)

# Diagnostic
-Xlog:gc+tlab=debug                # TLAB statistics
```

**Why TLAB matters:**
- Allocation inside TLAB: pointer bump (fast)
- Allocation outside TLAB: requires synchronization (slow, ~10x slower)
- Large objects skip TLAB entirely

**JFR allocation events:**
```bash
# Inside TLAB (cheap)
jfr print --events jdk.ObjectAllocationInNewTLAB recording.jfr

# Outside TLAB (expensive - investigate these)
jfr print --events jdk.ObjectAllocationOutsideTLAB recording.jfr

# Sampled allocations (low overhead)
jfr print --events jdk.ObjectAllocationSample recording.jfr
```

**Allocation profiling with async-profiler:**
```bash
# Allocation flame graph
./asprof -e alloc -d 60 -f alloc.html PID

# Interpretation: wide bars = allocating lots of objects
# Focus on outside-TLAB allocations for optimization
```

## Safepoint Analysis

Safepoints are points where all application threads must stop for JVM operations (GC, deoptimization, etc.). Long time-to-safepoint (TTSP) causes latency spikes.

```bash
# Enable safepoint logging
-Xlog:safepoint:file=safepoint.log:time,uptime

# Log output example:
# [safepoint] Safepoint "G1CollectFull", Time since last: 1234 ms, Reaching safepoint: 5 ms
#                                                          ^^^^^^^^^^^^^^^^^^^^^^^^^^^
#                                                          This is TTSP - should be <1ms
```

**Safepoint triggers:**
- GC pauses
- Deoptimization
- Biased lock revocation (removed in JDK 15+)
- Thread dump
- JVMTI operations
- Class redefinition

### Diagnosing Safepoint Stalls

```bash
# JFR safepoint events
jfr print --events jdk.SafepointBegin,jdk.SafepointEnd recording.jfr

# Find slow TTSP (sync time)
jfr print --events jdk.SafepointBegin recording.jfr | grep -A5 "sync"
```

**Common TTSP causes:**
1. **Counted loops without safepoint polls** - Long loops delay safepoint
2. **Native code (JNI)** - No safepoint while in native
3. **Large array operations** - System.arraycopy, Arrays.fill
4. **NIO mapped buffers** - File I/O

```bash
# Safepoint interval tuning
-XX:GuaranteedSafepointInterval=0      # Disable periodic safepoints (default: 1000ms)

# For low-latency: enable safepoints in counted loops (slight CPU overhead)
-XX:+UseCountedLoopSafepoints          # Default: on in recent JDKs

# Debug: show which threads delay safepoint
-XX:+SafepointTimeout -XX:SafepointTimeoutDelay=1000
```

## Escape Analysis & Scalar Replacement

Objects that don't escape a method can be stack-allocated (eliminated).

```bash
# Enable escape analysis (default: on)
-XX:+DoEscapeAnalysis

# Diagnostic: see which allocations are eliminated
-XX:+UnlockDiagnosticVMOptions -XX:+PrintEscapeAnalysis

# See scalar replacement (object fields → local variables)
-XX:+EliminateAllocations              # Default: on
```

**Escape analysis requirements:**
- Object doesn't escape method (not stored in field, not passed out)
- Object is small enough
- Method is inlined (important!)

```java
// ESCAPES - heap allocated
void process() {
    User user = new User(name);
    cache.put(id, user);  // Escapes via field store
}

// DOESN'T ESCAPE - candidate for stack allocation
void calculate() {
    Point p = new Point(x, y);  // Only used locally
    return p.x + p.y;
}
```

**Verify with JITWatch**: TriView shows eliminated allocations.

## Lock Contention Analysis

Beyond basic `-e lock` profiling.

```bash
# async-profiler lock profiling
./asprof -e lock -d 60 -f lock.html PID

# JFR lock events
jfr print --events jdk.JavaMonitorWait,jdk.JavaMonitorEnter recording.jfr
```

**Lock optimization (JIT):**
```bash
# Lock coarsening: merge adjacent locks
-XX:+EliminateLocks                    # Default: on

# Verify lock elimination in JITWatch
# Look for "lock coarsening" or "lock elision" in compilation log
```

**Biased locking (historical):**
- JDK 1-14: Biased locking default on (optimize uncontended locks)
- JDK 15+: Deprecated and disabled
- JDK 18+: Removed entirely
- Impact: Legacy code with uncontended `synchronized` may be slightly slower

**Monitor inflation stages:**
1. Thin lock (stack-based, uncontended)
2. Fat lock (OS mutex, contended)

```bash
# Inflation events
-Xlog:monitorinflation=debug
```

## False Sharing & @Contended

False sharing: Different threads modify adjacent memory, causing cache line bouncing.

```java
// PROBLEM: x and y on same cache line (64 bytes)
class Counter {
    volatile long x;  // Thread A writes
    volatile long y;  // Thread B writes - invalidates A's cache line
}

// SOLUTION: Pad to separate cache lines
class Counter {
    @jdk.internal.vm.annotation.Contended
    volatile long x;

    @jdk.internal.vm.annotation.Contended
    volatile long y;
}
```

```bash
# Enable @Contended (required for non-JDK classes)
--add-opens java.base/jdk.internal.vm.annotation=ALL-UNNAMED
-XX:-RestrictContended
```

**Detection with JOL (Java Object Layout):**
```bash
# Add JOL dependency
# org.openjdk.jol:jol-core:0.17

# Print object layout
java -jar jol-cli.jar internals com.example.Counter
# Shows field offsets - fields within 64 bytes = potential false sharing
```

## GC-Friendly Coding Patterns

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

### Heap Dump Analysis Workflow

**Step 1: Get the dump**
```bash
# Live dump (forces GC first - preferred for leak analysis)
jmap -dump:live,format=b,file=heap.hprof PID

# Full dump (includes unreachable objects)
jmap -dump:format=b,file=heap.hprof PID

# JFR-based (works when jmap hangs)
jcmd PID GC.heap_dump /tmp/heap.hprof
```

**Step 2: Open in Eclipse MAT**
```bash
# Download: https://eclipse.dev/mat/downloads.php
# Increase MAT heap for large dumps:
# Edit MemoryAnalyzer.ini: -Xmx8g

mat heap.hprof
```

**Step 3: Analysis workflow**
1. **Leak Suspects Report** - Auto-generated, start here
2. **Dominator Tree** - Objects that retain the most memory
3. **Histogram** - Object count by class (sort by retained heap)
4. **Path to GC Roots** - Why object isn't collected
5. **OQL Queries** - Custom queries

**Key concepts:**
- **Shallow heap**: Memory consumed by object itself
- **Retained heap**: Memory freed if object is GC'd (includes dominated objects)
- **Dominator**: Object X dominates Y if every path from GC root to Y goes through X

**MAT OQL examples:**
```sql
-- Find large byte arrays
SELECT * FROM byte[] WHERE @retainedHeapSize > 1000000

-- Find duplicate strings
SELECT toString(s), count(s), sum(s.@retainedHeapSize)
FROM java.lang.String s GROUP BY toString(s) HAVING count(s) > 100

-- Find classloader leaks
SELECT * FROM java.lang.ClassLoader WHERE @retainedHeapSize > 10000000
```

### Memory Leak Hunting Methodology

**Pattern: Multiple dumps over time**
```bash
# Baseline
jmap -dump:live,format=b,file=heap1.hprof PID
# Wait (1 hour, or N requests)
jmap -dump:live,format=b,file=heap2.hprof PID
# Wait more
jmap -dump:live,format=b,file=heap3.hprof PID

# In MAT: Compare histograms
# Objects/retained growing over dumps = leak candidates
```

**Common leak patterns:**
1. **Collection leak** - Map/List growing unbounded (caches without eviction)
2. **Listener leak** - Registered callbacks never unregistered
3. **ThreadLocal leak** - ThreadLocal values surviving thread pool reuse
4. **Classloader leak** - Classes not unloaded (web app redeployment)
5. **Native memory leak** - DirectByteBuffer, JNI, off-heap

**Classloader leak detection:**
```bash
# Count loaded classes over time
jstat -class PID 1000 | awk '{print $1}'  # Should stabilize

# Find classloader retained size
jcmd PID GC.class_stats | head -50

# In MAT: Group by classloader
# Multiple instances of same webapp classes = leak
```

### Direct Memory / Off-Heap Leak Detection

When RSS >> (Heap + Metaspace), suspect off-heap memory.

```bash
# Check direct buffer pool
jcmd PID VM.native_memory summary | grep -A5 "Internal"

# Track over time
jcmd PID VM.native_memory baseline
# ... wait ...
jcmd PID VM.native_memory summary.diff
# Growing categories = leak

# Java-level direct buffer tracking
jcmd PID VM.info | grep "Direct Memory"
```

**Netty ByteBuf leak detection:**
```bash
# Enable Netty leak detection
-Dio.netty.leakDetection.level=paranoid
# Levels: disabled, simple, advanced, paranoid

# Check Netty metrics
curl http://localhost:8080/actuator/metrics/netty.allocator.direct.memory.used
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

## JIT Internals & Tuning

### Tiered Compilation Levels

| Level | Compiler | Description |
|-------|----------|-------------|
| 0 | Interpreter | No compilation |
| 1 | C1 | Simple compilation, no profiling |
| 2 | C1 | Limited profiling |
| 3 | C1 | Full profiling (default entry) |
| 4 | C2 | Optimized compilation (target) |

```bash
# Print compilation with levels
-XX:+PrintCompilation
# Output: timestamp compile_id attributes tiered_level method size

# C1 only mode (faster startup, lower peak)
-XX:TieredStopAtLevel=1

# Reserved code cache (segmented in JDK 9+)
-XX:ReservedCodeCacheSize=512m
-XX:NonProfiledCodeHeapSize=100m   # Non-profiled methods
-XX:ProfiledCodeHeapSize=300m      # Profiled methods (largest)
-XX:NonMethodCodeHeapSize=8m       # JVM internal code
```

### Warmup Verification

```bash
# Confirm JIT compilation is complete
-XX:+PrintCompilation 2>&1 | grep -c "made not entrant"
# Count should stabilize when warm

# Watch compilation queue drain
jcmd PID Compiler.queue
# Empty queue = compilation caught up

# Typical warmup pattern: run 10-50k requests before benchmarking
```

### Deoptimization

Deoptimization discards compiled code and returns to interpreter. Common causes:
- Type speculation failure (unexpected class)
- Null check elimination failure
- Array bounds speculation failure

```bash
# Track deoptimization events
-XX:+UnlockDiagnosticVMOptions -XX:+TraceDeoptimization

# JFR deoptimization events
jfr print --events jdk.Deoptimization recording.jfr
```

### JITWatch

Visualize JIT compilation decisions, inlining trees, and assembly.

```bash
# Generate compilation logs
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+LogCompilation \
     -XX:LogFile=compilation.log \
     -jar app.jar

# Download and run JITWatch
git clone https://github.com/AdoptOpenJDK/jitwatch
cd jitwatch && mvn clean install -DskipTests
./launchUI.sh

# Load compilation.log in JITWatch for:
# - Inlining tree visualization
# - TriView: source → bytecode → assembly
# - Method compilation timeline
```

**Common inlining failures:**
- "callee is too large" - method bytecode > 35 bytes (default)
- "too big" - cumulative inlined size limit
- "hot method too big" - C2 won't inline
- "call site not reached" - never executed path

```bash
# Increase inlining limits (use with caution)
-XX:MaxInlineSize=50              # Default: 35
-XX:FreqInlineSize=400            # Default: 325 (hot methods)
```

### PrintAssembly (hsdis)

View generated machine code. Requires hsdis library.

```bash
# Install hsdis (Debian/Ubuntu)
# Download from https://chriswhocodes.com/hsdis/ or build from OpenJDK

# Print assembly for specific method
-XX:+UnlockDiagnosticVMOptions \
-XX:+PrintAssembly \
-XX:PrintAssemblyOptions=intel \
-XX:CompileCommand=print,*MyClass.myMethod

# Or use JITWatch TriView for visual assembly inspection
```

### Basic JIT Flags

```bash
# Print compilation
-XX:+PrintCompilation

# Tiered compilation (default)
-XX:+TieredCompilation

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

## JMH (Java Microbenchmark Harness)

JMH is the standard for reliable Java microbenchmarks. Essential for performance work.

### Setup

```xml
<!-- Maven -->
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-core</artifactId>
    <version>1.37</version>
</dependency>
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-generator-annprocess</artifactId>
    <version>1.37</version>
</dependency>
```

### Basic Benchmark

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
@Fork(2)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 5, time = 1)
public class MyBenchmark {

    private List<Integer> list;

    @Setup
    public void setup() {
        list = new ArrayList<>();
        for (int i = 0; i < 1000; i++) list.add(i);
    }

    @Benchmark
    public int sumIterator() {
        int sum = 0;
        for (Integer i : list) sum += i;
        return sum;  // Return to prevent dead code elimination
    }

    @Benchmark
    public int sumStream() {
        return list.stream().mapToInt(i -> i).sum();
    }
}
```

### Running Benchmarks

```bash
# Build and run
mvn clean package
java -jar target/benchmarks.jar

# With specific benchmarks
java -jar target/benchmarks.jar ".*sumIterator.*"

# With profilers
java -jar target/benchmarks.jar -prof gc       # GC stats
java -jar target/benchmarks.jar -prof async    # async-profiler flame graph
java -jar target/benchmarks.jar -prof perfnorm # Normalized perf counters
```

### Common Pitfalls

```java
// WRONG: JIT eliminates this (dead code)
@Benchmark
public void wrong() {
    Math.sqrt(x);  // Result unused - optimized away
}

// RIGHT: Return result or use Blackhole
@Benchmark
public double right() {
    return Math.sqrt(x);
}

@Benchmark
public void rightBlackhole(Blackhole bh) {
    bh.consume(Math.sqrt(x));
}

// WRONG: Constant folding
private final double x = 100;

@Benchmark
public double wrong() {
    return Math.sqrt(100);  // Computed at compile time
}

// RIGHT: @State prevents constant folding
@State(Scope.Benchmark)
public class MyBenchmark {
    double x = 100;  // Not final
}
```

**JMH benchmark modes:**
- `Throughput` - ops/time
- `AverageTime` - avg time/op
- `SampleTime` - time distribution
- `SingleShotTime` - cold start time

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

### Container CPU Throttling

**The invisible latency killer.** Your container gets throttled but JVM doesn't know.

```bash
# Check if you're being throttled (inside container)
cat /sys/fs/cgroup/cpu.stat
# nr_periods 12345
# nr_throttled 1234       # > 0 = you're being throttled!
# throttled_time 567890   # Nanoseconds spent throttled

# Calculate throttle percentage
awk '/nr_periods/ {p=$2} /nr_throttled/ {t=$2} END {print t/p*100 "%"}' /sys/fs/cgroup/cpu.stat
```

**Symptoms:** Latency spikes, p99 degradation, inconsistent response times despite low CPU in metrics.

**Root cause:** cgroup CPU quota exhausted before period ends.

```bash
# Check quota settings
cat /sys/fs/cgroup/cpu.max  # "100000 100000" = 1 CPU, quota period
# First number: quota (microseconds)
# Second: period (microseconds)
# Ratio = effective CPUs

# Kubernetes: requests vs limits mismatch
# limits.cpu: 1000m = 100ms quota per 100ms period
# If burst exceeds quota, throttling occurs
```

**Mitigations:**
1. Increase CPU limits (more quota)
2. Set requests = limits (guaranteed QoS)
3. Use `ActiveProcessorCount` to align JVM thread pools
4. Reduce parallelism (fewer threads contending for quota)

```bash
# Align JVM to container CPU limits
-XX:ActiveProcessorCount=2        # Match your CPU limit
# Prevents JVM from creating too many GC/compiler threads
```

### Container Memory: RSS vs Heap

**OOMKilled vs Java OOM - different problems:**

| Symptom | Cause | Fix |
|---------|-------|-----|
| Java `OutOfMemoryError` | Heap exhausted | Increase `-Xmx` |
| Container OOMKilled (137) | RSS > memory limit | Reduce heap, check native memory |

```bash
# What's using memory? (inside container)
cat /proc/1/smaps_rollup
# Or with NMT:
jcmd 1 VM.native_memory summary

# Common RSS contributors beyond heap:
# - Metaspace (class metadata)
# - Thread stacks (1MB default × threads)
# - Code cache (JIT compiled code)
# - Direct buffers (NIO, Netty)
# - Native libraries (JNI)
# - GC overhead
```

**Rule of thumb:** `memory.limit` should be `Xmx + 25-50%` for non-heap memory.

```bash
# Example for 4GB limit:
-Xmx3g                            # Leave ~1GB for non-heap
-XX:MaxMetaspaceSize=256m         # Bound metaspace
-XX:ReservedCodeCacheSize=128m    # Bound code cache
-XX:MaxDirectMemorySize=256m      # Bound direct buffers
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
