# eBPF & Tracing

BCC tools, bpftrace, ftrace, and kernel tracing.

## BCC Tools

BCC (BPF Compiler Collection) provides ready-to-use eBPF-based tools.

**Install:** `apt install bpfcc-tools` (tools get `-bpfcc` suffix)

### Process/CPU Tools

```bash
execsnoop-bpfcc                # Trace new processes
execsnoop-bpfcc -t             # With timestamps
execsnoop-bpfcc -x             # Include failed execs
execsnoop-bpfcc -n java        # Filter by name

runqlat-bpfcc                  # Run queue latency histogram
runqlat-bpfcc -m               # Milliseconds
runqlat-bpfcc 1 10             # 1s interval, 10 outputs

runqlen-bpfcc                  # Run queue length histogram
runqlen-bpfcc -C               # Per-CPU

cpudist-bpfcc                  # On-CPU time distribution
cpudist-bpfcc -O               # Off-CPU time
cpudist-bpfcc -p PID           # Specific process

offcputime-bpfcc               # Off-CPU time flame graph data
offcputime-bpfcc -p PID        # Specific process
offcputime-bpfcc -df 30        # Folded output, 30s

profile-bpfcc                  # CPU profiler
profile-bpfcc -p PID           # Specific process
profile-bpfcc -f               # Folded output (for flame graphs)
profile-bpfcc -F 99            # 99 Hz sampling
```

### File/Disk Tools

```bash
opensnoop-bpfcc                # Trace file opens
opensnoop-bpfcc -p PID         # Specific process
opensnoop-bpfcc -n java        # By name
opensnoop-bpfcc -x             # Failed opens only
opensnoop-bpfcc -T             # Timestamps

filelife-bpfcc                 # Short-lived files
fileslower-bpfcc               # Slow file operations
fileslower-bpfcc 10            # > 10ms threshold

biolatency-bpfcc               # Block I/O latency
biolatency-bpfcc -D            # Per-disk
biolatency-bpfcc -m            # Milliseconds

biosnoop-bpfcc                 # Block I/O tracing
biosnoop-bpfcc -Q              # Include queue time
biosnoop-bpfcc -d sda          # Specific disk

biotop-bpfcc                   # Top for block I/O
biotop-bpfcc 5                 # 5 second refresh

cachestat-bpfcc                # Page cache stats
cachestat-bpfcc 1              # 1 second interval

ext4slower-bpfcc               # Slow ext4 operations
xfsslower-bpfcc                # Slow XFS operations
```

### Memory Tools

```bash
memleak-bpfcc                  # Memory leak detector
memleak-bpfcc -p PID           # Specific process
memleak-bpfcc -a               # All allocations
memleak-bpfcc -o 10000         # Older than 10s

oomkill-bpfcc                  # OOM kill tracing

slabratetop-bpfcc              # Kernel slab allocations
```

### Network Tools

```bash
tcpconnect-bpfcc               # Trace TCP connects
tcpconnect-bpfcc -p PID        # Specific process
tcpconnect-bpfcc -P 80,443     # Specific ports
tcpconnect-bpfcc -t            # Timestamps

tcpaccept-bpfcc                # Trace TCP accepts
tcpaccept-bpfcc -p PID         # Specific process

tcplife-bpfcc                  # TCP connection lifespan
tcplife-bpfcc -D 5000          # Connections > 5s
tcplife-bpfcc -L 80            # Local port 80

tcpretrans-bpfcc               # TCP retransmissions
tcptracer-bpfcc                # Trace TCP state changes

tcpdrop-bpfcc                  # Trace dropped packets
tcprtt-bpfcc                   # TCP RTT distribution

sockstat-bpfcc                 # Socket statistics

softirqs-bpfcc                 # Soft IRQ time
hardirqs-bpfcc                 # Hard IRQ time
```

### Tracing Tools

```bash
trace-bpfcc 'sys_read'         # Trace syscall
trace-bpfcc 'r::sys_read'      # Return value
trace-bpfcc -p PID 'sys_*'     # Wildcards
trace-bpfcc 'do_sys_open "%s", arg2'  # With arguments

funccount-bpfcc 'vfs_*'        # Count function calls
funccount-bpfcc -i 1 'tcp_*'   # 1 second interval

funclatency-bpfcc vfs_read     # Function latency histogram
funclatency-bpfcc -m vfs_read  # Milliseconds

stackcount-bpfcc 'vfs_read'    # Stack traces by count
```

## bpftrace

High-level tracing language for eBPF.

**Install:** `apt install bpftrace`

### One-Liners

```bash
# List available probes
bpftrace -l 'tracepoint:*'
bpftrace -l 'kprobe:tcp_*'

# Trace syscalls
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'

# Syscall count by program
bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'

# Read size distribution
bpftrace -e 'tracepoint:syscalls:sys_exit_read /args->ret > 0/ { @bytes = hist(args->ret); }'

# Process start
bpftrace -e 'tracepoint:sched:sched_process_exec { printf("%s -> %s\n", comm, str(args->filename)); }'

# Timer distribution
bpftrace -e 'kprobe:do_nanosleep { @start[tid] = nsecs; } kretprobe:do_nanosleep /@start[tid]/ { @sleep = hist(nsecs - @start[tid]); delete(@start[tid]); }'

# Block I/O latency
bpftrace -e 'tracepoint:block:block_rq_issue { @start[args->dev, args->sector] = nsecs; } tracepoint:block:block_rq_complete /@start[args->dev, args->sector]/ { @usecs = hist((nsecs - @start[args->dev, args->sector]) / 1000); delete(@start[args->dev, args->sector]); }'

# TCP accepts
bpftrace -e 'kretprobe:inet_csk_accept { @[ntop(((struct sock *)retval)->__sk_common.skc_daddr)] = count(); }'
```

### Script Example

```bpftrace
#!/usr/bin/env bpftrace
// vfsstat.bt - VFS operation counts

BEGIN
{
    printf("Tracing VFS operations... Hit Ctrl-C to end.\n");
}

kprobe:vfs_read    { @reads = count(); }
kprobe:vfs_write   { @writes = count(); }
kprobe:vfs_fsync   { @fsyncs = count(); }
kprobe:vfs_open    { @opens = count(); }
kprobe:vfs_create  { @creates = count(); }

interval:s:1
{
    printf("\n%s ", strftime("%H:%M:%S", nsecs));
    print(@reads); print(@writes); print(@fsyncs);
    clear(@reads); clear(@writes); clear(@fsyncs);
}

END
{
    clear(@reads); clear(@writes); clear(@fsyncs);
    clear(@opens); clear(@creates);
}
```

### Built-in Variables

| Variable | Description |
|----------|-------------|
| `pid` | Process ID |
| `tid` | Thread ID |
| `uid` | User ID |
| `comm` | Process name |
| `nsecs` | Nanoseconds timestamp |
| `cpu` | CPU ID |
| `curtask` | Current task_struct |
| `args` | Tracepoint arguments |
| `retval` | Return value (kretprobe) |
| `kstack` | Kernel stack trace |
| `ustack` | User stack trace |

### Map Functions

| Function | Description |
|----------|-------------|
| `count()` | Count occurrences |
| `sum(x)` | Sum values |
| `avg(x)` | Average |
| `min(x)` | Minimum |
| `max(x)` | Maximum |
| `stats(x)` | Count, average, total |
| `hist(x)` | Power-of-2 histogram |
| `lhist(x,min,max,step)` | Linear histogram |

## ftrace

Kernel's built-in tracer.

### Basic Usage

```bash
# Tracer setup
cd /sys/kernel/debug/tracing
echo function > current_tracer
echo 1 > tracing_on
cat trace
echo 0 > tracing_on

# Function graph tracer
echo function_graph > current_tracer

# Filter specific functions
echo 'tcp_*' > set_ftrace_filter
echo '!tcp_poll' >> set_ftrace_filter  # Exclude

# Trace specific PID
echo PID > set_ftrace_pid

# Clear trace
echo > trace
```

### trace-cmd

```bash
trace-cmd record -e sched command
trace-cmd record -p function_graph command
trace-cmd record -e 'sched:*' -e 'irq:*' command
trace-cmd report
trace-cmd report -i trace.dat

# KernelShark GUI
kernelshark trace.dat
```

### Common Events

```bash
# List available events
cat /sys/kernel/debug/tracing/available_events

# Scheduler events
trace-cmd record -e sched:sched_switch -e sched:sched_wakeup

# Block I/O
trace-cmd record -e block:block_rq_issue -e block:block_rq_complete

# Network
trace-cmd record -e net:*

# Syscalls
trace-cmd record -e syscalls:sys_enter_open
```

## LTTng

Low-overhead tracing.

```bash
# Create session
lttng create my-session

# Enable kernel events
lttng enable-event -k sched_switch,sched_wakeup
lttng enable-event -k --syscall open,read,write

# Enable userspace events
lttng enable-event -u 'my_app:*'

# Start/stop
lttng start
lttng stop

# View results
lttng view
babeltrace ~/lttng-traces/my-session*
```

## perf + eBPF

```bash
# List eBPF programs
perf record -e 'bpf:*'

# Trace eBPF events
perf trace -e bpf:*
```

## System Tracing

### printk / dynamic debug

```bash
# Enable dynamic debug
echo 'file drivers/block/* +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module xfs +p' > /sys/kernel/debug/dynamic_debug/control
```

### kprobes directly

```bash
# Add kprobe
echo 'p:myprobe do_sys_open' > /sys/kernel/debug/tracing/kprobe_events

# Enable
echo 1 > /sys/kernel/debug/tracing/events/kprobes/myprobe/enable

# View
cat /sys/kernel/debug/tracing/trace

# Remove
echo '-:myprobe' >> /sys/kernel/debug/tracing/kprobe_events
```

## Quick Reference

| Task | Tool/Command |
|------|--------------|
| New processes | `execsnoop-bpfcc` |
| File opens | `opensnoop-bpfcc` |
| Block I/O latency | `biolatency-bpfcc` |
| TCP connections | `tcpconnect-bpfcc` |
| Run queue latency | `runqlat-bpfcc` |
| Memory leaks | `memleak-bpfcc -p PID` |
| Off-CPU time | `offcputime-bpfcc -p PID` |
| Function count | `funccount-bpfcc 'vfs_*'` |
| Custom tracing | `bpftrace -e '...'` |
| Kernel tracing | `trace-cmd record -e event` |

## Flame Graph Pipeline

```bash
# CPU flame graph with BCC
profile-bpfcc -f -p PID 30 | flamegraph.pl > cpu.svg

# Off-CPU flame graph
offcputime-bpfcc -df -p PID 30 | flamegraph.pl --color=io > offcpu.svg

# With bpftrace
bpftrace -e 'profile:hz:99 { @[kstack] = count(); }' > out.stacks
stackcollapse.pl out.stacks | flamegraph.pl > out.svg
```
