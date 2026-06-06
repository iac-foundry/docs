# Example Consumer Platforms

These are reference implementations showing how the `blueprints.*` collections are
composed into complete, deployable platforms. They are not production-ready as-is — they
use `example.internal` hostnames and stub inventory — but they are the canonical starting
point for an organisation adopting the framework.

## How to use an example

1. Copy the example directory into your organisation's platform repo.
2. Replace `example.internal` URLs and image versions with your actual values.
3. Fill in `inventory/hosts.yml` with real hostnames.
4. Adjust `group_vars/all.yml` to match your deployment (ports, data dirs, image tags).
5. Ensure your CI environment provides any secrets referenced as environment variables.
6. Run `ansible-galaxy collection install -r requirements.yml` then `ansible-playbook site.yml`.

## The composition rule

Composition always happens in `site.yml`, never inside a collection. The collections
provide building blocks; the platform decides which blocks to combine and in what order.
See [../design/BLUEPRINTS_DESIGN_PRINCIPLES.md](../design/BLUEPRINTS_DESIGN_PRINCIPLES.md) §5.

## Available examples

| Example | What it assembles | Collections used |
|---|---|---|
| [platform-ci/](platform-ci/) | Jenkins controller + Job DSL + Vault auth + Authentik SSO | `blueprints.jenkins` · `blueprints.jenkins_integrations` |
| [platform-monitoring/](platform-monitoring/) | Graylog log management (standalone, no auth backend required) | `blueprints.graylog` |
| [platform-identity/](platform-identity/) | HashiCorp Vault + OIDC auth backend against any IdP | `blueprints.vault` · `blueprints.vault_integrations` |
