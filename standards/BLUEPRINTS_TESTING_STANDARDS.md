# Blueprints Testing Standards

**Status:** Active
**Date:** 06 June 2026
**Owner:** Platform Engineering

Every `blueprints.*` role MUST be testable in isolation, on a clean host, with no surrounding
infrastructure.

---

## 1. ansible-lint (MUST)

- Every collection ships `.ansible-lint` using the **production** profile.
- CI fails on any lint error. Targeted `# noqa` is allowed only with an inline justification.

```yaml
# .ansible-lint
profile: production
exclude_paths:
  - molecule/
  - .github/
```

---

## 2. Molecule (MUST)

- Every role ships at least a `default` molecule scenario (`molecule/default/`).
- Driver: containers (Docker/Podman) by default; a role whose deployment model is itself a container
  uses a systemd-capable image or the `delegated`/`default` driver as appropriate.
- The scenario MUST converge with **only caller-supplied variables** — proving the role has no hidden
  dependencies (the litmus test from the design principles).
- **Idempotence** is mandatory: the second converge reports zero changes (`molecule test` runs the
  idempotence step).

```yaml
# molecule/default/molecule.yml
---
driver:
  name: default
platforms:
  - name: instance
    image: ${MOLECULE_IMAGE:-quay.io/ansible/molecule-ubuntu:noble}
provisioner:
  name: ansible
verifier:
  name: ansible
```

```yaml
# molecule/default/converge.yml
---
- name: Converge
  hosts: all
  gather_facts: true
  tasks:
    - name: Include role with the minimum required (caller-supplied) variables
      ansible.builtin.include_role:
        name: <namespace>.<collection>.<role>
      vars:
        <component>_server_image: "example/image:latest"   # supplied by caller, not discovered
```

---

## 3. Check mode (SHOULD)

Roles SHOULD converge cleanly under `--check` where the deployment model allows it. Unavoidable
imperative steps set `check_mode:` / `changed_when:` so check runs are honest.

---

## 4. No-hidden-dependency guard (MUST, CI)

A scripted CI guard enforces Principles 1 & 2 on **component** collections (not `*_integrations`):

```bash
# fails the build if a component role queries external systems or pulls a sibling collection
grep -REn \
  -e 'community\.hashi_vault' \
  -e '\bvault\b +(kv|read|login|write)' \
  -e "lookup\(['\"]?(hashi_vault|community\.hashi_vault" \
  -e '(import_role|include_role).*blueprints\.' \
  collections/<component>/roles/*/tasks/ && exit 1 || exit 0
```

(Integration collections are exempt from the "knows about another product" part but still MUST NOT
retrieve secrets — they receive them as variables.)

---

## 5. What tests must NOT require

- A running Vault / LDAP / SSO / CI server.
- Network access to any organisation's infrastructure.
- Real secrets. Test fixtures use obvious dummies and `no_log` still applies.

---

## Related

- [BLUEPRINTS_ROLE_LAYOUT.md](BLUEPRINTS_ROLE_LAYOUT.md)
- [../design/BLUEPRINTS_DESIGN_PRINCIPLES.md](../design/BLUEPRINTS_DESIGN_PRINCIPLES.md) §8
