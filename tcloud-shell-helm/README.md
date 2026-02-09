# TCloud Shell Helm Chart

GPU-enabled development shell for Kubernetes clusters with NVIDIA GPUs.

## Quick Start

### Prerequisites
- Kubernetes cluster with GPU nodes
- NVIDIA device plugin installed
- kubectl configured
- Helm 3.x installed

### Basic Installation

```bash
# Generate SSH key and encode
ssh-keygen -t rsa -b 4096 -f ~/.ssh/tcloud_shell_key -N ""
export SSH_KEY=$(cat ~/.ssh/tcloud_shell_key.pub | base64)

# Install with SSH key
helm install my-shell ./tcloud-shell-helm \
  --namespace tcloud-shell \
  --create-namespace \
  --set sshShell.authorizedKeys="$SSH_KEY"
```

### Custom Installation

```bash
# Create custom values file
cat > my-values.yaml <<EOF
sshShell:
  enabled: true
  resources:
    requests:
      nvidia.com/gpu: 2
  authorizedKeys: "<your-base64-encoded-ssh-key>"

webShell:
  enabled: true
  resources:
    requests:
      nvidia.com/gpu: 1

storage:
  size: 500Gi
  storageClassName: fast-ssd
EOF

# Install with custom values
helm install my-shell ./tcloud-shell-helm \
  --namespace tcloud-shell \
  --create-namespace \
  --values my-values.yaml
```

## Configuration

See [values.yaml](values.yaml) for all configurable parameters.

### Key Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `sshShell.enabled` | Enable SSH shell | `true` |
| `sshShell.resources.requests.nvidia.com/gpu` | GPU count | `1` |
| `webShell.enabled` | Enable web shell | `true` |
| `storage.size` | Storage size | `100Gi` |
| `storage.storageClassName` | Storage class | `shared-wekafs` |

## Usage

### Access SSH Shell

```bash
# Get external IP
export SSH_IP=$(kubectl get svc -n tcloud-shell my-shell-tcloud-shell-ssh -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Connect
ssh -i ~/.ssh/tcloud_shell_key tcloud@$SSH_IP
```

### Access Web Shell

```bash
# Get external IP
export WEB_IP=$(kubectl get svc -n tcloud-shell my-shell-tcloud-shell-web -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Open in browser
open http://$WEB_IP
```

## Upgrade

```bash
helm upgrade my-shell ./tcloud-shell-helm \
  --namespace tcloud-shell \
  --values my-values.yaml
```

## Uninstall

```bash
helm uninstall my-shell --namespace tcloud-shell
```

## Support

- Documentation: https://docs.together.ai
- Issues: https://github.com/together-ai/tcloud-shell
