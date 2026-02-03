# GPU & HPC Profiling

GPU performance monitoring, HPC tracing, and accelerator debugging.

## GPU Profiling Tools

### NVIDIA DCGM (Data Center GPU Manager)

Hardware-level GPU monitoring for data centers.

```bash
# Install DCGM
apt install datacenter-gpu-manager

# Start DCGM service
systemctl start nvidia-dcgm

# Basic monitoring
dcgmi discovery -l              # List GPUs
dcgmi dmon                      # Real-time monitoring
dcgmi dmon -e 155,156,203,204   # Specific metrics by ID

# Field Groups for monitoring
# 155 = SM Activity
# 156 = SM Occupancy
# 203 = Tensor Active
# 204 = DRAM Active
```

**Key DCGM Metrics:**

| Code | Metric | Purpose |
|------|--------|---------|
| 155 | SM Activity | Actual compute work happening |
| 156 | SM Occupancy | SMs with scheduled work |
| 203 | Tensor Active | Tensor core utilization |
| 1001 | FP16 Tensor | Half-precision matrix ops |
| 1002 | FP32 Tensor | Single-precision ops (TF32) |
| 1003 | FP64 Tensor | Double-precision ops |
| 1004 | INT8 Tensor | Integer matrix ops |

**Temperature Policy Monitoring:**

```bash
# Set temperature alert at 60C
dcgmi policy --set 60,0 --target temperature
dcgmi policy --reg                # Register policy
dcgmi policy --get                # Verify policy
# Violations reported in real-time
```

### NVIDIA SMI Limitations

`nvidia-smi` shows GPU utilization but **not efficiency**:

- 100% utilization does NOT mean 100% efficiency
- FP32 workloads may show 100% but use 0% tensor cores
- FP16 workloads use tensor cores (8-10x faster than CUDA cores)

```bash
nvidia-smi                        # Basic overview
nvidia-smi -l 1                   # Refresh every second
nvidia-smi -L                     # List GPU UUIDs
nvidia-smi --query-gpu=utilization.gpu,memory.used --format=csv
```

**Critical insight:** GPU utilization measures if SMs are doing work, not how much or how efficiently.

### Profiling Tools Comparison

| Tool | Scope | Open Source |
|------|-------|-------------|
| DCGM/DCGMI | System-level metrics | Open core |
| nvidia-smi | Basic utilization | Proprietary |
| Nsight Compute | Kernel-level (SM detail) | Proprietary |
| Nsight Systems | Micro-level profiling | Proprietary |

### Estimating TFLOPS

```
Estimated TFLOPS = Peak TFLOPS x SM Activity x SM Occupancy x Tensor Activity
```

Works for stable workloads; less accurate for bursty patterns.

## eBPF for GPU Auto-Instrumentation

### Beyla GPU Probes (Grafana)

eBPF-based auto-instrumentation for CUDA workloads.

**Advantages:**
- Auto-instrumentation (no code changes)
- Framework-agnostic (PyTorch, TensorFlow, etc.)
- Lower overhead than profiler-based approaches
- Kubernetes-native with automatic metadata decoration

**Instrumented CUDA Functions:**

```c
// CUDA Launch Kernel - tracks kernel dispatches
CUresult cuLaunchKernel(
    CUfunction func,          // Function offset for kernel name
    unsigned int gridDimX,    // Grid dimensions (cardinality)
    unsigned int gridDimY,
    unsigned int gridDimZ,
    unsigned int blockDimX,   // Block dimensions
    ...
);

// CUDA Memory Operations
cudaMalloc()                   // Memory allocation (model load)
cudaMemcpy()                   // Host<->Device transfers
```

**Metrics Generated:**

```promql
# Kernel launches by name
gpu_kernel_launch_total{kernel_name="matmul_fp16"}

# Memory transfer direction
gpu_memcpy_bytes{direction="host_to_device"}
gpu_memcpy_bytes{direction="device_to_host"}

# Grid/block cardinality for resource estimation
gpu_kernel_grid_size
gpu_kernel_block_size
```

**Limitations:**
- No kernel execution time (async dispatch)
- No GPU hardware metrics (temperature, power)
- CPU-side instrumentation only

**Stack Trace Capture:**

```c
// BPF helper for call stack
bpf_get_stack(ctx, stack, sizeof(stack), BPF_F_USER_STACK);
```

Note: Frame pointers often optimized away; use DWARF unwinding for full stacks.

## NVIDIA MIG (Multi-Instance GPU)

Hardware-level GPU partitioning for isolation.

### MIG vs MPS vs Time-Slicing

| Feature | MIG | MPS | Time-Slicing |
|---------|-----|-----|--------------|
| Isolation | Hardware | Software | None |
| Context Switch | None | Minimal | High |
| Max Instances | 7 | Many | 1 active |
| QoS Guarantee | Yes | No | No |
| Use Case | Production | Dev/test | Single job |

### MIG Concepts

**Hierarchy:**
1. GPU Instance (GI) - Memory + Compute slices
2. Compute Instance (CI) - Allocated within GI

**Slice allocation:** Memory first, then compute assigned.

### MIG Configuration

```bash
# Enable MIG mode (requires reboot/reset)
nvidia-smi -i 0 -mig 1

# List available profiles
nvidia-smi mig -lgip

# Create GPU Instance (profile ID 9 = 3g.40gb on H100)
nvidia-smi mig -cgi 9 -C            # Create GI + default CI
nvidia-smi mig -cgi 9,9             # Two 3g.40gb instances

# Create Compute Instance within GI
nvidia-smi mig -i 0 -gi 0 -cci 0    # Full CI for GI 0

# List instances
nvidia-smi -L                        # Shows MIG UUIDs
nvidia-smi mig -lgi                  # List GPU instances
nvidia-smi mig -lci                  # List compute instances
```

**Run workload on MIG instance:**

```bash
# Export MIG device UUID
export CUDA_VISIBLE_DEVICES=MIG-<uuid>

# Run application
python train.py
```

### MIG Partition Profiles (H100 80GB)

| Profile | Memory | SMs | Use Case |
|---------|--------|-----|----------|
| 1g.10gb | ~10GB | 16 | Small inference |
| 2g.20gb | ~20GB | 32 | Medium workloads |
| 3g.40gb | ~40GB | 60 | Large models |
| 4g.40gb | ~40GB | 60 | Balanced |
| 7g.80gb | ~80GB | 132 | Full GPU |

**Overhead:** Not exact 1/7 division. 7x 1g instances = ~70GB (10GB unused). 132 SMs / 7 = 18.8, rounded to 16 per instance.

### MIG Partition Combinations

**Efficient combinations (no wasted compute):**
- 7x 1g.10gb
- 1x 7g.80gb
- 3x 2g.20gb + 1x 1g.10gb
- 1x 4g.40gb + 1x 3g.40gb

**Wasteful combinations (1 compute slice unused):**
- 2x 3g.40gb (6/7 compute used)
- 3x 2g.20gb (6/7 compute used)

### MIG + MPS Hybrid

```bash
# MPS can run inside MIG instance
export CUDA_VISIBLE_DEVICES=MIG-<uuid>
nvidia-cuda-mps-control -d

# Note: Cannot create MIG while MPS running
# Must stop MPS first
echo quit | nvidia-cuda-mps-control
```

## HPC Tracing with LTTng

### Exa-Tracer Architecture

LTTng-based tracing for HPC supercomputers (El Capitan, Aurora scale).

**Instrumented Layers:**
- HIP/HSA (AMD GPU runtime)
- ROCm/RocTX
- GPU Kernel Dispatch
- MPI (OpenMPI, Cray MPI)
- Linux Kernel

**Installation:**

```bash
# LTTng session daemon
lttng-sessiond --daemonize

# Create trace session
lttng create hpc-session

# Enable kernel events
lttng enable-event -k sched_switch,sched_wakeup

# Enable userspace HIP/HSA events
lttng enable-event -u 'hip:*'
lttng enable-event -u 'hsa:*'
lttng enable-event -u 'mpi:*'
```

### LTTng Features for HPC

- **Per-CPU ring buffer:** ~100ns per event
- **Flight recorder mode:** In-memory tracing, capture on trigger
- **Session rotation:** Split traces into chunks for pipeline analysis
- **Cross-host correlation:** NTP/PTP time sync, TCP sequence realignment

```bash
# Flight recorder (snapshot) mode
lttng enable-channel -u --type=snapshot my-channel
lttng enable-event -u -c my-channel 'hip:*'
lttng start
# ... trigger condition ...
lttng snapshot record

# Session rotation (periodic chunks)
lttng enable-rotation --timer 60s
```

### CTF (Common Trace Format)

Binary trace format for HPC scale.

- Self-describing metadata (JSON in CTF2)
- Compact binary data stream
- Clock descriptions for correlation
- User-defined metadata extensions

```bash
# Read CTF trace
babeltrace2 ~/lttng-traces/my-session

# Convert to other formats
babeltrace2 input-trace --output-format=ctf2 --output=output-trace
```

### Trace Compass Visualization

Eclipse-based trace analyzer.

- Resource view (CPU states, frequency)
- Control flow view (thread timeline)
- Critical path analysis
- Network packet correlation

**Key views for HPC:**
- MPI rank timeline
- GPU kernel dispatch events
- Host-to-device memory transfers

### TAPI (Argonne National Lab)

LTTng-based instrumentation for Aurora (10K nodes, 9M cores).

Instruments: OpenCL, Level Zero, CUDA Runtime, HIP, OpenMP Target.

## HPC Cluster Monitoring

### Cluster Cockpit

Job-specific performance monitoring for HPC clusters.

**Architecture:**
```
[Metric Collector] --JSON--> [Metric Store] <--query-- [Backend/UI]
     (per node)               (in-memory)              (job archive)
```

**Components:**

```bash
# Metric collector plugins
- likwid    # Hardware performance counters
- cpustat   # CPU load
- memstat   # Memory usage
- ibstat    # InfiniBand metrics
- nvidia    # GPU metrics via NVML
```

**Metrics tracked per job:**
- CPU utilization (per core/socket/node)
- Memory bandwidth consumption
- Floating-point operations (FLOPS)
- Network I/O (InfiniBand)
- GPU utilization (if applicable)

### Puzzle Jobs (Automated Analysis)

Rule-based job quality assessment.

```json
{
  "tag": "low_cpu_load",
  "threshold": "config.low_cpu_threshold",
  "metrics": ["cpu_load"],
  "vars": {
    "avg_load": "mean(cpu_load.data)",
    "low_load": "avg_load < threshold"
  },
  "trigger": "low_load == True"
}
```

**Built-in rules:**
- Low CPU load
- CPU load imbalance
- Excessive CPU load
- Memory underutilization
- Network idle time

**Deployment insight:** 20-50% of jobs show problematic behavior patterns.

### Roofline Analysis

Spider/radar diagram for job efficiency:

```
          Memory BW
             /\
            /  \
           /    \
     FLOPS ------  Memory Used
```

Full triangle = efficient resource utilization.

## AMD GPU Internals

### ROCm Open Source Stack

AMD's GPU compute stack is fully open source.

**Components:**
- Kernel driver: `amdgpu` (in mainline Linux)
- User space: `rocm` (GitHub)
- ISA documentation: `amd.com/en/support/tech-docs`

```bash
# ROCm installation
apt install rocm-dev

# GPU info
rocm-smi
rocminfo
```

### SIMT vs Vector Programming

**SIMT (Single Instruction Multiple Thread):** CUDA/HIP programming model where scalar code implicitly operates on 32/64 lanes.

**Direct vector approach:** Write explicit SIMD/vector code targeting GPU vector units.

```
# SIMT model (implicit vectorization)
int x = compute();  // Actually vector of 32/64 ints

# Vector model (explicit)
v32int x = vector_compute();
```

AMD's ISA allows both approaches; LLVM defaults to SIMT model.

## FOSDEM Insights

### GPU Failure Rates (Meta Llama 3)

From 16K GPU training over 54 days:
- 58% of training stalls due to GPU issues
- 1-3% GPU hardware failure rate (overheat, fall off bus)
- eBPF cannot help with hardware errors

### Key Monitoring Patterns

**CPU-GPU data transfer is the bottleneck:**
- High SM occupancy + Low SM activity + High VRAM = Memory-bound
- Typical for inference workloads

**Tensor core utilization:**
- FP32 defaults to CUDA cores (no tensor acceleration)
- FP16/TF32 use tensor cores (8-10x speedup)
- Check with DCGM metrics 203, 1001-1004

### Tracing vs Profiling

| Aspect | Profiling | Tracing |
|--------|-----------|---------|
| Overhead | Low (sampling) | Must be low |
| Best for | Active resource use | Resource misuse |
| Output | Aggregated stats | Event sequence |
| Scale | Single job | Cluster-wide |

Tracing excels at finding why resources are NOT being used (waiting, stalls, dependencies).

### References

- DCGM Exporter: `github.com/NVIDIA/dcgm-exporter`
- Cluster Cockpit: `github.com/ClusterCockpit`
- Exa-Tracer: LTTng + ROCm instrumentation
- Beyla GPU: `github.com/grafana/beyla`
- Azure HPC Node Health: `github.com/Azure/azurehpc`
- TAPI: `github.com/argonne-lcf/THAPI`
