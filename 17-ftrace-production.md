# ftrace for Production Debugging

Kernel-level function and event tracing without external dependencies.

## When ftrace Over eBPF

### Decision Matrix

| Scenario | ftrace | bpftrace | perf |
|----------|--------|----------|------|
| Kernel < 4.4 | Yes | No | Yes |
| Locked-down kernel (no BPF) | Yes | No | Limited |
| Function call graphs | Best | Limited | No |
| Quick ad-hoc tracing | Good | Best | Good |
| Programmable aggregation | No | Yes | No |
| In-kernel histograms | Yes (4.7+) | Yes | No |
| Production overhead | Lowest | Low | Low |
| Trace specific function duration | Yes | Yes | Limited |
| Userspace tracing | No | Yes | Yes |

### Use ftrace When

- **Older kernels**: Works on kernels without eBPF support (pre-4.4)
- **Restricted environments**: BPF disabled via lockdown, secureboot, or policy
- **Function flow analysis**: `function_graph` tracer shows call hierarchy
- **Zero-dependency tracing**: Built into kernel, no tools to install
- **Lower overhead**: No BPF JIT compilation, minimal setup cost
- **Quick sanity checks**: Faster to enable than writing bpftrace scripts

### Use eBPF/bpftrace When

- **Programmable aggregation**: Calculate latency histograms in-kernel
- **Complex filtering**: Combine multiple conditions with logic
- **Userspace + kernel**: Trace across user/kernel boundary
- **Custom data structures**: Store state between events
- **Production monitoring**: Long-running aggregated metrics

### Use perf When

- **CPU profiling**: Statistical sampling with low overhead
- **Hardware counters**: Cache misses, branch mispredictions
- **Flame graph generation**: `perf script` output for FlameGraph
- **System-wide sampling**: Profile all processes simultaneously

## ftrace Architecture

### Directory Structure

```bash
# Modern kernels (4.1+)
/sys/kernel/tracing/

# Older kernels (via debugfs)
/sys/kernel/debug/tracing/

# Check which is available
ls /sys/kernel/tracing 2>/dev/null || ls /sys/kernel/debug/tracing
```

Key files:
```
/sys/kernel/tracing/
├── available_tracers      # List of compiled-in tracers
├── current_tracer         # Active tracer (function, function_graph, nop)
├── tracing_on             # 1=enabled, 0=disabled (quick toggle)
├── trace                  # Read trace buffer (non-consuming)
├── trace_pipe             # Consuming read (blocks for new events)
├── trace_options          # Tracer options
├── buffer_size_kb         # Per-CPU buffer size
├── buffer_total_size_kb   # Total buffer size
├── set_ftrace_filter      # Functions to trace (whitelist)
├── set_ftrace_notrace     # Functions to exclude (blacklist)
├── set_ftrace_pid         # PIDs to trace
├── available_filter_functions  # Functions that can be traced
├── available_events       # Static tracepoints
├── events/                # Event controls by subsystem
│   ├── sched/             # Scheduler events
│   ├── block/             # Block I/O events
│   ├── net/               # Network events
│   └── ...
├── kprobe_events          # Dynamic kprobe definitions
├── uprobe_events          # Dynamic uprobe definitions
├── instances/             # Parallel trace instances
└── per_cpu/               # Per-CPU buffers and controls
    ├── cpu0/
    │   ├── trace
    │   ├── trace_pipe
    │   ├── trace_pipe_raw
    │   └── buffer_size_kb
    └── ...
```

### Tracers

```bash
# List available tracers
cat /sys/kernel/tracing/available_tracers
# Output: hwlat blk function_graph wakeup_dl wakeup_rt wakeup function nop

# Current tracer
cat /sys/kernel/tracing/current_tracer
```

| Tracer | Purpose | Overhead |
|--------|---------|----------|
| `nop` | Disabled (events only) | None |
| `function` | Trace function entries | Medium |
| `function_graph` | Entry + exit with duration | Higher |
| `wakeup` | Wakeup latency (max) | Low |
| `wakeup_rt` | RT task wakeup latency | Low |
| `irqsoff` | IRQ disable latency | Low |
| `preemptoff` | Preemption disable latency | Low |
| `preemptirqsoff` | Combined irq+preempt latency | Low |
| `hwlat` | Hardware latency detector | Minimal |

### Ring Buffer Mechanics

- **Per-CPU buffers**: Each CPU has independent buffer (no lock contention)
- **Atomic writes**: Events written without global locks
- **Overwrite mode**: Default wraps, oldest events lost
- **Producer/consumer**: `trace_pipe` consumes, `trace` does not
- **Splicing**: `trace_pipe_raw` for efficient binary transfer

```bash
# Check buffer size
cat /sys/kernel/tracing/buffer_size_kb
# Default: 1408 KB per CPU

# Set buffer size (per-CPU)
echo 4096 > /sys/kernel/tracing/buffer_size_kb

# Set specific CPU buffer
echo 8192 > /sys/kernel/tracing/per_cpu/cpu0/buffer_size_kb

# Check total buffer usage
cat /sys/kernel/tracing/buffer_total_size_kb
```

### Trace Instances (Parallel Tracing)

Create isolated trace buffers for different use cases:

```bash
# Create instance
mkdir /sys/kernel/tracing/instances/my_trace

# Instance has full directory structure
ls /sys/kernel/tracing/instances/my_trace/
# trace, trace_pipe, events/, set_ftrace_filter, etc.

# Configure independently
echo function > /sys/kernel/tracing/instances/my_trace/current_tracer
echo 'tcp_*' > /sys/kernel/tracing/instances/my_trace/set_ftrace_filter

# Read from instance
cat /sys/kernel/tracing/instances/my_trace/trace

# Remove instance
rmdir /sys/kernel/tracing/instances/my_trace
```

Use cases:
- Different subsystems traced separately
- Team member traces don't interfere
- Production monitoring + ad-hoc debugging
- Different buffer sizes for different trace types

## Practical ftrace Patterns

### Quick Enable/Disable

```bash
# Variables for convenience
T=/sys/kernel/tracing

# Quick toggle (preserves config)
echo 0 > $T/tracing_on   # Stop
echo 1 > $T/tracing_on   # Start

# Clear buffer
echo > $T/trace

# Full reset to defaults
echo nop > $T/current_tracer
echo > $T/set_ftrace_filter
echo > $T/set_ftrace_notrace
echo > $T/set_ftrace_pid
echo > $T/trace
```

### Trace Specific Functions

```bash
T=/sys/kernel/tracing

# Trace single function
echo 0 > $T/tracing_on
echo function > $T/current_tracer
echo do_sys_openat2 > $T/set_ftrace_filter
echo 1 > $T/tracing_on

# Run workload, then check
cat $T/trace

# Trace multiple functions
echo 'do_sys_openat2
vfs_open
do_filp_open' > $T/set_ftrace_filter

# Wildcard patterns
echo 'tcp_*' > $T/set_ftrace_filter     # All tcp_ functions
echo '*_read' >> $T/set_ftrace_filter   # Append functions ending in _read
echo '*lock*' >> $T/set_ftrace_filter   # Functions containing "lock"

# Exclude functions
echo 'tcp_poll' > $T/set_ftrace_notrace
echo 'tcp_v4_rcv' >> $T/set_ftrace_notrace

# Check what's filtered
cat $T/set_ftrace_filter
```

### Function Graph Tracer (Call Flow)

```bash
T=/sys/kernel/tracing

# Enable function graph
echo function_graph > $T/current_tracer

# Trace specific function and its callees
echo do_sys_openat2 > $T/set_graph_function

# Set max depth (reduce noise)
echo 5 > $T/max_graph_depth

# Trace
echo 1 > $T/tracing_on
# ... run workload ...
echo 0 > $T/tracing_on

cat $T/trace
```

Sample output:
```
# tracer: function_graph
#
# CPU  DURATION                  FUNCTION CALLS
# |     |   |                     |   |   |   |
 0)               |  do_sys_openat2() {
 0)               |    getname_flags() {
 0)               |      kmem_cache_alloc() {
 0)   0.318 us    |        should_failslab();
 0)   1.237 us    |      }
 0)               |      strncpy_from_user() {
 0)   0.284 us    |        _cond_resched();
 0)   0.963 us    |      }
 0)   3.156 us    |    }
 0)               |    do_filp_open() {
 0)   ...         |      ...
 0)   45.234 us   |    }
 0)   52.891 us   |  }
```

### Trace by PID

```bash
T=/sys/kernel/tracing

# Trace specific PID
echo $$ > $T/set_ftrace_pid

# Trace multiple PIDs
echo "1234
5678
9012" > $T/set_ftrace_pid

# Enable tracing
echo function > $T/current_tracer
echo 1 > $T/tracing_on

# Check current PIDs
cat $T/set_ftrace_pid

# Clear PID filter (trace all)
echo > $T/set_ftrace_pid
```

### Measure Function Duration

```bash
T=/sys/kernel/tracing

# Use function_graph with filter to measure specific function
echo 0 > $T/tracing_on
echo function_graph > $T/current_tracer
echo vfs_read > $T/set_graph_function
echo 1 > $T/max_graph_depth  # Only the target function

# Enable
echo 1 > $T/tracing_on
# ... workload ...
echo 0 > $T/tracing_on

# Parse durations
grep 'vfs_read' $T/trace | awk '{print $2}'
```

### Find Who Calls a Slow Function

```bash
T=/sys/kernel/tracing

# Enable stack traces on function entry
echo 0 > $T/tracing_on
echo function > $T/current_tracer
echo mutex_lock > $T/set_ftrace_filter

# Enable stack trace option
echo 1 > $T/options/func_stack_trace

echo 1 > $T/tracing_on
# ... reproduce issue ...
echo 0 > $T/tracing_on

# View call stacks
cat $T/trace
```

Sample output:
```
 bash-1234  [001] ....  1234.567890: mutex_lock <-vfs_write
 bash-1234  [001] ....  1234.567890: <stack trace>
 => mutex_lock
 => vfs_write
 => ksys_write
 => do_syscall_64
 => entry_SYSCALL_64_after_hwframe
```

### Duration Threshold Tracing

Capture only slow operations:

```bash
T=/sys/kernel/tracing

# Set threshold in microseconds
echo 200 > $T/tracing_thresh  # 200us

# Use irqsoff/preemptoff tracers with threshold
echo preemptirqsoff > $T/current_tracer
echo 1 > $T/tracing_on

# Or with function_graph (shows functions exceeding threshold)
echo function_graph > $T/current_tracer
echo 1 > $T/tracing_on

# Check for slow functions (marked with + or !)
# + = exceeded threshold
# ! = exceeded threshold and interrupted
cat $T/trace | grep -E '^\s*[0-9]+\)\s+[+!]'
```

## Event Tracing

### Discover Available Events

```bash
T=/sys/kernel/tracing

# List all events
cat $T/available_events | wc -l  # Typically 1000+

# List by subsystem
ls $T/events/
# block  compaction  ext4  irq  kmem  net  power  sched  signal  syscalls  ...

# List sched events
ls $T/events/sched/
# sched_switch  sched_wakeup  sched_migrate_task  sched_process_fork  ...

# Event format (useful for filtering)
cat $T/events/sched/sched_switch/format
```

Sample format output:
```
name: sched_switch
ID: 316
format:
    field:unsigned short common_type;       offset:0;  size:2; signed:0;
    field:unsigned char common_flags;       offset:2;  size:1; signed:0;
    field:unsigned char common_preempt_count; offset:3; size:1; signed:0;
    field:int common_pid;                   offset:4;  size:4; signed:1;

    field:char prev_comm[16];               offset:8;  size:16; signed:0;
    field:pid_t prev_pid;                   offset:24; size:4;  signed:1;
    field:int prev_prio;                    offset:28; size:4;  signed:1;
    field:long prev_state;                  offset:32; size:8;  signed:1;
    field:char next_comm[16];               offset:40; size:16; signed:0;
    field:pid_t next_pid;                   offset:56; size:4;  signed:1;
    field:int next_prio;                    offset:60; size:4;  signed:1;

print fmt: "prev_comm=%s prev_pid=%d prev_prio=%d prev_state=%s%s ==> ..."
```

### Enable/Disable Events

```bash
T=/sys/kernel/tracing

# Single event
echo 1 > $T/events/sched/sched_switch/enable

# All events in subsystem
echo 1 > $T/events/sched/enable

# All events (careful - high overhead)
echo 1 > $T/events/enable

# Disable
echo 0 > $T/events/sched/sched_switch/enable

# Check enabled events
cat $T/set_event
```

### Event Filtering

```bash
T=/sys/kernel/tracing

# Filter by field value
echo 'prev_pid == 1234' > $T/events/sched/sched_switch/filter
echo 'bytes_req > 1024' > $T/events/kmem/kmalloc/filter

# Compound filters
echo 'prev_pid == 1234 || next_pid == 1234' > $T/events/sched/sched_switch/filter
echo 'bytes_req > 1024 && call_site != 0xffffffff12345678' > $T/events/kmem/kmalloc/filter

# String matching (for comm fields)
echo 'prev_comm ~ "*nginx*"' > $T/events/sched/sched_switch/filter

# Clear filter
echo 0 > $T/events/sched/sched_switch/filter

# Check filter
cat $T/events/sched/sched_switch/filter
```

### Event Triggers

```bash
T=/sys/kernel/tracing

# Stacktrace when event fires
echo 'stacktrace' > $T/events/block/block_rq_issue/trigger

# Stacktrace with filter
echo 'stacktrace if nr_sector > 256' > $T/events/block/block_rq_issue/trigger

# Enable another event when this fires
echo 'enable_event:kmem:kmalloc' > $T/events/syscalls/sys_enter_read/trigger

# Disable event chain
echo 'disable_event:kmem:kmalloc' > $T/events/syscalls/sys_exit_read/trigger

# Snapshot on event (save current buffer)
echo 'snapshot' > $T/events/sched/sched_switch/trigger
echo 'snapshot if prev_comm == "nginx"' > $T/events/sched/sched_switch/trigger

# Turn off tracing when event fires (catch rare issue)
echo 'traceoff' > $T/events/signal/signal_generate/trigger
echo 'traceoff if sig == 9' > $T/events/signal/signal_generate/trigger

# View triggers
cat $T/events/block/block_rq_issue/trigger

# Remove trigger
echo '!stacktrace' > $T/events/block/block_rq_issue/trigger
```

### Histogram Triggers (4.7+)

In-kernel aggregation without dumping every event:

```bash
T=/sys/kernel/tracing

# Count by key
echo 'hist:keys=call_site' > $T/events/kmem/kmalloc/trigger

# Count with value sum
echo 'hist:keys=call_site:vals=bytes_req' > $T/events/kmem/kmalloc/trigger

# Multiple keys
echo 'hist:keys=common_pid,call_site:vals=bytes_req' > $T/events/kmem/kmalloc/trigger

# Sort by value
echo 'hist:keys=call_site:vals=bytes_req:sort=bytes_req.desc' > $T/events/kmem/kmalloc/trigger

# Read histogram
cat $T/events/kmem/kmalloc/hist

# Clear histogram
echo 'hist:keys=call_site:clear' >> $T/events/kmem/kmalloc/trigger

# Pause histogram
echo 'hist:keys=call_site:pause' >> $T/events/kmem/kmalloc/trigger

# Continue histogram
echo 'hist:keys=call_site:continue' >> $T/events/kmem/kmalloc/trigger
```

Sample histogram output:
```
# event histogram
#
# trigger info: hist:keys=call_site:vals=bytes_req:sort=bytes_req.desc
#

{ call_site: 0xffffffff81234567 } hitcount:      12345  bytes_req:    5678901
{ call_site: 0xffffffff81234568 } hitcount:       5678  bytes_req:    1234567
{ call_site: 0xffffffff81234569 } hitcount:       1234  bytes_req:     456789
...

Totals:
    Hits: 20000
    Entries: 128
```

### Latency Histograms with Synthetic Events

```bash
T=/sys/kernel/tracing

# Create synthetic event for latency measurement
echo 'wakeup_lat u64 lat' > $T/synthetic_events

# Create histogram on sched_waking, save timestamp
echo 'hist:keys=pid:ts0=common_timestamp.usecs' > $T/events/sched/sched_waking/trigger

# On sched_switch, calculate latency and trigger synthetic event
echo 'hist:keys=next_pid:lat=common_timestamp.usecs-$ts0:onmatch(sched.sched_waking).wakeup_lat($lat)' > $T/events/sched/sched_switch/trigger

# Create histogram on synthetic event
echo 'hist:keys=lat:sort=lat' > $T/events/synthetic/wakeup_lat/trigger

# Enable events
echo 1 > $T/events/sched/sched_waking/enable
echo 1 > $T/events/sched/sched_switch/enable
echo 1 > $T/events/synthetic/wakeup_lat/enable

# Read latency histogram
cat $T/events/synthetic/wakeup_lat/hist
```

## trace-cmd Workflows

### Installation

```bash
# Debian/Ubuntu
apt install trace-cmd

# RHEL/CentOS
yum install trace-cmd

# From source
git clone git://git.kernel.org/pub/scm/utils/trace-cmd/trace-cmd.git
cd trace-cmd && make && make install
```

### Basic Recording

```bash
# Trace events during command
trace-cmd record -e sched:sched_switch ./myapp

# Trace function calls
trace-cmd record -p function ./myapp

# Function graph
trace-cmd record -p function_graph ./myapp

# Multiple events
trace-cmd record -e 'sched:*' -e 'irq:*' ./myapp

# All events (high overhead)
trace-cmd record -e all ./myapp

# System-wide tracing
trace-cmd record -e sched -e block sleep 10

# Trace running process
trace-cmd record -e sched -P 1234 sleep 10
```

### Filtering

```bash
# Filter specific functions
trace-cmd record -p function -l 'tcp_*' ./myapp

# Exclude functions
trace-cmd record -p function -n 'tcp_poll' -l 'tcp_*' ./myapp

# Filter events
trace-cmd record -e kmem:kmalloc -f 'bytes_req > 1024' ./myapp

# Filter by PID
trace-cmd record -e sched -P 1234 sleep 10

# Filter by CPU
trace-cmd record -e sched -M 0-3 sleep 10
```

### Report and Analysis

```bash
# Basic report
trace-cmd report

# Report from specific file
trace-cmd report -i trace.dat

# Show time differences between events
trace-cmd report --ts-diff

# Filter by event type
trace-cmd report -e sched_switch

# Filter by CPU
trace-cmd report --cpu 0

# Filter by PID
trace-cmd report -P 1234

# Filter by time range
trace-cmd report --ts 1234567890.123:1234567891.456

# Show raw format
trace-cmd report -R

# List events in trace file
trace-cmd report --events

# Statistics summary
trace-cmd report --stat
```

### Advanced Recording Options

```bash
# Set buffer size
trace-cmd record -b 8192 -e sched ./myapp

# Real-time output (stream to console)
trace-cmd record -e sched -O trace_printk ./myapp

# Record to network (remote collection)
trace-cmd listen -p 12345 &
trace-cmd record -N server:12345 -e sched ./myapp

# Split by CPU
trace-cmd record -o trace -s 100 -e sched sleep 10
# Creates trace.dat, trace.1.dat, trace.2.dat, ...

# Record only when tracing_on toggled
trace-cmd record -e sched --max-graph-depth 5 ./myapp
```

### Profile Mode

```bash
# Generate profile report
trace-cmd record -e sched --profile ./myapp
trace-cmd report --profile

# Profile with stack traces
trace-cmd record -e sched -T --profile ./myapp
```

### Extracting Flame Graph Data

```bash
# Record with stack traces
trace-cmd record -p function_graph -g do_sys_openat2 ./myapp

# Convert to folded format for FlameGraph
trace-cmd report --graph-tsp | \
  awk '/^[ ]+[0-9]/ {gsub(/[|+]/, ""); gsub(/;/, ""); print}' | \
  sort | uniq -c | \
  awk '{print $2, $1}' > trace_folded.txt

# Or use specialized tools
trace-cmd report | stackcollapse-ftrace.pl | flamegraph.pl > trace.svg
```

### Kernel Trace Snapshot

```bash
# Enable snapshot support
trace-cmd snapshot -s

# Take snapshot
trace-cmd snapshot

# Extract snapshot
trace-cmd snapshot -e

# Clear snapshot
trace-cmd snapshot -f
```

### KernelShark Integration

```bash
# Install KernelShark
apt install kernelshark

# Record and visualize
trace-cmd record -e sched ./myapp
kernelshark trace.dat

# GUI features:
# - Timeline view per CPU
# - Zoom and pan
# - Filter by task/CPU/event
# - Export filtered data
```

## Kprobe Dynamic Tracing

### Add Kprobe Events

```bash
T=/sys/kernel/tracing

# Probe function entry
echo 'p:my_probe do_sys_openat2' > $T/kprobe_events

# Probe with arguments
echo 'p:my_probe do_sys_openat2 dfd=%di filename=+0(%si):string flags=%dx' > $T/kprobe_events

# Return probe
echo 'r:my_retprobe do_sys_openat2 $retval' >> $T/kprobe_events

# Enable probe
echo 1 > $T/events/kprobes/my_probe/enable

# Check trace
cat $T/trace

# Remove probe
echo '-:my_probe' >> $T/kprobe_events
```

### Argument Syntax

```bash
# Register access (x86_64 calling convention)
# %di = arg1, %si = arg2, %dx = arg3, %cx = arg4, %r8 = arg5, %r9 = arg6

# Dereference pointer
+0(%si)     # First byte at address in %si
+8(%si)     # 8 bytes offset from %si

# Type casting
+0(%si):string   # Read as null-terminated string
+0(%si):u32      # Read as 32-bit unsigned
+0(%si):s64      # Read as 64-bit signed
+0(%si):x64      # Read as 64-bit hex

# Structure field access (needs vmlinux with debug info)
# Check /proc/kallsyms for function addresses
```

### Kretprobe for Return Values

```bash
T=/sys/kernel/tracing

# Trace return value
echo 'r:open_ret do_sys_openat2 ret=$retval' > $T/kprobe_events
echo 1 > $T/events/kprobes/open_ret/enable

# Filter failures
echo 'ret < 0' > $T/events/kprobes/open_ret/filter

cat $T/trace_pipe
```

## Production Safety

### Overhead Estimation

| Operation | Overhead | Notes |
|-----------|----------|-------|
| Event disabled | ~0 | NOP instruction |
| Event enabled, no filter | ~100-500ns | Ring buffer write |
| Event with filter | ~200-1000ns | Filter evaluation |
| Function tracer (all) | 5-50% | DON'T use in production |
| Function tracer (filtered) | <1% | Few hundred functions OK |
| Function graph (all) | 10-100% | NEVER in production |
| Function graph (filtered) | 1-5% | Single function tree OK |

### Buffer Sizing Guidelines

```bash
T=/sys/kernel/tracing

# Check current size
cat $T/buffer_size_kb       # Per-CPU size
cat $T/buffer_total_size_kb # Total

# Production recommendations:
# - Low-frequency events (signals, OOM): 1-4 MB
# - Medium (syscalls, block I/O): 8-32 MB
# - High (sched_switch, function): 64-256 MB

# Set based on event rate
# Events/sec * event_size * desired_retention_seconds / num_CPUs

echo 16384 > $T/buffer_size_kb  # 16 MB per CPU
```

### What NOT to Trace

**Never trace in production without filtering:**
- `function` tracer without `set_ftrace_filter` - every function call
- `sched_switch` without filter - fires thousands/second
- `kmalloc`/`kfree` without filter - extremely frequent
- Any syscall enter/exit without PID filter

**High-risk functions (can cause livelock):**
```bash
# These functions are called during tracing itself
# Tracing them causes infinite recursion or livelock
ring_buffer_*
trace_*
ftrace_*
rb_*
```

**Safe patterns:**
```bash
# Always filter by function
echo 'tcp_v4_connect' > $T/set_ftrace_filter
echo 'ext4_*' > $T/set_ftrace_filter

# Filter events by field
echo 'bytes_req > 4096' > $T/events/kmem/kmalloc/filter

# Filter by PID for syscall tracing
echo 'common_pid == 1234' > $T/events/syscalls/sys_enter_read/filter
```

### Production Enable/Disable Pattern

```bash
#!/bin/bash
# Safe ftrace wrapper for production

T=/sys/kernel/tracing
DURATION=${1:-10}

# Cleanup on exit
cleanup() {
    echo 0 > $T/tracing_on
    echo nop > $T/current_tracer
    echo > $T/set_ftrace_filter
    echo > $T/set_event
    echo > $T/trace
}
trap cleanup EXIT

# Setup (tracing off)
echo 0 > $T/tracing_on

# Configure FIRST, then enable
echo function_graph > $T/current_tracer
echo do_sys_openat2 > $T/set_graph_function
echo 3 > $T/max_graph_depth

# Small buffer to limit impact
echo 4096 > $T/buffer_size_kb

# Start tracing
echo 1 > $T/tracing_on
sleep $DURATION
echo 0 > $T/tracing_on

# Save results
cp $T/trace /tmp/ftrace_output.txt
echo "Trace saved to /tmp/ftrace_output.txt"
```

### Emergency Disable

```bash
# Immediate stop
echo 0 > /sys/kernel/tracing/tracing_on

# Full reset
echo nop > /sys/kernel/tracing/current_tracer
echo > /sys/kernel/tracing/events/enable  # Disable all events
echo > /sys/kernel/tracing/set_ftrace_filter
echo > /sys/kernel/tracing/kprobe_events
echo > /sys/kernel/tracing/trace

# Verify
cat /sys/kernel/tracing/current_tracer   # Should show 'nop'
cat /sys/kernel/tracing/set_event        # Should be empty
```

## Common Debug Scenarios

### Trace Slow System Calls

```bash
T=/sys/kernel/tracing

# Setup
echo 0 > $T/tracing_on
echo function_graph > $T/current_tracer
echo do_sys_openat2 > $T/set_graph_function
echo 200 > $T/tracing_thresh  # Only calls > 200us

# Trace
echo 1 > $T/tracing_on
# Run workload
echo 0 > $T/tracing_on

# Analyze slow opens
grep -E '^\s*[0-9]+\)\s+[+!]' $T/trace
```

### Identify Lock Contention

```bash
T=/sys/kernel/tracing

# Trace mutex operations
echo 0 > $T/tracing_on
echo function > $T/current_tracer
echo 'mutex_lock
mutex_lock_interruptible
mutex_unlock' > $T/set_ftrace_filter
echo 1 > $T/options/func_stack_trace

echo 1 > $T/tracing_on
# Run workload
echo 0 > $T/tracing_on

# Find frequent lock sites
grep -A5 'mutex_lock' $T/trace | grep '=>' | sort | uniq -c | sort -rn | head
```

### Trace Memory Allocation Failures

```bash
T=/sys/kernel/tracing

# Setup kmalloc tracing with failure filter
echo 1 > $T/events/kmem/kmalloc/enable
echo 'ptr == 0' > $T/events/kmem/kmalloc/filter
echo 'stacktrace' > $T/events/kmem/kmalloc/trigger

# Or trace page allocation failures
echo 1 > $T/events/kmem/mm_page_alloc/enable
echo 'page == 0' > $T/events/kmem/mm_page_alloc/filter
echo 'stacktrace' > $T/events/kmem/mm_page_alloc/trigger
```

### Debug Wake-up Latency

```bash
T=/sys/kernel/tracing

# Use wakeup tracer
echo 0 > $T/tracing_on
echo wakeup > $T/current_tracer

# Or for RT tasks
echo wakeup_rt > $T/current_tracer

echo 1 > $T/tracing_on
# Run latency-sensitive workload
echo 0 > $T/tracing_on

# Check max latency
cat $T/trace
# Shows: "wakeup latency trace v1.1.5 on..."
# With max latency recorded
```

### Catch Rare Events

```bash
T=/sys/kernel/tracing

# Pre-fill buffer with context
echo 1 > $T/events/sched/sched_switch/enable
echo 1 > $T/events/sched/sched_wakeup/enable

# Stop tracing when rare event occurs
echo 'traceoff if sig == 9' > $T/events/signal/signal_generate/trigger
echo 1 > $T/events/signal/signal_generate/enable

echo 1 > $T/tracing_on
# Wait for SIGKILL
# Tracing stops automatically, buffer contains context before kill
cat $T/trace
```

### Network Packet Drop Analysis

```bash
T=/sys/kernel/tracing

# Enable network drop events
echo 1 > $T/events/skb/kfree_skb/enable
echo 'stacktrace' > $T/events/skb/kfree_skb/trigger

# Or trace specific protocol drops
echo 1 > $T/events/tcp/tcp_drop/enable
echo 'stacktrace' > $T/events/tcp/tcp_drop/trigger

cat $T/trace_pipe  # Watch live
```

## Quick Reference

### One-Liners

```bash
T=/sys/kernel/tracing

# List available tracers
cat $T/available_tracers

# List traceable functions (slow - large output)
wc -l $T/available_filter_functions

# Search for function
grep tcp_sendmsg $T/available_filter_functions

# List event subsystems
ls $T/events/

# List events in subsystem
ls $T/events/sched/

# Quick function trace
echo function > $T/current_tracer; echo tcp_sendmsg > $T/set_ftrace_filter; echo 1 > $T/tracing_on; sleep 5; echo 0 > $T/tracing_on; cat $T/trace

# Watch trace live
cat $T/trace_pipe

# Save trace to file
cat $T/trace > /tmp/trace.txt

# Clear trace buffer
echo > $T/trace
```

### trace-cmd Quick Reference

| Task | Command |
|------|---------|
| Record events | `trace-cmd record -e sched:sched_switch ./app` |
| Record function graph | `trace-cmd record -p function_graph ./app` |
| Record with stack traces | `trace-cmd record -e sched -T ./app` |
| View report | `trace-cmd report` |
| Filter report by PID | `trace-cmd report -P 1234` |
| Filter report by event | `trace-cmd report -e sched_switch` |
| Show timestamp diffs | `trace-cmd report --ts-diff` |
| Profile mode | `trace-cmd record --profile -e sched ./app` |
| GUI visualization | `kernelshark trace.dat` |

### Common Events for Production

| Event | Use Case |
|-------|----------|
| `sched:sched_switch` | Context switch analysis |
| `sched:sched_wakeup` | Wake-up latency |
| `sched:sched_migrate_task` | CPU migration |
| `block:block_rq_issue` | Block I/O submission |
| `block:block_rq_complete` | Block I/O completion |
| `net:net_dev_xmit` | Network transmission |
| `tcp:tcp_retransmit_skb` | TCP retransmissions |
| `signal:signal_generate` | Signal delivery |
| `kmem:kmalloc` | Memory allocation |
| `kmem:mm_page_alloc` | Page allocation |
| `writeback:*` | Disk writeback |
| `workqueue:*` | Kernel workqueue |

## See Also

- [eBPF & Tracing](06-ebpf-tracing.md) - bpftrace, BCC tools, programmable tracing
- [Performance Profiling](05-performance-profiling.md) - perf, flame graphs
- [Kernel Tuning](08-kernel-tuning.md) - Scheduler tuning, sysctl
- [Latency Analysis](13-latency-analysis.md) - End-to-end latency debugging

## References

- [Linux Kernel ftrace Documentation](https://docs.kernel.org/trace/ftrace.html)
- [Event Tracing Documentation](https://docs.kernel.org/trace/events.html)
- [Event Histograms Documentation](https://docs.kernel.org/trace/histogram.html)
- [trace-cmd Manual](https://man7.org/linux/man-pages/man1/trace-cmd.1.html)
- [Brendan Gregg - Linux Performance](https://www.brendangregg.com/linuxperf.html)
- [Steven Rostedt - ftrace Presentations](https://events.linuxfoundation.org/wp-content/uploads/2022/10/Steven-Rostedt-ftrace-mentorship-2021.pdf)
- [LWN - Debugging the kernel using Ftrace](https://lwn.net/Articles/365835/)
- [KernelShark Documentation](https://kernelshark.org/Documentation.html)
