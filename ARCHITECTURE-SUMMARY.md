# TCloud Shell POC - Architecture Summary

## Overview

The TCloud Shell POC provides GPU-enabled development environments accessible via SSH and Web UI, deployed on Kubernetes with minimal complexity for end users.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Access Layer                        │
├──────────────────────────────┬──────────────────────────────────┤
│   SSH Client                 │   Web Browser                     │
│   ssh tcloud@172.252.69.159  │   http://172.252.69.170          │
└──────────────┬───────────────┴────────────────┬─────────────────┘
               │                                 │
               │                                 │
┌──────────────▼────────────────────────────────▼─────────────────┐
│                    LoadBalancer Services                         │
├──────────────────────────────┬──────────────────────────────────┤
│  tcloud-shell (SSH)          │  tcloud-web-shell (Web)          │
│  External IP: 172.252.69.159 │  External IP: 172.252.69.170     │
│  Port: 22                    │  Port: 80 → 7681                 │
└──────────────┬───────────────┴────────────────┬─────────────────┘
               │                                 │
               │                                 │
┌──────────────▼─────────────────────────────────▼────────────────┐
│                   Kubernetes Cluster (tcloud-test-pav)           │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │              Namespace: tcloud-shell                        ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                   │
│  ┌────────────────────────┐    ┌────────────────────────┐       │
│  │   StatefulSet: SSH     │    │  StatefulSet: Web      │       │
│  │   tcloud-shell-0       │    │  tcloud-web-shell-0    │       │
│  ├────────────────────────┤    ├────────────────────────┤       │
│  │ Container:             │    │ Container:             │       │
│  │ - CUDA 12.6 Base       │    │ - CUDA 12.6 Base       │       │
│  │ - SSH Server           │    │ - ttyd (Web Terminal)  │       │
│  │ - GPU: 1x H100         │    │ - GPU: 1x H100         │       │
│  │ - Memory: 32Gi         │    │ - Memory: 32Gi         │       │
│  │ - CPU: 8 cores         │    │ - CPU: 8 cores         │       │
│  └───────────┬────────────┘    └────────────┬───────────┘       │
│              │                               │                   │
│              └───────────┬───────────────────┘                   │
│                          │                                       │
│                          ▼                                       │
│              ┌───────────────────────┐                          │
│              │ Persistent Volume     │                          │
│              │ tcloud-shell-storage  │                          │
│              │ - Size: 1Ti           │                          │
│              │ - Type: WekaFS (RWX)  │                          │
│              │ - Mount: /workspace   │                          │
│              └───────────────────────┘                          │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              GPU Node: gpu-dp-n9cvq-68pzl                │   │
│  │  - Hardware: 8x NVIDIA H100 80GB HBM3                    │   │
│  │  - OS: Ubuntu 22.04                                      │   │
│  │  - NVIDIA Drivers: Installed                             │   │
│  │  - Device Plugin: Active                                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────┘
```

---

## Components Deployed

### 1. Namespace
- **Name:** `tcloud-shell`
- **Purpose:** Isolated environment for TCloud Shell resources
- **Resources:** All components deployed within this namespace

### 2. SSH Shell (StatefulSet)

**Pod:** `tcloud-shell-0`

**Container Details:**
- **Image:** `nvidia/cuda:12.6.0-devel-ubuntu22.04`
- **Purpose:** Provides SSH access to GPU environment
- **Entry Point:** Custom bash script that:
  - Installs OpenSSH server
  - Creates `tcloud` user (uid 1000)
  - Configures SSH with key-based auth
  - Mounts shared workspace
  - Starts SSH daemon

**Resource Allocation:**
- **GPU:** 1x NVIDIA H100 (can request up to 8)
- **Memory:** 32Gi request, 64Gi limit
- **CPU:** 8 cores request, 16 cores limit

**Volumes:**
- `/workspace` → Persistent shared storage (1Ti WekaFS)
- `/ssh-keys` → SSH authorized_keys secret
- `/scripts` → Setup scripts ConfigMap
- `/dev/shm` → Shared memory (32Gi)

**Networking:**
- **Service:** LoadBalancer
- **External IP:** 172.252.69.159
- **Port:** 22 (SSH)

### 3. Web Shell (StatefulSet)

**Pod:** `tcloud-web-shell-0`

**Container Details:**
- **Image:** `nvidia/cuda:12.6.0-devel-ubuntu22.04`
- **Purpose:** Provides browser-based terminal access
- **Entry Point:** Custom bash script that:
  - Installs ttyd (web terminal)
  - Creates `tcloud` user
  - Configures terminal theme
  - Starts ttyd server on port 7681

**Resource Allocation:**
- **GPU:** 1x NVIDIA H100 (can request up to 8)
- **Memory:** 32Gi request, 64Gi limit
- **CPU:** 8 cores request, 16 cores limit

**Volumes:**
- `/workspace` → Persistent shared storage (1Ti WekaFS)
- `/scripts` → Setup scripts ConfigMap
- `/dev/shm` → Shared memory (32Gi)

**Networking:**
- **Service:** LoadBalancer
- **External IP:** 172.252.69.170
- **Port:** 80 → 7681 (HTTP/WebSocket)

### 4. Persistent Storage

**PVC:** `tcloud-shell-storage`

**Details:**
- **Storage Class:** `shared-wekafs` (CSI driver)
- **Capacity:** 1Ti
- **Access Mode:** ReadWriteMany (RWX)
- **Status:** Bound to `test-storage` PV
- **Mount Path:** `/workspace`
- **Shared:** Both SSH and Web shells access same storage

### 5. Configuration Resources

**Secrets:**
- `tcloud-shell-ssh-keys`: Base64 encoded SSH authorized_keys

**ConfigMaps:**
- `tcloud-shell-setup`: SSH shell entrypoint and setup scripts
- `tcloud-web-shell-setup`: Web shell entrypoint and ttyd config

### 6. Services

| Service | Type | External IP | Port | Target |
|---------|------|-------------|------|--------|
| tcloud-shell | LoadBalancer | 172.252.69.159 | 22 | SSH pod port 22 |
| tcloud-web-shell | LoadBalancer | 172.252.69.170 | 80 | Web pod port 7681 |

---

## Data Flow

### SSH Access Flow
```
1. User runs: ssh -i key tcloud@172.252.69.159
2. LoadBalancer receives connection on port 22
3. Traffic routed to tcloud-shell-0:22
4. SSH daemon authenticates using public key
5. User logged in as 'tcloud' with shell access
6. User has access to:
   - 1x H100 GPU (via nvidia-smi, CUDA)
   - /workspace (1Ti shared storage)
   - Python, git, and other tools
```

### Web Access Flow
```
1. User opens: http://172.252.69.170 in browser
2. LoadBalancer receives HTTP request on port 80
3. Traffic routed to tcloud-web-shell-0:7681
4. ttyd serves web terminal interface
5. WebSocket connection established
6. User gets browser-based terminal with:
   - Same environment as SSH shell
   - Same workspace access
   - Same GPU access
```

### Storage Access Flow
```
1. Both pods mount PVC at /workspace
2. PVC backed by WekaFS CSI driver
3. WekaFS provides:
   - High-performance parallel filesystem
   - RWX (multi-pod concurrent access)
   - 1Ti capacity
4. Files written in SSH shell visible in Web shell (and vice versa)
```

---

## Key Technical Decisions

### 1. Why StatefulSet?
- **Stable hostname:** Pods get consistent names (tcloud-shell-0)
- **Persistent identity:** Pod recreation maintains identity
- **Ordered deployment:** Predictable deployment order
- **Storage binding:** Stable PVC associations

### 2. Why WekaFS Storage?
- **Performance:** High-throughput parallel filesystem
- **RWX Mode:** Multiple pods can read/write simultaneously
- **Shared Access:** Both shells access same workspace
- **Together.AI Standard:** Default storage class in cluster

### 3. Why LoadBalancer Services?
- **External Access:** Direct access from internet
- **Simple:** No ingress controller needed for POC
- **Session Affinity:** Maintains connections to same pod
- **Cloud Native:** Leverages cloud provider LB

### 4. Why Separate Pods?
- **Independence:** SSH and Web can scale separately
- **Resource Isolation:** Each has dedicated GPU
- **Fault Tolerance:** One fails, other continues
- **User Choice:** Users can choose access method

### 5. Why CUDA Base Image?
- **GPU Ready:** NVIDIA drivers pre-installed
- **CUDA Toolkit:** Development tools included
- **Minimal:** Start lean, install as needed
- **Official:** Maintained by NVIDIA

---

## Resource Requirements

### Per Environment (SSH + Web)
- **GPUs:** 2 (1 per shell)
- **Memory:** 64 GiB (32 GiB per shell)
- **CPUs:** 16 cores (8 per shell)
- **Storage:** 1 TiB (shared)
- **External IPs:** 2 (1 per LoadBalancer)

### Cluster Requirements
- **GPU Nodes:** At least 1 node with NVIDIA GPUs
- **GPU Driver:** NVIDIA device plugin installed
- **Storage:** CSI driver for dynamic PVC provisioning
- **LoadBalancer:** Cloud provider LB support

---

## Security Model

### Authentication
- **SSH Shell:** Public key authentication only
- **Web Shell:** No authentication (TODO for production)

### Authorization
- **User:** `tcloud` (uid 1000)
- **Sudo:** Full sudo access (NOPASSWD)
- **Container:** Non-privileged (SYS_ADMIN capability only)

### Network
- **Isolation:** Namespace-level isolation
- **Exposure:** Direct external access via LoadBalancer
- **Firewall:** Cloud provider security groups apply

### Storage
- **Access:** Root filesystem ephemeral, /workspace persistent
- **Permissions:** User owns /workspace
- **Isolation:** Namespace-scoped PVC

---

## Scaling Considerations

### Horizontal Scaling
- **Current:** 1 replica per StatefulSet
- **Possible:** Scale to N replicas for multi-user
- **Limitation:** GPU availability per node

### Vertical Scaling
- **GPU:** Increase from 1 to 8 per pod
- **Memory:** Increase up to node capacity
- **CPU:** Increase based on workload

### Multi-Tenant
- **Approach 1:** Multiple namespaces (1 per user)
- **Approach 2:** Multiple releases in same namespace
- **Approach 3:** Scale replicas + unique external IPs

---

## Monitoring Points

### Pod Health
- **Readiness Probes:** TCP check (SSH), HTTP check (Web)
- **Liveness Probes:** Restart on failure
- **Resource Usage:** GPU, memory, CPU metrics

### Service Availability
- **LoadBalancer Status:** External IP assigned
- **Endpoint Health:** Pod backing service
- **Connection Success:** Actual user connections

### Storage
- **PVC Status:** Bound state
- **Capacity:** Usage vs. available
- **Performance:** I/O metrics (if available)

### GPU
- **Allocation:** GPU assigned to pod
- **Utilization:** GPU compute usage
- **Memory:** GPU memory usage
- **Temperature:** GPU health metrics

---

## Future Enhancements

### Phase 2
1. **Auto-pause/resume:** Idle detection + scale to 0
2. **Authentication:** OAuth/SSO for web shell
3. **TLS/HTTPS:** Secure web access
4. **Resource quotas:** Limit GPU time/usage
5. **Monitoring:** Prometheus + Grafana dashboards

### Phase 3
1. **Jupyter Lab:** Alternative to ttyd
2. **VS Code Server:** Full IDE in browser
3. **Multi-user:** Shared environments
4. **CLI Tool:** `tcloud shell create/delete/list`
5. **Cost Tracking:** Usage-based billing

---

## Related Documents

- **[README.md](README.md)** - Detailed documentation and troubleshooting
- **[DEPLOYMENT-SUMMARY.md](DEPLOYMENT-SUMMARY.md)** - Deployment status and access info
- **[HELM-DESIGN.md](HELM-DESIGN.md)** - Helm chart design (in progress)

---

## Quick Reference

### Access URLs
- **SSH:** `ssh -i tcloud_shell_key tcloud@172.252.69.159`
- **Web:** `http://172.252.69.170`

### Management
```bash
# View all resources
kubectl get all -n tcloud-shell

# Check logs
kubectl logs -n tcloud-shell tcloud-shell-0
kubectl logs -n tcloud-shell tcloud-web-shell-0

# Scale down (pause)
kubectl scale sts tcloud-shell -n tcloud-shell --replicas=0
kubectl scale sts tcloud-web-shell -n tcloud-shell --replicas=0

# Scale up (resume)
kubectl scale sts tcloud-shell -n tcloud-shell --replicas=1
kubectl scale sts tcloud-web-shell -n tcloud-shell --replicas=1

# Delete everything
kubectl delete namespace tcloud-shell
```

---

**Status:** ✅ POC Deployed and Operational
**Cluster:** tcloud-test-pav
**Deployment Date:** 2026-02-09
**Version:** 0.1.0 (POC)
