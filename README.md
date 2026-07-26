# NetFRAME Operations Documentation (redacted sample)

A two-document sample from the operations library I maintain for my home lab (**NetFRAME**),
published in redacted form (hostnames and addresses generalized to roles such as `GPU-A`,
`STORAGE`, `NODE-N`). It is shared to demonstrate operational-documentation discipline, not as an
operational document.

## The documents

Each guide is available as a **Markdown edition** (renders inline on GitHub) and a **PDF edition**
(typeset original). The content is identical; use whichever you prefer.

- **Bare-Metal Build Guide** ([Markdown](NetFRAME-BareMetal-Build-Guide.md) &middot;
  [PDF](NetFRAME-BareMetal-Build-Guide.pdf)) - the authoritative rebuild-from-nothing procedure.
  Phased (Phase 0 out-of-band secrets bootstrap, then hypervisors, cluster/quorum, firewall, DNS,
  storage/backup, core services with the secret store restored first, Kubernetes, GPU nodes), each
  phase with an explicit **validation gate** before the next. It closes the classic "the person who
  built it is unavailable" gap.
- **Disaster Recovery Guide** ([Markdown](NetFRAME-Disaster-Recovery-Guide.md) &middot;
  [PDF](NetFRAME-Disaster-Recovery-Guide.pdf)) - recovery priorities, **RTO/RPO targets per service
  tier**, the four loss scenarios (node, storage, firewall, total-site), and the secret-store
  circular-dependency break that a total-site recovery must solve first.

## What this demonstrates

- A rebuild that a competent stranger could execute, with dependency ordering and validation gates.
- Recovery-time and recovery-point *targets*, stated and honest about what is drill-validated.
- The discipline of writing the recovery path down **before** it is needed.

The summary below is drawn from both documents so the substance is readable without opening a PDF.

---

## The problem this closes

Most home labs, and plenty of production estates, exist only in the head of the person who built
them. These two documents are written for the opposite case: the authoritative, phased sequence to
reconstruct the entire environment from bare metal, in dependency order, with an explicit
validation gate at the end of every phase. It is the single document a senior engineer who did not
build the system follows to rebuild the platform from nothing. It deliberately does not replace the
detailed per-component runbooks; it sequences them and states what must be true before each one
runs.

The sharpest version of that problem is the **secrets circular dependency**. Every credential in
the environment lives in the secret store, which runs as a container inside the cluster being
rebuilt. You cannot log in to restore the secret store using a secret that is stored in the secret
store. Phase 0 exists solely to break that loop with an offline, encrypted bootstrap bundle held on
removable media plus an off-site copy, independent of the storage node and the cluster. Restoring
the secret store first in the core-services phase is the moment the rebuild stops depending on that
offline bundle. Maintaining the bundle is a standing operational task, not a rebuild-time task.

## Rebuild phases and validation gates

The build guide is ordered by hard dependency, not convenience. Each phase produces a platform
layer the next phase requires, and each ends in an explicit validation gate. A gate that does not
pass is a stop-and-diagnose point, not a proceed-anyway point.

| Phase | What it produces | Validation gate |
|---|---|---|
| **0. Out-of-band secrets bootstrap** | The offline, encrypted bundle that breaks the secrets circular dependency (age key, secret-store bootstrap secret, restic password, backup and firewall credentials) | The engineer holds and has verified every item in the bundle *before* any node is reimaged |
| **1. Bare metal and hypervisor** | Every node running the hypervisor on the management network | All nodes boot, reachable by SSH, correct hostname and time sync, servers reach the server VLAN, the pinned release is reported, no node references any other yet |
| **2. Cluster formation and quorum** | A quorate 7-node cluster | Cluster status shows 7/7 members, quorate, quorum 4, secure transport, expected config version; all rings healthy |
| **3. Firewall edge** | Routing, dual-WAN, seven VLANs, DHCP, and the single recursion point | Default gateway reachable, internet up via WAN1, inter-VLAN policy enforcing default-deny with explicit allows, DHCP leasing on each scope, recursive resolver answering, state table healthy |
| **4. DNS (primary and secondary)** | Filtered, highly available DNS with a synced secondary | Both resolvers answer external and internal names, both forward to the single recursive resolver, sync shows the secondary current, clients on every VLAN receive both resolvers via DHCP |
| **5. Storage, backup server, and NFS** | ZFS pools, the backup server, and the NFS exports core services and Kubernetes need | Both pools ONLINE (scrub started or clean), backup server reachable with its datastore mounted, NFS exports mountable from a cluster node, and a test restore of one small guest succeeds |
| **6. Core containers** | Secret store (restored first), reverse proxy and TLS, observability, tailnet, portal | Secret store unlocks and serves credentials, proxy serves valid TLS, dashboards show live metrics from all nodes with logs ingesting, tailnet re-admits nodes, alerting fires on a test. Secrets are now sourced online, not from the offline bundle |
| **7. Kubernetes** | HA control plane, CNI, load balancing, private registry | All nodes Ready, no unhealthy pods, control-plane VIP answers, load balancer assigns its pool, registry serves over TLS with a non-expired cert, test image pull and push succeed, dynamic PVC provisioning against the NFS StorageClass works |
| **8. GPU nodes** | GPU-backed scheduling and LLM inference | GPU tooling shows the expected devices with persistence and ECC on, the pinned kernel is running *and* held, the scheduler accepts and runs a test job, the inference node serves a request, SIEM manager is up with agents reporting |
| **Final cutover** | Whole-environment sign-off | Cluster 7/7 quorate with zero failed units fleet-wide, both WAN gateways online, DHCP and DNS answering, all ZFS pools ONLINE with a verified test restore, Kubernetes all-Ready, GPUs healthy on the pinned and held kernel, monitoring green and alerting verified, and the offline secrets bundle refreshed to reflect any credential changed during the rebuild |

**Dependency spine.** Phase 0 runs out-of-band before Phase 1. Phases 1 through 4 chain (bare metal,
then cluster and quorum, then the edge, then DNS). Storage (Phase 5) is a hard prerequisite for both
core services (Phase 6) and Kubernetes persistent storage (Phase 7).

## Disaster recovery: objectives and tiers

### RTO / RPO by service tier

| Service tier | RTO | RPO | Basis |
|---|---|---|---|
| Edge (firewall) | Under 1 h | Last config backup | Verified serial console plus encrypted config repo |
| DNS (two resolvers) | Under 1 h | 15 min | Secondary plus sync mirror |
| Storage / backup server | 4 to 24 h | 24 h | Single node: rebuild plus restore |
| Cluster guests (VMs, CTs) | 2 to 8 h | 24 h | Restore, manual (HA not arming guests) |
| Kubernetes | 4 to 24 h | Varies | Rebuild control planes; PV data depends on storage |
| Secrets (secret store) | Bootstrap-gated | n/a | Circular-dependency break must run first |

> These objectives are stated as **targets to validate by drill, not guarantees**. The restore path
> is not yet rehearsed; the first drill should confirm or correct these numbers.

### Recovery tiers, in the order they must come back

1. **Secrets (bootstrap-gated).** Every credential lives in the secret store, which runs inside the
   cluster being rebuilt. During a total loss you cannot unlock it, because the cluster does not
   exist yet. The out-of-band encrypted bundle held offline is the only way in. Without it a
   total-site rebuild cannot complete.
2. **Edge.** Loss removes internet, inter-VLAN routing, DHCP, and DNS recursion. Recovery is install
   from ISO on a cluster node, import the latest age-encrypted config, verify all VLAN and firewall
   rules. The verified console and cold-restore path targets minutes, not hours.
3. **DNS.** Two resolvers with a sync mirror, both forwarding to a single recursive resolver at the
   edge.
4. **Storage and backup server.** The highest-impact partial failure: all backups, Kubernetes NFS
   storage, the registry PV, and the NFS exports live there. Until it is back, Kubernetes dynamic
   provisioning and cluster backups are unavailable. A named DR gap is that backups exist in only
   one place; the documented mitigation is an offsite copy of the critical subset.
5. **Cluster guests.** Restored from backup, manually, since HA is deliberately left in standby
   rather than arming guests.
6. **Kubernetes.** Control planes rebuilt; persistent-volume data recovery depends on storage being
   back first.

### The four loss scenarios

| Scenario | Recovery posture |
|---|---|
| **Single node loss** | Quorum tolerates up to three node losses (7 votes, needs 4). Restore the node's guests to a healthy node, or rebuild the node and restore. Do not force quorum. If the lost node hosted a Kubernetes control-plane VM, restore it only after confirming the remaining etcd members are healthy (2 of 3). |
| **Storage loss** | Rebuild the storage node, import the ZFS pools (or rebuild from disks), restore the backup server. Highest-impact partial failure. |
| **Firewall loss** | Reinstall from ISO on a cluster node, import the latest encrypted config (the age key is required), verify all VLAN and firewall rules. |
| **Total-site loss** | Execute the build guide in order. The critical first step is the secrets bootstrap, then hypervisors per node, cluster and quorum, edge, DNS, storage and backups, core containers (secret store first to close the loop), Kubernetes, GPU nodes. |

## How these documents are maintained

- **Identified and cross-referenced.** Every document carries an ID (`NF-RC-010`, `NF-RC-002`), an
  owner, a classification, a last-updated date, and an explicit related-documents list, so a reader
  can traverse from build order to restore mechanics to architecture, IPAM, VLAN, naming, and
  configuration-standard references without guessing.
- **Purpose, Scope, Assumptions, Prerequisites** open every document, including what the document
  deliberately does *not* cover.
- **Gated, not narrative.** The operating discipline of the environment is one change at a time,
  verify, then proceed. A rebuild follows the same discipline at the granularity of a phase, which
  is why every phase ends in a gate rather than a summary.
- **Honest about what is unproven.** Recovery objectives are labelled as targets to validate by
  drill. Known gaps are named in-line rather than omitted (single-location backups, registry running
  on the cluster it serves, kernel pinned without an explicit hold, a single recursion point behind
  both resolvers), and the rebuild guide flags the phases where those findings should be closed
  rather than faithfully reproduced.
- **Sequencing layer, not a replacement.** These sit above the detailed per-component runbooks kept
  in the operations vault; a completed rebuild is cross-checked against a production-readiness
  checklist and recorded in the change log.

## Companion work (also public)

- [Home-Lab](https://github.com/machismo0311/Home-Lab) - the infrastructure itself.
- [netframe-reliability-assessment-public](https://github.com/machismo0311/netframe-reliability-assessment-public) - an SRE assessment of the same estate (FMEA, DR planning, roadmap).
- [netframe-security-assessment-public](https://github.com/machismo0311/netframe-security-assessment-public) - a redacted security assessment.
- [netframe-monitor](https://github.com/machismo0311/netframe-monitor) - a read-only cluster health monitor.

The full operations library (30+ documents) and the unredacted editions are kept private.
