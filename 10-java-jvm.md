# Java/JVM

JVM monitoring, profiling, and tuning.

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

**GC columns:**
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
jinfo -flag +PrintGC PID       # Enable flag (if manageable)
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
jcmd PID Compiler.queue        # JIT queue
```

## Thread Analysis

### jstack
```bash
jstack PID                     # Thread dump
jstack -l PID                  # With locks
jstack -F PID                  # Force (hung JVM)
jstack -m PID                  # Mixed mode (native)
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

## Garbage Collection

### GC Selection
```bash
# G1GC (default Java 9+)
-XX:+UseG1GC

# ZGC (low latency, Java 15+)
-XX:+UseZGC

# Shenandoah (low latency)
-XX:+UseShenandoahGC

# Serial GC (single-threaded)
-XX:+UseSerialGC

# Parallel GC (throughput)
-XX:+UseParallelGC
```

### G1GC Tuning
```bash
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200       # Target pause time
-XX:G1HeapRegionSize=16m       # Region size (1-32MB)
-XX:InitiatingHeapOccupancyPercent=45  # Start mixed GC
-XX:G1ReservePercent=10        # Reserve for promotion
-XX:G1NewSizePercent=5         # Min young gen %
-XX:G1MaxNewSizePercent=60     # Max young gen %
```

### ZGC Tuning
```bash
-XX:+UseZGC
-XX:+ZGenerational             # Generational ZGC (Java 21+)
-XX:SoftMaxHeapSize=8g         # Soft limit
-XX:ZCollectionInterval=5      # GC interval (seconds)
```

### GC Logging
```bash
# Java 9+ Unified Logging
-Xlog:gc*:file=gc.log:time,uptime:filecount=5,filesize=10m

# Verbose GC logging
-Xlog:gc*=debug:file=gc.log

# Specific logging
-Xlog:gc+heap=debug:file=heap.log
-Xlog:safepoint:file=safepoint.log
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

Data-oriented design reduces per-object overhead. JVM object headers are 12-16 bytes with compressed oops.

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

**Key insight:** A boolean in an object costs 4-8 bytes (alignment) but only 1 bit of information - struct-of-arrays with BitSet achieves 64x better density for boolean flags.

### Object Pooling

Object pools reduce allocation pressure between GC cycles.

```java
// ThreadLocal pool pattern
private static final ThreadLocal<byte[]> BUFFER_POOL =
    ThreadLocal.withInitial(() -> new byte[8192]);

void process() {
    byte[] buf = BUFFER_POOL.get();
    try {
        // use buffer
    } finally {
        Arrays.fill(buf, (byte) 0);  // Reset for reuse
    }
}

// For object pools, consider:
// - Caffeine cache with size=0, expireAfterAccess (soft references)
// - Apache Commons Pool2
// - Custom ring buffer for fixed-size allocations
```

**Key insight:** Pool effectiveness depends on allocation frequency vs GC frequency - Java pools with soft references survive GC cycles but add lookup overhead.

## Memory Management

### Heap Sizing
```bash
-Xms4g                         # Initial heap
-Xmx4g                         # Max heap
-Xmn2g                         # Young generation (with Parallel)
-XX:MetaspaceSize=256m         # Initial metaspace
-XX:MaxMetaspaceSize=512m      # Max metaspace
```

### Heap Analysis with jmap
```bash
jmap -heap PID                 # Heap summary
jmap -histo PID                # Object histogram
jmap -histo:live PID           # Live objects only
jmap -dump:format=b,file=heap.hprof PID  # Heap dump
jmap -dump:live,format=b,file=heap.hprof PID  # Live only
```

### Heap Dump Analysis
```bash
# Eclipse MAT
mat heap.hprof

# VisualVM
jvisualvm
```

### OOM Heap Dump
```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/java/
```

### Native Memory Tracking
```bash
# Enable
-XX:NativeMemoryTracking=summary  # or detail

# View
jcmd PID VM.native_memory summary
jcmd PID VM.native_memory detail
jcmd PID VM.native_memory baseline
jcmd PID VM.native_memory summary.diff
```

### Class Metadata & Memory

Project Lilliput shrinks narrow class ID to 22 bits (from 32), requiring 1KB alignment for class structures. This causes cache hyper-aliasing - different classes collide in L1 cache. The CLUT (Class Look-Up Table) solution pre-computes iteration metadata into 4-byte tokens.

```bash
# Class limits by narrow class ID size:
# Stock JVM (32-bit):  ~5-6 million classes
# Lilliput (22-bit):   ~4 million classes

# Monitor class metadata with Native Memory Tracking
jcmd PID VM.native_memory summary | grep -A5 "Class"
```

**Key insight:** 96-99% of objects in typical heaps have instance sizes <512 bytes and <3 oop map entries - the CLUT token optimization covers these cases without additional memory loads during GC iteration.

### Direct Memory
```bash
-XX:MaxDirectMemorySize=1g     # Off-heap direct buffers
```

### Huge Pages
```bash
# Requires OS setup
-XX:+UseLargePages
-XX:+UseTransparentHugePages   # Or explicit huge pages
-XX:LargePageSizeInBytes=2m
```

## JVM Sizing and Flags

### Performance
```bash
-server                        # Server VM
-XX:+UseStringDeduplication    # String dedup (G1)
-XX:+OptimizeStringConcat      # String concat optimization
-XX:+UseCompressedOops         # Compressed pointers (< 32GB heap)
-XX:+UseCompressedClassPointers
```

### Debugging
```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/java/
-XX:ErrorFile=/var/log/java/hs_err_pid%p.log
-XX:+ExitOnOutOfMemoryError    # Or CrashOnOutOfMemoryError
```

### Diagnostics
```bash
-XX:+PrintFlagsFinal           # All flag values
-XX:+PrintCommandLineFlags     # Non-default flags
-XshowSettings:all             # All settings
```

## JIT Tuning

```bash
# Print compilation
-XX:+PrintCompilation

# Compiler thresholds
-XX:CompileThreshold=10000     # Invocations before compile

# Tiered compilation (default)
-XX:+TieredCompilation

# Reserved code cache
-XX:ReservedCodeCacheSize=256m
-XX:InitialCodeCacheSize=64m

# Print inlining
-XX:+PrintInlining
```

## jlink / Custom Runtimes

JDK 24+ enables running jlink without the jmods folder via JEP 493. This reduces JDK archive size by ~35% and extracted size by ~15%.

```bash
# Check if linking from runtime image is enabled
java -XshowSettings 2>&1 | grep "Linking from runtime image"

# Create custom runtime with only needed modules
jlink --add-modules java.logging,java.naming,jdk.unsupported,jdk.jfr,jdk.jdwp.agent \
      --output runtime-for-app

# Verify modules in custom runtime
./runtime-for-app/bin/java --list-modules
```

**Key insight:** Custom runtimes can reduce bundle size by 50% compared to shipping a full JDK/JRE - the `lib/modules` file shrinks from ~104MB to ~33MB when including only required modules.

## Foreign Function & Memory API (Panama)

The FFI API generates platform-specific code for native calls. Critical functions (no thread state transitions) provide lowest latency but block GC.

```java
// Standard downcall - includes thread state transition + memory barrier
var handle = linker.downcallHandle(funcAddr, FunctionDescriptor.of(JAVA_LONG, ADDRESS, JAVA_INT));

// Critical downcall - no state transition, faster but blocks GC
var criticalHandle = linker.downcallHandle(funcAddr, descriptor,
    Linker.Option.critical(true));  // true = allow heap access

// Capture errno after native call
var captureHandle = linker.downcallHandle(funcAddr, descriptor,
    Linker.Option.captureCallState("errno"));
```

```bash
# Debug FFI code generation (debug build required)
java -Xlog:foreign+downcall=debug ...

# Dump generated binding bytecode
-Djdk.internal.foreign.SPEC_DUMP=true
```

**Key insight:** The lock-add instruction (memory barrier) on x86 after native calls is the main FFI overhead - use `Linker.Option.critical()` for high-frequency calls to functions that never block.

### Swift-Java Interop via FFM

Swift's value types require special handling - size determined at runtime via type metadata. The `jextract-swift` tool generates Java wrappers that handle Swift calling conventions.

```java
// Swift type metadata provides runtime layout info
// Value Witness Table contains: size, stride, alignment, destroy/copy functions
MemoryLayout swiftLayout = SwiftRuntime.getLayout(swiftTypeMetadata);

// Swift arena manages reference counting for Swift objects
try (var arena = SwiftArena.ofConfined()) {
    var swiftStruct = MySwiftStruct.init(arena, length, capacity);
    // Arena calls destroy on all registered Swift values at close
}
```

**Key insight:** Swift value types can be stack-allocated but require calling into Swift runtime for size/layout - type metadata is immortal and can be cached on Java side.

## Profiling

### JFR (Java Flight Recorder)

#### Start Recording
```bash
# Via jcmd
jcmd PID JFR.start duration=60s filename=recording.jfr
jcmd PID JFR.start maxage=1h maxsize=100m disk=true name=continuous

# Via JVM args
-XX:StartFlightRecording=duration=60s,filename=out.jfr
-XX:StartFlightRecording=maxage=1h,maxsize=100m,disk=true,name=continuous
```

#### Control Recording
```bash
jcmd PID JFR.check             # List recordings
jcmd PID JFR.stop name=NAME    # Stop recording
jcmd PID JFR.dump name=NAME filename=dump.jfr
```

#### Analyze Recording
```bash
# JDK Mission Control
jmc

# Command line
jfr print recording.jfr
jfr print --events jdk.GCPhaseParallel recording.jfr
jfr summary recording.jfr
```

### async-profiler

Low-overhead profiler, better than JFR for CPU profiling.

```bash
# CPU profiling
./asprof -d 60 -f cpu.html PID

# Allocation profiling
./asprof -e alloc -d 60 -f alloc.html PID

# Wall clock (includes wait time)
./asprof -e wall -d 60 -f wall.html PID

# Lock profiling
./asprof -e lock -d 60 -f lock.html PID

# Specific thread
./asprof -t -d 60 -f out.html PID

# Native + Java frames
./asprof -d 60 --cstack dwarf -f out.html PID

# JFR output
./asprof -d 60 -f out.jfr PID
```

### jvisualvm / JDK Mission Control

```bash
# Launch VisualVM
jvisualvm

# Launch JMC
jmc
```

Features:
- CPU/memory profiling
- Thread analysis
- GC visualization
- MBean browser
- JFR integration

### Escape Analysis Visibility
```bash
# JIT inlining decisions visible in compilation logs
-XX:+PrintCompilation -XX:+PrintInlining
```

## Container-Specific Tuning

Containers require awareness of cgroup limits. Modern JVMs (10+) detect container limits automatically.

```bash
# Verify container awareness
java -XshowSettings:system

# Override container detection
-XX:+UseContainerSupport        # Default on (Java 10+)
-XX:MaxRAMPercentage=75.0       # Use % of container memory
-XX:InitialRAMPercentage=50.0
-XX:MinRAMPercentage=25.0       # For small heaps

# CPU limits
-XX:ActiveProcessorCount=4      # Override detected CPUs
```

## Quick Reference

| Task | Command |
|------|---------|
| List Java processes | `jps -lv` |
| GC stats | `jstat -gcutil PID 1000` |
| Thread dump | `jstack -l PID` |
| Heap histogram | `jmap -histo PID` |
| Heap dump | `jmap -dump:format=b,file=heap.hprof PID` |
| JVM flags | `jcmd PID VM.flags` |
| Start JFR | `jcmd PID JFR.start duration=60s filename=out.jfr` |
| CPU profile | `./asprof -d 60 -f cpu.html PID` |
| Native memory | `jcmd PID VM.native_memory summary` |
| Trigger GC | `jcmd PID GC.run` |
