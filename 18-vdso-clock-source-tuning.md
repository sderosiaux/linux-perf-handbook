# VDSO & Clock Source Performance on Cloud VMs

Critical performance issue: virtualized clock sources can cause 15-77% CPU overhead and make time-related syscalls 442x slower. This chapter covers detection, benchmarking, and fixes for AWS, GCP, and Azure.

## Executive Summary

**The Problem**: Xen-based hypervisors (older AWS EC2) and misconfigured KVM instances disable the vDSO fast path for time-related system calls. `gettimeofday()` and `clock_gettime()` become full kernel syscalls instead of fast user-space reads.

**Impact**:
- 77% slower syscall execution (15ns → 80ns minimum, often 1000ns+)
- 15%+ CPU overhead on time-heavy workloads (databases, metrics, logging)
- Single-digit millisecond latency becomes double-digit at P99

**Solution**: Switch to TSC clock source on AWS. Verify kvmclock/tsc on GCP/Azure. Benchmark before and after.

## Understanding the Stack

### vDSO (Virtual Dynamic Shared Object)

The vDSO is a kernel-provided shared library mapped into every process. It contains user-space implementations of frequently-called syscalls.

```
Normal syscall path:        vDSO-accelerated path:
  gettimeofday()              gettimeofday()
       |                            |
  syscall instruction           vDSO function
       |                            |
  kernel entry                  read shared page
  context switch                    |
  CPU mode switch              return immediately
  read clocksource             (no kernel entry)
       |
  context switch
  CPU mode switch
       |
  return to userland

  ~80-1000ns                    ~15-20ns
```

### Clock Sources

Linux supports multiple clock sources with different performance characteristics:

| Clock Source | Type | Latency | vDSO Support | Notes |
|--------------|------|---------|--------------|-------|
| `tsc` | Hardware counter | 15-20ns | Yes | Best. Requires constant TSC. |
| `kvm-clock` | Paravirtual | 20-30ns | Yes (recent) | GCP/Azure default. Good on Nitro. |
| `xen` | Paravirtual | 80-1000ns | **No** | Requires hypercall. Terrible performance. |
| `hpet` | Hardware timer | 1000ns+ | No | Legacy fallback. Avoid. |
| `acpi_pm` | ACPI timer | 1000ns+ | No | Legacy fallback. Avoid. |

### What Makes TSC Special

The Time Stamp Counter (TSC) is a CPU register incremented on every clock cycle. Reading it requires a single `rdtsc` instruction (~5 cycles).

**Requirements for TSC**:
1. **Constant TSC**: Frequency doesn't change with CPU frequency scaling
2. **Invariant TSC**: Frequency same across all cores
3. **Synchronized**: All cores have consistent values

Modern CPUs (Intel Nehalem+, AMD Bulldozer+) provide constant, invariant TSC. Older hypervisors (Xen) don't expose it to guests.

## Cloud Platform Specifics

### AWS EC2

**Xen-based instances** (i3, r4, older generations):
- Default: `xen` clock source
- vDSO: **Disabled** for time syscalls
- Impact: 77% slower, 15%+ CPU overhead

**Nitro-based instances** (5th gen+: m5, c5, r5, etc.):
- Default: `kvm-clock` with vDSO support
- Performance: Good (20-30ns)
- Recommendation: Still switch to `tsc` for best performance

**Detection**:
```bash
# Check instance generation
ec2-metadata --instance-type

# Xen instances
aws ec2 describe-instances --instance-ids $(ec2-metadata --instance-id | cut -d ' ' -f2) \
  --query 'Reservations[].Instances[].Hypervisor' --output text
# Output: "xen" = old, "nitro" = new
```

### GCP Compute Engine

**All generations**:
- Default: `kvm-clock` with vDSO support (modern kernels)
- Recommendation: Verify vDSO active, otherwise switch to `tsc`
- Additional: Uses `ptp_kvm` for sub-millisecond time sync

**Detection**:
```bash
# Check if PTP available
ls /dev/ptp*
```

### Azure VMs

**Recent hypervisor**:
- Default: Transitioning to `tsc` clock source
- Legacy: May still use `hyperv_clocksource`
- Recommendation: Verify `tsc` active

**Detection**:
```bash
# Azure-specific metadata
curl -H Metadata:true "http://169.254.169.254/metadata/instance?api-version=2021-02-01" | jq .compute.vmSize
```

## Detection Commands

### 1. Check Current Clock Source

```bash
# Primary method
cat /sys/devices/system/clocksource/clocksource0/current_clocksource
# Output: tsc (good), xen (bad), kvm-clock (ok), hpet (terrible)

# Available sources
cat /sys/devices/system/clocksource/clocksource0/available_clocksource
# Example: kvm-clock tsc hpet acpi_pm

# Per-CPU clocksource info
grep -r . /sys/devices/system/clocksource/clocksource0/
```

### 2. Verify vDSO Usage

**Method 1: strace (confirms vDSO by absence)**
```bash
# If vDSO active, these syscalls won't appear
strace -c -e trace=gettimeofday,clock_gettime sleep 1
# Output should show 0 calls or very few

# Detailed trace (should show NO gettimeofday if vDSO works)
timeout 1 strace -e trace=gettimeofday,clock_gettime myapp 2>&1 | wc -l
```

**Method 2: perf stat**
```bash
# Count syscalls (should be near zero if vDSO active)
perf stat -e 'syscalls:sys_enter_clock_gettime' -e 'syscalls:sys_enter_gettimeofday' \
  -- timeout 5 myapp

# Example output:
#         0      syscalls:sys_enter_clock_gettime  # vDSO working
#    50,000      syscalls:sys_enter_clock_gettime  # vDSO broken
```

**Method 3: Check vDSO mapping**
```bash
# Every process should have vDSO mapped
grep vdso /proc/self/maps
# Example: 7ffff7ffa000-7ffff7ffc000 r-xp 00000000 00:00 0 [vdso]

# Confirm vDSO has clock functions
LD_SHOW_AUXV=1 /bin/true | grep AT_SYSINFO_EHDR
```

### 3. Check TSC Capabilities

```bash
# Check if CPU supports constant TSC
grep -E 'constant_tsc|nonstop_tsc' /proc/cpuinfo
# constant_tsc: frequency constant with CPU scaling
# nonstop_tsc: continues counting in deep sleep

# Check TSC frequency
dmesg | grep -i tsc
# Look for: "tsc: Refined TSC clocksource calibration"
# Warning signs: "TSC found unstable", "Marking TSC unstable"

# Kernel detected TSC quality
cat /sys/devices/system/clocksource/clocksource0/current_clocksource
# If not "tsc", kernel decided it's unreliable
```

### 4. Quick Health Check

```bash
#!/bin/bash
# vdso_health_check.sh

echo "=== Clock Source Health Check ==="

CURRENT=$(cat /sys/devices/system/clocksource/clocksource0/current_clocksource)
echo "Current clock source: $CURRENT"

if [ "$CURRENT" != "tsc" ] && [ "$CURRENT" != "kvm-clock" ]; then
    echo "⚠️  WARNING: Using slow clock source ($CURRENT)"
    echo "   Expected: tsc or kvm-clock"
fi

# Check vDSO presence
if grep -q vdso /proc/self/maps; then
    echo "✓ vDSO mapped"
else
    echo "⚠️  WARNING: vDSO not found"
fi

# Check TSC support
if grep -q constant_tsc /proc/cpuinfo; then
    echo "✓ constant_tsc supported"
else
    echo "⚠️  WARNING: constant_tsc not available"
fi

# Quick syscall count (needs root)
if [ "$EUID" -eq 0 ]; then
    COUNT=$(timeout 1 perf stat -e 'syscalls:sys_enter_*' -x, -p $$ 2>&1 | grep syscalls | awk -F, '{sum+=$1} END {print sum}')
    echo "Syscall rate: $COUNT/sec (lower is better)"
fi

echo ""
echo "=== Recommendation ==="
if [ "$CURRENT" = "xen" ]; then
    echo "CRITICAL: Switch to tsc immediately"
    echo "Command: echo tsc | sudo tee /sys/devices/system/clocksource/clocksource0/current_clocksource"
elif [ "$CURRENT" = "hpet" ] || [ "$CURRENT" = "acpi_pm" ]; then
    echo "WARNING: Switch to tsc or kvm-clock"
else
    echo "Clock source acceptable. Benchmark to verify vDSO active."
fi
```

## Benchmark Methodology

### Microbenchmark: Direct Syscall Overhead

```c
// benchmark_gettimeofday.c
// Compile: gcc -O2 -o bench_gettimeofday benchmark_gettimeofday.c
#include <stdio.h>
#include <sys/time.h>
#include <time.h>
#include <stdint.h>

#define ITERATIONS 10000000

static inline uint64_t rdtsc(void) {
    uint32_t lo, hi;
    __asm__ __volatile__ ("rdtsc" : "=a" (lo), "=d" (hi));
    return ((uint64_t)hi << 32) | lo;
}

int main() {
    struct timeval tv;
    struct timespec ts;
    uint64_t start, end;

    // Warm up
    for (int i = 0; i < 1000; i++) {
        gettimeofday(&tv, NULL);
        clock_gettime(CLOCK_MONOTONIC, &ts);
    }

    // Benchmark gettimeofday
    start = rdtsc();
    for (int i = 0; i < ITERATIONS; i++) {
        gettimeofday(&tv, NULL);
    }
    end = rdtsc();
    printf("gettimeofday: %.2f cycles/call\n",
           (double)(end - start) / ITERATIONS);

    // Benchmark clock_gettime
    start = rdtsc();
    for (int i = 0; i < ITERATIONS; i++) {
        clock_gettime(CLOCK_MONOTONIC, &ts);
    }
    end = rdtsc();
    printf("clock_gettime(MONOTONIC): %.2f cycles/call\n",
           (double)(end - start) / ITERATIONS);

    // Benchmark CLOCK_REALTIME
    start = rdtsc();
    for (int i = 0; i < ITERATIONS; i++) {
        clock_gettime(CLOCK_REALTIME, &ts);
    }
    end = rdtsc();
    printf("clock_gettime(REALTIME): %.2f cycles/call\n",
           (double)(end - start) / ITERATIONS);

    return 0;
}
```

**Expected results:**

| Clock Source | vDSO Status | Cycles/Call | Nanoseconds (@3GHz) |
|--------------|-------------|-------------|---------------------|
| `tsc` | Active | 30-50 | 10-17ns |
| `kvm-clock` | Active | 60-90 | 20-30ns |
| `xen` | Disabled | 3000-5000 | 1000-1700ns |
| `hpet` | Disabled | 3000+ | 1000ns+ |

**Interpretation**:
- < 100 cycles: vDSO working
- 100-500 cycles: vDSO partially working or slow clock source
- \> 500 cycles: Full syscall (vDSO disabled or bad clock source)

### Application-Level Impact

```bash
# Measure database workload impact
# Before: xen clocksource
sysbench oltp_read_only --tables=10 --table-size=100000 \
  --mysql-host=localhost --mysql-user=bench --mysql-password=pass \
  --time=60 run | tee before.txt

# Switch to tsc
echo tsc | sudo tee /sys/devices/system/clocksource/clocksource0/current_clocksource

# After: tsc clocksource
sysbench oltp_read_only --tables=10 --table-size=100000 \
  --mysql-host=localhost --mysql-user=bench --mysql-password=pass \
  --time=60 run | tee after.txt

# Compare
echo "=== QPS Improvement ==="
echo "Before: $(grep 'queries:' before.txt)"
echo "After:  $(grep 'queries:' after.txt)"
```

### Production Measurement

```bash
# Count time-related syscalls in production
# (requires kernel tracing enabled)

# Method 1: perf
sudo perf top -e syscalls:sys_enter_clock_gettime -e syscalls:sys_enter_gettimeofday
# High counts = vDSO broken

# Method 2: bpftrace (overhead-aware)
sudo bpftrace -e '
  BEGIN { @start = nsecs; }
  tracepoint:syscalls:sys_enter_clock_gettime,
  tracepoint:syscalls:sys_enter_gettimeofday {
    @[probe] = count();
  }
  interval:s:10 {
    $elapsed = (nsecs - @start) / 1e9;
    printf("\n=== After %.0f seconds ===\n", $elapsed);
    print(@);
    if ($elapsed >= 60) { exit(); }
  }
'

# If counts > 10k/sec, investigate clock source
```

## Fixing Clock Source Issues

### Temporary Fix (Runtime)

```bash
# Check available sources
cat /sys/devices/system/clocksource/clocksource0/available_clocksource

# Switch to TSC (requires root)
echo tsc | sudo tee /sys/devices/system/clocksource/clocksource0/current_clocksource

# Verify
cat /sys/devices/system/clocksource/clocksource0/current_clocksource

# Check kernel logs for issues
dmesg | grep -i clocksource
# Warning signs:
#   "Switched to clocksource tsc"  (good)
#   "clocksource tsc unstable"     (bad - will revert)
#   "Marking TSC unstable"         (bad - CPU doesn't support)
```

### Permanent Fix (Boot Parameter)

**Method 1: Kernel command line**

```bash
# Edit GRUB configuration
sudo vim /etc/default/grub

# Add to GRUB_CMDLINE_LINUX:
GRUB_CMDLINE_LINUX="... clocksource=tsc tsc=reliable"

# Rebuild GRUB config
# Ubuntu/Debian:
sudo update-grub
# RHEL/CentOS:
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# Reboot
sudo reboot
```

**Method 2: systemd service (if kernel parameter doesn't work)**

```bash
# Create service
sudo tee /etc/systemd/system/clocksource-tsc.service > /dev/null <<'EOF'
[Unit]
Description=Set TSC as clock source
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'echo tsc > /sys/devices/system/clocksource/clocksource0/current_clocksource'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable clocksource-tsc.service
sudo systemctl start clocksource-tsc.service

# Verify
systemctl status clocksource-tsc.service
```

### AWS-Specific: EC2 User Data

```bash
#!/bin/bash
# Add to EC2 User Data for automatic configuration

# Set clock source to TSC
echo "tsc" > /sys/devices/system/clocksource/clocksource0/current_clocksource

# Verify and log
CURRENT=$(cat /sys/devices/system/clocksource/clocksource0/current_clocksource)
logger -t clocksource "Set clock source to: $CURRENT"

# Make permanent
if ! grep -q "clocksource=tsc" /etc/default/grub; then
    sed -i 's/GRUB_CMDLINE_LINUX="\(.*\)"/GRUB_CMDLINE_LINUX="\1 clocksource=tsc tsc=reliable"/' /etc/default/grub
    update-grub
fi
```

### GCP-Specific: Configure PTP

```bash
# Install chrony with PTP support
sudo apt-get install -y chrony

# Configure chrony for ptp_kvm
sudo tee /etc/chrony/chrony.conf > /dev/null <<'EOF'
# Use PTP hardware clock
refclock PHC /dev/ptp0 poll 2 dpoll -2 offset 0

# Allow larger clock adjustments
makestep 1.0 -1

# Log clock changes
logdir /var/log/chrony
log tracking measurements statistics
EOF

# Restart chrony
sudo systemctl restart chrony

# Verify time sync
chronyc tracking
# Look for: "Reference ID" showing "PTP0"
```

## Troubleshooting

### TSC Marked Unstable

**Symptom**: After setting `tsc`, kernel reverts to `kvm-clock` or `hpet`.

```bash
dmesg | grep -i tsc
# "Marking TSC unstable due to check_tsc_sync_source failed"
```

**Cause**: CPUs not synchronized or frequency scaling issues.

**Solution 1**: Force TSC with `tsc=reliable` boot parameter (bypasses stability checks).

**Solution 2**: Disable CPU frequency scaling:
```bash
# Set performance governor
echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

**Solution 3**: Accept `kvm-clock` if recent kernel (vDSO support added in 4.9+).

### vDSO Present But Not Used

**Symptom**: vDSO mapped but syscalls still go to kernel.

```bash
# vDSO exists
grep vdso /proc/self/maps
# Output: 7ffff7ffa000-7ffff7ffc000 r-xp 00000000 00:00 0 [vdso]

# But syscalls counted
sudo perf stat -e syscalls:sys_enter_clock_gettime -- sleep 1
# 1000+ syscalls (should be 0-10)
```

**Cause**: Application compiled with `--no-vdso` or old glibc.

**Debug**:
```bash
# Check glibc version (vDSO support added in 2.3+)
ldd --version

# Check binary's vDSO usage
readelf -d /path/to/app | grep VDSO
# Should show vDSO symbols

# Trace library calls
LD_DEBUG=libs ./myapp 2>&1 | grep vdso
```

**Solution**: Recompile with modern glibc or update system libraries.

### High Syscall Overhead Despite TSC

**Symptom**: Clock source is `tsc` but syscalls still slow.

**Cause**: Seccomp or audit framework intercepting syscalls.

**Debug**:
```bash
# Check for seccomp
grep Seccomp /proc/PID/status
# Seccomp: 2 (strict filtering active)

# Check audit rules
auditctl -l | grep -E 'clock_gettime|gettimeofday'
```

**Solution**: Disable audit rules for time syscalls or adjust seccomp filters.

### Cloud Platform Detection Failed

**Symptom**: Script doesn't detect cloud platform correctly.

**Universal detection**:
```bash
# Check DMI info
sudo dmidecode -s system-manufacturer
# AWS: "Amazon EC2"
# Azure: "Microsoft Corporation"
# GCP: "Google"

# Fallback to metadata services
# AWS
timeout 1 curl -s http://169.254.169.254/latest/meta-data/instance-id && echo "AWS"

# Azure
timeout 1 curl -s -H Metadata:true "http://169.254.169.254/metadata/instance?api-version=2021-02-01" && echo "Azure"

# GCP
timeout 1 curl -s -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/id && echo "GCP"
```

## Monitoring and Validation

### Production Metrics

**Add to your monitoring stack:**

```bash
# Prometheus node_exporter metric
node_timex_pps_frequency_hertz
node_timex_sync_status

# Custom metric: clock source
echo "# HELP node_clock_source Current kernel clock source
# TYPE node_clock_source gauge" > /var/lib/node_exporter/clocksource.prom
echo "node_clock_source{source=\"$(cat /sys/devices/system/clocksource/clocksource0/current_clocksource)\"} 1" >> /var/lib/node_exporter/clocksource.prom
```

**eBPF continuous monitoring:**

```bash
# Track time syscall overhead (bpftrace)
sudo bpftrace -e '
  tracepoint:syscalls:sys_enter_clock_gettime { @start[tid] = nsecs; }
  tracepoint:syscalls:sys_exit_clock_gettime /@start[tid]/ {
    $dur = nsecs - @start[tid];
    @latency_ns = hist($dur);
    delete(@start[tid]);
  }
  interval:s:30 {
    printf("\n=== clock_gettime() latency histogram ===\n");
    print(@latency_ns);
    clear(@latency_ns);
  }
'
# Healthy: P99 < 100ns
# Warning: P99 > 500ns
# Critical: P99 > 1000ns
```

### Regression Testing

**Include in CI/CD:**

```bash
#!/bin/bash
# test_clocksource.sh - CI check for clock source configuration

EXPECTED_SOURCE="${1:-tsc}"
CURRENT=$(cat /sys/devices/system/clocksource/clocksource0/current_clocksource)

if [ "$CURRENT" != "$EXPECTED_SOURCE" ]; then
    echo "FAIL: Clock source is $CURRENT, expected $EXPECTED_SOURCE"
    exit 1
fi

# Benchmark gettimeofday
CYCLES=$(./bench_gettimeofday | grep gettimeofday | awk '{print $2}')
THRESHOLD=100

if (( $(echo "$CYCLES > $THRESHOLD" | bc -l) )); then
    echo "FAIL: gettimeofday too slow: $CYCLES cycles (threshold: $THRESHOLD)"
    exit 1
fi

echo "PASS: Clock source $CURRENT, gettimeofday $CYCLES cycles"
exit 0
```

## Real-World Impact Examples

### Case Study 1: Database on AWS i3

**Before (xen clock source)**:
- PostgreSQL P99 query latency: 45ms
- CPU utilization: 65%
- `gettimeofday()` overhead: 1200ns/call
- 120k syscalls/sec

**After (tsc clock source)**:
- PostgreSQL P99 query latency: 28ms (-38%)
- CPU utilization: 52% (-20%)
- `gettimeofday()` overhead: 18ns/call (-98%)
- 0 syscalls/sec (vDSO active)

**Commands used**:
```bash
# Detection
cat /sys/devices/system/clocksource/clocksource0/current_clocksource
# xen

sudo perf top -e syscalls:sys_enter_clock_gettime
# 120,000 calls/sec

# Fix
echo tsc | sudo tee /sys/devices/system/clocksource/clocksource0/current_clocksource

# Validation
./bench_gettimeofday
# Before: 3600 cycles/call
# After:   54 cycles/call
```

### Case Study 2: Metrics Collection Service

**Before**:
- Prometheus scrapes: 50k/sec
- Each scrape calls `clock_gettime()` 10x
- 500k syscalls/sec
- 15% CPU on time syscalls alone

**After**:
- Same workload
- 0 time-related syscalls (vDSO active)
- 2% CPU on time operations (-87%)
- 13% CPU freed for actual work

### Case Study 3: High-Frequency Trading

**Before (hpet)**:
- Timestamp overhead: 1500ns
- Latency budget: 10μs
- 15% of budget spent on timestamps

**After (tsc + rdtsc)**:
- Timestamp overhead: 8ns
- Latency budget: 10μs
- 0.08% of budget on timestamps
- Enabled sub-microsecond SLAs

## Quick Reference: LLM Troubleshooting

When diagnosing time-related performance issues, run this sequence:

```bash
# 1. Identify clock source
cat /sys/devices/system/clocksource/clocksource0/current_clocksource
# BAD: xen, hpet, acpi_pm
# OK: kvm-clock (if recent kernel)
# BEST: tsc

# 2. Count time syscalls (requires root)
sudo timeout 5 perf stat -e 'syscalls:sys_enter_clock_gettime' \
  -e 'syscalls:sys_enter_gettimeofday' -p $(pgrep -f your_app_name)
# > 1000/sec = problem

# 3. Benchmark overhead
gcc -O2 -o /tmp/bench bench_gettimeofday.c && /tmp/bench
# > 500 cycles = vDSO disabled

# 4. Fix (AWS Xen instances)
echo tsc | sudo tee /sys/devices/system/clocksource/clocksource0/current_clocksource

# 5. Validate
dmesg | grep -i clocksource | tail -5
# Should show "Switched to clocksource tsc"
# No "unstable" messages

# 6. Re-benchmark
/tmp/bench
# Should show < 100 cycles/call
```

## Platform-Specific Quick Fixes

**AWS EC2 (Xen)**:
```bash
echo tsc | sudo tee /sys/devices/system/clocksource/clocksource0/current_clocksource
```

**AWS EC2 (Nitro)**:
```bash
# Already good, but can optimize:
echo tsc | sudo tee /sys/devices/system/clocksource/clocksource0/current_clocksource
```

**GCP Compute Engine**:
```bash
# Usually good. Verify:
cat /sys/devices/system/clocksource/clocksource0/current_clocksource
# If not tsc: echo tsc | sudo tee ...
```

**Azure VMs**:
```bash
# Check current:
cat /sys/devices/system/clocksource/clocksource0/current_clocksource
# Should be tsc. If not:
echo tsc | sudo tee /sys/devices/system/clocksource/clocksource0/current_clocksource
```

## Key Takeaways

1. **Always check clock source on new VMs**. Default != optimal.

2. **`xen` clock source is a 15%+ CPU tax**. Switch to `tsc` immediately on AWS i3/older generations.

3. **Benchmark with `perf stat` + syscall counters**. Microbenchmarks can lie about real-world impact.

4. **vDSO presence != vDSO usage**. Verify with strace or perf that syscalls aren't happening.

5. **TSC requires constant_tsc CPU flag**. Check `/proc/cpuinfo` before forcing.

6. **Make changes permanent**. Runtime changes reset on reboot.

7. **Test in production before rollout**. Some hypervisors have buggy TSC implementations.

## References

- Brendan Gregg: "The Speed of Time" (2021) - https://www.brendangregg.com/blog/2021-09-26/the-speed-of-time.html
- Packagecloud: "Two system calls are ~77% slower on AWS EC2" - https://blog.packagecloud.io/system-calls-are-much-slower-on-ec2/
- Heap: "Running a database on EC2? Your clock could be slowing you down" - https://www.heap.io/blog/running-a-database-on-ec2-your-clock-could-be-slowing-you-down
- AWS Well-Architected Labs: Changing clock type on Xen EC2 - https://wellarchitectedlabs.com/performance-efficiency/100_labs/100_clock_source_performance/
- Linux kernel docs: Timekeeping Virtualization - https://docs.kernel.org/virt/kvm/x86/timekeeping.html
- Arkanis: Measurements of system call performance and overhead - https://arkanis.de/weblog/2017-01-05-measurements-of-system-call-performance-and-overhead/

## See Also

- **16-scheduler-interrupts.md**: TSC for interrupt timestamping
- **05-performance-profiling.md**: Using `perf` for syscall analysis
- **06-ebpf-tracing.md**: eBPF-based time syscall monitoring
- **13-latency-analysis.md**: Coordinated omission and time measurement artifacts
