# Example: platform-identity

A secrets and identity platform that stands up HashiCorp Vault with an OIDC auth backend
wired to the organisation's IdP. The IdP is caller-supplied — this composition works with
Authentik, Keycloak, Entra ID, Okta, or any OIDC-compliant provider.

## What this stands up

| Component | Role | Purpose |
|---|---|---|
| Vault server | `blueprints.vault.container_server` | Secrets management, running in Docker |
| OIDC auth backend | `blueprints.vault_integrations.oidc_auth` | Configures Vault to accept OIDC tokens from an IdP |

## When to use this example

- You are establishing a secrets platform for the first time.
- You want operators and service accounts to authenticate to Vault via your existing IdP
  (rather than managing Vault-local users).
- You want a base Vault instance that other platform examples (`platform-ci`) can later
  connect to as a secret backend.

## Adapting for your organisation

| What to change | Where |
|---|---|
| Vault image version | `group_vars/all.yml` → `vault_server_image` |
| API port | `group_vars/all.yml` → `vault_server_api_port` |
| Cluster name | `group_vars/all.yml` → `vault_server_cluster_name` |
| OIDC discovery endpoint | `group_vars/all.yml` → `vault_integration_oidc_endpoint` |
| OIDC client ID | `group_vars/all.yml` → `vault_integration_oidc_client_id` |
| Target hosts | `inventory/hosts.yml` |
| OIDC client secret | Environment variable `VAULT_OIDC_CLIENT_SECRET` (or Vault lookup once bootstrapped) |

## Secret handling

The OIDC client secret is injected from the environment variable `VAULT_OIDC_CLIENT_SECRET`
in `pre_tasks`. Because this platform *is* Vault, it cannot bootstrap secrets from Vault
during its own first deployment — the env-var approach is correct for the identity platform.
Subsequent platforms deployed after this one can use `community.hashi_vault` lookups.

## Deployment

```bash
ansible-galaxy collection install -r requirements.yml
export VAULT_OIDC_CLIENT_SECRET="<your-oidc-client-secret>"
ansible-playbook -i inventory/hosts.yml site.yml
```

## Files

- [`site.yml`](site.yml) — composition playbook
- [`requirements.yml`](requirements.yml) — pinned collection sources
- [`group_vars/all.yml`](group_vars/all.yml) — configuration
- [`inventory/hosts.yml`](inventory/hosts.yml) — target host(s)
