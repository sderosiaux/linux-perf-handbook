# VDSO/Clock Source Quick Troubleshooting

LLM-friendly detection and fix commands for time-related performance issues on cloud VMs.

## One-Liner Health Check

```bash
# Run this first - shows current state and recommendations
bash -c 'CS=$(cat /sys/devices/system/clocksource/clocksource0/current_clocksource); echo "Clock source: $CS"; [ "$CS" = "xen" ] && echo "⚠️  CRITICAL: Switch to tsc (losing 15%+ CPU)" || [ "$CS" = "tsc" ] && echo "✓ Optimal" || echo "⚠️  Consider switching to tsc"; grep -q vdso /proc/self/maps && echo "✓ vDSO present" || echo "✗ vDSO missing"; grep -q constant_tsc /proc/cpuinfo && echo "✓ TSC supported" || echo "✗ TSC not available"'
```

## Detection Sequence

### 1. Check Clock Source (No Root Required)

```bash
cat /sys/devices/system/clocksource/clocksource0/current_clocksource
```

**Interpretation:**
- `xen` → **CRITICAL**: Switch immediately (losing 15-77% performance)
- `hpet` or `acpi_pm` → **BAD**: Switch to tsc or kvm-clock
- `kvm-clock` → **OK**: Acceptable, but tsc is better
- `tsc` → **GOOD**: Optimal

### 2. Verify vDSO Active (Requires Root)

```bash
# Quick test: count syscalls over 5 seconds
sudo timeout 5 perf stat -e 'syscalls:sys_enter_clock_gettime' -e 'syscalls:sys_enter_gettimeofday' -a 2>&1 | grep -E 'clock_gettime|gettimeofday'
```

**Interpretation:**
- 0-100 total → vDSO working
- \> 1000 → vDSO broken or disabled

### 3. Check TSC Support (No Root Required)

```bash
grep -E 'constant_tsc|nonstop_tsc' /proc/cpuinfo | head -1
```

**Interpretation:**
- Output present → TSC supported, safe to switch
- No output → TSC not reliable, stay with kvm-clock

## Fix Commands

### Immediate Fix (Lost on Reboot)

```bash
# For AWS Xen, old instances
echo tsc | sudo tee /sys/devices/system/clocksource/clocksource0/current_clocksource

# Verify
cat /sys/devices/system/clocksource/clocksource0/current_clocksource

# Check for issues
dmesg | grep -i clocksource | tail -5
```

### Permanent Fix (Survives Reboot)

```bash
# Add to GRUB
sudo sed -i.bak 's/GRUB_CMDLINE_LINUX="\(.*\)"/GRUB_CMDLINE_LINUX="\1 clocksource=tsc tsc=reliable"/' /etc/default/grub

# Rebuild config
# Ubuntu/Debian:
sudo update-grub
# RHEL/CentOS/Amazon Linux:
sudo grub2-mkconfig -o /boot/grub2/grub.cfg

# Reboot to apply
sudo reboot
```

## Validation Commands

### Before/After Benchmark

```bash
# Create benchmark (copy/paste entire block)
cat > /tmp/bench_time.c << 'EOF'
#include <stdio.h>
#include <sys/time.h>
#include <time.h>
#include <stdint.h>
#define ITERATIONS 1000000
static inline uint64_t rdtsc(void) {
    uint32_t lo, hi;
    __asm__ __volatile__ ("rdtsc" : "=a" (lo), "=d" (hi));
    return ((uint64_t)hi << 32) | lo;
}
int main() {
    struct timeval tv;
    uint64_t start = rdtsc();
    for (int i = 0; i < ITERATIONS; i++) gettimeofday(&tv, NULL);
    uint64_t end = rdtsc();
    printf("%.1f cycles/call\n", (double)(end - start) / ITERATIONS);
    return 0;
}
EOF

# Compile and run
gcc -O2 -o /tmp/bench_time /tmp/bench_time.c && /tmp/bench_time
```

**Interpretation:**
- < 100 cycles → vDSO working, optimal
- 100-500 cycles → Marginal performance
- \> 500 cycles → Full syscall, fix clock source

### Continuous Monitoring (Requires Root)

```bash
# Monitor for 60 seconds
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_clock_gettime { @[comm] = count(); } interval:s:10 { print(@); } END { clear(@); }' &
BPID=$!
sleep 60
sudo kill $BPID
```

High counts = applications making syscalls (vDSO broken)

## Platform-Specific Quick Fixes

### AWS EC2

```bash
# Detect generation
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" 2>/dev/null)
ITYPE=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-type 2>/dev/null)
echo "Instance type: $ITYPE"

# Xen instances (i3, r4, m4, older): MUST switch to tsc
# Nitro instances (5th gen+): Already good, but tsc still better
echo tsc | sudo tee /sys/devices/system/clocksource/clocksource0/current_clocksource
```

### GCP Compute Engine

```bash
# Usually already optimal, verify
cat /sys/devices/system/clocksource/clocksource0/current_clocksource
# Should be kvm-clock or tsc

# If not tsc, switch
echo tsc | sudo tee /sys/devices/system/clocksource/clocksource0/current_clocksource
```

### Azure VMs

```bash
# Check current
cat /sys/devices/system/clocksource/clocksource0/current_clocksource

# Azure switched to tsc recently, if not set:
echo tsc | sudo tee /sys/devices/system/clocksource/clocksource0/current_clocksource
```

## Troubleshooting

### Issue: TSC Reverts to kvm-clock

```bash
# Check dmesg
dmesg | grep -i "tsc unstable"

# If unstable, force it (bypass checks)
# Edit /etc/default/grub:
# GRUB_CMDLINE_LINUX="... clocksource=tsc tsc=reliable"
sudo nano /etc/default/grub
sudo update-grub
sudo reboot
```

### Issue: High Syscalls Despite TSC

```bash
# Check for seccomp/audit interference
grep Seccomp /proc/$(pgrep -f your_app)/status
sudo auditctl -l | grep -E 'clock|time'

# If audit rules present, disable for time syscalls
sudo auditctl -d exit,always -S clock_gettime -S gettimeofday
```

### Issue: Old glibc (No vDSO Support)

```bash
# Check glibc version
ldd --version
# vDSO support added in 2.3+ (2002)
# If ancient glibc, upgrade OS or recompile app
```

## Expected Impact

### Database Workloads

- **Before (xen)**: 65% CPU, P99 45ms
- **After (tsc)**: 52% CPU, P99 28ms
- **Improvement**: 20% CPU freed, 38% latency reduction

### Metrics/Observability

- **Before**: 15% CPU on time syscalls
- **After**: 2% CPU on time operations
- **Improvement**: 13% CPU freed

### High-Frequency Applications

- **Before**: 1500ns timestamp overhead
- **After**: 8ns timestamp overhead
- **Improvement**: 187x faster

## When NOT to Switch

1. **Kernel reports TSC unstable**: Some hypervisors have buggy TSC
2. **Frequent CPU hotplug**: TSC may desync
3. **Live migration active**: TSC not preserved across migration
4. **Older CPUs without constant_tsc**: Check /proc/cpuinfo first

In these cases, use `kvm-clock` (verify vDSO support with perf stat).

## Red Flags

**Switch immediately if:**
- Clock source is `xen` or `hpet`
- perf stat shows > 10k time syscalls/sec
- Application logs excessive time measurement overhead
- Top 3 perf record samples are in vdso or vsyscall

**Investigate if:**
- Clock source is `tsc` but benchmark shows > 500 cycles
- vDSO present but syscalls still counted
- Different performance across "identical" VMs (clock source mismatch)

## Copy-Paste Full Check

```bash
#!/bin/bash
# Complete health check and fix (requires root for perf)

echo "=== Clock Source Health Check ==="
CS=$(cat /sys/devices/system/clocksource/clocksource0/current_clocksource)
echo "Current: $CS"

if [ "$CS" = "xen" ] || [ "$CS" = "hpet" ] || [ "$CS" = "acpi_pm" ]; then
    echo "Status: ⚠️  CRITICAL - Poor clock source"
    FIX_NEEDED=1
elif [ "$CS" = "kvm-clock" ]; then
    echo "Status: ⚠️  OK - But tsc is better"
    FIX_NEEDED=0
else
    echo "Status: ✓ Optimal"
    FIX_NEEDED=0
fi

if grep -q constant_tsc /proc/cpuinfo; then
    echo "TSC: ✓ Supported"
    TSC_OK=1
else
    echo "TSC: ✗ Not available"
    TSC_OK=0
fi

if grep -q vdso /proc/self/maps; then
    echo "vDSO: ✓ Mapped"
else
    echo "vDSO: ✗ Missing (kernel issue)"
fi

if [ $FIX_NEEDED -eq 1 ] && [ $TSC_OK -eq 1 ]; then
    echo ""
    echo "=== Recommended Action ==="
    echo "Switch to TSC immediately:"
    echo "  echo tsc | sudo tee /sys/devices/system/clocksource/clocksource0/current_clocksource"
    echo ""
    echo "Make permanent (add to /etc/default/grub):"
    echo "  GRUB_CMDLINE_LINUX=\"... clocksource=tsc tsc=reliable\""
fi
```

## See Also

- Full chapter: **18-vdso-clock-source-tuning.md**
- Related: **05-performance-profiling.md** (perf usage)
- Related: **13-latency-analysis.md** (coordinated omission)
