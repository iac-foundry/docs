# Blueprints Variable Standards

**Status:** Active
**Date:** 06 June 2026
**Owner:** Platform Engineering

---

## 1. Namespacing (MUST)

Every variable a role reads or sets MUST be prefixed:

```
<component>_<role>_<thing>
```

Examples:

```yaml
jenkins_server_http_port: 8080
jenkins_server_https_port: 8443
jenkins_agent_work_dir: /var/lib/jenkins-agent
vault_server_cluster_name: ""
vault_server_api_addr: ""
graylog_server_http_bind: "0.0.0.0:9000"
keycloak_server_hostname: ""
```

Rationale: roles are composed in a shared variable space at the platform layer. Unprefixed names
collide across products. This mirrors the proven prefixing convention already used in the wider
estate (e.g. `docker_runtime_*`).

---

## 2. No generic variables (MUST NOT)

The following are **forbidden** because they leak across roles and create hidden coupling:

```yaml
# ❌ all forbidden
port: 8080
data_dir: /var/lib/x
admin_password: "..."
domain: "example.com"
image: "..."
```

Use the namespaced form instead (`jenkins_server_http_port`, etc.).

A role MAY read a small set of explicitly-documented **shared facts** a platform sets (e.g. a shared
Docker network name) **only** via a defaulted indirection, never as a bare required global:

```yaml
jenkins_server_network: "{{ blueprints_shared_network | default('bridge') }}"
```

---

## 3. Secrets (MUST)

- Secret variables are **inputs**, supplied by the caller. Roles never fetch them (Principle 2).
- Secret variable names end in a clear suffix: `_password`, `_token`, `_secret`, `_key`,
  `_bind_password`.
- Defaults for secrets MUST be empty string, never a real or placeholder credential:
  ```yaml
  vault_server_unseal_key: ""
  ```
- Any task consuming a secret sets `no_log: true`.

---

## 4. Defaults must be generic and safe (MUST)

- No organisation-specific values in `defaults/` (no real hostnames, paths, issuers).
- Network ports, dirs, and timeouts get sensible generic defaults.
- TLS-enabling vars default to "off until material supplied":
  ```yaml
  <component>_server_tls_cert: ""   # when set → TLS auto-enabled
  <component>_server_tls_key: ""
  ```

---

## 5. Common deployment-model interface (MUST)

Deployment-model sibling roles (`container_server`, `systemd_server`, `kubernetes_server`) expose
the **same** caller-facing variable names. Only internals differ.

```yaml
# identical contract regardless of deployment model
<component>_server_http_port
<component>_server_https_port
<component>_server_data_dir
<component>_server_tls_cert
<component>_server_tls_key
```

A consumer switching from `container_server` to `systemd_server` MUST NOT have to rename variables.

---

## 6. Typed and validated (MUST)

Every overridable variable appears in `meta/argument_specs.yml` with a `type` and either a `default`
or `required: true`. This makes the interface self-documenting and fail-fast.

---

## Related

- [BLUEPRINTS_ROLE_LAYOUT.md](BLUEPRINTS_ROLE_LAYOUT.md)
- [../design/BLUEPRINTS_DESIGN_PRINCIPLES.md](../design/BLUEPRINTS_DESIGN_PRINCIPLES.md)
