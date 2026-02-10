# TCloud Shell

**GPU-Enabled Development Shell for Kubernetes with Together AI**

Instantly provision GPU-enabled development environments with SSH and Web UI access on any Kubernetes cluster.

---

## 🚀 Quick Start

### Prerequisites
- Kubernetes cluster with GPU nodes
- NVIDIA GPU Operator installed
- Helm 3.x
- kubectl configured

### Installation

```bash
# 1. Generate SSH key
ssh-keygen -t rsa -b 4096 -f ~/.ssh/tcloud_key -N ""
export SSH_KEY=$(cat ~/.ssh/tcloud_key.pub | base64)

# 2. Install with Helm
helm install my-shell ./tcloud-shell-helm \
  --namespace tcloud-shell \
  --create-namespace \
  --set sshShell.authorizedKeys="$SSH_KEY"

# 3. Get access details
kubectl get svc -n tcloud-shell
```

### Access Your Shell

**SSH:**
```bash
export SSH_IP=$(kubectl get svc -n tcloud-shell my-shell-tcloud-shell-ssh -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
ssh -i ~/.ssh/tcloud_key tcloud@$SSH_IP
```

**Web UI:**
```bash
export WEB_IP=$(kubectl get svc -n tcloud-shell my-shell-tcloud-shell-web -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
open http://$WEB_IP
```

---

## 📋 Features

- **🎮 GPU Access** - NVIDIA H100/A100 GPUs with CUDA 12.6
- **🔐 SSH Shell** - Secure key-based authentication
- **🌐 Web Shell** - Browser-based terminal (ttyd)
- **💾 Persistent Storage** - Shared workspace with RWX volumes
- **🎨 Welcome Banner** - Branded Together AI experience
- **⚡ Fast Startup** - Ready in < 2 minutes
- **📦 Production Ready** - Battle-tested Helm chart

---

## ⚙️ Configuration

### Basic Configuration

Create a `values.yaml`:

```yaml
# GPU Configuration
sshShell:
  resources:
    requests:
      nvidia.com/gpu: 4
      memory: "64Gi"
      cpu: "16"

webShell:
  enabled: true
  resources:
    requests:
      nvidia.com/gpu: 2

# Storage
storage:
  size: 500Gi
  storageClassName: fast-ssd

# SSH Key
sshShell:
  authorizedKeys: "<base64-encoded-ssh-public-key>"
```

Install with custom values:

```bash
helm install my-shell ./tcloud-shell-helm \
  --namespace ml-team \
  --create-namespace \
  --values values.yaml
```

### Advanced Configuration

See [`tcloud-shell-helm/values.yaml`](tcloud-shell-helm/values.yaml) for all options:

| Parameter | Description | Default |
|-----------|-------------|---------|
| `sshShell.enabled` | Enable SSH shell | `true` |
| `webShell.enabled` | Enable web shell | `true` |
| `sshShell.resources.requests.nvidia.com/gpu` | GPU count per shell | `1` |
| `storage.size` | Persistent volume size | `100Gi` |
| `storage.storageClassName` | Storage class name | `shared-wekafs` |
| `user.name` | Shell username | `tcloud` |

---

## 📖 Usage Examples

### Example 1: High-Performance Setup (8 GPUs)

```bash
helm install gpu-shell ./tcloud-shell-helm \
  --namespace research \
  --create-namespace \
  --set sshShell.resources.requests.nvidia\.com/gpu=8 \
  --set sshShell.resources.requests.memory="256Gi" \
  --set storage.size="2Ti" \
  --set sshShell.authorizedKeys="$SSH_KEY"
```

### Example 2: Web-Only Environment

```bash
helm install web-shell ./tcloud-shell-helm \
  --namespace dev-team \
  --create-namespace \
  --set sshShell.enabled=false \
  --set webShell.enabled=true \
  --set sshShell.authorizedKeys="$SSH_KEY"
```

### Example 3: Multi-User Deployment

```bash
# User 1
helm install user1-shell ./tcloud-shell-helm \
  --namespace user1 \
  --create-namespace \
  --set sshShell.authorizedKeys="$USER1_SSH_KEY"

# User 2
helm install user2-shell ./tcloud-shell-helm \
  --namespace user2 \
  --create-namespace \
  --set sshShell.authorizedKeys="$USER2_SSH_KEY"
```

---

## 🔄 Management

### Upgrade

```bash
# Change GPU count
helm upgrade my-shell ./tcloud-shell-helm \
  --namespace tcloud-shell \
  --set sshShell.resources.requests.nvidia\.com/gpu=4 \
  --reuse-values

# View upgrade history
helm history my-shell -n tcloud-shell
```

### Pause/Resume

```bash
# Pause (scale to 0)
kubectl scale sts my-shell-tcloud-shell-ssh -n tcloud-shell --replicas=0
kubectl scale sts my-shell-tcloud-shell-web -n tcloud-shell --replicas=0

# Resume (scale to 1)
kubectl scale sts my-shell-tcloud-shell-ssh -n tcloud-shell --replicas=1
kubectl scale sts my-shell-tcloud-shell-web -n tcloud-shell --replicas=1
```

### Uninstall

```bash
# Remove deployment (keeps PVC)
helm uninstall my-shell --namespace tcloud-shell

# Delete PVC if needed
kubectl delete pvc -n tcloud-shell my-shell-tcloud-shell-storage
```

---

## 🛠️ Troubleshooting

### Pod Not Starting

```bash
# Check pod status
kubectl get pods -n tcloud-shell

# View logs
kubectl logs -n tcloud-shell my-shell-tcloud-shell-ssh-0

# Describe pod for events
kubectl describe pod -n tcloud-shell my-shell-tcloud-shell-ssh-0

# Check GPU availability
kubectl get nodes -o json | jq '.items[].status.allocatable."nvidia.com/gpu"'
```

### SSH Connection Issues

```bash
# Check service has external IP
kubectl get svc -n tcloud-shell

# Port forward as fallback
kubectl port-forward -n tcloud-shell svc/my-shell-tcloud-shell-ssh 2222:22

# Connect via localhost
ssh -i ~/.ssh/tcloud_key -p 2222 tcloud@localhost
```

### GPU Not Accessible

```bash
# Exec into pod
kubectl exec -it -n tcloud-shell my-shell-tcloud-shell-ssh-0 -- bash

# Check GPU
nvidia-smi
echo $CUDA_HOME
```

---

## 🏗️ Architecture

```
User Access
    │
    ├─── SSH (Port 22) ──────┐
    │                        │
    └─── HTTP (Port 80) ─────┤
                             │
                        LoadBalancer
                             │
                        ┌────┴────┐
                        │         │
                   SSH Shell  Web Shell
                   (StatefulSet) (StatefulSet)
                        │         │
                        └────┬────┘
                             │
                    Persistent Volume
                      (Shared Storage)
                             │
                        GPU Node Pool
                    (NVIDIA H100/A100)
```

**Components:**
- **StatefulSets**: Stable pods with persistent identity
- **LoadBalancer Services**: External access via cloud LB
- **PersistentVolume**: Shared workspace (RWX)
- **ConfigMaps**: Setup scripts and configuration
- **Secrets**: SSH authorized keys

---

## 📚 Documentation

- **[Architecture Details](ARCHITECTURE-SUMMARY.md)** - Complete system architecture
- **[Deployment Guide](DEPLOYMENT-SUMMARY.md)** - Detailed deployment walkthrough
- **[Helm Chart Guide](tcloud-shell-helm/README.md)** - Helm-specific documentation
- **[Complete Summary](COMPLETION-SUMMARY.md)** - Full project overview

---

## 🔐 Security

- **SSH**: Key-based authentication only (password auth disabled)
- **Root Access**: Root login disabled, sudo via `tcloud` user
- **Network**: LoadBalancer with session affinity
- **Storage**: Namespace-scoped PVCs
- **Container**: Non-privileged with minimal capabilities

**Production Recommendations:**
- Add OAuth/SSO for web shell
- Enable TLS/HTTPS via Ingress + cert-manager
- Implement NetworkPolicies for pod isolation
- Set resource quotas per namespace
- Regular security audits

---

## 🚧 Roadmap

### Phase 1 (Current - MVP)
- [x] SSH and Web shell deployment
- [x] GPU resource allocation
- [x] Persistent storage
- [x] Production-ready Helm chart

### Phase 2 (Next)
- [ ] Auto-pause/resume on idle
- [ ] OAuth/SSO authentication
- [ ] TLS/HTTPS support
- [ ] Pre-built images with ML libraries
- [ ] Together CLI integration

### Phase 3 (Future)
- [ ] Jupyter Lab integration
- [ ] VS Code Server option
- [ ] Multi-user collaboration
- [ ] Resource usage monitoring
- [ ] CLI wrapper (`tcloud shell create`)

---

## 🤝 Contributing

Contributions welcome! Please follow these steps:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

---

## 📄 License

Apache 2.0

---

## 💬 Support

- **Documentation**: See `/docs` directory
- **Issues**: https://github.com/pahluwalia-tcloud/tcloud-shell-poc/issues
- **Slack**: #tcloud-shell
- **Email**: support@together.ai

---

## ⭐ Acknowledgments

Built with:
- [NVIDIA GPU Operator](https://github.com/NVIDIA/gpu-operator)
- [ttyd](https://github.com/tsl0922/ttyd) - Web terminal
- [CUDA](https://developer.nvidia.com/cuda-toolkit) - GPU computing platform
- Kubernetes + Helm

---

**Made with ❤️ by Together AI**

*Reducing time-to-first-GPU from "confused and filing tickets" to "coding in 60 seconds"*
