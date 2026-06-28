# LADR-006: Vault File-Based Storage for Phase 3 (v1)

**Status:** Accepted
**Date:** 28 June 2026
**Owner:** Platform Engineering
**Supersedes:** N/A
**Superseded By:** N/A (upgrade path to Raft in Phase 6)
**Related:** [../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md](../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md) · [LADR-008-auto-unseal-script-pattern.md](LADR-008-auto-unseal-script-pattern.md)

---

## 1. Context

- **Deployment model:** Vernify Phase 3 is a greenfield, single-instance Vault deployment on sec01.
- **Scale:** ~50 secrets in Phase 3; expected growth to ~200 by Phase 5. No multi-tenant isolation needed.
- **Availability:** Single instance; no automatic failover expected. Operator accepts downtime during sec01 restart.
- **Storage options evaluated:** File-based (local filesystem), Raft (cluster), Consul (external backend), Cloud storage (not applicable to Proxmox).
- **HA constraint:** Proxmox environment (not cloud provider); no managed HA service available.

## 2. Decision

**Vernify Phase 3 will use Vault file-based storage with Shamir sealing on sec01.**

**Configuration:**
- **Backend:** File-based (path: `/var/lib/vault/data`)
- **High availability:** Disabled (single instance; `ha_enabled = false`)
- **Sealing:** Shamir seal (no auto-unseal in Phase 3; manual operator-managed unseal keys)
- **Instance count:** 1 (sec01 only; no clustering)
- **Snapshot strategy:** Daily automated backups of `/var/lib/vault/data` volume
- **Upgrade path:** Phase 6+ can add Raft clustering (integrated storage) without re-initializing Vault

## 3. Rationale

### Why file-based storage for Phase 3:
- **Simplicity:** Minimal configuration; Vault runs standalone on one host; no external dependencies (no Consul cluster, no cloud provider).
- **Operational readiness:** Single-instance Vault can be stood up, tested, and debugged quickly. Ideal for Phase 3 scope (day 1 deployment).
- **Cost:** No additional infrastructure (no HA cluster, no external backend).
- **Sufficient for current scope:** ~200 secrets fit easily in file-based backend; query performance non-critical.

### Trade-offs accepted:
- **No automatic failover:** sec01 restart causes Vault downtime. Operator must manually unseal after restart.
  - **Mitigation:** Auto-unseal via systemd service + script (see LADR-008) restores availability quickly (60 seconds typical).
- **Single point of failure:** Loss of sec01 storage volume means data loss.
  - **Mitigation:** Daily automated snapshots of `/var/lib/vault/data`; operator can restore from backup in <5 minutes.
- **No multi-instance scaling:** Cannot scale Vault horizontally while staying on file-based backend.
  - **Mitigation:** Raft backend added in Phase 6 (non-breaking upgrade; existing `/var/lib/vault/data` migrated to Raft nodes).

## 4. Alternatives Considered

### Option A — Raft clustering (Vault Integrated Storage)
| | |
|---|---|
| **Pros** | Built-in HA; automatic failover; no external backend dependency; can scale to 3-5 nodes |
| **Cons** | Requires 3 Vault instances for HA (adds cost/complexity); not needed for Phase 3 scale; Raft consensus adds latency |
| **Reason rejected** | Overkill for single-instance Phase 3. Standard upgrade path: Phase 3 file-based → Phase 6 Raft (transparent migration). |

### Option B — Consul as Vault backend
| | |
|---|---|
| **Pros** | Distributed KV store; built-in HA; can serve as service discovery |
| **Cons** | Requires Consul cluster (3+ instances); adds operational complexity; tightly couples Vault to Consul |
| **Reason rejected** | Vernify has no planned use for Consul. Simpler to use file-based in Phase 3, Raft in Phase 6. |

### Option C — Cloud storage (AWS S3, Azure Blob, etc.)
| | |
|---|---|
| **Pros** | Infinite scalability; managed service; geo-redundancy |
| **Cons** | Not applicable to Proxmox homelab; requires cloud credentials; latency/cost concerns; off-premise data |
| **Reason rejected** | Vernify is on-premise Proxmox; cloud storage not an option. |

### Option D — NFS shared storage for HA failover
| | |
|---|---|
| **Pros** | Shared backend enables multiple Vault instances to mount same data |
| **Cons** | NFS is a single point of failure; no HA benefit; adds NFS server overhead; complex mount coordination |
| **Reason rejected** | Shared filesystem doesn't provide HA (NFS is SPOF); Raft is simpler and more robust. |

## 5. Consequences

### Positive:
- **Fast deployment:** Vault boots and initializes in <1 minute; no cluster consensus overhead.
- **Operator friendly:** Single instance to debug; straightforward backup/restore procedures; all state in one directory.
- **Cost:** No additional hosts, no external services.

### Negative:
- **Downtime during restart:** sec01 reboot means Vault is sealed until unseal script runs (~60 seconds typical).
  - **Acceptable for Phase 3-5:** Limited impact on dev/test environment. Production deployments (Phase 6+) will have Raft HA.
- **Storage volume loss → data loss:** If `/var/lib/vault/data` is deleted or corrupted, Vault data is lost.
  - **Mitigation:** Automated daily snapshots; operator can restore in <5 minutes; documented in DISASTER_RECOVERY_PROCEDURES.md.
- **Scaling limit:** Single instance cannot handle >1000 TPS. Phase 6+ must migrate to Raft or external backend.
  - **Acceptable:** Vernify Phase 3-5 throughput <<100 TPS; not a blocker.

## 6. Implementation Notes

- **Storage backend config:** See `ansible-collection-vault` role `vault_configure_storage`. Sets `storage "file"` block in `/etc/vault/config.hcl`.
- **Backup strategy:** Daily Proxmox snapshot of sec01 `/var/lib/vault/data` volume; backups retained for 30 days.
- **Restore procedure:** Documented in DISASTER_RECOVERY_PROCEDURES.md. Steps: stop Vault → restore snapshot → start Vault → run unseal script.
- **Monitoring:** Prometheus alerts on `/metrics` endpoint; alert if Vault sealed for >5 minutes.
- **Validation:** CI test: stop Vault → restore from day-old snapshot → verify data integrity → unseal → confirm secrets readable.

## 7. References

- [../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md](../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md) — Section 4: Vault Design
- [LADR-008-auto-unseal-script-pattern.md](LADR-008-auto-unseal-script-pattern.md) — Unseal automation via systemd script
- [DISASTER_RECOVERY_PROCEDURES.md](../DISASTER_RECOVERY_PROCEDURES.md) — Backup and restore procedures
