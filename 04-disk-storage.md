# Disk & Storage

I/O benchmarking, filesystem tools, and storage analysis.

## I/O Monitoring

### iostat
```bash
iostat                         # Basic summary
iostat -x 1                    # Extended, 1s interval
iostat -xz 1                   # Skip inactive devices
iostat -p sda 1                # Specific device + partitions
iostat -m 1                    # MB/s instead of blocks
iostat -t 1                    # Add timestamp
iostat -d 1 10                 # Disk only, 10 samples
```
**Key columns:**
- `r/s`, `w/s` - Reads/writes per second
- `rMB/s`, `wMB/s` - Throughput
- `await` - Average I/O wait time (ms)
- `%util` - Device utilization

### iotop
```bash
iotop                          # Interactive view
iotop -o                       # Only processes doing I/O
iotop -a                       # Accumulated I/O
iotop -P                       # Processes only (no threads)
iotop -b -n 2                  # Batch mode, 2 iterations
iotop -p 1234                  # Specific PID
```

### pidstat (disk)
```bash
pidstat -d 1                   # Disk I/O per process
pidstat -d -p 1234 1           # Specific PID
```

### ioping (latency)
```bash
ioping .                       # Current directory
ioping -c 10 /path             # 10 requests
ioping -R /path                # Random IOPS
ioping -RL /path               # Sequential IOPS
ioping -s 4k /path             # 4KB requests
ioping -D /dev/sda             # Direct device access
```

## Benchmarking

### fio (flexible I/O tester)
```bash
# Random read test
fio --name=test --rw=randread --bs=4k --size=1G --numjobs=4 --runtime=60 --group_reporting

# Random write test
fio --name=test --rw=randwrite --bs=4k --size=1G --numjobs=4 --runtime=60 --group_reporting

# Sequential read
fio --name=test --rw=read --bs=1m --size=4G --numjobs=1 --runtime=60

# Mixed workload (70% read, 30% write)
fio --name=test --rw=randrw --rwmixread=70 --bs=4k --size=1G --numjobs=4 --runtime=60

# Direct I/O (bypass cache)
fio --name=test --rw=randread --bs=4k --size=1G --direct=1 --runtime=60

# Latency test
fio --name=lat --rw=randread --bs=4k --size=1G --direct=1 --ioengine=libaio --iodepth=1
```

**Job file example (`test.fio`):**
```ini
[global]
ioengine=libaio
direct=1
bs=4k
size=1G
runtime=60
group_reporting

[random-read]
rw=randread
numjobs=4

[random-write]
rw=randwrite
numjobs=4
```
Run: `fio test.fio`

### dd (simple benchmark)
```bash
# Write test (bypass cache)
dd if=/dev/zero of=testfile bs=1M count=1024 oflag=direct conv=fdatasync

# Read test (bypass cache)
dd if=testfile of=/dev/null bs=1M iflag=direct

# Measure with time
dd if=/dev/zero of=testfile bs=1G count=1 oflag=direct 2>&1 | grep -E 'copied|bytes'

# Clear cache before read test
sync && echo 3 > /proc/sys/vm/drop_caches
```

### bonnie++
```bash
bonnie++ -d /path -s 8G        # Test directory, 8GB file
bonnie++ -d /path -s 8G -n 0   # Skip file creation test
bonnie++ -u username           # Run as user
```

### hdparm
```bash
hdparm -Tt /dev/sda            # Buffered + cached read test
hdparm -I /dev/sda             # Drive info
hdparm --direct /dev/sda       # Direct I/O test
```

## Block Tracing

### blktrace
```bash
blktrace -d /dev/sda -o trace  # Capture trace
blkparse -i trace              # Parse trace
blktrace -d /dev/sda -o - | blkparse -i -  # Real-time

# Combined with btt
blktrace -d /dev/sda -w 60
btt -i sda.blktrace.0          # Block trace timing
```

### iowatcher
```bash
# Record and generate visualization
blktrace -d /dev/sda -w 30
iowatcher -t sda.blktrace.0 -o video.mp4
iowatcher -t sda.blktrace.0 -o graph.svg
```

### biosnoop (BCC)
```bash
biosnoop                       # Block I/O with latency
biosnoop -Q                    # Include queue time
biosnoop -d sda                # Specific device
```

### biotop (BCC)
```bash
biotop                         # Top for block I/O
biotop 5                       # 5 second interval
biotop -r 10                   # Top 10 rows
```

### biolatency (BCC)
```bash
biolatency                     # I/O latency histogram
biolatency -D                  # Per-disk
biolatency -m                  # Milliseconds
biolatency 1 10                # 1s interval, 10 outputs
```

## Disk Usage Analysis

### ncdu (NCurses disk usage)
```bash
ncdu /                         # Scan root
ncdu -x /                      # Stay on filesystem
ncdu -e --exclude .git         # Exclude pattern
ncdu --color dark              # Color scheme
ncdu -o output.json /          # Export JSON
ncdu -f output.json            # Load JSON
```

### dust
```bash
dust                           # Current directory
dust -n 15                     # Top 15
dust -d 3                      # Max depth 3
dust -r                        # Reverse sort
dust -b                        # No bar
```

### gdu (fast ncdu)
```bash
gdu                            # Current directory
gdu -d                         # Show apparent size
gdu -np                        # No cross-device, progress
gdu -i /pattern/               # Ignore pattern
```

### du
```bash
du -sh *                       # Summary, human
du -sh * | sort -h             # Sorted
du -h --max-depth=1            # One level
du -h --exclude='*.log'        # Exclude pattern
du -a | sort -n -r | head -20  # Top 20 files
```

### df
```bash
df -h                          # Human readable
df -T                          # Filesystem type
df -i                          # Inode usage
df -h --total                  # With total
```

## SMART Monitoring

### smartctl
```bash
smartctl -a /dev/sda           # All info
smartctl -H /dev/sda           # Health status
smartctl -i /dev/sda           # Device info
smartctl -A /dev/sda           # Attributes
smartctl -l error /dev/sda     # Error log
smartctl -t short /dev/sda     # Short self-test
smartctl -t long /dev/sda      # Long self-test
smartctl -l selftest /dev/sda  # Self-test results
```

**Key attributes to watch:**
- Reallocated_Sector_Ct (5)
- Reported_Uncorrect (187)
- Current_Pending_Sector (197)
- Offline_Uncorrectable (198)

### smartd (daemon)
```bash
# /etc/smartd.conf
/dev/sda -a -o on -S on -s (S/../.././02|L/../../6/03) -m root
# -a: all, -o: offline test, -S: attribute autosave
# -s: schedule (short daily 2am, long Sat 3am)
# -m: mail alerts
```

## NVMe Tools

### nvme-cli
```bash
nvme list                      # List NVMe devices
nvme smart-log /dev/nvme0      # SMART log
nvme error-log /dev/nvme0      # Error log
nvme id-ctrl /dev/nvme0        # Controller info
nvme id-ns /dev/nvme0n1        # Namespace info
nvme fw-log /dev/nvme0         # Firmware log
nvme get-feature /dev/nvme0 -f 0x02  # Get feature
```

## Filesystem Tools

### xfs_info / xfs_repair
```bash
xfs_info /dev/sda1             # XFS info
xfs_repair /dev/sda1           # Repair (unmounted)
xfs_repair -n /dev/sda1        # Check only
xfs_fsr /mountpoint            # Defragment
```

### e2fsck / tune2fs
```bash
e2fsck -f /dev/sda1            # Force check
e2fsck -n /dev/sda1            # Check only
tune2fs -l /dev/sda1           # Filesystem info
tune2fs -c 30 /dev/sda1        # Mount count check
tune2fs -i 180d /dev/sda1      # Time-based check
tune2fs -m 1 /dev/sda1         # Reserved blocks 1%
resize2fs /dev/sda1            # Resize filesystem
```

### lsblk
```bash
lsblk                          # Block devices
lsblk -f                       # Filesystems
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE,UUID
lsblk -J                       # JSON output
lsblk -d                       # Disks only
```

### blkid
```bash
blkid                          # All block devices
blkid /dev/sda1                # Specific partition
blkid -s UUID                  # UUID only
blkid -s TYPE                  # Type only
```

### findmnt
```bash
findmnt                        # Mount tree
findmnt -t ext4                # Type filter
findmnt -o TARGET,SOURCE,FSTYPE,OPTIONS
findmnt /home                  # Specific mount
```

### mount options
```bash
# Performance options
mount -o noatime,nodiratime /dev/sda1 /mnt  # Skip access time
mount -o barrier=0 ...         # Disable barriers (risky)
mount -o data=writeback ...    # ext4 writeback (risky)
mount -o discard ...           # Enable TRIM (SSD)
```

## I/O Scheduler

### Check/Set Scheduler
```bash
cat /sys/block/sda/queue/scheduler  # Current
echo mq-deadline > /sys/block/sda/queue/scheduler  # Set

# View available
cat /sys/block/sda/queue/scheduler
# Output: [mq-deadline] kyber bfq none
```

**Schedulers:**
- `none` - NVMe (best for fast devices)
- `mq-deadline` - Default, good for most
- `bfq` - Fair queuing, desktop use
- `kyber` - Low latency

### Tune Queue Parameters
```bash
# Queue depth
echo 256 > /sys/block/sda/queue/nr_requests

# Read-ahead
echo 4096 > /sys/block/sda/queue/read_ahead_kb

# Max sectors per request
echo 1024 > /sys/block/sda/queue/max_sectors_kb
```

## Quick Reference

| Task | Command |
|------|---------|
| I/O stats | `iostat -xz 1` |
| I/O per process | `iotop -o` |
| Latency test | `ioping -c 10 /path` |
| Random 4K IOPS | `fio --rw=randread --bs=4k --direct=1` |
| Write throughput | `dd if=/dev/zero of=test bs=1M count=1024 oflag=direct` |
| Disk usage | `ncdu /path` |
| SMART health | `smartctl -H /dev/sda` |
| Block trace | `blktrace -d /dev/sda -w 30` |
| NVMe info | `nvme smart-log /dev/nvme0` |
| Filesystem check | `e2fsck -f /dev/sda1` |
