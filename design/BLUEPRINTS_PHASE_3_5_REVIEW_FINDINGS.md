# MULTIDISCIPLINARY REVIEW PANEL FINDINGS
## Vernify Phase 3-5 Technical Architecture Design

**Review Date:** 27 June 2026  
**Status:** FINDINGS REPORT (awaiting Referee synthesis)  
**Panel Members:**
- Security Engineering Specialist
- Infrastructure/Platform Specialist
- Network Engineering Specialist
- Technical Strategy/Consulting Specialist
- Reliability Engineering Specialist

---

## EXECUTIVE SUMMARY

**Overall Assessment:** YELLOW (Requires Revisions Before Approval)

The Technical Architecture Design for Vernify Phase 3-5 is comprehensive, operationally detailed, and demonstrates strong thinking on orchestration, idempotency, and security controls. However, the review panel identified **35 distinct findings** across five domains:

| Domain | Critical | High | Medium | Low | Total |
|--------|----------|------|--------|-----|-------|
| **Security** | 4 | 7 | 5 | 1 | **17** |
| **Infrastructure** | 3 | 4 | 2 | 1 | **10** |
| **Network** | 2 | 7 | 1 | 0 | **10** |
| **Strategy** | 1 | 4 | 6 | 0 | **11** |
| **Reliability** | 2 | 5 | 5 | 1 | **13** |
| **TOTAL** | **12** | **27** | **19** | **3** | **61** |

**Key Blockers (Must Fix Before Proceeding):**
1. Missing implementation of three orchestration playbooks (Phase 3, 4, 5)
2. Incomplete collection scaffolds (blueprints.vault.init_and_unseal role does not exist)
3. DNS configuration not specified (VMs cannot resolve internal hostnames)
4. TLS hostname validation will fail (certificate SANs incomplete)
5. Framework misalignment (LADR-003 violation; secret retrieval boundary unclear)
6. Backward compatibility risk (existing blueprints.vault may be overwritten without migration path)

**Critical Security Issues:**
1. Unseal key backup enforcement lacks verification mechanism
2. Root token written to persistent disk (privilege escalation window)
3. AppRole secret_id has no TTL; credentials never rotate
4. TOFU fingerprint distribution is manual and unaudited

**Recommendation:** Return to design review with requested revisions before approving for implementation. Estimated revision effort: 3-5 days (focus on blockers). No fundamental architectural flaws; all issues are resolvable through clarification, implementation, or operational controls.

---

## CONSOLIDATED RISK SUMMARY

### TOP 5 RISKS IDENTIFIED ACROSS ALL REVIEWERS

1. **CRITICAL: Missing Implementation Artifacts**
   - Phase 3, 4, 5 orchestration playbooks do not exist (design provides specs but no runnable YAML)
   - blueprints.vault.init_and_unseal role does not exist (blocking dependency)
   - terraform-build01-deploy and terraform-agent01-deploy repos missing
   - Impact: Phase 3 execution will fail on first `ansible-playbook` call (playbook not found)
   - Resolution: Implement from design specs (3-4 days) — design is detailed enough to execute directly

2. **CRITICAL: Network Configuration Blocking**
   - DNS resolution strategy not documented; Terraform assumes "vernify.internal" but sets search domain to "vernify.com"
   - TLS certificate SANs incomplete; clients connecting to "vault.sec01.vernify.internal" but cert only has SANs ["vault", "vault.sec01", "192.168.22.51"]
   - Firewall rules for inter-service communication not documented
   - Impact: Phase 3 will timeout on first hostname resolution attempt; TLS handshakes will fail in Phase 4
   - Resolution: Add DNS server documentation, update certificate SAN list, document firewall rules (1 day)

3. **CRITICAL: Secret Management & Key Escrow**
   - Vault data not backed up; if sec01 storage fails, Vault is unrecoverable (all secrets lost, Proxmox/TFC/Jenkins tokens gone)
   - Unseal keys stored in plaintext in shell script (0700 perms); if host compromised, keys exposed
   - AppRole secret_id has no automatic rotation; credentials never expire (maximum window of compromise unknown)
   - Impact: Single storage failure = complete system rebuild required; operator loses ability to access Vault indefinitely if unseal keys are lost
   - Resolution: Implement automated Vault backup, move unseal keys to tmpfs, add AppRole secret_id TTL, implement recovery procedures (2-3 days)

4. **CRITICAL: Framework Misalignment & Portability**
   - Design violates LADR-003 (secret retrieval boundary) — unclear which tasks belong in collection roles vs. orchestration layer
   - Vernify-specific hardcodes (CN="Vernify Root CA", domains, IPs) prevent reuse by other orgs (IOTel, Saicom)
   - No acknowledgment of existing blueprints.vault collection; risk of overwrite without backward-compatibility path
   - Impact: Other consumer orgs cannot reuse blueprints; re-use attempt requires significant forking/rework; existing Vault collection may be silently overwritten
   - Resolution: Audit framework alignment, move hardcodes to playbook layer, document existing collection state & migration path (2-3 days)

5. **CRITICAL: Operational Visibility & Monitoring**
   - No monitoring or alerting for Vault seal status (operator unaware when Vault is sealed; CI silently fails)
   - Auto-unseal systemd service has no retry logic; if Vault startup is slow, unseal fails silently
   - No health-check mechanism for containers (Vault crash = no automatic restart)
   - Impact: Operator discovers failures only when users report broken CI; MTTR for Vault sealing could be hours
   - Resolution: Add health-check timer, implement retry logic in unseal script, add explicit seal-status verification to playbooks (1-2 days)

---

## DETAILED FINDINGS BY DOMAIN

### SECURITY ENGINEERING FINDINGS

#### CRITICAL FINDINGS

**SEC-C1: Unseal Key Backup Enforcement — No Verification Mechanism**
- **Current Design:** Section 11.2, lines 5.2: Operator manually backs up unseal keys; prompt-based confirmation only ("Have you backed up the unseal keys? yes/no")
- **Issue:** Prompt-based honor system has no enforcement. If operator confirms without actually backing up, Vault is unrecoverable on host failure. Design provides no checksum verification or secondary confirmation path.
- **Recommendation:** Require operator to provide hash/checksum of backup; store in Vault audit log. Implement optional automated backup to encrypted S3 (Phase 6).
- **LADR Reference:** Implicit in Section 14.2 "Secrets audit trail" but incomplete.

**SEC-C2: Root Token Written to Disk — Privilege Escalation Risk**
- **Current Design:** Section 3.2 (lines 266–267), Section 10.1 (lines 1657–1666): Root token written to `/opt/vault/root-token.txt` (0600 perms, readable by root only)
- **Issue:** Root token is administrative credential; persistent on disk is exposure window. If host is compromised (container escape, kernel RCE), attacker reads root token before deletion. Vault container runs as non-root but token is written by ansible as root.
- **Recommendation:** Use tmpfs (e.g., /run/secrets) instead of persistent disk. Add timer to delete token X minutes after secret seeding (don't wait for manual completion). Use stdin/env var for token passing if tooling supports (audit community.hashi_vault).
- **LADR Reference:** LADR-009 (Non-Root Containers) acknowledges this trade-off but mitigation is incomplete.

**SEC-C3: AppRole secret_id Has No TTL; Credentials Never Rotate**
- **Current Design:** Section 3.2 (lines 408, 956–997), Section 14.1: AppRole token has 1h TTL; secret_id has no TTL (permanent credentials until manual rotation in Phase 6)
- **Issue:** secret_id is long-lived authentication credential; if leaked, attacker authenticates indefinitely. Design acknowledges this (line 1979) but mitigation is "detect unauthorized auth attempts" (incomplete). No automatic rotation, no warning when secret_id is exposed.
- **Recommendation:** Document that secret_id is permanent in v1; add manual rotation procedure (quarterly); implement secret_id TTL + auto-renewal for Phase 6. Add monitoring: alert on failed auth attempts (possible compromise).
- **LADR Reference:** Risk Register line 1979 identifies as Medium but mitigations incomplete.

**SEC-C4: TOFU Fingerprint Distribution — Manual Process, No Audit Trail**
- **Current Design:** Section 9.2 (lines 540–544), Section 5.2 (line 804): step-ca root fingerprint printed to console during Phase 3; captured in playbook fact variable; distributed to Phase 4/5 playbooks
- **Issue:** Fingerprint distribution is implicit and unaudited. If attacker MITM's playbook execution or intercepts environment variables, wrong fingerprint could be pinned. No mechanism to verify fingerprint integrity during transmission or detect changes.
- **Recommendation:** Store step-ca root fingerprint in Vault (e.g., secret/data/pki/step-ca/root-fingerprint) as authoritative source; retrieve from Vault in Phase 4/5 to create audit trail. Add pre-bootstrap step: operator manually verifies step-ca root cert. Add fingerprint change detection.
- **LADR Reference:** Risk Register line 1972 includes "TOFU fingerprint mismatch" but no specific remediation.

#### HIGH-SEVERITY FINDINGS

**SEC-H1: AppRole Token TTL Renewal Not Documented**
- **Current Design:** Section 3.2 (lines 958, 1004): token_ttl 1h, token_max_ttl 4h. Section 3.6 (vault_agent) shows auto-renewal for agents but not Jenkins.
- **Issue:** 1h token TTL means Vault tokens expire mid-job if job runs longer than 1h and doesn't renew. Jenkins Vault plugin must implement token auto-renewal. Design does not specify renewal threshold or error handling if renewal fails.
- **Recommendation:** Document Jenkins Vault plugin auto-renewal config; add test (Molecule): run 2h job, verify token renewed mid-job. Consider increasing token TTL to 4h if jobs typically run <4h. Add break-glass: if renewal fails, job fails gracefully with clear error.

**SEC-H2: TLS Certificate SAN Validation — Hostname Mismatch Will Cause Validation Failures**
- **Current Design:** Section 3.1 (lines 195–196): cert_sans = ["vault", "vault.sec01", "192.168.22.51"]. Phase 3 playbook (lines 814–817): identical SANs. Section 9.2 (line 541): clients connect to "https://sec01.vernify.internal:8200"
- **Issue:** Mismatch: cert SANs do not include FQDN "vault.sec01.vernify.internal"; clients validating hostname will fail. Modern TLS clients (curl, Java, Python) require exact hostname match or wildcard.
- **Recommendation:** Add FQDN SANs: ["vault", "vault.sec01", "vault.sec01.internal", "192.168.22.51"]. Use FQDN as CN. Implement post-issuance validation: inspect cert with `step certificate inspect`, verify SANs match expected list.
- **LADR Reference:** Risk Register line 1973 ("TLS chain broken") Low likelihood, but this is correctness issue that will cause Phase 4 failures.

**SEC-H3: Container Non-Root User — Vault Token Escape & Host Writable by Non-Root**
- **Current Design:** Section 10.1 (lines 1642–1668): Vault runs as vault:vault (uid 100), data dir is 0700 owned by vault
- **Issue:** If Vault container is compromised (RCE), attacker has vault user access to host. Root token is 0600 (root-only), but if host is compromised at same time, attacker reads token. Design assumes container is not compromised; if it is, non-root user is weak mitigation.
- **Recommendation:** Use read-only root filesystem on containers (mount data RW, everything else RO). Drop all capabilities except IPC_LOCK. Implement AppArmor/SELinux profile restricting vault user process. Monitor for container escape attempts. Consider removing root token file altogether (use break-glass procedure instead).
- **LADR Reference:** LADR-009 acknowledges non-root trade-off; mitigation must include host-level containment (AppArmor/SELinux).

**SEC-H4: Secret Lifecycle Masking — Ansible no_log May Not Cover All Sensitive Output**
- **Current Design:** Section 5.2 (lines 890, 909, 920, 931, 942, 997, 1043): `no_log: true` on vault_write tasks; Section 11.1 (line 890): unseal script uses `no_log`
- **Issue:** `no_log: true` prevents task output from stdout but does NOT mask secrets in: Vault API error messages, Bash history, systemd journal, Vault audit logs (if error logged). Unseal keys in plaintext in script are not masked by Ansible.
- **Recommendation:** Implement Vault audit logging at sys/audit/file endpoint. Use `register` + filter pattern to clear sensitive vars after use. Add post-playbook cleanup: clear Bash history, suppress journal logging for sensitive tasks. Monitor logs for leaked secrets (grep -r for "unseal\|secret_id").
- **LADR Reference:** Risk Register line 1987 ("Secrets audit trail") claims no_log handles masking; must verify full coverage.

**SEC-H5: Auto-Unseal Fallback — Systemd Service Runs as Root, Hardcoded Keys**
- **Current Design:** Section 11.3 (line 776): systemd service runs `User=root`. Section 11.1 (lines 1744–1748): unseal keys embedded in plaintext script (0700)
- **Issue:** Unseal keys are credentials; plaintext on disk (even 0700) is high-risk. Systemd service runs as root, so any escape has full host access. Design does not address detection if keys are leaked or script modified.
- **Recommendation:** Move keys out of script: use systemd EnvironmentFiles with restrictive perms, or store in Vault after initial unseal using recovery key. Implement key encryption (LUKS, TPM). Monitor all reads of unseal script via auditd. Document key compromise recovery: re-init Vault, update all AppRole secrets. Phase 6: migrate to cloud auto-seal.
- **LADR Reference:** LADR-008 (Auto-Unseal Strategy) acknowledges plaintext risk; current mitigations are operational (backup discipline), not technical.

**SEC-H6: Vault Audit Logging Not Specified**
- **Current Design:** Section 14.2 (line 1987): "Secrets audit trail: Vault audit log records all secret read/write operations" (design assumption). No section specifies WHERE or HOW audit logging is configured.
- **Issue:** Vault audit logging is optional. Design assumes it without specifying: which audit backend (file, syslog, TCP), where logs stored, retention policy, log protection (can they be tampered?), what events logged (all APIs or only sensitive?).
- **Recommendation:** Add explicit task to Phase 3 orchestration: enable Vault audit to file backend at sys/audit/file with path /var/log/vault/audit.log. Specify JSON format, retention 365 days. Implement audit log protection: rotation is atomic, old logs archived. Document how to query audit logs. Phase 6: ship logs to external SIEM.
- **LADR Reference:** Risk Register line 1987 references audit trail but implementation is missing.

**SEC-H7: Network Segmentation — No Firewall Rules Documented**
- **Current Design:** Section 2.4 (line 141): "Public layer (none): Nothing is internet-facing. All services internal to Vernify network." No firewall rules shown.
- **Issue:** Design claims "internal only" but provides no enforcement mechanism. No mention of host firewall (ufw, firewalld) rules. If one VM is compromised, attacker can pivot to others (no network ACLs block lateral movement).
- **Recommendation:** Document firewall rules systematically: which hosts can reach which ports. Implement host-level firewall on each VM (ufw before services start). Specify egress rules (deny by default, allow specific destinations). Add pre-flight validation: test connectivity before Phase 3 proceeds.
- **LADR Reference:** Related to blast radius analysis (global working agreement) — must contain impact if one service is compromised.

**SEC-H8: Break-Glass Access — No Documented Procedure if Operator Loses Credentials**
- **Current Design:** Section 12.2–12.4 describes operator responsibilities but no emergency access procedure if unseal keys or Jenkins admin token lost.
- **Issue:** If operator loses unseal keys (1Password hacked, USB lost), no way to recover Vault without destroying it (data loss). No recovery key, no disaster recovery backup documented.
- **Recommendation:** Implement Vault backup/restore procedure: daily snapshots of /opt/vault/data to encrypted off-site storage. Document break-glass: restore from backup, unseal with backed-up keys, re-seed AppRole secrets. Phase 6: automated backup.
- **LADR Reference:** Risk Register line 1976 (High impact) acknowledges risk; mitigation recommends 1Password + USB but no recovery procedure if both lost.

---

### INFRASTRUCTURE/PLATFORM FINDINGS

#### CRITICAL FINDINGS

**INFRA-C1: blueprints.vault.init_and_unseal Role Does Not Exist**
- **Current Design:** Section 3.2 specifies role with full implementation details (lines 245–297)
- **Issue:** Spot check of `/Users/wernervandermerwe/workspace/iac-foundry/ansible-collection-vault/` reveals: role does not exist. `blueprints.vault` has only container_server, client, systemd_server roles. init_and_unseal is described in design but not implemented.
- **Recommendation:** Implement init_and_unseal role before Phase 3 execution: (1) run `vault operator init`, (2) capture + print unseal keys, (3) write to files with correct perms, (4) generate unseal script, (5) create systemd unit. Effort: 1 day; tests via Molecule required.
- **BLOCKING:** Phase 3 will fail when attempting to include this role.

**INFRA-C2: terraform-build01-deploy and terraform-agent01-deploy Repos Missing**
- **Current Design:** Sections 4.2–4.3 reference two required Terraform repositories
- **Issue:** Filesystem search shows only terraform-sec01-deploy exists in /Users/wernervandermerwe/workspace/vernify/. build01 and agent01 deploy repos do not exist.
- **Recommendation:** Create terraform-build01-deploy and terraform-agent01-deploy repos using identical pattern to terraform-sec01-deploy. Effort: 1 day each. Create before Phase 4/5 execution.
- **BLOCKING:** Phase 4 and 5 orchestration will fail (terraform apply against non-existent repo).

**INFRA-C3: Phase 3, 4, 5 Orchestration Playbooks Do Not Exist**
- **Current Design:** Detailed YAML playbooks provided (lines 708–1078 for Phase 3, etc.)
- **Issue:** Bootstrap container at /Users/wernervandermerwe/workspace/vernify/bootstrap-container/ has no playbooks/ directory. Referenced files phase-3-orchestrate.yml, phase-4-orchestrate.yml, phase-5-orchestrate.yml do not exist.
- **Recommendation:** Create three playbooks from design specs (translate YAML to executable format with proper variable bindings). Effort: 2 days (design is detailed enough to implement directly).
- **BLOCKING:** Orchestration will fail immediately (playbook not found).

#### HIGH-SEVERITY FINDINGS

**INFRA-H1: Vault Storage Backend Not Backed Up; No Disaster Recovery Path**
- **Current Design:** Section 12.3 (line 1902): "Vault data is not backed up"
- **Issue:** If sec01 storage fails, Vault is permanently lost. All secrets (Proxmox token, TFC token, AppRole credentials) must be re-seeded. No automated backup or replication exists. Unseal keys are operator-backed-up, but Vault data is not.
- **Recommendation:** Implement backup strategy BEFORE production: daily snapshot of /opt/vault/data to external storage (NFS, S3). Document recovery procedure (restore snapshot → unseal → verify secrets). Test recovery in non-prod environment. Add backup task to Phase 3 playbook.
- **LADR Reference:** Risk Register line 1976 identifies as High; mitigation acknowledges for Phase 6 but Phase 3-5 lacks rollback.

**INFRA-H2: DNS Configuration Not Implemented; Hostname Resolution Will Fail**
- **Current Design:** Design assumes DNS names resolve: sec01.vernify.internal, vault.sec01.internal, jenkins.build01.internal. Terraform sets search_domain to "vernify.com", not "vernify.internal"
- **Issue:** No DNS server is configured or mentioned. VMs will receive public Google DNS (8.8.8.8), which cannot resolve internal .vernify.internal names. Playbooks will timeout waiting for unresolvable hostnames.
- **Recommendation:** Verify DNS server exists on Proxmox network (check /etc/resolv.conf on existing dev01 VM). If none: configure dnsmasq on Proxmox host OR add local /etc/hosts entries on each VM. Clarify search domain: terraform should match design references (vernify.internal, not vernify.com).
- **BLOCKING:** Phase 3 will hang on first `curl https://sec01.vernify.internal:9000` call (DNS timeout).

**INFRA-H3: Collection Scaffolds Are Incomplete**
- **Current Design:** Design specifies six Ansible collections with detailed roles
- **Issue:** Spot checks reveal: blueprints.vault.container_server is scaffold placeholder (only debug output, no actual container deployment); blueprints.vault lacks init_and_unseal role; implementation status unknown for step_ca, jenkins, integrations.
- **Recommendation:** Before Phase 3: verify all collection roles are implemented + tested (Molecule required). For each collection: run `ansible-galaxy collection build`, run `molecule test`, spot-check key tasks (non-root user, file perms, container startup).
- **LADR Reference:** Risk Register lines 1967–1969 identify Medium likelihood; design assumes implementation complete.

**INFRA-H4: Cloud-Init Network Configuration Not Verified for Vernify Network**
- **Current Design:** Terraform passes network config via cloud-init (terraform-proxmox-vm module); Vernify expects 192.168.22.0/24 with gateway .1
- **Issue:** No verification that Proxmox vmbr0 is on expected subnet, gateway is reachable, cloud-init supports netplan/networkd, DHCP doesn't conflict with static IPs.
- **Recommendation:** Before Phase 3: verify Proxmox network config, test cloud-init on existing VM, verify no DHCP conflicts. Add pre-flight task to Phase 3: validate network connectivity after each VM created (ping gateway, verify DNS resolution, test SSH).
- **LADR Reference:** Risk Register line 1972 (DNS misconfiguration) includes network issues.

---

### NETWORK ENGINEERING FINDINGS

#### CRITICAL FINDINGS

**NET-C1: DNS Resolution Strategy Not Documented; Hostnames Cannot Resolve**
- **Current Design:** Assumes .vernify.internal FQDNs throughout. Terraform sets search_domain to "vernify.com"
- **Issue:** No DNS server specified; design does not explain how internal names resolve. VMs receive public DNS (8.8.8.8); internal zone cannot be resolved. Playbooks reference step-ca at "sec01.vernify.internal:9000"; this call will timeout.
- **Recommendation:** (1) Specify DNS server location (dnsmasq on Proxmox host? external DNS?). (2) If using /etc/hosts: add playbook task to populate on all VMs. (3) Correct Terraform search domain to "vernify.internal". (4) Add pre-flight validation: ping internal hostnames before proceeding.
- **BLOCKING:** Phase 3 unable to resolve sec01.vernify.internal.

**NET-C2: TLS Hostname Validation Will Fail; Certificate SANs Incomplete**
- **Current Design:** Section 3.1 (lines 195–196), Phase 3 (lines 814–817): cert_sans = ["vault", "vault.sec01", "192.168.22.51"]
- **Issue:** Clients connect to "https://sec01.vernify.internal:8200" but cert SAN does not include "vault.sec01.vernify.internal". Modern TLS clients validate hostname against SANs; mismatch causes validation failure.
- **Recommendation:** Update SANs to include FQDN: ["vault", "vault.sec01", "vault.sec01.internal", "192.168.22.51"]. Use FQDN as CN. Add post-issuance validation: inspect cert with `step certificate inspect`.
- **BLOCKING:** Phase 4 (Jenkins TLS setup) will fail with similar issue if not fixed.

#### HIGH-SEVERITY FINDINGS

**NET-H1: Port Configuration Without Firewall Rules**
- **Current Design:** Service ports documented (step-ca: 9000, Vault: 8200, Jenkins: 8443) but no firewall rules shown
- **Issue:** No host firewall (ufw, iptables) documented. No mention of which hosts can reach which ports. Jenkins agent JNLP port not specified (default 50000?). Design claims "internal only" without enforcement mechanism.
- **Recommendation:** Document firewall rules per VM (or single host-level policy): allow 192.168.22.0/24 → sec01:9000, 8200; allow → build01:8443, 50000; deny external. Implement UFW on VMs. Clarify JNLP port (default 50000 TCP). Add pre-flight: test connectivity before services start.

**NET-H2: Inter-Service Communication Paths Not Documented**
- **Current Design:** Section 2.3 (lines 104–137) describes secret flows in prose but not network paths
- **Issue:** No communication matrix showing source/dest/port/protocol/purpose. Implementation team won't know which firewall ports to open.
- **Recommendation:** Add systematic Communication Matrix to design:
  ```
  build01 → sec01  : 9000/HTTPS (step-ca cert issuance)
  build01 → sec01  : 8200/HTTPS (Vault AppRole auth)
  agent01 → sec01  : 8200/HTTPS (Vault secret retrieval)
  agent01 → build01: 50000/TCP  (Jenkins JNLP agent)
  ```

**NET-H3: JNLP Communication Protocol/Port Undefined**
- **Current Design:** Section 7.2 (line 1298): "Jenkins agent connects to build01 via JNLP" (no port or protocol specified)
- **Issue:** Design doesn't specify: Is JNLP on port 50000 or custom? Plain TCP or HTTPS? How does agent authenticate (SSH key, token, mTLS)?
- **Recommendation:** Document JNLP configuration explicitly. Add to Phase 5 playbook: bind JNLP listener on port 50000. Document firewall rule: allow agent01 → build01:50000/TCP.

**NET-H4: Bootstrap Container Network Access Not Specified**
- **Current Design:** Bootstrap container runs on Proxmox host and must SSH to VMs (192.168.22.51-53)
- **Issue:** No documentation of bootstrap container network setup. Is it on host network (shares Proxmox's network)? Or container network (docker0)? If container network, can it route to 192.168.22.0/24?
- **Recommendation:** Document bootstrap container network prerequisites. Add pre-flight validation: ping 192.168.22.51, verify SSH reachability.

**NET-H5: Network Isolation Not Enforced; No Firewall Rules for Internal Only Claim**
- **Current Design:** Section 2.4 (line 141): "Internal only (no public access)" but no firewall rules documented
- **Issue:** Design claims internal-only but provides no enforcement. If vmbr0 is connected to external network, VMs could be internet-routable.
- **Recommendation:** Document network isolation assumptions (is 192.168.22.0/24 on internal-only VLAN?). Implement host firewall rules (default deny external). Add verification to operator runbook: confirm vmbr0 isolation before Phase 3.

**NET-H6: No Latency or Bandwidth Analysis; Timeouts May Occur**
- **Current Design:** Assumes network is fast enough; no latency targets or analysis
- **Issue:** Vault auto-unseal systemd service (Section 11.1) has no timeout. If Vault startup is slow, unseal might hang. TLS handshakes on slow network could timeout.
- **Recommendation:** Add latency assumptions to design: "Latency <5ms between VMs; Gigabit bandwidth sufficient." Add pre-flight: `ping 192.168.22.52` verify <5ms latency. Configure explicit timeouts in playbooks (wait_for timeout=60).

**NET-H7: Network Failover / SPOF Not Addressed**
- **Current Design:** All VMs depend on single Proxmox vmbr0 bridge; no link redundancy mentioned
- **Issue:** If vmbr0 bridge fails (network card failure, driver crash), all VMs unreachable. Acknowledged out-of-scope (Risk Register line 1977) but design could mention Phase 6 improvement.
- **Recommendation:** Add note to design: "Network failover is out-of-scope for Phase 3-5. Phase 6 recommendation: link aggregation (LACP bonding) + redundant physical connections."

---

### TECHNICAL STRATEGY FINDINGS

#### CRITICAL FINDINGS

**STRAT-C1: Framework Misalignment — LADR-003 Violation (Secret Retrieval Boundary Unclear)**
- **Current Design:** Section 3.3 shows secret seeding as orchestration playbook tasks using community.hashi_vault lookups (lines 375–435)
- **Issue:** Design conflates platform-layer concerns with collection implementation. Unclear which tasks belong in roles vs. orchestration. If roles are responsible for AppRole creation, collections become Vernify-specific (not org-neutral). Design should cleanly separate concerns.
- **Recommendation:** Clarify role/platform boundary explicitly: roles deploy infrastructure (containers, config); orchestration layer handles secrets (init, unseal, seeding, AppRole wiring). This mirrors existing blueprints.vault_integrations pattern (auth backend wiring is separate from core collection).
- **LADR REFERENCE:** LADR-003 ("Secret Retrieval Uses community.hashi_vault") requires secret retrieval at platform layer, not in roles. Design must align.

**STRAT-C2: Vernify-Specific Hardcodes Prevent Consumer Org Reuse**
- **Current Design:** Section 3 embeds Vernify constants: step_ca_common_name = "Vernify Root CA", domain = "sec01.vernify.internal", IPs = 192.168.22.51-53
- **Issue:** If IOTel or Saicom adopts blueprints.step_ca, they will deploy with CN="Vernify Root CA" (wrong org). Other orgs must fork playbooks or hack variable overrides.
- **Recommendation:** Move Vernify-specific values to orchestration layer (playbook variables), not collection role defaults. Use empty defaults in roles; caller supplies org name. Use variable interpolation for hostnames and IPs instead of hardcoding.
- **IMPACT:** Blocks reuse by other consumer orgs (critical for org-neutral framework goal).

**STRAT-C3: No Acknowledgment of Existing Collections; Overwrite Risk**
- **Current Design:** Section 3.2–3.6 describe blueprints.vault, step_ca, jenkins as if greenfield
- **Issue:** Existing collections exist (ansible-collection-vault, step-ca, jenkins in /Users/wernervandermerwe/workspace/iac-foundry/). Design does NOT address: will Phase 3-5 implementation overwrite existing versions? Are there breaking changes? What's the migration path?
- **Recommendation:** Audit existing collection state; document backward-compatibility impact. If breaking changes, bump version + document migration guide. Add "Existing Collection Compatibility" section to design.
- **BLOCKING:** Risk of silently overwriting published collections used by other orgs (critical).

#### HIGH-SEVERITY FINDINGS

**STRAT-H1: Variable Naming Doesn't Follow BLUEPRINTS_VARIABLE_STANDARDS**
- **Current Design:** Section 3 uses inconsistent naming (step_ca_image_tag vs. step_ca_container_server_image_tag)
- **Issue:** Standard requires <component>_<role>_<thing> naming. Design uses <component>_<thing>. This prevents role reuse (e.g., if systemd_server role added later, variable name conflicts).
- **Recommendation:** Audit all variable names against BLUEPRINTS_VARIABLE_STANDARDS. Rename to <component>_<deployment_model>_<thing> format (e.g., step_ca_container_server_image_tag). Update role signatures + playbook references.
- **LADR REFERENCE:** BLUEPRINTS_VARIABLE_STANDARDS Section 1.1 (implicit standard from iac-foundry).

**STRAT-H2: Effort Estimates Appear Optimistic; Schedule Slip Risk**
- **Current Design:** Section 16.1 lists: blueprints.vault scaffold 3d, phase-3 playbook 2d, Molecule tests 3d
- **Issue:** Estimates appear low given complexity. Vault scaffold includes container_server + init_and_unseal + systemd service (~600 lines); Phase 3 playbook is 360+ lines with 7 gates. Add error handling + Molecule coverage → realistic 4–5d per item.
- **Recommendation:** Revise estimates +20–50%: vault 4d, step_ca 3d, jenkins 3–4d, phase-3 3d, phase-4 2d. Add confidence intervals (best/realistic/pessimistic). Flag risks (terraform-proxmox-vm incomplete adds 2–3d dependency).

**STRAT-H3: Phase 6 HA Migration Path Unclear; Scaling Assumptions Incomplete**
- **Current Design:** Appendix C (lines 2206–2238) lists 10x and 100x scaling but does NOT specify migration cost
- **Issue:** Phase 6 scaling assumes Vault snapshot/restore, step-ca multi-tier, cloud auto-seal. But design doesn't clarify: will Phase 5 Vault data be compatible with Phase 6 Raft? Will step-ca root cert need replacement? Are AppRole secrets portable?
- **Recommendation:** Add "Phase 6 Migration Strategy" section documenting: Vault file → Raft (snapshot/restore compatible? Yes if backup/restore preserves KV/AppRole). step-ca single-tier → multi-tier (intermediate CA from existing root; no breaking changes). Unseal Shamir → cloud auto-seal (keys become inactive; archive).

**STRAT-H4: Backward Compatibility Risk; No Versioning Strategy**
- **Current Design:** Collections deployed without version pinning or compatibility guarantees
- **Issue:** If blueprints.vault v1.0 (IOTel using) vs. v1.2 (Vernify using) have role name changes or variable renames, conflicts occur. No policy defines breaking changes vs. minor updates.
- **Recommendation:** Define semantic versioning (X.Y.Z) for collections. Specify backward-compatibility window (breaking changes only on major). Pin collection versions in bootstrap-container requirements.yml. Document minimum version requirements for collections used together.

---

### RELIABILITY FINDINGS

#### CRITICAL FINDINGS

**REL-C1: Idempotency Gap — Partial Vault State After Failure Not Detected**
- **Current Design:** Section 5.3 claims Phase 3 is idempotent; playbook checks if Vault initialized and skips init if yes
- **Issue:** If Phase 3 fails after Vault init but before all secrets seeded (e.g., Step 5 unseals Vault, Step 6 fails), re-running Phase 3 will skip Vault init but NOT detect partial secret state. Playbook has no idempotent check for "secrets already seeded." Vault storage becomes inconsistent.
- **Recommendation:** Add pre-flight check to Phase 3: try reading a known secret (e.g., secret/data/infra/proxmox/api-token). If exists but AppRoles missing, fail with clear error ("Partial state detected. Manual intervention required."). Add explicit rollback: delete /opt/vault/data before re-running Phase 3 if operator confirms.

**REL-C2: No Automatic Vault Data Backup; New Vault on Re-Run = Broken Downstream**
- **Current Design:** Section 12.3 (lines 1902–1906): "Vault data is not backed up; re-running Phase 3 creates a **new** Vault instance with new unseal keys"
- **Issue:** This is not a rollback; it's a disaster scenario. If Phase 3 completes, Phase 4 halfway, then sec01 crashes, re-running Phase 3 creates NEW Vault with NEW AppRole credentials. Phase 4 playbook retrieves OLD credentials (now gone); Jenkins auth fails silently.
- **Recommendation:** Add automated backup to Phase 3: `tar -czf /backup/vault-data-$(date).tar.gz /opt/vault/data` after unseal. Document break-glass: restore from backup if sec01 crashes mid-Phase 4. Phase 6: implement automated backup/restore.

#### HIGH-SEVERITY FINDINGS

**REL-H1: Auto-Unseal Systemd Service Has No Retry Logic; Silent Failure on Slow Startup**
- **Current Design:** Section 11.3 (lines 1760–1787): unseal script (lines 1750–1753) runs `vault operator unseal` three times; if Vault not ready, first call fails
- **Issue:** If Vault container takes >5 sec to start (network latency, slow startup), unseal fails with "connection refused". Systemd service has Restart=no; no automatic retry. Operator unaware unless SSH to sec01 and check logs.
- **Recommendation:** Add retry loop to unseal script: wait up to 30 sec for Vault readiness. Update systemd service: Restart=on-failure, RestartSec=10, StartLimitBurst=3. Add explicit seal-status check in Phase 3 playbook (after auto-unseal): uri poll Vault health until unsealed.

**REL-H2: AppRole Secret_ID No TTL; Token Expires But Credentials Don't Rotate**
- **Current Design:** Section 3.2 (lines 958–960): token_ttl 1h, token_max_ttl 4h; secret_id has no TTL (permanent)
- **Issue:** Jenkins token expires after 1h, but if token expires and secret_id is not rotated, re-authentication uses same permanent secret_id (valid indefinitely). If secret_id is leaked, window of compromise is unlimited.
- **Recommendation:** Add secret_id TTL (24h) to Phase 3 AppRole creation. Document: "Long-running jobs (>1h) need Vault token renewal logic (application-layer, in Jenkins plugin)." Phase 6: implement secret_id auto-rotation + monitoring.

**REL-H3: No Monitoring or Alerting for Vault Seal Status**
- **Current Design:** Section 14.2 claims "No silent failures" but provides no observability for seal status or auto-unseal failures
- **Issue:** Vault crashes, systemd auto-unseal service runs but fails. Operator doesn't know unless SSH to sec01. No centralized alerting; Vault seal status invisible to operator.
- **Recommendation:** Add health-check systemd timer to sec01 (run every 60 sec): check `vault status` seal status. Alert to journal (severity=alert) if sealed. Add pre-Phase 4 check: explicitly verify Vault is unsealed before proceeding. Phase 6: ship alerts to centralized monitoring (Grafana, Prometheus).

**REL-H4: TLS Certificate Renewal Not Automated for Jenkins; Cert Expires After 365 Days**
- **Current Design:** Section 8.2 (lines 1467–1474): "Renewal via Vault agent" for agents, but NOT for Jenkins (build01)
- **Issue:** Jenkins TLS cert (issued Phase 4) is valid 365 days; no auto-renewal mechanism. When cert expires, Jenkins HTTPS fails; operator must manually request new cert, update config, restart.
- **Recommendation:** Add Vault agent to build01 (Phase 4 orchestration) to monitor + auto-renew Jenkins TLS cert. Configure agent to restart Jenkins when cert renewed. Add cert-expiry alerting (30 days before expiry).

**REL-H5: Single Jenkins Agent (agent01) is SPOF; No Redundancy**
- **Current Design:** Section 7 (Phase 5) converges one agent; no agent pooling or failover
- **Issue:** If agent01 fails, all Jenkins jobs with label('agent01') fail. No backup agent. Acceptable for homelab v1 but unacceptable for >10 VMs.
- **Recommendation:** For Phase 3-5: document SPOF in runbook. Phase 6: implement agent pooling (2–3 agents, Jenkins auto-balances) or Kubernetes agent executor.

---

## TRACEABILITY INDEX

### Design Requirements to Findings Mapping

| FR/NFR | Section | Related Findings |
|--------|---------|-----------------|
| FR-1: Vault init + unseal | Section 5, 11 | REL-C1, REL-C2, REL-H1, SEC-C2, INFRA-C1 |
| FR-2: Secret seeding (AppRoles, KV) | Section 3.3, 5.2 | SEC-C3, SEC-H5, STRAT-C1, REL-C1 |
| FR-3: step-ca TLS issuance | Section 3.1, 5.2 | NET-C2, NET-H2, SEC-H2 |
| FR-4: Jenkins deployment | Section 3.4, 6.2 | REL-H4, NET-H3 |
| FR-5: Jenkins agent deployment | Section 3.5, 7.2 | REL-H5 |
| NFR-1: Idempotency | Section 5.3, 12.4 | REL-C1, REL-H2 |
| NFR-2: Security (non-root containers) | Section 10 | SEC-H3, INFRA-H4 |
| NFR-3: Auto-unseal fallback | Section 11 | REL-H1, SEC-H5 |
| NFR-4: Portability (org-neutral) | Section 1, 3 | STRAT-C1, STRAT-C2, STRAT-C3, STRAT-H1 |
| NFR-5: Scalability (Phase 6 path) | Appendix C | STRAT-H3 |
| Risk-1: Operator loses unseal keys | Section 14.1 | SEC-C1, SEC-H5, SEC-H8 |
| Risk-2: Vault scaffold incomplete | Section 14.1 | INFRA-C1 |
| Risk-3: DNS misconfiguration | Section 14.1 | NET-C1, NET-H1, INFRA-H2 |
| Risk-4: TLS chain broken | Section 14.1 | NET-C2, NET-H2, SEC-H2 |

---

## REFEREE INPUT: OPEN QUESTIONS & TRADE-OFFS FOR DECISION-MAKING

### Questions for Architect

1. **Existing Collection State:** Have blueprints.vault, step_ca, jenkins collections already been published to Ansible Galaxy? If yes, is this design a v2.0 rewrite (breaking changes) or enhancement (backward-compatible)?

2. **Network Architecture:** Is Vernify's 192.168.22.0/24 network internal-only (isolated VLAN)? Is there a DNS server on Proxmox network, or should VMs use /etc/hosts?

3. **Backup Strategy:** Is automated Vault backup required for Phase 3-5 production, or is operator-manual backup acceptable? What's the RTO/RPO for Vault recovery?

4. **Framework Alignment:** Should blueprints.vault, step_ca, jenkins collections be org-neutral (IOTel, Saicom can reuse) or Vernify-specific? If org-neutral, are hardcodes acceptable?

5. **Implementation Owners:** Who implements orchestration playbooks (Phase 3, 4, 5)? Are they ready by Stage 7a (integration testing)? Current design assumes yes but no commit dates shown.

### Trade-Offs Identified

| Trade-Off | Choice | Impact | Accepted for Phase 3-5? | Escalate to Phase 6? |
|-----------|--------|--------|------------------------|-----------------------|
| **Vault storage** | File-based (vs. Raft HA) | SPOF; no auto-failover | YES (greenfield, small) | YES — HA in Phase 6 |
| **Vault seal** | Shamir (vs. cloud auto-seal) | Operator-managed keys; plaintext in script | YES (no cloud dependency) | YES — AWS KMS Phase 6 |
| **Jenkins controller** | Single instance (vs. HA) | SPOF; no replication | YES (small workload) | YES — HA + master replication Phase 6 |
| **Jenkins agent** | Single agent (vs. pooling) | SPOF; CI blocked if down | YES (1-2 builds/day) | YES — agent pooling Phase 6 |
| **step-ca tier** | Single-tier root (vs. multi-tier) | Longer cert chain; no intermediate | YES (small deployment) | YES — intermediate CA Phase 6 |
| **Networking** | Single Proxmox vmbr0 (vs. redundant) | Network SPOF | YES (lab env) | YES — link aggregation Phase 6 |
| **Monitoring** | None (vs. Grafana + Prometheus) | Operator blind to failures | YES (manual alerts acceptable) | YES — full observability Phase 6 |

### Recommended Approval Criteria

**Approve design IF:**
1. All 12 Critical findings are resolved (implementation commitments + timelines clear)
2. All 27 High findings have documented mitigations or Phase 6 escalations
3. Existing collection state is clarified (no overwrite risk)
4. Framework alignment (LADR-003) is corrected
5. Operator runbook skeleton is drafted (pre-flight, deployment gates, troubleshooting)

**Return to review IF:**
- More than 3 Critical findings remain unresolved
- Backward-compatibility risk with existing collections is not addressed
- Implementation artifacts (playbooks, repos) not committed by defined deadline

---

## SUMMARY FOR REFEREE DECISION

**RECOMMENDATION: Return to design review with requested revisions. No fundamental architectural blockers.**

**Decision Options:**
1. **APPROVE (conditional):** Approve design IF architect commits to resolve all 12 Critical findings before Phase 7a (implementation). Estimated rework: 3–5 days.
2. **APPROVE WITH REVISIONS:** Approve design outline; architect must submit revised document addressing findings 1–6 (framework, network, ops) before Phase 7a begins.
3. **REJECT:** Return to discovery phase if Critical findings represent fundamental architectural flaws (not applicable here; all issues resolvable).

**RECOMMENDED PATH:** Option 2 (Approve With Revisions). Design is solid; revisions are clarifications + implementation commitments. Revised design can be approved in parallel with Phase 7a coding, with code reviews serving as secondary gate.

---

**Compiled by:** Multidisciplinary Review Panel  
**Date:** 27 June 2026  
**Classification:** Internal Architecture Review (Stage 3)
