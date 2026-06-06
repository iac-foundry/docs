# Blueprints Example Collections

**Status:** Active
**Date:** 06 June 2026
**Owner:** Platform Engineering

Worked examples of conformant collections. The actual scaffolds live under
[`collections/`](../../collections/). Each shows the design principles applied; task bodies in the
scaffolds are documented stubs (assert + structure), not full product implementations.

---

## `blueprints.jenkins` (component)

Owns Jenkins, nothing else. No SSO, no Vault, no CA logic (those are integrations).

```
collections/jenkins/roles/
  container_server     # Jenkins as a container
  systemd_server       # Jenkins as a systemd service
  kubernetes_server    # Jenkins on Kubernetes
  agent                # inbound build agent
```

Common interface across the three server roles:

```yaml
jenkins_server_image: ""            # required, caller-supplied
jenkins_server_http_port: 8080
jenkins_server_https_port: 8443
jenkins_server_data_dir: /var/lib/jenkins
jenkins_server_tls_cert: ""         # set → TLS auto-enabled
jenkins_server_tls_key: ""
jenkins_server_admin_token: ""      # caller-supplied secret, no_log
```

**Non-goals:** SSO/OIDC, Vault auth, CA trust → see `blueprints.jenkins_integrations`.

---

## `blueprints.vault` (component)

Owns a Vault deployment. Does not own identity or the secret-*consumption* story.

```
collections/vault/roles/
  container_server
  systemd_server
  client               # configure a host as a Vault client (agent/templating)
```

```yaml
vault_server_image: ""
vault_server_api_port: 8200
vault_server_cluster_name: ""
vault_server_data_dir: /var/lib/vault
vault_server_tls_cert: ""
vault_server_tls_key: ""
```

**Non-goals:** Configuring auth backends (OIDC/LDAP) → `blueprints.vault_integrations`. Reading
secrets for other roles → that is a platform-layer concern (see secret consumption standard).

---

## `blueprints.graylog` (component)

```
collections/graylog/roles/
  container_server
  systemd_server
  inputs               # declare log inputs
  pipelines            # declare processing pipelines
```

```yaml
graylog_server_image: ""
graylog_server_http_bind: "0.0.0.0:9000"
graylog_server_data_dir: /var/lib/graylog
graylog_server_password_secret: ""   # caller-supplied, no_log
```

---

## `blueprints.packer` (component / image building block)

```
collections/packer/roles/
  install              # install Packer (pinned version) + plugins
  template_build       # run a build from a caller-supplied template + vars
```

```yaml
packer_install_version: ""           # required; no implicit "latest"
packer_template_build_template_dir: ""
packer_template_build_var_files: []
```

**Non-goals:** Fetching cloud/vSphere credentials → caller supplies them as variables.

---

## `blueprints.common` (shared building blocks)

Small, generic helpers used **by composition** — never auto-pulled by another collection.

```
collections/common/roles/
  tls_material         # place caller-supplied cert/key with correct perms
  service_user         # create a least-privilege service account
```

These deliberately do **not** include any secret-retrieval plugin (see LADR-003).

---

## Related

- [BLUEPRINTS_DESIGN_PRINCIPLES.md](BLUEPRINTS_DESIGN_PRINCIPLES.md)
- [../standards/BLUEPRINTS_ROLE_LAYOUT.md](../standards/BLUEPRINTS_ROLE_LAYOUT.md)
- [BLUEPRINTS_EXAMPLE_PLATFORMS.md](BLUEPRINTS_EXAMPLE_PLATFORMS.md)
