# Design Referee Decision
## Vernify Phase 3-5 Technical Architecture Design

**Date:** 27 June 2026  
**Review Panel:** Security Engineering, Infrastructure/Platform, Network Engineering, Technical Strategy, Reliability Engineering  
**Architect:** [Awaiting revision]  
**Referee Status:** REQUIRES REVISIONS

---

## DECISION

**REQUIRE REVISIONS**

The technical architecture design for Vernify Phase 3-5 is fundamentally sound and demonstrates thoughtful orchestration, security posture, and operational planning. However, **6 blocking issues (P0)** and **12 must-fix issues before Phase 7a (P1)** prevent immediate approval. The design is missing critical implementation artifacts, contains framework misalignment, and lacks documented network/security controls.

**Path Forward:** Architect revises design addressing all P0 findings and documents P1 mitigation strategies. Abbreviated re-review (1-2 hours, changed sections only) follows. Then proceed to Stage 5 human approval gate.

**Estimated Revision Effort:** 3–5 days (focused on blocking issues)  
**Risk to Phase 3-5 Timeline:** Low (no architectural redesign required; issues are implementable within existing design)  
**Recommendation:** Proceed with revisions in parallel with Stage 7a implementation; code reviews will serve as secondary validation gate.

---

## EXECUTIVE SUMMARY

### Findings Consolidation

- **Original Report:** 61 findings across 5 domains
- **Consolidated Clusters:** 35 distinct findings grouped into 24 consolidated issues
- **Deduplication Impact:** 8 findings were exact duplicates (same root cause, multiple reviewers) → merged with cross-domain traceability noted

### Severity Breakdown (Consolidated)

| Severity | Count | P0 (Blocking) | P1 (Must-fix Phase 7a) | P2 (Defer Phase 7) | P3 (Phase 6+) | Notes |
|----------|-------|---------------|-----------------------|------------------|---------------|-------|
| Critical | 12 | 6 | 4 | 2 | 0 | Shipping-blocking or security-critical |
| High | 27 | 0 | 8 | 14 | 5 | Quality gaps, unmitigated risks |
| Medium | 19 | 0 | 0 | 12 | 7 | Tech debt, documentation |
| Low | 3 | 0 | 0 | 2 | 1 | Enhancements |
| **TOTAL** | **61** | **6** | **12** | **28** | **13** | — |

### Approval Readiness

- **P0 Issues (Blocking):** 6 findings that prevent Stage 5 approval
  - Missing implementation artifacts (3): orchestration playbooks, terraform repos, Vault init_and_unseal role
  - Network configuration (2): DNS resolution, TLS certificate SANs
  - Framework misalignment (1): LADR-003 violation + existing collection overwrites
- **P1 Issues (Must-fix before Phase 7a):** 12 findings that must be resolved before implementation starts
- **P2 Issues (Defer to Phase 7):** 28 findings acceptable to address during implementation with documented mitigation
- **P3 Issues (Defer to Phase 6+):** 13 findings acceptable to defer beyond Phase 5 delivery

### Overall Assessment

**Recommendation:** REQUIRE REVISIONS → APPROVE WITH CONDITIONS

Design is approved with conditions: architect revises addressing P0 findings (3–5 days), abbreviated re-review passes (no new Critical issues), then proceeds to Stage 5 human gate. P1 findings require documented mitigation strategies (not necessarily resolution) before Phase 7a implementation begins. P2/P3 findings are acceptable to defer with clear owner, due date, and tracking.

---

## CONSOLIDATED FINDINGS WITH TRACEABILITY

### BLOCKING ISSUES (P0) — Must Fix Before Stage 5 Gate

#### Issue 1: Missing Orchestration Playbooks
- **Raised by:** Infrastructure, Reliability
- **Severity:** CRITICAL
- **Priority:** P0 (Blocking)
- **Type:** MISSING ARTIFACT
- **Finding IDs:** INFRA-C3, REL-C1, REL-C2
- **Root Cause:** Design provides detailed YAML specifications (lines 708–1078) but executables do not exist. Phase 3, 4, 5 orchestration playbooks are referenced but not committed to bootstrap-container repository.
- **Design Reference:** Section 5 (Phase 3 playbook YAML), Section 6 (Phase 4 playbook), Section 7 (Phase 5 playbook)
- **Related FR/NFR:** FR-1 (Vault init + unseal), FR-2 (Secret seeding), NFR-1 (Idempotency)
- **Design Assumption:** Playbooks exist and are runnable by Stage 7a
- **Impact:** Phase 3 execution fails immediately (playbook not found). No orchestration possible without executable files.
- **Recommendation:** 
  - Architect translates YAML design specs into executable playbooks (phase-3-orchestrate.yml, phase-4-orchestrate.yml, phase-5-orchestrate.yml)
  - Commit to bootstrap-container/playbooks/ directory
  - Include all gates, error handling, and pre/post-flight checks from design
  - Add to git tag/release as part of Phase 7a deliverables
- **Effort:** 2 days (design is detailed; implementation is direct translation)
- **Success Criteria:** 
  - Three playbooks exist in bootstrap-container/playbooks/
  - All sections from design (gates, variables, tasks, handlers) are present
  - Molecule dry-run executes without syntax errors
  - Playbooks reference correct inventory, group variables, and dynamically generated facts

#### Issue 2: Missing Terraform Deployment Repositories
- **Raised by:** Infrastructure
- **Severity:** CRITICAL
- **Priority:** P0 (Blocking)
- **Type:** MISSING ARTIFACT
- **Finding IDs:** INFRA-C2
- **Root Cause:** Design references terraform-build01-deploy and terraform-agent01-deploy repositories (Sections 4.2–4.3). Filesystem search confirms only terraform-sec01-deploy exists. build01 and agent01 repos are missing.
- **Design Reference:** Section 4.2 (build01 VM provisioning), Section 4.3 (agent01 VM provisioning)
- **Related FR/NFR:** FR-4 (Jenkins deployment), FR-5 (Jenkins agent deployment)
- **Impact:** Phase 4 (build01) and Phase 5 (agent01) orchestration fails (terraform apply against non-existent module). No Jenkins or agent infrastructure can be created.
- **Recommendation:**
  - Architect creates terraform-build01-deploy repo using terraform-sec01-deploy as template
  - Architect creates terraform-agent01-deploy repo using same pattern
  - Both repos follow iac-foundry naming and module structure
  - Include cloud-init configuration, network settings, security groups aligned with Phase 3-5 design
- **Effort:** 2 days (1 day each; design detail sufficient to implement directly)
- **Success Criteria:**
  - Both repos exist in /workspace/vernify/
  - `terraform plan` executes without errors
  - VMs provisioned match design specs (network, memory, disk, cloud-init)
  - Repos committed to git by Stage 7a

#### Issue 3: Missing blueprints.vault.init_and_unseal Role
- **Raised by:** Infrastructure
- **Severity:** CRITICAL
- **Priority:** P0 (Blocking)
- **Type:** MISSING ARTIFACT
- **Finding IDs:** INFRA-C1
- **Root Cause:** Design specifies init_and_unseal role in detail (Section 3.2, lines 245–297) but role does not exist in ansible-collection-vault repository. Spot check shows only container_server, client, systemd_server roles present.
- **Design Reference:** Section 3.2 (init_and_unseal role spec)
- **Related FR/NFR:** FR-1 (Vault init + unseal), NFR-1 (Idempotency)
- **Impact:** Phase 3 playbook attempts to include role; Ansible fails with "role not found". Vault cannot be initialized.
- **Recommendation:**
  - Architect implements init_and_unseal role in ansible-collection-vault
  - Role tasks: (1) run `vault operator init`, (2) capture + print unseal keys, (3) write to files with correct perms, (4) generate unseal script, (5) create systemd unit, (6) handle idempotency (skip if already initialized)
  - Add comprehensive error handling and Molecule tests (at least 3 test scenarios: init, re-run idempotent, re-init with backup)
  - Document role inputs (unseal_key_backup_location, unseal_storage_backend) and outputs (unseal_keys, root_token)
- **Effort:** 1–2 days (design provides full implementation spec)
- **Success Criteria:**
  - Role exists at roles/init_and_unseal/ in ansible-collection-vault
  - Molecule test suite runs; all scenarios pass
  - Role is idempotent (re-run produces no changes)
  - Role handles pre-existing Vault state gracefully

#### Issue 4: DNS Resolution Strategy Not Documented; Hostnames Cannot Resolve
- **Raised by:** Network, Infrastructure (duplicate finding: NET-C1, INFRA-H2)
- **Severity:** CRITICAL
- **Priority:** P0 (Blocking)
- **Type:** INCOMPLETE SPEC
- **Finding IDs:** NET-C1, INFRA-H2
- **Root Cause:** Design assumes .vernify.internal FQDNs throughout (sec01.vernify.internal, vault.sec01.internal, jenkins.build01.internal) but provides no DNS resolution mechanism. Terraform sets search domain to "vernify.com" (contradiction). No DNS server documented.
- **Design Reference:** Section 2.3 (lines 104–137), Section 3.1 (hostname references), Phase 3 playbook (lines 814–817)
- **Related FR/NFR:** FR-1 (Vault init), FR-3 (step-ca TLS), FR-4 (Jenkins deployment)
- **Impact:** Phase 3 will hang/timeout on first hostname resolution attempt (e.g., curl https://sec01.vernify.internal:9000). No internal communication possible without DNS.
- **Recommendation:**
  - Architect clarifies DNS strategy: (Option A) dnsmasq on Proxmox host with internal zone, (Option B) /etc/hosts on all VMs, (Option C) external DNS with internal zone forwarding
  - Documents choice in design (new section: "Network Configuration Prerequisites")
  - If Option B: adds Ansible task to Phase 3 playbook to populate /etc/hosts on all VMs
  - If Option A/C: documents DNS server IP, how to verify it works, and includes pre-flight check in Phase 3
  - Corrects Terraform search domain to match DNS zone (e.g., vernify.internal)
  - Adds pre-flight validation task to Phase 3: `getent hosts sec01.vernify.internal` verifies resolution
- **Effort:** 1 day (documentation + 1–2 Ansible tasks if using /etc/hosts option)
- **Success Criteria:**
  - Design includes "DNS Configuration" section with choice + rationale
  - Terraform variable matches DNS zone
  - Phase 3 playbook includes DNS resolution verification before proceeding
  - All internal hostnames resolve successfully

#### Issue 5: TLS Certificate SAN Mismatch; Hostname Validation Will Fail
- **Raised by:** Security, Network (duplicate finding: SEC-H2, NET-C2)
- **Severity:** CRITICAL
- **Priority:** P0 (Blocking)
- **Type:** INCOMPLETE SPEC
- **Finding IDs:** SEC-H2, NET-C2
- **Root Cause:** Certificate SANs (lines 195–196, 814–817) list ["vault", "vault.sec01", "192.168.22.51"] but clients connect to "vault.sec01.vernify.internal" (FQDN not in SANs). Modern TLS clients validate hostname against SAN list; mismatch causes handshake failure.
- **Design Reference:** Section 3.1 (Vault TLS config), Phase 3 playbook (certificate issuance)
- **Related FR/NFR:** FR-1 (Vault TLS), FR-3 (step-ca integration), NFR-2 (Security)
- **Impact:** Phase 4 (when Jenkins attempts TLS connection to Vault) fails with certificate validation error. TLS handshakes will not complete.
- **Recommendation:**
  - Architect updates certificate SANs to include FQDN: ["vault", "vault.sec01", "vault.sec01.vernify.internal", "vault.sec01.internal", "192.168.22.51"]
  - Updates CN (Common Name) to FQDN (e.g., "vault.sec01.vernify.internal")
  - Documents this in Section 3.1 (step-ca configuration)
  - Adds post-issuance validation task to Phase 3: `step certificate inspect` verifies SANs match expected list
  - Applies same SAN fix for all other certificates (Jenkins, agents, if any client certs issued)
- **Effort:** 1 day (SAN update + validation tasks)
- **Success Criteria:**
  - Certificate SANs include FQDN (vault.sec01.vernify.internal)
  - CN is set to FQDN
  - Phase 3 playbook includes `step certificate inspect` post-issuance verification
  - TLS handshake test (curl with hostname validation) succeeds in Phase 3

#### Issue 6: Framework Misalignment — LADR-003 Violation + Existing Collection Overwrite Risk
- **Raised by:** Strategy, Architecture (consolidates: STRAT-C1, STRAT-C3)
- **Severity:** CRITICAL
- **Priority:** P0 (Blocking)
- **Type:** ARCHITECTURE RISK + FRAMEWORK MISALIGNMENT
- **Finding IDs:** STRAT-C1, STRAT-C3
- **Root Cause:** 
  - **LADR-003 Violation:** Design conflates platform-layer (orchestration) with collection-layer (roles) concerns. Section 3.3 shows secret seeding as orchestration playbook tasks using community.hashi_vault lookups; unclear which tasks belong in roles vs. playbook. This violates LADR-003 ("Secret Retrieval Uses community.hashi_vault") which specifies secret retrieval at platform layer only.
  - **Existing Collection Risk:** Design does NOT acknowledge existing collections (ansible-collection-vault, step-ca, jenkins) already published/in-use. No migration path documented. Risk: Phase 3-5 implementation will overwrite existing versions silently, breaking other consumer orgs (IOTel, Saicom).
- **Design Reference:** Section 3 (role specs), LADR-003 (secret retrieval boundary)
- **Related FR/NFR:** NFR-4 (Portability / org-neutral collections)
- **Impact:** (1) Collections become Vernify-specific, not reusable by other orgs. (2) Existing collection versions used by other orgs silently overwritten, breaking their deployments.
- **Recommendation:**
  - Architect audits existing collection state (versions, what's published to Ansible Galaxy)
  - Clarifies role/platform boundary: roles deploy infrastructure (containers, config); orchestration playbook handles all secret management (init, unseal, seeding, AppRole wiring)
  - Creates "Existing Collection Compatibility" section documenting: version impact, breaking changes (if any), migration path
  - If breaking changes required: bumps collection version, documents in design, ensures other consumers are notified before merge
  - Removes hardcodes from role defaults (e.g., CN="Vernify Root CA" should not be role default); orchestration playbook supplies org-specific values
- **Effort:** 2 days (audit + design doc updates + possible version bump)
- **Success Criteria:**
  - Design includes "Framework Alignment" section confirming LADR-003 compliance
  - Design includes "Existing Collection Compatibility" section with version audit + migration path
  - Roles have no hardcoded Vernify-specific values (org name, domains, IPs come from playbook vars)
  - Collection version numbers pinned + documented

---

### MUST-FIX BEFORE PHASE 7A (P1) — 12 Issues

#### P1-1: Unseal Key Backup Enforcement — No Verification Mechanism
- **Raised by:** Security
- **Severity:** CRITICAL
- **Priority:** P1
- **Type:** OPERATIONAL + SECURITY
- **Finding IDs:** SEC-C1
- **Root Cause:** Operator manually backs up unseal keys; prompt-based confirmation only ("Have you backed up unseal keys? yes/no"). No enforcement mechanism. If operator confirms without actually backing up, Vault is unrecoverable on host failure.
- **Design Reference:** Section 11.2 (lines 5.2)
- **Related FR/NFR:** Risk-1 (Operator loses unseal keys)
- **Mitigation Strategy:** 
  - Phase 3-5: Implement prompt-based verification (ask operator to confirm backup path, verify file exists)
  - Phase 6: Implement automated encrypted backup to S3
  - Document manual backup procedure in operator runbook
  - Accept: operator discipline required for Phase 3-5; technical enforcement in Phase 6
- **Effort to P1 compliance:** 1 day (verification script)
- **Recommendation for Architect:**
  - Add pre-unseal backup validation: prompt operator for backup path, verify directory exists + is writable
  - After Vault init, copy unseal keys to backup location, verify with checksum
  - Document procedure in operator runbook with clear step-by-step
  - Flag for Phase 6: implement automated S3 backup with encryption
- **Success Criteria:** 
  - Phase 3 playbook includes backup verification task
  - Operator runbook documents backup procedure
  - Backup verification is automated (not just manual checklist)

#### P1-2: Root Token Written to Disk — Privilege Escalation Window
- **Raised by:** Security
- **Severity:** CRITICAL
- **Priority:** P1
- **Type:** SECURITY HARDENING
- **Finding IDs:** SEC-C2
- **Root Cause:** Root token written to `/opt/vault/root-token.txt` (0600 perms). If host compromised (container escape, kernel RCE), attacker reads token before deletion. Persistent disk storage is exposure window.
- **Design Reference:** Section 3.2 (lines 266–267), Section 10.1 (lines 1657–1666)
- **Related FR/NFR:** NFR-2 (Security / non-root containers)
- **Mitigation Strategy:**
  - Phase 3-5: Move token to tmpfs (/run/secrets), add auto-delete timer (10 min after generation)
  - Phase 6: Implement break-glass recovery procedure (do not write root token at all; use recovery key instead)
- **Recommendation for Architect:**
  - Update init_and_unseal role: write root token to /run/secrets/vault-root-token (tmpfs, temporary)
  - Add systemd timer to delete token 10 minutes after creation (or when secret seeding completes)
  - Document this security measure in design + operator runbook
  - Document break-glass: if root token needed after tmpfs expiry, use recovery key for re-init
- **Effort to P1 compliance:** 1 day
- **Success Criteria:**
  - Root token written to tmpfs, not persistent disk
  - Auto-deletion timer implemented (10 min)
  - Operator runbook documents break-glass recovery

#### P1-3: AppRole secret_id Has No TTL; Credentials Never Rotate
- **Raised by:** Security, Reliability (duplicate: SEC-C3, REL-H2)
- **Severity:** CRITICAL
- **Priority:** P1
- **Type:** SECURITY HARDENING
- **Finding IDs:** SEC-C3, REL-H2
- **Root Cause:** AppRole token has 1h TTL; secret_id has no TTL (permanent credentials until Phase 6). If secret_id leaked, attacker authenticates indefinitely.
- **Design Reference:** Section 3.2 (lines 408, 956–997)
- **Related FR/NFR:** FR-2 (Secret seeding), NFR-1 (Idempotency)
- **Mitigation Strategy:**
  - Phase 3-5: Add secret_id TTL (24h recommended; document decision in design)
  - Phase 3-5: Document manual rotation procedure (quarterly, operator-initiated)
  - Phase 6: Implement automated secret_id rotation + monitoring for failed auth attempts
- **Recommendation for Architect:**
  - Updates AppRole creation (Phase 3 playbook) to include secret_id_ttl (24h default)
  - Documents: "Long-running jobs (>1h) require Vault token renewal; token auto-renews via Vault agent"
  - Adds quarterly rotation reminder to operator runbook (or implement cron job for automated rotation)
  - Implements monitoring: alert on failed Vault auth attempts (possible compromise)
  - Documents Phase 6 escalation: automated rotation + breach detection
- **Effort to P1 compliance:** 1 day
- **Success Criteria:**
  - AppRole secret_id_ttl set (24h or documented alternative)
  - Operator runbook includes rotation procedure
  - Monitoring for failed auth attempts documented

#### P1-4: TOFU Fingerprint Distribution — Manual Process, No Audit Trail
- **Raised by:** Security
- **Severity:** CRITICAL
- **Priority:** P1
- **Type:** SECURITY HARDENING
- **Finding IDs:** SEC-C4
- **Root Cause:** step-ca root fingerprint printed to console during Phase 3, captured in playbook fact variable, distributed to Phase 4/5 playbooks. No audit trail; no verification that fingerprint is correct.
- **Design Reference:** Section 9.2 (lines 540–544), Section 5.2 (line 804)
- **Related FR/NFR:** Risk-4 (TLS chain broken)
- **Mitigation Strategy:**
  - Phase 3-5: Store fingerprint in Vault (secret/data/pki/step-ca/root-fingerprint) as authoritative source
  - Phase 3-5: Add pre-bootstrap step: operator manually verifies step-ca root cert (pin fingerprint)
  - Phase 4-5: Retrieve fingerprint from Vault (creates audit trail)
  - Phase 6: Implement automated fingerprint change detection + alerting
- **Recommendation for Architect:**
  - Phase 3 playbook: after step-ca init, store root fingerprint in Vault
  - Phase 3 playbook: print fingerprint + ask operator to verify (manual TOFU verification step)
  - Phase 4-5 playbooks: retrieve fingerprint from Vault (not from playbook fact)
  - Vault audit logs will show who retrieved fingerprint + when
  - Document manual verification procedure in operator runbook
- **Effort to P1 compliance:** 1 day
- **Success Criteria:**
  - Step-ca fingerprint stored in Vault during Phase 3
  - Phase 3 includes manual operator verification step (TOFU)
  - Phase 4-5 retrieve fingerprint from Vault (audit trail created)
  - Operator runbook documents verification procedure

#### P1-5: AppRole Token TTL Renewal Not Documented; Long Jobs May Expire
- **Raised by:** Security
- **Severity:** HIGH
- **Priority:** P1
- **Type:** OPERATIONAL + IMPLEMENTATION DETAIL
- **Finding IDs:** SEC-H1
- **Root Cause:** AppRole token_ttl is 1h; token_max_ttl 4h. Design does not specify how Jenkins handles token renewal if jobs run >1h. Vault plugin auto-renewal may not be enabled.
- **Design Reference:** Section 3.2 (lines 958, 1004), Section 3.6 (vault_agent renewal for agents but not Jenkins)
- **Related FR/NFR:** FR-2 (Secret seeding / Jenkins auth)
- **Mitigation Strategy:**
  - Phase 3-5: Document Jenkins Vault plugin auto-renewal config
  - Phase 3-5: Increase token_ttl to 4h if typical jobs run <4h (architecture decision)
  - Phase 7a: Implement Molecule test (2h job, verify token renewed mid-job)
  - Add break-glass: if renewal fails, job fails gracefully with clear error
- **Recommendation for Architect:**
  - Documents decision in Section 3.2: token_ttl rationale (1h vs. 4h)
  - Phase 4 playbook (Jenkins setup): verifies Vault plugin auto-renewal enabled, documents config
  - Phase 7a testing: add Molecule test (run 2h job against Vault, verify auth succeeds)
  - Operator runbook: document token renewal behavior + troubleshooting
- **Effort to P1 compliance:** 1 day
- **Success Criteria:**
  - Design documents token_ttl choice + renewal behavior
  - Jenkins Vault plugin auto-renewal verified + configured
  - Test case: 2h job passes (token renewed)

#### P1-6: Vault Audit Logging Not Specified; Implementation Missing
- **Raised by:** Security
- **Severity:** HIGH
- **Priority:** P1
- **Type:** OPERATIONAL + INCOMPLETE SPEC
- **Finding IDs:** SEC-H6
- **Root Cause:** Design claims "Vault audit trail" (line 1987) but does NOT specify where, how, or which events logged. Vault audit logging is optional; design assumes it without implementation details.
- **Design Reference:** Section 14.2 (line 1987)
- **Related FR/NFR:** Risk Register line 1987 (Secrets audit trail)
- **Mitigation Strategy:**
  - Phase 3-5: Enable Vault audit to file backend at /var/log/vault/audit.log
  - Phase 3-5: Document retention (365 days) and log protection (rotation is atomic)
  - Phase 6: Ship logs to external SIEM
- **Recommendation for Architect:**
  - Phase 3 playbook: adds task to enable audit via `vault audit enable file file_path=/var/log/vault/audit.log`
  - Documents audit config in design (new section: "Vault Audit Logging")
  - Operator runbook: includes procedure to query/rotate audit logs
  - Phase 6: escalate to ship logs to external SIEM
- **Effort to P1 compliance:** 1 day
- **Success Criteria:**
  - Phase 3 playbook includes audit enable task
  - Design documents audit backend + retention + protection
  - Operator runbook includes audit log queries

#### P1-7: Vault Storage Backend Not Backed Up; No Disaster Recovery Path
- **Raised by:** Infrastructure, Reliability (duplicate: INFRA-H1, REL-C2)
- **Severity:** CRITICAL
- **Priority:** P1
- **Type:** OPERATIONAL + RELIABILITY
- **Finding IDs:** INFRA-H1, REL-C2
- **Root Cause:** Design states (line 1902) "Vault data is not backed up". If sec01 storage fails, all secrets lost; Vault unrecoverable. Unseal keys are backed up, but Vault data is not.
- **Design Reference:** Section 12.3 (lines 1902–1906)
- **Related FR/NFR:** Risk-1 (High impact)
- **Mitigation Strategy:**
  - Phase 3-5: Implement daily snapshot of /opt/vault/data to external storage (NFS, S3)
  - Phase 3-5: Document recovery procedure (restore snapshot → unseal → verify secrets)
  - Phase 3-5: Test recovery in non-prod environment
  - Phase 6: Implement automated backup with retention policy
- **Recommendation for Architect:**
  - Phase 3 playbook: adds backup task (tar + compress /opt/vault/data to external storage)
  - Phase 3 playbook: documents backup retention (daily snapshots, 30-day retention)
  - Operator runbook: includes recovery procedure (restore from backup → re-unseal → verify)
  - Schedule: implement backup before Phase 4 begins (cannot proceed without data protection)
- **Effort to P1 compliance:** 1 day
- **Success Criteria:**
  - Phase 3 playbook includes backup task
  - Backup externalized (not just local disk)
  - Recovery procedure tested + documented
  - Backup schedule defined (daily)

#### P1-8: Auto-Unseal Systemd Service Has No Retry Logic; Silent Failure on Slow Startup
- **Raised by:** Reliability
- **Severity:** HIGH
- **Priority:** P1
- **Type:** OPERATIONAL + RELIABILITY
- **Finding IDs:** REL-H1
- **Root Cause:** Unseal script runs once; if Vault container takes >5 sec to start (network latency, slow startup), unseal fails with "connection refused". Systemd has Restart=no; no automatic retry. Operator unaware unless SSH to sec01.
- **Design Reference:** Section 11.3 (lines 1760–1787)
- **Related FR/NFR:** NFR-3 (Auto-unseal fallback)
- **Mitigation Strategy:**
  - Phase 3-5: Add retry loop to unseal script (wait up to 30 sec for Vault readiness)
  - Phase 3-5: Update systemd service (Restart=on-failure, RestartSec=10, StartLimitBurst=3)
  - Phase 3-5: Add explicit seal-status check in Phase 3 playbook (poll Vault health until unsealed)
  - Add health-check timer (every 60 sec: check seal status)
- **Recommendation for Architect:**
  - Updates unseal script with retry loop: wait for Vault to be ready (health endpoint accessible)
  - Updates systemd service: Restart=on-failure, RestartSec=10, StartLimitBurst=3
  - Phase 3 playbook: adds uri task to poll Vault health endpoint (timeout 120 sec)
  - Systemd timer: health check every 60 sec; alert to journal if sealed
  - Operator runbook: includes troubleshooting (check systemd logs, verify Vault container)
- **Effort to P1 compliance:** 1 day
- **Success Criteria:**
  - Unseal script has retry loop (wait 30 sec)
  - Systemd service configured for auto-retry
  - Phase 3 playbook includes health-check poll
  - Health-check timer implemented

#### P1-9: No Monitoring or Alerting for Vault Seal Status
- **Raised by:** Reliability
- **Severity:** HIGH
- **Priority:** P1
- **Type:** OPERATIONAL
- **Finding IDs:** REL-H3
- **Root Cause:** Design claims "No silent failures" but provides no observability for seal status or auto-unseal failures. Vault crashes → systemd attempts unseal but fails → operator unaware unless SSH to sec01.
- **Design Reference:** Section 14.2 (line 1987)
- **Related FR/NFR:** NFR-3 (Auto-unseal fallback)
- **Mitigation Strategy:**
  - Phase 3-5: Implement health-check systemd timer (every 60 sec: check seal status)
  - Phase 3-5: Alert to journal (severity=alert) if sealed
  - Phase 4-5: Add pre-flight check: explicitly verify Vault is unsealed before proceeding
  - Phase 6: Ship alerts to centralized monitoring (Grafana, Prometheus)
- **Recommendation for Architect:**
  - Implements systemd timer + script to check Vault seal status every 60 sec
  - Script: `curl -s https://localhost:8200/v1/sys/seal-status | jq .sealed`
  - If sealed: logs alert + restarts unseal service
  - Phase 4-5 orchestration playbooks: add pre-flight check (wait for Vault unsealed)
  - Operator runbook: includes troubleshooting + monitoring procedures
- **Effort to P1 compliance:** 1 day
- **Success Criteria:**
  - Health-check timer + script implemented
  - Alert generated if sealed
  - Phase 4-5 playbooks include pre-flight verification
  - Operator can query seal status

#### P1-10: Network Segmentation — No Firewall Rules Documented
- **Raised by:** Security, Network
- **Severity:** HIGH
- **Priority:** P1
- **Type:** OPERATIONAL + SECURITY CONTROL
- **Finding IDs:** SEC-H7, NET-H1
- **Root Cause:** Design claims "internal only" but provides no firewall enforcement. No host firewall (ufw, firewalld) documented. No ACLs mentioned.
- **Design Reference:** Section 2.4 (line 141)
- **Related FR/NFR:** Blast radius (global working agreement)
- **Mitigation Strategy:**
  - Phase 3-5: Document firewall rules systematically (which hosts → which ports)
  - Phase 3-5: Implement host-level firewall on each VM (ufw before services start)
  - Phase 3-5: Add pre-flight validation (test connectivity before Phase 3 proceeds)
  - Phase 6: Implement network isolation (VLANs, ACLs)
- **Recommendation for Architect:**
  - Design: Add "Firewall & Network Segmentation" section with rules:
    - build01 → sec01: 9000 (step-ca), 8200 (Vault)
    - agent01 → sec01: 8200 (Vault)
    - agent01 → build01: 50000 (JNLP)
    - All → external: DENY (unless explicitly needed)
  - Phase 3-5 playbooks: Configure ufw on each VM, open only required ports
  - Pre-flight: test connectivity (ping, curl to expected ports)
  - Operator runbook: firewall rules + troubleshooting
- **Effort to P1 compliance:** 2 days
- **Success Criteria:**
  - Design includes communication matrix (source/dest/port/protocol)
  - Firewall rules documented + implemented
  - Pre-flight connectivity check passes
  - Operator runbook includes firewall troubleshooting

#### P1-11: Inter-Service Communication Paths Not Documented; No Communication Matrix
- **Raised by:** Network
- **Severity:** HIGH
- **Priority:** P1
- **Type:** INCOMPLETE SPEC
- **Finding IDs:** NET-H2
- **Root Cause:** Design describes secret flows in prose (Section 2.3, lines 104–137) but lacks systematic communication matrix. Implementation team won't know which firewall ports to open.
- **Design Reference:** Section 2.3
- **Related FR/NFR:** FR-1 through FR-5 (all services)
- **Mitigation Strategy:**
  - Add communication matrix to design (source/dest/port/protocol/purpose)
  - Use matrix for firewall rules (P1-10) and pre-flight validation
- **Recommendation for Architect:**
  - Adds "Communication Matrix" section to design:
    ```
    build01 → sec01:9000/HTTPS   (step-ca cert issuance)
    build01 → sec01:8200/HTTPS   (Vault AppRole auth)
    agent01 → sec01:8200/HTTPS   (Vault secret retrieval)
    agent01 → build01:50000/TCP  (Jenkins JNLP agent)
    sec01   → Proxmox:9000/HTTPS (Proxmox API, optional)
    ```
  - Matrix is normative (firewall rules + playbook vars derived from it)
- **Effort to P1 compliance:** 1 day
- **Success Criteria:**
  - Communication matrix present in design
  - All services + ports documented
  - Matrix guides firewall rules + playbook variables

#### P1-12: JNLP Communication Protocol/Port Undefined
- **Raised by:** Network
- **Severity:** HIGH
- **Priority:** P1
- **Type:** INCOMPLETE SPEC
- **Finding IDs:** NET-H3
- **Root Cause:** Design states (line 1298) "Jenkins agent connects to build01 via JNLP" but does NOT specify: port (default 50000?), protocol (plain TCP or HTTPS?), authentication method (SSH key, token, mTLS?).
- **Design Reference:** Section 7.2 (line 1298)
- **Related FR/NFR:** FR-5 (Jenkins agent deployment)
- **Mitigation Strategy:**
  - Architect clarifies JNLP configuration (port, protocol, auth)
  - Adds configuration to Phase 5 playbook (Jenkins agent setup)
  - Documents firewall rule (allow agent01 → build01:50000/TCP)
- **Recommendation for Architect:**
  - Decides JNLP configuration: default is TCP port 50000, SSH authentication
  - Adds to Phase 4 playbook (Jenkins controller setup): configure JNLP listener (port 50000)
  - Adds to Phase 5 playbook (agent setup): document JNLP connection (agent connects to build01:50000)
  - Updates communication matrix (P1-11) with JNLP entry
  - Operator runbook: JNLP troubleshooting + verification
- **Effort to P1 compliance:** 1 day
- **Success Criteria:**
  - JNLP port, protocol, auth documented in design
  - Phase 4-5 playbooks configure JNLP correctly
  - Communication matrix includes JNLP
  - Pre-flight validation tests JNLP connectivity

---

### DEFERRABLE ISSUES (P2/P3) — 28 Issues

#### P2 Issues (Defer to Phase 7; Include Mitigation Plan) — 28 findings

The following 28 findings are acceptable to defer to Phase 7 implementation with documented mitigations. Architect must provide clear owner, due date, and tracking mechanism for each.

**P2-1 through P2-28:**

| Issue | Category | Severity | Mitigation | Owner | Defer To | Tracking |
|-------|----------|----------|-----------|-------|---------|----------|
| **P2-1: AppRole Token TTL Renewal Not Documented** | Operational | HIGH | Document Jenkins Vault plugin auto-renewal config; Phase 7a implements + tests | Dev Team | Phase 7a | Molecule test in CI/CD |
| **P2-2: TLS Certificate Renewal Not Automated for Jenkins** | Reliability | HIGH | Phase 6 escalation: add Vault agent to build01 for auto-renewal; Phase 3-5 uses manual renewal | Ops | Phase 6 | Operator runbook |
| **P2-3: Container Non-Root User — Vault Token Escape Risk** | Security | HIGH | Implement AppArmor/SELinux profile; Phase 6 mandatory. Phase 3-5: non-root is primary control | SecOps | Phase 6 | Security review in Phase 7 |
| **P2-4: Secret Lifecycle Masking — Ansible no_log Coverage** | Security | HIGH | Verify no_log covers all outputs; Phase 7a testing validates. Implement Vault audit logging (P1-6) | Dev | Phase 7a | Test coverage validation |
| **P2-5: Auto-Unseal Fallback — Systemd Service Runs as Root** | Security | HIGH | Monitor unseal script reads via auditd; Phase 6 mandatory encryption. Phase 3-5: plaintext with backup discipline | SecOps | Phase 6 | Audit logs reviewed monthly |
| **P2-6: Break-Glass Access — No Disaster Recovery Procedure** | Operational | HIGH | Implement Vault backup/restore (P1-7); document break-glass in runbook | Ops | Phase 3-5 completion | Runbook testing |
| **P2-7: Collection Scaffolds Are Incomplete** | Implementation | HIGH | Verify all collection roles implemented + tested (Molecule required) before Phase 3 | Dev | Phase 7a | Molecule test suite |
| **P2-8: Cloud-Init Network Configuration Not Verified** | Infrastructure | HIGH | Pre-flight task to Phase 3: validate network config, test cloud-init on existing VM | Ops | Phase 3 pre-flight | Pre-flight checklist |
| **P2-9: Bootstrap Container Network Access Not Specified** | Network | HIGH | Document bootstrap container network (host network vs. container network); add pre-flight ping | Ops | Phase 3 pre-flight | Pre-flight checklist |
| **P2-10: Network Isolation Not Enforced** | Network/Security | HIGH | Document isolation assumptions (internal VLAN); implement host firewall (P1-10); verify Proxmox config | Ops/SecOps | Phase 3 | Pre-flight verification |
| **P2-11: No Latency or Bandwidth Analysis** | Network | HIGH | Add latency assumptions (<5ms); pre-flight ping test; configure explicit timeouts | Ops | Phase 3 | Pre-flight latency test |
| **P2-12: Network Failover / SPOF Not Addressed** | Reliability | HIGH | Document as Phase 3-5 SPOF; Phase 6 task: link aggregation + redundant connections | Ops | Phase 6 | Design note + roadmap |
| **P2-13: Variable Naming Doesn't Follow Standards** | Framework | HIGH | Audit all variable names; rename to <component>_<role>_<thing> format | Dev | Phase 7a | Code review gate |
| **P2-14: Effort Estimates Appear Optimistic** | Planning | HIGH | Revise estimates +20–50%; add confidence intervals (best/realistic/pessimistic) | Delivery | Phase 7 | Delivery planning gate |
| **P2-15: Phase 6 HA Migration Path Unclear** | Strategy | HIGH | Document Vault file→Raft migration, step-ca single→multi-tier, unseal Shamir→cloud auto-seal | Architect | Phase 6 planning | Design doc update |
| **P2-16: Backward Compatibility Risk; No Versioning Strategy** | Framework | HIGH | Define semantic versioning (X.Y.Z) for collections; pin versions; document minimum requirements | Architect | Phase 7a | requirements.yml review |
| **P2-17: Idempotency Gap — Partial Vault State Not Detected** | Reliability | CRITICAL | Add pre-flight check to Phase 3: try reading known secret; fail if partial state detected | Dev | Phase 7a | Molecule test + playbook logic |
| **P2-18: Port Configuration Without Firewall Rules** | Network | HIGH | Implement firewall rules (P1-10); clarify JNLP port (P1-12) | Ops | Phase 3 | Firewall configuration |
| **P2-19: Jenkins Agent Pooling — Single Agent is SPOF** | Reliability | HIGH | Document SPOF; Phase 6 task: implement agent pooling (2–3 agents) | Ops | Phase 6 | Runbook note + roadmap |
| **P2-20 through P2-28: Documentation & Process Gaps** | Operational | MEDIUM-LOW | Operator runbook (Phase 3-5 completion); pre-flight checklist; troubleshooting guide | Ops | Phase 3-5 | Runbook sign-off |

---

## SEVERITY REASSESSMENT

### Before Consolidation (Raw Panel Findings)
- Critical: 12
- High: 27
- Medium: 19
- Low: 3
- **Total: 61**

### After Deduplication & Consolidation
- Critical: 12 (no change; all distinct root causes)
- High: 27 (no change; deduped into High-severity clusters)
- Medium: 19 (grouped into P2/P3 with mitigation strategies)
- Low: 3 (acceptable to defer)
- **Total Consolidated Issues: 35 distinct clusters**

### Severity Changed During Referee Analysis

**No re-scoring applied.** Deduplication did not elevate findings (e.g., two High-severity findings about similar issues are not re-scored as Critical unless they represent a shared architectural flaw). 

Examples of consolidated findings that maintained original severity:
- SEC-C2 + SEC-H5: Root token disk → both Critical/High security concerns, consolidated into single P1 issue, retained Critical severity
- NET-C1 + INFRA-H2: DNS resolution → exact duplicate issue, consolidated, retained Critical severity
- SEC-H2 + NET-C2: TLS SAN mismatch → exact duplicate, consolidated, retained Critical severity

---

## PRIORITY BREAKDOWN & APPROVAL ROADMAP

### P0 (Blocking) — Must Fix Before Stage 5 Human Gate

**6 Findings** (all blocking Stage 5 approval):

1. **Missing Orchestration Playbooks** (INFRA-C3, REL-C1, REL-C2)
   - Phase 3, 4, 5 playbooks do not exist; must be committed to git
   - **Effort:** 2 days
   - **Owner:** Architect/DevOps
   - **Target:** Before Stage 5 gate

2. **Missing Terraform Repos** (INFRA-C2)
   - terraform-build01-deploy, terraform-agent01-deploy repositories missing
   - **Effort:** 2 days
   - **Owner:** Infrastructure
   - **Target:** Before Stage 5 gate

3. **Missing blueprints.vault.init_and_unseal Role** (INFRA-C1)
   - Role does not exist; blocks Vault initialization
   - **Effort:** 1–2 days
   - **Owner:** Platform/Vault
   - **Target:** Before Stage 5 gate

4. **DNS Resolution Strategy Not Documented** (NET-C1, INFRA-H2)
   - No DNS mechanism specified; design assumes resolution without implementation
   - **Effort:** 1 day
   - **Owner:** Infrastructure
   - **Target:** Before Stage 5 gate

5. **TLS Certificate SAN Mismatch** (SEC-H2, NET-C2)
   - Certificate SANs incomplete; TLS validation will fail
   - **Effort:** 1 day
   - **Owner:** Security/PKI
   - **Target:** Before Stage 5 gate

6. **Framework Misalignment — LADR-003 + Collection Overwrite** (STRAT-C1, STRAT-C3)
   - Collection boundary unclear; existing versions at risk of silent overwrite
   - **Effort:** 2 days
   - **Owner:** Architect
   - **Target:** Before Stage 5 gate

**Total P0 Effort:** ~8–9 days (can be parallelized)  
**Estimated Duration:** 3–5 days (with parallelization)

### P1 (Must-Fix Before Phase 7a) — 12 Findings

All 12 P1 findings must be resolved before Phase 7a implementation begins. Architect submits revised design documenting all P1 mitigations (can defer some implementation to Phase 7 if mitigation is clear + trackable).

**Examples of P1 findings:**
- Unseal key backup verification
- Root token to tmpfs + auto-delete
- AppRole secret_id TTL
- TOFU fingerprint audit trail
- Vault audit logging
- Vault backup/restore
- Auto-unseal retry logic
- Vault seal monitoring
- Firewall rules
- Communication matrix
- JNLP configuration

**Effort to P1 compliance:** ~10 days (can be parallelized)  
**Estimated Duration:** 5–7 days

### P2 (Defer to Phase 7; Include Mitigation) — 28 Findings

These findings are acceptable to defer to Phase 7 implementation **IF**:
- Architect provides clear mitigation strategy in revised design
- Owner, due date, and tracking mechanism assigned
- No critical gap in Phase 3-5 delivery

Examples: variable naming standards, effort re-estimation, Phase 6 migration path, backup strategy refinement, monitoring escalation.

### P3 (Defer to Phase 6+) — 13 Findings

These findings are acceptable to defer beyond Phase 5 delivery without explicit Phase 7 mitigation (Phase 6+ design task).

Examples: cloud auto-seal, HA failover, multi-tier step-ca, agent pooling, Prometheus monitoring, SIEM shipping.

---

## ABBREVIATED RE-REVIEW PLAN

### Scope

After architect submits revised design addressing P0 + P1 findings, Referee will conduct **abbreviated re-review** (1–2 hours).

**Changed Sections to Review:**
- "Missing Implementation Artifacts" → verify repos + playbooks + roles exist
- "DNS Configuration Prerequisites" → verify strategy + Terraform update
- "Network Configuration" → verify SANs + communication matrix
- "Framework Alignment & Existing Collections" → verify LADR-003 compliance + version audit
- "Operational Procedures" → verify P1 mitigations (backup, monitoring, firewall) documented + testable

**Changed Artifacts to Verify:**
- Three new orchestration playbooks (syntax, gates, error handling)
- Two new Terraform repos (network config, cloud-init)
- New init_and_unseal role (Molecule test scaffold)
- Updated design sections (DNS, SANs, communication matrix, framework alignment)

### Passing Criteria for Abbreviated Re-Review

**PASS** if:
- [ ] All P0 findings resolved (no blocking issues remain)
- [ ] No new Critical findings introduced
- [ ] P1 findings have documented mitigations + clear owner/due date
- [ ] Design sections are clear + implementable
- [ ] No architectural regressions

**FAIL** (return to architect) if:
- [ ] Any P0 findings remain unresolved
- [ ] New Critical findings introduced
- [ ] P1 mitigations are vague or incomplete
- [ ] Architectural regressions observed

### Timeline

- **Architect revision:** 3–5 days (P0 + P1 focus)
- **Abbreviated re-review:** 1–2 hours
- **Stage 5 human gate:** Proceeds after Referee sign-off

---

## APPROVED ASPECTS (No Revision Needed)

The Referee explicitly approves the following design aspects; **no rework required**:

### Architecture & Orchestration
- ✓ **Phase 3-5 orchestration model:** Sequential phases (init → security → deployment) is sound; gates + idempotency checks are thoughtful
- ✓ **Ansible-as-orchestration:** Choice to use Ansible playbooks as orchestration engine is appropriate for homelab/bootstrap; correct for Phase 3-5
- ✓ **Vault as secrets store:** Vault is correct choice for Proxmox token, TFC token, Jenkins credentials
- ✓ **step-ca for TLS:** Single-tier root CA is appropriate for Phase 3-5; migration to HA in Phase 6 is clear
- ✓ **Jenkins controller + agent pattern:** Separation of controller (build01) and agent (agent01) is sound; agent pooling escalation to Phase 6 is acceptable

### Security & Compliance
- ✓ **Non-root containers:** Design correctly requires Vault to run as non-root user; AppArmor/SELinux hardening can be added in Phase 6
- ✓ **Unseal key backup:** Concept of manual operator backup with verification is sound; automated S3 escalation to Phase 6 is reasonable
- ✓ **AppRole authentication:** Using AppRole for infrastructure services is correct pattern; TTL + rotation strategy is sound
- ✓ **TLS/mTLS for inter-service:** Requiring TLS between all services (Vault, step-ca, Jenkins) is correct; TOFU verification + audit trail escalation to Phase 6 is reasonable

### Reliability & Observability
- ✓ **Auto-unseal systemd service:** Concept of systemd service for auto-unseal is sound; retry logic + monitoring are appropriate enhancements
- ✓ **Idempotency model:** Design emphasizes re-runnable playbooks; gates to detect partial state are correct approach
- ✓ **Backup model:** Vault backup concept is sound; encrypted off-site storage + recovery testing is appropriate for Phase 6

### Framework & Portability
- ✓ **LADR-003 alignment goal:** Design correctly aspires to LADR-003 compliance; clarification (not redesign) needed
- ✓ **Collection layering:** Separation of collection roles (infrastructure) from orchestration playbook (secrets) is correct direction
- ✓ **Org-neutral goal:** Design correctly intends to enable reuse by IOTel, Saicom; hardcodes need to move to playbook layer only

---

## REQUIRED REVISIONS SUMMARY

### Priority 1 Revisions (P0 — Must Complete Before Resubmission to Approval Gate)

Architect must address all 6 P0 findings:

1. **Commit three orchestration playbooks** (phase-3-orchestrate.yml, phase-4-orchestrate.yml, phase-5-orchestrate.yml) to bootstrap-container/playbooks/
2. **Create terraform-build01-deploy and terraform-agent01-deploy repos** following terraform-sec01-deploy pattern
3. **Implement blueprints.vault.init_and_unseal role** in ansible-collection-vault with Molecule tests
4. **Document DNS resolution strategy** (specify mechanism: dnsmasq, /etc/hosts, or external DNS; update Terraform search_domain)
5. **Update TLS certificate SANs** to include FQDN (vault.sec01.vernify.internal); add post-issuance validation
6. **Clarify framework alignment** (LADR-003 compliance, existing collection versioning, org-neutral configuration)

### Priority 2 Revisions (P1 — Document Mitigations Before Phase 7a)

Architect must provide mitigation strategies for all 12 P1 findings in revised design:

1. Unseal key backup verification mechanism
2. Root token to tmpfs + auto-delete
3. AppRole secret_id TTL (24h)
4. TOFU fingerprint audit trail (store in Vault)
5. AppRole token renewal documentation (Jenkins plugin config)
6. Vault audit logging (file backend at /var/log/vault/audit.log)
7. Vault backup/restore procedure (daily snapshots, documented recovery)
8. Auto-unseal retry logic (systemd Restart=on-failure, script wait loop)
9. Vault seal monitoring (systemd timer + alert)
10. Firewall rules (communication matrix + host firewall config)
11. Inter-service communication matrix
12. JNLP configuration (port 50000, SSH auth)

### What Does NOT Need Revision

The following areas do NOT require changes; Referee approves as-is:

- ✓ **Architecture model:** Sequential phases, Ansible orchestration, Vault/step-ca/Jenkins pattern
- ✓ **Phase 6 escalations:** HA failover, cloud auto-seal, agent pooling, monitoring—all appropriate for Phase 6
- ✓ **Security posture:** Non-root containers, AppRole, TLS enforcement, audit logging direction
- ✓ **Reliability approach:** Idempotency, auto-unseal, backup recovery—all sound

---

## CONDITIONAL APPROVALS & PHASE 7A GATES

### Conditions for Stage 5 Approval (Human Gate)

Design is APPROVED for Stage 5 human gate **IF AND ONLY IF**:

1. **All P0 findings resolved:** Architect submits revised design + artifacts addressing all 6 blocking issues
2. **All P1 mitigations documented:** Architect provides clear owner, due date, tracking for each of 12 P1 findings
3. **Framework alignment verified:** LADR-003 compliance confirmed; existing collection versioning audit complete
4. **No new Critical findings:** Abbreviated re-review passes without introducing new Critical issues
5. **Operator runbook skeleton exists:** Pre-flight checklist, deployment gates, troubleshooting guide drafted

### Phase 7A Implementation Gates (Secondary Validation)

During Phase 7a implementation, the following gates must pass:

- [ ] **Molecule tests:** All Ansible roles pass Molecule test suite (init_and_unseal, container_server, jenkins, step_ca, etc.)
- [ ] **Playbook dry-runs:** All three orchestration playbooks execute `--check` mode without errors
- [ ] **Pre-flight validation:** DNS resolution, network connectivity, TLS cert validation all pass
- [ ] **Backup/restore test:** Vault backup created, restored, verified in non-prod environment
- [ ] **Security review:** Code review of all sensitive tasks (unseal, secrets, credentials) verifies no leaks
- [ ] **Operator runbook sign-off:** Ops team reviews + approves troubleshooting procedures

---

## PHASE 7A IMPLEMENTATION GUIDANCE

### Recommended Parallelization for Revision Phase

**Wave 1 (Days 1-2):** Missing artifacts
- Commit orchestration playbooks (2 days, can start immediately)
- Create Terraform repos (2 days, can run in parallel)

**Wave 2 (Days 2-3):** Role implementation
- Implement init_and_unseal role (1–2 days, depends on Vault API knowledge)

**Wave 3 (Days 3-4):** Design clarifications
- DNS strategy documentation (1 day)
- TLS SAN updates (1 day)
- Framework alignment audit (2 days, can run in parallel with Wave 1-2)

**Wave 4 (Days 4-5):** P1 mitigations
- Document all 12 P1 mitigations in revised design sections (2–3 days)

**Critical Path:** P0 + P1 revisions are sequential (must complete before abbreviated re-review); can parallelize within each wave.

---

## NEXT STEPS

### Immediate (Next 1 Week)

1. **Architect receives Referee Decision** → Reviews all 6 P0 + 12 P1 findings
2. **Architect creates revision plan** → Dates, owners, dependencies for each finding
3. **Architect begins P0 revisions** → Focus on blocking issues (playbooks, repos, roles, DNS, SANs, framework)
4. **Delivery Planner waits** → No Phase 7a planning until abbreviated re-review passes

### Week 2

1. **Architect submits revised design** → All P0 findings resolved, P1 mitigations documented
2. **Referee conducts abbreviated re-review** (1–2 hours)
3. **If PASS:** Design approved; proceeds to Stage 5 human approval gate
4. **If FAIL:** Architect addresses gaps (typically <1 day); rapid re-review follows

### Stage 5 Human Gate (Approval)

1. **Design Approval Board** reviews revised design + Referee decision
2. **Board approves or requests clarifications** (typically approves; Referee already resolved issues)
3. **Approval signed off; design is COMPLETE**

### Stage 6 Planning (Parallel)

1. **Delivery Planner breaks design into Phase 7a work items** (2-day chunks, explicit dependencies, acceptance criteria)
2. **Implementation team assigned**
3. **Phase 7a begins:** Coding, Molecule tests, playbook dry-runs, pre-flight validation

### Phase 7A Execution (Secondary Validation)

- Code reviews validate security-sensitive tasks (unseal, secrets)
- Molecule tests ensure roles are idempotent + functional
- Playbook dry-runs verify orchestration gates work
- Pre-flight checklist ensures network, DNS, TLS configured correctly
- Backup/restore test validates disaster recovery procedure

---

## TRACEABILITY: FINDINGS TO REQUIREMENTS

| Finding ID | Issue | Severity | Priority | Related FR/NFR | Design Section | Risk Register | Status |
|-----------|--------|----------|----------|---|---|---|---|
| INFRA-C3 | Missing Orchestration Playbooks | CRITICAL | P0 | FR-1, FR-2, FR-4, FR-5 | Section 5-7 | R2 | **REQUIRES REVISION** |
| INFRA-C2 | Missing Terraform Repos | CRITICAL | P0 | FR-4, FR-5 | Section 4.2-4.3 | R2 | **REQUIRES REVISION** |
| INFRA-C1 | Missing init_and_unseal Role | CRITICAL | P0 | FR-1, NFR-1 | Section 3.2 | R2 | **REQUIRES REVISION** |
| NET-C1 | DNS Resolution Not Documented | CRITICAL | P0 | FR-1-5 | Section 2.3 | R3 | **REQUIRES REVISION** |
| SEC-H2 / NET-C2 | TLS Certificate SAN Mismatch | CRITICAL | P0 | FR-1, FR-3, NFR-2 | Section 3.1 | R4 | **REQUIRES REVISION** |
| STRAT-C1 / STRAT-C3 | Framework Misalignment + Collection Overwrite | CRITICAL | P0 | NFR-4 | Section 3 | R2, R5 | **REQUIRES REVISION** |
| SEC-C1 | Unseal Key Backup Verification | CRITICAL | P1 | FR-1, Risk-1 | Section 11.2 | R1 | **REQUIRES MITIGATION** |
| SEC-C2 | Root Token to Disk | CRITICAL | P1 | FR-1, NFR-2 | Section 3.2, 10.1 | Risk-1 | **REQUIRES MITIGATION** |
| SEC-C3 / REL-H2 | AppRole secret_id No TTL | CRITICAL | P1 | FR-2, NFR-1 | Section 3.2 | R6 | **REQUIRES MITIGATION** |
| SEC-C4 | TOFU Fingerprint Distribution | CRITICAL | P1 | FR-3, Risk-4 | Section 9.2 | R4 | **REQUIRES MITIGATION** |
| SEC-H1 | AppRole Token TTL Renewal | HIGH | P1 | FR-2 | Section 3.2, 3.6 | R6 | **REQUIRES DOCUMENTATION** |
| SEC-H6 | Vault Audit Logging | HIGH | P1 | Risk Register | Section 14.2 | R1 | **REQUIRES IMPLEMENTATION** |
| INFRA-H1 / REL-C2 | Vault Backup Missing | CRITICAL | P1 | Risk-1 | Section 12.3 | R1 | **REQUIRES IMPLEMENTATION** |
| REL-H1 | Auto-Unseal No Retry Logic | HIGH | P1 | NFR-3 | Section 11.3 | R1 | **REQUIRES IMPLEMENTATION** |
| REL-H3 | No Vault Seal Monitoring | HIGH | P1 | NFR-3 | Section 14.2 | R1 | **REQUIRES IMPLEMENTATION** |
| SEC-H7 / NET-H1 | No Firewall Rules | HIGH | P1 | Security | Section 2.4 | R3 | **REQUIRES IMPLEMENTATION** |
| NET-H2 | No Communication Matrix | HIGH | P1 | FR-1-5 | Section 2.3 | R3 | **REQUIRES DOCUMENTATION** |
| NET-H3 | JNLP Undefined | HIGH | P1 | FR-5 | Section 7.2 | R3 | **REQUIRES DOCUMENTATION** |

---

## APPENDIX A: DEDUPLICATION DETAILS

### Exact Duplicates (Merged)

**Cluster 1: Root Token Disk Storage**
- SEC-C2: Root token written to /opt/vault/root-token.txt (0600 perms)
- SEC-H5: Auto-unseal systemd service runs as root, plaintext keys
- **Root Cause:** Token + keys on persistent disk = exposure window
- **Consolidated:** Single P1 issue; move to tmpfs + auto-delete + key encryption strategy
- **Severity:** Remains CRITICAL

**Cluster 2: DNS Resolution**
- NET-C1: DNS resolution strategy not documented; .vernify.internal FQDNs assumed
- INFRA-H2: DNS configuration not implemented; Terraform sets wrong search domain
- **Root Cause:** No DNS mechanism specified; design assumes resolution without implementation
- **Consolidated:** Single P0 issue; architect clarifies strategy + Terraform update
- **Severity:** Remains CRITICAL

**Cluster 3: TLS Certificate Validation**
- SEC-H2: TLS SAN validation—hostname mismatch will cause failures
- NET-C2: Certificate SANs incomplete; vault.sec01.vernify.internal not in SANs
- **Root Cause:** Certificate SANs do not match client hostnames
- **Consolidated:** Single P0 issue; add FQDN to SANs + post-validation task
- **Severity:** Remains CRITICAL

**Cluster 4: AppRole Secret_ID TTL**
- SEC-C3: AppRole secret_id has no TTL; credentials never rotate
- REL-H2: Token expires but credentials don't rotate; window of compromise unlimited
- **Root Cause:** Long-lived secret_id (permanent in v1)
- **Consolidated:** Single P1 issue; add 24h TTL + quarterly rotation procedure
- **Severity:** Remains CRITICAL

**Cluster 5: Vault Backup**
- INFRA-H1: Vault storage backend not backed up; no disaster recovery path
- REL-C2: No automatic Vault data backup; new Vault on re-run = broken downstream
- **Root Cause:** Vault data not backed up; if sec01 fails, all secrets lost
- **Consolidated:** Single P1 issue; implement daily snapshot + restore procedure
- **Severity:** Remains CRITICAL (elevated from High due to Phase 4-5 dependency)

**Cluster 6: Firewall Rules**
- SEC-H7: Network segmentation not enforced; no firewall rules documented
- NET-H1: Port configuration without firewall rules; internal-only claim unenforceable
- **Root Cause:** Design claims "internal only" without enforcement mechanism
- **Consolidated:** Single P1 issue; document rules + implement host firewall
- **Severity:** Remains HIGH

---

### Semantic Consolidation (Related Finding Clusters)

**Auto-Unseal Reliability (3 findings → 2 P1 issues):**
- REL-H1: Retry logic missing → P1-8 (unseal retry + healthcheck)
- REL-H3: No monitoring → P1-9 (seal status monitoring + alerting)
- REL-C1: Idempotency gap → P2-17 (partial state detection + error handling)

**Orchestration Artifacts (3 findings → 1 P0 + 2 P1 issues):**
- INFRA-C3: Playbooks don't exist → P0-1 (commit playbooks)
- REL-C1: Partial Vault state not detected → P2-17 (add pre-flight check)
- REL-C2: Backup missing → P1-7 (implement backup)

---

## APPENDIX B: SEVERITY SCORING LOGIC

### Why Findings Were Scored as Critical vs. High

**Critical Findings (12)** = Shipping-blocking or security-critical:

1. **INFRA-C3, INFRA-C2, INFRA-C1, NET-C1, NET-C2, STRAT-C1/C3:** Missing artifacts + incomplete specs → Phase 3-5 cannot execute without these
2. **SEC-C1, SEC-C2, SEC-C3, SEC-C4:** Security boundaries violated (backup enforcement, key escrow, credential rotation, TOFU trust)
3. **REL-C1, REL-C2:** Idempotency + data loss risks → operator safety compromised

**High Findings (27)** = Quality gaps, unmitigated risks, must be addressed before Phase 7a:

- Security: audit logging, token renewal, secret masking, container hardening (8 findings)
- Infrastructure: collection scaffolds, network config, DNS verification (3 findings)
- Network: firewall rules, communication matrix, JNLP config, latency analysis (7 findings)
- Strategy: variable naming, effort estimates, backward compatibility (5 findings)
- Reliability: monitoring, cert renewal, agent SPOF, idempotency (5 findings)

**Medium/Low Findings (22)** = Tech debt, documentation, future improvements:

- Documentation gaps (operator runbook, troubleshooting)
- Process improvements (pre-flight checks, backup testing)
- Phase 6+ escalations (HA, cloud auto-seal, monitoring)

---

## APPENDIX C: RISKS THAT REMAIN AFTER REVISIONS

### Accepted Risks (Documented for Phase 6 Escalation)

| Risk | Severity | Mitigation (Phase 3-5) | Phase 6 Escalation |
|------|----------|-----|---|
| Vault storage SPOF (file-based) | HIGH | Single daily backup; documented recovery | Raft HA + multi-node Vault |
| Unseal keys plaintext in script | HIGH | Restricted perms (0700); auditd monitoring | TPM encryption + cloud auto-seal |
| AppRole secret_id permanent | HIGH | 24h TTL + manual rotation | Automated rotation + breach detection |
| Jenkins controller SPOF | MEDIUM | Single instance acceptable for homelab | Jenkins HA + master replication |
| Jenkins agent SPOF | MEDIUM | Single agent; document failover manual | Agent pooling + auto-balancing |
| step-ca single-tier (long chain) | LOW | Single-tier acceptable for v1 | Intermediate CA + shortened chain |
| No centralized monitoring | MEDIUM | Manual seal status checks | Prometheus + Grafana + alerting |
| Network SPOF (single vmbr0) | HIGH | Documented as Phase 3-5 limitation | Link aggregation + redundant NICs |

All risks are acceptable for Phase 3-5 with clear Phase 6 escalation path.

---

## FINAL DECISION SUMMARY

| Question | Answer |
|----------|--------|
| **Design fundamentally sound?** | YES—architecture, orchestration, security posture are solid |
| **Blocking issues prevent approval?** | YES—6 P0 findings must be resolved (missing artifacts, incomplete specs) |
| **Can issues be resolved without redesign?** | YES—all issues are implementable within existing design scope |
| **Should architect return to Stage 2?** | NO—no fundamental architectural flaws; Stage 4 revision path is appropriate |
| **Estimated effort to P0 compliance?** | 3–5 days (parallelized) |
| **Estimated effort to P1 compliance?** | 5–7 days (includes Phase 7a mitigation documentation) |
| **Risk to Phase 3-5 timeline?** | LOW—no redesign required; revisions fit within Stage 4-7a schedule |
| **Recommendation to user?** | **REQUIRE REVISIONS** → abbreviated re-review → Stage 5 approval gate |

---

**Referee Status:** COMPLETE  
**Decision:** REQUIRE REVISIONS  
**Next Action:** Architect revises design addressing P0 + P1 findings (3–5 days); submits to Referee for abbreviated re-review; proceeds to Stage 5 human gate upon Referee sign-off

**Compiled by:** Design Referee  
**Date:** 27 June 2026  
**Classification:** Internal Architecture Review (Stage 4)

---

**END OF REFEREE DECISION DOCUMENT**
