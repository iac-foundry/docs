# LADR-005: Single-Tier step-ca Root CA for Small Environments

**Status:** Accepted
**Date:** 28 June 2026
**Owner:** Platform Engineering
**Supersedes:** N/A
**Superseded By:** N/A (upgrade path to multi-tier in Phase 6)
**Related:** [../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md](../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md) · [LADR-004-step-ca-single-tier-pki.md](LADR-004-step-ca-single-tier-pki.md)

---

## 1. Context

- **Existing situation:** Vernify Phase 3 requires a root Certificate Authority to issue certificates for internal services (Vault, Jenkins, agents). Multi-tier PKI (root + intermediate) is industry best practice for large environments but adds operational complexity.
- **Scale:** Vernify Phase 3-5 serves <100 hosts (3 VMs in Phases 3-5; expected to grow to <50 in Phase 6).
- **Operational constraints:** Single operator team, no dedicated PKI engineer, minimal downtime tolerance.
- **Security model:** Internal homelab PKI; Vernify components trust a single root CA via `/etc/ssl/certs/` anchored root. No hierarchical delegation needed.
- **Risk:** Root CA private key must be protected; compromise requires flag-day re-trust of all issued certificates.

## 2. Decision

**Vernify will use a single-tier root CA (step-ca running as root) for Phase 3-5.**

The root CA is the only certificate authority. All leaf certificates (Vault, Jenkins, agents, etc.) are issued directly by the root CA. There is no intermediate CA.

**Configuration:**
- **CA role:** `step-ca` container on sec01, running as root user (trusted execution environment)
- **Certificate lifespan:** Leaf certs issued with 1-year validity; renewal via Vault Cert Auth before expiry
- **Root cert validity:** 10 years (sufficient for Phase 3-5 timeline; reviewed in Phase 6 planning)
- **Storage:** Root private key stored in Vault KV (backed up off-host)
- **Trusted root:** Root certificate distributed to all consumers via `/etc/ssl/certs/` + system trust store refresh

## 3. Rationale

### Benefits of single-tier for this scale:
- **Operational simplicity:** Fewer CA operations to document and train operators on (no intermediate CA renewal, no hierarchical trust verification).
- **Sufficient for small environments:** <100 hosts do not require delegation; a single root CA can issue all needed certificates within a reasonable time budget.
- **Faster certificate issuance:** Direct root-to-leaf eliminates intermediate CA signing step; improves provisioning speed during Phase 3 deployment.
- **Reduced attack surface:** Fewer CA instances to secure; operator can focus on protecting the single root.

### Trade-offs accepted:
- **No intermediate revocation:** If an intermediate CA were compromised, an entire cert category could be revoked without flag-day re-trust. Single-tier cannot do this; a compromised root requires re-trust of all certs.
- **Root rotation requires flag-day re-trust:** If the root CA must be rotated (key rotation, algorithm upgrade), all consumers must be redeployed with the new root cert. Acceptable for Phases 3-5 (small environment); becomes pain point at >500 hosts (triggers Phase 6 migration to multi-tier).

### Security considerations:
- **Root key protection:** step-ca runs in a container on sec01 with restricted network access. Container runs as root to sign certificates. Root private key is backed up in Vault KV and on operator's secure storage.
- **Trust anchor:** Root certificate is deployed to all consumers during provisioning (embedded in Terraform modules; Ansible playbooks refresh system trust store).

## 4. Alternatives Considered

### Option A — Multi-tier PKI (root + intermediate)
| | |
|---|---|
| **Pros** | Industry standard; intermediate revocation possible without flag-day re-trust; supports delegation to teams |
| **Cons** | Added operational complexity (intermediate renewal, hierarchical trust verification); overhead not justified for <100 hosts; longer cert issuance chain |
| **Reason rejected** | Operational cost outweighs security benefit for this scale. Revisit if Vernify grows to >500 hosts or requires delegated issuance. |

### Option B — Certificate-only CA (no autonomy)
| | |
|---|---|
| **Pros** | All certificates signed by external vendor CA (e.g., LetsEncrypt); no private key management |
| **Cons** | LetsEncrypt rate limits (50 certs/domain/week); no internal wildcard support; renewal latency; dependency on external service |
| **Reason rejected** | Vernify requires internal-only certificates; external CA not an option. Homelab must host its own root. |

### Option C — Hardware security module (HSM) for root key
| | |
|---|---|
| **Pros** | Highest security posture; prevents key extraction; auditability |
| **Cons** | Significant cost and operational complexity; Proxmox doesn't have native HSM integration; overcomplicated for Phase 3-5 |
| **Reason rejected** | Out of scope for current phase. Evaluate in Phase 6+ if compliance requirements change. |

## 5. Consequences

### Positive:
- **Simpler operations:** Operators debug one CA, not multiple; provisioning scripts simpler; fewer runbook procedures.
- **Faster deployment:** Certificate issuance pipeline faster (no intermediate signing); Phase 3 deployment completes on schedule.
- **Cost:** No additional infrastructure (HSM, external CA).

### Negative:
- **Root compromise exposure:** Private key theft means all issued certificates are compromised; requires flag-day re-issue of all leaf certs.
  - **Mitigation:** Key stored in Vault KV (encrypted at rest); backup copy on operator's secure storage; container runs with network isolation; Vault access logs all key retrievals.
- **Scaling limit:** Single root CA not suitable for >500 hosts; Phase 6 must migrate to multi-tier.
  - **Mitigation:** Explicitly tracked in Phase 6 requirements; not a blocker for Phase 3-5.

## 6. Implementation Notes

- **Deployment:** See `ansible-collection-step-ca` role `configure_root_ca`. Root private key generated during first provisioning; stored in Vault KV at `kv/platform/pki/root-key`.
- **Certificate distribution:** Terraform `local-exec` provisioner distributes root cert to all consumers; Ansible playbooks refresh system trust store via `update-ca-certificates`.
- **Renewal:** Leaf certificates renewed by Vault Cert Auth endpoint before expiry (90-day renewal window). Vault handles renewal logic; operators monitor via Prometheus alerts.
- **Validation:** CI test: provision 10 leaf certificates in parallel; verify all issued by root CA; verify leaf cert chain resolves to root.

## 7. References

- [../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md](../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md) — Section 3: step-ca PKI Design
- [LADR-004-step-ca-single-tier-pki.md](LADR-004-step-ca-single-tier-pki.md) — Original PKI framework decision
- [../design/BLUEPRINTS_STEP_CA_PKI_DESIGN.md](../design/BLUEPRINTS_STEP_CA_PKI_DESIGN.md) — Detailed PKI architecture
