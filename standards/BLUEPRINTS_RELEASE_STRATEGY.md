# Blueprints Release Strategy

**Status:** Active
**Date:** 06 June 2026
**Owner:** Platform Engineering

Each collection is released and versioned **independently**. A change to one product never forces a
release of another. This is what gives consumers a small, predictable upgrade blast radius.

---

## 1. Semantic versioning (MUST)

Each collection version in `galaxy.yml` follows semver `MAJOR.MINOR.PATCH`:

- **MAJOR** — backwards-incompatible change to a role's variable interface or behaviour.
- **MINOR** — backwards-compatible feature (new role, new optional var with a default).
- **PATCH** — backwards-compatible fix.

The **variable interface** (`argument_specs.yml`) is the public contract. Renaming/removing a
required variable, or changing a default in a way that changes behaviour, is a MAJOR change.

---

## 2. Changelog (MUST)

Each collection maintains `changelogs/changelog.yaml` (antsibull-changelog format). Every
user-visible change adds a fragment under `changelogs/fragments/` which is compiled on release.

```yaml
# changelogs/changelog.yaml (seed)
ancestor: null
releases: {}
```

---

## 3. Tagging & branching (MUST)

- `main` is always releasable; work lands via PR with review.
- A release is cut by bumping `version` in `galaxy.yml`, compiling the changelog, and tagging.
- Tags are **per collection** and namespaced by collection name so one repo-of-record per collection
  stays unambiguous: `v<MAJOR>.<MINOR>.<PATCH>` within each `ansible-collection-<name>` repo.

This mirrors the existing estate's tag-based release approach, generalised per collection.

---

## 4. Compatibility & pinning (MUST for consumers)

- Consumers pin exact versions in `requirements.yml`.
- Component collections declare `dependencies: {}`; there is no transitive collection resolution to
  surprise a consumer.
- A platform adopts a new collection release deliberately, on its own schedule.

---

## 5. Deprecation (MUST)

- Deprecating a role or variable: mark it in the changelog and role README, keep it working for at
  least one MINOR cycle, then remove in the next MAJOR.
- Deprecating a whole collection (e.g. the legacy singular `blueprint.*`): announce in the roadmap,
  point consumers at the replacement, freeze, then archive.

---

## Related

- [BLUEPRINTS_GALAXY_PUBLISHING.md](BLUEPRINTS_GALAXY_PUBLISHING.md)
- [../design/BLUEPRINTS_FRAMEWORK_ARCHITECTURE.md](../design/BLUEPRINTS_FRAMEWORK_ARCHITECTURE.md) §4
