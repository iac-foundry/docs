# LADR-003: Secret Retrieval Uses `community.hashi_vault` at the Platform Layer

**Status:** Accepted
**Date:** 06 June 2026
**Owner:** Platform Engineering
**Supersedes:** N/A (org-neutral; does not govern IOTel's own `iotel.common.vault_kv` standard)
**Superseded By:** N/A
**Related:** [../standards/BLUEPRINTS_SECRET_CONSUMPTION.md](../standards/BLUEPRINTS_SECRET_CONSUMPTION.md) · [../design/BLUEPRINTS_DESIGN_PRINCIPLES.md](../design/BLUEPRINTS_DESIGN_PRINCIPLES.md)

---

## 1. Context

- **Existing situation:** IOTel built a bespoke `iotel.common.vault_kv` lookup that shells out to the
  `vault` CLI, with an addr/token/namespace fallback chain and `missing_ok` bootstrap tolerance, and
  a standard that prohibits `community.hashi_vault`. That decision is tightly coupled to IOTel's
  topology (Jenkins-agent container reaching a host Vault Agent over the Docker bridge, AppRole-only,
  a specific trust-zone model).
- **Constraints:** The `blueprints` framework serves unrelated organisations. Some use Vault with
  different auth methods; some use a non-Vault backend (Entra ID, AWS Secrets Manager); some inject
  secrets via CI. The framework must not assume any of these, and must be maintainable for 10+ years.
- **Drivers:** We need a secret-retrieval story that is org-agnostic, low-maintenance, and consistent
  with Principle 2 (roles never retrieve secrets).
- **Risks of inaction:** Re-adopting a bespoke, CLI-coupled, IOTel-specific lookup would re-import an
  organisation's topology assumptions into a framework meant to be org-neutral, and saddle us with
  security-sensitive code to maintain.

## 2. Decision

**We will not ship any secret-retrieval code in the `blueprints` framework. Component roles never
retrieve secrets; the consumer resolves secrets at the platform layer and passes them in as
variables. When the backend is HashiCorp Vault, the recommended controller-side mechanism is the
`community.hashi_vault` collection, declared as a *platform* dependency.**

## 3. Rationale

- **Benefits:** Org-agnostic (any backend, any auth method); zero framework maintenance of
  security-sensitive code; `hvac`/HTTP-based so no `vault` CLI binary needed; familiar to engineers
  (low cognitive load); fully consistent with Principle 2.
- **Tradeoffs accepted:** We give up a single bespoke "canonical surface" and the custom `missing_ok`
  ergonomics (recoverable with `default(omit)` / `failed_when` at the platform layer).
- **Security considerations:** Secrets remain `no_log`, caller-supplied, fail-fast; no backend URL or
  credential is shipped in a collection.
- **Operational considerations:** community.hashi_vault is upstream-maintained; auth-method coverage
  grows without framework changes.

## 4. Alternatives Considered

### Option A — Port the IOTel custom lookup into `blueprints` (own it)
| | |
|---|---|
| Pros | One canonical surface we control; familiar to the IOTel team |
| Cons | 10-year maintenance of security-sensitive code; reimplements auth methods hashi already has; CLI-coupled or hvac-reinventing; its fallback chain contradicts fail-fast |
| Reason rejected | High long-term cost and re-imports topology assumptions into an org-neutral framework |

### Option B — Hybrid: hashi_vault default + a thin `blueprints.common` wrapper
| | |
|---|---|
| Pros | Ergonomic canonical helper; still uses hashi under the hood |
| Cons | Extra moving part we maintain; another abstraction to learn |
| Reason rejected | Marginal ergonomic gain not worth the maintenance and cognitive load |

## 5. Consequences

- **Positive:** Framework owns no secret code; works for Vault and non-Vault orgs alike.
- **Negative / tradeoffs:** Consumers wire `community.hashi_vault` (or their backend) themselves;
  example platforms show the pattern.
- **Migration impact:** None for the framework (greenfield). IOTel's own standard is unaffected and
  out of scope.

## 6. Implementation Notes

- **Rollout:** Document the pattern in `BLUEPRINTS_SECRET_CONSUMPTION.md`; show it in the example
  platforms; add `community.hashi_vault` to platform `requirements.yml` (not to any collection).
- **Validation:** CI no-hidden-dependency guard confirms component roles contain no Vault/secret
  retrieval; example platforms `--syntax-check`.
- **Review trigger:** Revisit if `community.hashi_vault` is deprecated upstream, or if a strong
  cross-org ergonomic need emerges for a shared wrapper.

## 7. References

- [../standards/BLUEPRINTS_SECRET_CONSUMPTION.md](../standards/BLUEPRINTS_SECRET_CONSUMPTION.md)
- [../design/BLUEPRINTS_DESIGN_PRINCIPLES.md](../design/BLUEPRINTS_DESIGN_PRINCIPLES.md) §2
- community.hashi_vault — https://docs.ansible.com/ansible/latest/collections/community/hashi_vault/
