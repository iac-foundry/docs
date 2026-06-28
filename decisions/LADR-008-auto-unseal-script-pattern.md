# LADR-008: Auto-Unseal via Systemd Service + Script

**Status:** Accepted
**Date:** 28 June 2026
**Owner:** Platform Engineering
**Supersedes:** N/A
**Superseded By:** N/A (upgrade path to cloud auto-unseal or HSM in Phase 6)
**Related:** [../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md](../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md) · [LADR-006-vault-file-storage.md](LADR-006-vault-file-storage.md)

---

## 1. Context

- **Unsealing requirement:** Vault uses Shamir sealing in Phase 3. After sec01 restart, Vault is sealed and must be unsealed before use (manual + 3 unseal keys required by default).
- **Availability constraint:** Vernify environment expects <5 min recovery from host restart. Manual unsealing takes longer and requires operator intervention.
- **Authentication options:**
  - Cloud provider auto-unseal (not applicable to Proxmox; requires AWS KMS, Azure KeyVault, etc.)
  - Hardware security module (HSM) auto-unseal (too expensive for Phase 3; consider Phase 6)
  - Root-only mode (unsafe; root token in plaintext on host)
  - Operator-managed script with secured unseal keys
- **Operator responsibility:** Operator must securely back up and protect unseal keys.

## 2. Decision

**Vernify Phase 3 will auto-unseal Vault via a systemd service that calls a local script. The script holds Shamir unseal keys (decrypted at runtime from Vault KV storage). Operator is responsible for backing up unseal keys securely off-host.**

**Unseal mechanism:**
- **Entry point:** systemd service `vault-unsealer.service` on sec01
- **Trigger:** Runs after `vault.service` reaches `started` state; waits up to 30 seconds for Vault to seal
- **Script:** `/usr/local/bin/vault-unseal.sh` orchestrates unseal
- **Key storage:** Shamir unseal keys stored in Vault KV at `kv/platform/vault/unseal-keys` (encrypted at rest by Vault)
- **Fallback:** If script fails, operator can manually unseal via `vault unseal` CLI
- **Operator gate:** Deployment checklist requires operator to confirm unseal keys backed up before Phase 3 completion

## 3. Rationale

### Why script-based auto-unseal:
- **Applicable to Proxmox:** No cloud provider required; runs entirely on-premise.
- **Operator-controlled:** Unseal keys held by operator (not hardcoded in code); operator knows where they are and maintains backups.
- **Fast recovery:** Automatic unseal on restart (~60 seconds typical); meets <5 min recovery SLA.
- **Auditable:** Vault access logs all key retrievals; operator can verify unseal events.
- **Sufficient for Phase 3:** Small environment (<100 hosts); operator overhead acceptable.

### Trade-offs accepted:
- **Operator must back up keys securely:** Unseal keys must not be lost or leaked. Process burden on operator.
  - **Mitigation:** Deployment runbook has explicit gate: "Confirm unseal keys backed up before proceeding to Phase 4." Backup stored off-host (e.g., operator's encrypted USB drive, password manager).
- **No cloud-native auto-unseal:** Unlike AWS Lambda + KMS, this is not managed by cloud provider. Operator responsible for key lifecycle.
  - **Acceptable for Phase 3-5:** Manageable operational burden for small team.
- **Script complexity:** bash script adds to codebase; requires shell knowledge to debug.
  - **Mitigation:** Script is simple (~100 lines); thoroughly tested in CI; fallback is manual unseal (documented in runbook).

## 4. Alternatives Considered

### Option A — Root-only mode (no sealing)
| | |
|---|---|
| **Pros** | No unseal keys to manage; automatic startup |
| **Cons** | Root token stored unencrypted on host; compromised host = compromised Vault. Major security risk. |
| **Reason rejected** | Unacceptable security posture. Shamir sealing is mandatory. |

### Option B — Cloud auto-unseal (AWS KMS, Azure KeyVault)
| | |
|---|---|
| **Pros** | Managed by cloud provider; highly auditable; industry standard |
| **Cons** | Not applicable to Proxmox (on-premise). Requires cloud account + credentials; latency; cost. |
| **Reason rejected** | Vernify is on-premise; cloud services not available. Reconsidered in Phase 6+ if Vernify migrates to public cloud. |

### Option C — Hardware security module (HSM) auto-unseal
| | |
|---|---|
| **Pros** | Highest security posture; unseal key never exposed in plaintext |
| **Cons** | Significant cost (e.g., Thales Luna HSM ~$10k+); requires HSM ops expertise; Proxmox doesn't natively integrate |
| **Reason rejected** | Over-engineered for Phase 3 scale. Evaluate in Phase 6+ if compliance requirements mandate HSM. |

### Option D — Manual operator unsealing (no automation)
| | |
|---|---|
| **Pros** | Simplest implementation (zero automation code); explicit operator control |
| **Cons** | Slow recovery (operator must manually run `vault unseal` three times); high operational burden; violates <5 min SLA |
| **Reason rejected** | Violates availability SLA. Automation expected for modern infrastructure. |

## 5. Consequences

### Positive:
- **Fast recovery:** Vault unseals automatically within 60 seconds of restart; meets <5 min SLA.
- **Transparent to consumers:** Applications don't need to implement retry logic; Vault is available within 1 minute of boot.
- **Auditable:** Vault logs all key retrievals; operator can verify unseal events in Vault audit log.
- **Reversible:** Operator can switch to manual unsealing at any time by disabling systemd service.

### Negative:
- **Operator responsibility:** Operator must securely back up unseal keys. Loss of keys = inability to unseal Vault (catastrophic).
  - **Mitigation:** Deployment gate requires explicit backup confirmation. Runbook documents backup procedure. Backups checked monthly (audit task).
- **Key in plaintext at runtime:** During unseal script execution, unseal keys are briefly in plaintext (decrypted from Vault). Compromise during this window exposes keys.
  - **Mitigation:** Script runs in secure environment (sec01 only); Vault access logs capture timing; /proc filesystem not world-readable.
- **No automatic key rotation:** Unseal keys don't auto-rotate. Operator must manually manage key lifecycle.
  - **Acceptable:** Unseal keys changed only during major Vault operations (rebuild, migration). Infrequent enough to be manual.

## 6. Implementation Notes

- **systemd service:** See `ansible-collection-vault` role `vault_configure_autounseal`. Service unit at `roles/vault_configure_autounseal/files/vault-unsealer.service`.
- **Unseal script:** `/usr/local/bin/vault-unseal.sh` retrieves unseal keys from Vault KV, decrypts locally, calls `vault unseal` three times.
- **Key storage:** Unseal keys stored at `kv/platform/vault/unseal-keys` (3 keys in JSON array). Readable only by `vault-unsealer` user.
- **Logging:** All unseal attempts logged to syslog (`/var/log/syslog`). Vault audit log captures every key retrieval.
- **Fallback:** If script fails, operator can manually unseal:
  ```bash
  vault login -method=userpass username=operator password=<password>
  vault kv get kv/platform/vault/unseal-keys
  # Extract keys and unseal manually
  vault unseal <key1> && vault unseal <key2> && vault unseal <key3>
  ```
- **Validation:** CI test: stop Vault → verify sealed → start Vault → wait 90 seconds → verify unsealed automatically.

## 7. References

- [../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md](../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md) — Section 4.2: Vault Initialization & Unsealing
- [LADR-006-vault-file-storage.md](LADR-006-vault-file-storage.md) — File-based storage design
- [PHASE_3_5_OPERATOR_RUNBOOK.md](../PHASE_3_5_OPERATOR_RUNBOOK.md) — Deployment procedures and unseal key backup gate
- Vault Auto-Unseal Documentation: https://www.vaultproject.io/docs/configuration/seal
