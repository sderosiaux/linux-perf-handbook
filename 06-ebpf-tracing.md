# eBPF & Tracing

BCC tools, bpftrace, ftrace, and kernel tracing.

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

# Trace inline function (requires debug symbols)
bpftrace -e 'uprobe:/path/to/binary:inlined_func { printf("hit\n"); }'
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

### OOM Profiling Pattern

Intercept SIGKILL from oomd (user-space OOM killer) using bpftrace since SIGKILL cannot be intercepted in user space.

```bpftrace
# Intercept oomd kills
tracepoint:signal:signal_generate
/args->sig == 9 && str(comm) == "oomd"/ {
    @[pid] = 1;  // Mark as oomd victim
}
```

Pattern for memory-efficient profiling:
- Use bucket rotation (current/previous) to bound memory usage
- Track aggregations by (table, source, SQL) triplets
- Use `avg()` built-in for rolling averages of memory/CPU cost per aggregation

**Key insight:** LRU hashmaps are not yet supported in bpftrace; use bucket rotation as workaround for bounded memory profiling.

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

### Inline Function Tracing

bpftrace uses LLDB from the LLVM ecosystem to trace inline functions. When tracing a binary with debug symbols (vmlinux or user binaries with DWARF), bpftrace can attach probes to all inline instances of a function.

**Key insight:** This also fixes missing stack entries in user space tracing that occurred when the first entry was lost due to probe placement.

### Container/Namespace Support

bpftrace handles PID/TID helpers correctly when running inside containers with PID namespacing. The tool automatically switches between different BPF helpers depending on whether bpftrace and the traced binary are in the same namespace.

**Limitation:** Not yet supported when bpftrace is in a child namespace and the target is in the root namespace.

### Upcoming Features

Features in development:
- **Map type declarations**: Explicit hash map declarations with key/value types
- **User-defined functions**: Split probes into reusable functions
- **External function imports**: Call functions from external eBPF programs (enables Python stack unwinding)
- **Command-line options**: Script arguments via `option()` built-in

```bpftrace
# Future syntax for map declarations
@counter: hash<u32, s64>;

# Future syntax for script options
option("interval", int, 10);
interval:s:$interval { print(@stats); }
```

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

## sched_ext: eBPF-Based Schedulers (Mainline 6.12+)

sched_ext is now merged into mainline Linux (6.12+), enabling custom CPU schedulers via eBPF without kernel patches. Production-ready schedulers available in the [scx repository](https://github.com/sched-ext/scx).

### Available Schedulers

| Scheduler | Use Case | Notes |
|-----------|----------|-------|
| `scx_lavd` | Latency-sensitive (gaming, desktop) | Meta's fleet default |
| `scx_rusty` | Throughput-optimized | Good for batch workloads |
| `scx_simple` | Learning/debugging | Minimal implementation |
| `scx_central` | Single-queue experiments | For scheduler research |

### Quick Start (6.12+)

```bash
# Install from scx repo
git clone https://github.com/sched-ext/scx
cd scx && meson setup build && ninja -C build
sudo ./build/scheds/rust/scx_lavd

# Check active scheduler
cat /sys/kernel/sched_ext/root/ops

# Revert to default (CFS/EEVDF)
# Simply terminate the scx process (Ctrl+C)
```

### Minimal FIFO Scheduler Pattern

```c
// enqueue: task ready to run
SEC("struct_ops/enqueue")
void BPF_PROG(enqueue, struct task_struct *p, u64 flags) {
    u64 slice = 5000000 / scx_bpf_dsq_nr_queued(SHARED_DSQ);
    scx_bpf_dispatch(p, SHARED_DSQ, slice, flags);
}

// dispatch: CPU needs work
SEC("struct_ops/dispatch")
void BPF_PROG(dispatch, s32 cpu, struct task_struct *p) {
    scx_bpf_consume(SHARED_DSQ);
}
```

### Lottery Scheduler Pattern

```c
// Random backend selection from queue
u64 random_idx = bpf_get_prandom_u32() % queue_len;
bpf_for_each(scx_dsq, p, SHARED_DSQ, 0) {
    if (idx++ == random_idx) {
        scx_bpf_dispatch(p, cpu, slice, 0);
        break;
    }
}
```

**Safety:** If scheduler hangs for 30 seconds, kernel automatically kicks it and falls back to default scheduler.

**Key insight:** 10 lines of C can implement a working scheduler. Meta runs scx_lavd as default fleet scheduler across diverse workloads.

## Netkit: BPF-Programmable Network Device (Stable 6.7+)

Netkit is stable in Linux 6.7+ and solves network namespace performance overhead. Traditional veth pairs add extra hops (TX -> virtual cable -> RX -> softIRQ -> TX to NIC). Netkit creates a primary/peer device pair with built-in BPF attach points.

**Performance Impact:**
- veth: ~60% of host network performance
- veth + BPF optimization: ~90%
- netkit + BPF: ~100% (matches host)

**Architecture:**
- Primary device in host namespace, peer in container namespace
- BPF programs at TX side redirect directly to physical NIC queue
- Greatly reduces softIRQ overhead

### Quick Start (6.7+)

```bash
# Create netkit pair
ip link add nk0 type netkit peer name nk1

# Move peer to container namespace
ip link set nk1 netns <container_pid>

# Attach BPF program (required - default drops all traffic)
bpftool net attach netkit_primary id <prog_id> dev nk0

# Check attached programs
bpftool net show dev nk0
```

**Requirements:**
- Linux 6.7+ with CONFIG_NETKIT=y
- iproute2 6.7+ / bpftool with netkit support
- BPF programs must be attached (default drops all traffic)

**Key insight:** Netkit is transparent to containers - no application changes required, just swap veth for netkit.

## XDP (eXpress Data Path)

### Layer-2 Load Balancer (Direct Server Return)

DSR at Layer 2: load balancer only modifies MAC addresses, backend responds directly to client.

**Advantages:**
- Millions of connections through single load balancer (no port allocation)
- No state required - can failover between load balancers like routers
- Asymmetric routing: responses bypass load balancer

**Implementation Pattern:**
```c
// Update MAC addresses only
__builtin_memcpy(eth->h_source, lb_mac, ETH_ALEN);
__builtin_memcpy(eth->h_dest, backend_mac, ETH_ALEN);
return XDP_TX;
```

**Maps Needed:**
- Service -> backend info (IPs, MACs)
- VLAN ID -> interface index (for bpf_redirect_map)
- Flow tracking for connection persistence

**Health Check via NAT Map:**
- Create special veth interface with routes to NAT addresses
- Map (VIP, real_IP) -> MAC address
- Probe VIP through backend's actual MAC

**Key insight:** DSR requirement: VIP must be configured on backend loopback - health checks must verify VIP presence, not just backend health.

## eBPF Development

### Aya (Rust)

Aya is a pure Rust eBPF library without libbpf dependency. Uses libpc (minimal dependencies) and supports BPF CO-RE (Compile Once, Run Everywhere).

**Quick Start:**
```bash
cargo install cargo-generate
cargo generate https://github.com/aya-rs/aya-template
# Select XDP, kprobe, uprobe, TC, etc.
```

**Generated Structure:**
- `<name>-ebpf/`: BPF program (restricted Rust subset)
- `<name>/`: User-space loader
- Uses `aya-log` for logging from BPF programs

**Templates available for:** XDP, kprobes, uprobes, tracepoints, TC (traffic control), LSM hooks.

**Key insight:** Aya aims for feature parity with libbpf while providing Rust safety guarantees and async runtime support (tokio, async-std).

### Cilium/eBPF (Go)

**bpf2go Workflow:**
```go
//go:generate go run github.com/cilium/ebpf/cmd/bpf2go counter counter.c

func main() {
    objs := counterObjects{}
    loadCounterObjects(&objs, nil)

    link.AttachXDP(link.XDPOptions{
        Program:   objs.XdpCounter,
        Interface: ifindex,
    })

    // Read from map
    objs.PacketCount.Lookup(&key, &count)
}
```

**Map Operations from Go:**
```go
// Write to map from user space
objs.PolicyMap.Put(&srcIdentity, &policy)

// Read from map
objs.PacketCount.Lookup(&key, &value)
```

**Key insight:** Cilium uses eBPF to bypass iptables chain entirely - XDP programs hook before kernel network stack for near-line-rate policy enforcement.

### Nix Development Environment

Nix provides reproducible eBPF development environments with synchronized headers, compiler versions, and kernel configs.

**nixos-test for Multi-VM Testing:**
```nix
runNixOSTest {
  nodes.exporter = { config, ... }: {
    boot.kernelPackages = pkgs.linuxPackages_custom;
    # Enable error injection for bpf_override_return
    boot.kernelPatches = [{ extraConfig = "BPF_KPROBE_OVERRIDE y"; }];
  };

  nodes.collector = { ... }: {
    services.grafana.enable = true;
    services.prometheus.enable = true;
  };

  testScript = ''
    exporter.wait_for_unit("ebpf-exporter.service")
    collector.succeed("curl http://exporter:9090/metrics")
  '';
}
```

**Benefits:**
- Binary cache: build once, share everywhere
- Exact kernel version with custom patches/configs
- Virtual networking between VMs automatic

**Key insight:** Less than 250 lines of Nix code for complete multi-node eBPF test environment with metrics collection.

## Security & Observability

### Short-Lived Process Tracing

EDR (Endpoint Detection and Response) solutions need reliable process metadata. procfs unreliable for short-lived processes (data deleted on exit).

**Process Path Extraction via dentry Traversal:**
```c
// Get dentry from task_struct
struct dentry *dentry = task->fs->pwd.dentry;  // or mm->exe_file

// Walk dentry tree to root
bpf_loop(MAX_PATH_DEPTH, get_path_component, &ctx, 0);
// Collect d_name.name at each level

// Adjust for mount points
struct vfsmount *mnt = task->fs->pwd.mnt;
```

**Command Line Extraction:**
```c
// For execve: args in user space
bpf_probe_read_user(&arg_ptr, sizeof(arg_ptr), &argv[i]);
bpf_probe_read_user_str(buf, sizeof(buf), arg_ptr);
```

**Performance Results:**
- eBPF: ~100% data completeness
- procfs under stress (stress-ng): ~30% data gaps

**Key insight:** Static loop bounds required for verifier; use reasonable depth limits for path traversal (performance vs completeness tradeoff).

### Beyla: Zero-Instrumentation Observability

Beyla uses eBPF to auto-instrument applications without code changes. Runs as daemon/helm chart, hooks into network calls at kernel level.

**Features:**
- Works for any language (including compiled: Go, Rust)
- Emits OpenTelemetry semantic convention metrics
- HTTP/gRPC request metrics, service topology, CPU/memory

**Limitations:**
- No in-depth instrumentation (garbage collection, custom spans)
- Requires BTF-enabled kernel (Raspberry Pi kernels need recompilation with `CONFIG_DEBUG_INFO_BTF=y`)

**Caveat:** SLO alerting without synthetic monitoring fails silently when nodes are down (no metrics = 100% SLI).

**Key insight:** For alerting, combine Beyla metrics with blackbox-exporter health checks - SLOs alone miss infrastructure failures.

### VFS-Level I/O Monitoring

Zero-instrumentation I/O profiling by tracing VFS layer functions. Works across all filesystems (local, NFS, Lustre, GPFS).

**Key VFS Functions to Trace:**
```c
vfs_read, vfs_write, vfs_open, vfs_create
vfs_mkdir, vfs_unlink, vfs_rename
// Variants: vfs_iter_read, vfs_iter_write (newer kernels)
```

**Tracking by cgroup + mount point:**
```c
struct event_key {
    u64 cgroup_id;      // bpf_get_current_cgroup_id() for v2
    char mount_path[64];
};

struct event_value {
    u64 bytes;
    u64 calls;
    u64 errors;
};
```

**Mount Path Extraction (tricky part):**
```c
// Walk dentry->d_parent recursively
// Limited loop iterations (verifier constraint)
// Combine with mount point dentry
```

**Production deployment:** Running on 2500-node HPC cluster for 2+ years with negligible overhead.

**Key insight:** cgroup ID + mount point gives per-job, per-filesystem I/O attribution without touching application code.

### Container Image Profiling

Reduce container image bloat by tracing which files are actually accessed at runtime.

**Approach:**
1. Use OCI pre-start hook to load eBPF program before container starts
2. Hook into LSM `file_open` (catches all file access: open, openat, exec)
3. Filter by mount namespace ID (identifies container processes)
4. Dump accessed file list for later analysis

**Hook Configuration:**
```json
{
  "hook": "/path/to/profiler",
  "when": {
    "annotations": {"profile.file": "/output/path"}
  },
  "stage": ["prestart"]
}
```

**Why LSM file_open:**
- `sys_open` misses `openat` calls
- `execve` doesn't necessarily trigger open syscalls
- LSM hook catches everything: read, execute, mmap

**Use Cases:**
- Identify unused binaries/libraries for removal
- Reduce CVE surface area
- CI/CD integration: run tests, collect profile, build minimal image

**Key insight:** Mount namespace ID is stable across all container processes - use it to filter eBPF events to single container.

## Benchmarking Methodology

**Open vs Closed System Benchmarking:**
- Tight loops (closed systems) give different results than realistic Poisson-distributed workloads
- With turbo boost enabled, randomized workloads can show fewer event drops than tight loops
- Ring buffer benchmarks from kernel self-tests use closed-loop producers

**Environment Setup:**
```bash
# Reduce variability
# Disable hyper-threading
# Disable turbo boost
# Pin to single core
# Use fixed CPU frequency
```

**Top-down analysis for eBPF:**
```bash
perf stat -M TopdownL1 -- ./your_bpf_benchmark
# Look for core-bound vs memory-bound issues
```

**Tool:** `berserker` - generates extreme workloads with Poisson distribution for stress-testing eBPF programs. Tests process-based workloads, syscalls, user-space networking simulation.

**Key insight:** Turbo boost + tight loops can overwhelm ring buffer infrastructure leading to more dropped events than slower, randomized workloads.

## BPF Arena & Modern kfuncs (6.9+)

BPF Arena (Linux 6.9+) provides shared memory regions accessible from both BPF programs and userspace, enabling complex data structures previously impossible.

### Arena Basics

```c
// Define arena in BPF program
struct {
    __uint(type, BPF_MAP_TYPE_ARENA);
    __uint(map_flags, BPF_F_MMAPABLE);
    __uint(max_entries, 1024 * 1024);  // 1MB
} arena SEC(".maps");

// Allocate within arena
void *ptr = bpf_arena_alloc(&arena, size);
```

### Key New kfuncs (6.8+)

| kfunc | Purpose | Added |
|-------|---------|-------|
| `bpf_arena_alloc` | Arena memory allocation | 6.9 |
| `bpf_arena_free` | Arena memory deallocation | 6.9 |
| `bpf_iter_css_task_new` | Iterate cgroup tasks | 6.8 |
| `bpf_cpumask_*` | CPU mask manipulation | 6.3 |
| `bpf_rcu_read_lock` | RCU read-side locking | 6.4 |

### Memory Tiering with BPF (CXL Support)

Linux 6.x series added CXL (Compute Express Link) memory tiering support. BPF can now influence memory placement:

```c
// Use with DAMON for tiered memory management
// Hot pages on fast NUMA nodes, cold pages on CXL-attached memory
bpf_cpumask_set_cpu(cpu, mask);  // Control placement
```

**Key insight:** Arena enables linked lists, trees, and other pointer-based structures in BPF - previously limited to fixed-size maps.

## Verifier Tips

Verifier comparison across kernel versions (4.18 to 6.10):
- Kernel 4.18 verifier generates 7-8 more branches for same programs -> higher memory/time
- State pruning in verifier reduces branch exploration
- Complexity limit (1M instructions) reached around 155 loop iterations

**prevail (alternative verifier):**
- Uses zone abstract domain (more precise but slower)
- Fixed-point computation for loops (more efficient than unrolling)
- Zone domain: 8-9 seconds for large programs vs <1s for kernel verifier

**Observations:**
- Program rejection not always verifier's fault: check kernel configs, capabilities, lockdown mode
- Memory footprint: prevail uses more memory but recent versions reduced by ~1GB

**Key insight:** Verifier performance improved across kernel versions with no regressions; consider zone-domain approaches for loop-heavy programs.

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

## Flame Graph Pipeline

For detailed flame graph analysis, see [Performance Profiling](05-performance-profiling.md#flame-graphs).

```bash
# CPU flame graph with BCC
profile-bpfcc -f -p PID 30 | flamegraph.pl > cpu.svg

# Off-CPU flame graph
offcputime-bpfcc -df -p PID 30 | flamegraph.pl --color=io > offcpu.svg

# With bpftrace
bpftrace -e 'profile:hz:99 { @[kstack] = count(); }' > out.stacks
stackcollapse.pl out.stacks | flamegraph.pl > out.svg
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

## See Also

- [Performance Profiling](05-performance-profiling.md) - perf, flame graphs, language profilers
- [Kernel Tuning](08-kernel-tuning.md) - sched_ext schedulers, CPU/memory tuning
- [Network Tuning](09-network-tuning.md) - XDP use cases, network optimization
- [Containers & K8s](07-containers-k8s.md) - container-aware tracing
