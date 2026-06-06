# Example: platform-monitoring

A standalone log-management platform composed from Graylog — the "Org B partial" example
showing that monitoring can be deployed independently with no dependency on Jenkins, Vault,
or any IdP.

## What this stands up

| Component | Role | Purpose |
|---|---|---|
| Graylog server | `blueprints.graylog.container_server` | Log collection and search, running in Docker |
| Input configuration | `blueprints.graylog.inputs` | Configures syslog UDP (and any other inputs) |
| Pipeline configuration | `blueprints.graylog.pipelines` | Applies processing rules (GeoIP enrichment etc.) |

## When to use this example

- You need centralised log collection and want a quick start.
- You do not yet have (or need) Jenkins or Vault — this platform has zero external dependencies.
- You want to add monitoring to an existing org environment incrementally.

## Adapting for your organisation

| What to change | Where |
|---|---|
| Graylog image version | `group_vars/all.yml` → `graylog_server_image` |
| HTTP bind address / port | `group_vars/all.yml` → `graylog_server_http_bind` |
| Syslog input port | `group_vars/all.yml` → `graylog_inputs_config.syslog_udp.port` |
| Additional inputs | `group_vars/all.yml` → `graylog_inputs_config` (add keys) |
| Pipeline rules | `group_vars/all.yml` → `graylog_pipelines_config` |
| Target hosts | `inventory/hosts.yml` |

## Secret handling

This example resolves `graylog_server_password_secret` from the environment variable
`GRAYLOG_PASSWORD_SECRET` at runtime in `pre_tasks`. In a production setup you would
replace this with a `community.hashi_vault` lookup (following the same pattern as
`platform-ci`) once a Vault instance is available.

## Deployment

```bash
ansible-galaxy collection install -r requirements.yml
export GRAYLOG_PASSWORD_SECRET="<your-secret>"
ansible-playbook -i inventory/hosts.yml site.yml
```

## Files

- [`site.yml`](site.yml) — composition playbook
- [`requirements.yml`](requirements.yml) — pinned collection sources
- [`group_vars/all.yml`](group_vars/all.yml) — configuration
- [`inventory/hosts.yml`](inventory/hosts.yml) — target host(s)
