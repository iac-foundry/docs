# Blueprints Legacy Retirement Roadmap

**Status:** Active
**Date:** 06 June 2026
**Owner:** Platform Engineering

The new `blueprints.*` (plural) collections are built **alongside** the legacy `blueprint.*`
(singular) collections. The legacy ones are deprecated and retired only after consumers migrate.

## Retirement stages

| Stage | Legacy `blueprint.*` | Gate to advance |
|---|---|---|
| 1. Deprecate (now) | banner added to `ansible-collection-{jenkins,vault}` READMEs | new `blueprints.*` collections lint + build green |
| 2. Reach parity | `blueprints.{jenkins,vault}` implement the roles consumers actually use | molecule converge + idempotence per role |
| 3. Migrate consumers | platforms repin `requirements.yml` to `blueprints.*` | each platform `--syntax-check` + `--check` clean |
| 4. Freeze | no new changes to `blueprint.*` | all known consumers migrated |
| 5. Archive | remove `ansible-collection-{jenkins,vault}` source | one release cycle after freeze |

## Migration notes

- Authentik OIDC and Step-CA logic moves out of the base Jenkins collection into
  `blueprints.jenkins_integrations` (LADR-002).
- The planned in-collection `vault_kv` lookup is **dropped**; secret retrieval becomes a platform
  concern via `community.hashi_vault` (LADR-003).
- Namespace changes `blueprint` → `blueprints`; consumers update collection names accordingly.

## References

- [../decisions/LADR-001-blueprints-composable-collections.md](../decisions/LADR-001-blueprints-composable-collections.md)
- [../decisions/LADR-002-integrations-separated-from-core.md](../decisions/LADR-002-integrations-separated-from-core.md)
- [../decisions/LADR-003-secret-retrieval-community-hashi-vault.md](../decisions/LADR-003-secret-retrieval-community-hashi-vault.md)
- [../legacy/README.md](../legacy/README.md)
