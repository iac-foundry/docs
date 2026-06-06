# Blueprints Design Principles

**Status:** Active
**Date:** 06 June 2026
**Owner:** Platform Engineering
**Applies to:** every collection in the `blueprints.*` namespace and every consumer platform that uses them.

---

## Purpose

This is the **anchor document** for the entire `blueprints.*` framework. It exists so that we never
again build collections that are silently *dependent and confused* about what infrastructure happens
to exist around them.

Every rule below is **normative** (MUST / MUST NOT / SHOULD) and **referenceable**: in a pull
request you may reject a change with `BLUEPRINTS_DESIGN_PRINCIPLES.md §N`. Where a rule can be
machine-checked, the check is named so it can be enforced in CI rather than relying on reviewer
memory.

The framework serves **multiple unrelated organisations**. No collection may assume it runs inside
any particular organisation, alongside any particular product, or with any particular identity,
secrets, CI, or cloud provider.

---

## The litmus test

> A `blueprints.<component>` collection MUST deploy successfully on a clean host with **nothing else
> present** — no Vault, no Keycloak/Authentik, no LDAP, no GitHub/GitLab, no SSO, no CI server, no
> cloud credentials — given only the variables its `argument_specs.yml` declares.

If a role cannot pass this test, it has a hidden dependency and is **non-conformant**.

---

## Principle 1 — No hidden dependencies

A collection MUST NOT assume another collection or external system exists.

- A component role MUST function with **only** its own variables supplied.
- A component role MUST NOT `import_role` / `include_role` / `meta: dependencies` another
  `blueprints` component. Composition happens at the platform layer (Principle 5).
- `meta/main.yml` MUST declare `dependencies: []`.

❌ `roles/server/meta/main.yml` → `dependencies: [blueprints.vault.client]`
✅ `roles/server/meta/main.yml` → `dependencies: []`; the platform playbook runs the Vault client
   role separately if the org wants it.

**How it's checked:** CI greps `roles/*/meta/main.yml` for non-empty `dependencies`; CI greps
`roles/*/tasks/**` for `include_role`/`import_role` referencing another `blueprints.*` collection.

---

## Principle 2 — Variable-driven configuration (no discovery, no retrieval)

Everything a role needs MUST arrive as a variable. Roles are **pure functions of their inputs**.

A role MUST NOT, at runtime:

- query Vault, Jenkins, LDAP, Keycloak/Authentik, or any cloud API;
- retrieve secrets from any store;
- discover infrastructure (DNS lookups for peers, scanning, service discovery).

Secrets are **passed in**, never fetched by the role:

```yaml
vault_server_unseal_key:   "{{ supplied_by_caller }}"
jenkins_server_admin_token: "{{ supplied_by_caller }}"
ldap_bind_password:        "{{ supplied_by_caller }}"
```

The **caller** is responsible for obtaining values. The sanctioned caller-side pattern for secret
retrieval is a controller-side lookup plugin (e.g. the existing `vault_kv` lookup), whose *result*
is then passed into the role as an ordinary variable. The role itself stays oblivious to where the
value came from.

❌ inside a role: `command: vault kv get -field=password secret/grafana`
✅ in the platform: `grafana_server_admin_password: "{{ lookup('community.hashi_vault.vault_kv2_get', ...) }}"`
   then pass `grafana_server_admin_password` into the role.

**How it's checked:** CI greps `roles/*/tasks/**` in **component** collections for `vault`, `lookup(`,
`community.hashi_vault`, `uri:` calls to identity/cloud endpoints, and `ansible.builtin.command`
invoking discovery CLIs. Matches outside `*_integrations` collections fail the build.

---

## Principle 3 — Separation of concerns (one collection owns one component)

Each collection owns **exactly one** product and nothing else.

- `blueprints.jenkins` owns Jenkins. It does not own Vault, LDAP, or an SSO provider.
- A role MUST NOT configure a second product. Wiring two products together is an *integration*
  (Principle 4), which lives in a separate collection.

❌ A `blueprints.jenkins` role that also writes a Vault policy.
✅ `blueprints.jenkins` deploys Jenkins; `blueprints.jenkins_integrations.vault_auth` wires Jenkins
   to a Vault that already exists.

---

## Principle 4 — Integrations are separate, optional roles

Integration logic (product A talks to product B) MUST live in a dedicated
`blueprints.<component>_integrations` collection, never in the base collection.

```
blueprints.jenkins                 blueprints.jenkins_integrations
  roles:                             roles:
    container_server                   vault_auth
    systemd_server                     ldap_auth
    kubernetes_server                  authentik_sso
    agent                              github_oauth
                                       gitlab_oauth
```

- The base collection MUST contain zero integration roles.
- An integration role MAY assume the *shape* of the thing it integrates with (it is allowed to know
  "this is Vault"), but it still receives all endpoints/credentials as variables (Principle 2).
- Integrations are **opt-in**: a platform includes them only if the org wants that wiring.

This separation is the subject of **LADR-002**.

---

## Principle 5 — Composition happens only at the platform layer

The **consumer** composes a platform from building blocks. A collection MUST NOT automatically
deploy its own dependencies.

```yaml
# platform-ci/site.yml  — the ONLY place composition is allowed
roles:
  - blueprints.jenkins.container_server
  - blueprints.jenkins.job_dsl
  - blueprints.jenkins_integrations.authentik_sso   # opt-in
  - blueprints.jenkins_integrations.vault_auth       # opt-in
```

If Jenkins "needs" Vault for a given org, the **platform** runs the Vault client/integration role —
the Jenkins collection never does it implicitly.

---

## Principle 6 — Multiple deployment models, one common interface

Where it makes sense, a component MUST offer more than one deployment model, exposed as sibling
roles that share a **common variable interface** so callers can switch models without rewriting
their variables.

```
blueprints.jenkins:  container_server | systemd_server | kubernetes_server | agent
blueprints.vault:    container_server | systemd_server | client
```

Common interface example (identical across deployment models):

```yaml
jenkins_server_http_port:  8080
jenkins_server_https_port: 8443
jenkins_server_data_dir:   /var/lib/jenkins
jenkins_server_tls_cert:   ""        # TLS enabled automatically when supplied (Principle 7)
```

Only the *implementation* differs between `container_server` and `systemd_server`; the **contract**
(the variable names a caller sets) stays the same.

---

## Principle 7 — Secure, opinionated defaults

Defaults MUST be safe out of the box:

- **TLS on when certs are supplied.** If `<component>_server_tls_cert` is set, TLS is enabled
  automatically; plaintext requires an explicit opt-out.
- **Least privilege.** Dedicated service users, minimal file modes, no broad sudo.
- **Modern ciphers / security headers** where the component terminates TLS or serves HTTP.
- **Secrets are `no_log`.** Any task handling a secret variable MUST set `no_log: true`.
- Defaults MUST NOT contain organisation-specific values (no real hostnames, no real Vault paths,
  no real OIDC issuers). Defaults are generic; the org supplies specifics.

❌ default `jenkins_agent_ca_bootstrap_url: "http://vault.iotel.internal/root_ca.crt"`
✅ default `jenkins_agent_ca_bootstrap_url: ""` (caller supplies; role asserts when the feature is on)

---

## Principle 8 — Enterprise engineering standards

Every role MUST:

- be **idempotent** (a second converge reports zero changes);
- pass **ansible-lint** (production profile);
- ship a **molecule** scenario (at least `default`);
- support **check mode** where practical;
- support **tags** (every task block tagged so callers can run subsets);
- declare a typed interface via **`meta/argument_specs.yml`** (this is how Principle 2 is enforced —
  required inputs fail fast and self-document);
- mark secret-handling tasks **`no_log: true`**;
- ship a **README** stating inputs, the deployment model, and *what the role explicitly does not do*.

---

## Quick conformance checklist (for PR reviewers)

- [ ] `meta/main.yml` → `dependencies: []`
- [ ] No `include_role`/`import_role` of another `blueprints.*` component (§1)
- [ ] No Vault/LDAP/cloud/secret retrieval inside component `tasks/` (§2)
- [ ] Role configures only its own product (§3)
- [ ] Any cross-product wiring lives in `*_integrations` (§4)
- [ ] No org-specific values in `defaults/` (§7)
- [ ] Secret tasks set `no_log: true` (§7/§8)
- [ ] `meta/argument_specs.yml` present and complete (§8)
- [ ] molecule scenario present; idempotent; ansible-lint clean (§8)
- [ ] README states inputs + deployment model + non-goals (§8)

---

## Related

- [BLUEPRINTS_FRAMEWORK_ARCHITECTURE.md](BLUEPRINTS_FRAMEWORK_ARCHITECTURE.md)
- [../standards/BLUEPRINTS_VARIABLE_STANDARDS.md](../standards/BLUEPRINTS_VARIABLE_STANDARDS.md)
- [../standards/BLUEPRINTS_INTEGRATION_STANDARDS.md](../standards/BLUEPRINTS_INTEGRATION_STANDARDS.md)
- [../decisions/LADR-001-blueprints-composable-collections.md](../decisions/LADR-001-blueprints-composable-collections.md)
- [../decisions/LADR-002-integrations-separated-from-core.md](../decisions/LADR-002-integrations-separated-from-core.md)
