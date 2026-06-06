# LADR-001: Blueprints Composable Platform Collection Framework

**Status:** Accepted
**Date:** 06 June 2026
**Owner:** Platform Engineering
**Supersedes:** N/A (legacy `blueprint.*` singular collections to be retired — see roadmap)
**Superseded By:** N/A
**Related:** [../design/BLUEPRINTS_FRAMEWORK_ARCHITECTURE.md](../design/BLUEPRINTS_FRAMEWORK_ARCHITECTURE.md) · [../design/BLUEPRINTS_DESIGN_PRINCIPLES.md](../design/BLUEPRINTS_DESIGN_PRINCIPLES.md)

---

## 1. Context

- **Existing situation:** Two collections existed under the singular `blueprint.*` namespace
  (`blueprint.jenkins`, `blueprint.vault`). They carried hidden dependencies and organisation-
  specific assumptions: Jenkins hardcoded an Authentik OIDC issuer and a Step-CA URL in defaults,
  integration logic lived inside base roles, and the surrounding estate had roles that read Vault
  directly at runtime. Strategy docs assumed a fixed product menu and specific organisations.
- **Constraints:** The framework must serve **multiple unrelated organisations** that each pick a
  different subset of technologies and mature at different paces. No shared runtime infrastructure
  between them. Intended to be maintained for 10+ years.
- **Drivers:** Duplication across orgs causes drift and inconsistent security fixes; hidden
  dependencies make components impossible to reuse or reason about ("dependent and confused").
- **Risks of inaction:** Continued coupling, per-org reimplementation, and inability to onboard a new
  org without forking.

## 2. Decision

**We will build a library of composable, independently-versioned platform building blocks under the
`blueprints` (plural) namespace, governed by [BLUEPRINTS_DESIGN_PRINCIPLES.md](../design/BLUEPRINTS_DESIGN_PRINCIPLES.md).**
One product per component collection; zero cross-component dependencies; integrations split into
separate optional `*_integrations` collections; composition only at a consumer-owned platform layer;
multiple deployment models behind a common variable interface; secure org-neutral defaults.

The new `blueprints.*` collections are built **alongside** the legacy singular `blueprint.*`
collections; the legacy ones are deprecated and retired after migration.

## 3. Rationale

- **Benefits:** Any org assembles only what it wants; a change to one product never forces a change
  to another; components are reusable because they assume nothing about their surroundings.
- **Tradeoffs accepted:** More collections/repos to manage; consumers must do explicit composition
  rather than getting a turnkey monolith.
- **Security considerations:** Org-neutral defaults remove leaked hostnames/issuers/paths; secrets
  are caller-supplied and `no_log`.
- **Operational considerations:** Independent semver per collection gives a small, predictable
  upgrade blast radius.

## 4. Alternatives Considered

### Option A — Keep evolving the singular `blueprint.*` collections in place
| | |
|---|---|
| Pros | No migration |
| Cons | Hidden deps and org-coupling already baked in; namespace mismatch with the intended design |
| Reason rejected | Cannot reach an org-agnostic, composable model by patching coupled collections |

### Option B — One monolithic platform automation product
| | |
|---|---|
| Pros | Turnkey for a single org |
| Cons | Forces every org to carry unused tech; cross-product upgrade ripple |
| Reason rejected | Cannot serve unrelated orgs with different stacks |

## 5. Consequences

- **Positive:** Clear principles to reject coupling in review; reusable across orgs.
- **Negative / tradeoffs:** Composition burden shifts to consumers (mitigated by example platforms).
- **Migration impact:** Legacy `blueprint.*` collections are deprecated, frozen, then archived.

## 6. Implementation Notes

- **Rollout:** Scaffold `blueprints.{jenkins,vault,graylog,packer,common}` and
  `blueprints.{jenkins,vault}_integrations`; ship example platforms; add CI guards for the design
  principles.
- **Validation:** The litmus test — every component role converges on a clean host with only its
  declared variables. CI no-hidden-dependency guard passes.
- **Review trigger:** Revisit if a component genuinely cannot be expressed without a sibling
  dependency, or if the namespace must change.

## 7. References

- [../design/BLUEPRINTS_FRAMEWORK_ARCHITECTURE.md](../design/BLUEPRINTS_FRAMEWORK_ARCHITECTURE.md)
- [../design/BLUEPRINTS_DESIGN_PRINCIPLES.md](../design/BLUEPRINTS_DESIGN_PRINCIPLES.md)
- [LADR-002-integrations-separated-from-core.md](LADR-002-integrations-separated-from-core.md)
- [LADR-003-secret-retrieval-community-hashi-vault.md](LADR-003-secret-retrieval-community-hashi-vault.md)
