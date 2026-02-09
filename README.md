# TCloud Shell POC

A GPU-enabled development environment with SSH access for Together.AI clusters.

## Overview

TCloud Shell provides developers with instant SSH access to GPU-enabled environments, abstracting away Kubernetes complexity.

### Features

- **Instant GPU Access**: Pre-configured NVIDIA H100 GPUs with CUDA toolkit
- **SSH Workflow**: Standard SSH access, no kubectl required
- **Persistent Storage**: Shared WekaFS volume for persistent workspace
- **Pre-installed Tools**: CUDA, PyTorch, TensorFlow, JAX, Together CLI
- **Production-Ready**: Built on Kubernetes StatefulSets with proper resource management

## Architecture

```
User → LoadBalancer → StatefulSet (GPU Pod) → Persistent Volume
                            ↓
                      CUDA Container + SSH Server
```

### Components

1. **StatefulSet**: Manages GPU-enabled pods with persistent identity
2. **CUDA Container**: nvidia/cuda:12.6.0-devel-ubuntu22.04 base image
3. **SSH Server**: Configured for key-based authentication
4. **PVC**: 100Gi shared storage on WekaFS
5. **LoadBalancer Service**: External SSH access

## Prerequisites

- Kubernetes cluster with GPU nodes
- kubectl configured for target cluster
- GPU node with NVIDIA device plugin installed

## Quick Start

### 1. Deploy to Cluster

```bash
# Switch to tcloud-test-pav cluster
kubectl config use-context <cluster-context>

# Deploy all resources
kubectl apply -k .

# Or deploy individually
kubectl apply -f namespace.yaml
kubectl apply -f ssh-secret.yaml
kubectl apply -f setup-configmap.yaml
kubectl apply -f pvc.yaml
kubectl apply -f statefulset.yaml
kubectl apply -f service.yaml
```

### 2. Wait for LoadBalancer IP

```bash
kubectl get svc -n tcloud-shell tcloud-shell -w
```

### 3. Connect via SSH

```bash
# Get the external IP
EXTERNAL_IP=$(kubectl get svc -n tcloud-shell tcloud-shell -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Connect using the private key
ssh -i tcloud_shell_key tcloud@${EXTERNAL_IP}
```

### 4. Test GPU Access

```bash
# Once connected
nvidia-smi
python3 -c "import torch; print(torch.cuda.is_available())"
```

## Configuration

### GPU Resources

Edit `statefulset.yaml` to adjust GPU allocation:

```yaml
resources:
  requests:
    nvidia.com/gpu: 1  # Change to desired GPU count
```

### Storage

Edit `pvc.yaml` to adjust storage size:

```yaml
resources:
  requests:
    storage: 100Gi  # Change to desired size
```

### Node Selection

StatefulSet includes node selector for GPU nodes:

```yaml
nodeSelector:
  nvidia.com/gpu.present: "true"
```

## Monitoring

```bash
# Check pod status
kubectl get pods -n tcloud-shell

# View logs
kubectl logs -n tcloud-shell tcloud-shell-0

# Describe pod for events
kubectl describe pod -n tcloud-shell tcloud-shell-0

# Check PVC
kubectl get pvc -n tcloud-shell

# Check service
kubectl get svc -n tcloud-shell
```

## Troubleshooting

### Pod not starting

```bash
# Check events
kubectl describe pod -n tcloud-shell tcloud-shell-0

# Check GPU allocation
kubectl get nodes -o json | jq '.items[].status.allocatable."nvidia.com/gpu"'
```

### SSH connection fails

```bash
# Verify service has external IP
kubectl get svc -n tcloud-shell

# Port forward as fallback
kubectl port-forward -n tcloud-shell svc/tcloud-shell 2222:22

# Then connect via localhost
ssh -i tcloud_shell_key -p 2222 tcloud@localhost
```

### GPU not accessible

```bash
# Exec into pod to debug
kubectl exec -it -n tcloud-shell tcloud-shell-0 -- bash

# Check GPU visibility
nvidia-smi
echo $CUDA_HOME
```

## Cleanup

```bash
# Delete all resources
kubectl delete -k .

# Or delete namespace (deletes everything)
kubectl delete namespace tcloud-shell
```

## Security Notes

- SSH uses key-based authentication only (password auth disabled)
- Root login is disabled
- User runs as non-root (uid 1000)
- Private key is stored locally in `tcloud_shell_key`

## Next Steps

1. ✅ Basic deployment working
2. 🔄 Create Helm chart for parameterization
3. 🔄 Add auto-pause/resume functionality
4. 🔄 Implement CLI wrapper (`tcloud shell create`)
5. 🔄 Add web UI integration

## Files

- `namespace.yaml`: Creates tcloud-shell namespace
- `ssh-secret.yaml`: Contains SSH authorized_keys
- `setup-configmap.yaml`: Setup scripts for SSH and tools
- `pvc.yaml`: Persistent volume claim for storage
- `statefulset.yaml`: Main GPU-enabled pod definition
- `service.yaml`: LoadBalancer for external access
- `kustomization.yaml`: Kustomize configuration
- `tcloud_shell_key`: Private SSH key (keep secure!)
- `tcloud_shell_key.pub`: Public SSH key
