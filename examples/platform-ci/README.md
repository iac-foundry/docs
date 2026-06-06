# Example: platform-ci

A CI platform that composes a Jenkins controller with Vault secret management and
Authentik SSO — the "Org A" example from
[BLUEPRINTS_EXAMPLE_PLATFORMS.md](../../design/BLUEPRINTS_EXAMPLE_PLATFORMS.md).

## What this stands up

| Component | Role | Purpose |
|---|---|---|
| Jenkins controller | `blueprints.jenkins.container_server` | Core CI server running in Docker |
| Job DSL seed | `blueprints.jenkins.job_dsl` | Bootstrap pipeline definitions from code |
| Vault secret backend | `blueprints.jenkins_integrations.vault_auth` | Lets Jenkins pipelines consume Vault secrets |
| Authentik SSO | `blueprints.jenkins_integrations.authentik_sso` | OIDC login via Authentik IdP |

## When to use this example

- You already have a running Vault instance and an Authentik IdP.
- You want Jenkins pipelines to retrieve secrets from Vault (not stored in Jenkins).
- Your org uses Authentik for SSO; if you use Keycloak instead, swap
  `authentik_sso` for a `keycloak_sso` integration role — no change to
  `blueprints.jenkins.container_server`.

## Adapting for your organisation

| What to change | Where |
|---|---|
| Jenkins image version | `group_vars/all.yml` → `jenkins_server_image` |
| HTTP/HTTPS ports | `group_vars/all.yml` → `jenkins_server_http_port` / `jenkins_server_https_port` |
| Authentik endpoint | `group_vars/all.yml` → `jenkins_integration_authentik_endpoint` |
| Vault endpoint | `group_vars/all.yml` → `jenkins_integration_vault_endpoint` |
| Target hosts | `inventory/hosts.yml` |
| Vault address / AppRole creds | Environment variables `PLATFORM_VAULT_ROLE_ID` / `PLATFORM_VAULT_SECRET_ID` |

## Secret handling

Secrets are resolved in `site.yml` `pre_tasks` via `community.hashi_vault`, then passed
as variables into the roles. The roles never retrieve secrets themselves. See
[../../standards/BLUEPRINTS_SECRET_CONSUMPTION.md](../../standards/BLUEPRINTS_SECRET_CONSUMPTION.md)
and [../../decisions/LADR-003-secret-retrieval-community-hashi-vault.md](../../decisions/LADR-003-secret-retrieval-community-hashi-vault.md).

Two secrets are resolved at runtime:

| Variable | Vault path | Description |
|---|---|---|
| `jenkins_server_admin_token` | `kv/platform/ci/jenkins` → `admin_token` | Jenkins admin API token |
| `jenkins_integration_authentik_client_secret` | `kv/platform/ci/jenkins-sso` → `client_secret` | Authentik OIDC client secret |

## Deployment

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook -i inventory/hosts.yml site.yml
```

## Files

- [`site.yml`](site.yml) — composition playbook; the only place roles are combined
- [`requirements.yml`](requirements.yml) — pinned collection sources
- [`group_vars/all.yml`](group_vars/all.yml) — org-specific configuration
- [`inventory/hosts.yml`](inventory/hosts.yml) — target host(s)
