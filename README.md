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
