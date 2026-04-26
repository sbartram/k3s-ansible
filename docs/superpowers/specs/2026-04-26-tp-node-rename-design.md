# Design — Rename k3s nodes to `tp-1` … `tp-4`

**Date:** 2026-04-26
**Status:** Approved (pending written-spec review)
**Related:** `TODO.md` ("Rename cluster nodes for consistency"), `docs/superpowers/specs/2026-04-26-turingpi-os-k3s-upgrade-design.md`

## 1. Goal and scope

Rename the four k3s nodes so their names match the Turing Pi slot numbers.

| Slot | IP             | Current k3s node name | Target |
|------|----------------|-----------------------|--------|
| 1    | 192.168.44.71  | `control-plane.local` | `tp-1` |
| 2    | 192.168.44.72  | `worker2.local`       | `tp-2` |
| 3    | 192.168.44.73  | `tp-3`                | `tp-3` (no change) |
| 4    | 192.168.44.74  | `worker4.local`       | `tp-4` |

After this work, `kubectl get nodes` is self-explanatory and `inventory-turingpi.yml` no longer needs per-host `os_upgrade_node_name` overrides.

### In scope

- Change OS hostname (`/etc/hostname`, `/etc/hosts`) on each node.
- Drain → delete-node → restart k3s so each node re-registers under its new name.
- Mandatory pre-rename backup of `/var/lib/rancher/k3s/server/` before renaming `tp-1`.
- A markdown runbook at `docs/superpowers/runbooks/rename-tp-nodes.md` with copy-paste commands and verification gates between every step.
- Update `inventory-turingpi.yml` to remove the per-host `os_upgrade_node_name` overrides once all four hostnames match the k3s node names.
- Delete the corresponding entry from `TODO.md`.

### Out of scope

- Any new playbook or Ansible role. The rename is a one-shot operation; a runbook with explicit verification gates is safer than wrapping it in untested automation.
- Migrating local-path PVCs. Cluster state lives on NFS, not on agent-local storage, so node-name-bound `local-path` PVs are not a concern.
- Proactive TLS SAN reconfiguration. `kubectl` and agents use `https://192.168.44.71:6443` (an IP, via `api_endpoint`), so cert SANs based on hostname are irrelevant. Verified post-rename.
- Touching mDNS / avahi configuration up-front. Observe behavior after rename and act only if `.local` reappears (the runbook contains the fix if it does).
- Any change to HA topology, etcd, or node roles.

## 2. Context and key facts

These shape the design and must hold for the procedure to be correct.

- **Topology:** 1 server + 3 agents (per `inventory-turingpi.yml`). NOT HA. The control plane is a single point of failure — renaming it means full Kubernetes API downtime. Embedded sqlite at `/var/lib/rancher/k3s/server/db/state.db`. No etcd quorum to preserve.
- **`--node-name` is not set anywhere** in `roles/k3s_server` or `roles/k3s_agent`. k3s defaults `--node-name` to the system hostname, so renaming = changing `/etc/hostname` and restarting k3s. No new Ansible variable needed.
- **Agents reference the server by IP** (`api_endpoint: 192.168.44.71`), not hostname. Renaming the control plane does not break agent rejoin config.
- **Workloads:** stateful data lives on NFS. Brief outages are acceptable. No local-path PV nodeAffinity surgery required.
- **MetalLB** is deployed (L2 mode, pool `192.168.44.200-220`). Speakers run as DaemonSet and re-elect on node restart.
- **`os_upgrade_node_name`** in `roles/os_upgrade/defaults/main.yml` defaults to `{{ ansible_hostname }}`. Once OS hostnames match k3s node names, the per-host overrides in `inventory-turingpi.yml` become unnecessary and are removed.

## 3. Design decisions

| # | Decision | Choice | Rationale |
|---|----------|--------|-----------|
| 1 | Rollout granularity | One node at a time, manually invoked, verify between each | Maximum control; halt-on-anomaly is trivial. |
| 2 | Rename mechanism | Hostname change + drain + delete-node + restart k3s | Preserves in-cluster state, no image re-pull, no full reinstall. |
| 3 | Deliverable shape | Runbook (markdown) + small inventory commit | One-shot operation; a tested-once playbook offers little value. |
| 4 | Node order | `tp-3` (no-op verify) → `tp-2` → `tp-4` → `tp-1` | Shakedown pass on the already-correct node, then real renames, control plane last. |
| 5 | Rollback strategy | Revert hostname + restart for agents; pre-rename sqlite backup + revert for control plane | Agents are stateless from cluster's POV; control plane is irreplaceable. |

## 4. Per-node procedure (agents)

For each agent node (`tp-2`, `tp-4`; `tp-3` runs only the non-destructive parts as a verification pass), the runbook prescribes:

### Preflight (from laptop)

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide --field-selector spec.nodeName=<old>
```

**Gate:** all nodes Ready, no PodDisruptionBudgets blocking drain.

### 1. Cordon and drain

```bash
kubectl cordon <old>
kubectl drain <old> --ignore-daemonsets --delete-emptydir-data --timeout=300s
```

**Gate:** drain returns 0; only DaemonSet pods remain on `<old>`.

### 2. Stop k3s-agent on the target

```bash
sudo systemctl stop k3s-agent
```

### 3. Rename the host

```bash
sudo hostnamectl set-hostname <new>
sudo sed -i "s/\b<old>\b/<new>/g" /etc/hosts
grep -rn "<old>" /etc/hostname /etc/hosts   # must return nothing
```

For nodes whose old name has a `.local` suffix (`worker2.local`, `worker4.local`), the runbook spells out the exact replacement explicitly per host — no clever sed for the suffix case.

### 4. Delete the old node from the cluster (from laptop)

```bash
kubectl delete node <old>
```

### 5. Restart k3s-agent (on target)

```bash
sudo systemctl start k3s-agent
```

k3s-agent re-registers using the new hostname.

### 6. Verify (from laptop)

```bash
kubectl get nodes
kubectl wait --for=condition=Ready node/<new> --timeout=180s
kubectl uncordon <new>
kubectl get pods -A -o wide --field-selector spec.nodeName=<new>
```

**Gate:** `<new>` Ready, `<old>` gone, workloads schedule back. **Do not proceed to next node until this gate passes.**

### `tp-3` dry-run

`tp-3` is already named correctly, so the rename steps (2, 3, 4, 5) are skipped. The runbook directs only:
- Preflight
- Step 1 (cordon and drain) — **this is a real drain**: workloads on `tp-3` are rescheduled to `tp-2`/`tp-4` for the duration. The point is to validate drain mechanics (PDBs, stuck pods, taint behavior) on a node that we don't have to bring back under a new identity.
- Step 6 (uncordon and verify) — workloads can return to `tp-3`.

Cost: a few minutes of pod rescheduling. Benefit: every gate in the runbook is exercised before we touch anything destructive on a misnamed node.

## 5. Per-node procedure (control plane, `tp-1`)

The control plane rename is qualitatively different: the API stops while k3s is down, and there's no second control plane to drain to. Acceptance: API downtime of ~1–3 minutes is expected.

### Preflight (from laptop)

```bash
kubectl get nodes -o wide                 # tp-2, tp-3, tp-4 already renamed
kubectl get pods -A -o wide               # snapshot
kubectl get pv,pvc -A                     # snapshot
```

**Gate:** `tp-2`, `tp-3`, `tp-4` all Ready; only the control plane still carries an old name.

### 1. Backup, rename, restart (on `192.168.44.71`)

```bash
sudo systemctl stop k3s
sudo tar -czf /root/k3s-server-backup-$(date +%Y%m%d-%H%M%S).tar.gz \
    -C /var/lib/rancher k3s/server
ls -la /root/k3s-server-backup-*.tar.gz   # confirm exists, non-zero

sudo hostnamectl set-hostname tp-1
sudo sed -i "s/\bcontrol-plane\.local\b/tp-1/g; s/\bcontrol-plane\b/tp-1/g" /etc/hosts
grep -rn "control-plane" /etc/hostname /etc/hosts   # must return nothing

sudo systemctl start k3s
sudo systemctl status k3s --no-pager
```

k3s starts and registers itself as a new node `tp-1`. The old `control-plane.local` node entry remains in the cluster but is NotReady (no kubelet posts to it). The backup is mandatory and is the prerequisite for rollback Case 2 (Section 6).

**Why not delete the old node entry before stopping k3s?** With k3s still running, its kubelet would re-register the node under the same hostname (`control-plane.local`) seconds later — the delete is racy and effectively a no-op. The clean approach is to rename first and let the new entry register, then remove the now-stale old entry in step 3.

### 2. Verify (from laptop, may need to wait ~60s for API)

```bash
kubectl get nodes
kubectl wait --for=condition=Ready node/tp-1 --timeout=300s
```

**Gate:** `tp-1` is Ready. There may also be a `control-plane.local` entry in NotReady state — that is expected and cleaned up in the next step.

### 3. Delete the stale old entry (from laptop)

```bash
kubectl delete node control-plane.local
```

### 4. Final verification (from laptop)

```bash
kubectl get nodes
kubectl get pods -A -o wide | grep -v Running
kubectl get apiservice
kubectl version
```

**Gate:** four nodes named `tp-1`–`tp-4`, all Ready; no stale `control-plane.local` entry; all APIServices True; `kubectl version` returns server version with no cert errors.

### 5. mDNS / `.local` watch

After reboot or NetworkManager reload, `hostname -f` may re-acquire `.local`. The runbook includes a follow-up step to set the hostname statically (`hostnamectl set-hostname tp-1 --static`) and to verify `/etc/systemd/resolved.conf` does not have `MulticastDNS=yes` re-publishing.

## 6. Rollback

### Agent rollback (`tp-2` or `tp-4` won't come back Ready)

On the affected agent:

```bash
sudo systemctl stop k3s-agent
sudo hostnamectl set-hostname <old>
sudo sed -i "s/\b<new>\b/<old>/g" /etc/hosts
sudo systemctl start k3s-agent
```

From laptop:

```bash
kubectl delete node <new>
kubectl wait --for=condition=Ready node/<old> --timeout=180s
kubectl uncordon <old>
```

**Halt the rollout** and investigate before continuing.

### Control plane rollback

**Case 1: k3s started but is in a bad state** (NotReady, won't accept connections, cert errors). Try the lightweight rollback first:

```bash
# On 192.168.44.71:
sudo systemctl stop k3s
sudo hostnamectl set-hostname control-plane.local
sudo sed -i "s/\btp-1\b/control-plane.local/g" /etc/hosts
sudo systemctl start k3s
# From laptop:
kubectl wait --for=condition=Ready node/control-plane.local --timeout=300s
kubectl delete node tp-1   # drop the dead entry
```

**Case 2: k3s won't start at all, or in-cluster state looks corrupted.** Restore from the backup taken in Section 5 step 1:

```bash
# On 192.168.44.71:
sudo systemctl stop k3s
sudo rm -rf /var/lib/rancher/k3s/server
sudo tar -xzf /root/k3s-server-backup-<timestamp>.tar.gz -C /var/lib/rancher
sudo hostnamectl set-hostname control-plane.local
sudo sed -i "s/\btp-1\b/control-plane.local/g" /etc/hosts
sudo systemctl start k3s
# From laptop:
kubectl wait --for=condition=Ready node/control-plane.local --timeout=300s
```

### Don't-do list

- **Never** run `reset.yml --limit 192.168.44.71` as a rollback. `reset.yml` calls `k3s-uninstall.sh` which wipes the cluster. There is no rejoin because there's no other server (single-server topology).
- **Never** run `site.yml` partway through the rollout. It will reconfigure k3s based on `ansible_hostname`, which may now mismatch the node name k3s already registered.
- Inventory rollback is `git checkout inventory-turingpi.yml` if needed.

If both control-plane rollback paths fail, the cluster's control plane is dead. Agent kubelets keep workload pods running locally without the API. Recovery is a fresh `site.yml` run plus redeploying MetalLB and workloads, with NFS-backed state rejoinable. This is the worst case and is why the backup is mandatory.

## 7. Success criteria

`kubectl get nodes` shows exactly four nodes named `tp-1`, `tp-2`, `tp-3`, `tp-4`, all Ready, all on `v1.33.10+k3s1` (or whichever `k3s_version` is current in inventory). On each host, `hostname` and `hostname -f` both return `tp-N` with no `.local` suffix.

### End-state verification (from laptop)

```bash
# Names and readiness
kubectl get nodes
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' | sort
# Expect: tp-1, tp-2, tp-3, tp-4. Nothing else.

# System health
kubectl get pods -n kube-system
kubectl get pods -n metallb-system
kubectl get pods -A | grep -vE '^(NAMESPACE|.*Running|.*Completed)'
# Expect: empty.

# Storage
kubectl get pvc -A
# Expect: all Bound.

# LoadBalancer IPs still advertised
kubectl get svc -A --field-selector spec.type=LoadBalancer
# Expect: each has an EXTERNAL-IP from 192.168.44.200-220.

# Hostname stickiness across reboot (sanity check on tp-2)
ssh ubuntu@192.168.44.72 'sudo reboot'
# wait ~60s
kubectl wait --for=condition=Ready node/tp-2 --timeout=300s
ssh ubuntu@192.168.44.72 'hostname; hostname -f'
# Expect: tp-2 in both. If hostname -f returns tp-2.local, apply the runbook's mDNS fix.
```

### Inventory cleanup

After all four nodes verify, edit `inventory-turingpi.yml`:

```yaml
# Before:
server:
  hosts:
    192.168.44.71:
      os_upgrade_node_name: control-plane.local
agent:
  hosts:
    192.168.44.72:
      os_upgrade_node_name: worker2.local
    192.168.44.73:
      os_upgrade_node_name: tp-3
    192.168.44.74:
      os_upgrade_node_name: worker4.local

# After:
server:
  hosts:
    192.168.44.71:
agent:
  hosts:
    192.168.44.72:
    192.168.44.73:
    192.168.44.74:
```

The defaults in `roles/os_upgrade/defaults/main.yml` (`os_upgrade_node_name: "{{ ansible_hostname }}"`) now resolve correctly because `ansible_hostname` matches the k3s node name on every host. Verified by:

```bash
ansible-playbook playbooks/os-upgrade.yml -i inventory-turingpi.yml --check --diff
```

(Should be a no-op aside from the inventory variable change.)

### Commit shape

1. **Commit 1** (before rollout): runbook lives in the repo as an executable doc — `docs/superpowers/runbooks/rename-tp-nodes.md`.
2. **Commit 2** (after all nodes verified): `chore(turingpi): drop os_upgrade_node_name overrides` — the inventory cleanup.
3. **Commit 3** (final): `docs: remove completed node-rename TODO` — strip the entry from `TODO.md`.

## 8. Open questions / risks

- **Avahi `.local` reappearance** is the most likely surprise. If it happens, the runbook has a fix; if the fix doesn't stick, follow-up work is to disable avahi on these nodes (out of scope for this design but tracked as a risk).
- **Cert SAN warnings** on `kubectl version` are noise so long as nothing resolves the API by hostname. If a future workload (e.g., an in-cluster controller) tries to connect via a hostname-based URL and fails, that's a separate issue requiring a tls-san addition and k3s restart on `tp-1` — also out of scope here.
