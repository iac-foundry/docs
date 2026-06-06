# Blueprints Secret Consumption Standard

**Status:** Active
**Date:** 06 June 2026
**Owner:** Platform Engineering
**Decision:** [../decisions/LADR-003-secret-retrieval-community-hashi-vault.md](../decisions/LADR-003-secret-retrieval-community-hashi-vault.md)

How secrets reach a `blueprints.*` role. This standard is org-agnostic: it works for orgs that use
Vault, and equally for orgs that use a different backend or none at all.

---

## 1. The rule (MUST)

> A `blueprints.*` role **never retrieves a secret**. Secrets are resolved by the **caller at the
> platform layer** and passed into the role as ordinary variables.

This is Principle 2 of [../design/BLUEPRINTS_DESIGN_PRINCIPLES.md](../design/BLUEPRINTS_DESIGN_PRINCIPLES.md).
A role therefore has no opinion on *where* a secret came from — Vault, a CI secret store, Entra ID,
AWS Secrets Manager, or a manually-supplied `--extra-vars`. This is what keeps the framework usable
by unrelated organisations.

---

## 2. Recommended retrieval mechanism: `community.hashi_vault` (SHOULD)

When the backend **is** HashiCorp Vault, the recommended controller-side mechanism is the
**`community.hashi_vault`** collection. The framework does **not** ship its own Vault lookup — we do
not own secret-retrieval code (see LADR-003).

Why this collection:

- HTTP/`hvac`-based — no dependency on the `vault` CLI binary being installed on the controller.
- Supports many auth methods (token, approle, jwt/OIDC, aws, azure, kubernetes, ldap, cert) — orgs
  differ, and this covers them without us reimplementing auth.
- Externally maintained — zero long-term maintenance burden on the framework.
- Familiar to Ansible engineers — low cognitive load.

```yaml
# platform layer (group_vars / a pre_task in site.yml) — NOT inside a blueprints role
- name: Resolve Jenkins admin token from Vault (caller side)
  ansible.builtin.set_fact:
    jenkins_server_admin_token: >-
      {{ lookup('community.hashi_vault.vault_kv2_get',
                'platform/ci/jenkins',
                engine_mount_point='kv',
                url=platform_vault_addr,
                auth_method='approle',
                role_id=platform_vault_role_id,
                secret_id=platform_vault_secret_id).secret.admin_token }}
  no_log: true

# then the resolved variable is passed into the role as a normal input
- name: Deploy Jenkins
  ansible.builtin.include_role:
    name: blueprints.jenkins.container_server
  # jenkins_server_admin_token is now in scope as a plain variable
```

The Vault address and auth config are **platform variables** (`platform_vault_*`), never hardcoded
and never shipped in a collection.

---

## 3. Other backends (MAY)

An org not using Vault resolves secrets however it likes and sets the same role variables:

```yaml
# Org C — secret comes from a CI-injected environment variable
jenkins_server_admin_token: "{{ lookup('ansible.builtin.env', 'CI_JENKINS_ADMIN_TOKEN') }}"
```

The `blueprints.jenkins` role is identical in both cases — it only sees
`jenkins_server_admin_token`.

---

## 4. Security requirements (MUST)

- `no_log: true` on every task that resolves, transforms, or assigns a secret.
- Required secrets fail fast — a role asserts the variable is set; it does not default to a
  placeholder.
- No hardcoded backend URLs, paths, role-ids, or tokens in any collection file.
- No secret-resolution logic inside component roles (CI guard enforces this — see the testing
  standard).

---

## 5. What is explicitly NOT done

- The framework ships **no** custom Vault lookup plugin and **no** `vault kv get` shell-outs.
- Component roles contain **no** `community.hashi_vault` calls — that collection is a **platform**
  dependency, declared in the platform's `requirements.yml`, used in `group_vars`/`pre_tasks`.

---

## Related

- [../decisions/LADR-003-secret-retrieval-community-hashi-vault.md](../decisions/LADR-003-secret-retrieval-community-hashi-vault.md)
- [../design/BLUEPRINTS_DESIGN_PRINCIPLES.md](../design/BLUEPRINTS_DESIGN_PRINCIPLES.md) §2
- [BLUEPRINTS_TESTING_STANDARDS.md](BLUEPRINTS_TESTING_STANDARDS.md) §4 (no-hidden-dependency CI guard)
