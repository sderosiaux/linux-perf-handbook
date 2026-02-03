# GPU & HPC Profiling

GPU performance monitoring, accelerator optimization, and multi-GPU profiling for NVIDIA and AMD systems.

---

## GPU Architecture Primer

| Vendor | Unit | Description |
|--------|------|-------------|
| NVIDIA | SM (Streaming Multiprocessor) | CUDA cores, tensor cores, shared memory |
| AMD | CU (Compute Unit) | Stream processors, LDS, vector/scalar units |

**Key insight:** Utilization measures if SMs/CUs are busy, NOT efficiency. 100% utilization with CUDA cores is far slower than 50% with tensor cores.

### Memory Hierarchy

```
Registers → Shared Memory/LDS (~100KB) → L2 Cache (~50MB) → HBM (~80GB) → System RAM
```

### Interconnects

| Type | Bandwidth | Use Case |
|------|-----------|----------|
| PCIe 4.0 x16 | 32 GB/s | Consumer, single GPU |
| PCIe 5.0 x16 | 64 GB/s | Workstations |
| NVLink 4.0 | 900 GB/s | Multi-GPU training |
| Infinity Fabric | 896 GB/s | AMD multi-die (MI300X) |

---

## NVIDIA Profiling Stack

### nvidia-smi Deep Dive

```bash
nvidia-smi                              # Overview
nvidia-smi -l 1                         # Refresh every second
nvidia-smi -q -d MEMORY                 # Memory details
nvidia-smi -q -d PERFORMANCE            # Throttle reasons
nvidia-smi -q -d POWER                  # Power limits

# CSV for scripting
nvidia-smi --query-gpu=index,name,temperature.gpu,utilization.gpu,\
memory.used,power.draw --format=csv,noheader,nounits -l 1
```

**Device monitoring (dmon):**

```bash
nvidia-smi dmon -s pucmet -d 1          # All metrics, 1s interval
nvidia-smi dmon -s t                    # PCIe throughput only
```

| Flag | Metrics |
|------|---------|
| p | Power, temperature |
| u | SM, memory utilization |
| c | Clocks |
| m | FB, BAR1 memory |
| e | ECC errors |
| t | PCIe throughput |

**Topology and NVLink:**

```bash
nvidia-smi topo -m                      # GPU topology matrix
nvidia-smi nvlink -s                    # NVLink status
nvidia-smi nvlink -g 0                  # GPU 0 NVLink info
```

### Nsight Systems

System-wide profiler for CPU-GPU interaction.

```bash
nsys profile -o report ./my_app         # Generate .nsys-rep
nsys profile --stats=true ./my_app      # With summary stats
nsys profile -t cuda,nvtx,cudnn ./my_app # Targeted tracing
nsys profile -o report_%q{OMPI_COMM_WORLD_RANK} mpirun -np 4 ./app  # MPI
```

**Key analysis patterns:**
- Kernel launch gaps = CPU bottleneck
- Memory copy not overlapping compute = poor stream usage
- Single stream execution = missing parallelism

### Nsight Compute

Kernel-level profiler for CUDA optimization.

```bash
ncu ./my_app                            # Profile all kernels
ncu --set full ./my_app                 # All metrics (slow)
ncu -k "matmul" -c 5 ./my_app           # Specific kernel, 5 invocations
ncu --launch-skip 100 -c 10 ./my_app    # Skip warmup
ncu --set full -o roofline ./my_app     # Roofline analysis
```

**Key metrics:**

| Metric | Target |
|--------|--------|
| sm__throughput.avg.pct_of_peak | >80% |
| sm__inst_executed_pipe_tensor.avg | High for matrix ops |

### DCGM (Data Center GPU Manager)

```bash
apt install datacenter-gpu-manager
systemctl enable --now nvidia-dcgm

dcgmi discovery -l                      # List GPUs
dcgmi dmon -e 155,156,203               # SM activity, occupancy, tensor
dcgmi health -c                         # GPU health
dcgmi diag -r 3                         # Full stress test
```

**Key field IDs:**

| ID | Metric |
|----|--------|
| 155 | SM Activity |
| 156 | SM Occupancy |
| 203 | Tensor Active |
| 1001-1004 | FP16/TF32/FP64/INT8 |

**Prometheus:**

```bash
docker run -d --gpus all -p 9400:9400 nvcr.io/nvidia/k8s/dcgm-exporter:latest
```

---

## Multi-GPU Profiling

### NVLink Topology

```bash
nvidia-smi topo -m
# Output: NV12 = 12 NVLink connections between GPUs

# Bandwidth test
git clone https://github.com/NVIDIA/cuda-samples.git
cd cuda-samples/Samples/5_Domain_Specific/p2pBandwidthLatencyTest && make
./p2pBandwidthLatencyTest
```

### PCIe Bottleneck Detection

```bash
nvidia-smi dmon -s t                    # Watch PCIe throughput
nvidia-smi -q -d PCIE                   # Link generation/width
```

**Mitigation:** Use pinned memory, overlap transfers, batch small transfers.

### NCCL Collective Profiling

```bash
export NCCL_DEBUG=INFO                  # Basic logging
export NCCL_DEBUG=TRACE                 # Detailed
export NCCL_DEBUG_FILE=/tmp/nccl.%h.%p  # Per-host logs

# Benchmark
git clone https://github.com/NVIDIA/nccl-tests.git && cd nccl-tests && make
./build/all_reduce_perf -b 8 -e 256M -f 2 -g 4
```

**Tuning variables:**

| Variable | Purpose |
|----------|---------|
| NCCL_ALGO | Ring, Tree, CollNet |
| NCCL_PROTO | Simple, LL, LL128 |
| NCCL_BUFFSIZE | Per-channel buffer |

---

## AMD ROCm Profiling

### Basic Tools

```bash
apt install rocm-dev
rocm-smi                                # GPU status
rocminfo                                # Hardware details
rocm-smi --showuse                      # Utilization
```

### rocprof

```bash
rocprof ./my_hip_app                    # Default counters
echo "pmc: SQ_WAVES,SQ_INSTS_VALU" > counters.txt
rocprof -i counters.txt ./my_hip_app    # Custom counters
```

### Omniperf / rocprof-compute

High-level profiler (renamed in ROCm 6.3).

```bash
omniperf profile -n workload -- ./my_hip_app
omniperf analyze -p workloads/workload/MI300X/  # Web UI
# ROCm 6.3+: rocprof-compute profile/analyze
```

### MI300X Specifics

| Feature | Spec |
|---------|------|
| Memory | 192 GB HBM3 |
| Bandwidth | 5.3 TB/s |
| CUs | 304 |
| Infinity Fabric | 896 GB/s |

### ROCm vs CUDA

| Aspect | CUDA | ROCm |
|--------|------|------|
| Kernel profiler | Nsight Compute | Omniperf |
| System profiler | Nsight Systems | Omnitrace |
| CLI | nvidia-smi | rocm-smi |
| Partitioning | MIG | N/A |
| Open source | No | Yes |

---

## CPU-GPU Optimization

### Sync Bottleneck Detection

**Symptoms:** Long `cudaDeviceSynchronize`, GPU idle between kernels.

```cpp
// Bad: blocking
kernel1<<<...>>>(); cudaDeviceSynchronize(); kernel2<<<...>>>();

// Good: streams
kernel1<<<..., stream1>>>(); kernel2<<<..., stream2>>>();
cudaEventRecord(event, stream1); cudaStreamWaitEvent(stream2, event);
```

### Memory Transfer Profiling

```bash
nsys profile -t cuda --cuda-memory-usage=true ./my_app
```

### Pinned Memory

```cpp
// Pinned (2x bandwidth vs pageable)
cudaHostAlloc(&h_data, size, cudaHostAllocDefault);
cudaMemcpyAsync(d_data, h_data, size, cudaMemcpyHostToDevice, stream);
```

### Unified Memory

```bash
nsys profile --cuda-um-cpu-page-faults=true --cuda-um-gpu-page-faults=true ./app
```

```cpp
cudaMallocManaged(&data, size);
cudaMemPrefetchAsync(data, size, deviceId, stream);  // Prefetch to GPU
kernel<<<...>>>();
cudaMemPrefetchAsync(data, size, cudaCpuDeviceId, stream);  // Back to CPU
```

---

## Mixed Precision & Tensor Cores

### Precision Formats

| Format | Bits | Range | Use Case |
|--------|------|-------|----------|
| FP32 | 32 | ~1e38 | Default |
| TF32 | 19 | ~1e38 | A100+ training |
| FP16 | 16 | ~65k | Inference |
| BF16 | 16 | ~1e38 | Training |
| FP8 | 8 | ~448 | H100 |

### H100 SXM Throughput

| Precision | TFLOPS |
|-----------|--------|
| FP32 CUDA | 67 |
| TF32 Tensor | 989 |
| FP16/BF16 | 1,979 |
| FP8 | 3,958 |

### Profiling Tensor Cores

```bash
dcgmi dmon -e 203,1001,1002             # Tensor, FP16, TF32
ncu --metrics sm__inst_executed_pipe_tensor.avg ./my_app
```

### PyTorch AMP

```python
from torch.cuda.amp import autocast, GradScaler
scaler = GradScaler()
with autocast():
    output = model(data)
    loss = criterion(output, target)
scaler.scale(loss).backward()
scaler.step(optimizer)
```

---

## Container GPU Profiling

### MPS in Containers

```bash
# Host
nvidia-cuda-mps-control -d

# Container
docker run --gpus all -e CUDA_MPS_PIPE_DIRECTORY=/tmp/mps -v /tmp/mps:/tmp/mps app
```

### MIG in Containers

```bash
nvidia-smi mig -cgi 9,9 -C
docker run --gpus '"device=MIG-xxxxx"' my_app
```

**Kubernetes strategies:** none, single, mixed (nvidia.com/mig-1g.10gb)

### GPU Metrics in Kubernetes

```yaml
# DCGM Exporter DaemonSet
image: nvcr.io/nvidia/k8s/dcgm-exporter:latest
ports: [9400]
```

```promql
DCGM_FI_DEV_GPU_UTIL{pod=~"training-.*"}
DCGM_FI_PROF_PIPE_TENSOR_ACTIVE
```

---

## Power & Thermal Management

### Power Limiting

```bash
nvidia-smi -q -d POWER                  # Check limits
nvidia-smi -pl 300                      # Set 300W
nvidia-smi -pm 1                        # Persistence mode
```

**Systemd service:**

```ini
[Service]
ExecStart=/usr/bin/nvidia-smi -pm 1
ExecStart=/usr/bin/nvidia-smi -pl 300
```

### Thermal Throttle Detection

```bash
nvidia-smi -q -d PERFORMANCE
# Check: SW/HW Thermal Slowdown, HW Power Brake

watch -n 1 'nvidia-smi -q -d PERFORMANCE | grep -A5 "Clocks Throttle"'
```

| Reason | Severity |
|--------|----------|
| SwPowerCap | Normal |
| SwThermalSlowdown | Warning |
| HwThermalSlowdown | Critical |
| HwPowerBrakeSlowdown | Critical |

---

## Quick Diagnosis

| Symptom | Cause | Tool |
|---------|-------|------|
| High util, low throughput | Memory-bound | ncu roofline |
| GPU idle between kernels | CPU bottleneck | nsys timeline |
| High PCIe traffic | Transfer-bound | dmon -s t |
| Tensor Active = 0% | Wrong precision | DCGM 203 |
| HW Thermal Slowdown | Cooling issue | -q -d PERFORMANCE |

```bash
# Memory vs compute bound
ncu --metrics gpu__compute_memory_throughput.avg.pct_of_peak_sustained_elapsed,\
sm__throughput.avg.pct_of_peak_sustained_elapsed ./my_app
```

---

## MIG (Multi-Instance GPU)

Hardware-level partitioning for isolation (A100, H100).

### MIG vs MPS vs Time-Slicing

| Feature | MIG | MPS | Time-Slicing |
|---------|-----|-----|--------------|
| Isolation | Hardware | Software | None |
| Context Switch | None | Minimal | High |
| Max Instances | 7 | Many | 1 active |
| QoS Guarantee | Yes | No | No |

### Configuration

```bash
nvidia-smi -i 0 -mig 1                  # Enable MIG (requires reset)
nvidia-smi mig -lgip                    # List profiles
nvidia-smi mig -cgi 9 -C                # Create 3g.40gb instance
nvidia-smi mig -lgi                     # List instances
nvidia-smi -L                           # Show MIG UUIDs

export CUDA_VISIBLE_DEVICES=MIG-<uuid>
python train.py
```

### H100 Profiles

| Profile | Memory | SMs | Instances |
|---------|--------|-----|-----------|
| 1g.10gb | ~10GB | 16 | Up to 7 |
| 2g.20gb | ~20GB | 32 | Up to 3 |
| 3g.40gb | ~40GB | 60 | Up to 2 |
| 7g.80gb | ~80GB | 132 | 1 |

---

## Clock Management

### Lock Clocks for Benchmarking

```bash
# Lock GPU clocks (reduces variance)
nvidia-smi -lgc 1500,1500               # Lock at 1500 MHz
nvidia-smi -rgc                         # Reset to default

# Lock memory clocks
nvidia-smi -lmc 1215,1215
nvidia-smi -rmc
```

### Query Clock Reasons

```bash
nvidia-smi -q -d CLOCK
# Shows: Graphics, SM, Memory, Video clocks
# Shows: Max clocks and current clocks
```

---

## Profiling Workflows

### Training Job Workflow

```bash
# 1. Quick health check
nvidia-smi -q -d HEALTH
dcgmi diag -r 1

# 2. Profile first few iterations
nsys profile --duration=60 --delay=30 -o train_profile ./train.py

# 3. Check for tensor core usage
dcgmi dmon -e 203,1001 -c 60

# 4. Deep-dive slow kernels
ncu -k "slow_kernel" --set full ./train.py
```

### Inference Optimization Workflow

```bash
# 1. Baseline latency
nsys profile --stats=true ./inference.py

# 2. Check memory vs compute bound
ncu --set basic ./inference.py

# 3. Profile with different batch sizes
for bs in 1 4 16 64; do
    nsys profile -o batch_$bs ./inference.py --batch=$bs
done

# 4. Compare reports
nsys stats batch_*.nsys-rep
```

### Multi-Node Debugging

```bash
# Enable NCCL debug on all nodes
export NCCL_DEBUG=INFO
export NCCL_DEBUG_SUBSYS=INIT,NET

# Profile with node-specific output
nsys profile -o node_${SLURM_NODEID} ./train.py

# Check for network issues
NCCL_DEBUG=TRACE NCCL_DEBUG_SUBSYS=NET ./train.py 2>&1 | grep -E "NCCL|error"
```

---

## References

- NVIDIA Nsight: docs.nvidia.com/nsight-systems, docs.nvidia.com/nsight-compute
- NVIDIA DCGM: docs.nvidia.com/datacenter/dcgm
- AMD ROCm: rocm.docs.amd.com
- Omniperf: rocm.docs.amd.com/projects/omniperf
- NCCL: docs.nvidia.com/deeplearning/nccl
- CUDA Samples: github.com/NVIDIA/cuda-samples
- NCCL Tests: github.com/NVIDIA/nccl-tests
