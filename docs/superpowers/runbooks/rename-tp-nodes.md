# Runbook — Rename k3s nodes to `tp-1` … `tp-4`

**Spec:** `docs/superpowers/specs/2026-04-26-tp-node-rename-design.md`
**Operator:** runs from a laptop with `kubectl` configured for the cluster (`k3s-ansible` context) and SSH access to all four nodes as `ubuntu`.

## Goal

After this runbook completes, `kubectl get nodes` shows exactly:

```
NAME   STATUS   ROLES                  VERSION
tp-1   Ready    control-plane,master   v1.33.10+k3s1
tp-2   Ready    <none>                 v1.33.10+k3s1
tp-3   Ready    <none>                 v1.33.10+k3s1
tp-4   Ready    <none>                 v1.33.10+k3s1
```

On each host, `hostname` and `hostname -f` both return `tp-N` with no `.local` suffix.

## Per-node substitutions

| Phase | IP             | `<old>`               | `<new>` | Notes |
|-------|----------------|-----------------------|---------|-------|
| 1     | 192.168.44.73  | `tp-3`                | `tp-3`  | Dry-run; rename steps skipped (already correct). Real drain still runs. |
| 2     | 192.168.44.72  | `worker2.local`       | `tp-2`  | First real rename. |
| 3     | 192.168.44.74  | `worker4.local`       | `tp-4`  | Second real rename. |
| 4     | 192.168.44.71  | `control-plane.local` | `tp-1`  | Control plane. API outage ~1–3 min. Mandatory backup. |

**Order is mandatory.** Do not skip ahead. Verify each phase's gate before starting the next.

## Pre-rollout setup (do once)

```bash
# Confirm working context and current state
kubectl config current-context              # expect: k3s-ansible
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get pv,pvc -A
```

**Gate:** all four nodes Ready. Save the `pods -A -o wide` output somewhere (terminal scrollback is fine) — you'll compare to it after the rollout.

## Procedure: rename an agent node

Use for `tp-2`, `tp-4`. For `tp-3`, see the next section (it skips the destructive middle steps).

### A1. Preflight (from laptop)

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide --field-selector spec.nodeName=<old>
kubectl get pdb -A
```

**Gate:** all nodes Ready, no PodDisruptionBudgets that would block draining `<old>`.

### A2. Cordon and drain (from laptop)

```bash
kubectl cordon <old>
kubectl drain <old> --ignore-daemonsets --delete-emptydir-data --timeout=300s
```

**Gate:** drain returns 0; `kubectl get pods -A -o wide --field-selector spec.nodeName=<old>` shows only DaemonSet pods.

### A3. Stop k3s-agent (on the target node, via SSH)

```bash
ssh ubuntu@<ip> 'sudo systemctl stop k3s-agent'
ssh ubuntu@<ip> 'systemctl is-active k3s-agent || true'
```

Expected: `inactive`.

### A4. Rename the host (on the target)

For `tp-2` (`worker2.local` → `tp-2`):

```bash
ssh ubuntu@192.168.44.72 'sudo hostnamectl set-hostname tp-2'
ssh ubuntu@192.168.44.72 'sudo sed -i "s/\bworker2\.local\b/tp-2/g; s/\bworker2\b/tp-2/g" /etc/hosts'
ssh ubuntu@192.168.44.72 'grep -E "worker2" /etc/hostname /etc/hosts || echo "clean"'
```

For `tp-4` (`worker4.local` → `tp-4`):

```bash
ssh ubuntu@192.168.44.74 'sudo hostnamectl set-hostname tp-4'
ssh ubuntu@192.168.44.74 'sudo sed -i "s/\bworker4\.local\b/tp-4/g; s/\bworker4\b/tp-4/g" /etc/hosts'
ssh ubuntu@192.168.44.74 'grep -E "worker4" /etc/hostname /etc/hosts || echo "clean"'
```

**Gate:** the final `grep` prints exactly `clean` (no remaining references).

### A5. Delete the old node from the cluster (from laptop)

```bash
kubectl delete node <old>
kubectl get nodes
```

**Gate:** `<old>` no longer present; the three remaining names are unchanged.

### A6. Restart k3s-agent (on target)

```bash
ssh ubuntu@<ip> 'sudo systemctl start k3s-agent'
ssh ubuntu@<ip> 'sudo systemctl status k3s-agent --no-pager | head -20'
```

Expected: status `active (running)`. k3s-agent re-registers using the new hostname.

### A7. Verify (from laptop)

```bash
kubectl get nodes
kubectl wait --for=condition=Ready node/<new> --timeout=180s
kubectl uncordon <new>
kubectl get pods -A -o wide --field-selector spec.nodeName=<new>
```

**Gate (do not skip):** `<new>` is Ready, `<old>` is gone, workloads schedule onto `<new>`. **Halt and consult the rollback section if any of these fail.**

## Procedure: `tp-3` dry-run (Phase 1)

`tp-3` is already named correctly. Steps A4 (rename) and A5 (delete-node) are skipped, but the drain and verify steps are real — workloads on `tp-3` reschedule to `tp-2`/`tp-4` for a few minutes, then return after uncordon.

The point: exercise drain mechanics, PDB checks, and the verification gates before touching a node we have to bring back under a new identity.

### Steps

1. **A1 (preflight):** as written, with `<old>` = `tp-3`.
2. **A2 (cordon and drain):** as written, with `<old>` = `tp-3`.
3. **Skip A3, A4, A5, A6.**
4. **A7 (verify), modified:**

```bash
kubectl uncordon tp-3
kubectl get nodes
kubectl get pods -A -o wide --field-selector spec.nodeName=tp-3
```

**Gate:** `tp-3` Ready, uncordoned, scheduling new pods. If anything is wrong here, **stop the entire rollout** — your drain mechanics or cluster health is off, and you don't want to find that out mid-control-plane-rename.

## Procedure: rename the control plane (`tp-1`, Phase 4)

Run only after `tp-3`, `tp-2`, and `tp-4` are all verified Ready under their target names. The Kubernetes API is unavailable for ~1–3 minutes during this procedure — workloads on agents keep running (kubelet keeps pods alive without the API), but no scheduling, no `kubectl`.

### C1. Preflight (from laptop)

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get pv,pvc -A
```

**Gate:** `tp-2`, `tp-3`, `tp-4` are all Ready. Only the control plane (`192.168.44.71`) still carries an old name (`control-plane.local`). Save the pods/PVC output for post-rollout comparison.

### C2. Backup, rename, restart (on `192.168.44.71`)

The backup is mandatory — it's the rollback path of last resort.

```bash
ssh ubuntu@192.168.44.71 'sudo systemctl stop k3s'
ssh ubuntu@192.168.44.71 'sudo tar -czf /root/k3s-server-backup-$(date +%Y%m%d-%H%M%S).tar.gz -C /var/lib/rancher k3s/server'
ssh ubuntu@192.168.44.71 'sudo ls -la /root/k3s-server-backup-*.tar.gz'
```

**Gate:** the backup file exists and is non-zero (typically tens to hundreds of MB depending on cluster age).

```bash
ssh ubuntu@192.168.44.71 'sudo hostnamectl set-hostname tp-1'
ssh ubuntu@192.168.44.71 'sudo sed -i "s/\bcontrol-plane\.local\b/tp-1/g; s/\bcontrol-plane\b/tp-1/g" /etc/hosts'
ssh ubuntu@192.168.44.71 'grep -E "control-plane" /etc/hostname /etc/hosts || echo "clean"'
```

**Gate:** the final `grep` prints exactly `clean`.

```bash
ssh ubuntu@192.168.44.71 'sudo systemctl start k3s'
ssh ubuntu@192.168.44.71 'sudo systemctl status k3s --no-pager | head -20'
```

Expected: status `active (running)`.

**Why we don't `kubectl delete node control-plane.local` before stopping k3s:** with k3s running, its kubelet would re-register the node under `control-plane.local` seconds later — the delete is racy and effectively a no-op. We rename first, let k3s register the new entry, then clean up the stale one in step C4.

### C3. Wait for the API to come back (from laptop)

```bash
# Retry until the API responds; takes ~30–90s typically.
for i in $(seq 1 30); do
  kubectl get nodes >/dev/null 2>&1 && break
  echo "wait $i..."
  sleep 5
done
kubectl get nodes
kubectl wait --for=condition=Ready node/tp-1 --timeout=300s
```

**Gate:** `tp-1` is Ready. There may also be a `control-plane.local` entry in NotReady state — that is expected and cleaned up in C4.

### C4. Delete the stale old entry (from laptop)

```bash
kubectl delete node control-plane.local
kubectl get nodes
```

**Gate:** four nodes total, all named `tp-N`, all Ready.

### C5. Final verification (from laptop)

```bash
kubectl get pods -A -o wide | grep -vE '^(NAMESPACE|.*Running|.*Completed)'
kubectl get apiservice
kubectl version
```

**Gate:** the first command returns nothing (no stuck pods); all APIServices show `True`; `kubectl version` returns server version with no cert errors.

### C6. mDNS / `.local` watch

After NetworkManager reload or reboot, `hostname -f` may re-acquire `.local`. Make the change sticky:

```bash
ssh ubuntu@192.168.44.71 'sudo hostnamectl set-hostname tp-1 --static'
ssh ubuntu@192.168.44.71 'hostname; hostname -f'
ssh ubuntu@192.168.44.71 'grep -E "MulticastDNS|LLMNR" /etc/systemd/resolved.conf || echo "defaults"'
```

Expected: `hostname` and `hostname -f` both return `tp-1`. If `hostname -f` returns `tp-1.local`, set `MulticastDNS=no` in `/etc/systemd/resolved.conf` and `systemctl restart systemd-resolved`. Apply the same fix to any other node where `hostname -f` shows `.local`.

## Rollback

If a verification gate fails, do NOT push forward. Pick the matching rollback case and **halt the rollout** afterward to investigate.

### R1. Agent rollback (`tp-2` or `tp-4` won't come back Ready)

On the affected agent:

```bash
ssh ubuntu@<ip> 'sudo systemctl stop k3s-agent'
ssh ubuntu@<ip> 'sudo hostnamectl set-hostname <old>'
ssh ubuntu@<ip> 'sudo sed -i "s/\b<new>\b/<old>/g" /etc/hosts'
ssh ubuntu@<ip> 'sudo systemctl start k3s-agent'
```

From laptop:

```bash
kubectl delete node <new>
kubectl wait --for=condition=Ready node/<old> --timeout=180s
kubectl uncordon <old>
```

Per-node substitutions for rollback:

| IP             | `<new>` (failed) | `<old>` (target after rollback) |
|----------------|------------------|---------------------------------|
| 192.168.44.72  | `tp-2`           | `worker2.local`                 |
| 192.168.44.74  | `tp-4`           | `worker4.local`                 |

**Halt the rollout** and investigate before continuing.

### R2. Control plane rollback — Case 1 (k3s started but is in a bad state)

NotReady, won't accept connections, cert errors. Try the lightweight rollback first:

```bash
ssh ubuntu@192.168.44.71 'sudo systemctl stop k3s'
ssh ubuntu@192.168.44.71 'sudo hostnamectl set-hostname control-plane.local'
ssh ubuntu@192.168.44.71 'sudo sed -i "s/\btp-1\b/control-plane.local/g" /etc/hosts'
ssh ubuntu@192.168.44.71 'sudo systemctl start k3s'
# From laptop:
kubectl wait --for=condition=Ready node/control-plane.local --timeout=300s
kubectl delete node tp-1   # drop the dead entry
```

### R3. Control plane rollback — Case 2 (k3s won't start, or in-cluster state corrupted)

Restore from the backup taken in C2:

```bash
ssh ubuntu@192.168.44.71 'sudo systemctl stop k3s'
ssh ubuntu@192.168.44.71 'sudo rm -rf /var/lib/rancher/k3s/server'
ssh ubuntu@192.168.44.71 'sudo tar -xzf /root/k3s-server-backup-<timestamp>.tar.gz -C /var/lib/rancher'
ssh ubuntu@192.168.44.71 'sudo hostnamectl set-hostname control-plane.local'
ssh ubuntu@192.168.44.71 'sudo sed -i "s/\btp-1\b/control-plane.local/g" /etc/hosts'
ssh ubuntu@192.168.44.71 'sudo systemctl start k3s'
# From laptop:
kubectl wait --for=condition=Ready node/control-plane.local --timeout=300s
```

Substitute `<timestamp>` with the actual filename from `ls /root/k3s-server-backup-*.tar.gz`.

### R4. Don't-do list

- **Never** run `ansible-playbook playbooks/reset.yml -i inventory-turingpi.yml --limit 192.168.44.71` as a rollback. `reset.yml` runs `k3s-uninstall.sh` which wipes the cluster. There is no rejoin (single-server topology).
- **Never** run `ansible-playbook playbooks/site.yml -i inventory-turingpi.yml` partway through a rollout. It would reconfigure k3s based on `ansible_hostname`, which may now mismatch the node name k3s already registered.
- Inventory rollback (if you'd already committed inventory changes): `git checkout inventory-turingpi.yml`.

If both R2 and R3 fail, the cluster's control plane is dead. Agent kubelets keep workload pods running locally without the API. Recovery is a fresh `site.yml` run plus redeploying MetalLB and workloads, with NFS-backed state rejoinable. This is the worst case — and exactly why C2's backup is mandatory.

## Post-rollout verification (after all four phases pass)

Run from your laptop:

```bash
# 1. Names and readiness
kubectl get nodes
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' | sort
# Expect: tp-1, tp-2, tp-3, tp-4. Nothing else.

# 2. System health
kubectl get pods -n kube-system
kubectl get pods -n metallb-system
kubectl get pods -A | grep -vE '^(NAMESPACE|.*Running|.*Completed)'
# Expect: empty.

# 3. Storage
kubectl get pvc -A
# Expect: all Bound.

# 4. LoadBalancer IPs still advertised
kubectl get svc -A --field-selector spec.type=LoadBalancer
# Expect: each has an EXTERNAL-IP from 192.168.44.200-220.

# 5. Hostname stickiness across reboot (sanity check on tp-2)
ssh ubuntu@192.168.44.72 'sudo reboot'
sleep 60
kubectl wait --for=condition=Ready node/tp-2 --timeout=300s
ssh ubuntu@192.168.44.72 'hostname; hostname -f'
# Expect: tp-2 in both. If hostname -f returns tp-2.local, apply the C6 mDNS fix to tp-2.
```

## Inventory cleanup (separate commit, after rollout)

Edit `inventory-turingpi.yml` to remove the per-host `os_upgrade_node_name` overrides. The defaults in `roles/os_upgrade/defaults/main.yml` (`os_upgrade_node_name: "{{ ansible_hostname }}"`) now resolve correctly because `ansible_hostname` matches the k3s node name on every host.

Before:

```yaml
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
```

After:

```yaml
server:
  hosts:
    192.168.44.71:
agent:
  hosts:
    192.168.44.72:
    192.168.44.73:
    192.168.44.74:
```

Verify by running `os-upgrade.yml` in check mode:

```bash
ansible-playbook playbooks/os-upgrade.yml -i inventory-turingpi.yml --check --diff
```

Expected: no errors; the per-host node-name resolves to `tp-N` via `ansible_hostname` on each host.

## TODO removal

After the inventory cleanup commit lands, remove the "Rename cluster nodes for consistency" section from `TODO.md` in a separate commit. The clean state is `TODO.md` with no remaining unfinished cluster-rename work.
