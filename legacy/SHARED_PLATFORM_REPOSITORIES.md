# Shared Platform Repositories

Status: Draft
Date: 2026-06-01
Owner: ICS Platform Engineering

## Purpose

Define the repositories to be created for the shared-platform program.

This inventory supports:

- ICS-owned product implementation and curation.
- Technology-by-technology migration for IOTel and Saicom.
- Clear ownership boundaries between shared products and consumer repos.

## Naming Convention

Use a tool-first pattern so repository purpose is obvious at a glance:

- <tool>-<artifact>-<name>

Use these approved forms:

- Ansible collections: ansible-collection-<name>
- Terraform modules: terraform-<provider>-<module>
- Packer templates: packer-template-<name>

Why this pattern:

- Matches Terraform module expectations.
- Keeps naming consistent across Ansible, Terraform, and Packer.
- Makes ownership and intent clear in search results and automation.

Ansible examples:

- ansible-collection-vault
- ansible-collection-grafana

Use this naming pattern for Terraform module repositories:

- terraform-<provider>-<module>

Examples:

- terraform-vsphere-vm
- terraform-tfe-workspace

Use this naming pattern for shared platform supporting repositories:

- platform-<capability>

Examples:

- platform-ci-policy
- platform-release-catalog

## Repositories To Create

## 1. Shared Ansible Collection Repositories

| Repo | Collection Namespace | Primary Scope | Priority |
|---|---|---|---|
| ansible-collection-common | blueprint.common | host baseline, shared helper roles, common guardrails | P1 |
| ansible-collection-vault | blueprint.vault | Vault server and client lifecycle, secret-consumption standards | P0 |
| ansible-collection-jenkins | blueprint.jenkins | Jenkins controller and agent lifecycle, plugin/JCasC baseline | P1 |
| ansible-collection-terraform-agents | blueprint.terraform_agents | Terraform agent lifecycle and scaling | P2 |
| ansible-collection-twingate | blueprint.twingate | connector lifecycle and registration | P3 |
| ansible-collection-graylog | blueprint.graylog | Graylog lifecycle, inputs, streams, pipelines | P2 |
| ansible-collection-grafana | blueprint.grafana | Grafana lifecycle, datasources, dashboards, auth integration | P2 |
| ansible-collection-graphite | blueprint.graphite | Graphite lifecycle and retention management | P3 |
| ansible-collection-openvas | blueprint.openvas | scanner lifecycle and scheduling | P4 |
| ansible-collection-openscap | blueprint.openscap | compliance scan lifecycle and reporting | P4 |

## 2. Shared Terraform Module Repositories

| Repo | Module Scope | Primary Consumers | Priority |
|---|---|---|---|
| terraform-vsphere-vm | canonical VM provisioning module for vSphere | IOTel, Saicom | P1 |
| terraform-vsphere-networking | standard network constructs and interfaces | IOTel, Saicom | P2 |
| terraform-vsphere-platform-storage | disks, volume attachment, baseline storage patterns | IOTel, Saicom | P2 |
| terraform-tfe-workspace | Terraform Cloud workspace and variable-set contracts | IOTel, Saicom, ICS | P1 |
| terraform-vsphere-vault-bootstrap | bootstrap contracts for Vault-aware infrastructure | IOTel, Saicom | P1 |

## 3. Shared Packer Template Repositories

| Repo | Template Scope | Primary Consumers | Priority |
|---|---|---|---|
| packer-template-ubuntu-base | baseline Ubuntu template with hardened defaults | IOTel, Saicom | P1 |
| packer-template-ubuntu-platform | platform-ready Ubuntu template (agent/runtime prereqs) | IOTel, Saicom | P2 |
| packer-template-image-manifest | manifest generation and publication for deterministic image selection | IOTel, Saicom, ICS | P2 |

## 4. Shared CI/CD And Release Repositories

| Repo | Scope | Priority |
|---|---|---|
| platform-jenkins-shared-lib | shared pipeline steps and policy-aligned wrappers | P1 |
| platform-ci-policy | lint and policy checks for Ansible, Terraform, and secret usage patterns | P1 |
| platform-release-catalog | curated release metadata, compatibility matrix, deprecation state | P2 |

## 5. Shared Documentation And Standards Repositories

| Repo | Scope | Priority |
|---|---|---|
| platform-standards | standards, operating model, security and release rules | P1 |
| platform-runbooks | migration, rollback, break-glass, and incident runbooks | P1 |

## 6. Consumer Platform Repositories

These remain separate per company and consume ICS-curated releases.

| Repo | Company | Scope |
|---|---|---|
| iotel-platform | IOTel | inventory, environment vars, orchestration only |
| saicom-platform | Saicom | inventory, environment vars, orchestration only |

## Recommended Creation Order

1. ansible-collection-vault
2. ansible-collection-common
3. platform-jenkins-shared-lib
4. platform-ci-policy
5. terraform-vsphere-vm
6. terraform-tfe-workspace
7. ansible-collection-jenkins
8. ansible-collection-terraform-agents
9. ansible-collection-graylog
10. ansible-collection-grafana
11. ansible-collection-graphite
12. ansible-collection-twingate
13. ansible-collection-openvas
14. ansible-collection-openscap
15. platform-release-catalog
16. platform-standards
17. platform-runbooks

## Notes

1. Vault-first remains mandatory due to current blocker status and downstream dependency chain.
2. Keep shared repos product-generic; company-specific behavior belongs only in consumer inventories and variables.
3. Do not migrate a later technology repo before upstream dependency repos are operating with passing safety gates.
