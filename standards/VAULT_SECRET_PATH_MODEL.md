# Canonical Vault Secret Path Model

**Status:** Draft
**Date:** 28 June 2026
**Owner:** Platform Engineering
**Context:** Standardized KV secret organization for Vernify + future org-neutral reuse (IOTel, Saicom)

---

## Executive Summary

This document defines the canonical path structure for secrets stored in HashiCorp Vault KV v2 backend across Vernify and reusable for other organizations.

**Path structure:** `kv/<domain>/<system>/<secret_name>`

**Domains:**
- `platform`: Internal platform services (Vault, Jenkins, step-ca, Proxmox agent, agent-specific)
- `applications`: Business applications (MyApp, customer-specific services)
- `saas`: External third-party vendors (TFC, Cloudflare, etc.)

**Benefits:**
- Predictable, hierarchical organization
- Clear separation of concerns (platform vs. app vs. external)
- Enables role-based access control (RBAC) by domain
- Reusable across organizations with minimal customization
- Required metadata enforces governance (owner, usage type, rotation policy)

---

## Path Structure

### Format

```
kv/<domain>/<system>/<secret_name>
```

### Components

| Component | Purpose | Example |
|---|---|---|
| `<domain>` | Broad category: `platform`, `applications`, `saas` | `platform` |
| `<system>` | Service or system name (kebab-case) | `jenkins`, `step-ca`, `proxmox` |
| `<secret_name>` | Specific secret identifier (kebab-case, role-based) | `admin-token`, `approle`, `api-key` |

### Examples

```
kv/platform/step-ca/root-ca-cert
kv/platform/vault/admin-password
kv/platform/jenkins/admin-token
kv/platform/jenkins/approle
kv/platform/agent01/approle
kv/platform/proxmox/api-token
kv/saas/tfc/team-token
kv/saas/cloudflare/api-token
kv/applications/myapp/database-password
```

---

## Domain Definitions

### `platform`

**Purpose:** Internal platform services; infrastructure and CI/CD machinery.

**Includes:**
- PKI (step-ca, certificates, provisioner credentials)
- Vault admin credentials, unseal keys metadata, root tokens
- Jenkins (admin token, AppRole credentials, automation tokens)
- Consul, etcd, or other platform backends
- Agent-specific secrets (AppRole for build agents, deploy agents)
- Proxmox management credentials
- DNS, monitoring, logging infrastructure

**Scope:** Operational team only; not accessible to business applications (unless explicit override).

**Example paths:**
```
kv/platform/step-ca/root-ca-cert
kv/platform/step-ca/provisioner-password
kv/platform/vault/tls-cert
kv/platform/vault/tls-key
kv/platform/vault/admin-password (for break-glass only)
kv/platform/jenkins/admin-token
kv/platform/jenkins/approle
kv/platform/jenkins/webhook-secret
kv/platform/agent01/approle
kv/platform/agent02/approle
kv/platform/proxmox/api-token
kv/platform/proxmox/root-password (for initial setup)
```

---

### `applications`

**Purpose:** Business application secrets; organization-specific or customer-specific credentials.

**Includes:**
- Database passwords (shared databases, not service-specific)
- OAuth/API credentials for external SaaS integrations used by apps
- Application configuration secrets (API keys, encryption keys)
- Customer-specific deployment credentials
- License keys

**Scope:** Application teams; may include business owners (depending on RBAC policies).

**Example paths:**
```
kv/applications/myapp/database-password
kv/applications/myapp/oauth-client-secret
kv/applications/myapp/encryption-key
kv/applications/myapp-prod/slack-webhook-url
kv/applications/customer-abc/api-key
```

---

### `saas`

**Purpose:** External third-party vendor credentials and API tokens.

**Includes:**
- Terraform Cloud / Terraform Enterprise tokens
- GitHub / GitLab tokens
- Cloudflare API tokens
- HashiCorp Cloud Platform (HCP) credentials
- AWS / Azure / GCP service account keys
- Monitoring SaaS (Datadog, NewRelic, etc.)
- Communication SaaS (Slack, PagerDuty, etc.)

**Scope:** Infrastructure/DevOps team primarily; shared with applications as needed.

**Example paths:**
```
kv/saas/tfc/team-token
kv/saas/github/personal-access-token
kv/saas/cloudflare/api-token
kv/saas/aws/service-account-key
kv/saas/slack/webhook-url
kv/saas/pagerduty/integration-key
```

---

## Naming Rules

### General Rules

1. **Lowercase:** All path components must be lowercase.
2. **Kebab-case (new names):** Use hyphens to separate words in new secret names.
   - Good: `admin-token`, `api-key`, `root-ca-cert`
   - Bad: `admin_token`, `adminToken`, `Admin-Token`

3. **Underscores preserved (legacy):** If migrating from existing inventory with underscores, preserve them for backward compatibility.
   - Example: `kv/platform/infra_orch_01/approle` (keeps `infra_orch_01` from Ansible inventory)

4. **Explicit, role-based names:** Secret names should indicate the role/principal accessing them.
   - Good: `agent-token`, `admin-user`, `deploy-key`, `read-only-token`
   - Bad: `token`, `credentials`, `secret`

5. **Don't mix principals:** Each principal (user, service, agent) gets its own secret path.
   - Good: `kv/platform/jenkins/admin-token` + `kv/platform/jenkins/deployment-token`
   - Bad: `kv/platform/jenkins/credentials` (ambiguous who it's for)

6. **Split user vs. machine credentials:** If both humans and machines need access, create separate paths.
   - Paths: `kv/platform/step-ca/provisioner-password` (for services) + `kv/platform/step-ca/admin-user` (for operators)

7. **Group PKI artifacts by service:** Certificates, keys, and CA materials for a service stay in the same path family.
   - Paths:
     ```
     kv/platform/step-ca/root-ca-cert
     kv/platform/step-ca/provisioner-password
     kv/platform/jenkins/tls-cert
     kv/platform/jenkins/tls-key
     ```

8. **Keep bootstrap/recovery material separate:** One-time bootstrap secrets (like Vault root token, unseal keys) are NOT stored in KV after Phase 3.
   - Unseal keys: operator-backed-up externally (not in Vault, not in KV)
   - Root token: deleted after secret seeding (temporary, not persisted)
   - Rationale: These enable full system compromise if leaked; kept offline.

---

## Required Metadata

All secrets stored in KV must include metadata fields (within the secret data):

```hcl
kv/platform/jenkins/admin-token
├── value: "11e9b8f7c3d2e1a0c9b8f7e6d5c4b3a2"
├── owner: "platform-team"  # Required: owner team/service
├── usage: "machine"         # Required: "machine" or "user"
├── managed_by: "automation" # Required: "automation" or "manual"
└── rotation_period: "30d"   # Required: "30d", "90d", "manual", "never"
```

### Metadata Fields

| Field | Required | Values | Purpose |
|---|---|---|---|
| `owner` | Yes | Team/service name (e.g., "platform-team", "jenkins-ci") | Accountability; for audit + access control |
| `usage` | Yes | `machine` or `user` | Machine: service/agent; User: human operator |
| `managed_by` | Yes | `automation` or `manual` | Automation: bootstrap script; Manual: operator-managed |
| `rotation_period` | Yes | `30d`, `90d`, `manual`, `never` | Rotation cadence for compliance |
| `description` | No | Free text | Human-readable explanation of what this secret is for |
| `last_rotated` | No | ISO8601 datetime | When this secret was last rotated (for audit) |

### Examples

**Jenkins AppRole (automation-managed, machine, needs rotation):**
```json
{
  "role_id": "11e9b8f7c3d2e1a0c...",
  "secret_id": "22f8c7d6e5f4g3h2i...",
  "owner": "jenkins-ci",
  "usage": "machine",
  "managed_by": "automation",
  "rotation_period": "30d",
  "description": "Jenkins AppRole for Vault auth (Phase 4+)",
  "last_rotated": "2026-06-28T10:00:00Z"
}
```

**Proxmox API Token (manual, machine, long-lived):**
```json
{
  "value": "proxmox!tokenid=11e9b8f7c3d2e1a0",
  "owner": "infrastructure-team",
  "usage": "machine",
  "managed_by": "manual",
  "rotation_period": "90d",
  "description": "Proxmox API token for Terraform provisioning",
  "last_rotated": "2026-04-15T14:30:00Z"
}
```

---

## Cross-Domain Examples

### Scenario 1: Jenkins Secret Access Flow

**Setup Phase:**
1. Bootstrap script creates: `kv/platform/jenkins/approle` (role_id + secret_id)
2. Jenkins retrievals AppRole credentials from Vault
3. Jenkins uses AppRole to authenticate to Vault

**Runtime:**
1. Jenkins job: `kv/saas/tfc/team-token` (for Terraform Cloud API)
2. Jenkins job: `kv/saas/proxmox/api-token` (for Packer provisioning)
3. Jenkins job: `kv/platform/step-ca/provisioner-password` (for certificate issuance)

**Paths:**
```
kv/platform/jenkins/approle          ← Jenkins auth credentials
kv/platform/jenkins/admin-token      ← Jenkins operator break-glass
kv/saas/tfc/team-token               ← Jenkins job: Terraform Cloud
kv/saas/proxmox/api-token            ← Jenkins job: Packer/Proxmox
kv/platform/step-ca/provisioner-password ← Jenkins job: CA cert requests
```

---

### Scenario 2: Agent Secret Access Flow

**Setup Phase:**
1. Bootstrap script creates: `kv/platform/agent01/approle` (role_id + secret_id)
2. Vault agent on agent01 retrieves AppRole credentials
3. Vault agent authenticates to Vault; auto-renews token

**Runtime:**
1. Vault agent: Pulls `kv/platform/proxmox/api-token` (cached locally on agent)
2. Build job: Packer uses cached Proxmox token from Vault agent
3. Build job: Terraform uses TFC token (pulled by Vault agent)

**Paths:**
```
kv/platform/agent01/approle          ← Agent auth credentials
kv/platform/proxmox/api-token        ← Agent: Packer (provisioning)
kv/saas/tfc/team-token               ← Agent: Terraform (deployment)
```

---

### Scenario 3: Multi-Customer Application

**Setup Phase (per customer):**
1. Each customer has app-specific secrets (database, API keys)
2. App auth credentials live in `applications` domain

**Paths:**
```
kv/applications/customer-abc/database-password
kv/applications/customer-abc/api-encryption-key
kv/applications/customer-xyz/database-password
kv/applications/customer-xyz/api-encryption-key
kv/saas/aws/service-account-key      ← Shared SaaS credential (all customers)
```

---

## Access Control (RBAC)

### Vault Policies by Domain

**Platform Policy (Ops team):**
```hcl
path "secret/data/platform/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
path "secret/data/saas/*" {
  capabilities = ["read", "list"]  # Read only for app consumption
}
path "secret/data/applications/*" {
  capabilities = ["read", "list"]   # Read only; app teams manage their own
}
```

**Application Team Policy (MyApp team):**
```hcl
path "secret/data/applications/myapp/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
path "secret/data/saas/*" {
  capabilities = ["read", "list"]  # Read shared SaaS credentials
}
```

**Jenkins AppRole Policy:**
```hcl
path "secret/data/saas/*" {
  capabilities = ["read", "list"]
}
path "secret/data/platform/step-ca/*" {
  capabilities = ["read", "list"]
}
path "secret/data/platform/proxmox/*" {
  capabilities = ["read", "list"]
}
```

---

## Implementation in Vernify Phase 3-5

### Bootstrap Script Seeding

The unified bootstrap script (see `BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md`, Section 4.5.3) seeds all secrets during Phase 3:

```bash
# Phase 3: Vault seeding (idempotent)
vault kv put secret/platform/step-ca/root-ca-cert \
  value="$(cat /opt/step-ca/certs/root_ca.crt)" \
  owner="platform-team" \
  usage="machine" \
  managed_by="automation" \
  rotation_period="never"

vault kv put secret/platform/vault/tls-cert \
  value="$(cat /etc/ssl/certs/vault.sec01.internal.crt)" \
  owner="platform-team" \
  usage="machine" \
  managed_by="automation" \
  rotation_period="365d"

vault kv put secret/saas/proxmox/api-token \
  value="$PROXMOX_TOKEN" \
  owner="infrastructure-team" \
  usage="machine" \
  managed_by="automation" \
  rotation_period="90d"

# ... repeat for all secrets
```

### Jenkins Integration

Phase 4: Jenkins retrieves secrets by role-based path:

```groovy
// In Job DSL or Jenkins Pipeline
job('packer-image-build') {
  steps {
    shell('''
      PROXMOX_TOKEN=$(vault kv get -field=value secret/saas/proxmox/api-token)
      packer build packer/template.pkr.hcl
    ''')
  }
}
```

### Agent Integration

Phase 5: Vault agent auto-fetches and caches secrets:

```hcl
# vault-agent.hcl on agent01
vault {
  path = "secret/data/saas/proxmox/api-token"
  command = "systemctl restart packer-service"
}

vault {
  path = "secret/data/saas/tfc/team-token"
  command = "systemctl restart terraform-service"
}
```

---

## Future Scalability

### Multi-Team Expansion (Phase 6+)

As Vernify grows, `applications` domain scales with customer/team namespacing:

```
kv/applications/team-a/myapp/database-password
kv/applications/team-a/myapp/api-key
kv/applications/team-b/other-app/database-password
```

### Multi-Environment Separation (Phase 6+)

Environments can be segregated by Vault mount or by path family:

```
# Option A: Separate Vault mounts
secret-prod/platform/jenkins/admin-token
secret-staging/platform/jenkins/admin-token

# Option B: Same mount, path family
secret/platform/jenkins-prod/admin-token
secret/platform/jenkins-staging/admin-token
```

**Recommended:** Use separate Vault mounts (`secret-prod`, `secret-staging`) for stronger isolation + separate backup/restore.

---

## Migration & Backward Compatibility

### Legacy Names

If migrating from existing systems with different naming conventions (e.g., Ansible inventory with underscores):

1. **Keep legacy paths alive** for 1-2 phases (Phase 4-5)
2. **Add aliases** for new canonical paths
3. **Deprecate old paths** with operator notice
4. **Example:**
   ```
   OLD: secret/infra_orch_01/api_token (Ansible inventory style)
   NEW: secret/platform/infra-orch-01/api-token (canonical style)
   ALIAS: Create both paths; deprecate old after 2 phases
   ```

---

## Audit & Compliance

### Vault Audit Log

All secret operations are logged:

```json
{
  "type": "response",
  "auth": { "metadata": { "role_id": "..." } },
  "request": { "operation": "read", "path": "secret/data/platform/jenkins/admin-token" },
  "response": { "auth": { "token_ttl": 3600 } },
  "time": "2026-06-28T10:00:00Z"
}
```

### Policy Compliance

**Required for all orgs:**
- All production secrets in `platform` and `saas` require rotation policy
- Metadata fields are mandatory (enforced by bootstrap script validation)
- No secrets in comments, logs, or code (checked by pipeline lint)

---

## References

- **Vernify Phase 3-5 Architecture:** See `BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md`, Section 4.5.3 (Bootstrap Script Model)
- **Vault KV Documentation:** https://www.vaultproject.io/docs/secrets/kv/kv-v2
- **Vault Audit Logging:** https://www.vaultproject.io/docs/audit
- **HashiCorp Vault Best Practices:** https://learn.hashicorp.com/tutorials/vault/secrets-engines

---

## Appendix: Complete Secret Inventory (Vernify Phase 3-5)

| Path | Usage | Owner | Rotation | Description |
|---|---|---|---|---|
| `kv/platform/step-ca/root-ca-cert` | machine | platform-team | never | Root CA certificate (self-signed, valid ~10 years) |
| `kv/platform/step-ca/provisioner-password` | machine | platform-team | 90d | step-ca provisioner password (for TLS issuance) |
| `kv/platform/vault/tls-cert` | machine | platform-team | 30d | Vault TLS certificate (issued by step-ca) |
| `kv/platform/vault/tls-key` | machine | platform-team | 30d | Vault TLS private key |
| `kv/platform/jenkins/admin-token` | machine | jenkins-ci | 30d | Jenkins API admin token (for Job DSL + automation) |
| `kv/platform/jenkins/approle` | machine | jenkins-ci | 30d | Jenkins AppRole (role_id + secret_id) for Vault auth |
| `kv/platform/agent01/approle` | machine | build-agents | 30d | agent01 AppRole (role_id + secret_id) for Vault auth |
| `kv/saas/proxmox/api-token` | machine | infrastructure-team | 90d | Proxmox API token (for Terraform/Packer provisioning) |
| `kv/saas/tfc/team-token` | machine | infrastructure-team | 90d | Terraform Cloud team token (for state + plan execution) |

---

## Document Version History

| Version | Date | Author | Changes |
|---|---|---|---|
| 1.0 | 2026-06-28 | Platform Engineering | Initial canonical path model for Vernify Phase 3-5 |
