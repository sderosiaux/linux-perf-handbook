# Julia Evans: Systems Debugging & Linux Internals

Extracted from jvns.ca - practical debugging wisdom, mental models, and tool usage patterns.

---

## 1. strace: The Swiss Army Knife

### Core Philosophy

> "strace will never lie to me."

strace intercepts system calls between a program and the kernel. Since every file open, network connection, and process spawn goes through syscalls, strace reveals ground truth about program behavior.

### Essential Commands

```bash
# Basic tracing (output goes to stderr)
strace program_name

# Save to file (follow forks with -f)
strace -f -o /tmp/trace.log program_name

# Attach to running process
strace -p PID

# Filter specific syscalls
strace -e open,read,write program_name
strace -e openat program_name          # Modern file opens
strace -e connect,sendto program_name  # Network connections
strace -e execve program_name          # Command execution

# Timing information
strace -t program_name   # Timestamps
strace -T program_name   # Time spent in each syscall

# Summary statistics
strace -c program_name
```

### Practical Use Cases

| Problem | Command | What to Look For |
|---------|---------|------------------|
| Find config file location | `strace -e openat prog` | Which files opened successfully |
| Debug library loading | `strace -f -o log prog && grep libname log \| grep -v ENOENT` | Actual loaded path vs failed lookups |
| Program hanging | `strace -p PID` | Blocking syscall (select, wait, read) |
| Silent permission errors | `strace prog 2>&1 \| grep -i permission` | EACCES or EPERM errors |
| Find what command runs | `strace -e execve -f script.sh` | Exact arguments passed to subcommands |
| Debug network issues | `strace -e connect,sendto prog` | IP addresses and ports |
| Compare working vs broken | diff traces from both | First divergence point |

### Performance Warning

strace adds significant overhead. A program with 260K syscalls:
- Without strace: 0.04s
- With strace: 2.66s (65x slower)

For high-syscall programs, use `perf` instead.

### Spying on SSH (Security Implication)

Root can capture passwords of users logging in:
```bash
sudo strace -p [sshd_child_pid] 2> strace_out
```
Password appears in `read()` syscall arguments. Demonstrates why root compromise is catastrophic.

---

## 2. perf: Low-Overhead Profiling

### Why perf Over strace

perf uses kernel-level sampling and hardware counters - minimal overhead even on syscall-heavy programs.

### Essential Commands

```bash
# Basic stats (cycles, instructions, cache misses)
perf stat ./program

# Specific events
perf stat -e L1-dcache-misses,L1-dcache-loads ./program

# Live function-level CPU usage (like top for functions)
sudo perf top

# Count syscalls by process (live)
sudo perf top -e raw_syscalls:sys_enter -ns comm -d 1

# Count sent network packets by process
stdbuf -oL perf top -e net:net_dev_xmit -ns comm | strings

# Record for later analysis
sudo perf record -g ./program
sudo perf report

# Trace syscalls (low overhead alternative to strace)
sudo perf record -e 'syscalls:sys_enter_*' ./program
sudo perf script

# List available events
perf list
```

### Flame Graphs

```bash
sudo perf record -g ./program
sudo perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

Width of each stack frame = proportion of CPU samples.

### How perf Works

1. **Hardware counters**: Modern CPUs have built-in performance counters
2. **rdpmc instruction**: Kernel reads counters directly
3. **Sampling**: Periodic snapshots instead of every-instruction tracing
4. **Ring buffer**: Events written to mmap'd memory, forwarded to userspace
5. **perf_event_open()**: Core syscall connecting userspace to kernel events

### Real-World Debugging Examples

| Symptom | perf top Revealed | Root Cause |
|---------|-------------------|------------|
| Slow Node.js app | V8 GC functions dominating | Memory pressure, GC thrashing |
| High CPU, no obvious work | Kernel crypto functions | Swapping encrypted memory to disk |
| Slow HTTPS downloads | libcrypto.so heavy | SSL/TLS overhead (expected) |

---

## 3. Debugging Mental Models

### The Debugging Manifesto

1. **Inspect before fix**: Understand exactly what's wrong before changing code
2. **Every bug has a cause**: Nothing is random; unexplained != unexplainable
3. **Verify assumptions**: Even trusted code can be wrong (but 95% of the time it's your code)
4. **Talk to people**: Rubber ducking works; colleagues provide context
5. **Frame as learning**: Bugs that break your mental model teach the most

### Getting Better at Debugging

- **Build tool proficiency**: strace, tcpdump, gdb, perf
- **Deepen system knowledge**: TCP, GC, kernel behavior
- **Maintain confidence**: You've fixed bugs before; you'll fix this one
- **Observe directly**: "Being able to observe what a program is actually doing is incredibly valuable"

### Debug Like It's Closed Source

Your OS is a better debugging investment than language-specific tools - you switch languages more than operating systems.

Key observability points:
- `strace -e open` - what files accessed
- `strace -e execve -f` - what commands executed
- `/proc/[PID]/fd/*` - currently open file descriptors
- `ltrace` - library calls (string comparisons, malloc)

---

## 4. TCP Fundamentals

### Nagle's Algorithm + Delayed ACKs = Latency Bug

**Nagle's Algorithm**: Hold small packets, wait for ACK before sending next

**Delayed ACKs**: Wait ~200ms before ACKing, hoping to piggyback on response data

**The Problem**: Client sends partial request (2 packets), Nagle waits for ACK, server waits for more data before ACKing → 200ms delay on localhost requests.

**Fix**: `TCP_NODELAY` socket option disables Nagle.

### TCP Concepts

- **Three-way handshake**: SYN → SYNACK → ACK
- **Sequence numbers**: Track packet order, detect loss
- **ACKs**: Confirm receipt; missing ACKs trigger retransmission

### Building TCP Stack (Learning Approach)

Using Scapy in Python:
```python
ip_header = IP(dst=dest_ip, src=src_ip)
syn = TCP(dport=80, sport=59333, ack=0, flags="S")
response = srp(ip_header / syn)
```

Key insight: Python too slow to ACK fast enough - connection reset by server.

---

## 5. DNS Internals

### "DNS Propagation" is a Myth

DNS doesn't propagate (push). It's pull-based:
- Resolvers query authoritative nameservers
- Results cached based on TTL
- "Propagation delay" = cache expiration across resolvers

**New records are instant**: No cache exists, so first query hits authoritative server.

### Why Updates Seem Slow

1. **TTL violations**: Some resolvers ignore your TTL
2. **Negative caching**: "Record doesn't exist" is cached (RFC 2308)
3. **Browser/OS caches**: Additional layers beyond DNS resolvers
4. **Nameserver changes**: Registrar coordination, longer TTLs (~1 day)

### Check Negative Cache TTL

```bash
dig soa jvns.ca
# Look at last number in SOA record - that's negative cache TTL
```

### Running Your Own DNS Server

**Authoritative nameserver reasons**:
- Security (reduce vendor access)
- Custom routing logic
- Cost savings (avoid per-query fees)
- Support newer record types

**Resolver reasons**:
- Privacy (prevent ISP monitoring)
- Content filtering (Pi-Hole, Quad9)
- Internal domain resolution
- MITM prevention via DNS-over-HTTPS

---

## 6. Container Internals

### Containers Are Not VMs

Containers = isolated processes on same kernel, using:

| Feature | Purpose |
|---------|---------|
| **Namespaces** | Isolation (PID, network, mount, user) |
| **Cgroups** | Resource limits (CPU, memory) |
| **seccomp-bpf** | Syscall filtering |
| **Capabilities** | Fine-grained privileges |
| **Overlay FS** | Layered filesystems |
| **pivot_root** | Change root filesystem |

### Namespace Demo

```bash
# Create new PID namespace - only see bash and children
sudo unshare --fork --pid --mount-proc bash
ps aux  # Only shows processes in this namespace
```

### Cgroups Demo

```bash
# Limit memory to 10MB
sudo cgcreate -a bork -g memory:mycoolgroup
echo 10000000 > /sys/fs/cgroup/memory/mycoolgroup/memory.limit_in_bytes
sudo cgexec -g memory:mycoolgroup bash
# Now memory-intensive operations will fail
```

### Container Debugging Commands

```bash
nsenter -t PID -n    # Enter network namespace of PID
lsns                 # List all namespaces
ls /proc/PID/ns/     # See namespaces for specific process
ls /sys/fs/cgroup/   # Cgroup hierarchy
getpcaps PID         # Show process capabilities
```

---

## 7. Process Creation: fork/exec

### The Pattern

Linux creates processes in two steps:

1. **fork()**: Clone current process (same code, memory, env)
2. **exec()**: Replace with new program (keeps env vars, file descriptors, signal handlers)

```c
int pid = fork();
if (pid == 0) {
    // Child: transform into new program
    exec(["ls"]);
} else if (pid > 0) {
    // Parent: continues execution
    wait(pid);
}
```

### Why This Matters for Debugging

exec preserves:
- Environment variables
- Open file descriptors
- Signal handlers
- Working directory

This is why environment issues persist across subprocess calls.

### View Process Tree

```bash
pstree -p  # Show process hierarchy with PIDs
```

---

## 8. Quick Reference

### Debugging Workflow

```
1. Reproduce → 2. Observe (strace/perf) → 3. Hypothesize → 4. Test → 5. Fix
```

### Tool Selection

| Need | Tool |
|------|------|
| What files does X open? | `strace -e openat` |
| What's using CPU? | `perf top` |
| What syscalls, low overhead? | `perf record -e 'syscalls:*'` |
| What library calls? | `ltrace` |
| Network packets? | `tcpdump` |
| Open file descriptors? | `ls /proc/PID/fd` |
| Memory maps? | `cat /proc/PID/maps` |

### Common strace Filters

```bash
-e trace=file      # File operations
-e trace=process   # fork, exec, exit
-e trace=network   # Socket operations
-e trace=signal    # Signal handling
-e trace=ipc       # IPC operations
-e trace=memory    # Memory mapping
```

---

Source: jvns.ca - Julia Evans' blog
