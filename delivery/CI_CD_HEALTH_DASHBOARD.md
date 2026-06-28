# CI/CD Health Dashboard

## Overview

This dashboard tracks the health and status of all CI/CD pipelines for the Vernify Phase 3-5 infrastructure implementation. Each workflow runs automatically on push and pull requests to validate code quality, security, and infrastructure correctness.

---

## Workflow Status Summary

### Ansible Collections - Linting & Testing

| Collection | Ansible Lint | Molecule Tests | Security Scan | Status |
|---|---|---|---|---|
| **step-ca** | [View Runs](../../ansible-collection-step-ca/actions/workflows/ansible-lint.yml) | [View Runs](../../ansible-collection-step-ca/actions/workflows/molecule.yml) | [View Runs](../../ansible-collection-step-ca/actions/workflows/security-scan.yml) | 🟢 |
| **vault** | [View Runs](../../ansible-collection-vault/actions/workflows/ansible-lint.yml) | [View Runs](../../ansible-collection-vault/actions/workflows/molecule.yml) | [View Runs](../../ansible-collection-vault/actions/workflows/security-scan.yml) | 🟢 |
| **jenkins** | [View Runs](../../ansible-collection-jenkins/actions/workflows/ansible-lint.yml) | [View Runs](../../ansible-collection-jenkins/actions/workflows/molecule.yml) | [View Runs](../../ansible-collection-jenkins/actions/workflows/security-scan.yml) | 🟢 |
| **vault-integrations** | [View Runs](../../ansible-collection-vault-integrations/actions/workflows/ansible-lint.yml) | [View Runs](../../ansible-collection-vault-integrations/actions/workflows/molecule.yml) | [View Runs](../../ansible-collection-vault-integrations/actions/workflows/security-scan.yml) | 🟢 |
| **jenkins-integrations** | [View Runs](../../ansible-collection-jenkins-integrations/actions/workflows/ansible-lint.yml) | [View Runs](../../ansible-collection-jenkins-integrations/actions/workflows/molecule.yml) | [View Runs](../../ansible-collection-jenkins-integrations/actions/workflows/security-scan.yml) | 🟢 |

### Terraform Deployments - Validation & Security

| Deployment | Validate & Plan | Security Scan | Status |
|---|---|---|---|
| **terraform-sec01-deploy** | [View Runs](../../../terraform-sec01-deploy/actions/workflows/terraform-validate.yml) | [View Runs](../../../terraform-sec01-deploy/actions/workflows/security-scan.yml) | 🟢 |
| **terraform-agent01-deploy** | [View Runs](../../../terraform-agent01-deploy/actions/workflows/terraform-validate.yml) | [View Runs](../../../terraform-agent01-deploy/actions/workflows/security-scan.yml) | 🟢 |
| **terraform-build01-deploy** | [View Runs](../../../terraform-build01-deploy/actions/workflows/terraform-validate.yml) | [View Runs](../../../terraform-build01-deploy/actions/workflows/security-scan.yml) | 🟢 |

### Bootstrap Container - E2E Validation

| Script | Location | Status |
|---|---|---|
| **E2E Dry-Run** | [`scripts/e2e-test-dryrun.sh`](../../../bootstrap-container/scripts/e2e-test-dryrun.sh) | 🟢 Ready |

---

## Workflow Details

### E-01: Ansible Lint Workflow

**Purpose:** Validate Ansible role syntax and best practices across all blueprint collections.

**Trigger:** On push and pull requests to `main`/`develop` branches when role files change

**Steps:**
1. Checkout code
2. Run ansible-lint on `roles/` directory
3. Fail if lint violations found

**Expected Output:**
- All roles pass ansible-lint (no Critical/High violations)
- Workflow fails on violations, preventing merge

**Repository:** All 5 Ansible collections
- `ansible-collection-step-ca`
- `ansible-collection-vault`
- `ansible-collection-jenkins`
- `ansible-collection-vault-integrations`
- `ansible-collection-jenkins-integrations`

---

### E-02: Molecule Test Workflow

**Purpose:** Validate Ansible role functionality through automated testing with Docker.

**Trigger:** On push and pull requests when role files change

**Steps:**
1. Checkout code
2. Install Python 3.11
3. Install molecule + docker plugin
4. Run `molecule test` for each role in parallel (matrix strategy)
5. Upload test logs as artifacts

**Expected Output:**
- All roles converge (apply successfully)
- Idempotence tests pass (role is idempotent)
- Verification tests pass (role achieves desired state)
- Docker driver ensures reproducible testing

**Roles Tested:**

- **step-ca:** container_server, issue_certificate
- **vault:** client, container_server, init_and_unseal, systemd_server
- **jenkins:** agent, container_server, job_dsl, kubernetes_server, systemd_server
- **vault-integrations:** ldap_auth, oidc_auth
- **jenkins-integrations:** authentik_sso, github_oauth, gitlab_oauth, ldap_auth, vault_auth

**Local Testing:**
```bash
cd roles/<role>
pip install molecule molecule-plugins[docker] docker
molecule test
```

---

### E-03: Terraform Validate & Plan Workflow

**Purpose:** Validate Terraform syntax, formatting, and plan correctness across deployment repos.

**Trigger:** On push and pull requests when .tf or .tfvars files change

**Steps:**
1. Checkout code
2. Setup Terraform 1.15.6
3. `terraform fmt -check` - Enforces formatting standard
4. `terraform validate` - Validates syntax and schema
5. `terraform init -upgrade` - Initializes with latest provider versions
6. `terraform plan` - Generates and uploads execution plan

**Expected Output:**
- All .tf files follow HCL formatting standard
- Terraform syntax is valid
- All variables and outputs are correctly defined
- Plan succeeds (no provider/auth errors)

**Repositories:**
- `terraform-sec01-deploy`
- `terraform-agent01-deploy`
- `terraform-build01-deploy`

**Local Testing:**
```bash
cd <terraform-repo>
terraform fmt -check -recursive
terraform validate
terraform plan
```

---

### E-04: Security Scanning Workflow

**Purpose:** Automated security checks for infrastructure-as-code and Ansible playbooks.

**For Ansible Collections:**
- `ansible-playbook --syntax-check` - Validates YAML syntax
- `ansible-lint --strict` - Enforces security and best practice rules

**For Terraform Deployments:**
- `tfsec` - Security static analysis for Terraform
- Upload SARIF format to GitHub Security tab for visibility

**Trigger:** On push/PR and weekly schedule (Sunday 2 AM UTC)

**Expected Output:**
- No Critical security findings
- Medium/Low findings documented and acknowledged
- SARIF uploaded to GitHub Security dashboard

**Local Testing:**

Ansible:
```bash
ansible-playbook --syntax-check playbooks/*.yml
ansible-lint roles/ --strict
```

Terraform:
```bash
# Install tfsec
brew install tfsec

# Run security scan
tfsec .
```

---

### E-05: E2E Dry-Run Validation Script

**Purpose:** Validates all Phase 3-5 deployment prerequisites without actually provisioning infrastructure.

**Location:** `bootstrap-container/scripts/e2e-test-dryrun.sh`

**Validations:**
1. Environment variables check (PROXMOX_*, VAULT_*, ANSIBLE_* vars)
2. Packer templates exist and are accessible
3. LXC image configuration (when Proxmox is configured)
4. Terraform modules present and structure valid
5. Terraform syntax validation (terraform validate)
6. Ansible playbook syntax check
7. Ansible collection roles present
8. Vault configuration and CLI available

**Running Locally:**
```bash
cd bootstrap-container
./scripts/e2e-test-dryrun.sh          # Standard output
./scripts/e2e-test-dryrun.sh --verbose  # Detailed debug output
```

**Expected Output:**
```
E2E Dry-Run Validation
======================
✅ Environment variables check
✅ Packer templates check
✅ LXC image check
✅ Terraform modules check
✅ Terraform validate: terraform-sec01-deploy
✅ Terraform validate: terraform-agent01-deploy
✅ Terraform validate: terraform-build01-deploy
✅ Ansible playbook syntax
✅ Ansible collections check
✅ Vault configuration check

Overall: PASSED (ready for Phase D integration)
```

---

## Troubleshooting Guide

### Ansible Lint Failures

**Common Causes:**
- Undefined Jinja2 variables: Ensure all variables are defined in defaults/main.yml
- Missing handlers: Declare handlers for notified tasks
- Deprecated modules: Update to current module names
- Indentation errors: YAML must have consistent spacing

**Resolution:**
```bash
# Run ansible-lint locally to see detailed errors
cd roles/<role>
ansible-lint

# Fix issues based on output, then test:
molecule test
```

**References:**
- [Ansible Lint Rules](https://ansible.readthedocs.io/projects/lint/rules/)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/tips_tricks/index.html)

---

### Molecule Test Failures

**Common Causes:**
- Docker not running: Molecule tests use Docker driver
- Missing dependencies in converge.yml
- Role fails to converge (syntax/logic errors)
- Idempotence check fails (role has side effects)
- Verify.yml assertions fail

**Resolution:**
```bash
cd roles/<role>

# Run with verbose output
molecule test -vv

# Troubleshoot step-by-step:
molecule create     # Create test container
molecule converge   # Run role once
molecule idempotence  # Run role twice, verify idempotence
molecule verify     # Run verify.yml assertions
molecule destroy    # Cleanup
```

**Check molecule.yml for:**
- Docker image availability
- Port mappings correct
- Environment variables passed
- Volume mounts needed

---

### Terraform Validation Failures

**Common Causes:**
- Formatting inconsistent: `terraform fmt` applies formatting
- Syntax errors: Check .tf file YAML/HCL syntax
- Variable type mismatch: Ensure tfvars match variable types
- Provider configuration missing: Check backend and provider blocks
- Module not found: Verify module paths and versions

**Resolution:**
```bash
cd <terraform-repo>

# Auto-format all files
terraform fmt -recursive

# Show detailed errors
terraform validate

# Check specific variable types
terraform plan -out=plan.tfplan
terraform show plan.tfplan
```

**Common Variable Issues:**
```hcl
# ❌ WRONG: String vs list
variable "vpc_id" {
  type = string
  default = ["vpc-123"]  # ERROR: list, not string
}

# ✅ CORRECT:
variable "vpc_id" {
  type = string
  default = "vpc-123"
}
```

---

### Security Scan Findings

**Ansible Lint Security Rules:**

| Rule | Severity | Action |
|---|---|---|
| `risky-shell-pipe` | High | Avoid piping in shell module, use proper modules |
| `no-handler` | Medium | Notify handlers in tasks |
| `var-naming` | Low | Follow variable naming conventions |

**tfsec Security Rules:**

| Check | Severity | Example |
|---|---|---|
| `aws-s3-enable-versioning` | Medium | Enable S3 versioning for data protection |
| `aws-security-group-ingress` | High | Restrict ingress CIDR blocks (no 0.0.0.0/0) |
| `aws-iam-require-mfa` | Medium | Require MFA for IAM users |

**Resolution:**
1. Review finding in GitHub Security tab
2. Determine if applicable to deployment
3. If acceptable: Document in PR, file tracking issue
4. If not acceptable: Fix issue and re-run scan

---

### Running Workflows Locally

All workflows can be run locally for faster iteration:

**Ansible Lint:**
```bash
cd <collection>
ansible-lint roles/
```

**Molecule:**
```bash
cd <collection>/roles/<role>
molecule test
```

**Terraform Validation:**
```bash
cd <terraform-repo>
terraform fmt -check -recursive
terraform validate
terraform plan
```

**Security Scan:**
```bash
# Ansible
ansible-playbook --syntax-check playbooks/*.yml
ansible-lint roles/ --strict

# Terraform
tfsec .
```

**E2E Validation:**
```bash
cd bootstrap-container
./scripts/e2e-test-dryrun.sh --verbose
```

---

## CI/CD Pipeline Stages

```
GitHub Push/PR
    ↓
┌───────────────────────────┐
│ E-01: Ansible Lint (5 min) │
├───────────────────────────┤
│ Validates role syntax     │
│ Fails if Critical/High     │
└───────────────────────────┘
    ↓
┌───────────────────────────┐
│ E-02: Molecule (25 min)    │
├───────────────────────────┤
│ Docker-based role tests   │
│ Parallel by role          │
│ Fails if test fails       │
└───────────────────────────┘
    ↓
┌───────────────────────────┐
│ E-03: TF Validate (10 min) │
├───────────────────────────┤
│ Fmt + Validate + Plan     │
│ Fails if validation fails  │
└───────────────────────────┘
    ↓
┌───────────────────────────┐
│ E-04: Security Scan (5 min)│
├───────────────────────────┤
│ tfsec + Ansible-lint      │
│ Fails if Critical found    │
└───────────────────────────┘
    ↓
┌───────────────────────────┐
│ Status: All Checks Passed │
│ PR Ready for Merge         │
└───────────────────────────┘
```

---

## Performance & Metrics

### Workflow Run Times

| Workflow | Avg Duration | Max Parallel |
|---|---|---|
| Ansible Lint | 2-3 min | Per collection |
| Molecule Tests | 20-30 min | 5 roles (step-ca) to 5 roles (jenkins-int) |
| Terraform Validate | 5-10 min | 3 repos in parallel |
| Security Scan | 3-5 min | Sequential |
| E2E Dry-Run | 2-3 min | Sequential |

**Total PR Check Time:** ~30-40 minutes (parallel execution across repos)

### Optimization Tips

- **Molecule:** Use smaller Docker images to reduce pull time
- **Terraform:** Cache terraform init across runs (setup-terraform@v2 handles this)
- **Lint:** Run locally before push to catch issues early
- **Parallel:** All workflows run in parallel; collect results before merge decision

---

## Integration with Phase D

The Phase E workflows enable Phase D (Integration) to proceed with confidence:

1. **Code Quality:** Ansible Lint ensures all roles follow standards
2. **Functionality:** Molecule tests validate role behavior in Docker
3. **Infrastructure:** Terraform validation ensures deployment configs are sound
4. **Security:** Security scans identify CVEs and misconfigurations early
5. **Prerequisites:** E2E script validates all dependencies before provisioning

**Gate:** All workflows must pass before PR can be merged to main

---

## Maintenance & Updates

### Weekly Tasks

- Review security scan findings (Sunday 2 AM UTC)
- Update molecule/ansible/terraform versions if needed
- Monitor workflow run times for performance degradation

### Monthly Tasks

- Audit ansible-lint rule changes
- Update tfsec baseline (if findings are approved)
- Review test coverage for new roles

### Version Management

Current versions (as of Phase E implementation):
- `ansible-lint`: v6.x
- `molecule`: 4.x
- `molecule-plugins`: Docker driver
- `terraform`: 1.15.6
- `tfsec`: Latest (auto-updated in action)

Updates tracked in: `/delivery/PHASE_3_5_DELIVERY_PLAN.md`

---

## References

- [Ansible Lint Documentation](https://ansible.readthedocs.io/projects/lint/)
- [Molecule Testing](https://molecule.readthedocs.io/)
- [Terraform Best Practices](https://www.terraform.io/docs/configuration/best-practices.html)
- [tfsec Security Checks](https://aquasecurity.github.io/tfsec/latest/)
- [GitHub Actions Workflows](https://docs.github.com/en/actions/using-workflows)

---

**Last Updated:** 2026-06-28
**Phase:** 3-5 Infrastructure Implementation
**Ownership:** Werner van der Merwe (CI/CD Engineer)
