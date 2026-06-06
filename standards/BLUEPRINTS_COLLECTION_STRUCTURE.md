# Blueprints Collection Directory Structure

**Status:** Active
**Date:** 06 June 2026
**Owner:** Platform Engineering

This standard prescribes the on-disk layout of every `blueprints.*` collection. It is mandatory
(MUST) unless noted.

---

## 1. Naming

- Collection namespace: **`blueprints`** (plural).
- Component collection name: the product, singular: `jenkins`, `vault`, `graylog`, `packer`,
  `common`.
- Integration collection name: `<component>_integrations`: `jenkins_integrations`,
  `vault_integrations`.
- Source repo name (one repo per collection at publish time): `ansible-collection-<name>`
  (e.g. `ansible-collection-jenkins`), per the repository naming standard.

During incubation, all collections live together under `iac-foundry/collections/<name>/`. Each maps
1:1 to its own `ansible-collection-<name>` repo when extracted for publishing.

---

## 2. Collection layout

```
collections/<name>/
├── galaxy.yml                 # namespace: blueprints, name, semver version, MIT, deps {} for components
├── README.md                  # scope, reuse contract, explicit "no hidden dependencies" statement
├── meta/
│   └── runtime.yml            # requires_ansible: ">=2.16.0"
├── changelogs/
│   └── changelog.yaml         # antsibull-changelog format
├── .ansible-lint              # production profile
├── roles/
│   └── <role>/                # see BLUEPRINTS_ROLE_LAYOUT.md
├── plugins/                   # optional; only collection-owned plugins (no cross-collection imports)
│   └── ...
└── extensions/                # optional shared molecule tooling
```

- `galaxy.yml` for **component** collections MUST set `dependencies: {}`.
- `galaxy.yml` for **integration** collections MAY depend on `ansible.posix`, `community.docker`,
  etc., but MUST NOT declare a dependency on a sibling `blueprints` **component** collection.
- A collection MUST NOT contain roles for more than one product (component collections) or for the
  base product itself (integration collections only hold wiring roles).

---

## 3. galaxy.yml template (component)

```yaml
namespace: blueprints
name: jenkins
version: 0.1.0
readme: README.md
authors:
  - Platform Engineering
description: Reusable, org-agnostic Jenkins building blocks (no hidden dependencies).
license:
  - MIT
tags:
  - jenkins
  - ci
  - platform
repository: https://github.com/iac-foundry/ansible-collection-jenkins
dependencies: {}          # components NEVER depend on other blueprints collections
build_ignore:
  - .github
  - "*.tar.gz"
  - extensions/molecule
```

---

## 4. What MUST NOT appear in a collection

- Organisation names, real hostnames, real Vault paths, real OIDC issuers (anywhere, including
  defaults and examples — use placeholders).
- A `meta/main.yml` (collection level) or role `meta` that pulls another `blueprints` component.
- Integration logic inside a component collection.

---

## Related

- [../design/BLUEPRINTS_DESIGN_PRINCIPLES.md](../design/BLUEPRINTS_DESIGN_PRINCIPLES.md)
- [BLUEPRINTS_ROLE_LAYOUT.md](BLUEPRINTS_ROLE_LAYOUT.md)
- [BLUEPRINTS_GALAXY_PUBLISHING.md](BLUEPRINTS_GALAXY_PUBLISHING.md)
