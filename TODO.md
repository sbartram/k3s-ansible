# TODO

## Rename cluster nodes for consistency

The k3s cluster currently registers nodes under inconsistent names that
don't match the Turing Pi slot numbering:

| Slot | Inventory IP   | Current k3s node name | Target k3s node name |
|------|----------------|-----------------------|----------------------|
| 1    | 192.168.44.71  | `control-plane.local` | `tp-1`               |
| 2    | 192.168.44.72  | `worker2.local`       | `tp-2`               |
| 3    | 192.168.44.73  | `tp-3`                | `tp-3` (no change)   |
| 4    | 192.168.44.74  | `worker4.local`       | `tp-4`               |

**Why:** Operational consistency — `tp-N` matches the BMC slot number, makes
`kubectl get nodes` self-explanatory, and removes the gratuitous `.local`
suffix on three nodes.

**Why this is its own project, not a quick fix:** Renaming a live k3s node
isn't a config change — it's destructive. Touch points to think through:

- `/etc/hostname` and `/etc/hosts` on each node
- k3s node identity: drain → delete from cluster → re-register with new
  hostname (or wipe via `reset.yml` and re-provision)
- TLS SANs on the API server (current cert may include `control-plane.local`)
- PVCs / `local-path` provisioner state — anything bound by node name
- Workloads with `nodeAffinity` / `nodeSelector` referring to old names
- mDNS / avahi behavior — the `.local` suffix may come back automatically
  depending on how `systemd-resolved` or `avahi` is configured
- Inventory: `inventory-turingpi.yml` `os_upgrade_node_name` overrides
  need to be updated (or removed, once `ansible_hostname` matches)

**Suggested next step:** run `/superpowers:brainstorming` for this
rename project; produce a spec covering rollout strategy
(in-place vs reset-and-rejoin), order (workers first, control plane last),
verification steps, and rollback plan.
