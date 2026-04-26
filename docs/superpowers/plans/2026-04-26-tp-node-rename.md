# Turing Pi Node Rename Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rename the four k3s nodes from a mix of `control-plane.local` / `worker2.local` / `tp-3` / `worker4.local` to `tp-1` / `tp-2` / `tp-3` / `tp-4`, matching Turing Pi slot numbers.

**Architecture:** A single markdown runbook at `docs/superpowers/runbooks/rename-tp-nodes.md` that the operator (the user) executes manually, one node at a time, with explicit verification gates between steps. Order: `tp-3` (no-op verification) → `tp-2` → `tp-4` → `tp-1` (control plane last). After all four nodes are renamed and verified, two cleanup commits: drop `os_upgrade_node_name` overrides from `inventory-turingpi.yml`; remove the completed entry from `TODO.md`. No new playbook or role.

**Tech Stack:** k3s (single-server topology, sqlite backend), Ubuntu, `kubectl`, `hostnamectl`, `systemd`. Runbook is plain Markdown.

**Spec:** `docs/superpowers/specs/2026-04-26-tp-node-rename-design.md`

---

## Conventions used in this plan

- **Phase A** (Tasks 1–7) authors the runbook. Each task adds one section and shows the exact Markdown content. The runbook itself is committed once at the end of Phase A.
- **Phase B** (Tasks 8–11) executes the rename, one node per task, each with a hard verification gate. These are operational tasks — the "edit" is on the live cluster, not on files in the repo. Do NOT proceed past a failed gate; instead, fall back to the rollback section of the runbook.
- **Phase C** (Tasks 12–13) is small inventory / TODO cleanup, one commit per task.
- All cluster-side verification commands assume `KUBECONFIG=~/.kube/config` and a working context for the cluster (current default per the project: `k3s-ansible`).
- All shell snippets in the runbook use `<old>` and `<new>` as substitution placeholders. The runbook's per-node tables show what to substitute for each node so there's no copy-paste guessing at execution time.
- Conventional commit prefixes match repo style (`docs:`, `chore:`).

---

### Task 1: Scaffold the runbook with header, goal, and per-node table

**Files:**
- Create: `docs/superpowers/runbooks/rename-tp-nodes.md`

- [ ] **Step 1: Create the runbook with the opening sections**

Create `docs/superpowers/runbooks/rename-tp-nodes.md` with the following content:

````markdown
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
````

- [ ] **Step 2: Verify the file exists and renders**

Run:
```bash
ls -la docs/superpowers/runbooks/rename-tp-nodes.md
head -40 docs/superpowers/runbooks/rename-tp-nodes.md
```

Expected: file exists; first 40 lines show the header through the per-node table.

**Do not commit yet** — Tasks 2–6 append more sections; Task 7 is the single commit.

---

### Task 2: Add the agent rename procedure section

**Files:**
- Modify: `docs/superpowers/runbooks/rename-tp-nodes.md` (append)

- [ ] **Step 1: Append the agent procedure to the runbook**

Append the following content to `docs/superpowers/runbooks/rename-tp-nodes.md`:

````markdown
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
````

- [ ] **Step 2: Confirm the section was appended**

Run:
```bash
grep -c '^## Procedure: rename an agent node' docs/superpowers/runbooks/rename-tp-nodes.md
```

Expected: `1`.

---

### Task 3: Add the `tp-3` dry-run section

**Files:**
- Modify: `docs/superpowers/runbooks/rename-tp-nodes.md` (append)

- [ ] **Step 1: Append the `tp-3` dry-run section**

Append:

````markdown
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
````

---

### Task 4: Add the control-plane (`tp-1`) procedure section

**Files:**
- Modify: `docs/superpowers/runbooks/rename-tp-nodes.md` (append)

- [ ] **Step 1: Append the control-plane procedure**

Append:

````markdown
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
````

---

### Task 5: Add the rollback section

**Files:**
- Modify: `docs/superpowers/runbooks/rename-tp-nodes.md` (append)

- [ ] **Step 1: Append the rollback section**

Append:

````markdown
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
````

---

### Task 6: Add the post-rollout verification and inventory cleanup section

**Files:**
- Modify: `docs/superpowers/runbooks/rename-tp-nodes.md` (append)

- [ ] **Step 1: Append the post-rollout verification + cleanup section**

Append:

````markdown
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
````

- [ ] **Step 2: Verify the runbook is structurally complete**

Run:
```bash
grep -E '^## ' docs/superpowers/runbooks/rename-tp-nodes.md
```

Expected output (in order):
```
## Goal
## Per-node substitutions
## Pre-rollout setup (do once)
## Procedure: rename an agent node
## Procedure: `tp-3` dry-run (Phase 1)
## Procedure: rename the control plane (`tp-1`, Phase 4)
## Rollback
## Post-rollout verification (after all four phases pass)
## Inventory cleanup (separate commit, after rollout)
## TODO removal
```

If a section is missing, re-run the corresponding task.

---

### Task 7: Commit the runbook

**Files:**
- Stage: `docs/superpowers/runbooks/rename-tp-nodes.md`

- [ ] **Step 1: Confirm only the runbook is staged**

```bash
git status
git diff --stat docs/superpowers/runbooks/rename-tp-nodes.md
```

Expected: a single new file `docs/superpowers/runbooks/rename-tp-nodes.md`. If anything else is staged or modified, sort that out before committing.

- [ ] **Step 2: Commit**

```bash
git add docs/superpowers/runbooks/rename-tp-nodes.md
git commit -m "$(cat <<'EOF'
docs(turingpi): add runbook for renaming nodes to tp-1..tp-4

Documents the manual procedure for renaming the four cluster nodes to
match Turing Pi slot numbers. Order is tp-3 (dry-run) -> tp-2 -> tp-4 ->
tp-1, with explicit verification gates between phases and a mandatory
sqlite backup before the control-plane rename. Spec: docs/superpowers/
specs/2026-04-26-tp-node-rename-design.md.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
git log -1 --stat
```

Expected: a single new commit adding the runbook, no other files touched.

---

## Phase B — Execute the rollout

Phase B tasks edit live cluster state, not files in the repo. Each task is one node. Each task **must** end at a passing verification gate; if a gate fails, switch to the relevant rollback section and halt.

The runbook from Phase A is the authoritative source of commands. The plan tasks below are scaffolding — they tell the operator which runbook section to follow and what "done" looks like at task granularity.

---

### Task 8: Execute Phase 1 — `tp-3` dry-run

**Operating on:** `192.168.44.73` (already named `tp-3`).
**Runbook section:** "Procedure: `tp-3` dry-run (Phase 1)".

- [ ] **Step 1: Run preflight (runbook section A1, with `<old>=tp-3`)**

Capture the output. Confirm all four nodes Ready and no PDBs blocking.

- [ ] **Step 2: Cordon and drain `tp-3` (runbook section A2)**

```bash
kubectl cordon tp-3
kubectl drain tp-3 --ignore-daemonsets --delete-emptydir-data --timeout=300s
```

Expected: drain returns 0; non-DaemonSet pods reschedule to `tp-2`/`tp-4` (or the control plane node).

- [ ] **Step 3: Skip A3–A6 (rename steps not applicable)**

- [ ] **Step 4: Uncordon and verify**

```bash
kubectl uncordon tp-3
kubectl get nodes
kubectl get pods -A -o wide --field-selector spec.nodeName=tp-3
```

**Gate:** `tp-3` Ready and uncordoned; pods schedule onto `tp-3` again. If not, halt — your cluster has a pre-existing health problem to investigate before any rename.

- [ ] **Step 5: Confirm cluster snapshot for next phase**

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide --field-selector spec.nodeName!=tp-3 | head -40
```

Expected: cluster steady-state, all four nodes Ready.

---

### Task 9: Execute Phase 2 — rename `192.168.44.72` to `tp-2`

**Operating on:** `192.168.44.72` (`worker2.local` → `tp-2`).
**Runbook section:** "Procedure: rename an agent node".

- [ ] **Step 1: Preflight (runbook A1, with `<old>=worker2.local`)**

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide --field-selector spec.nodeName=worker2.local
kubectl get pdb -A
```

**Gate:** all Ready, no blocking PDBs.

- [ ] **Step 2: Cordon and drain (runbook A2)**

```bash
kubectl cordon worker2.local
kubectl drain worker2.local --ignore-daemonsets --delete-emptydir-data --timeout=300s
```

**Gate:** drain returns 0; only DaemonSet pods remain on `worker2.local`.

- [ ] **Step 3: Stop k3s-agent (runbook A3)**

```bash
ssh ubuntu@192.168.44.72 'sudo systemctl stop k3s-agent'
ssh ubuntu@192.168.44.72 'systemctl is-active k3s-agent || true'
```

Expected: `inactive`.

- [ ] **Step 4: Rename the host (runbook A4)**

```bash
ssh ubuntu@192.168.44.72 'sudo hostnamectl set-hostname tp-2'
ssh ubuntu@192.168.44.72 'sudo sed -i "s/\bworker2\.local\b/tp-2/g; s/\bworker2\b/tp-2/g" /etc/hosts'
ssh ubuntu@192.168.44.72 'grep -E "worker2" /etc/hostname /etc/hosts || echo "clean"'
```

**Gate:** the final `grep` prints `clean`.

- [ ] **Step 5: Delete the old node entry (runbook A5)**

```bash
kubectl delete node worker2.local
kubectl get nodes
```

- [ ] **Step 6: Restart k3s-agent (runbook A6)**

```bash
ssh ubuntu@192.168.44.72 'sudo systemctl start k3s-agent'
ssh ubuntu@192.168.44.72 'sudo systemctl status k3s-agent --no-pager | head -20'
```

Expected: `active (running)`.

- [ ] **Step 7: Verify (runbook A7)**

```bash
kubectl get nodes
kubectl wait --for=condition=Ready node/tp-2 --timeout=180s
kubectl uncordon tp-2
kubectl get pods -A -o wide --field-selector spec.nodeName=tp-2
```

**Gate (mandatory):** `tp-2` Ready, `worker2.local` gone, workloads schedule onto `tp-2`. If any check fails, switch to runbook section R1 (Agent rollback) for `tp-2` and halt.

---

### Task 10: Execute Phase 3 — rename `192.168.44.74` to `tp-4`

**Operating on:** `192.168.44.74` (`worker4.local` → `tp-4`).
**Runbook section:** "Procedure: rename an agent node".

- [ ] **Step 1: Preflight**

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide --field-selector spec.nodeName=worker4.local
kubectl get pdb -A
```

**Gate:** all Ready, no blocking PDBs. Confirm `tp-2` (renamed in Task 9) is still Ready.

- [ ] **Step 2: Cordon and drain**

```bash
kubectl cordon worker4.local
kubectl drain worker4.local --ignore-daemonsets --delete-emptydir-data --timeout=300s
```

- [ ] **Step 3: Stop k3s-agent**

```bash
ssh ubuntu@192.168.44.74 'sudo systemctl stop k3s-agent'
ssh ubuntu@192.168.44.74 'systemctl is-active k3s-agent || true'
```

Expected: `inactive`.

- [ ] **Step 4: Rename the host**

```bash
ssh ubuntu@192.168.44.74 'sudo hostnamectl set-hostname tp-4'
ssh ubuntu@192.168.44.74 'sudo sed -i "s/\bworker4\.local\b/tp-4/g; s/\bworker4\b/tp-4/g" /etc/hosts'
ssh ubuntu@192.168.44.74 'grep -E "worker4" /etc/hostname /etc/hosts || echo "clean"'
```

**Gate:** the final `grep` prints `clean`.

- [ ] **Step 5: Delete the old node entry**

```bash
kubectl delete node worker4.local
kubectl get nodes
```

- [ ] **Step 6: Restart k3s-agent**

```bash
ssh ubuntu@192.168.44.74 'sudo systemctl start k3s-agent'
ssh ubuntu@192.168.44.74 'sudo systemctl status k3s-agent --no-pager | head -20'
```

- [ ] **Step 7: Verify**

```bash
kubectl get nodes
kubectl wait --for=condition=Ready node/tp-4 --timeout=180s
kubectl uncordon tp-4
kubectl get pods -A -o wide --field-selector spec.nodeName=tp-4
```

**Gate (mandatory):** `tp-4` Ready, `worker4.local` gone. If any check fails, switch to runbook section R1 for `tp-4` and halt.

After this task, three of four nodes are at their target names: `tp-2`, `tp-3`, `tp-4`. Only the control plane remains.

---

### Task 11: Execute Phase 4 — rename the control plane to `tp-1`

**Operating on:** `192.168.44.71` (`control-plane.local` → `tp-1`).
**Runbook section:** "Procedure: rename the control plane (`tp-1`, Phase 4)".

This is the high-stakes step. The Kubernetes API will be down for ~1–3 minutes. Read the entire runbook section once before executing.

- [ ] **Step 1: Preflight (runbook C1)**

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get pv,pvc -A
```

**Gate:** `tp-2`, `tp-3`, `tp-4` all Ready; only the control plane (`192.168.44.71`) carries `control-plane.local`.

- [ ] **Step 2: Backup (runbook C2, first block)**

```bash
ssh ubuntu@192.168.44.71 'sudo systemctl stop k3s'
ssh ubuntu@192.168.44.71 'sudo tar -czf /root/k3s-server-backup-$(date +%Y%m%d-%H%M%S).tar.gz -C /var/lib/rancher k3s/server'
ssh ubuntu@192.168.44.71 'sudo ls -la /root/k3s-server-backup-*.tar.gz'
```

**Gate:** backup file exists, non-zero size. **Record the exact filename** — you'll need it if R3 (Case 2 rollback) becomes necessary.

- [ ] **Step 3: Rename and restart (runbook C2, second + third blocks)**

```bash
ssh ubuntu@192.168.44.71 'sudo hostnamectl set-hostname tp-1'
ssh ubuntu@192.168.44.71 'sudo sed -i "s/\bcontrol-plane\.local\b/tp-1/g; s/\bcontrol-plane\b/tp-1/g" /etc/hosts'
ssh ubuntu@192.168.44.71 'grep -E "control-plane" /etc/hostname /etc/hosts || echo "clean"'
```

**Gate:** the final `grep` prints `clean`.

```bash
ssh ubuntu@192.168.44.71 'sudo systemctl start k3s'
ssh ubuntu@192.168.44.71 'sudo systemctl status k3s --no-pager | head -20'
```

Expected: status `active (running)`. If it fails to start at all, jump to runbook R3 (Case 2) using the backup from step 2.

- [ ] **Step 4: Wait for the API (runbook C3)**

```bash
for i in $(seq 1 30); do
  kubectl get nodes >/dev/null 2>&1 && break
  echo "wait $i..."
  sleep 5
done
kubectl get nodes
kubectl wait --for=condition=Ready node/tp-1 --timeout=300s
```

**Gate:** `tp-1` is Ready. If after 5 minutes it's still NotReady, switch to runbook R2 (Case 1).

- [ ] **Step 5: Delete the stale entry (runbook C4)**

```bash
kubectl delete node control-plane.local
kubectl get nodes
```

**Gate:** four nodes total, all named `tp-N`, all Ready.

- [ ] **Step 6: Final verification (runbook C5)**

```bash
kubectl get pods -A -o wide | grep -vE '^(NAMESPACE|.*Running|.*Completed)'
kubectl get apiservice
kubectl version
```

**Gate:** no stuck pods; all APIServices `True`; no cert errors.

- [ ] **Step 7: mDNS / `.local` watch (runbook C6) — and apply to all four nodes**

```bash
for ip in 192.168.44.71 192.168.44.72 192.168.44.73 192.168.44.74; do
  ssh ubuntu@$ip 'sudo hostnamectl set-hostname $(hostname) --static'
  ssh ubuntu@$ip 'echo "=== '$ip' ==="; hostname; hostname -f'
done
```

Expected: each host prints its `tp-N` name twice (no `.local`). If `hostname -f` shows `.local` for any node, apply the `MulticastDNS=no` fix from runbook C6 to that node.

- [ ] **Step 8: Hostname stickiness reboot test on `tp-2`**

```bash
ssh ubuntu@192.168.44.72 'sudo reboot'
sleep 60
kubectl wait --for=condition=Ready node/tp-2 --timeout=300s
ssh ubuntu@192.168.44.72 'hostname; hostname -f'
```

Expected: `tp-2` in both lines. If `hostname -f` shows `.local` after reboot, the static hostname didn't stick — apply the runbook C6 fix and re-run this step.

After this task, the cluster is in its target state. Move to Phase C.

---

## Phase C — Cleanup commits

---

### Task 12: Drop `os_upgrade_node_name` overrides from inventory

**Files:**
- Modify: `inventory-turingpi.yml`

- [ ] **Step 1: Read current inventory**

```bash
cat inventory-turingpi.yml
```

Confirm the current `hosts:` block matches the pre-cleanup state shown in the runbook's "Inventory cleanup" section.

- [ ] **Step 2: Edit `inventory-turingpi.yml`**

Replace the `hosts:` blocks under `server:` and `agent:` so they look like:

```yaml
k3s_cluster:
  children:
    server:
      hosts:
        192.168.44.71:
    agent:
      hosts:
        192.168.44.72:
        192.168.44.73:
        192.168.44.74:
```

(All four `os_upgrade_node_name: ...` lines are removed. All other content in the file — `vars:`, comments, `extra_server_args`, etc. — is preserved unchanged.)

- [ ] **Step 3: Lint**

```bash
yamllint inventory-turingpi.yml
ansible-lint inventory-turingpi.yml || true
```

Expected: yamllint passes. (`ansible-lint` may not have a rule that applies to inventory; `|| true` keeps the step from failing if lint output is informational.)

- [ ] **Step 4: Verify the role's default now resolves correctly**

```bash
ansible-playbook playbooks/os-upgrade.yml -i inventory-turingpi.yml --check --diff
```

Expected: no errors; `os_upgrade_node_name` resolves to `tp-N` via the default `{{ ansible_hostname }}` on each host. The check-mode run should not actually change anything (it's `--check`).

If `os-upgrade.yml` requires running tasks before reaching the cordon/drain step (e.g., it may try to apt-update in check mode), and you don't want a full check-mode run, this verification is optional — the change is small and the default expression is correct by inspection. Document the skip in the commit message if so.

- [ ] **Step 5: Commit**

```bash
git diff inventory-turingpi.yml
git add inventory-turingpi.yml
git commit -m "$(cat <<'EOF'
chore(turingpi): drop os_upgrade_node_name overrides

After renaming the four nodes to tp-1..tp-4, ansible_hostname matches
the k3s node name on every host, so the per-host overrides in
inventory-turingpi.yml are redundant. The role default in
roles/os_upgrade/defaults/main.yml resolves correctly without them.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
git log -1 --stat
```

Expected: single-file commit, only `inventory-turingpi.yml` modified.

---

### Task 13: Remove the completed entry from `TODO.md`

**Files:**
- Modify: `TODO.md`

- [ ] **Step 1: Read current TODO.md**

```bash
cat TODO.md
```

Confirm the "Rename cluster nodes for consistency" section is present and is the only top-level task.

- [ ] **Step 2: Replace `TODO.md` content**

If the rename section is the only content (other than the `# TODO` heading), the cleanest result is a `TODO.md` with no remaining items:

```markdown
# TODO

(none)
```

If other unrelated TODO items exist, keep those and remove only the "## Rename cluster nodes for consistency" section and its body, including the entire table, "Why" paragraph, "Why this is its own project" list, and "Suggested next step" paragraph.

- [ ] **Step 3: Verify no orphaned references**

```bash
grep -rn "control-plane.local\|worker2.local\|worker4.local" . \
  --include="*.md" --include="*.yml" --include="*.yaml" \
  --exclude-dir=.git --exclude-dir=.ansible || echo "no references"
```

Expected: `no references`. (If anything turns up, it's likely in `inventory-turingpi.yml` and means Task 12 wasn't completed — fix before continuing.)

- [ ] **Step 4: Commit**

```bash
git diff TODO.md
git add TODO.md
git commit -m "$(cat <<'EOF'
docs: remove completed node-rename TODO

The cluster nodes are now named tp-1..tp-4 and the inventory has been
cleaned up; the rename project is done.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
git log -3 --oneline
```

Expected: three-commit history at HEAD: (1) runbook, (2) inventory cleanup, (3) TODO removal.

---

## Self-review notes (kept for traceability)

- **Spec coverage:** every section of the spec maps to at least one task. Sections 1 (goal/scope) and 2 (context/key facts) → encoded in this plan's header and Task 1's runbook scaffold. Section 3 (decisions) → embedded throughout. Section 4 (agent procedure) → Tasks 2, 8 (verify), 9, 10. Section 5 (control-plane procedure) → Tasks 4, 11. Section 6 (rollback) → Task 5 (runbook content). Section 7 (success criteria + inventory cleanup) → Tasks 6, 11 step 8, 12, 13.
- **Placeholder check:** `<old>`, `<new>`, `<ip>`, `<timestamp>` are intentional substitution markers, always paired with explicit per-node values shown in the runbook's per-node table or in the relevant task. No "TBD"/"TODO"/"figure out later" anywhere in the plan.
- **Type/name consistency:** runbook section IDs (A1–A7, C1–C6, R1–R4) are used consistently across the plan — Task 9 references runbook A2; Task 11 references runbook C2. Per-node IPs match throughout (`.71` = control plane, `.72`/`.74` = agents being renamed, `.73` = `tp-3` no-op).
- **Single-server topology** is reinforced in three places (header, control-plane procedure, R4 don't-do list) so an out-of-context reader doesn't accidentally treat this like an HA setup.
