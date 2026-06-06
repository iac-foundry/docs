# Shared Platform Collections Strategy

Status: Draft Standard
Date: 2026-06-01
Owner: ICS Platform Engineering

## Problem Statement

The organization consists of multiple independent operating companies and an ICS environment providing development and testing capabilities.

Current constraints:

- Subsidiaries do not permit shared infrastructure.
- Technology stacks are intentionally similar.
- Platform teams are small.
- Operational knowledge must be reused where possible.
- Long-term divergence creates operational risk.
- Security and reliability improvements should propagate consistently.

Core platform technologies include:

- Jenkins
- Terraform Cloud
- Terraform Agents
- HashiCorp Vault
- Ansible
- Graylog
- Grafana
- Graphite
- Docker
- Twingate
- OpenVAS
- OpenSCAP

### Risks of Duplication

Maintaining separate roles and automation code per organization leads to:

- Configuration drift
- Security fixes applied inconsistently
- Increased maintenance burden
- Multiple incompatible implementations
- Reduced confidence in automation
- Higher onboarding and support costs

### Architectural Goal

Create reusable platform products that:

1. Are developed once.
2. Are consumed by multiple organizations.
3. Can be versioned independently.
4. Minimize upgrade blast radius.
5. Support full service lifecycle management.

---

# Proposed Architecture

## Operating Ownership

ICS owns the shared collection products end-to-end:

- Design and implementation.
- Release curation and versioning.
- Compatibility and regression testing.
- Documentation and support model.

IOTel and Saicom consume curated releases from ICS.

- No permanent company forks of shared collections.
- Company-specific behavior remains in each consumer repo inventory and variables.
- Runtime infrastructure remains separate per company.

## Guiding Principles

### 1. One Codebase, Multiple Consumers

Develop shared collections once and consume them from:

- ICS
- IOTel
- Saicom

The ICS environment acts as:

- Development environment
- Integration environment
- Canary environment
- Architecture proving ground

The ICS environment is **not** the source of truth.

The collection repositories are the source of truth.

ICS is the maintainer and release authority for those repositories.

---

### 2. Collections Follow Products

Collections should map to technology products rather than organizational functions.

Good:

```text
blueprint.vault
blueprint.jenkins
blueprint.graylog
blueprint.grafana
```

Avoid:

```text
blueprint.monitoring
blueprint.security
blueprint.build
```

Reason:

Product-based collections allow independent versioning and upgrades.

---

### 3. Independent Lifecycles

Each technology evolves independently.

Example:

```text
blueprint.vault      3.1.0
blueprint.jenkins    5.4.2
blueprint.graylog    2.8.0
```

A Vault upgrade should never force a Jenkins upgrade.

---

## Target Collection Layout

```text
blueprint.common

blueprint.vault
blueprint.jenkins
blueprint.terraform
blueprint.terraform_agents
blueprint.twingate

blueprint.graylog
blueprint.grafana
blueprint.graphite

blueprint.openvas
blueprint.openscap
```

---

# Collection Responsibilities

## blueprint.common

Purpose:

Shared platform functionality used by many collections.

Example contents:

```text
roles/
├── linux_baseline
├── docker_host
├── certificate_management
├── backup_client
├── monitoring_client
└── logging_client
```

Example usage:

```yaml
collections:
  - blueprint.common
```

---

## blueprint.vault

Purpose:

Complete Vault lifecycle management.

Example contents:

```text
roles/
├── server
├── agent
├── backup
├── restore
├── upgrade
└── monitoring
```

Capabilities:

- Installation
- Configuration
- HA clustering
- Backups
- Recovery
- Monitoring
- Hardening

---

## blueprint.jenkins

Purpose:

Jenkins platform lifecycle management.

Example contents:

```text
roles/
├── controller
├── agent
├── backup
├── restore
└── plugins
```

Capabilities:

- Controller deployment
- Agent deployment
- Plugin management
- Backup and recovery
- Monitoring

---

## blueprint.terraform

Purpose:

Terraform Cloud integration and operations.

Example contents:

```text
roles/
├── cli
├── workspace_management
├── policy_management
└── credentials
```

Capabilities:

- Terraform installation
- Workspace provisioning
- Policy management
- Credential integration

---

## blueprint.terraform_agents

Purpose:

Terraform Agent deployment and operations.

Example contents:

```text
roles/
├── install
├── upgrade
├── monitoring
└── scaling
```

---

## blueprint.twingate

Purpose:

Twingate platform automation.

Example contents:

```text
roles/
├── connector
├── upgrade
├── monitoring
└── registration
```

---

## blueprint.graylog

Purpose:

Graylog service lifecycle management.

Example contents:

```text
roles/
├── server
├── inputs
├── pipelines
├── dashboards
└── backup
```

Capabilities:

- Cluster deployment
- Input management
- Pipeline management
- Dashboard provisioning

---

## blueprint.grafana

Purpose:

Grafana lifecycle management.

Example contents:

```text
roles/
├── server
├── datasources
├── dashboards
└── auth
```

Capabilities:

- Grafana deployment
- Dashboard provisioning
- Datasource management
- Authentication integration

---

## blueprint.graphite

Purpose:

Metrics platform lifecycle management.

Example contents:

```text
roles/
├── server
├── relay
├── retention
└── backup
```

---

## blueprint.openvas

Purpose:

Vulnerability management automation.

Example contents:

```text
roles/
├── scanner
├── schedules
├── reports
└── integrations
```

---

## blueprint.openscap

Purpose:

Compliance and hardening validation.

Example contents:

```text
roles/
├── scan
├── remediation
├── reporting
└── baselines
```

---

# Consumer Pattern

Each organization maintains its own platform repository.

Example:

```text
iotel-platform
saicom-platform
ics-platform
```

These repositories contain:

- Inventory
- Variables
- Environment-specific configuration
- CI/CD pipelines

They do not contain service implementation logic.

They consume pinned, ICS-curated collection releases.

Example:

```yaml
collections:
  - blueprint.vault
  - blueprint.graylog
  - blueprint.grafana
```

---

# Recommended Development Workflow

## Stage 1

ICS develops feature in collection repository.

Example:

```text
blueprint.vault
```

---

## Stage 2

Deploy to ICS environment.

Validate:

- Installation
- Upgrade
- Recovery
- Monitoring

---

## Stage 3

ICS releases version.

Example:

```text
blueprint.vault 3.2.0
```

---

## Stage 4

IOTel consumes curated release.

---

## Stage 5

Saicom consumes curated release.

## Stage 6

ICS retires superseded versions after a controlled adoption window.

---

# Expected Benefits

## Technical

- Reduced duplication
- Consistent operational patterns
- Reduced drift
- Independent upgrades
- Smaller blast radius

## Operational

- Faster onboarding
- Easier support
- Shared operational knowledge
- Predictable platform lifecycle management

## Strategic

Treat automation as platform products rather than collections of roles.

Each collection becomes an internally supported product with:

- Ownership
- Documentation
- Testing
- Release management
- Lifecycle support

This creates a sustainable platform engineering model across all subsidiaries and the ICS environment.

---

# Migration Planning Note

This document defines the operating standard.

- It is not an IOTel roadmap document.
- It is a Mosaic company roadmap standard for customer-serving platform delivery.
- Safe migration sequencing is maintained in:

  - [Shared Collections Migration Order](./SHARED_COLLECTIONS_MIGRATION_ORDER.md)
