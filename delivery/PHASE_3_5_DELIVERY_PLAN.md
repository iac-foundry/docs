# Vernify Phase 3-5 Delivery Plan & Work Tickets

**Status:** Final Delivery Plan (Stage 6)  
**Date:** 28 June 2026  
**Owner:** Platform Engineering (Delivery Planning)  
**Context:** Comprehensive work breakdown for Phases 3–5 bootstrap deployment  
**Design Reference:** `/docs/design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md`

---

## Executive Summary

This delivery plan breaks the approved technical design for Vernify Phases 3–5 into 47 independently testable work items (tickets), each completable in ≤2 days, with explicit dependencies and acceptance criteria.

**Scope:**
- **Phase A:** Ansible collection scaffolding & dependency audit (4 collections)
- **Phase B:** Ansible collection role implementations (12 roles across 5 collections)
- **Phase C:** Terraform repository setup & LXC provisioning (3 repos)
- **Phase D:** Unified bootstrap script & integration orchestration (2 scripts + 1 integration)
- **Phase E:** CI/CD & automated testing (GitHub Actions workflows + Molecule tests)
- **Phase F:** Documentation & runbooks (architecture reference, operator guides)

**Key Metrics:**
- **Total Tickets:** 47
- **Total Effort:** 38.5 story days (~77 story points)
- **Estimated Duration (at 8 points/week):** 9.6 weeks (10 weeks with buffer)
- **Critical Path:** 21 days (if strictly sequenced; 7-8 days with parallelization)
- **Number of Parallel Workstreams:** 5 (collections, terraform repos, infrastructure, ci/cd, docs)

**Delivery Phases:**
- **Phase A (Week 1):** Collection scaffolding & Terraform module dependency verification
- **Phase B (Weeks 2–4):** Ansible collection implementations + Molecule tests
- **Phase C (Weeks 3–4):** Terraform repositories (parallel with Phase B)
- **Phase D (Week 5):** Bootstrap script & orchestration
- **Phase E (Week 6):** CI/CD workflows & automated testing
- **Phase F (Weeks 7–8):** Documentation & runbooks
- **Testing & Sign-Off (Week 9–10):** Full end-to-end testing, QA, final sign-off

---

## Implementation Phases

### Phase A: Collection Scaffolding & Dependencies (Weeks 1–2)
**Objective:** Verify all Ansible collections exist and identify missing roles/dependencies.  
**Deliverables:** Collection directory structure, requirements.yml with all dependencies, external dependency audit.  
**Story Points:** 4 (2 days)  
**Blocks:** Phase B (all collection implementations depend on this)

### Phase B: Ansible Collection Implementations (Weeks 2–5)
**Objective:** Implement all 12 roles across 5 collections with full test coverage.  
**Deliverables:** Complete role task bodies, Molecule test plays, local Docker testing capability.  
**Story Points:** 24 (12 days)  
**Parallelizable:** Collections can be implemented in parallel (step_ca, vault, jenkins, jenkins_integrations, vault_integrations).  
**Blocks:** Phase D (orchestration script depends on all roles being complete)

### Phase C: Terraform Repositories (Weeks 3–5)
**Objective:** Provision sec01, docker01, agent01 VMs/LXC on Proxmox.  
**Deliverables:** terraform-sec01-deploy, terraform-docker01-deploy, terraform-agent01-deploy (all functional).  
**Story Points:** 6 (3 days)  
**Dependency:** terraform-proxmox-vm module must be complete (external, from Phase 2).  
**Blocks:** Phase D bootstrap script (needs all three repos ready to apply)

### Phase D: Bootstrap Script & Orchestration (Weeks 5–6)
**Objective:** Unified bootstrap-phase-3-5.sh script orchestrating Terraform + Ansible across all phases.  
**Deliverables:** Single orchestration script, phase-specific playbooks, operator gates, idempotency checks.  
**Story Points:** 8 (4 days)  
**Depends On:** Phase B (roles) + Phase C (terraform repos)  
**Blocks:** Phase E testing

### Phase E: CI/CD & Testing (Weeks 6–7)
**Objective:** GitHub Actions workflows for lint, test, security scanning; Molecule test plays.  
**Deliverables:** GitHub Actions workflows, Molecule playbooks, test automation framework.  
**Story Points:** 6 (3 days)  
**Depends On:** Phase B (roles must be testable)

### Phase F: Documentation & Runbooks (Weeks 7–9)
**Objective:** Architecture reference, operator guides, troubleshooting, disaster recovery.  
**Deliverables:** LADRs (005–010), operator runbook, troubleshooting guide, API reference.  
**Story Points:** 4 (2 days)  
**Parallel with:** Phase E (can start while testing is ongoing)

---

## Dependency Graph

```
┌──────────────────────────────────────────────────────────────────┐
│                    Vernify Phase 3-5 Build Plan                  │
└──────────────────────────────────────────────────────────────────┘

EXTERNAL DEPENDENCY (Phase 2 closure):
  terraform-proxmox-vm module (must be complete)
         ↓
    ┌────┴────┐
    │          │
┌───▼───┐  ┌──▼──────────────────┐
│ Phase A│  │   terraform-build01   │  (rename → docker01)
│ Collect  │  │   -deploy completion  │
│ Scaffold │  └────────────────────┘
└───┬───┘
    │
    ├────────────────────────────────┐
    │                                │
    ▼                                ▼
┌──────────────────┐       ┌────────────────────────┐
│  Phase B (Parallel Tracks)       │  Phase C (Parallel)    │
│  5 Collections:              │  3 Terraform Repos:   │
│ - blueprints.step_ca     │ - sec01-deploy       │
│ - blueprints.vault       │ - docker01-deploy    │
│ - blueprints.jenkins     │ - agent01-deploy     │
│ - blueprints.jenkins_    │                      │
│   integrations           │ (3 days total)       │
│ - blueprints.vault_      │                      │
│   integrations           │                      │
│ (12 days total)          │                      │
└───────┬──────────────────┘       └────────┬───────┘
        │                                   │
        └──────────────────┬────────────────┘
                           │
                           ▼
                    ┌──────────────┐
                    │   Phase D    │
                    │ Bootstrap    │
                    │ Script (4d)  │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │   Phase E    │
                    │  CI/CD Tests │
                    │   (3 days)   │
                    └──────┬───────┘
                           │
            ┌──────────────┴──────────────┐
            │                             │
            ▼                             ▼
     ┌──────────────┐          ┌─────────────────┐
     │  Phase F     │          │ End-to-End Test │
     │ Docs/Runbks  │          │  & QA (2 weeks) │
     │  (2 days)    │          └─────────────────┘
     └──────────────┘
            │
            └────────────────┬──────────────────┐
                             │                  │
                             ▼                  ▼
                      ┌────────────┐    ┌────────────────┐
                      │ Production │    │  Deployment    │
                      │ Ready      │    │  & Hand-off    │
                      └────────────┘    └────────────────┘
```

**Critical Path (Longest Dependency Chain):**
```
terraform-proxmox-vm → Phase A (Scaffolding) → Phase B (Collections) 
  → Phase D (Bootstrap) → Phase E (CI/CD) → Phase F (Docs) → Testing
  
Duration: ~21 calendar days (if sequential) → 9-10 weeks (with QA, sign-off)
```

**Parallelizable Tracks:**
- Phase B + Phase C can run in parallel (independent delivery)
- Phase E + Phase F can run in parallel after Phase D

---

## Work Tickets

### Work Ticket Categories

1. **Collection Scaffolding** (A-01 to A-04): Verify collections exist, setup role directories
2. **Step-CA Collection** (B-01 to B-03): Implement 3 roles (container_server, issue_certificate, plus tests)
3. **Vault Collection** (B-04 to B-07): Implement 3 roles (container_server, init_and_unseal, tests)
4. **Jenkins Collection** (B-08 to B-10): Implement 2 roles (container_server, tests) + agent
5. **Jenkins Integrations** (B-11): vault_auth role
6. **Vault Integrations** (B-12): vault_agent role
7. **Terraform Repositories** (C-01 to C-06): Three repos (sec01, docker01, agent01)
8. **Bootstrap Script & Orchestration** (D-01 to D-03)
9. **CI/CD & Testing** (E-01 to E-06)
10. **Documentation** (F-01 to F-04)

---

## Phase A: Collection Scaffolding & Dependencies

### TICKET A-01: Audit Existing Collections & Role Inventory
**Priority:** P0 (critical)  
**Phase:** A (scaffolding)  
**Story Points:** 1  
**Type:** Infrastructure  
**Assigned To:** Infrastructure Engineer / Tech Lead

**Description:**  
Audit all five Ansible collections to verify they exist in `/iac-foundry/` and identify which roles are already present. Capture role names, status (stub/complete), and dependencies.

**Acceptance Criteria:**
- [ ] All 5 collections exist on disk (`blueprints.step_ca`, `blueprints.vault`, `blueprints.jenkins`, `blueprints.jenkins_integrations`, `blueprints.vault_integrations`)
- [ ] Role directory structure verified (each role has `tasks/main.yml`, `defaults/main.yml`, `meta/main.yml`)
- [ ] Inventory document created listing all 12 roles with status (stub/incomplete/complete)
- [ ] External dependencies identified (e.g., `community.docker`, `community.hashi_vault`, etc.) and documented
- [ ] No blockers identified; all collections are writable + have git access

**Deliverables:**
- `/docs/delivery/COLLECTION_ROLE_INVENTORY.md` (role status + dependencies)
- Git clone/pull commands for all 5 repos (verified working)

**Technical Notes:**
- Verify git remote is `github.com/iac-foundry/ansible-collection-*`
- Check for existing roles in `roles/` directory
- Confirm no naming conflicts with community collections

**Dependencies:**
- None (foundational ticket; unblocks all Phase B)

**Estimated Effort:**
- Audit + documentation: 1 day

---

### TICKET A-02: Set Up External Dependencies (requirements.yml)
**Priority:** P0  
**Phase:** A  
**Story Points:** 1  
**Type:** Infrastructure

**Description:**  
Create/update `requirements.yml` files in orchestration repos (bootstrap-container, terraform-sec01-deploy, etc.) to pin external Ansible collections and versions.

**Acceptance Criteria:**
- [ ] `requirements.yml` created in bootstrap-container with all dependencies pinned
- [ ] Collections include: community.docker, community.hashi_vault, community.general (minimum)
- [ ] Versions specified (e.g., `~> 1.0.0` for semantic versioning)
- [ ] `ansible-galaxy collection install -r requirements.yml` runs without error
- [ ] Documentation added explaining how to update dependencies

**Deliverables:**
- `/bootstrap-container/requirements.yml`
- Comment block explaining pinned versions + how to update

**Technical Notes:**
- Pin to known-good versions as of June 2026
- Use semantic versioning (`~> X.Y.Z`) to allow patch updates
- Test with `ansible-galaxy` to verify syntax

**Dependencies:**
- Depends on: A-01 (role inventory completed)

**Estimated Effort:**
- 1 day (research versions, test, document)

---

### TICKET A-03: Verify terraform-proxmox-vm Module Completeness
**Priority:** P0  
**Phase:** A  
**Story Points:** 1  
**Type:** Infrastructure  

**Description:**  
Verify that the `terraform-proxmox-vm` module (Phase 2 dependency) is complete, tested, and provides all required outputs (ip_address, vm_id, hostname).

**Acceptance Criteria:**
- [ ] Module exists at `github.com/iac-foundry/terraform-proxmox-vm`
- [ ] Module has `variables.tf`, `main.tf`, `outputs.tf` with no TODOs
- [ ] All required outputs present: `ip_address`, `vm_id`, `hostname`
- [ ] Terraform validate passes on module root
- [ ] README documents usage + examples
- [ ] No external blockers identified for Phase C (terraform repos)

**Deliverables:**
- Verification report (OK / Issues) documented in delivery plan
- If issues found: escalation ticket to Phase 2 team

**Technical Notes:**
- Run `terraform validate` on module directory
- Check that outputs match expected types (string, list)
- Verify usage in example main.tf (if exists)

**Dependencies:**
- Foundational (external Phase 2 dependency)

**Estimated Effort:**
- 0.5 day (review module, test validate, document findings)

---

### TICKET A-04: Create Ansible Playbook Bootstrap Template
**Priority:** P1  
**Phase:** A  
**Story Points:** 1  
**Type:** Infrastructure

**Description:**  
Create template/skeleton playbook for orchestration phases that demonstrates best practices: error handling, gates, logging, idempotency checks.

**Acceptance Criteria:**
- [ ] Playbook template created at `/bootstrap-container/playbooks/template-orchestration.yml`
- [ ] Template includes sections: pre-flight validation, phase execution, gates, verification, error handling
- [ ] Documented comments explain each section
- [ ] Template used as reference for phase-specific playbooks (Phase D)
- [ ] Syntax validates (ansible-playbook --syntax-check)

**Deliverables:**
- `/bootstrap-container/playbooks/template-orchestration.yml`
- Documentation link from this delivery plan

**Technical Notes:**
- Base structure: import roles, handle errors, wait for services, print summary
- Include examples of: `assert`, `pause` (for gates), `wait_for`, error handling
- Best practices: no_log for secrets, clear debug messages

**Dependencies:**
- Depends on: A-01 (collections audited)

**Estimated Effort:**
- 0.5 day (template creation + documentation)

---

## Phase B: Ansible Collection Implementations

### TICKET B-01: blueprints.step_ca — container_server Role
**Priority:** P0  
**Phase:** B  
**Story Points:** 2  
**Type:** Implementation  
**Assigned To:** Ansible Expert

**Description:**  
Implement the `container_server` role for `blueprints.step_ca` collection. Deploy step-ca as a non-root Docker container, initialize root CA, and output TOFU fingerprint.

**Acceptance Criteria:**
- [ ] Role tasks complete: Docker install, user creation, volume setup, container deployment
- [ ] Container runs as non-root user (uid 1000 for step-ca)
- [ ] Root CA initialized on first run; idempotent on re-runs (check `/opt/step-ca/certs/root_ca.crt`)
- [ ] TOFU fingerprint captured and printed to console + stored as fact
- [ ] Container health verified: API responds on port 9000
- [ ] No hardcoded Vernify-specific values (all parameterized via defaults/vars)
- [ ] No log output contains passwords; all sensitive data marked `no_log: true`
- [ ] Full integration test runs in Molecule (next ticket)

**Deliverables:**
- `/ansible-collection-step-ca/roles/container_server/tasks/main.yml` (complete)
- `/ansible-collection-step-ca/roles/container_server/defaults/main.yml` (defaults + schema comments)
- `/ansible-collection-step-ca/roles/container_server/meta/main.yml` (dependencies)

**Technical Notes:**
- Docker image: `smallstep/step-ca:{{ step_ca_image_tag }}`
- User creation: `useradd -m -u 1000 step-ca` (idempotent)
- Provisioner password injected via env var `STEPCA_PASSWORD` (transient)
- Fingerprint extraction: run `step ca fingerprint` CLI after init
- Volume ownership: `chown -R step-ca:step-ca /opt/step-ca`

**Key Task Sequence:**
1. Install Docker (community.docker.docker_* modules)
2. Create step-ca user + group
3. Create volume directory with correct ownership
4. Pull container image
5. Deploy container (non-root user, cap_drop=ALL)
6. Wait for API readiness (curl https://localhost:9000)
7. Extract fingerprint + store as fact
8. Print fingerprint to console

**Dependencies:**
- Depends on: A-01 (collections audited), A-02 (requirements.yml ready)
- Blocks: B-02 (issue_certificate role needs container running), B-03 (tests)

**Estimated Effort:**
- 1.5 days (role writing, local testing, error handling)

---

### TICKET B-02: blueprints.step_ca — issue_certificate Role
**Priority:** P0  
**Phase:** B  
**Story Points:** 2  
**Type:** Implementation

**Description:**  
Implement the `issue_certificate` role for `blueprints.step_ca`. Request and issue a TLS certificate from step-ca for a given CN/SANs, with TOFU fingerprint validation and post-issuance SAN verification.

**Acceptance Criteria:**
- [ ] Role tasks complete: certificate request, fingerprint verification, SANs validation
- [ ] Idempotent: if cert exists + is valid, skip issuance (check expiration)
- [ ] **P0-5 SAN validation:** Post-issuance, verify FQDN is in SAN list via `step certificate inspect` (mandatory)
- [ ] Fail clearly if FQDN not in SAN list (indicates misconfiguration)
- [ ] Cert output: saved to `{{ cert_output_dir }}/{{ cert_cn }}.crt` + `.key`
- [ ] Permissions: cert 0644, key 0600
- [ ] Retry logic: 3 attempts if CA unreachable (with delay)
- [ ] Full integration test in Molecule (next ticket)

**Deliverables:**
- `/ansible-collection-step-ca/roles/issue_certificate/tasks/main.yml` (complete)
- `/ansible-collection-step-ca/roles/issue_certificate/defaults/main.yml`
- Returned facts: `issued_cert_path`, `issued_key_path`, `cert_not_after`, `cert_days_remaining`

**Technical Notes:**
- Use `step ca certificate` CLI with TOFU fingerprint
- SAN list must include FQDN + all variants (e.g., vault, vault.sec01, vault.sec01.vernify.internal, 192.168.22.51)
- Fingerprint validation: `step ca certificate --fingerprint <fingerprint>`
- **Critical:** `step certificate inspect <path> | grep -A 5 "Subject Alternative Name"` to verify SANs
- Fail task if FQDN not found in SAN output
- Permissions handled via Ansible `file` module

**Key Task Sequence:**
1. Check if cert already exists + is valid (no re-issue)
2. Prepare SANs list from input variable
3. Call `step ca certificate` with CN + SANs + fingerprint
4. Wait for cert files to appear (timeout 30s)
5. Parse cert with `step certificate inspect`
6. Verify FQDN in SAN list (fail if not found)
7. Set fact with cert path + expiration
8. Log certificate metadata (not secret)

**Dependencies:**
- Depends on: B-01 (step-ca container must be running)
- Blocks: Phase D (orchestration playbooks need certs for Vault + Jenkins)

**Estimated Effort:**
- 1.5 days (certificate issuance, SAN validation, error handling)

---

### TICKET B-03: blueprints.step_ca — Molecule Test Playbook
**Priority:** P1  
**Phase:** B  
**Story Points:** 1  
**Type:** Testing

**Description:**  
Write Molecule test playbook for `blueprints.step_ca` collection. Test container deployment, CA initialization, certificate issuance, and SAN validation in local Docker environment.

**Acceptance Criteria:**
- [ ] Molecule playbook created at `/ansible-collection-step-ca/molecule/default/converge.yml`
- [ ] Tests cover: container deployment, CA init, cert issuance with SANs, fingerprint extraction
- [ ] Molecule test runs locally (via Docker) without errors
- [ ] Test result: all assertions pass (container healthy, certs present, SANs valid)
- [ ] Molecule destroy removes test containers + volumes cleanly

**Deliverables:**
- `/ansible-collection-step-ca/molecule/default/converge.yml`
- `/ansible-collection-step-ca/molecule/default/molecule.yml` (config)
- `/ansible-collection-step-ca/molecule/default/verify.yml` (post-test assertions)

**Technical Notes:**
- Molecule driver: docker (lightweight, fast)
- Container platform: Ubuntu 22.04 (matches prod)
- Test scenario: single role + dependencies (community.docker, etc.)
- Verify tasks: check container running, cert files exist, SANs correct

**Key Test Assertions:**
1. Container `step-ca` is running + healthy (curl HTTPS)
2. Root CA cert exists at `/opt/step-ca/certs/root_ca.crt`
3. Issued cert has correct CN + SANs in list
4. Fingerprint can be extracted (non-empty)

**Dependencies:**
- Depends on: B-01 (container_server role), B-02 (issue_certificate role)

**Estimated Effort:**
- 1 day (test playbook, molecule config, assertions)

---

### TICKET B-04: blueprints.vault — container_server Role
**Priority:** P0  
**Phase:** B  
**Story Points:** 2  
**Type:** Implementation

**Description:**  
Implement the `container_server` role for `blueprints.vault`. Deploy Vault as a non-root Docker container, configure listeners + storage backend, prepare for init (do NOT run init).

**Acceptance Criteria:**
- [ ] Role tasks complete: Docker install, user creation, volume setup, config file, container deployment
- [ ] Container runs as non-root user (uid 100 for vault)
- [ ] Capabilities: `--cap-add IPC_LOCK --cap-drop=ALL` (allow memory encryption, drop everything else)
- [ ] Config file: `/etc/vault/config.hcl` with listener + file storage backend config
- [ ] Idempotent: if storage exists, skip container setup
- [ ] Container health verified: curl API endpoint (expect 501/503 if sealed = expected)
- [ ] No init/unseal performed (that is platform-layer work, next ticket)
- [ ] Full integration test in Molecule

**Deliverables:**
- `/ansible-collection-vault/roles/container_server/tasks/main.yml` (complete)
- `/ansible-collection-vault/roles/container_server/defaults/main.yml`
- `/ansible-collection-vault/roles/container_server/templates/vault-config.hcl.j2` (config template)

**Technical Notes:**
- Docker image: `vault:{{ vault_image_tag }}`
- User creation: `useradd -m -u 100 vault`
- Config: listener on `{{ vault_listener_address }}:{{ vault_listener_port }}` with TLS
- Storage: file backend at `/vault/data` (mounted volume)
- Volume ownership: `chown vault:vault /opt/vault/data` + perms 0700

**Key Task Sequence:**
1. Install Docker
2. Create vault user + group
3. Create volume directory with correct ownership
4. Create config file (jinja2 template) with listener + storage config
5. Pull container image
6. Deploy container (non-root, cap_add IPC_LOCK, cap_drop ALL)
7. Wait for API readiness (poll, expect 501 or 503 if sealed)
8. Verify storage directory exists (no init yet)

**Dependencies:**
- Depends on: A-01 (collections audited)
- Blocks: B-05 (init_and_unseal role needs container running), B-06 (tests)

**Estimated Effort:**
- 1.5 days (config templating, container setup, health checks)

---

### TICKET B-05: blueprints.vault — init_and_unseal Role
**Priority:** P0  
**Phase:** B  
**Story Points:** 2  
**Type:** Implementation

**Description:**  
Implement the `init_and_unseal` role for `blueprints.vault`. Initialize Vault (one-time), capture unseal keys + root token, create auto-unseal systemd service and script.

**Acceptance Criteria:**
- [ ] Vault init runs once (idempotent: check if keyring exists, skip if yes)
- [ ] Unseal keys + root token captured from init output (JSON parsing)
- [ ] Unseal keys printed to console with **CRITICAL** warning header
- [ ] Root token written to `/opt/vault/root-token.txt` (0600 perms)
- [ ] Unseal keys written to `/opt/vault/unseal-keys.json` (0600 perms)
- [ ] Unseal script created at `/opt/vault/unseal.sh` (0700 perms) with embedded keys
- [ ] Systemd service created: `/etc/systemd/system/vault-auto-unseal.service`
- [ ] Service enabled but NOT started (operator gates this)
- [ ] Role returns facts: `vault_root_token`, `vault_unseal_keys`, `vault_threshold`, `vault_is_initialized`
- [ ] Full integration test in Molecule

**Deliverables:**
- `/ansible-collection-vault/roles/init_and_unseal/tasks/main.yml` (complete)
- `/ansible-collection-vault/roles/init_and_unseal/templates/unseal.sh.j2` (unseal script template)
- `/ansible-collection-vault/roles/init_and_unseal/templates/vault-auto-unseal.service.j2` (systemd unit template)

**Technical Notes:**
- Vault init command: `vault operator init -key-shares=5 -key-threshold=3 -format=json`
- Key parsing: use Ansible `from_json | map(attribute='keys_b64')` etc.
- Unseal script: `for key in "${KEYS[@]}"; do vault operator unseal "$key"; done`
- Systemd unit: Type=oneshot, After=network.target vault.service, no Restart
- Security: keys in script are plaintext (0700); mitigation = operator backs up externally

**Key Task Sequence:**
1. Check idempotency: `vault status` or check keyring file
2. If not initialized: run `vault operator init` → capture JSON output
3. Parse unseal_keys_b64 + root_token_b64
4. Write keys + token to secure files (0600)
5. Generate unseal script (template) with embedded keys
6. Create systemd service (template)
7. Enable service (systemctl enable)
8. Set facts with captured outputs
9. Print critical warning to console

**Dependencies:**
- Depends on: B-04 (Vault container must be running)
- Blocks: B-06 (tests), D-01 (orchestration playbooks)

**Estimated Effort:**
- 1.5 days (init logic, script/service templates, idempotency, error handling)

---

### TICKET B-06: blueprints.vault — Molecule Test Playbook
**Priority:** P1  
**Phase:** B  
**Story Points:** 1  
**Type:** Testing

**Description:**  
Write Molecule test playbook for `blueprints.vault`. Test container deployment, Vault init, unseal script creation, and systemd service in local Docker environment.

**Acceptance Criteria:**
- [ ] Molecule playbook created + runs locally without errors
- [ ] Tests cover: container deploy, Vault init, unseal script + systemd service creation
- [ ] Unseal script can run manually (test it)
- [ ] Auto-unseal systemd service enabled + ready
- [ ] Vault is unsealed after running unseal script
- [ ] Molecule destroy cleans up all artifacts

**Deliverables:**
- `/ansible-collection-vault/molecule/default/converge.yml`
- `/ansible-collection-vault/molecule/default/molecule.yml`
- `/ansible-collection-vault/molecule/default/verify.yml`

**Key Test Assertions:**
1. Vault container running + healthy
2. Init output captured (unseal keys + root token present)
3. Unseal script present + executable
4. Systemd service file exists + enabled
5. Running unseal script results in `vault status` showing unsealed

**Dependencies:**
- Depends on: B-04 (container_server), B-05 (init_and_unseal)

**Estimated Effort:**
- 1 day (test playbook, assertions, service verification)

---

### TICKET B-07: blueprints.vault — Role Documentation & Defaults Schema
**Priority:** P2  
**Phase:** B  
**Story Points:** 0.5  
**Type:** Documentation

**Description:**  
Document both vault roles (container_server + init_and_unseal) with input/output specifications and schema comments.

**Acceptance Criteria:**
- [ ] Each role has `README.md` explaining purpose, inputs, outputs, idempotency
- [ ] `defaults/main.yml` includes schema comments (e.g., `# @var vault_image_tag: Container image tag (string)`)
- [ ] Examples provided for typical usage
- [ ] Cross-references to design doc (section 3.2)

**Deliverables:**
- `/ansible-collection-vault/roles/container_server/README.md`
- `/ansible-collection-vault/roles/init_and_unseal/README.md`
- Schema comments in both `defaults/main.yml`

**Dependencies:**
- Depends on: B-04, B-05 (roles complete)

**Estimated Effort:**
- 0.5 day (documentation + schema comments)

---

### TICKET B-08: blueprints.jenkins — container_server Role
**Priority:** P0  
**Phase:** B  
**Story Points:** 2  
**Type:** Implementation

**Description:**  
Implement the `container_server` role for `blueprints.jenkins`. Deploy Jenkins as a non-root Docker container, configure HTTPS listener, prepare for Job DSL seeding.

**Acceptance Criteria:**
- [ ] Role tasks complete: Docker install, user creation, volume setup, TLS config, container deployment
- [ ] Container runs as non-root user (uid 1000 for jenkins)
- [ ] TLS certificate + key mounted from host (`/etc/ssl/certs/jenkins.*`)
- [ ] Idempotent: if `/opt/jenkins/.initialized` exists, skip setup
- [ ] Jenkins web UI accessible on HTTPS (port 8443 by default)
- [ ] Jenkins CLI jar downloaded + ready for job creation
- [ ] Admin token stored securely (Jenkins credentials store)
- [ ] Container health verified: curl HTTPS API endpoint
- [ ] Full integration test in Molecule

**Deliverables:**
- `/ansible-collection-jenkins/roles/container_server/tasks/main.yml` (complete)
- `/ansible-collection-jenkins/roles/container_server/defaults/main.yml`
- `/ansible-collection-jenkins/roles/container_server/templates/jenkins-config.xml.j2` (optional, if TLS config needed)

**Technical Notes:**
- Docker image: `jenkins/jenkins:{{ jenkins_image_tag }}`
- User creation: `useradd -m -u 1000 jenkins`
- HTTPS: TLS cert/key paths passed as env vars or config
- Jenkins CLI: download `jenkins-cli.jar` from Jenkins download server
- Admin token: stored in Jenkins credentials (encrypted by Jenkins)
- Volume ownership: `chown jenkins:jenkins /opt/jenkins`

**Key Task Sequence:**
1. Install Docker
2. Create jenkins user
3. Create volume directory (`/opt/jenkins`) with ownership
4. Pull container image
5. Create Jenkins configuration (listener on 8443 HTTPS)
6. Deploy container (non-root, volume mounts)
7. Wait for Jenkins boot (poll API)
8. Download jenkins-cli.jar
9. Store admin token in Jenkins credentials
10. Create marker file `.initialized`

**Dependencies:**
- Depends on: A-01 (collections audited)
- Blocks: B-09 (vault_auth integration), B-10 (agent role)

**Estimated Effort:**
- 1.5 days (container setup, TLS config, Jenkins readiness, CLI download)

---

### TICKET B-09: blueprints.jenkins — Molecule Test Playbook
**Priority:** P1  
**Phase:** B  
**Story Points:** 1  
**Type:** Testing

**Description:**  
Write Molecule test playbook for `blueprints.jenkins.container_server`. Test container deployment, HTTPS accessibility, CLI jar availability, and Jenkins API responsiveness.

**Acceptance Criteria:**
- [ ] Molecule playbook runs locally without errors
- [ ] Jenkins container running + healthy
- [ ] HTTPS API accessible (curl with --insecure for self-signed)
- [ ] Jenkins CLI jar present + executable
- [ ] Jenkins web UI responds on port 8443
- [ ] Molecule destroy cleans up resources

**Deliverables:**
- `/ansible-collection-jenkins/molecule/default/converge.yml`
- `/ansible-collection-jenkins/molecule/default/verify.yml`

**Key Test Assertions:**
1. Jenkins container running
2. HTTPS API returns 200 (or 403 if auth required, expected)
3. CLI jar exists + file type is JAR
4. Jenkins UI accessible (curl -k https://localhost:8443)

**Dependencies:**
- Depends on: B-08 (container_server role)

**Estimated Effort:**
- 1 day (test playbook, assertions)

---

### TICKET B-10: blueprints.jenkins — Jenkins Agent Role (Optional/Phase 5)
**Priority:** P1  
**Phase:** B  
**Story Points:** 1.5  
**Type:** Implementation

**Description:**  
Implement agent role for `blueprints.jenkins` (used in Phase 5 on agent01). Deploy Jenkins agent as systemd service, configure JNLP connection to Jenkins controller, set agent labels.

**Acceptance Criteria:**
- [ ] Role tasks complete: agent jar download, systemd service setup, JNLP connection config
- [ ] Systemd service: `/etc/systemd/system/jenkins-agent.service`
- [ ] Service connects to Jenkins controller via JNLP (port 50000)
- [ ] Agent labels applied (e.g., packer-build, terraform-plan, ansible-execute)
- [ ] Idempotent: if service exists, skip setup (or upgrade if config changed)
- [ ] Full integration test in Molecule

**Deliverables:**
- `/ansible-collection-jenkins/roles/agent/tasks/main.yml`
- `/ansible-collection-jenkins/roles/agent/templates/jenkins-agent.service.j2`

**Technical Notes:**
- Agent jar: download from Jenkins controller via slave-agent.jnlp
- JNLP: Jenkins controller must have port 50000 open (firewall rule)
- Service user: jenkins (non-root)
- Labels: passed as var (list)

**Key Task Sequence:**
1. Download agent jar from controller
2. Create systemd service file
3. Start + enable service
4. Verify connection to controller (log check)
5. Verify agent appears in Jenkins UI

**Dependencies:**
- Depends on: B-08 (Jenkins container running + ready)
- Blocks: Phase 5 (agent01 deployment)

**Estimated Effort:**
- 1 day (agent setup, service config, connection verification)

---

### TICKET B-11: blueprints.jenkins_integrations — vault_auth Role
**Priority:** P0  
**Phase:** B  
**Story Points:** 1.5  
**Type:** Implementation

**Description:**  
Implement the `vault_auth` role for `blueprints.jenkins_integrations`. Configure Jenkins Vault plugin with AppRole credentials, enable Vault as credentials provider.

**Acceptance Criteria:**
- [ ] Vault plugin installed + enabled in Jenkins
- [ ] Vault plugin config written: endpoint URL, role_id, secret_id, CA cert path
- [ ] Jenkins credentials store includes Vault provider
- [ ] Test: Jenkins can authenticate to Vault via AppRole
- [ ] AppRole token TTL configured
- [ ] Full integration test in Molecule

**Deliverables:**
- `/ansible-collection-jenkins-integrations/roles/vault_auth/tasks/main.yml`
- `/ansible-collection-jenkins-integrations/roles/vault_auth/templates/vault-config.xml.j2`

**Technical Notes:**
- Vault plugin: `hashicorp-vault-plugin` (install via Jenkins CLI)
- Config location: `$JENKINS_HOME/secrets/vault-config.xml`
- AppRole: retrieved from Vault (role_id + secret_id)
- CA cert: path to step-ca root cert on Jenkins host

**Key Task Sequence:**
1. Install Vault plugin (jenkins-cli.jar)
2. Create vault-config.xml with credentials
3. Restart Jenkins to load config
4. Test authentication (curl Vault API via Jenkins)

**Dependencies:**
- Depends on: B-08 (Jenkins container), and Vault must be initialized + unsealed (Phase 3 output)
- Blocks: Phase D (orchestration playbooks need this for secret retrieval)

**Estimated Effort:**
- 1 day (plugin install, config template, auth test)

---

### TICKET B-12: blueprints.vault_integrations — vault_agent Role
**Priority:** P1  
**Phase:** B  
**Story Points:** 1.5  
**Type:** Implementation

**Description:**  
Implement the `vault_agent` role for `blueprints.vault_integrations`. Deploy Vault agent as systemd service on agent01, auto-renew secrets + TLS certificates.

**Acceptance Criteria:**
- [ ] Vault agent config created: `/etc/vault-agent/vault-agent.hcl`
- [ ] Systemd service: `/etc/systemd/system/vault-agent.service`
- [ ] AppRole auth method configured (role_id + secret_id files)
- [ ] Agent auto-renews secrets (TTL + renewal before expiry)
- [ ] TLS certs templated to local files
- [ ] Service enabled + running
- [ ] Full integration test in Molecule

**Deliverables:**
- `/ansible-collection-vault-integrations/roles/vault_agent/tasks/main.yml`
- `/ansible-collection-vault-integrations/roles/vault_agent/templates/vault-agent.hcl.j2`
- `/ansible-collection-vault-integrations/roles/vault_agent/templates/vault-agent.service.j2`

**Technical Notes:**
- Vault agent: runs as non-root (vault user)
- Auto-auth: AppRole with role_id + secret_id files
- Secrets templating: output to `/etc/vault-agent/secrets/` (readable by agent)
- TTL + renewal: Vault agent handles automatically

**Key Task Sequence:**
1. Create vault-agent.hcl config with AppRole auth
2. Create systemd service file
3. Start + enable service
4. Verify service running (systemctl status)
5. Verify secrets templated to local files

**Dependencies:**
- Depends on: Vault must be unsealed + AppRole created (Phase 3 output)
- Blocks: Phase 5 (agent01 deployment)

**Estimated Effort:**
- 1 day (agent config, service setup, secret templating)

---

## Phase C: Terraform Repositories

### TICKET C-01: terraform-sec01-deploy — VM Provisioning
**Priority:** P0  
**Phase:** C  
**Story Points:** 1.5  
**Type:** Infrastructure

**Description:**  
Implement terraform-sec01-deploy repository to provision sec01 VM on Proxmox. Specs: 8 vCPU, 16 GB RAM, 50 GB disk, static IP 192.168.22.51/24.

**Acceptance Criteria:**
- [ ] `main.tf` uses terraform-proxmox-vm module (from Phase 2)
- [ ] `variables.tf` includes all required inputs (Proxmox endpoint, user, password, template, IP, etc.)
- [ ] `outputs.tf` returns: `sec01_ip_address`, `sec01_vm_id`, `sec01_hostname`
- [ ] `terraform validate` passes without errors
- [ ] `terraform plan` produces expected output (no errors, shows VM creation)
- [ ] `terraform.tfvars.example` provided (secrets redacted)
- [ ] Remote backend configured (TFC workspace)
- [ ] `.gitignore` prevents committing `terraform.tfvars` + `.terraform/`
- [ ] README documents usage + variable descriptions

**Deliverables:**
- `/terraform-sec01-deploy/main.tf`
- `/terraform-sec01-deploy/variables.tf`
- `/terraform-sec01-deploy/outputs.tf`
- `/terraform-sec01-deploy/terraform.tf` (backend config)
- `/terraform-sec01-deploy/README.md`

**Technical Notes:**
- Module source: `git::https://github.com/iac-foundry/terraform-proxmox-vm.git`
- Template name: `ubuntu-base-container-host` (from Phase 1 Packer)
- Network: bridge `vmbr0`, DHCP or static IP via cloud-init
- Tags: environment=production, service=security, phase=phase-3

**Key Variables:**
- `proxmox_ve_endpoint`: https://proxmox.vernify.internal:8006
- `proxmox_ve_username`: root@pam or tf@pve
- `proxmox_ve_password`: (from TFC variable set)
- `vm_cpu`: 8
- `vm_memory`: 16384
- `vm_disk_size`: 50
- `ip_address`: 192.168.22.51/24
- `gateway`: 192.168.22.1
- `dns_servers`: ["8.8.8.8", "1.1.1.1"]

**Dependencies:**
- Depends on: A-03 (terraform-proxmox-vm module verified complete)
- Blocks: D-01 (bootstrap script Phase 3a)

**Estimated Effort:**
- 1 day (repo setup, module usage, variable definition, testing)

---

### TICKET C-02: terraform-docker01-deploy — VM Provisioning
**Priority:** P0  
**Phase:** C  
**Story Points:** 1.5  
**Type:** Infrastructure

**Description:**  
Implement terraform-docker01-deploy repository to provision docker01 VM (formerly build01). Specs: 4 vCPU, 8 GB RAM, 50 GB disk, static IP 192.168.22.52/24.

**Acceptance Criteria:**
- [ ] Identical pattern to terraform-sec01-deploy (uses same module)
- [ ] Variables specific to docker01: 4 vCPU, 8 GB RAM, IP .52
- [ ] `terraform validate` passes
- [ ] `terraform plan` shows expected docker01 VM creation
- [ ] Backend, README, .gitignore as per sec01 repo
- [ ] Remote backend configured (TFC workspace)

**Deliverables:**
- `/terraform-docker01-deploy/main.tf`
- `/terraform-docker01-deploy/variables.tf`
- `/terraform-docker01-deploy/outputs.tf`
- `/terraform-docker01-deploy/terraform.tf`
- `/terraform-docker01-deploy/README.md`

**Technical Notes:**
- Same pattern as C-01 (copy + customize vars)
- VM specs: 4 vCPU, 8192 MB RAM
- Tags: environment=production, service=ci, role=container-host

**Key Variables:**
- `vm_cpu`: 4
- `vm_memory`: 8192
- `ip_address`: 192.168.22.52/24
- Other vars same as sec01

**Dependencies:**
- Depends on: A-03 (terraform-proxmox-vm module)
- Blocks: D-01 (bootstrap script Phase 4a)

**Estimated Effort:**
- 1 day (copy sec01 repo, customize vars, test)

---

### TICKET C-03: terraform-agent01-deploy — LXC Container Provisioning
**Priority:** P0  
**Phase:** C  
**Story Points:** 2  
**Type:** Infrastructure

**Description:**  
Implement terraform-agent01-deploy repository to provision agent01 as LXC container on Proxmox. Specs: 2 vCPU, 4 GB RAM, 20 GB disk, static IP 192.168.22.53/24.

**Acceptance Criteria:**
- [ ] `main.tf` provisions LXC container (not full VM) via Proxmox provider
- [ ] LXC-specific variables: container_type=lxc, container_cpu=2, container_memory=4096
- [ ] Network configuration: static IP 192.168.22.53/24 (DHCP reservation or direct config)
- [ ] Storage: 20 GB root filesystem
- [ ] `terraform validate` passes
- [ ] `terraform plan` shows LXC container creation
- [ ] Outputs: agent01_ip, agent01_container_id
- [ ] README documents LXC-specific setup + assumptions
- [ ] **Fallback note:** If LXC Terraform module not available, design supports Ansible provisioning (fallback option)

**Deliverables:**
- `/terraform-agent01-deploy/main.tf`
- `/terraform-agent01-deploy/variables.tf`
- `/terraform-agent01-deploy/outputs.tf`
- `/terraform-agent01-deploy/terraform.tf`
- `/terraform-agent01-deploy/README.md`

**Technical Notes:**
- Provider: Telmate/proxmox (Proxmox provider)
- Resource: `proxmox_lxc` module (or use Proxmox provider directly)
- Template: base LXC image (from Phase 1, or Ubuntu cloud image)
- Network namespace isolated; static IP via DHCP or direct config
- **Alternative (fallback):** If Terraform LXC module unavailable, bootstrap script can use Ansible `proxmox_lxc` module instead

**Key Variables:**
- `container_type`: "lxc"
- `container_name`: "agent01"
- `container_cpu`: 2
- `container_memory`: 4096 (in MB)
- `container_disk_size`: 20
- `ip_address`: 192.168.22.53/24
- `gateway`: 192.168.22.1

**Dependencies:**
- Depends on: A-03 (terraform-proxmox-vm module, LXC variant if available)
- Blocks: D-01 (bootstrap script Phase 5a)
- **Risk:** If Terraform LXC module not ready, escalate to bootstrap script fallback (Ansible provisioning)

**Estimated Effort:**
- 2 days (LXC provisioning setup, network config, testing; may include fallback logic)

---

### TICKET C-04: terraform-sec01-deploy — CI/CD Testing
**Priority:** P2  
**Phase:** C  
**Story Points:** 0.5  
**Type:** Testing

**Description:**  
Set up CI/CD pipeline for terraform-sec01-deploy (terraform validate + plan).

**Acceptance Criteria:**
- [ ] GitHub Actions workflow created: `.github/workflows/terraform-validate.yml`
- [ ] Workflow runs `terraform init` + `terraform validate`
- [ ] Workflow runs on pull requests (blocks merge if validate fails)
- [ ] Workflow runs on push to main (informational)

**Deliverables:**
- `/terraform-sec01-deploy/.github/workflows/terraform-validate.yml`

**Dependencies:**
- Depends on: C-01 (repo complete)

**Estimated Effort:**
- 0.5 day (workflow setup)

---

### TICKET C-05: terraform-docker01-deploy — CI/CD Testing
**Priority:** P2  
**Phase:** C  
**Story Points:** 0.5  
**Type:** Testing

**Description:**  
Set up CI/CD pipeline for terraform-docker01-deploy.

**Acceptance Criteria:**
- [ ] GitHub Actions workflow similar to C-04
- [ ] Runs terraform validate + plan

**Deliverables:**
- `/terraform-docker01-deploy/.github/workflows/terraform-validate.yml`

**Dependencies:**
- Depends on: C-02

**Estimated Effort:**
- 0.5 day

---

### TICKET C-06: terraform-agent01-deploy — CI/CD Testing
**Priority:** P2  
**Phase:** C  
**Story Points:** 0.5  
**Type:** Testing

**Description:**  
Set up CI/CD pipeline for terraform-agent01-deploy.

**Acceptance Criteria:**
- [ ] GitHub Actions workflow similar to C-04
- [ ] Runs terraform validate + plan

**Deliverables:**
- `/terraform-agent01-deploy/.github/workflows/terraform-validate.yml`

**Dependencies:**
- Depends on: C-03

**Estimated Effort:**
- 0.5 day

---

## Phase D: Bootstrap Script & Orchestration

### TICKET D-01: Unified Bootstrap Script (bootstrap-phase-3-5.sh)
**Priority:** P0  
**Phase:** D  
**Story Points:** 3  
**Type:** Implementation

**Description:**  
Implement the unified bootstrap-phase-3-5.sh script that orchestrates all three phases (3, 4, 5) in a single idempotent entry point.

**Acceptance Criteria:**
- [ ] Script accepts command-line arguments: `--proxmox-endpoint`, `--proxmox-password`, `--tfc-token`, `--step-ca-provisioner-password`, `--jenkins-admin-token`
- [ ] Pre-flight validation: all args provided, Proxmox connectivity, TFC workspace exists
- [ ] Phase 3a: Terraform apply terraform-sec01-deploy
- [ ] Phase 3b: Wait for sec01 SSH (retries: 10, delay: 10s)
- [ ] Phase 3c-f: Run Ansible playbooks (step-ca, Vault container, init, unseal, seeding)
- [ ] Phase 3 completion: Print unseal keys + operator gate (backup confirmation)
- [ ] Phase 4a: Terraform apply terraform-docker01-deploy
- [ ] Phase 4b-f: Run Ansible playbooks (Jenkins, Vault integration, Job DSL seeding)
- [ ] Phase 5a-f: Similar for agent01
- [ ] Idempotency: each phase checks completion markers, skips if already done
- [ ] Error handling: clear error messages + rollback instructions
- [ ] Completion summary: status of all phases + next steps

**Deliverables:**
- `/bootstrap-container/scripts/bootstrap-phase-3-5.sh` (executable)
- Usage documentation embedded as help text (./script.sh --help)

**Technical Notes:**
- Language: bash (shell script) or Python (if orchestration complexity requires)
- Entry checks: terraform, ansible, vault CLI available
- Idempotency pattern: `terraform state show <resource>` + check marker files
- Error handling: trap errors, print diagnostics, suggest remediation
- Logging: all output to stdout/stderr + optionally to logfile

**Key Functions:**
1. `pre_flight_checks()` — validate args, check CLI tools, test connectivity
2. `phase_3()` — sec01 provisioning + Vault setup
3. `phase_4()` — docker01 provisioning + Jenkins setup
4. `phase_5()` — agent01 provisioning + agent setup
5. `completion_summary()` — print infrastructure status + next steps
6. `error_handler()` — trap + log errors, suggest recovery

**Phase 3 Orchestration:**
```bash
terraform apply terraform-sec01-deploy
wait_for_ssh sec01 192.168.22.51
ansible-playbook converge-step-ca.yml
ansible-playbook converge-vault.yml
ansible-playbook init-vault.yml
# GATE: operator backs up unseal keys
ansible-playbook unseal-vault.yml
ansible-playbook seed-vault.yml
```

**Dependencies:**
- Depends on: B-01 to B-12 (all roles complete), C-01 to C-03 (all terraform repos)
- Blocks: E-01 (E2E testing)

**Estimated Effort:**
- 2 days (script structure, orchestration logic, error handling, testing)

---

### TICKET D-02: Phase 3 Orchestration Playbook (phase-3-orchestrate.yml)
**Priority:** P0  
**Phase:** D  
**Story Points:** 2  
**Type:** Implementation

**Description:**  
Implement phase-3-orchestrate.yml playbook that orchestrates Step 3a–3j: sec01 provisioning, step-ca setup, Vault init/unseal, secret seeding.

**Acceptance Criteria:**
- [ ] Playbook includes: Terraform apply, SSH wait, step-ca convergence, Vault convergence, init, unseal, secret seeding
- [ ] Operator gates implemented: pause for unseal key backup (task: wait_for user input)
- [ ] Vault root token + unseal keys printed to console with critical warning
- [ ] Secret seeding uses community.hashi_vault lookups (no hardcoded secrets)
- [ ] Ansible `no_log: true` on all sensitive tasks (keys, tokens)
- [ ] Idempotent: step-ca init skipped if CA already exists, Vault init skipped if initialized
- [ ] Error handling: clear error messages + remediation suggestions
- [ ] Full dry-run test in Molecule (mock Vault API)

**Deliverables:**
- `/bootstrap-container/playbooks/phase-3-orchestrate.yml` (complete)
- All referenced role invocations (include_role) work correctly
- Example execution log (redacted) showing expected output

**Technical Notes:**
- Terraform module: reference sec01 outputs (ip_address, vm_id)
- SSH wait: use wait_for module (delay=10, timeout=300)
- Role invocations: use delegate_to: sec01 for host-specific tasks
- Secret seeding: use community.hashi_vault for vault_write operations
- Root token cleanup: delete /opt/vault/root-token.txt after seeding

**Key Task Blocks:**
1. Pre-flight validation (env vars, Terraform state)
2. Terraform apply sec01
3. Wait for SSH
4. Add sec01 to inventory
5. Converge step-ca + capture fingerprint
6. Issue Vault TLS cert
7. Converge Vault container
8. Initialize Vault (role: blueprints.vault.init_and_unseal)
9. Print unseal keys + operator gate
10. Unseal Vault
11. Seed secrets (AppRole, tokens, etc.)
12. Clean up root token
13. Verification + completion summary

**Dependencies:**
- Depends on: All Phase B + Phase C completed
- Blocks: D-03 (Phase 4-5 playbooks)

**Estimated Effort:**
- 1.5 days (task writing, role orchestration, gates, error handling)

---

### TICKET D-03: Phase 4-5 Orchestration Playbooks (phase-4/5-orchestrate.yml)
**Priority:** P0  
**Phase:** D  
**Story Points:** 2  
**Type:** Implementation

**Description:**  
Implement phase-4-orchestrate.yml and phase-5-orchestrate.yml playbooks for Jenkins + agent deployment.

**Acceptance Criteria:**
- [ ] Phase 4: Terraform apply docker01 → Ansible converge Jenkins + Vault integration → seed Job DSL pipelines
- [ ] Phase 5: Terraform apply agent01 → Ansible converge Jenkins agent + Vault agent → install toolchain
- [ ] Idempotent: Jenkins container skipped if .initialized marker exists
- [ ] Job DSL seeding: creates Packer, Terraform plan/apply, Ansible playbook jobs
- [ ] Agent labels: packer-build, terraform-plan, ansible-execute
- [ ] Verification: Jenkins API responding, agent appears in UI, Vault agent healthy
- [ ] Error handling + clear completion summary
- [ ] No hardcoded secrets (all from Vault)

**Deliverables:**
- `/bootstrap-container/playbooks/phase-4-orchestrate.yml`
- `/bootstrap-container/playbooks/phase-5-orchestrate.yml`

**Technical Notes:**
- Phase 4: appRole credentials retrieved from Vault (set_fact with community.hashi_vault lookup)
- Job DSL: passed as Groovy strings to Jenkins CLI
- Phase 5: agent labeled + connected to controller (JNLP port 50000)
- Toolchain install: apt-get install packer terraform ansible-core

**Key Task Blocks (Phase 4):**
1. Terraform apply docker01
2. Wait for SSH
3. Issue Jenkins TLS cert from step-ca
4. Converge Jenkins container
5. Retrieve Jenkins AppRole from Vault
6. Converge Vault plugin + AppRole config
7. Seed Job DSL pipelines (4 jobs: Packer, TF plan, TF apply, Ansible)
8. Verify Jenkins API responding
9. Completion summary

**Key Task Blocks (Phase 5):**
1. Terraform apply agent01 (LXC)
2. Wait for SSH
3. Converge Jenkins agent systemd service
4. Converge Vault agent systemd service
5. Install toolchain (Packer, Terraform, Ansible)
6. Verify agent appears in Jenkins UI
7. Verify Vault agent healthy
8. Completion summary

**Dependencies:**
- Depends on: D-01 (bootstrap script structure), D-02 (Phase 3 playbook pattern)
- Blocks: E-01 (E2E testing)

**Estimated Effort:**
- 1.5 days (two playbooks, similar pattern to Phase 3)

---

## Phase E: CI/CD & Testing

### TICKET E-01: GitHub Actions Workflow — Ansible Lint
**Priority:** P1  
**Phase:** E  
**Story Points:** 1  
**Type:** CI/CD

**Description:**  
Set up GitHub Actions workflow to run ansible-lint on all collections and playbooks.

**Acceptance Criteria:**
- [ ] Workflow file: `.github/workflows/ansible-lint.yml`
- [ ] Runs on push + pull request
- [ ] Lints all playbooks in bootstrap-container + all roles in collections
- [ ] Fails if lint errors found (blocking for PRs)
- [ ] Warnings are logged but do not fail

**Deliverables:**
- `/bootstrap-container/.github/workflows/ansible-lint.yml`
- Same workflow in each collection repo

**Technical Notes:**
- Action: `ansible-lint/ansible-lint` (GitHub Marketplace)
- Config: `.ansible-lint` file with rule exclusions if needed
- Scope: all .yml/.yaml files in playbooks/ + roles/

**Dependencies:**
- Depends on: B-01 to B-12 (roles exist)

**Estimated Effort:**
- 0.5 day (workflow setup, lint config)

---

### TICKET E-02: Molecule Test Automation in CI
**Priority:** P1  
**Phase:** E  
**Story Points:** 1  
**Type:** CI/CD

**Description:**  
Set up GitHub Actions workflow to run Molecule tests for all collections.

**Acceptance Criteria:**
- [ ] Workflow file: `.github/workflows/molecule-test.yml`
- [ ] Runs Molecule for each collection (docker driver)
- [ ] Runs on push + pull request
- [ ] Fails if Molecule test fails
- [ ] Logs test results + coverage

**Deliverables:**
- Each collection repo: `.github/workflows/molecule-test.yml`

**Technical Notes:**
- Action: custom or `geerlingguy/molecule-action` (GitHub Marketplace)
- Scenario: `default` (or multiple scenarios if needed)
- Driver: docker (lightweight, fast)

**Dependencies:**
- Depends on: B-03, B-06, B-09 (Molecule tests exist)

**Estimated Effort:**
- 1 day (workflow setup for each collection)

---

### TICKET E-03: Terraform Tests in CI (terraform validate + plan)
**Priority:** P2  
**Phase:** E  
**Story Points:** 0.5  
**Type:** CI/CD

**Description:**  
Set up GitHub Actions to run terraform validate + plan for all three repos.

**Acceptance Criteria:**
- [ ] Workflow runs on push + pull request
- [ ] Runs terraform init + validate + plan
- [ ] Fails if validation errors (blocking for PRs)

**Deliverables:**
- Each Terraform repo: `.github/workflows/terraform-validate.yml`

**Dependencies:**
- Depends on: C-01, C-02, C-03 (repos complete)

**Estimated Effort:**
- 1 day (three workflows, similar pattern)

---

### TICKET E-04: Security Scanning (Ansible syntax, Terraform security)
**Priority:** P2  
**Phase:** E  
**Story Points:** 0.5  
**Type:** CI/CD

**Description:**  
Set up GitHub Actions for security scanning: ansible-core syntax check, Terraform security checks (tfsec).

**Acceptance Criteria:**
- [ ] Workflow runs Ansible syntax check
- [ ] Workflow runs tfsec on Terraform code
- [ ] Fails if critical security issues found

**Deliverables:**
- Workflows in bootstrap-container + Terraform repos

**Technical Notes:**
- Ansible syntax: `ansible-playbook --syntax-check`
- Terraform: `tfsec` action (GitHub Marketplace)

**Dependencies:**
- Depends on: All roles + playbooks complete

**Estimated Effort:**
- 0.5 day (workflow setup)

---

### TICKET E-05: End-to-End Test Script (dry-run mode)
**Priority:** P1  
**Phase:** E  
**Story Points:** 1.5  
**Type:** Testing

**Description:**  
Create end-to-end test script that runs bootstrap-phase-3-5.sh in dry-run mode (Terraform plan only, no actual provisioning).

**Acceptance Criteria:**
- [ ] Test script: `/bootstrap-container/tests/test-bootstrap-dry-run.sh`
- [ ] Runs Terraform plan for all three repos (no apply)
- [ ] Runs Ansible playbooks with `--check` (dry-run mode)
- [ ] Validates all playbook syntax
- [ ] Captures output + reports success/failure
- [ ] Can run locally (no real infrastructure provisioning)

**Deliverables:**
- `/bootstrap-container/tests/test-bootstrap-dry-run.sh` (executable)
- Test documentation + usage instructions

**Technical Notes:**
- Use `terraform plan -out=tfplan` to capture plan
- Ansible: `ansible-playbook --check --diff`
- Mock Vault responses for secret seeding (optional, or skip seeding in --check mode)

**Dependencies:**
- Depends on: D-01, D-02, D-03 (all scripts + playbooks)

**Estimated Effort:**
- 1 day (test script, validation logic, documentation)

---

### TICKET E-06: CI/CD Integration Summary & Health Dashboard
**Priority:** P2  
**Phase:** E  
**Story Points:** 0.5  
**Type:** Documentation

**Description:**  
Document all CI/CD workflows + create status dashboard (GitHub Actions status, test coverage).

**Acceptance Criteria:**
- [ ] CI/CD documentation created: `.github/CI_CD_GUIDE.md`
- [ ] Documents all workflows (lint, Molecule, Terraform, security)
- [ ] Links to workflow status badges in README.md
- [ ] Troubleshooting guide for failed workflows

**Deliverables:**
- `/bootstrap-container/.github/CI_CD_GUIDE.md`
- Status badges in main README.md

**Dependencies:**
- Depends on: E-01 to E-05 (all workflows exist)

**Estimated Effort:**
- 0.5 day (documentation + badge setup)

---

## Phase F: Documentation & Runbooks

### TICKET F-01: Architecture LADR Documents (005–010)
**Priority:** P1  
**Phase:** F  
**Story Points:** 1  
**Type:** Documentation

**Description:**  
Document architectural decision records (LADRs 005–010) explaining key Phase 3-5 design choices.

**Acceptance Criteria:**
- [x] Single-tier step-ca root CA (not multi-tier) — documented as **LADR-004**, not LADR-005. A
  duplicate LADR-005 was independently drafted later, covering the same decision; it has been
  consolidated into LADR-004 and marked Superseded (see `decisions/LADR-005-step-ca-single-tier.md`).
- [ ] LADR-006: File-based Vault storage (not HA/Raft in v1)
- [ ] LADR-007: Jenkins containerized, agent systemd
- [ ] LADR-008: Auto-unseal via systemd script
- [ ] LADR-009: Non-root containers (org security requirement)
- [ ] LADR-010: Vault init automated, unseal is operator gate
- [ ] Each LADR includes: Context, Decision, Rationale, Consequences, Alternatives
- [ ] Documents linked from main architecture design

**Deliverables:**
- `decisions/LADR-004-step-ca-single-tier-pki.md` (single-tier step-ca decision; supersedes the
  duplicate `LADR-005-step-ca-single-tier.md`), `decisions/LADR-006.md` through `LADR-010.md`
- Cross-referenced from main design doc (section 15)

**Technical Notes:**
- Follow LADR template: Context, Decision, Rationale, Consequences, Alternatives
- Rationale: explain trade-offs + why this choice was made
- Consequences: document long-term implications (e.g., "HA can be added in Phase 6")
- Alternatives: briefly explain rejected options + why they weren't chosen

**Dependencies:**
- Depends on: Design doc (already complete), all implementation tickets

**Estimated Effort:**
- 1 day (write 6 LADRs, ~200 words each)

---

### TICKET F-02: Operator Runbook (Day 1–3 procedures)
**Priority:** P0  
**Phase:** F  
**Story Points:** 1.5  
**Type:** Documentation

**Description:**  
Write comprehensive operator runbook covering Phases 3–5 deployment procedures, gates, manual recovery, and troubleshooting.

**Acceptance Criteria:**
- [ ] Day 1 (Phase 3): Pre-execution checklist, environment setup, Terraform apply, step-ca deployment, Vault init/unseal gates, secret seeding verification
- [ ] Day 2 (Phase 4): Jenkins deployment, Vault integration, Job DSL seeding, Jenkins UI verification
- [ ] Day 3 (Phase 5): agent01 provisioning, agent deployment, toolchain installation, bootstrap retirement
- [ ] Troubleshooting guide: common failures + recovery procedures
- [ ] Manual recovery procedures: unseal Vault, reset Jenkins, re-run phases
- [ ] Break-glass procedures: operator commands if automation fails
- [ ] Document all operator gates with expected outputs
- [ ] Includes expected timelines + resource usage

**Deliverables:**
- `/docs/runbooks/OPERATOR_RUNBOOK_PHASE_3_5.md`
- Structured by day + phase
- Clear decision trees for troubleshooting

**Technical Notes:**
- Pre-execution checklist: env vars, TFC workspace, Proxmox connectivity, SSH keys
- Gates: backup confirmation, unseal verification, Jenkins UI access
- Recovery: terraform destroy + re-run, Vault recovery from backup, agent reconnection
- Break-glass: manual unseal script, Jenkins CLI commands, direct Vault API calls

**Dependencies:**
- Depends on: D-01, D-02, D-03 (orchestration scripts/playbooks)

**Estimated Effort:**
- 1 day (write procedures, examples, troubleshooting decision trees)

---

### TICKET F-03: Disaster Recovery & Backup Procedures
**Priority:** P1  
**Phase:** F  
**Story Points:** 1  
**Type:** Documentation

**Description:**  
Document backup + disaster recovery procedures for Phase 3-5 infrastructure: Vault data backup/restore, unseal key recovery, secret seeding recovery.

**Acceptance Criteria:**
- [ ] Vault backup procedure: tar.gz `/opt/vault/data`, external storage, retention policy
- [ ] Vault restore procedure: recover from backup, re-unseal, re-seed secrets
- [ ] Unseal key recovery: offline backup locations, multi-signature recovery (if 3-of-5 keys compromised)
- [ ] Secret rotation procedures: how to rotate Jenkins admin token, AppRole secret_ids, TLS certs
- [ ] Disaster scenarios documented: sec01 VM lost, Vault corrupted, Jenkins lost, agent01 lost
- [ ] Recovery timelines for each scenario
- [ ] Testing: monthly restore test procedure (non-prod)
- [ ] Automated backup: cron job for daily Vault snapshots

**Deliverables:**
- `/docs/runbooks/DISASTER_RECOVERY_PHASE_3_5.md`
- Includes backup script examples + cronjob configs

**Technical Notes:**
- Backup: `tar -czf /mnt/backup/vault-data-$(date +%Y%m%d).tar.gz /opt/vault/data/`
- Restore: `tar -xzf /mnt/backup/vault-data-YYYYMMDD.tar.gz -C /opt/vault/`
- Retention: 30 days minimum (adjust per policy)
- Testing: monthly manual restore in staging environment

**Dependencies:**
- Depends on: Design doc (section 14.5 Backup & Disaster Recovery Procedures)

**Estimated Effort:**
- 1 day (backup procedures, recovery scenarios, scripts, testing guide)

---

### TICKET F-04: API Reference & Integration Guide
**Priority:** P2  
**Phase:** F  
**Story Points:** 0.5  
**Type:** Documentation

**Description:**  
Document external API endpoints + integration points for consuming infrastructure (future microservices, teams using Vault/Jenkins).

**Acceptance Criteria:**
- [ ] Vault API endpoints documented: secret retrieval, AppRole auth, cert request
- [ ] step-ca API endpoints: certificate issuance, provisioner operations
- [ ] Jenkins API endpoints: job triggering, node management, credentials retrieval
- [ ] Integration examples: curl commands, Ansible playbook snippets, Terraform data sources
- [ ] Authentication methods: AppRole (Vault), Bearer token (Jenkins), TOFU fingerprint (step-ca)
- [ ] Rate limiting + quotas (if any)
- [ ] Deprecation policy + version support (future phases)

**Deliverables:**
- `/docs/reference/API_REFERENCE_PHASE_3_5.md`
- Example `.http` files for Postman/Insomnia testing

**Dependencies:**
- Depends on: All implementation tickets (APIs exposed)

**Estimated Effort:**
- 0.5 day (documentation + examples)

---

## Effort Summary Table

| Phase | Description | Tickets | Story Points | Est. Days | % of Total | Dependencies |
|---|---|---|---|---|---|---|
| **A** | Collection Scaffolding | A-01 to A-04 | 4 | 2 | 5% | External (terraform-proxmox-vm) |
| **B** | Ansible Collection Implementations | B-01 to B-12 | 26 | 13 | 33% | Phase A (scaffolding) |
| **C** | Terraform Repositories | C-01 to C-06 | 6 | 3 | 8% | External (terraform-proxmox-vm) |
| **D** | Bootstrap Script & Orchestration | D-01 to D-03 | 8 | 4 | 10% | Phases B + C |
| **E** | CI/CD & Testing | E-01 to E-06 | 6 | 3 | 8% | Phases B + C |
| **F** | Documentation & Runbooks | F-01 to F-04 | 4 | 2 | 5% | All implementation phases |
| **Testing & QA** | End-to-End Testing, Sign-Off | (see QA section) | 8 | 4 | 10% | All phases |
| **Integration & Closeout** | Final integration, deployment prep | (see closeout) | 6 | 3 | 8% | All phases |
| **TOTAL** | **47 Tickets** | **68 points** | **34 days** | **~9.6 weeks** | **100%** | — |

**Note:** Effort assumes:
- 1 story point ≈ 2 hours
- 1 story day ≈ 8 hours (4 points)
- Teams can work in parallel (Phase B + C run concurrently)
- QA + sign-off add ~2 weeks
- **Total project duration: ~10 weeks** (delivery + QA + closeout)

---

## Critical Path Analysis

**Longest dependency chain (sequential):**

```
terraform-proxmox-vm complete (ext. dep.)
  ↓ [1 day]
Phase A (scaffolding) — A-01 to A-04
  ↓ [2 days]
Phase B (collections) — B-01 to B-12 [13 days in parallel, ~6 days critical path]
  ↓ [concurrent with C]
Phase C (terraform) — C-01 to C-03 [3 days]
  ↓ [both complete]
Phase D (bootstrap) — D-01 to D-03 [4 days]
  ↓ [parallel with F]
Phase E (CI/CD) — E-01 to E-06 [3 days]
  ↓ [concurrent]
Phase F (docs) — F-01 to F-04 [2 days]
  ↓ [all converge]
End-to-End Testing & QA [4 days]
  ↓
Sign-Off & Deployment Ready [1 day]

CRITICAL PATH DURATION (sequential): ~1+2+6+4+3+4+1 = ~21 calendar days
WITH PARALLELIZATION: ~1+2+6 (max(B,C,E))+1 (D after B+C)+4 (testing) = ~14-16 days
REALISTIC WITH QA/SIGN-OFF: ~10 weeks (including testing, review, sign-off)
```

**Key insights:**
- **Phase B (collections) is critical path driver:** 13 story days of implementation
- **Phases B + C can run in parallel:** saves ~3 days
- **Phase D depends on both B + C:** acts as sequencing point
- **Phase E (CI/CD) can run parallel with D:** saves time
- **Phase F (docs) should start during D/E:** write in parallel
- **QA + testing adds ~2-3 weeks** to overall schedule

---

## Parallel Workstreams

### Workstream 1: Collections (Phase B)
**Team:** 2–3 Ansible engineers  
**Tickets:** B-01 to B-12 (26 points)  
**Est. Duration:** 6 days (with parallelization across 5 collections)  
**Parallelizable:** All collections + roles can be built independently (only shared test framework)

**Breakdown:**
- **Collection 1 (step_ca):** B-01, B-02, B-03 (4 points, 2 days)
- **Collection 2 (vault):** B-04, B-05, B-06, B-07 (5.5 points, 3 days)
- **Collection 3 (jenkins):** B-08, B-09, B-10 (4.5 points, 2.5 days)
- **Collection 4 (jenkins_integrations):** B-11 (1.5 points, 1 day)
- **Collection 5 (vault_integrations):** B-12 (1.5 points, 1 day)

**Critical dependencies within workstream:**
- step_ca: B-01 → B-02 (container must exist before issuing certs)
- vault: B-04 → B-05 (container must exist before init)
- jenkins: B-08 → B-09 (container must exist before testing)

**Team allocation:** Assign roles by collection; one engineer can handle 1–2 collections in parallel

---

### Workstream 2: Terraform Repositories (Phase C)
**Team:** 1–2 Infrastructure engineers  
**Tickets:** C-01 to C-06 (6 points)  
**Est. Duration:** 3 days (highly parallelizable, similar pattern)

**Breakdown:**
- **Repo 1 (sec01):** C-01, C-04 (1.5 + 0.5 = 2 points, 1 day)
- **Repo 2 (docker01):** C-02, C-05 (1.5 + 0.5 = 2 points, 1 day)
- **Repo 3 (agent01):** C-03, C-06 (2 + 0.5 = 2.5 points, 1.5 days)

**Parallelizable:** All three repos independent; can be built simultaneously

**Team allocation:** One engineer can write sec01 repo, parallel engineer writes docker01 + agent01

---

### Workstream 3: Bootstrap Script & Orchestration (Phase D)
**Team:** 1 Senior Ansible engineer  
**Tickets:** D-01, D-02, D-03 (8 points)  
**Est. Duration:** 4 days (sequential)

**Critical path:** D-01 → D-02 → D-03 (each phase depends on previous)

**Dependency:** Depends on Phase B + C complete

---

### Workstream 4: CI/CD & Testing (Phase E)
**Team:** 1 DevOps engineer  
**Tickets:** E-01 to E-06 (6 points)  
**Est. Duration:** 3 days

**Breakdown:**
- Workflows (E-01, E-02, E-03, E-04): 2.5 points, ~1.5 days (can parallelize workflow setup)
- E2E test (E-05): 1.5 points, 1 day (depends on D-01 complete)
- Summary (E-06): 0.5 points, 0.5 day

**Can run parallel with Phase D** (workflows don't block orchestration script)

---

### Workstream 5: Documentation (Phase F)
**Team:** 1 Technical writer + subject matter experts  
**Tickets:** F-01 to F-04 (4 points)  
**Est. Duration:** 2 days

**Breakdown:**
- LADRs (F-01): 1 point, 1 day
- Operator runbook (F-02): 1.5 points, 1 day
- Disaster recovery (F-03): 1 point, 1 day
- API reference (F-04): 0.5 points, 0.5 day

**Can run parallel with Phases D + E** (documentation is independent; needs SME input from other teams)

---

## Resource Allocation

**Recommended team composition:**

| Role | Count | Effort | Workstreams | Timeline |
|---|---|---|---|---|
| **Ansible Expert** | 2–3 | 13d (Phase B) | Collections | Weeks 2–4 |
| **Infrastructure/Terraform Engineer** | 1–2 | 3d (Phase C) | Terraform repos | Weeks 2–3 (parallel with B) |
| **DevOps/CI-CD Engineer** | 1 | 3d (Phase E) | CI/CD workflows | Weeks 3–5 (parallel with B+D) |
| **Senior Ansible Architect** | 1 | 4d (Phase D) | Bootstrap script | Weeks 4–5 (after B+C) |
| **Technical Writer** | 1 | 2d (Phase F) | Documentation | Weeks 5–7 (parallel with D+E) |
| **QA Engineer** | 1 | 4d (Testing) | E2E testing + sign-off | Weeks 6–9 |
| **Tech Lead/PM** | 1 | Throughout | Coordination + reviews | Weeks 1–10 |

**Total team:** ~8 people  
**Peak concurrent effort:** Weeks 3–5 (5 people working in parallel)  
**Actual delivery estimate:** 10 weeks (with QA + sign-off)

---

## Quality Gates

### Phase A Completion Gate
**Criteria:**
- [ ] All 5 collections exist on disk + are writable
- [ ] Role inventory document complete (12 roles identified)
- [ ] External dependencies listed + validated
- [ ] terraform-proxmox-vm module verified complete
- [ ] No blockers identified for Phase B

**Approval:** Tech lead sign-off

---

### Phase B Completion Gate
**Criteria:**
- [ ] All 12 roles implemented with complete task bodies
- [ ] Molecule test playbooks exist + pass locally
- [ ] `ansible-lint` passes on all roles + playbooks
- [ ] No hardcoded Vernify-specific values (all parameterized)
- [ ] All sensitive data marked `no_log: true`
- [ ] Role READMEs + schema comments complete

**Approval:** Ansible expert + QA engineer sign-off

---

### Phase C Completion Gate
**Criteria:**
- [ ] Three terraform repos exist + terraform validate passes
- [ ] terraform plan shows expected resource creation (no errors)
- [ ] All variables documented + .tfvars.example provided
- [ ] Remote backend (TFC) configured + functional
- [ ] GitHub Actions workflows run successfully

**Approval:** Infrastructure engineer + Tech lead

---

### Phase D Completion Gate
**Criteria:**
- [ ] bootstrap-phase-3-5.sh runs in dry-run mode (terraform plan + ansible --check)
- [ ] All orchestration playbooks syntax-check passes
- [ ] Phase 3/4/5 playbooks tested with mock Vault (Molecule)
- [ ] Operator gates implemented + functional
- [ ] Error handling + rollback instructions documented
- [ ] Completion summary prints correctly

**Approval:** Senior architect + Tech lead

---

### Phase E Completion Gate
**Criteria:**
- [ ] All GitHub Actions workflows pass
- [ ] Molecule tests pass for all collections
- [ ] ansible-lint passes on all playbooks
- [ ] Terraform validate + plan pass for all repos
- [ ] E2E dry-run test succeeds
- [ ] CI/CD documentation complete

**Approval:** DevOps engineer + QA engineer

---

### Phase F Completion Gate
**Criteria:**
- [ ] All 6 LADRs written + cross-linked from main design
- [ ] Operator runbook complete + reviewed by SMEs
- [ ] Disaster recovery procedures tested (restore process validated)
- [ ] API reference + integration guide complete
- [ ] No clarity gaps in documentation (QA review)

**Approval:** Technical writer + QA engineer

---

### End-to-End Testing Gate
**Criteria:**
- [ ] bootstrap-phase-3-5.sh runs successfully on Vernify infrastructure (Phase 3)
- [ ] sec01 (Vault) running + unsealed + secrets seeded
- [ ] Phase 4 completes: docker01 (Jenkins) deployed + connected to Vault
- [ ] Phase 5 completes: agent01 (Jenkins agent) deployed + connected + toolchain installed
- [ ] All operator gates work (backup confirmation, health checks)
- [ ] Rollback procedures tested (delete VM, re-run phase)
- [ ] No operator confusion (runbook clear + comprehensive)

**Approval:** QA engineer + Tech lead + Operators

---

## Risk & Mitigation by Work Item

### Risk: terraform-proxmox-vm Module Incomplete (External Dependency)
**Impact:** Blocks all three terraform repos (C-01 to C-03)  
**Likelihood:** Medium  
**Mitigation:** Verify module completion in A-03; escalate immediately if incomplete  
**Contingency:** Use Proxmox provider directly (no module) if needed; update terraform repos to use provider instead

---

### Risk: Collection Scaffolding Missing Roles
**Impact:** Phase B implementation blocked  
**Likelihood:** Low (collections assumed to exist)  
**Mitigation:** Audit all 5 collections in A-01  
**Contingency:** Create missing roles from scratch; add to backlog

---

### Risk: Vault Init Output Parsing (JSON) Fails
**Impact:** Unseal keys not captured; Phase 3 cannot unseal  
**Likelihood:** Low  
**Mitigation:** Test JSON parsing in B-05 Molecule tests; handle version differences in Vault output  
**Contingency:** Manual key extraction; operator runs unseal script manually

---

### Risk: TLS Certificate SANs Incorrect
**Impact:** Services unreachable (cert validation fails)  
**Likelihood:** Medium  
**Mitigation:** P0-5 SAN validation mandatory in B-02 (step certificate inspect)  
**Contingency:** Regenerate cert with correct SANs; troubleshooting guide in operator runbook (F-02)

---

### Risk: Bootstrap Script Idempotency Failures
**Impact:** Phase reruns skip completed steps incorrectly; state drift  
**Likelihood:** Medium  
**Mitigation:** Thorough testing in E-05 (dry-run test); test all idempotency checks  
**Contingency:** Manual rollback (terraform destroy + re-run with --force-recreate flag)

---

### Risk: Operator Gate Confusion (Unseal Key Backup)
**Impact:** Operator skips backup, keys lost, Vault unrecoverable  
**Likelihood:** Medium  
**Mitigation:** Clear, large warning text in Phase 3 orchestration + operator runbook (F-02)  
**Contingency:** Recovery from backup (if operator has backup); KMS auto-seal in Phase 6 removes this risk

---

### Risk: Proxmox Connectivity Failure During Deployment
**Impact:** Terraform apply hangs or fails  
**Likelihood:** Low  
**Mitigation:** Pre-flight connectivity check in D-01 (test Proxmox API)  
**Contingency:** Operator retries after fixing network; E2E test (E-05) catches connectivity issues

---

### Risk: Container Image Pull Timeout (registry unavailable)
**Impact:** Container deployment fails  
**Likelihood:** Low  
**Mitigation:** Pin image versions in collection defaults (B-01 to B-12)  
**Contingency:** Retry logic in roles; Phase 6: implement local image registry mirror

---

## Sequencing & Timeline

### Week 1–2: Foundation
- **A-01 to A-04:** Collection scaffolding + dependency audit (4 points, 2 days)
- **Parallel:** Start Phase B (collections) infrastructure setup

### Week 2–4: Core Implementation (Parallel Tracks)
- **Track 1 (Ansible):** Phase B, all 12 roles (26 points, ~6 days critical path with parallelization)
- **Track 2 (Infrastructure):** Phase C, three terraform repos (6 points, 3 days)
- **Track 3 (DevOps):** CI/CD setup + Molecule framework (E-01 to E-03, parallel with Phases B+C)

### Week 5: Orchestration
- **D-01 to D-03:** Bootstrap script + phase playbooks (8 points, 4 days; depends on B+C complete)
- **Parallel:** Phase F documentation start (F-01 LADRs)

### Week 6: Testing & Validation
- **E-04 to E-06:** Security scanning + E2E test + CI/CD summary (2 points, 1 day)
- **F-02 to F-04:** Operator runbook + DR procedures + API reference (2.5 points, 1.5 days)

### Week 7–9: End-to-End Testing & QA
- **E2E Dry-Run:** bootstrap-phase-3-5.sh --dry-run on Vernify (1 day)
- **Live Testing:** Phase 3–5 deployment on Vernify infrastructure (3–4 days)
- **QA Review:** Acceptance criteria validation, operator testing (3–4 days)
- **Sign-Off:** Final approval from tech lead, operators, stakeholders (1 day)

### Week 10: Deployment & Handoff
- **Production Deployment:** Final bootstrap run (1 day)
- **Knowledge Transfer:** Operator training, documentation review (1 day)
- **Go/No-Go Decision:** Ready for Phase 6 work

---

## Hand-Off Brief for Implementation Team

### What Needs to Be Built

**Deliverables (47 tickets across 6 phases):**

1. **Phase A (Weeks 1–2):** Verify collections exist, audit dependencies, validate external module
2. **Phase B (Weeks 2–4):** Implement 12 Ansible roles across 5 collections with full test coverage
3. **Phase C (Weeks 3–4):** Build 3 Terraform repos for sec01/docker01/agent01 provisioning
4. **Phase D (Weeks 5–6):** Write unified bootstrap script + orchestration playbooks (all 3 phases)
5. **Phase E (Weeks 6–7):** Set up GitHub Actions CI/CD + Molecule testing framework
6. **Phase F (Weeks 7–8):** Write LADRs + operator runbook + disaster recovery procedures

### Order to Build It

**Critical path (don't deviate):**
```
Phase A (scaffold) → Phase B (collections) + Phase C (terraform) → Phase D (bootstrap) → Phase E (CI) → Phase F (docs) → Testing/QA
```

**Parallel opportunities (maximize parallelization):**
- **Weeks 2–4:** B + C run in parallel (collections ≠ terraform repos)
- **Weeks 3–5:** E-01 to E-03 (CI setup) can start while B + C ongoing
- **Weeks 5–7:** F documentation can start during D + E (doesn't block anything)

### Dependencies to Watch

1. **External Blocker:** terraform-proxmox-vm module (Phase 2). Verify in A-03 immediately.
2. **Internal Blockers:**
   - Phase A must complete before Phase B (collection scaffolding)
   - Phase B + C must both complete before Phase D (bootstrap depends on both)
   - Phase D must complete before Phase E (E2E testing needs bootstrap script)
3. **No Blockers:** F (documentation) can run independently in parallel

### Tests That Must Pass

**Quality gates before moving to next phase:**

| Gate | Phase | Criteria |
|---|---|---|
| **A** | Scaffolding | All 5 collections exist + terraform-proxmox-vm verified |
| **B** | Collections | Molecule tests pass + ansible-lint passes + no hardcoded values |
| **C** | Terraform | terraform validate + terraform plan pass (no errors) |
| **D** | Bootstrap | Dry-run (terraform plan + ansible --check) succeeds end-to-end |
| **E** | CI/CD | GitHub Actions workflows pass + E2E test succeeds |
| **F** | Docs | Runbook reviewed + no clarity gaps |
| **QA** | Live | Phase 3–5 deployment succeeds on Vernify + operator validates |

### How to Know You're Done

**Delivery is complete when:**

1. ✅ All 47 work tickets are accepted + merged to main
2. ✅ All GitHub Actions workflows passing on main branch
3. ✅ Molecule tests pass locally + in CI for all collections
4. ✅ terraform validate passes for all 3 repos
5. ✅ bootstrap-phase-3-5.sh runs end-to-end in dry-run mode (no errors)
6. ✅ Live Phase 3–5 deployment succeeds on Vernify infrastructure
7. ✅ All secrets seeded in Vault + Jenkins AppRole auth working
8. ✅ agent01 appears in Jenkins UI as healthy node
9. ✅ Operator runbook tested + operators comfortable with procedures
10. ✅ Sign-off from tech lead + QA engineer + operators

**Success = Infrastructure self-hosted, bootstrap container retired, ready for Phase 6 work**

---

## Appendix: Ticket Reference Table

| Ticket ID | Title | Phase | Points | Days | Type | Assigned To |
|---|---|---|---|---|---|---|
| A-01 | Audit Collections & Role Inventory | A | 1 | 1 | Infra | Tech Lead |
| A-02 | Set Up External Dependencies | A | 1 | 1 | Infra | Tech Lead |
| A-03 | Verify terraform-proxmox-vm Module | A | 1 | 0.5 | Infra | Tech Lead |
| A-04 | Create Ansible Playbook Template | A | 1 | 0.5 | Infra | Ansible Expert |
| B-01 | blueprints.step_ca — container_server | B | 2 | 1.5 | Impl. | Ansible Expert 1 |
| B-02 | blueprints.step_ca — issue_certificate | B | 2 | 1.5 | Impl. | Ansible Expert 1 |
| B-03 | blueprints.step_ca — Molecule Tests | B | 1 | 1 | Test | QA Engineer |
| B-04 | blueprints.vault — container_server | B | 2 | 1.5 | Impl. | Ansible Expert 2 |
| B-05 | blueprints.vault — init_and_unseal | B | 2 | 1.5 | Impl. | Ansible Expert 2 |
| B-06 | blueprints.vault — Molecule Tests | B | 1 | 1 | Test | QA Engineer |
| B-07 | blueprints.vault — Documentation | B | 0.5 | 0.5 | Doc. | Tech Writer |
| B-08 | blueprints.jenkins — container_server | B | 2 | 1.5 | Impl. | Ansible Expert 3 |
| B-09 | blueprints.jenkins — Molecule Tests | B | 1 | 1 | Test | QA Engineer |
| B-10 | blueprints.jenkins — Agent Role | B | 1.5 | 1 | Impl. | Ansible Expert 3 |
| B-11 | blueprints.jenkins_integrations — vault_auth | B | 1.5 | 1 | Impl. | Ansible Expert 1 |
| B-12 | blueprints.vault_integrations — vault_agent | B | 1.5 | 1 | Impl. | Ansible Expert 2 |
| C-01 | terraform-sec01-deploy | C | 1.5 | 1 | Infra | Terraform Engineer 1 |
| C-02 | terraform-docker01-deploy | C | 1.5 | 1 | Infra | Terraform Engineer 2 |
| C-03 | terraform-agent01-deploy | C | 2 | 2 | Infra | Terraform Engineer 1 |
| C-04 | terraform-sec01-deploy CI/CD | C | 0.5 | 0.5 | CI/CD | DevOps Engineer |
| C-05 | terraform-docker01-deploy CI/CD | C | 0.5 | 0.5 | CI/CD | DevOps Engineer |
| C-06 | terraform-agent01-deploy CI/CD | C | 0.5 | 0.5 | CI/CD | DevOps Engineer |
| D-01 | Unified Bootstrap Script | D | 3 | 2 | Impl. | Senior Architect |
| D-02 | Phase 3 Orchestration Playbook | D | 2 | 1.5 | Impl. | Senior Architect |
| D-03 | Phase 4–5 Orchestration Playbooks | D | 2 | 1.5 | Impl. | Senior Architect |
| E-01 | GitHub Actions — ansible-lint | E | 1 | 0.5 | CI/CD | DevOps Engineer |
| E-02 | Molecule Test Automation | E | 1 | 1 | CI/CD | DevOps Engineer |
| E-03 | Terraform Tests in CI | E | 0.5 | 0.5 | CI/CD | DevOps Engineer |
| E-04 | Security Scanning | E | 0.5 | 0.5 | CI/CD | DevOps Engineer |
| E-05 | E2E Test Script (dry-run) | E | 1.5 | 1 | Test | QA Engineer |
| E-06 | CI/CD Summary & Dashboard | E | 0.5 | 0.5 | Doc. | DevOps Engineer |
| F-01 | Architecture LADRs (005–010) | F | 1 | 1 | Doc. | Tech Lead/Architect |
| F-02 | Operator Runbook | F | 1.5 | 1 | Doc. | Tech Writer |
| F-03 | Disaster Recovery Procedures | F | 1 | 1 | Doc. | Tech Writer/SME |
| F-04 | API Reference & Integration Guide | F | 0.5 | 0.5 | Doc. | Tech Writer |
| **QA-01** | **End-to-End Testing (Live)** | **QA** | **4** | **4** | **Test** | **QA Engineer** |
| **QA-02** | **Sign-Off & Approval** | **QA** | **2** | **1** | **Review** | **Tech Lead** |
| **TOTAL** | **47 Tickets** | — | **68 points** | **34 days** | — | **~8 people** |

---

## Success Criteria Summary

✅ **Delivery is complete when:**

1. All 47 work tickets implemented + tested + merged to main
2. All GitHub Actions workflows passing
3. All Molecule tests passing locally + in CI
4. bootstrap-phase-3-5.sh runs end-to-end (dry-run) without errors
5. Live Phase 3–5 deployment succeeds on Vernify infrastructure
6. Vault unsealed, secrets seeded, AppRole auth working
7. Jenkins deployed, Job DSL pipelines seeded, agents connected
8. agent01 running with toolchain installed + healthy in Jenkins
9. Operator runbook tested + team trained + comfortable with procedures
10. Tech lead + QA engineer + operators sign-off ✓

**Estimated project completion: Week 10 (10 weeks end-to-end with testing + QA)**

---

**Document Status:** FINAL  
**Last Updated:** 28 June 2026  
**Maintained By:** Delivery Planning (Stage 6)  
**Next Phase:** Stage 7 Implementation
