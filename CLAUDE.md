# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

k3s-ansible is an Ansible Collection (`k3s.orchestration`) that automates deployment of K3s Kubernetes clusters. It supports single-server, multi-server HA (with embedded etcd), and air-gapped installations across multiple Linux distributions (Debian, Ubuntu, RHEL family, SUSE family, ArchLinux, Alpine) and architectures (x64, arm64, armhf).

## Commands

### Linting
```bash
pip3 install yamllint ansible-lint ansible
yamllint .
ansible-lint
```

### Running Playbooks
```bash
# Install collection dependencies first
ansible-galaxy collection install -r collections/requirements.yml

# Deploy K3s cluster
ansible-playbook playbooks/site.yml -i inventory.yml

# Upgrade K3s version (update k3s_version in inventory first)
ansible-playbook playbooks/upgrade.yml -i inventory.yml

# Apply OS package updates with reboot-if-required (drain/cordon for agents)
ansible-playbook playbooks/os-upgrade.yml -i inventory.yml

# Apply OS patches and then upgrade k3s in a single run
ansible-playbook playbooks/upgrade-all.yml -i inventory.yml

# Remove K3s from all nodes
ansible-playbook playbooks/reset.yml -i inventory.yml

# Reboot cluster nodes
ansible-playbook playbooks/reboot.yml -i inventory.yml

# Reboot a single node (hosts are IPs in inventory-turingpi.yml)
ansible-playbook playbooks/reboot.yml -i inventory-turingpi.yml --limit 192.168.44.73

# Re-fetch kubeconfig only
ansible-playbook playbooks/site.yml -i inventory.yml --tags kubeconfig
```

### Operational Plays (bartram homelab)

Cluster nodes: `192.168.44.71` (server), `.72`–`.74` (agents); tp-3 = `192.168.44.73`.

```bash
# Graceful cluster shutdown: agents first, then the server, then verify down
ansible all -i inventory.yml -m ping -o                                  # confirm reachable
ansible agent -i inventory.yml -b -m shell -a "shutdown -h +0" -B 1 -P 0
ansible server -i inventory.yml -b -m shell -a "shutdown -h +0" -B 1 -P 0
for ip in 71 72 73 74; do ping -c 1 -W 2 192.168.44.$ip >/dev/null 2>&1 && echo "$ip UP" || echo "$ip down"; done
```

**LoadBalancer IP unreachable from outside (metallb L2, historically tp-3):** symptom is pod Running and in-cluster curl 200 while the external LB IP times out — metallb elected the IP's announcer onto a node whose speaker is crash-looping (`bind: address already in use` on port 7946, caused by an orphaned `/speaker` host process). Deleting the speaker pod does NOT clear the orphan; **reboot the node** (resolved the 2026-06-06 budget outage on tp-3). Full playbook: budget project memory `tp3-metallb-speaker-broken.md`.

### Rebooting a node when SSH is broken

If a node refuses SSH (so `reboot.yml` can't reach it) but its kubelet is still healthy,
reboot it through Kubernetes with a privileged node-debug pod:

```bash
kubectl debug node/tp-3 --image=busybox --profile=sysadmin -- chroot /host reboot
```

The pod mounts the node's root filesystem at `/host`, so `chroot /host <cmd>` runs as root
on the node itself. Used 2026-08-24 to clear the tp-3 metallb-speaker outage (see
HOMELAB.md "Known flaky") while tp-3's `authorized_keys` was broken.

### Local Testing with Vagrant
```bash
vagrant up  # Provisions 5-node cluster (3 servers, 2 agents)
```

### CI Integration Tests (Docker-based)
The CI runs tests against `tests/basic.yml` (single server + agent) and `tests/ha.yml` (3-server HA) inventories using Docker containers with both systemd and openrc init systems.

## Architecture

### Playbook Execution Flow

**site.yml** runs three plays in sequence:
1. **Cluster prep** (all nodes): `prereq` → `airgap` → `raspberrypi` roles
2. **K3s server setup** (server group): `k3s_server` role
3. **K3s agent setup** (agent group): `k3s_agent` role

**upgrade.yml** upgrades servers serially (one at a time), then agents in parallel.

### Roles

| Role | Purpose |
|------|---------|
| `prereq` | System preparation: IPv4/IPv6 forwarding, iptables/nftables, SELinux, kernel modules |
| `k3s_server` | Control plane: downloads K3s, initializes first server with `--cluster-init` for HA, joins additional servers |
| `k3s_agent` | Worker nodes: downloads K3s, joins agents to cluster |
| `k3s_upgrade` | Handles version upgrades with service file regeneration |
| `os_upgrade` | OS package patching: apt update/upgrade, optional autoremove and cordon/drain on agents, reboot when `/var/run/reboot-required` is set |
| `airgap` | Distributes K3s binary, install script, and images for offline installations |
| `raspberrypi` | Pi-specific cgroup and device tree fixes |

### Inventory Structure

```yaml
k3s_cluster:
  children:
    server:    # Control plane nodes (odd number for HA: 3, 5, 7)
      hosts: ...
    agent:     # Worker nodes
      hosts: ...
  vars:
    k3s_version: v1.31.12+k3s1
    token: "..."           # Cluster token (auto-generated if omitted)
    api_endpoint: "..."    # API server endpoint for agents
```

### Key Variables

**Required:** `k3s_version`, `api_endpoint`, `ansible_user`

**Optional:** `extra_server_args`, `extra_agent_args`, `server_config_yaml`, `agent_config_yaml`, `registries_config_yaml`, `airgap_dir`

## Code Style

- YAML truthy values: use `true`/`false` (not `yes`/`no`)
- Max line length: 180 characters (warning level)
- ansible-lint warnings allowed: `var-naming[no-role-prefix]`, `yaml[comments-indentation]`, `yaml[line-length]`

## Dependencies

- Ansible 8.0+ (ansible-core 2.15+)
- Collections: `community.general` (>=7.0.0), `ansible.posix` (>=1.5.0)

## Homelab

Shared infra facts (registry, pg, k3s nodes, LB IPs, deploy conventions): @../HOMELAB.md
