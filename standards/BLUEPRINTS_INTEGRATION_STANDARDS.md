# Blueprints Integration Standards

**Status:** Active
**Date:** 06 June 2026
**Owner:** Platform Engineering

How to build the optional `blueprints.<component>_integrations` collections that wire one product to
another. The rationale for keeping these separate from core is in
[../decisions/LADR-002-integrations-separated-from-core.md](../decisions/LADR-002-integrations-separated-from-core.md).

---

## 1. Where integration logic lives (MUST)

- All wiring (product A ↔ product B) lives in `blueprints.<component>_integrations`.
- A base/component collection MUST contain **zero** integration roles.
- An integration collection contains **only** wiring roles — it does not deploy the base product.

```
blueprints.jenkins_integrations
  roles:
    vault_auth       # configure Jenkins to use an existing Vault
    ldap_auth        # configure Jenkins to use an existing LDAP
    authentik_sso    # configure Jenkins OIDC against an existing Authentik
    github_oauth
    gitlab_oauth
```

---

## 2. Integration role contract

An integration role:

- targets **one side** (it configures Jenkins to talk to Vault; it does not also deploy/manage
  Vault);
- receives **all** endpoints and credentials as variables (Principle 2) — it never discovers or
  retrieves them;
- is **idempotent** and safe to run after the base product is already deployed;
- assumes the base product is **already present** (the platform ordered it first) but MUST fail with
  a clear message if prerequisites/variables are missing, rather than trying to create them.

Naming: `<component>_integration_<target>_<thing>`, e.g.:

```yaml
jenkins_integration_authentik_issuer_url: ""        # caller-supplied
jenkins_integration_authentik_client_id: ""
jenkins_integration_authentik_client_secret: ""     # no_log
jenkins_integration_vault_addr: ""
jenkins_integration_vault_role: ""
```

---

## 3. What an integration MAY assume vs MUST NOT

| MAY | MUST NOT |
|---|---|
| Know the *shape* of the target ("this is Vault / OIDC") | Hardcode the target's hostname/issuer/path |
| Require the base product to already be deployed | Deploy or upgrade the base product itself |
| Require specific variables and assert on them | Discover/scan/query to find the target |
| Be opt-in in a platform | Be auto-included by a component collection |

---

## 4. Composition (platform layer)

A platform includes a base role and then, optionally, integration roles:

```yaml
roles:
  - blueprints.jenkins.container_server
  - blueprints.jenkins_integrations.authentik_sso   # only if the org uses Authentik
  - blueprints.jenkins_integrations.vault_auth       # only if the org uses Vault
```

Swapping Authentik for Keycloak is a platform edit (different integration role + vars), with **no
change** to `blueprints.jenkins`.

---

## Related

- [../design/BLUEPRINTS_DESIGN_PRINCIPLES.md](../design/BLUEPRINTS_DESIGN_PRINCIPLES.md) §4
- [../decisions/LADR-002-integrations-separated-from-core.md](../decisions/LADR-002-integrations-separated-from-core.md)
