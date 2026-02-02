# MetalLB Deployment Playbook Design

## Overview

Add a new Ansible playbook (`playbooks/metallb.yml`) that deploys MetalLB to the K3s cluster using Helm. This provides LoadBalancer service support for bare metal clusters.

## Configuration

| Setting | Value |
|---------|-------|
| IP Range | 192.168.44.200-192.168.44.220 |
| Mode | Layer 2 |
| Namespace | metallb-system |
| Chart Version | 0.14.9 |

## Implementation

### 1. Update Collection Dependencies

Add `kubernetes.core` to `collections/requirements.yml`:

```yaml
collections:
  - name: community.general
  - name: ansible.posix
  - name: kubernetes.core
```

### 2. Create Playbook

Create `playbooks/metallb.yml` with these tasks:

1. **Add Helm repository** - Register MetalLB Helm repo
2. **Deploy chart** - Install MetalLB via `kubernetes.core.helm`
3. **Wait for controller** - Ensure MetalLB pods are ready
4. **Create IPAddressPool** - Define the IP range for LoadBalancer services
5. **Create L2Advertisement** - Enable L2 mode announcements
6. **Verify deployment** - Confirm resources exist

### 3. Playbook Variables

```yaml
metallb_namespace: metallb-system
metallb_chart_version: "0.14.9"
metallb_ip_range: "192.168.44.200-192.168.44.220"
kubeconfig_path: "~/.kube/config"
kubeconfig_context: "k3s-ansible"
```

All variables can be overridden at runtime via `-e`.

## Usage

```bash
# Install dependencies
ansible-galaxy collection install -r collections/requirements.yml

# Deploy MetalLB
ansible-playbook playbooks/metallb.yml

# Override IP range
ansible-playbook playbooks/metallb.yml -e metallb_ip_range="192.168.44.150-192.168.44.180"
```

## Files Changed

- `collections/requirements.yml` - Add kubernetes.core dependency
- `playbooks/metallb.yml` - New playbook (create)
