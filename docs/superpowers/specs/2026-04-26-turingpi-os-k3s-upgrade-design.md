# Turing Pi OS + k3s Upgrade — Design

Date: 2026-04-26
Status: Draft for review
Target inventory: `inventory-turingpi.yml` (1 server + 3 agents, Ubuntu)

## Goal

Add a repeatable, safe way to apply Ubuntu package updates and bump the k3s
version on the Turing Pi cluster. Patching and k3s upgrades must be runnable
independently, and chainable as a single maintenance-window operation.

## Scope

In scope:
- `apt update` + `apt upgrade` (no `full-upgrade`/`dist-upgrade`).
- Reboot only when `/var/run/reboot-required` is present (with an override).
- Cordon/drain/uncordon for agent nodes.
- A new playbook to do OS patching and a thin wrapper that chains OS patching
  with the existing `upgrade.yml`.
- Documentation updates in `README.md` and `CLAUDE.md`.

Out of scope (explicit non-goals):
- Distribution release upgrades (`do-release-upgrade`, e.g., 22.04 → 24.04).
- Automated rollback on failure. Recovery is "fix the cause and re-run."
- Non-Debian OS support. The role will fail loudly on other `ansible_os_family`
  values; adding RHEL/SUSE is a separate role addition.
- Pre-upgrade etcd snapshots. The cluster runs a single control-plane node;
  there is no etcd HA story to protect here.
- BMC / `tpi` power management.
- New CI tests. The repo's container-based CI cannot exercise reboots or
  `kubectl drain` against real workloads, so adding a contrived test target is
  more cost than value.

## Constraints and assumptions

- Single control-plane node. There is no etcd quorum: when the server reboots,
  the API is briefly unavailable. Workloads keep running on agents during the
  outage but new scheduling and `kubectl` commands block.
- The control-plane node holds a working kubeconfig at the path the existing
  `k3s_server` role uses (`/etc/rancher/k3s/k3s.yaml`), and `kubectl` is on
  root's `$PATH` (a default k3s install).
- All inventory hosts are Ubuntu (consistent with `ansible_user: ubuntu` and
  the existing inventory). The role asserts `ansible_os_family == "Debian"`
  and fails loudly otherwise.

## Architecture

Three new artifacts. No changes to existing playbooks, roles, or inventory.

1. **`roles/os_upgrade/`** — single role. Updates apt cache, runs
   `apt upgrade`, optionally `autoremove`, checks for a required reboot,
   optionally cordons/drains, reboots, optionally waits for `Ready` and
   uncordons. Behavior is gated by `os_upgrade_drain` (per-play variable).
2. **`playbooks/os-upgrade.yml`** — server first (no drain), then agents
   one-at-a-time with drain.
3. **`playbooks/upgrade-all.yml`** — wrapper:
   `import_playbook: os-upgrade.yml` then `import_playbook: upgrade.yml`.
   No logic of its own.

### Order of operations (`upgrade-all.yml`)

1. Server: `apt upgrade` → reboot if required → wait for SSH.
2. For each agent (`serial: 1`): cordon → drain → `apt upgrade` →
   reboot if required → wait `Ready` → uncordon.
3. Existing `upgrade.yml`: server (serial: 1), then agents (parallel).

### kubectl delegation

Drain, cordon, uncordon, and `kubectl wait` all run with
`delegate_to: "{{ groups['server'][0] }}"`. This:

- Centralizes API access on the one node guaranteed to have a kubeconfig.
- Avoids requiring kubectl on agent nodes.
- Is a no-op concern for the server play because `os_upgrade_drain: false`
  there — drain is never delegated to self.

## Role internals (`roles/os_upgrade/`)

Layout:

```
roles/os_upgrade/
├── defaults/main.yml
├── tasks/main.yml
├── tasks/debian.yml
└── meta/main.yml
```

### `defaults/main.yml`

| Variable | Default | Purpose |
|---|---|---|
| `os_upgrade_drain` | `false` | Whether to cordon and drain this node before reboot. Set to `true` for agents in the playbook. |
| `os_upgrade_apt_upgrade_type` | `safe` | Passed to `ansible.builtin.apt: upgrade=`. `safe` ≡ `apt upgrade` (no removals). |
| `os_upgrade_autoremove` | `true` | Run `apt autoremove` after upgrade. |
| `os_upgrade_reboot_timeout` | `600` | Seconds to wait for SSH after reboot. |
| `os_upgrade_drain_timeout` | `300` | Seconds for `kubectl drain` to complete. |
| `os_upgrade_node_ready_timeout` | `300` | Seconds to wait for node `Ready` after reboot. |
| `os_upgrade_force_reboot` | `false` | Reboot even when `/var/run/reboot-required` is absent. Escape hatch. |

### `tasks/main.yml` flow

1. Assert `ansible_os_family == "Debian"`.
2. `include_tasks: debian.yml`.
3. `stat: path=/var/run/reboot-required` → register `reboot_required`.
4. Skip the rest if `not reboot_required.stat.exists` and not
   `os_upgrade_force_reboot`. Logs "no reboot needed."
5. If `os_upgrade_drain`: cordon, then drain with
   `--ignore-daemonsets --delete-emptydir-data --timeout=<configurable>`.
6. `ansible.builtin.reboot:` with `test_command: uptime` (cheap, doesn't
   depend on k3s being up post-reboot).
7. If `os_upgrade_drain`: `kubectl wait --for=condition=Ready node/<name>`
   (delegated), then `kubectl uncordon`.

### `tasks/debian.yml`

1. `apt: update_cache=yes cache_valid_time=3600`.
2. `apt: upgrade={{ os_upgrade_apt_upgrade_type }}`. No `force` flag —
   conflicts surface as failures.
3. `apt: autoremove=yes` when `os_upgrade_autoremove`.

### Failure modes

- `apt` fails → playbook stops, node has not been rebooted or drained.
  Re-run after fixing the underlying issue.
- `kubectl drain` times out → playbook stops, node is left **cordoned**.
  Investigate (PDBs, stuck pods), re-run; cordon is idempotent.
- Node fails to come back from reboot → `wait_for_connection` errors,
  playbook stops, agent is left cordoned. Manual recovery via console/BMC.

## Playbook contents

### `playbooks/os-upgrade.yml`

```yaml
---
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

`serial: 1` on the server play keeps control planes sequential if the
inventory grows to multi-server later. `gather_facts: true` is required for
the OS-family assert.

### `playbooks/upgrade-all.yml`

```yaml
---
- import_playbook: os-upgrade.yml
- import_playbook: upgrade.yml
```

## Invocation

```bash
# OS patches only
ansible-playbook playbooks/os-upgrade.yml -i inventory-turingpi.yml

# k3s version bump only (existing)
ansible-playbook playbooks/upgrade.yml -i inventory-turingpi.yml

# Full maintenance window: OS, then k3s
ansible-playbook playbooks/upgrade-all.yml -i inventory-turingpi.yml

# Force a reboot even if not flagged required
ansible-playbook playbooks/os-upgrade.yml -i inventory-turingpi.yml \
  -e os_upgrade_force_reboot=true
```

## Validation plan

Manual, on the real cluster:

1. `--check --diff` dry run of `os-upgrade.yml`.
2. `--limit 192.168.44.74` smoke test against one agent.
3. Full agent run (no limit; `serial: 1` walks the three agents).
4. Server run; brief API outage during reboot is expected.
5. Re-run idempotency: every node should report "no reboot needed" with
   zero changes.
6. Combined run at the next maintenance window with a bumped `k3s_version`.

Linting (matches existing repo CI):

- `yamllint .`
- `ansible-lint`

## Documentation changes

- **`README.md`** — add an "OS upgrades" section listing the three new
  invocations, in the same style as the existing playbook descriptions.
- **`CLAUDE.md`** — add `os-upgrade.yml` and `upgrade-all.yml` to the
  "Running Playbooks" block; add `os_upgrade` to the roles table.
