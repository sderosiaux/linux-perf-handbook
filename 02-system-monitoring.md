# System Monitoring

Tools for CPU, memory, process, and system-wide monitoring.

## Process Monitoring

### top
```bash
top                            # Basic view
top -c                         # Show command line
top -H                         # Thread view
top -p 1234,5678               # Specific PIDs
top -u username                # User's processes
top -b -n 1                    # Batch mode (scripting)
```
**Interactive:**
- `1` - Per-CPU view
- `M` - Sort by memory
- `P` - Sort by CPU
- `k` - Kill process
- `f` - Field selection

### htop
```bash
htop                           # Interactive view
htop -u username               # Filter by user
htop -p 1234                   # Specific PID
htop -t                        # Tree view
htop -d 10                     # 1 second delay
```
**Interactive:**
- `F5` - Tree view
- `F6` - Sort by column
- `F9` - Kill process
- `Space` - Tag process
- `\` - Filter

### ps
```bash
ps aux                         # BSD style, all processes
ps -ef                         # System V style
ps fauxww                      # Full command + tree
ps -eo pid,ppid,%cpu,%mem,cmd  # Custom columns
ps -o pid,ni,pri,cmd -p 1234   # Nice/priority
ps --forest                    # Process tree
ps -T -p 1234                  # Threads of PID
```

### pstree
```bash
pstree                         # Process tree
pstree -p                      # Show PIDs
pstree -a                      # Show arguments
pstree -u                      # Show UID transitions
pstree -H 1234                 # Highlight PID
```

## CPU Monitoring

### mpstat
```bash
mpstat                         # Average all CPUs
mpstat -P ALL 1                # All cores, 1s interval
mpstat -P 0,1 1                # Cores 0 and 1
mpstat -I ALL 1                # Interrupt stats
```
**Output columns:**
- `%usr` - User mode
- `%sys` - Kernel mode
- `%iowait` - I/O wait
- `%idle` - Idle time

### vmstat
```bash
vmstat 1                       # 1 second interval
vmstat 1 10                    # 10 samples
vmstat -w 1                    # Wide output
vmstat -d                      # Disk stats
vmstat -s                      # Memory stats summary
vmstat -a                      # Active/inactive memory
```
**Key columns:**
- `r` - Runnable processes
- `b` - Blocked processes
- `si/so` - Swap in/out
- `bi/bo` - Block I/O
- `wa` - Wait time
- `st` - Steal time (VM)

### uptime / w
```bash
uptime                         # Load average 1/5/15 min
w                              # Who's logged in + load
```

### pidstat
```bash
pidstat 1                      # Per-process CPU
pidstat -d 1                   # Per-process disk
pidstat -r 1                   # Per-process memory
pidstat -t 1                   # Thread-level
pidstat -p 1234 1              # Specific PID
```

## Memory Monitoring

### free
```bash
free -h                        # Human readable
free -w                        # Wide (buffers separate)
free -s 1                      # Update every second
free -t                        # Show totals
```
**Key columns:**
- `total` - Total RAM
- `used` - Used (excluding buffers/cache)
- `buff/cache` - Buffer + cache memory
- `available` - Actually available

### /proc/meminfo
```bash
cat /proc/meminfo              # Detailed memory
grep -E 'MemTotal|MemFree|MemAvailable|Buffers|Cached|Swap' /proc/meminfo
```

### smem
```bash
smem                           # Memory by process
smem -t                        # With totals
smem -k                        # Show units
smem -p                        # Percentages
smem -c pss -s pss             # Sort by PSS
```
**Columns:**
- `USS` - Unique Set Size (process alone)
- `PSS` - Proportional Set Size (shared divided)
- `RSS` - Resident Set Size (includes shared)

### pmap
```bash
pmap -x 1234                   # Process memory map
pmap -X 1234                   # Extended format
pmap -XX 1234                  # Extra extended
```

## System Stats

### dstat (all-in-one)
```bash
dstat                          # Default view
dstat -cdngy                   # CPU, disk, net, paging, sys
dstat -tcms --top-cpu          # Time + CPU + mem + swap + top
dstat --tcp --udp              # Network sockets
dstat -d --disk-util           # Disk utilization
dstat -lp --top-io             # Load + proc + top I/O
dstat 1 10                     # 1s interval, 10 samples
```

### sar (System Activity Reporter)
```bash
sar                            # CPU stats
sar -u 1 10                    # CPU, 1s, 10 samples
sar -r 1                       # Memory
sar -b 1                       # I/O stats
sar -n DEV 1                   # Network
sar -n TCP,ETCP 1              # TCP stats
sar -q 1                       # Queue length/load
sar -d 1                       # Block device
sar -w 1                       # Context switches
sar -f /var/log/sa/sa01        # Historical data
```

### sysdig
```bash
sysdig                         # All events (verbose)
sysdig proc.name=java          # Filter by process
sysdig -c topprocs_cpu         # Top CPU processes
sysdig -c topfiles_bytes       # Top file I/O
sysdig -c topconns             # Top connections
sysdig -c bottlenecks          # Find bottlenecks
sysdig -w trace.scap           # Write capture
sysdig -r trace.scap           # Read capture
```

### nmon
```bash
nmon                           # Interactive mode
nmon -f -s 30 -c 120           # Background: 30s interval, 120 samples
nmon -f -T                     # Include top processes
```
**Interactive keys:** `c` CPU, `m` memory, `d` disk, `n` network

## Resource Limits

### ulimit
```bash
ulimit -a                      # Show all limits
ulimit -n                      # Open files (soft)
ulimit -Hn                     # Open files (hard)
ulimit -n 65535                # Set open files
ulimit -u                      # Max processes
ulimit -s                      # Stack size
ulimit -c unlimited            # Enable core dumps
```

### /proc/sys/fs
```bash
cat /proc/sys/fs/file-max      # Kernel file handle limit
cat /proc/sys/fs/file-nr       # Current/unused/max handles
cat /proc/sys/fs/nr_open       # Per-process limit cap
sysctl fs.file-max=2000000     # Increase limit
```

## File & Handle Monitoring

### lsof
```bash
lsof                           # All open files
lsof -p 1234                   # By PID
lsof -u username               # By user
lsof -c java                   # By command
lsof -i                        # Network files
lsof -i :8080                  # By port
lsof -i TCP                    # TCP only
lsof +D /var                   # Files in directory
lsof /var/log/syslog           # Who has file open
lsof -n -P                     # Numeric (faster)
```

### fuser
```bash
fuser -v /var/log/syslog       # Who uses file
fuser -k /var/log/syslog       # Kill processes using file
fuser -m /dev/sda1             # Processes using mount
fuser -n tcp 8080              # Processes using port
```

## System Information

### lscpu
```bash
lscpu                          # CPU info
lscpu -e                       # Extended format
lscpu -p                       # Parseable
```

### lsmem
```bash
lsmem                          # Memory ranges
lsmem -s                       # Summary
```

### lstopo (hwloc)
```bash
lstopo                         # GUI topology
lstopo --output-format txt     # Text mode
lstopo --output-format png > topo.png
lstopo -v                      # Verbose
```

### dmidecode
```bash
dmidecode                      # All hardware info
dmidecode -t memory            # Memory modules
dmidecode -t processor         # CPU details
dmidecode -t bios              # BIOS info
```

## Kernel Messages

### dmesg
```bash
dmesg                          # All messages
dmesg -T                       # Human timestamps
dmesg -w                       # Follow
dmesg -l err,warn              # Errors/warnings
dmesg --since "1 hour ago"     # Time filter
dmesg -H                       # Human readable
dmesg | grep -i error          # Search
```

### journalctl
```bash
journalctl                     # All logs
journalctl -f                  # Follow
journalctl -k                  # Kernel only
journalctl -p err              # Errors only
journalctl -u nginx            # Specific unit
journalctl --since "1 hour ago"
journalctl -b                  # Current boot
journalctl --disk-usage        # Log space used
```

## Quick Reference

| Task | Command |
|------|---------|
| CPU per-core | `mpstat -P ALL 1` |
| Memory usage | `free -wh` |
| Process list | `ps auxf` |
| Top CPU proc | `top -c` |
| System stats | `vmstat 1` |
| IO + CPU + net | `dstat -cdngy` |
| Open files | `lsof -p PID` |
| Limits | `ulimit -a` |
| Kernel msgs | `dmesg -T` |
| Historical | `sar -u 1` |
