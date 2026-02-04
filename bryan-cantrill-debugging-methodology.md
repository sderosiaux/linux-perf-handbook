# Bryan Cantrill's Debugging Methodology

**A Practical Framework for Systematic Production Debugging**

Bryan Cantrill (co-creator of DTrace, CTO of Oxide Computer Company) has developed rigorous debugging methodologies through decades of production systems work at Sun Microsystems, Joyent, and Oxide. This guide extracts actionable frameworks for LLM-driven and human debugging.

---

## Core Philosophy: Questions Before Hypotheses

### The Problem with Hypothesis-First Debugging

Traditional approach: Form hypothesis → Test → Revise → Repeat

**Why this fails:**
- Focuses on what you already know
- Anchors on preconceptions and folklore
- Wastes time testing wrong theories
- Computer science programs "do not really teach debugging methodology"

### The Question-First Method

**Process:**

```
1. Formulate informed questions about system behavior
2. Use creativity and tooling to answer those questions
3. Use answers to inform tighter, more specific questions
4. Repeat until hypothesis space collapses
5. Only then form hypothesis — when "you don't have a choice"
```

**Example Decision Tree:**

```
❓ Is CPU saturated?
   ├─ Yes → Which processes? → Which functions? → Why inefficient?
   └─ No → Is I/O saturated?
           ├─ Yes → Disk or network? → Which operations? → Why waiting?
           └─ No → Check memory pressure → Check lock contention → ...
```

**Key Insight:** "You should be iterating between questions and answers, questions and answers, questions and answers... the possible space of hypotheses gets smaller and smaller until you don't have a choice."

### Starting Questions: USE Method

Use Brendan Gregg's USE method as **starting point only:**

- **U**tilization: Is resource busy?
- **S**aturation: Is resource queued?
- **E**rrors: Are there failures?

**Critical:** These are frameworks, not recipes. They provoke questions, they don't solve problems.

---

## The Anti-Pattern: Recovery Without Understanding

### What It Looks Like

```
Problem occurs → Restart system → Problem disappears → Move on
                                            ↑
                                    LEARNING = ZERO
```

### Why It's Dangerous

1. **Normalizes broken software** — accepts failure as inevitable
2. **Destroys evidence** — loses all debugging state
3. **Builds folklore** — "just restart it" becomes institutional knowledge
4. **Guarantees recurrence** — root cause remains unfixed

### The Right Approach

```
Problem occurs → Capture state (core dump) → Restart for availability
                       ↓
                 Postmortem analysis → Root cause → Fix → Verification
```

**Cantrill's Law:** "Prove you fixed it. 'It went away' is not evidence."

---

## Preserving Debugging State

### Why Restart Is the Enemy

**What you lose when you restart:**
- Process memory contents
- File descriptor state
- Thread states and call stacks
- Pending I/O operations
- Lock states
- Connection states

**Cantrill quote:** "As Docker is increasingly used for workloads that matter, one can no longer insist that failures don't happen — or that restarts will cure any that do."

### The Postmortem Workflow

#### 1. Capture State

```bash
# Before restart, capture core dump
gcore -o /var/tmp/myapp.core $(pgrep myapp)

# For Node.js (with metadata)
kill -ABRT $(pgrep node)  # Generates core automatically

# For containers
docker exec <container> gcore 1
docker cp <container>:/core.1 ./
```

#### 2. Restart for Availability

```bash
# Now safe to restart
systemctl restart myapp

# Container example
docker restart <container>
```

#### 3. Asynchronous Analysis

```bash
# Debug core file later with full context
gdb /usr/bin/myapp /var/tmp/myapp.core

# For Node.js with mdb_v8
mdb /usr/bin/node /var/tmp/node.core
::jsstack
::findjsobjects
```

### Key Benefits

- **Zero runtime overhead** — overhead only at time of death
- **Consistent state** — the invalid state is static and analyzable
- **No time pressure** — debug at your own pace
- **Reproducible** — can revisit the same state multiple times

---

## Failure Taxonomy: Four Categories

Cantrill categorizes software failures along two axes:

### The Matrix

```
                  Fatal                  Non-Fatal
              ┌─────────────────────┬─────────────────────┐
   Explicit   │ Assertion failure   │ Error return code   │
              │ Uncaught exception  │ Error message       │
              │                     │                     │
              │ Easy to debug       │ Moderate            │
              ├─────────────────────┼─────────────────────┤
   Implicit   │ Segmentation fault  │ Wrong answer        │
              │ Unhandled signal    │ Resource leak       │
              │                     │ Pathological perf   │
              │                     │                     │
              │ Moderate            │ HARDEST TO DEBUG    │
              └─────────────────────┴─────────────────────┘
```

### Debugging Techniques by Category

| Category | Technique | Tools |
|----------|-----------|-------|
| **Fatal + Explicit** | Read error message, check assertions | Logs, stack traces |
| **Fatal + Implicit** | Postmortem core dump analysis | gdb, mdb, crash |
| **Non-fatal + Explicit** | Log analysis, trace error paths | ELK, dynamic tracing |
| **Non-fatal + Implicit** | Dynamic instrumentation, observation | DTrace, eBPF, profilers |

**Hardest case:** Non-fatal + Implicit
- System appears functional
- Gives wrong answers or terrible performance
- Must be "understood against its will"
- Requires deep observability

---

## The Aggregation Problem

### The 60% Utilization Lie

**Real case study from Cantrill:**

```
$ iostat 1
Device     util%
sda        60%    ← Looks like headroom available
sda        60%
sda        60%
```

**Reality after disaggregation:**

```
Time (ms)    Utilization
0-100        100%    ← Saturated
100-200        0%    ← Idle
200-300      100%    ← Saturated
300-400        0%    ← Idle
400-500      100%    ← Saturated
```

**Average: 60%**
**Actual behavior: 100%/0% oscillation**

### Why Aggregation Hides Problems

1. **Time dimension eliminated** — temporal patterns invisible
2. **Outliers smoothed** — P99 hidden in averages
3. **Oscillations masked** — periodic behavior looks constant
4. **Causality obscured** — can't see event ordering

### Detection Strategy

**When stuck on an aggregated metric, disaggregate:**

```bash
# Instead of: iostat -x 1
# Use: iostat -x 0.1 | tee raw.txt  # High-frequency sampling

# Instead of: average response time
# Use: Latency heatmaps (time on X, latency on Y, color = count)

# Instead of: CPU utilization %
# Use: Per-core flame graphs over time (statemaps)
```

**Visualization over numbers:** The visual cortex excels at pattern discovery superior to automated detection.

---

## Systematic Elimination Framework

### The Decision Tree Approach

**Goal:** Eliminate possibilities methodically until one remains.

```
Symptom: High latency on API endpoint
│
├─ Q: Is CPU saturated?
│  ├─ Yes → Profile CPU usage (flame graph)
│  │        └─ Identify hot function → Optimize or capacity plan
│  └─ No → Continue
│
├─ Q: Is disk I/O saturated?
│  ├─ Yes → Check iostat, iotop
│  │        └─ Slow queries? High IOPS? → Optimize or cache
│  └─ No → Continue
│
├─ Q: Is network saturated?
│  ├─ Yes → Check bandwidth, packet loss
│  │        └─ Bandwidth limit? Packet loss? → Tune or upgrade
│  └─ No → Continue
│
├─ Q: Is it waiting on locks?
│  ├─ Yes → Off-CPU profiling
│  │        └─ Which locks? → Reduce contention
│  └─ No → Continue
│
├─ Q: Is it waiting on external service?
│  ├─ Yes → Distributed trace
│  │        └─ Which service slow? → Debug downstream
│  └─ No → Must be application logic
│           └─ Profile wall-clock time distribution
```

### Template for Creating Decision Trees

```markdown
## Problem: [Symptom]

### Phase 1: Resource Exhaustion
- [ ] CPU saturated? → perf top, flame graphs
- [ ] Memory exhausted? → free -h, slabtop
- [ ] Disk I/O saturated? → iostat -x 1
- [ ] Network saturated? → iftop, ss -s

### Phase 2: Contention
- [ ] Lock contention? → Off-CPU profiling, futex tracing
- [ ] Scheduler delays? → perf sched latency
- [ ] Page faults? → perf stat -e page-faults

### Phase 3: External Dependencies
- [ ] Downstream services slow? → Distributed tracing
- [ ] DNS resolution slow? → strace -T -e connect
- [ ] TLS handshake slow? → tcpdump, wireshark

### Phase 4: Application Logic
- [ ] Inefficient algorithm? → CPU flame graphs
- [ ] Memory allocations? → Heap profiling
- [ ] GC pressure? → Language-specific GC logs

### Verification
- [ ] Fix applied
- [ ] Metrics improved
- [ ] Load test passed
- [ ] Monitoring alerts updated
```

---

## Observability: The Foundation

### Cantrill's Observability Principles

1. **"Software observability is not a boolean"** — exists on a spectrum
2. **"Unobservable systems force guessing"** — there is no excuse for this
3. **"DTrace can answer your questions... but it can't do it for you"** — tools enable, humans drive
4. **"The human is the foundation of observability"** — no tool substitutes for thoughtful questions

### Two Types of Instrumentation

**Static Instrumentation:**
- Logging (application logs)
- Accounting (OS counters: CPU time, I/O bytes)
- Tracepoints (kernel static probes)

**Pros:** Always available, low overhead
**Cons:** Fixed granularity, may miss your question

**Dynamic Instrumentation:**
- DTrace
- eBPF (bpftrace, bcc)
- Systemtap

**Pros:** Arbitrary questions on live systems
**Cons:** Requires kernel support, learning curve

### The Right Tool for the Question

```
Question Type                    Tool Choice
─────────────────────────────────────────────────────────────
"Which process using CPU?"       top, ps
"Which function using CPU?"      perf, flame graphs
"Why is this specific syscall    strace -T
 taking 5 seconds?"
"How many mutex locks per        bpftrace (dynamic)
 second in this function?"
"Show me all network packets     tcpdump, wireshark
 for this connection"
"What's the latency distribution perf + histogram
 of disk I/O?"
```

---

## Visualization-Driven Debugging

### Why Visualization Matters

**Cantrill:** "The value of visualizing data is not merely providing answers, but also (and especially) provoking new questions."

**Human visual cortex advantages:**
- Pattern recognition superior to algorithms
- Detects anomalies instantly
- Spots correlations across dimensions
- Reveals structure in chaos

### Visualization Arsenal

| Tool | Purpose | When to Use |
|------|---------|-------------|
| **Flame graphs** | Visualize work being performed | CPU profiling, identify hot paths |
| **Off-CPU flame graphs** | Visualize blocking time | I/O wait, lock contention |
| **Heat maps** | Latency distribution over time | Spot latency spikes, patterns |
| **Statemaps** | Entity state over time | Find idle periods, concurrency bottlenecks |
| **Histograms** | Distribution of values | Understand percentiles, outliers |
| **Time series** | Metric evolution | Correlate events, detect trends |

### Example Workflow: Performance Investigation

```bash
# 1. Start with flamegraph for CPU
perf record -F 99 -a -g -- sleep 30
perf script | flamegraph.pl > cpu.svg

# If CPU not saturated, check off-CPU
bpftrace -e 'profile:hz:49 /pid == $1/ { @[kstack, ustack] = count(); }' $(pgrep myapp) > offcpu.txt
flamegraph.pl offcpu.txt > offcpu.svg

# 2. Generate latency heatmap
bpftrace -e 'kprobe:blk_account_io_done {
  @[hist(nsecs - args->start_time)] = count();
}' > io_latency.txt

# 3. Create statemap for concurrency view
dtrace -n 'sched:::on-cpu /pid == $target/ { self->ts = timestamp; }
           sched:::off-cpu /self->ts/ {
             printf("%d %d %lld\n", pid, tid, timestamp - self->ts);
           }' -p $(pgrep myapp) > statemap.txt
```

---

## The Psychological Requirements

### Perspiration Over Inspiration

**Cantrill's insight:** "Debugging rewards persistence, grit, and resilience much more than intuition or insight — it is more perspiration than inspiration!"

**What this means:**

```
Intuition:    10%  ← Helpful but not essential
Persistence:  90%  ← Absolutely required
```

**Qualities needed:**
1. **Persistence** — willing to grind through blind alleys
2. **Grit** — won't give up when stuck
3. **Resilience** — handles frustration and failure
4. **Faith** — believes synthetic systems remain comprehensible

**Anti-qualities:**
- Impatience (leads to random changes)
- Ego (resists asking for help or reading docs)
- Superstition (follows folklore without verification)

### Debugging as Science, Not Art

**Cantrill's thesis:** Software debugging is "a pure distillation of scientific thinking" — but with systems that allow experiments in seconds instead of weeks.

**The cycle:**
1. Make observations
2. Formulate questions (not hypotheses!)
3. Design experiments to answer questions
4. Execute experiments
5. Analyze results
6. Refine questions
7. Repeat until root cause identified

**Advantages over physical sciences:**
- Experiments run in seconds
- Systems are deterministic (at their root)
- Can reproduce conditions exactly
- Can instrument without Heisenberg effects (mostly)

---

## Common Debugging Anti-Patterns

### 1. The "20 Questions" Anti-Pattern

**Symptom:** Leaping to conclusions based on past experience.

```
Engineer: "I bet it's GC again."
Reality:  Network partition causing retries
```

**Fix:** Resist folklore. Ask questions, gather data, eliminate possibilities.

### 2. The "Random Alteration" Anti-Pattern

**Symptom:** Changing system configuration without understanding.

```
Engineer: "Let me try disabling hyperthreading."
Result:   Problem persists, now debugging is harder
```

**Fix:** Form hypothesis, predict outcome, measure, verify.

### 3. The "Aggregation Trap" Anti-Pattern

**Symptom:** Debugging based on averaged metrics.

```
Metric:   60% CPU utilization
Belief:   "We have headroom"
Reality:  Oscillating between 100% and 0%
```

**Fix:** When stuck, disaggregate. Look at raw timeline data.

### 4. The "Ignore the Odd" Anti-Pattern

**Symptom:** Dismissing anomalies as ancillary.

```
Engineer: "That's weird, but probably unrelated."
Result:   Weird thing was the clue to root cause
```

**Fix:** Odd behavior is worth understanding. At worst, enhances understanding. At best, reveals something deeply amiss.

### 5. The "Correlation = Causation" Anti-Pattern

**Symptom:** Fixing a problem accidentally, learning nothing.

```
Engineer: "I restarted the service and it's fast now!"
Reality:  Cleared a memory leak by accident, will recur
```

**Fix:** Understand the mechanism. If fix worked, explain WHY it worked.

---

## Production Debugging Playbook

### Triage: First 5 Minutes

**Goal:** Determine severity and capture initial state.

```bash
#!/bin/bash
# Save to /usr/local/bin/triage.sh

echo "=== System Load ==="
uptime
w

echo "=== Resource Utilization ==="
iostat -x 1 2
vmstat 1 2
sar -n DEV 1 2

echo "=== Process States ==="
ps auxf | head -30

echo "=== Top Consumers ==="
top -b -n 1 | head -20

echo "=== Recent Errors ==="
journalctl -p err -n 50 --no-pager

echo "=== Network Connections ==="
ss -s
```

**Decision tree:**

```
Is service down completely?
├─ Yes → Restore service first, debug later
│        1. Capture core dump if possible
│        2. Restart service
│        3. Begin postmortem analysis
│
└─ No → Is it degraded performance?
         ├─ Yes → Safe to debug in situ
         │        1. Start sampling (perf, DTrace)
         │        2. Capture flame graphs
         │        3. Check for saturation (USE method)
         │
         └─ No → Is it intermittent?
                  1. Set up continuous monitoring
                  2. Wait for recurrence
                  3. Trigger instrumentation on threshold
```

### Investigation: Systematic Approach

**Phase 1: Characterize the Problem**

```markdown
## Problem Statement
- **Symptom:** [Specific observable behavior]
- **When:** [First occurrence, frequency, pattern]
- **Where:** [Which systems, components, users]
- **Impact:** [SLA breach? User-visible? Revenue loss?]
- **Changes:** [Deployments, config changes in last 24h]
```

**Phase 2: Resource Exhaustion Check (2 minutes)**

```bash
# CPU
mpstat -P ALL 1 5

# Memory
free -h
slabtop -o | head -20

# Disk
iostat -x 1 5

# Network
iftop -t -s 10
```

**Phase 3: Application-Level Instrumentation (5 minutes)**

```bash
# Flame graph
perf record -F 99 -a -g -- sleep 30
perf script | flamegraph.pl > /tmp/flame.svg

# If CPU not saturated, off-CPU
bpftrace -e 'profile:hz:49 { @[kstack, ustack, comm] = count(); }' -c 30

# Strace top-N syscalls
strace -c -p $(pgrep myapp)
```

**Phase 4: Hypothesis Formation**

Only after gathering data:

```markdown
## Hypotheses (ranked by likelihood)
1. [Most likely based on data]
   - Evidence: [Specific observations]
   - Test: [How to confirm/refute]

2. [Second most likely]
   - Evidence: [Specific observations]
   - Test: [How to confirm/refute]
```

**Phase 5: Systematic Elimination**

```markdown
## Elimination Log
- [ ] Hypothesis 1: [Description]
  - Test: [Command/procedure]
  - Result: [Refuted/Confirmed]
  - Conclusion: [Next steps]

- [ ] Hypothesis 2: [Description]
  - Test: [Command/procedure]
  - Result: [Refuted/Confirmed]
  - Conclusion: [Next steps]
```

### Resolution: Verification Loop

**Don't stop at "it works now":**

```markdown
## Resolution Verification
1. **Root Cause Identified:** [Specific mechanism]
2. **Fix Applied:** [Exact change made]
3. **Why Fix Works:** [Explanation of mechanism]
4. **Metrics Before:** [Baseline numbers]
5. **Metrics After:** [Post-fix numbers]
6. **Load Test Results:** [Synthetic verification]
7. **Monitoring Alerts:** [What to alert on to catch recurrence]
8. **Postmortem Document:** [Link to detailed writeup]
```

---

## Advanced Techniques

### Multiple Root Causes

**Reality:** Production systems often have several interacting problems.

**Approach:**

```
Problem: API latency P99 = 5 seconds

Discovered issues:
1. Database connection pool exhausted (Fix 1) → P99 drops to 2s
2. Lock contention in cache layer (Fix 2) → P99 drops to 500ms
3. DNS resolution timeout (Fix 3) → P99 drops to 100ms
```

**Strategy:**
1. Fix most impactful issue first
2. Re-measure
3. If still above SLA, continue investigation
4. Repeat until satisfactory

**Don't assume one cause!**

### Dynamic Systems: The Moving Target

**Challenge:** Production systems change during debugging.

**Strategies:**

1. **Snapshot state early**
   ```bash
   # Capture baseline immediately
   perf record -o before.data -F 99 -a -g -- sleep 30
   ```

2. **Control for changes**
   ```bash
   # Freeze deployments during investigation
   # Document all changes in investigation log
   ```

3. **Correlate with events**
   ```bash
   # Cross-reference investigation timeline with:
   # - Deployment logs
   # - Auto-scaling events
   # - Traffic pattern changes
   ```

4. **A/B comparison**
   ```bash
   # Compare good instance vs bad instance side-by-side
   # Diff flame graphs, diff config, diff versions
   ```

### Unobservable Systems: Last Resort

**When you can't instrument directly:**

1. **Add "half-measures"** — intentionally modify system to capture MORE state
   ```c
   // Before (opaque):
   fast_path();

   // After (observable):
   if (unlikely(debug_flag)) {
       log_state();
       dump_counters();
   }
   fast_path();
   ```

2. **Proxy instrumentation**
   ```bash
   # Can't trace app? Trace kernel on its behalf
   bpftrace -e 'tracepoint:syscalls:sys_enter_write /comm == "myapp"/ {
     @bytes[tid] = hist(args->count);
   }'
   ```

3. **Statistical inference**
   ```bash
   # Sample /proc at high frequency
   while true; do
     cat /proc/$(pgrep myapp)/status >> state.log
     sleep 0.01
   done
   ```

---

## Troubleshooting Decision Trees

### Generic Performance Problem

```
┌─────────────────────────────────────┐
│ Symptom: Slow response times        │
└─────────────────┬───────────────────┘
                  │
                  ▼
    ┌─────────────────────────────┐
    │ Q: Is latency consistent    │
    │    or intermittent?         │
    └──┬─────────────────┬────────┘
       │ Consistent      │ Intermittent
       ▼                 ▼
  CPU/IO/Memory     ┌───────────────┐
  saturation        │ Q: Periodic?  │
  (USE method)      └─┬─────────┬───┘
                      │ Yes     │ No
                      ▼         ▼
                   GC/cron   Noisy
                   batch     neighbor
                   jobs      contention
```

### Memory Leak Investigation

```
┌──────────────────────────────────┐
│ Symptom: Memory usage climbing  │
└────────────┬─────────────────────┘
             │
             ▼
   ┌─────────────────────────┐
   │ Q: RSS or shared mem?   │
   └──┬──────────────┬───────┘
      │ RSS          │ Shared
      ▼              ▼
 Heap growth    File mappings
      │         (mmap leaks)
      ▼
 ┌────────────────────────┐
 │ Q: Reachable or not?   │
 └──┬────────────┬────────┘
    │ Reachable  │ Not reachable
    │            │
    ▼            ▼
 Unbounded    True leak
 cache growth (free missing)
```

### Lock Contention

```
┌─────────────────────────────┐
│ Symptom: Throughput drops   │
│          as load increases  │
└───────────┬─────────────────┘
            │
            ▼
   ┌────────────────────────┐
   │ CPU scaling linearly?  │
   └──┬──────────────┬──────┘
      │ No           │ Yes
      ▼              ▼
 Lock contention  Not CPU bound
      │            (check I/O)
      ▼
 ┌──────────────────────────┐
 │ Q: Kernel or userspace?  │
 └──┬──────────────┬────────┘
    │ Kernel       │ User
    ▼              ▼
 futex tracing  App profiling
 /proc/PID/     with lock
 wchan          instrumentation
```

---

## Tools Reference

### Essential Observability Stack

**Tier 1: Always Available**
```bash
# Process-level
top, ps, pgrep, pidstat

# System-level
uptime, vmstat, iostat, mpstat, free

# Network
ss, ip, netstat, iftop

# Disk
df, du, lsblk, blkid
```

**Tier 2: Profiling & Tracing**
```bash
# CPU profiling
perf record/report/top
flamegraph.pl

# System calls
strace, ltrace

# I/O tracing
iotop, biosnoop (bcc)

# Network tracing
tcpdump, wireshark, tshark
```

**Tier 3: Dynamic Instrumentation**
```bash
# eBPF
bpftrace, bcc tools

# DTrace (Solaris/BSD/Linux)
dtrace

# Kernel function tracing
ftrace, trace-cmd

# Application-specific
perf-java, async-profiler (JVM)
py-spy (Python)
rbspy (Ruby)
```

### Quick Reference Commands

**CPU:**
```bash
# Top CPU consumers
perf top -g

# Flame graph
perf record -F 99 -a -g -- sleep 30 && perf script | flamegraph.pl > cpu.svg

# Per-process CPU
pidstat -u 1

# Context switches
perf stat -e sched:sched_switch -I 1000 -a
```

**Memory:**
```bash
# Top memory consumers
ps aux --sort=-%mem | head

# Slab allocator
slabtop -o

# Page faults
perf stat -e page-faults -p $(pgrep myapp) -- sleep 10

# Memory map
pmap -x $(pgrep myapp)
```

**Disk:**
```bash
# I/O by process
iotop -o

# Block I/O latency
bpftrace -e 'kprobe:blk_account_io_done { @[hist(nsecs - args->start_time)] = count(); }'

# I/O size distribution
biolatency-bpfcc -D

# Disk utilization timeline
iostat -x 1 | tee disk.log
```

**Network:**
```bash
# Bandwidth by process
nethogs

# Connection states
ss -tan state established

# Retransmits
nstat -az | grep TcpRetransSegs

# Packet drops
netstat -s | grep -i drop
```

**Locks:**
```bash
# Futex contention
perf record -e 'syscalls:sys_enter_futex' -a -g -- sleep 10

# Off-CPU profile
bpftrace -e 'profile:hz:49 { @[kstack, ustack, comm] = count(); }'

# Process states
while true; do ps -eo pid,comm,state | grep myapp; sleep 0.1; done
```

---

## Postmortem Debugging Techniques

### Node.js with mdb_v8

**Setup:**
```bash
# Ensure Node built with postmortem metadata
node --v8-options | grep post_mortem

# Generate core on fatal error
ulimit -c unlimited
export NODE_ABORT_ON_UNCAUGHT=1
```

**Analysis:**
```bash
# Open core dump
mdb /usr/bin/node core.12345

# JavaScript stack traces
> ::jsstack

# Find all JS objects
> ::findjsobjects

# Inspect specific object
> 0xaddr::jsprint

# Find objects by constructor
> ::findjsobjects -c MyClass
```

### Go with Delve

**Capture:**
```bash
# Attach to running process
dlv attach $(pgrep myapp)

# Or debug core dump
dlv core /path/to/binary /path/to/core
```

**Analysis:**
```
(dlv) goroutines
(dlv) goroutine 1 bt
(dlv) frame 0 locals
(dlv) print varname
```

### JVM with jmap/jstack

**Capture:**
```bash
# Heap dump
jmap -dump:format=b,file=heap.hprof $(pgrep java)

# Thread dump
jstack $(pgrep java) > threads.txt

# Core dump
gcore $(pgrep java)
```

**Analysis:**
```bash
# Analyze heap
jhat heap.hprof  # Web UI at :7000

# Or use Eclipse MAT for large dumps
mat_analyze.sh heap.hprof
```

---

## LLM-Specific Guidelines

### Prompting for Cantrill-Style Debugging

**Good prompt structure:**

```
I have [symptom] occurring [when/where].

Step 1: What questions should I ask the system to narrow
        down the root cause?

Step 2: What commands will answer each question?

Step 3: Based on hypothetical outputs [paste outputs],
        what should I check next?

Do NOT guess the root cause yet. Guide me through
systematic elimination.
```

**Bad prompt:**
```
My app is slow. How do I fix it?
```

### LLM Decision Tree Generation

**Prompt:**
```
Create a debugging decision tree for [symptom] using
Bryan Cantrill's question-first methodology.

Format:
- Start with observable symptom
- Each node is a question that can be answered with tools
- Include specific commands at each decision point
- Ensure each path leads to actionable root cause
- No hypotheses until possibilities are eliminated
```

### Verification Prompt

**After suggesting a fix:**
```
You suggested [fix]. Before I apply it:

1. Explain WHY this fix should work (mechanism)
2. Predict WHAT metrics should improve
3. Provide commands to VERIFY the fix worked
4. Suggest MONITORING to catch recurrence

Remember: "It went away" is not evidence.
```

---

## Real-World Case Studies

### Case Study 1: The 60% I/O Utilization Lie

**Symptom:** Cassandra benchmark performing poorly despite iostat showing 60% disk utilization.

**Initial hypothesis (wrong):** "We have I/O headroom, bottleneck must be elsewhere."

**Question-first approach:**
1. Q: Is 60% consistent or varying?
   - A: Consistent average

2. Q: What does high-frequency sampling show?
   - Command: `iostat -x 0.1 100`
   - A: Oscillates between 100% and 0%

3. Q: What's causing the oscillation?
   - Visualization: Timeline plot
   - A: Firmware bug causes periodic stalls

**Root cause:** Aggregation was hiding 100%/0% oscillation pattern.

**Fix:** Firmware update eliminated stalls.

**Lesson:** Disaggregate when stuck on averaged metrics.

### Case Study 2: The Docker Restart Trap

**Symptom:** Containers intermittently exiting with status code 139 (SIGSEGV).

**Anti-pattern response:** "Just restart the container automatically."

**Cantrill approach:**
1. Configure core dumps in container
   ```dockerfile
   RUN echo '/tmp/core.%p' > /proc/sys/kernel/core_pattern
   ```

2. Capture core on next occurrence
   ```bash
   docker cp <container>:/tmp/core.1234 ./
   ```

3. Postmortem analysis
   ```bash
   gdb /path/to/binary ./core.1234
   (gdb) bt
   (gdb) frame 3
   ```

4. Root cause: Null pointer dereference in C library

**Fix:** Update vulnerable library version.

**Verification:** Load test for 48 hours with no crashes.

**Lesson:** Restarting destroyed evidence. Core dump preserved root cause.

### Case Study 3: The Phantom GC Pause

**Symptom:** JVM application showing 5-second pauses every 10 minutes.

**Premature hypothesis:** "Must be GC, it's always GC."

**Question-first approach:**

1. Q: Are pauses correlated with GC?
   - Command: `jstat -gc $(pgrep java) 1000`
   - A: No GC during pauses

2. Q: Is application blocked on something?
   - Command: `jstack $(pgrep java)` during pause
   - A: Threads blocked in `read()`

3. Q: What file descriptor?
   - Command: `lsof -p $(pgrep java) | grep 0r`
   - A: Reading from `/proc/net/dev`

4. Q: Why is /proc/net/dev slow?
   - Investigation: High packet rate, AF_PACKET socket
   - Root cause: Monitoring agent reading /proc/net/dev

**Fix:** Move monitoring to dedicated sidecar, reduce frequency.

**Lesson:** Hypothesis "it's always GC" would have been wrong. Questions led to truth.

---

## Summary: The Cantrill Debugging Checklist

**Before investigating:**
- [ ] Capture baseline state (perf data, logs, metrics)
- [ ] Document known changes in last 24h
- [ ] Set up core dump capture (don't lose evidence)

**During investigation:**
- [ ] Ask questions, not hypotheses
- [ ] Use USE method as starting point
- [ ] Check disaggregated data when stuck on averages
- [ ] Visualize temporal patterns (flame graphs, heatmaps)
- [ ] Document elimination decisions

**When stuck:**
- [ ] Disaggregate aggregated metrics
- [ ] Check for odd behavior you dismissed
- [ ] Verify instrumentation is measuring what you think
- [ ] Take a break, come back fresh
- [ ] Ask for help (bring data, not theories)

**After resolution:**
- [ ] Explain WHY fix works (mechanism)
- [ ] Verify metrics improved
- [ ] Load test to confirm
- [ ] Update monitoring/alerts
- [ ] Write postmortem
- [ ] Share knowledge

**Never:**
- [ ] Restart without capturing state
- [ ] Accept "it went away" as resolution
- [ ] Follow folklore without verification
- [ ] Change multiple things simultaneously
- [ ] Debug based on averaged metrics alone

---

## Further Reading

### Primary Sources

- **"The Hurricane's Butterfly: Debugging Pathologically Performing Systems"** (Jane Street Talk)
  - [Jane Street Talk](https://www.janestreet.com/tech-talks/hurricanes-butterfly/)
  - [Speaker Deck Slides](https://speakerdeck.com/bcantrill/the-hurricanes-butterfly-debugging-pathologically-performing-systems)

- **"Running Aground: Debugging Docker in Production"**
  - [Speaker Deck](https://speakerdeck.com/bcantrill/running-aground-debugging-docker-in-production)

- **"Things I Learned the Hard Way"**
  - [Speaker Deck](https://speakerdeck.com/bcantrill/things-i-learned-the-hard-way)
  - [Hacker News Discussion](https://news.ycombinator.com/item?id=38216502)

- **"The Joy of Debugging"** by Bryan Cantrill and David Pacheco (Book)
  - [Amazon](https://www.amazon.com/Joy-Debugging-Bryan-Cantrill/dp/0134578724)

- **"Postmortem Debugging in Dynamic Environments"** (ACM Queue)
  - [ACM Queue Article](https://queue.acm.org/detail.cfm?id=2039361)

### DTrace and Dynamic Instrumentation

- **"Hidden in Plain Sight"** (ACM Queue) - DTrace paper
  - [ACM Queue](https://queue.acm.org/detail.cfm?id=1117401)

- **DTrace Book** - "DTrace: Dynamic Tracing in Oracle Solaris, Mac OS X, and FreeBSD"
  - [O'Reilly](https://www.oreilly.com/library/view/dtrace-dynamic-tracing/9780137061839/)

### mdb_v8 for Node.js Postmortem

- **mdb_v8 GitHub Repository**
  - [GitHub](https://github.com/joyent/mdb_v8)
  - [Usage Guide](https://github.com/TritonDataCenter/mdb_v8/blob/master/docs/usage.md)

- **"Node.js Postmortem Debugging for Fun and Production"**
  - [cjihrig.com](https://cjihrig.com/postmortem_debugging)

- **"Debugging Node.js with MDB"**
  - [Triton DataCenter](https://www.tritondatacenter.com/blog/debugging-nodejs-with-mdb)

### Visualization

- **"Visualizing Systems with Statemaps"**
  - [Speaker Deck](https://speakerdeck.com/bcantrill/visualizing-systems-with-statemaps)
  - [Statemap GitHub](https://github.com/joyent/statemap)

- **Brendan Gregg's Flame Graphs**
  - [brendangregg.com](https://www.brendangregg.com/flamegraphs.html)

- **Brendan Gregg's Heat Maps**
  - [brendangregg.com](https://www.brendangregg.com/heatmaps.html)

### Related Methodologies

- **USE Method** by Brendan Gregg
  - [brendangregg.com](https://www.brendangregg.com/usemethod.html)

- **"The Linux Scheduler: A Decade of Wasted Cores"** (referenced by Cantrill)
  - [EuroSys 2016 Paper](https://people.ece.ubc.ca/sasha/papers/eurosys16-final29.pdf)

### Podcast Appearances

- **"Bryan Cantrill - Persistence and Action"** (Developer on Fire)
  - [Episode 198](https://developeronfire.com/podcast/episode-198-bryan-cantrill-persistence-and-action)

- **"From Sun to Oxide with Bryan Cantrill"** (Changelog Interviews)
  - [Episode 592](https://changelog.com/podcast/592)

- **"Software as a Reflection of Values"** (CoRecursive)
  - [CoRecursive Podcast](https://corecursive.com/024-software-as-a-reflection-of-values-with-bryan-cantrill/)

### Performance Talks

- **Performance Talks: Episode 1 & 2 with Bryan Cantrill**
  - [Episode 1](https://edgemesh.com/blog/performance-talks-episode-1-bryan-cantrill)
  - [Episode 2](https://edgemesh.com/blog/performance-talk-episode-2-bryan-cantrill-part-2)

### InfoQ Presentations

- **"Debugging Microservices in Production"**
  - [InfoQ](https://www.infoq.com/presentations/debugging-microservices-production/)

- **"Debugging Under Fire"** (mentioned in HN discussions)
  - [Hacker News](https://news.ycombinator.com/item?id=36793801)

### Additional Context

- **Bryan Cantrill's Blog: The Observation Deck**
  - [dtrace.org](https://bcantrill.dtrace.org/)

- **Bryan Cantrill on Wikipedia**
  - [Wikipedia](https://en.wikipedia.org/wiki/Bryan_Cantrill)

---

**Document Version:** 1.0
**Last Updated:** 2026-02-03
**Maintainer:** Based on public talks, papers, and documentation by Bryan Cantrill
**License:** Educational use, cite original sources
