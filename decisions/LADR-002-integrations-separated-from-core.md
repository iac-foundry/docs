# LADR-002: Integrations Are Separated From Core Services

**Status:** Accepted
**Date:** 06 June 2026
**Owner:** Platform Engineering
**Supersedes:** N/A
**Superseded By:** N/A
**Related:** [../standards/BLUEPRINTS_INTEGRATION_STANDARDS.md](../standards/BLUEPRINTS_INTEGRATION_STANDARDS.md) · [LADR-001-blueprints-composable-collections.md](LADR-001-blueprints-composable-collections.md)

---

## 1. Context

- **Existing situation:** The legacy `blueprint.jenkins` collection baked integration logic into its
  base roles — Authentik OIDC config and a Step-CA bootstrap URL lived in `defaults`, and the agent
  role contained Vault wiring. This made the base Jenkins roles unusable for an org that uses a
  different identity provider, a different CA, or no Vault at all.
- **Constraints:** Different organisations wire the same product to *different* peers (Authentik vs
  Keycloak vs Entra; Vault vs none; GitHub vs GitLab).
- **Drivers:** A base component must be deployable on its own; wiring it to a specific peer must be
  optional and swappable.
- **Risks of inaction:** Base collections remain coupled to one org's identity/secrets/CA choices and
  cannot be reused.

## 2. Decision

**We will keep all cross-product integration logic in dedicated, optional
`blueprints.<component>_integrations` collections, and forbid integration logic in base/component
collections.** A base collection deploys exactly one product; an integration collection wires an
already-deployed product to a peer, receiving every endpoint and credential as a variable.

## 3. Rationale

- **Benefits:** Base components stay reusable and assumption-free; orgs opt into exactly the wiring
  they need; swapping a peer (Authentik → Keycloak) is a platform-layer change with no edit to the
  base collection.
- **Tradeoffs accepted:** More collections; consumers explicitly include integration roles.
- **Security considerations:** Integration credentials are caller-supplied and `no_log`; no peer
  endpoints are hardcoded.
- **Operational considerations:** A broken or evolving integration (e.g. an OIDC provider change)
  versions and releases independently of the base product.

## 4. Alternatives Considered

### Option A — Integrations as toggled code inside base roles (`when: oidc_enabled`)
| | |
|---|---|
| Pros | Single collection per product |
| Cons | Base role must know every possible peer; defaults leak org choices; untestable in isolation |
| Reason rejected | Recreates exactly the coupling this framework exists to remove |

### Option B — A single shared `integrations` collection for everything
| | |
|---|---|
| Pros | One place for wiring |
| Cons | Becomes a god-collection coupling all products; version blast radius across unrelated wirings |
| Reason rejected | Violates one-concern-per-collection; couples unrelated products |

## 5. Consequences

- **Positive:** Clean, reusable base components; opt-in, swappable integrations.
- **Negative / tradeoffs:** Consumers compose base + integration explicitly.
- **Migration impact:** Authentik/Step-CA/Vault logic moves out of `blueprints.jenkins` into
  `blueprints.jenkins_integrations`.

## 6. Implementation Notes

- **Rollout:** Create `blueprints.jenkins_integrations` (vault_auth, ldap_auth, authentik_sso,
  github_oauth, gitlab_oauth) and `blueprints.vault_integrations` (oidc_auth, ldap_auth).
- **Validation:** CI guard confirms zero integration roles and zero peer-specific defaults in base
  collections.
- **Review trigger:** Revisit if a product's core function genuinely cannot work without a specific
  peer.

## 7. References

- [../standards/BLUEPRINTS_INTEGRATION_STANDARDS.md](../standards/BLUEPRINTS_INTEGRATION_STANDARDS.md)
- [../design/BLUEPRINTS_DESIGN_PRINCIPLES.md](../design/BLUEPRINTS_DESIGN_PRINCIPLES.md) §4
