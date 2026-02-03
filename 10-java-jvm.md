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

## Heap Analysis

### jmap
```bash
jmap -heap PID                 # Heap summary
jmap -histo PID                # Object histogram
jmap -histo:live PID           # Live objects only
jmap -dump:format=b,file=heap.hprof PID  # Heap dump
jmap -dump:live,format=b,file=heap.hprof PID  # Live only
```

### Heap Dump Analysis
```bash
# Using jhat (deprecated, use MAT instead)
jhat heap.hprof

# Eclipse MAT
mat heap.hprof

# VisualVM
jvisualvm
```

### OOM Heap Dump
```bash
# JVM options to dump on OOM
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/java/
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
jcmd PID JFR.check             # List recordings
jcmd PID JFR.stop name=NAME    # Stop recording
jcmd PID JFR.dump name=NAME filename=dump.jfr
```

### Analyze Recording
```bash
# JDK Mission Control
jmc

# Command line
jfr print recording.jfr
jfr print --events jdk.GCPhaseParallel recording.jfr
jfr summary recording.jfr
```

## async-profiler

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

## GC Tuning

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

### Heap Sizing
```bash
-Xms4g                         # Initial heap
-Xmx4g                         # Max heap
-Xmn2g                         # Young generation (with Parallel)
-XX:MetaspaceSize=256m         # Initial metaspace
-XX:MaxMetaspaceSize=512m      # Max metaspace
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

## Memory Tuning

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

## Useful JVM Options

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

## jvisualvm / JDK Mission Control

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
