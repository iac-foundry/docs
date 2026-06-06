# Legacy documents (superseded — preserved for history)

**Status:** Superseded
**Date:** 06 June 2026

These documents predate the org-agnostic `blueprints.*` framework. They assumed specific
organisations (IOTel / Saicom / ICS) and a fixed product menu, and described the singular
`blueprint.*` collections. They are kept here for historical reference only and are **not**
authoritative.

| File | Superseded by |
|---|---|
| `SHARED_PLATFORM_COLLECTIONS_STRATEGY.md` | [../design/BLUEPRINTS_FRAMEWORK_ARCHITECTURE.md](../design/BLUEPRINTS_FRAMEWORK_ARCHITECTURE.md) · [../design/BLUEPRINTS_DESIGN_PRINCIPLES.md](../design/BLUEPRINTS_DESIGN_PRINCIPLES.md) |
| `SHARED_COLLECTIONS_MIGRATION_ORDER.md` | [../decisions/LADR-001-blueprints-composable-collections.md](../decisions/LADR-001-blueprints-composable-collections.md) (rollout) |
| `SHARED_PLATFORM_REPOSITORIES.md` | [../standards/BLUEPRINTS_COLLECTION_STRUCTURE.md](../standards/BLUEPRINTS_COLLECTION_STRUCTURE.md) · [../standards/BLUEPRINTS_GALAXY_PUBLISHING.md](../standards/BLUEPRINTS_GALAXY_PUBLISHING.md) |

The singular `blueprint.*` collections (`ansible-collection-jenkins`, `ansible-collection-vault`)
are deprecated and will be retired once consumers migrate to the plural `blueprints.*` equivalents.
See [LADR-001](../decisions/LADR-001-blueprints-composable-collections.md).
