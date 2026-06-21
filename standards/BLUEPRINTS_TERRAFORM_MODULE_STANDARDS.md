# Terraform Module Standard

Status: Draft
Date: 2026-06-20
Owner: ICS Platform Engineering

## Purpose

Define how shared, org-neutral Terraform **modules** in iac-foundry are structured,
so they are predictable, reusable across all consumers (Vernify, IOTel, Saicom), and
hold no organisation-specific assumptions. Mirrors the Ansible collection/role standards
for the Terraform layer.

This standard covers reusable **modules** (e.g. `terraform-proxmox-vm`,
`terraform-tfe-workspace`). It does **not** cover consumer root configurations — those
live in the consuming org's repos and may be concrete.

## Repository

- Name follows `REPOSITORY_NAMING_STANDARD.md`: `terraform-<provider>-<module>`.
- One module per repository (the repo root *is* the module).
- Ships: `LICENSE`, `README.md`, `AGENTS.md`, `CODEOWNERS`, `.github/dependabot.yml`
  (with the `terraform` ecosystem enabled), `.gitignore`.

## Module layout

| File | Purpose |
|---|---|
| `versions.tf` | `required_version` (permissive lower bound, e.g. `>= 1.9`) + `required_providers` with a pinned **lower bound**. No `provider` blocks. |
| `variables.tf` | All inputs. Org-specific values (names, nodes, IPs) have **no defaults**. |
| `main.tf` | Resources/data sources. |
| `outputs.tf` | Outputs consumers need (IDs, addresses). |
| `examples/<name>/` | At least one minimal caller, used for `terraform validate`. |

## Rules

1. **No organisation-specific content.** No hostnames, IPs, domains, org names, or secret
   paths in code, defaults, examples, or comments. Use generic placeholders
   (`your-node`, `your-org`, `192.0.2.0/24` per RFC 5737).
2. **No `provider` configuration in the module.** Declare provider *requirements* only;
   the consumer configures the provider and supplies credentials (from the environment or
   a credentials file) — never as module inputs. Aligns with `BLUEPRINTS_SECRET_CONSUMPTION.md`.
3. **Org-specific inputs are required (no default); generic conventions may default.**
   `vm_name`/`node`/`organization` have no defaults; a Proxmox `vmbr0` bridge or a
   `local-lvm` datastore may default.
4. **No state in git.** `.gitignore` must exclude `*.tfstate*`, `*.tfvars`, `.terraform/`,
   and `.env`. State lives in TFC at the consumer layer.
5. **Sensitive variables are marked `sensitive = true`** and never written to logs/outputs
   in plaintext.
6. **Validated through pinned, containerised tooling.** Each module repo ships a
   `Dockerfile` + `docker-compose.yml` pinning the Terraform toolchain; `fmt`/`validate`
   (and any tests) run through the container so execution does not depend on locally
   installed tools. Bump pins deliberately; never float.
7. **Versioned releases.** Consumers pin modules by git tag (`?ref=vX.Y.Z`), never a
   floating branch. Follow `BLUEPRINTS_RELEASE_STRATEGY.md`.

## Validation (minimum bar for a PR)

```bash
docker compose run --rm terraform fmt -recursive -check
docker compose run --rm terraform -chdir=examples/basic init -backend=false
docker compose run --rm terraform -chdir=examples/basic validate
```

## Consumer expectations (informative)

Consumer root configs (in the org repos) hold the concrete values, the `cloud {}`/backend
block, the `provider` configuration, and resolve secrets at their layer. They reference
modules by pinned tag. This keeps the module reusable and the org specifics downstream —
the same one-way `framework → consumer` rule the rest of the framework follows.
