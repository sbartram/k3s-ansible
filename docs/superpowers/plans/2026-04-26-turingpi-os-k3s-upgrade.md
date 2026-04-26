# Turing Pi OS + k3s Upgrade Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a new `os_upgrade` role plus `os-upgrade.yml` and `upgrade-all.yml` playbooks so the Turing Pi cluster can be patched (Ubuntu apt) and version-upgraded (k3s) in one or two-phase flows, with cordon/drain on agents.

**Architecture:** A single `os_upgrade` role does `apt update`/`apt upgrade`, conditionally reboots when `/var/run/reboot-required` is present, and conditionally cordons/drains/uncordons via `kubectl` delegated to the control-plane node. `os-upgrade.yml` runs server-then-agents (server without drain). `upgrade-all.yml` chains `os-upgrade.yml` then the existing `upgrade.yml` via `import_playbook`.

**Tech Stack:** Ansible (ansible-core 2.15+), `ansible.builtin.apt`, `ansible.builtin.reboot`, `kubectl`, k3s, Ubuntu. Linting via `yamllint` and `ansible-lint`.

**Spec:** `docs/superpowers/specs/2026-04-26-turingpi-os-k3s-upgrade-design.md`

---

## Conventions used in this plan

- All variables in the new role are prefixed `os_upgrade_` so no `# noqa var-naming[no-role-prefix]` markers are needed (unlike `roles/k3s_upgrade/defaults/main.yml`, which uses unprefixed vars and needs those markers).
- Code style follows existing roles: YAML truthy `true`/`false` (not `yes`/`no`), 180-char max line, 2-space indent.
- Each task ends with `yamllint .`, `ansible-lint`, and (where applicable) `ansible-playbook --syntax-check`. These are the project's correctness gates.
- Each task ends with a commit. Conventional commit prefixes (`feat:`, `docs:`, `chore:`) match the repo's existing style (see `git log --oneline`).

---

### Task 1: Scaffold the `os_upgrade` role

**Files:**
- Create: `roles/os_upgrade/defaults/main.yml`
- Create: `roles/os_upgrade/meta/main.yml`
- Create: `roles/os_upgrade/tasks/main.yml` (placeholder body so the role is loadable)
- Create: `roles/os_upgrade/tasks/debian.yml` (placeholder)

- [ ] **Step 1: Create `roles/os_upgrade/defaults/main.yml`**

```yaml
---
# Whether to cordon and drain this node before reboot.
# Set true for agent plays; leave false for the control plane (single-server cluster).
os_upgrade_drain: false

# Maps to ansible.builtin.apt: upgrade=. "safe" == apt upgrade (no removals).
# Use "full" or "dist" only after confirming pending package removals.
os_upgrade_apt_upgrade_type: safe

# Run apt autoremove after the upgrade.
os_upgrade_autoremove: true

# Seconds to wait for SSH after reboot.
os_upgrade_reboot_timeout: 600

# Seconds for kubectl drain to complete before failing.
os_upgrade_drain_timeout: 300

# Seconds to wait for the node to report Ready after reboot.
os_upgrade_node_ready_timeout: 300

# Reboot even when /var/run/reboot-required is absent. Escape hatch.
os_upgrade_force_reboot: false
```

- [ ] **Step 2: Create `roles/os_upgrade/meta/main.yml`**

```yaml
---
galaxy_info:
  author: k3s-ansible contributors
  description: Apply OS package updates and reboot nodes safely, optionally draining via kubectl.
  license: Apache-2.0
  min_ansible_version: "2.15"
  platforms:
    - name: Ubuntu
      versions:
        - all
    - name: Debian
      versions:
        - all
dependencies: []
```

- [ ] **Step 3: Create placeholder `roles/os_upgrade/tasks/main.yml`**

```yaml
---
- name: Assert supported OS family
  ansible.builtin.assert:
    that:
      - ansible_os_family == "Debian"
    fail_msg: "os_upgrade currently only supports Debian/Ubuntu (ansible_os_family=Debian); got {{ ansible_os_family }}"

- name: Run Debian-family upgrade tasks
  ansible.builtin.include_tasks: debian.yml
```

- [ ] **Step 4: Create placeholder `roles/os_upgrade/tasks/debian.yml`**

```yaml
---
# Filled in by Task 2.
- name: Placeholder
  ansible.builtin.debug:
    msg: "debian.yml not yet implemented"
```

- [ ] **Step 5: Lint**

Run: `yamllint roles/os_upgrade/`
Expected: clean exit (return code 0).

Run: `ansible-lint roles/os_upgrade/`
Expected: clean exit, or only the same warning categories the repo already accepts (`var-naming[no-role-prefix]`, `yaml[comments-indentation]`, `yaml[line-length]`). No errors.

- [ ] **Step 6: Commit**

```bash
git add roles/os_upgrade/
git commit -m "feat: scaffold os_upgrade role with defaults and OS assert"
```

---

### Task 2: Implement apt update / upgrade / autoremove (`debian.yml`)

**Files:**
- Modify: `roles/os_upgrade/tasks/debian.yml` (replace the placeholder)

- [ ] **Step 1: Replace `roles/os_upgrade/tasks/debian.yml` with the real implementation**

```yaml
---
- name: Update apt cache
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600

- name: Upgrade installed packages
  ansible.builtin.apt:
    upgrade: "{{ os_upgrade_apt_upgrade_type }}"

- name: Autoremove unused packages
  when: os_upgrade_autoremove
  ansible.builtin.apt:
    autoremove: true
```

- [ ] **Step 2: Lint**

Run: `yamllint roles/os_upgrade/tasks/debian.yml`
Expected: clean exit.

Run: `ansible-lint roles/os_upgrade/`
Expected: clean exit (same accepted warnings as before).

- [ ] **Step 3: Commit**

```bash
git add roles/os_upgrade/tasks/debian.yml
git commit -m "feat(os_upgrade): implement apt update/upgrade/autoremove for Debian family"
```

---

### Task 3: Implement reboot detection, drain, reboot, and wait-Ready (`main.yml`)

**Files:**
- Modify: `roles/os_upgrade/tasks/main.yml` (extend beyond the assert + include)

This is the central task. We replace the placeholder body of `main.yml` with the full flow:

1. Assert OS family.
2. Include debian.yml (apt operations).
3. Stat `/var/run/reboot-required`.
4. Skip remaining tasks when no reboot needed and not forced.
5. (If `os_upgrade_drain`) cordon + drain via kubectl, delegated to the control plane.
6. Reboot the host.
7. (If `os_upgrade_drain`) wait for Ready, then uncordon.

- [ ] **Step 1: Replace `roles/os_upgrade/tasks/main.yml`**

```yaml
---
- name: Assert supported OS family
  ansible.builtin.assert:
    that:
      - ansible_os_family == "Debian"
    fail_msg: "os_upgrade currently only supports Debian/Ubuntu (ansible_os_family=Debian); got {{ ansible_os_family }}"

- name: Run Debian-family upgrade tasks
  ansible.builtin.include_tasks: debian.yml

- name: Check whether a reboot is required
  ansible.builtin.stat:
    path: /var/run/reboot-required
  register: os_upgrade_reboot_required

- name: Decide whether to reboot
  ansible.builtin.set_fact:
    os_upgrade_should_reboot: "{{ os_upgrade_reboot_required.stat.exists or os_upgrade_force_reboot }}"

- name: Report no reboot needed
  when: not os_upgrade_should_reboot
  ansible.builtin.debug:
    msg: "No reboot required for {{ inventory_hostname }}; /var/run/reboot-required is absent."

- name: Drain and reboot block
  when: os_upgrade_should_reboot
  block:
    - name: Cordon node before reboot
      when: os_upgrade_drain
      ansible.builtin.command: >-
        kubectl cordon {{ inventory_hostname }}
      delegate_to: "{{ groups['server'][0] }}"
      changed_when: true

    - name: Drain node before reboot
      when: os_upgrade_drain
      ansible.builtin.command: >-
        kubectl drain {{ inventory_hostname }}
        --ignore-daemonsets
        --delete-emptydir-data
        --timeout={{ os_upgrade_drain_timeout }}s
      delegate_to: "{{ groups['server'][0] }}"
      changed_when: true

    - name: Reboot node
      ansible.builtin.reboot:
        reboot_timeout: "{{ os_upgrade_reboot_timeout }}"
        test_command: uptime

    - name: Wait for node to report Ready after reboot
      when: os_upgrade_drain
      ansible.builtin.command: >-
        kubectl wait --for=condition=Ready node/{{ inventory_hostname }}
        --timeout={{ os_upgrade_node_ready_timeout }}s
      delegate_to: "{{ groups['server'][0] }}"
      changed_when: false

    - name: Uncordon node after reboot
      when: os_upgrade_drain
      ansible.builtin.command: >-
        kubectl uncordon {{ inventory_hostname }}
      delegate_to: "{{ groups['server'][0] }}"
      changed_when: true
```

Notes for the implementer:
- `delegate_to: "{{ groups['server'][0] }}"` is intentional: it runs `kubectl` on the control-plane node (which has a kubeconfig) regardless of which agent is being patched.
- `changed_when: true` on cordon/drain/uncordon is honest — those mutate cluster state. `kubectl wait` is `changed_when: false` because it's a query.
- `inventory_hostname` is what we use for the kubectl node argument. If you find the cluster registered nodes under different names (e.g., short hostnames vs. IPs), this will fail visibly with "node not found"; fix via `ansible_host` mapping in inventory or by adjusting the variable here. Verify this assumption during real-cluster validation (see Task 8).

- [ ] **Step 2: Syntax check**

Run from repo root:
```bash
ansible-playbook playbooks/site.yml -i inventory-turingpi.yml --syntax-check
```
This won't include `os_upgrade` yet (no playbook uses it), but it confirms the role files at least parse against the inventory. To check the role itself, do a one-shot inline play:

```bash
cat <<'EOF' > /tmp/os-upgrade-syntax.yml
---
- hosts: localhost
  gather_facts: false
  roles:
    - role: os_upgrade
EOF
ansible-playbook /tmp/os-upgrade-syntax.yml -i inventory-turingpi.yml --syntax-check
rm /tmp/os-upgrade-syntax.yml
```
Expected: `playbook: /tmp/os-upgrade-syntax.yml` printed with no errors.

- [ ] **Step 3: Lint**

Run: `yamllint roles/os_upgrade/`
Expected: clean exit.

Run: `ansible-lint roles/os_upgrade/`
Expected: clean exit (only the same accepted warning categories as the rest of the repo).

- [ ] **Step 4: Commit**

```bash
git add roles/os_upgrade/tasks/main.yml
git commit -m "feat(os_upgrade): add reboot detection, drain, reboot, and wait-Ready flow"
```

---

### Task 4: Create `os-upgrade.yml` playbook

**Files:**
- Create: `playbooks/os-upgrade.yml`

- [ ] **Step 1: Create `playbooks/os-upgrade.yml`**

```yaml
---
# OS package patching for the cluster.
# Server is patched first without drain (single control-plane).
# Agents are patched one at a time with cordon/drain/uncordon.

- name: OS upgrade — control plane
  hosts: server
  become: true
  gather_facts: true
  serial: 1
  roles:
    - role: os_upgrade
      vars:
        os_upgrade_drain: false

- name: OS upgrade — agents
  hosts: agent
  become: true
  gather_facts: true
  serial: 1
  roles:
    - role: os_upgrade
      vars:
        os_upgrade_drain: true
```

- [ ] **Step 2: Syntax check**

Run: `ansible-playbook playbooks/os-upgrade.yml -i inventory-turingpi.yml --syntax-check`
Expected: `playbook: playbooks/os-upgrade.yml` with no errors.

- [ ] **Step 3: Lint**

Run: `yamllint playbooks/os-upgrade.yml`
Expected: clean exit.

Run: `ansible-lint playbooks/os-upgrade.yml`
Expected: clean exit (only accepted warning categories).

- [ ] **Step 4: Commit**

```bash
git add playbooks/os-upgrade.yml
git commit -m "feat: add os-upgrade.yml playbook (server then agents with drain)"
```

---

### Task 5: Create `upgrade-all.yml` wrapper playbook

**Files:**
- Create: `playbooks/upgrade-all.yml`

- [ ] **Step 1: Create `playbooks/upgrade-all.yml`**

```yaml
---
# Convenience wrapper: OS patches first, then k3s version upgrade.
# Each phase is independently re-runnable; this just chains them.

- import_playbook: os-upgrade.yml
- import_playbook: upgrade.yml
```

- [ ] **Step 2: Syntax check**

Run: `ansible-playbook playbooks/upgrade-all.yml -i inventory-turingpi.yml --syntax-check`
Expected: both `playbooks/os-upgrade.yml` and `playbooks/upgrade.yml` parse, no errors.

- [ ] **Step 3: Lint**

Run: `yamllint playbooks/upgrade-all.yml`
Expected: clean exit.

Run: `ansible-lint playbooks/upgrade-all.yml`
Expected: clean exit (only accepted warning categories).

- [ ] **Step 4: Commit**

```bash
git add playbooks/upgrade-all.yml
git commit -m "feat: add upgrade-all.yml wrapper chaining OS upgrade and k3s upgrade"
```

---

### Task 6: Update `README.md` with the new playbooks

**Files:**
- Modify: `README.md`

The README has a section starting at line 149 describing the existing upgrade playbook (`### Upgrading` or similar — confirm by reading lines 145–170). Add a new subsection right after that section ends, describing the OS upgrade playbooks.

- [ ] **Step 1: Read the current upgrade section**

Run: `sed -n '145,170p' README.md` to see the existing upgrade-section formatting and copy its style.

- [ ] **Step 2: Insert a new "OS upgrades" subsection**

Use Edit to add the following block immediately after the existing upgrade section (before the next `## ` or `### ` heading). The exact insertion point will be just before the next top-level heading after the k3s upgrade discussion ends.

```markdown
### OS package upgrades

A separate playbook applies Ubuntu/Debian package updates and reboots nodes when required. Server is patched first without drain (single control-plane cluster); agents are patched one at a time with `kubectl drain`/`uncordon`.

```bash
ansible-playbook playbooks/os-upgrade.yml -i inventory.yml
```

To do an OS patch and a k3s version bump in one maintenance window (update `k3s_version` in inventory first):

```bash
ansible-playbook playbooks/upgrade-all.yml -i inventory.yml
```

To force a reboot even if `/var/run/reboot-required` is absent:

```bash
ansible-playbook playbooks/os-upgrade.yml -i inventory.yml \
  -e os_upgrade_force_reboot=true
```

The `os_upgrade` role only supports Debian/Ubuntu — it asserts `ansible_os_family == "Debian"` and fails on other distributions.
```

- [ ] **Step 3: Lint**

Run: `yamllint README.md` (will report no targets, harmless — yamllint won't lint markdown; this is fine).

Run: `ansible-lint` from repo root.
Expected: clean exit (only accepted warning categories).

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "docs: document os-upgrade.yml and upgrade-all.yml playbooks"
```

---

### Task 7: Update `CLAUDE.md`

**Files:**
- Modify: `CLAUDE.md`

`CLAUDE.md` has a "Running Playbooks" code block (around the top of the file) and a roles table. Both need entries.

- [ ] **Step 1: Add the new playbooks to the "Running Playbooks" block**

Find the bash block under `### Running Playbooks` that lists the existing playbook invocations. Add these two entries before the "Re-fetch kubeconfig only" entry (so the order remains: deploy, upgrade, OS upgrade, full upgrade, reset, reboot, kubeconfig):

```bash
# Apply OS package updates with reboot-if-required (drain/cordon for agents)
ansible-playbook playbooks/os-upgrade.yml -i inventory.yml

# Apply OS patches and then upgrade k3s in a single run
ansible-playbook playbooks/upgrade-all.yml -i inventory.yml
```

Use the Edit tool with sufficient surrounding context (the `# Upgrade K3s version...` line just above) to make the insertion unambiguous.

- [ ] **Step 2: Add `os_upgrade` to the roles table**

Find the `| Role | Purpose |` table and add a new row immediately after the `k3s_upgrade` row:

```markdown
| `os_upgrade` | OS package patching: `apt update`/`apt upgrade`, optional autoremove, optional cordon/drain, reboot when `/var/run/reboot-required` is set |
```

- [ ] **Step 3: Lint**

Run: `ansible-lint` from repo root.
Expected: clean exit (only accepted warning categories).

- [ ] **Step 4: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add os_upgrade role and OS upgrade playbooks to CLAUDE.md"
```

---

### Task 8: Final verification — full-repo lint pass

**Files:** none modified; this is a verification gate.

- [ ] **Step 1: Run yamllint across the whole repo**

Run: `yamllint .`
Expected: clean exit. If existing files surface warnings, those are pre-existing — only fail this step on warnings/errors in files this plan touched (`roles/os_upgrade/**`, `playbooks/os-upgrade.yml`, `playbooks/upgrade-all.yml`).

- [ ] **Step 2: Run ansible-lint across the whole repo**

Run: `ansible-lint`
Expected: clean exit, or only the warning categories the repo already accepts (`var-naming[no-role-prefix]`, `yaml[comments-indentation]`, `yaml[line-length]`). No errors. If errors surface in files this plan created, fix them before proceeding.

- [ ] **Step 3: Syntax-check both new playbooks together**

Run:
```bash
ansible-playbook playbooks/os-upgrade.yml -i inventory-turingpi.yml --syntax-check
ansible-playbook playbooks/upgrade-all.yml -i inventory-turingpi.yml --syntax-check
```
Expected: both print `playbook: ...` with no errors.

- [ ] **Step 4: No commit needed** — this task is verification only. If anything failed, fix it inline (small fixes can be amended; non-trivial fixes should be a new commit) and re-run the gate.

---

### Task 9: Manual cluster validation (post-merge, on real hardware)

**Files:** none modified — this is a real-cluster validation checklist.

This task cannot be automated; it documents the validation steps the spec requires. Mark each step as you complete it.

- [ ] **Step 1: Dry run**

```bash
ansible-playbook playbooks/os-upgrade.yml -i inventory-turingpi.yml --check --diff
```
Expected: apt tasks report pending changes; reboot/drain tasks are skipped (check mode doesn't run them) but the playbook does not error.

- [ ] **Step 2: Single-agent smoke test**

```bash
ansible-playbook playbooks/os-upgrade.yml -i inventory-turingpi.yml \
  --limit 192.168.44.74
```
Expected: cordon → drain → apt upgrade → reboot (if required) → wait Ready → uncordon. The node ends up `Ready` and `Schedulable`.

Verify after completion:
```bash
kubectl get nodes
```
The targeted node should show `Ready` with no `SchedulingDisabled` taint.

- [ ] **Step 3: Full agent run**

```bash
ansible-playbook playbooks/os-upgrade.yml -i inventory-turingpi.yml \
  --limit agent
```
Expected: `serial: 1` walks all three agents in turn. Each ends `Ready`.

- [ ] **Step 4: Server run**

```bash
ansible-playbook playbooks/os-upgrade.yml -i inventory-turingpi.yml \
  --limit server
```
Expected: brief API outage during the server reboot is normal. After SSH returns, the API comes back, and `kubectl get nodes` works.

- [ ] **Step 5: Idempotency re-run**

Run `os-upgrade.yml` again immediately:
```bash
ansible-playbook playbooks/os-upgrade.yml -i inventory-turingpi.yml
```
Expected: every node logs "No reboot required..." and the play reports zero `changed=` items in the recap (apt may report a small cache update; the reboot/drain block must be skipped).

- [ ] **Step 6: Combined run (next maintenance window)**

After bumping `k3s_version` in `inventory-turingpi.yml`:
```bash
ansible-playbook playbooks/upgrade-all.yml -i inventory-turingpi.yml
```
Expected: OS phase runs, then k3s upgrade phase runs. Final `kubectl get nodes -o wide` shows all nodes on the new k3s version and `Ready`.

---

## Self-review

**Spec coverage:**
- `apt update`/`apt upgrade` — Task 2.
- Reboot only when `/var/run/reboot-required` present (with override) — Task 3 (stat + `os_upgrade_force_reboot`).
- Cordon/drain/uncordon for agents — Task 3 (gated on `os_upgrade_drain`).
- `os-upgrade.yml` playbook (server first no drain, agents with drain, `serial: 1`) — Task 4.
- `upgrade-all.yml` wrapper — Task 5.
- `roles/os_upgrade/{defaults,meta,tasks/main,tasks/debian}` layout — Task 1 + Task 2 + Task 3.
- All seven `os_upgrade_*` defaults from the spec — Task 1.
- `delegate_to: groups['server'][0]` for kubectl — Task 3.
- Ubuntu/Debian-only assert — Task 1 (placeholder), preserved in Task 3 (real implementation).
- README.md and CLAUDE.md updates — Tasks 6 and 7.
- Linting (`yamllint`, `ansible-lint`) — every task plus full pass in Task 8.
- Manual validation plan — Task 9 mirrors the six spec validation steps.

**Placeholder scan:** No "TBD"/"TODO"/"implement later" remain. The Task 1 placeholders for `tasks/main.yml` and `tasks/debian.yml` are explicitly marked as filled in by Tasks 2 and 3, and both files are fully replaced in those tasks.

**Type/name consistency:** Variable names match across tasks: `os_upgrade_drain`, `os_upgrade_apt_upgrade_type`, `os_upgrade_autoremove`, `os_upgrade_reboot_timeout`, `os_upgrade_drain_timeout`, `os_upgrade_node_ready_timeout`, `os_upgrade_force_reboot` — defined in Task 1 defaults, used unchanged in Tasks 3, 4, and 9. Registered facts: `os_upgrade_reboot_required`, `os_upgrade_should_reboot` — both used only within Task 3.
