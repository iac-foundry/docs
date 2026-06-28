# LADR-010: Bootstrap Script as Single Orchestration Entry Point

**Status:** Accepted
**Date:** 28 June 2026
**Owner:** Platform Engineering
**Supersedes:** N/A
**Superseded By:** N/A (upgrade path to Terraform/Ansible orchestration in Phase 6)
**Related:** [../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md](../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md)

---

## 1. Context

- **Deployment scope:** Phases 3-5 require orchestration of 3 major phases, each with dependencies:
  - **Phase 3:** Provision sec01 VM; deploy step-ca + Vault; initialize Vault; seed secrets
  - **Phase 4:** Provision docker01 VM; deploy Jenkins container; wire Vault auth; seed Job DSL
  - **Phase 5:** Provision agent01 LXC; deploy Jenkins agent; install toolchain; verify connectivity
- **Orchestration options:**
  - Three separate Ansible playbooks (requires operator to run each in order; error handling complex)
  - Terraform automation (can't converge Ansible; limited to infrastructure provisioning)
  - Single shell script (orchestrates everything; all logic in one place; single entry point)
- **Operator experience:** Vernify target user is platform operator, not Terraform/Ansible expert. Prefers single command to remember.

## 2. Decision

**Vernify will use a single shell script (`bootstrap-phase-3-5.sh`) as the unified orchestration entry point for all three phases. The script is idempotent, handles errors, and provides clear progress feedback.**

**Script responsibilities:**
- **Phase selection:** Accepts `--phase 3|4|5|all` flag; defaults to `all` (run all phases).
- **Terraform provisioning:** Invokes `terraform apply` to provision VMs/LXCs.
- **Ansible convergence:** Invokes Ansible playbooks to deploy services.
- **Secret seeding:** Runs Job DSL pipelines, seeds Vault secrets, configures Vault auth.
- **Operator gates:** Pauses for critical human decisions (e.g., "Unseal keys backed up? [y/n]").
- **Idempotency:** Re-running script after failures is safe; completed phases are skipped.
- **Rollback instructions:** On failure, script prints clear next steps.

**Location:** `github.com/vernify/bootstrap-container/scripts/bootstrap-phase-3-5.sh`

**Usage:**
```bash
./scripts/bootstrap-phase-3-5.sh \
  --proxmox-endpoint https://proxmox.vernify.internal:8006 \
  --proxmox-password "$PROXMOX_PASSWORD" \
  --tfc-token "$TFC_TOKEN" \
  --step-ca-provisioner-password "$STEP_CA_PASSWORD"
```

## 3. Rationale

### Benefits of single script entry point:
- **Ease of use:** Operator runs one command; script handles all coordination internally.
- **Centralized logic:** All orchestration decisions in one file (600-800 lines); easier to understand flow.
- **Idempotency:** Script checks which phases are complete; re-run after failures without re-initializing Vault.
- **Operator gates:** Human approval points (e.g., "unseal keys backed up?") built into script; ensures safety checks are not skipped.
- **Error handling:** Failed step halts script; operator sees clear error message + next steps.
- **Single rollback point:** To undo deployment, operator deletes Proxmox VMs; re-run script from beginning.

### Trade-offs accepted:
- **Shell script overhead:** bash is not a programming language designed for complex orchestration; limited error handling, no type safety.
  - **Mitigation:** Script uses strict mode (`set -e`, `set -u`, `set -o pipefail`); extensive comments; functions modularized by phase.
- **Non-declarative:** Script is imperative (step-by-step); harder to reason about than Terraform (declarative).
  - **Acceptable:** Phases are inherently sequential (can't deploy Phase 5 before Phase 3); imperative model maps naturally to sequential deployment.
- **Limited debugging:** bash errors don't always provide stack traces; operator must read script to understand failure.
  - **Mitigation:** Detailed logging at each step; script prints `[STEP X] <description>` before each major operation.
- **No dry-run mode:** Script actually runs Terraform/Ansible; no way to preview what will happen.
  - **Acceptable for Phase 3-5:** Immutable Proxmox VMs allow safe re-run; dry-run deferred to Phase 6 if needed.

## 4. Alternatives Considered

### Option A — Three separate Ansible playbooks
| | |
|---|---|
| **Pros** | All logic in declarative YAML; reusable playbooks; idempotent |
| **Cons** | Operator must manually run 3 commands in order; Terraform provisioning separate; error handling across playbooks complex; no central gates |
| **Reason rejected** | Higher operator burden; coordination nightmare. Single script simpler. |

### Option B — Terraform-only orchestration
| | |
|---|---|
| **Pros** | Declarative; version-controllable; works with TFC |
| **Cons** | Terraform cannot converge Ansible playbooks; limited to infrastructure provisioning (VMs/LXCs); can't seed Vault secrets or Job DSL jobs |
| **Reason rejected** | Terraform alone insufficient. Must combine Terraform + Ansible. Script orchestrates both. |

### Option C — Ansible playbook that calls terraform
| | |
|---|---|
| **Pros** | Declarative YAML for orchestration; Ansible is familiar to operators |
| **Cons** | YAML verbosity for control flow; Ansible not designed for heavy scripting; `terraform` module needs error handling |
| **Reason rejected** | Shell script is simpler and more transparent. Ansible playbooks already converge services (good use); orchestration script is separate concern. |

### Option D — Manual step-by-step procedures (no automation)
| | |
|---|---|
| **Pros** | Complete operator control; no hidden logic |
| **Cons** | Error-prone; slow (Phase 3 takes 30+ minutes manually); high operational burden; no idempotency |
| **Reason rejected** | Violates modern infrastructure-as-code principles. Automation expected. |

## 5. Consequences

### Positive:
- **Fast deployment:** Operator runs one command; entire Phase 3-5 completes in ~2-3 hours (minimal human interaction).
- **Reproducible:** Script can be run on different Proxmox environments (dev, test, prod) with same code; only input variables change.
- **Auditable:** Script is version-controlled in `bootstrap-container` repository; all deployment logic is tracked.
- **Reversible:** Deleting VMs + re-running script achieves rollback; no special rollback procedures.

### Negative:
- **Bash complexity:** Shell script error handling is fragile; unexpected failures can be hard to debug.
  - **Mitigation:** Extensive CI testing (unit tests for script logic; integration tests with real Proxmox). Runbook includes debugging section.
- **Limited flexibility:** Operator must run script with all 5 flags; can't easily customize (e.g., "run Phase 3 with different step-ca config").
  - **Mitigation:** Phase 6+ can refactor to Terraform/Ansible if customization needs grow. Script sufficient for current scope.
- **Tight coupling:** Bootstrap script tightly coupled to current Terraform module + Ansible role structure. Refactoring Terraform/Ansible requires updating script.
  - **Acceptable:** Script and playbooks live in same repository (`bootstrap-container`); versions stay in sync naturally.

## 6. Implementation Notes

- **Script location:** `github.com/vernify/bootstrap-container/scripts/bootstrap-phase-3-5.sh`
- **Phases:** Each phase is a function: `phase_3()`, `phase_4()`, `phase_5()`.
- **Main flow:**
  ```bash
  main() {
    validate_inputs
    check_prerequisites
    [[ $PHASES =~ 3 ]] && phase_3
    [[ $PHASES =~ 4 ]] && phase_4
    [[ $PHASES =~ 5 ]] && phase_5
    print_summary
  }
  ```
- **Idempotency checks:** Each phase checks if VM exists / services are running before re-provisioning.
  ```bash
  phase_3() {
    if terraform show | grep -q "sec01"; then
      log "Phase 3 already deployed; skipping provisioning"
    else
      terraform apply -target=proxmox_vm.sec01
    fi
    ansible-playbook playbooks/phase-3.yml
  }
  ```
- **Operator gates:** Critical points require human approval:
  ```bash
  read -p "Unseal keys backed up securely? (y/n) " -n 1 -r
  [[ $REPLY =~ ^[Yy]$ ]] || { log "Backup unseal keys first"; exit 1; }
  ```
- **Error handling:** Script exits on any error; prints clear next steps:
  ```bash
  trap 'log "ERROR at line $LINENO"; print_recovery_instructions' ERR
  set -e
  ```
- **Validation:** CI test: run script in sandbox Proxmox environment; verify all services healthy after completion.

## 7. References

- [../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md](../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md) — Section 4.5: Bootstrap Script Model
- [PHASE_3_5_OPERATOR_RUNBOOK.md](../PHASE_3_5_OPERATOR_RUNBOOK.md) — Operator-facing deployment procedures
- `bootstrap-container` repository: https://github.com/vernify/bootstrap-container
