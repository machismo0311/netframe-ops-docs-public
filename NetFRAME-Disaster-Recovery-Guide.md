# Disaster Recovery Guide

| Field | Value |
|---|---|
| **Document ID** | NF-RC-002 |
| **Owner** | Infrastructure Engineering |
| **Classification** | PUBLIC (redacted) |
| **Last updated** | 2026-07-13 |
| **Related documents** | NF-RC-001, NF-RC-010, NF-OP-030, NF-AR-030, NF-GV-001 |

> This is the Markdown edition of the guide, provided so it renders natively on GitHub.
> The [PDF edition](NetFRAME-Disaster-Recovery-Guide.pdf) is the typeset original; both carry the
> same content. This is a **redacted public sample**: addresses and hostnames are generalized to
> roles (`GPU-A`, `STORAGE`, `NODE-N`), and container/VM IDs, service ports, exact versions, and
> internal repository names are omitted. It is not an operational document. The full internal edition
> is kept private.

**Purpose.** Provide recovery scenarios, objectives, and ordered procedures for partial and total
loss.

**Scope.** Node loss, storage loss, firewall loss, and total-site loss. The full bare-metal build
sequence is in NF-RC-010; restore mechanics are in NF-RC-001.

**Assumptions.** Backups are current and verified (dead-man switch alert active). PBS lives on
STORAGE.

**Prerequisites.** Physical/console access; the out-of-band secrets bootstrap (see below and
NF-OP-030).

---

## 1. Recovery objectives (RTO / RPO)

| Service tier | RTO | RPO | Basis |
|---|---|---|---|
| Edge (OPNsense) | Under 1 h | Last config backup | Verified serial console + encrypted config repo |
| DNS (Pi-hole x2) | Under 1 h | 15 min | Secondary + sync mirror |
| Storage / PBS (STORAGE) | 4 to 24 h | 24 h | Single node; rebuild + restore |
| Cluster guests (VMs/CTs) | 2 to 8 h | 24 h | PBS restore, manual (HA not arming guests) |
| Kubernetes (RKE2) | 4 to 24 h | Varies | Rebuild CPs; PV data depends on STORAGE |
| Secrets (the secret store) | Bootstrap-gated | n/a | See circular-dependency break below |

> **Note.** These objectives are targets to validate by drill, not guarantees. The restore path is
> not yet rehearsed (see NF-RC-001); the first drill should confirm or correct these numbers.

---

## 2. Scenario: single node loss

Quorum tolerates up to three node losses (7 votes, needs 4). Recover: restore the node's guests from
PBS to a healthy node (NF-RC-001), or rebuild the node (NF-RC-010) and restore. Do not force quorum.
If the lost node hosts a RKE2 control-plane VM, restore it only after confirming the remaining etcd
members are healthy (2 of 3).

---

## 3. Scenario: STORAGE (storage) loss

This is the highest-impact partial failure: all PBS backups, Kubernetes NFS storage, the registry PV,
and NFS exports are on STORAGE. Recovery: rebuild STORAGE (NF-RC-010), import the ZFS pools (or
rebuild from disks), and restore PBS. Until STORAGE is back, Kubernetes dynamic provisioning and
cluster backups are unavailable.

> **Caution.** Mitigate before it happens: add an offsite copy of the critical backup subset (an
> offsite object store or a second PBS) so STORAGE is not the only place backups exist. This is a
> documented DR gap.

---

## 4. Scenario: OPNsense (firewall) loss

Loss removes internet, inter-VLAN routing, DHCP, and DNS recursion. Recover: install OPNsense from
ISO on a cluster node, import the latest encrypted config from the config backup repo (the age key is
required, see below), and verify all VLAN/firewall rules. The verified serial console procedure and
cold-restore runbook target minutes, not hours.

---

## 5. Scenario: total-site loss (bare-metal rebuild)

Execute NF-RC-010 (Build Guide) in order. The critical first step is the secrets bootstrap.

### 5.1 The secrets circular dependency (must solve first)

All secrets live in the secret store, which runs as a container inside the cluster being rebuilt.
During a total loss you cannot unlock the secret store because the cluster does not exist yet.

> **Caution.** Maintain an out-of-band, encrypted bootstrap bundle stored offline (not in the
> cluster): the secret store export (or master credential), the age key for the OPNsense config repo,
> and the restic repository password. Without this bundle a total-site rebuild cannot complete. See
> NF-OP-030.

### 5.2 Rebuild order (summary)

Bootstrap secrets -> Proxmox on each node -> cluster/quorum -> OPNsense (network) -> Pi-hole DNS ->
STORAGE storage + PBS -> core LXCs (the secret store first to close the loop) -> RKE2 -> GPU nodes.
Each phase has a validation gate in NF-RC-010.

---

## 6. Troubleshooting

| Symptom | Action |
|---|---|
| Cannot unlock the secret store during rebuild | Use the out-of-band bootstrap bundle; do not block the rebuild waiting for it. |
| PBS restore fails (no backups reachable) | Confirm STORAGE pools imported; check the STORAGE PBS datastore; fall back to offsite copy if present. |
| DNS down cluster-wide after edge restore | Confirm the edge resolver and both Pi-hole upstreams; see NF-OP-004. |
| Quorum will not form | Bring up 4 of 7 nodes; check corosync and time sync; never force before that. |

---

**See also:** NF-RC-001 Backup/Restore; NF-RC-010 Build Guide; NF-OP-030 Identity and Secrets;
NF-AR-030 Storage; NF-GV-001 Risk Register.
