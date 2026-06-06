# Blueprints Galaxy Publishing Strategy

**Status:** Active
**Date:** 06 June 2026
**Owner:** Platform Engineering

How `blueprints.*` collections are built and distributed to consumers.

---

## 1. Repo-of-record (MUST)

Each collection has its own source repo `ansible-collection-<name>` (extracted from the incubation
monorepo `iac-foundry/collections/<name>/`). One collection ↔ one repo ↔ one independent release
stream.

---

## 2. Distribution channels

Two supported channels; a consuming org picks per its maturity:

| Channel | When | requirements.yml |
|---|---|---|
| **Git source (pinned tag)** | default for private/internal use; no registry needed | `source: git+https://github.com/iac-foundry/ansible-collection-jenkins.git`, `version: v0.1.0`, `type: git` |
| **Galaxy/Automation Hub artifact** | when published to a registry | `name: blueprints.jenkins`, `version: "0.1.0"` |

Either way consumers **pin a version**. Git-source pinning matches the existing estate convention and
needs no registry; the artifact channel is available for orgs that run a Galaxy/AH instance.

---

## 3. Build (MUST)

```bash
cd collections/<name>
ansible-galaxy collection build --output-path dist/
```

`galaxy.yml` `build_ignore` excludes `.github`, molecule extensions, and build artifacts from the
tarball.

---

## 4. Publish (CI)

A reusable `release.yml` workflow (see `.github/workflows-templates/`) runs on tag:

1. lint + molecule gate,
2. `ansible-galaxy collection build`,
3. publish — either attach the tarball to the GitHub release (git-source channel) or
   `ansible-galaxy collection publish` to the configured registry using a token secret.

Publishing credentials are CI secrets, never committed; the workflow sets `no_log`-equivalent
masking for the token.

---

## 5. Namespace ownership

The `blueprints` namespace is owned centrally. Only collections that conform to
[BLUEPRINTS_DESIGN_PRINCIPLES.md](../design/BLUEPRINTS_DESIGN_PRINCIPLES.md) may publish under it; the
no-hidden-dependency CI guard is a release gate.

---

## Related

- [BLUEPRINTS_RELEASE_STRATEGY.md](BLUEPRINTS_RELEASE_STRATEGY.md)
- [BLUEPRINTS_COLLECTION_STRUCTURE.md](BLUEPRINTS_COLLECTION_STRUCTURE.md)
