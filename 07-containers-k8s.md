# Containers & Kubernetes

Docker debugging, Kubernetes tools, and container analysis.

## Docker Debugging

### docker stats
```bash
docker stats                   # Live resource usage
docker stats --no-stream       # Single snapshot
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
docker stats container1 container2  # Specific containers
```

### docker inspect
```bash
docker inspect container       # Full JSON
docker inspect -f '{{.State.Pid}}' container  # Host PID
docker inspect -f '{{.NetworkSettings.IPAddress}}' container
docker inspect -f '{{json .Config.Env}}' container
docker inspect -f '{{.HostConfig.Memory}}' container
```

### docker logs
```bash
docker logs container          # All logs
docker logs -f container       # Follow
docker logs --tail 100 container  # Last 100 lines
docker logs --since 1h container  # Last hour
docker logs -t container       # Timestamps
docker logs container 2>&1 | grep ERROR
```

### docker exec
```bash
docker exec -it container bash
docker exec -it container sh   # Alpine/minimal
docker exec container ps aux
docker exec container cat /proc/1/status
docker exec -u root container command  # As root
```

### docker top
```bash
docker top container           # Process list
docker top container aux       # With aux options
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

## Container TUI Tools

### ctop
```bash
ctop                           # Interactive view
ctop -a                        # All containers
ctop -s cpu                    # Sort by CPU
ctop -s mem                    # Sort by memory
ctop -f string                 # Filter
```

### lazydocker
```bash
lazydocker                     # TUI dashboard
```
- Navigate with arrow keys
- `d` - Remove container
- `s` - Stop container
- `r` - Restart
- `l` - View logs
- `e` - Exec shell

### dive (image layers)
```bash
dive image:tag                 # Analyze image
dive build -t image:tag .      # Build + analyze
```
- Shows layer sizes
- Identifies wasted space
- Suggests optimizations
- Tab to switch panes

### dry
```bash
dry                            # Docker manager TUI
```

## Docker Networking

### Network inspection
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

## Image Analysis

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

## Resource Analysis

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

## Pod Troubleshooting

### Common checks
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

## cgroup Tools

### cgroupfs v2
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

### systemd-cgtop
```bash
systemd-cgtop                  # cgroup resource usage
systemd-cgtop -m               # Sort by memory
systemd-cgtop -c               # Sort by CPU
systemd-cgtop -d 1             # 1 second delay
```

## crictl (CRI debugging)

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

## nerdctl (containerd CLI)

```bash
nerdctl ps                     # List containers
nerdctl images                 # List images
nerdctl logs container         # Logs
nerdctl exec -it container sh  # Exec
nerdctl stats                  # Stats
nerdctl top container          # Top
nerdctl compose up             # Docker Compose compatible
```

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
