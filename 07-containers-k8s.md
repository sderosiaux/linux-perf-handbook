# Containers & Kubernetes

Container runtimes, orchestration debugging, and container analysis tools.

## Container Runtimes

### containerd (nerdctl CLI)
```bash
nerdctl ps                     # List containers
nerdctl images                 # List images
nerdctl logs container         # Logs
nerdctl exec -it container sh  # Exec
nerdctl stats                  # Stats
nerdctl top container          # Top
nerdctl compose up             # Docker Compose compatible
```

### containerd Extensibility
Three extension points: clients (GRPC/Go SDK), snapshotters (filesystem), shims (isolation).

```bash
# Snapshotter interface - proxy to external binary
# Built-in: overlay, btrfs, zfs, devmapper, native
# Proxy snapshotters: SOCHI (AWS), Stargz, Nydus, image streaming (GKE)

# Shim interface - process isolation
# containerd passes OCI spec to shim binary
# Shims: runc, crun, runsc (gVisor), kata, youki (Rust), runwasi
# nerdbox: LibKRun-based, runs on macOS

# Run container on macOS with nerdbox shim
ctr run --runtime io.containerd.nerdbox.v1 docker.io/library/alpine:latest test sh

# Transfer service - pluggable source/sink for copy operations
# Stream processors - binary in pull/push stream (decryption, verification)

# containerd config for custom shim
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.custom]
  runtime_type = "io.containerd.custom.v1"
```

**Key insight:** containerd's plugin architecture enabled ecosystem growth without core changes. Shims especially: gVisor, Kata, Firecracker, WASM runtimes all integrate without modifying containerd.

### crictl (CRI debugging)
```bash
crictl ps                      # List containers
crictl pods                    # List pods
crictl images                  # List images
crictl logs CONTAINER          # Container logs
crictl exec -it CONTAINER sh   # Exec into container
crictl stats                   # Container stats
crictl inspect CONTAINER       # Inspect container
crictl inspectp POD            # Inspect pod
```

### Docker Debugging

#### docker stats
```bash
docker stats                   # Live resource usage
docker stats --no-stream       # Single snapshot
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
docker stats container1 container2  # Specific containers
```

#### docker inspect
```bash
docker inspect container       # Full JSON
docker inspect -f '{{.State.Pid}}' container  # Host PID
docker inspect -f '{{.NetworkSettings.IPAddress}}' container
docker inspect -f '{{json .Config.Env}}' container
docker inspect -f '{{.HostConfig.Memory}}' container
```

#### docker logs
```bash
docker logs container          # All logs
docker logs -f container       # Follow
docker logs --tail 100 container  # Last 100 lines
docker logs --since 1h container  # Last hour
docker logs -t container       # Timestamps
docker logs container 2>&1 | grep ERROR
```

#### docker exec
```bash
docker exec -it container bash
docker exec -it container sh   # Alpine/minimal
docker exec container ps aux
docker exec container cat /proc/1/status
docker exec -u root container command  # As root
```

#### docker top
```bash
docker top container           # Process list
docker top container aux       # With aux options
```

## Rootless Containers

Core primitives: user namespaces + mount namespaces enable rootless operation. `pivot_root` preferred over `chroot` (chroot is escapable).

```bash
# User namespace mapping for rootless
unshare --user --map-root-user --mount

# newuidmap/newgidmap (setuid helpers) for multi-user mapping
# Enables apt and other tools requiring multiple UIDs
newuidmap <pid> 0 1000 1 1 100000 65536
# Maps: container UID 0 -> host 1000, UIDs 1-65536 -> host 100000+

# Pivot root (secure) vs chroot (escapable)
# pivot_root swaps root AND unmounts old root
mount --bind newroot newroot
pivot_root newroot newroot/oldroot
umount -l /oldroot

# Escape chroot (why pivot_root is better)
mkdir /tmp/escape && chroot /tmp/escape && cd ../../../..

# Capabilities to drop for container security (Docker default set)
CAP_CHOWN, CAP_DAC_OVERRIDE, CAP_FSETID, CAP_FOWNER, CAP_MKNOD,
CAP_NET_RAW, CAP_SETGID, CAP_SETUID, CAP_SETFCAP, CAP_SETPCAP,
CAP_NET_BIND_SERVICE, CAP_SYS_CHROOT, CAP_KILL, CAP_AUDIT_WRITE
```

**Key insight:** Full rootless still needs setuid helpers (newuidmap) for multi-user mapping. Fully isolated privileged user namespaces (coming) will eliminate this requirement.

## Checkpoint/Restore (CRIU)

CRIU (Checkpoint Restore in Userspace) enables transparent process checkpointing without application modification. It "infects" the process with parasite code, captures state, then removes the parasite - the process never knows it was checkpointed.

```bash
# Kubernetes checkpoint API (beta in 1.30, defaults ON)
# Kubelet-only API - requires direct node access
curl -X POST "https://<node>:10250/checkpoint/<namespace>/<pod>/<container>"

# Inspect checkpoint with checkpointctl
checkpointctl show checkpoint.tar
checkpointctl show --all checkpoint.tar  # Full details: env vars, TCP sockets, mounts

# Convert checkpoint to OCI image for restore
buildah from scratch
buildah copy $container checkpoint.tar /
buildah config --annotation=io.kubernetes.cri-o.annotations.checkpoint.name=<name> $container
buildah commit $container checkpoint-image:v1
buildah push checkpoint-image:v1 registry/checkpoint-image:v1

# Restore: use checkpoint image in pod spec (CRI-O hooks into container create)
# CRI-O detects checkpoint annotation and restores instead of creating fresh
```

**Key insight:** Checkpoint images include ALL memory pages (passwords, secrets, random numbers) - treat as sensitive. Banks use this for forensic analysis of suspicious containers.

### GPU Checkpoint/Restore with CUDA

NVIDIA's `cuda-checkpoint` tool moves GPU state to host memory, then CRIU checkpoints everything. No API interception overhead, works with static linking.

```bash
# CRIU with CUDA plugin for NVIDIA GPUs
# Requires: cuda-checkpoint tool from NVIDIA
# GPU state moves to host memory -> CRIU captures unified checkpoint

# Checkpoint sizes: 90-97% is GPU state for large models
# Restore time dominated by disk I/O, not GPU restore
# Works for migration between SAME GPU types only (A100 -> A100, not A100 -> H100)
```

**Key insight:** GPU checkpointing enables preempting training jobs for inference, improving GPU utilization. Compression is most effective for reducing checkpoint size.

## Kubernetes Debugging

### kubectl basics
```bash
kubectl get pods -A            # All namespaces
kubectl get pods -o wide       # More columns
kubectl get pods -w            # Watch
kubectl describe pod POD
kubectl logs POD               # Logs
kubectl logs -f POD            # Follow
kubectl logs POD -c container  # Specific container
kubectl logs POD --previous    # Previous instance
kubectl exec -it POD -- bash
kubectl port-forward POD 8080:80
kubectl cp POD:/path ./local
```

### kubectl debug
```bash
# Debug with ephemeral container
kubectl debug POD -it --image=busybox

# Copy pod for debugging
kubectl debug POD --copy-to=debug-pod -it --image=ubuntu

# Debug node
kubectl debug node/NODE -it --image=ubuntu

# With share process namespace
kubectl debug POD -it --image=busybox --share-processes
```

### k9s (TUI for Kubernetes)
```bash
k9s                            # Current context
k9s -n namespace               # Specific namespace
k9s --context ctx              # Specific context
k9s --readonly                 # Read-only mode
```
**Keys:**
- `:` - Command mode
- `/` - Filter
- `d` - Describe
- `l` - Logs
- `s` - Shell
- `y` - YAML
- `ctrl-k` - Kill

### stern (multi-pod logs)
```bash
stern pod-name                 # Pods matching name
stern .                        # All pods
stern -n namespace .           # Specific namespace
stern --since 10m .            # Last 10 minutes
stern -l app=nginx             # By label
stern --tail 50 pod            # Last 50 lines
stern -o json .                # JSON output
stern --exclude 'health'       # Exclude pattern
```

### kubectx / kubens
```bash
kubectx                        # List contexts
kubectx prod                   # Switch context
kubectx -                      # Previous context

kubens                         # List namespaces
kubens kube-system             # Switch namespace
kubens -                       # Previous namespace
```

### kubectl top
```bash
kubectl top nodes              # Node resources
kubectl top pods               # Pod resources
kubectl top pods -A            # All namespaces
kubectl top pods --containers  # Per container
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory
```

### kube-capacity
```bash
kube-capacity                  # Cluster capacity
kube-capacity -p               # Include pods
kube-capacity -u               # Utilization
kube-capacity -n namespace
```

### ksniff (packet capture)
```bash
ksniff POD                     # Capture packets
ksniff POD -n namespace        # With namespace
ksniff POD -o capture.pcap     # To file
ksniff POD -f "port 80"        # Filter
ksniff POD -p                  # Privileged mode
```

### kubectl-trace (bpftrace in K8s)
```bash
kubectl trace run NODE -e 'tracepoint:syscalls:sys_enter_open { printf("%s %s\n", comm, str(args->filename)); }'
kubectl trace run POD -e 'profile:hz:99 { @[kstack] = count(); }'
```

### Pod Troubleshooting

```bash
# Pod status
kubectl get pod POD -o jsonpath='{.status.phase}'

# Container states
kubectl get pod POD -o jsonpath='{.status.containerStatuses[*].state}'

# Events
kubectl get events --field-selector involvedObject.name=POD

# Describe (events at bottom)
kubectl describe pod POD

# Previous logs (if crashed)
kubectl logs POD --previous

# Init container logs
kubectl logs POD -c init-container-name
```

### Debug stuck pods
```bash
# Pending - check events
kubectl describe pod POD | grep -A 20 Events

# CrashLoopBackOff - check logs
kubectl logs POD --previous

# ImagePullBackOff - check image
kubectl describe pod POD | grep -A 5 Image

# Debug with ephemeral container
kubectl debug POD -it --image=busybox --target=container
```

## Container Networking

### Docker Network inspection
```bash
docker network ls
docker network inspect bridge
docker network inspect -f '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{println}}{{end}}' bridge
```

### Container networking debug
```bash
# tcpdump from host
docker inspect -f '{{.State.Pid}}' container
nsenter -t PID -n tcpdump -i eth0

# Using nicolaka/netshoot
docker run --rm -it --net container:target nicolaka/netshoot
```

### nsenter (access container namespaces)
```bash
# Get container PID
PID=$(docker inspect -f '{{.State.Pid}}' container)

# Enter namespaces
nsenter -t $PID -n ip addr     # Network namespace
nsenter -t $PID -m ls /        # Mount namespace
nsenter -t $PID -p ps aux      # PID namespace
nsenter -t $PID -a bash        # All namespaces
```

### User-Defined Networking in Kubernetes

UDN provides native namespace isolation, overlapping pod IPs, persistent IPs for VM migration.

```yaml
# UserDefinedNetwork CRD for namespace isolation
apiVersion: k8s.ovn.org/v1
kind: UserDefinedNetwork
metadata:
  name: tenant-network
  namespace: green
spec:
  topology: Layer2
  layer2:
    role: Primary
    subnets:
      - cidr: "192.168.1.0/24"
    ipamLifecycle: Persistent  # For VM migration

# ClusterUserDefinedNetwork for cross-namespace grouping
apiVersion: k8s.ovn.org/v1
kind: ClusterUserDefinedNetwork
metadata:
  name: shared-network
spec:
  namespaceSelector:
    matchLabels:
      network: shared
  topology: Layer2
  layer2:
    role: Primary
    subnets:
      - cidr: "10.0.0.0/16"
    ipamLifecycle: Persistent
```

**Key insight:** Each UDN creates isolated OVN layer-2 switch. Enables tenant isolation without network policies, supports overlapping subnets across tenants.

## Image Management

### docker history
```bash
docker history image           # Layer history
docker history --no-trunc image
docker history --format "{{.Size}}\t{{.CreatedBy}}" image
```

### docker diff
```bash
docker diff container          # Changed files
# A = Added, C = Changed, D = Deleted
```

### dive (image layers)
```bash
dive image:tag                 # Analyze image
dive build -t image:tag .      # Build + analyze
```
- Shows layer sizes
- Identifies wasted space
- Suggests optimizations
- Tab to switch panes

### Trivy (vulnerability scanning)
```bash
trivy image nginx:latest       # Scan image
trivy image --severity HIGH,CRITICAL nginx
trivy fs /path                 # Scan filesystem
trivy config /path             # Scan IaC
trivy image --format json nginx > results.json
```

### Grype (vulnerability scanning)
```bash
grype image:tag                # Scan image
grype dir:/path                # Scan directory
grype sbom:./sbom.json         # Scan SBOM
grype image:tag -o json        # JSON output
```

### Syft (SBOM generation)
```bash
syft image:tag                 # Generate SBOM
syft image:tag -o json         # JSON output
syft image:tag -o spdx-json    # SPDX format
syft dir:/path                 # From directory
```

### Cosign (signing/verification)
```bash
cosign sign image:tag          # Sign image
cosign verify image:tag        # Verify signature
cosign triangulate image:tag   # Find signature location
```

### Spegel: P2P Image Distribution

Nodes pull images from each other using distributed hash table (Kademlia/libp2p). Zero state - piggybacks on containerd's existing image storage.

```bash
# Spegel architecture:
# - Each node runs OCI registry on localhost
# - containerd configured with localhost as mirror
# - DHT advertises local images to cluster
# - Falls back to upstream registry if not found locally

# containerd mirror config
# /etc/containerd/certs.d/docker.io/hosts.toml
server = "https://registry-1.docker.io"
[host."http://127.0.0.1:5000"]
  capabilities = ["pull", "resolve"]

# Images stored in containerd are uncompressed on disk
# /var/lib/containerd/io.containerd.content.v1.content/blobs/sha256/

# Key benefits:
# - 80% improvement in pull times (internal network faster than internet)
# - Survives registry outages (images exist somewhere in cluster)
# - Embedded in k3s/rke2 as library
```

**Key insight:** Container images already exist on disk in your cluster. Spegel just makes them discoverable via P2P without additional storage or state.

## systemd Integration (Quadlet)

Quadlet is a systemd generator converting declarative container specs to systemd units. Predictable lifecycle, survives reboots.

```bash
# Quadlet file location
~/.config/containers/systemd/  # User
/etc/containers/systemd/       # System

# Example: nginx.container
[Container]
ContainerName=nginx-web
Image=docker.io/library/nginx:latest
PublishPort=8080:80

[Service]
Restart=always

[Install]
WantedBy=default.target

# Reload and start
systemctl --user daemon-reload
systemctl --user start nginx-web.service

# Generate quadlet from running container
podlet generate container nginx-web

# Kubernetes YAML as quadlet (.kube file)
# Podman can run K8s manifests directly, quadlet wraps them in systemd
```

**Key insight:** Quadlet bridges compose/development and production. Same Kubernetes YAML locally without full cluster overhead, with systemd lifecycle management.

## Bootable Containers (bootc)

OCI images containing full OS (kernel + bootloader). Build OS with container tools, deploy anywhere.

```bash
# Base images: quay.io/fedora/fedora-bootc, quay.io/centos-bootc/centos-bootc-dev
# Inherit and customize with standard Containerfile

# Containerfile for bootable OS
FROM quay.io/fedora/fedora-bootc:41
RUN dnf install -y nginx
COPY nginx.conf /etc/nginx/

# Build and push (standard container workflow)
podman build -t myos:latest .
podman push myos:latest registry/myos:latest

# Convert to disk image with bootc-image-builder
# Outputs: qcow2, vmdk, raw, AMI
bootc-image-builder --type qcow2 registry/myos:latest

# On running system: update from container registry
bootc upgrade  # Pulls new image, stages for reboot
bootc rollback # Revert to previous image

# Automated rollback on boot failure (5 failures triggers rollback)
# ComposeFS: fs-verity backed, content-addressed, immutable
```

**Key insight:** /usr is immutable (fs-verity), /var is persistent mutable, /etc does 3-way merge on updates. Same CI/CD for OS and apps.

## MicroVMs

### urnc: Containers in Minimal VMs

urnc runs containers inside minimal Linux VMs for hardware isolation. One VM per container, treats VM as the process.

```bash
# urnc: OCI-compatible runtime, VM per container
# Uses stripped-down Linux kernel (~13MB vs Kata's larger kernel)

# Build minimal Linux kernel for specific container
# ~1000 config options in Kata kernel vs ~100 for specialized kernel
# Achieves boot times close to runc with VM isolation

# Comparison (lower is better):
# Image size: Kata ~large, specialized Linux ~13MB, Unikraft HTTP ~few KB
# Start time: runc fastest, urnc close second, Kata slower
```

**Key insight:** Specialized Linux kernels can match unikernel sizes while maintaining full Linux compatibility. urnc provides VM isolation with container-like startup times.

## Observability

### Fleet-Wide Kubernetes Observability

Three strategies: metrics selection, noise-to-signal alerts, correlation.

```bash
# Metric usefulness checklist:
# 1. Actionable? (triggers investigation/action)
# 2. Provides context? (correlates with other metrics)
# 3. Predictive? (forecasts future issues)
# 4. Real-time AND historic? (immediate + trend analysis)

# Alert actionability flow:
# 1. Relevant to SLOs/business goals?
# 2. Clear context (what component, expected outcome)?
# 3. Actionable in real-time?
# 4. Prioritized appropriately?
# 5. Can be automated?

# SLO-driven example
# SLO: 95% of clusters upgrade within 2 hours
# Alert: Trigger when control plane upgrade stalls > 1 hour
# Proactive: Alert before SLO breach, not after
```

**Key insight:** Over-collection and cardinality explosions are the main scaling problems. Prioritize high-signal metrics aligned with SLOs, not infrastructure noise.

### Container TUI Tools

#### ctop
```bash
ctop                           # Interactive view
ctop -a                        # All containers
ctop -s cpu                    # Sort by CPU
ctop -s mem                    # Sort by memory
ctop -f string                 # Filter
```

#### lazydocker
```bash
lazydocker                     # TUI dashboard
```
- Navigate with arrow keys
- `d` - Remove container
- `s` - Stop container
- `r` - Restart
- `l` - View logs
- `e` - Exec shell

#### dry
```bash
dry                            # Docker manager TUI
```

### cgroup Tools

#### cgroupfs v2
```bash
# View hierarchy
ls /sys/fs/cgroup/

# CPU stats
cat /sys/fs/cgroup/system.slice/docker-CONTAINERID.scope/cpu.stat

# Memory
cat /sys/fs/cgroup/system.slice/docker-CONTAINERID.scope/memory.current
cat /sys/fs/cgroup/system.slice/docker-CONTAINERID.scope/memory.max

# I/O
cat /sys/fs/cgroup/system.slice/docker-CONTAINERID.scope/io.stat
```

#### systemd-cgtop
```bash
systemd-cgtop                  # cgroup resource usage
systemd-cgtop -m               # Sort by memory
systemd-cgtop -c               # Sort by CPU
systemd-cgtop -d 1             # 1 second delay
```

## JVM in Containers

JVM defaults to 25% of container memory for heap (conservative for non-container contexts). Red Hat builds default to 80%.

```bash
# JVM container awareness (default ON since JDK 8 backports)
# -XX:+UseContainerSupport  # Not needed anymore, defaults ON

# Heap sizing - override the 25% default
-XX:MaxRAMPercentage=80.0  # Use 80% of container memory for heap

# Garbage collectors for containers
-XX:+UseG1GC              # Default balanced GC
-XX:+UseShenandoahGC      # Low latency (not in Oracle builds)
-XX:+UseZGC               # Low latency alternative
-XX:+UseEpsilonGC         # No-op GC: fast but crashes on OOM (good for functions)

# Metaspace tuning (JDK 16+)
-XX:MetaspaceReclaimPolicy=balanced  # Or aggressive

# Compact object headers (JDK 24+) - 20% memory reduction
-XX:+UseCompactObjectHeaders

# Class Data Sharing for faster startup (JDK 19+)
# Step 1: Profile
java -XX:AOTMode=record -XX:AOTConfiguration=app.aotconf -jar app.jar
# Step 2: Build cache
java -XX:AOTMode=create -XX:AOTConfiguration=app.aotconf -XX:AOTCache=app.aot -jar app.jar
# Step 3: Run with cache
java -XX:AOTCache=app.aot -jar app.jar
```

**Key insight:** JVM heap is just one memory consumer. Account for metaspace, native memory, thread stacks, subprocesses, and Kubernetes probes when sizing container memory limits.

## Quick Reference

| Task | Command |
|------|---------|
| Container resources | `docker stats` |
| Container PID | `docker inspect -f '{{.State.Pid}}' c` |
| Image layers | `dive image:tag` |
| Vulnerability scan | `trivy image nginx:latest` |
| K8s TUI | `k9s` |
| Multi-pod logs | `stern .` |
| Pod debug | `kubectl debug POD -it --image=busybox` |
| Packet capture | `ksniff POD` |
| Node capacity | `kube-capacity -p` |
| cgroup usage | `systemd-cgtop` |
