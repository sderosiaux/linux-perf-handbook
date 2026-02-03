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
