# Blueprints Knowledge Base

Org-neutral, self-contained documentation for the `blueprints.*` composable platform
collection framework. Any organisation may adopt this framework and mirror these docs
without edits.

## New to iac-foundry?

👉 **[AGENTS.md](AGENTS.md)** — Fast navigation guide. Find the right docs based on what you're building (implementing a collection, reviewing a PR, composing a platform, etc.).

## Start here

- **[design/BLUEPRINTS_DESIGN_PRINCIPLES.md](design/BLUEPRINTS_DESIGN_PRINCIPLES.md)** — the anchor.
  The 8 enforceable rules that keep collections free of hidden dependencies. Reference these in PRs.
- [design/BLUEPRINTS_FRAMEWORK_ARCHITECTURE.md](design/BLUEPRINTS_FRAMEWORK_ARCHITECTURE.md) — the
  three-layer model (component / integration / platform).

## Standards (mandatory practices)

| Standard | Output |
|---|---|
| [BLUEPRINTS_COLLECTION_STRUCTURE.md](standards/BLUEPRINTS_COLLECTION_STRUCTURE.md) | collection directory structure |
| [BLUEPRINTS_ROLE_LAYOUT.md](standards/BLUEPRINTS_ROLE_LAYOUT.md) | role layout |
| [BLUEPRINTS_VARIABLE_STANDARDS.md](standards/BLUEPRINTS_VARIABLE_STANDARDS.md) | variable naming |
| [BLUEPRINTS_INTEGRATION_STANDARDS.md](standards/BLUEPRINTS_INTEGRATION_STANDARDS.md) | integrations |
| [BLUEPRINTS_SECRET_CONSUMPTION.md](standards/BLUEPRINTS_SECRET_CONSUMPTION.md) | secret retrieval |
| [BLUEPRINTS_TESTING_STANDARDS.md](standards/BLUEPRINTS_TESTING_STANDARDS.md) | testing |
| [BLUEPRINTS_DOCUMENTATION_STANDARDS.md](standards/BLUEPRINTS_DOCUMENTATION_STANDARDS.md) | documentation |
| [BLUEPRINTS_RELEASE_STRATEGY.md](standards/BLUEPRINTS_RELEASE_STRATEGY.md) | release strategy |
| [BLUEPRINTS_GALAXY_PUBLISHING.md](standards/BLUEPRINTS_GALAXY_PUBLISHING.md) | Galaxy publishing |
| [REPOSITORY_NAMING_STANDARD.md](standards/REPOSITORY_NAMING_STANDARD.md) | repository naming |
| [REPO_VISIBILITY_AND_PROTECTION.md](standards/REPO_VISIBILITY_AND_PROTECTION.md) | repo visibility and branch protection |

## Decisions (LADRs)

- [LADR-001](decisions/LADR-001-blueprints-composable-collections.md) — adopt the framework.
- [LADR-002](decisions/LADR-002-integrations-separated-from-core.md) — integrations split from core.
- [LADR-003](decisions/LADR-003-secret-retrieval-community-hashi-vault.md) — secret retrieval via
  `community.hashi_vault` at the platform layer.

## Vernify Phase 3-5 (Vault + step-ca + Jenkins bootstrap)

Phase 3-5 is a concrete consumer deliverable built on this framework (see
[AGENTS.md](AGENTS.md) for the firm's stage-by-stage delivery process). The documentation for it
is split across this repo and two repos in `vernify/` — use this table to find the right
Diataxis-type document instead of searching:

| Type | Document | Where |
|---|---|---|
| Explanation | [design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md](design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md) | this repo |
| Explanation | [decisions/LADR-004](decisions/LADR-004-step-ca-single-tier-pki.md) — single-tier step-ca PKI | this repo |
| Explanation | [decisions/LADR-006](decisions/LADR-006-vault-file-storage.md) — Vault file storage | this repo |
| Explanation | [decisions/LADR-007](decisions/LADR-007-jenkins-container-agent-systemd.md) — Jenkins container/agent split | this repo |
| Explanation | [decisions/LADR-008](decisions/LADR-008-auto-unseal-script-pattern.md) — Vault auto-unseal pattern | this repo |
| Explanation | [decisions/LADR-009](decisions/LADR-009-non-root-containers.md) — non-root containers | this repo |
| Explanation | [decisions/LADR-010](decisions/LADR-010-bootstrap-script-orchestration.md) — bootstrap script orchestration | this repo |
| Reference | [reference/PHASE_3_5_INTEGRATION_GUIDE.md](reference/PHASE_3_5_INTEGRATION_GUIDE.md) — Vault/step-ca/Jenkins API & integration reference | this repo |
| Reference | [standards/VAULT_SECRET_PATH_MODEL.md](standards/VAULT_SECRET_PATH_MODEL.md) — canonical Vault secret path model | this repo |
| Reference | [delivery/PHASE_3_5_DELIVERY_PLAN.md](delivery/PHASE_3_5_DELIVERY_PLAN.md), [delivery/CI_CD_HEALTH_DASHBOARD.md](delivery/CI_CD_HEALTH_DASHBOARD.md) | this repo |
| Tutorial | [`bootstrap-container/RUNBOOK.md`](https://github.com/Vernify/bootstrap-container/blob/main/RUNBOOK.md) — first-time, zero-to-bootstrapped walkthrough of Phase 0-5 | `vernify/bootstrap-container` |
| How-to | `roadmap/PHASE_3_5_OPERATOR_RUNBOOK.md` — day-by-day operator deployment, verification, and troubleshooting procedures | `vernify/roadmap` (not yet version-controlled or hosted on GitHub — read from the local path) |
| How-to | `roadmap/DISASTER_RECOVERY_PROCEDURES.md` — recovery procedures for Vault/step-ca/VM/Jenkins loss | `vernify/roadmap` (not yet version-controlled or hosted on GitHub — read from the local path) |

> **Note on LADR-005:** a duplicate single-tier-PKI decision record was independently drafted as
> `LADR-005`; it has been consolidated into LADR-004 and marked Superseded. It is retained for
> history at [decisions/LADR-005-step-ca-single-tier.md](decisions/LADR-005-step-ca-single-tier.md)
> but new references should point to LADR-004.

## Examples

Working reference implementations showing how to compose collections into complete platforms:

| Example | What it assembles |
|---|---|
| [examples/platform-ci/](examples/platform-ci/) | Jenkins + Vault auth + Authentik SSO |
| [examples/platform-monitoring/](examples/platform-monitoring/) | Graylog log management |
| [examples/platform-identity/](examples/platform-identity/) | HashiCorp Vault + OIDC auth backend |

See [examples/README.md](examples/README.md) for how to use and adapt these.

## Roadmap & legacy

- [roadmap/BLUEPRINTS_LEGACY_RETIREMENT.md](roadmap/BLUEPRINTS_LEGACY_RETIREMENT.md)
- [legacy/](legacy/) — superseded documents, preserved for history.
