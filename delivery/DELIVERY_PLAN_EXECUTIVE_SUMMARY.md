# Vernify Phase 3-5 Delivery Plan — Executive Summary

**Document:** `/docs/delivery/PHASE_3_5_DELIVERY_PLAN.md`  
**Date:** 28 June 2026  
**Status:** Ready for Implementation Handoff

---

## Overview

The approved technical design for Vernify Phases 3–5 (sec01 security core, docker01 CI core, agent01 build capacity) has been broken into a comprehensive delivery plan consisting of **47 independently testable work items**, each completable in ≤2 days.

**Key Numbers:**
- **Total effort:** 34 story days (~68 story points)
- **Estimated duration:** 10 weeks (with QA + sign-off)
- **Critical path (strictly sequential):** ~21 days
- **With parallelization:** ~14-16 days execution + 4 weeks QA/testing
- **Parallel workstreams:** 5 (collections, terraform, bootstrap, CI/CD, docs)
- **Recommended team:** 8 people (peak concurrent: 5 in weeks 3–5)

---

## What Gets Built (6 Implementation Phases)

### Phase A: Collection Scaffolding & Dependencies (2 days)
Audit 5 Ansible collections, verify they exist and are complete, identify missing roles, set up external dependencies.

**Deliverables:**
- Collection inventory (12 roles identified)
- requirements.yml with all dependencies pinned
- Verification that terraform-proxmox-vm module is complete

**Risk:** terraform-proxmox-vm module is external blocker; verify immediately in A-03.

---

### Phase B: Ansible Collection Implementations (13 days, parallelizable across 6 days with teams)
Implement all 12 roles across 5 collections:
- **blueprints.step_ca** (2 roles): container_server, issue_certificate
- **blueprints.vault** (3 roles): container_server, init_and_unseal, + docs
- **blueprints.jenkins** (2 roles): container_server, agent + tests
- **blueprints.jenkins_integrations** (1 role): vault_auth
- **blueprints.vault_integrations** (1 role): vault_agent
- **Plus:** Molecule test playbooks for all roles (automated testing framework)

**Deliverables:** 12 complete role implementations + 4 Molecule test suites

**Quality Gate:** All Molecule tests pass locally; ansible-lint passes; no hardcoded secrets

**Effort Breakdown:**
- 5 collections can be implemented in parallel (1–2 engineers per collection)
- Critical dependencies: step_ca container → issue_certificate; vault container → init_and_unseal
- Critical path: ~6 days if 2–3 Ansible engineers working in parallel

---

### Phase C: Terraform Repositories (3 days, parallelizable)
Three repositories for VM/LXC provisioning:
- **terraform-sec01-deploy:** 8 vCPU, 16 GB, 50 GB (security core)
- **terraform-docker01-deploy:** 4 vCPU, 8 GB, 50 GB (container host)
- **terraform-agent01-deploy:** 2 vCPU, 4 GB, 20 GB LXC (build capacity)

All use terraform-proxmox-vm module (external dependency).

**Deliverables:** Three production-ready repos + GitHub Actions workflows

**Quality Gate:** terraform validate + terraform plan pass (no errors)

**Effort Breakdown:**
- All three repos can be built in parallel (copy pattern, customize vars)
- ~1 Terraform engineer can complete all 3 in 3 days

---

### Phase D: Bootstrap Script & Orchestration (4 days)
**Single unified orchestration entry point** that coordinates all three phases:

- `bootstrap-phase-3-5.sh` — main script (idempotent, handles all 3 phases)
- `phase-3-orchestrate.yml` — Terraform + Ansible for sec01 (step-ca, Vault, unseal gate)
- `phase-4-orchestrate.yml` — Jenkins deployment + Vault integration + Job DSL seeding
- `phase-5-orchestrate.yml` — agent01 provisioning + Jenkins agent + Vault agent

**Deliverables:** Single entry script + three orchestration playbooks

**Quality Gate:** Dry-run mode (terraform plan + ansible --check) succeeds end-to-end

**Critical Path:** Must wait for Phase B + C complete; ~1 senior architect, 4 days

---

### Phase E: CI/CD & Testing (3 days)
GitHub Actions workflows + Molecule test automation:

- ansible-lint on all playbooks
- Molecule test automation (docker driver)
- Terraform validate + plan in CI
- Security scanning (Ansible syntax, tfsec)
- End-to-end dry-run test script
- CI/CD documentation + health dashboard

**Deliverables:** 6 GitHub Actions workflows + test infrastructure

**Quality Gate:** All workflows passing on main branch

**Can run parallel with Phase D** (doesn't block orchestration)

---

### Phase F: Documentation & Runbooks (2 days)
Comprehensive documentation:

- **LADRs 005–010:** Architectural decisions (single-tier CA, file-based Vault, auto-unseal, non-root containers, operator gates)
- **Operator Runbook:** Day 1–3 procedures, gates, troubleshooting, manual recovery
- **Disaster Recovery:** Vault backup/restore, unseal key recovery, secret rotation, recovery scenarios
- **API Reference:** Vault, step-ca, Jenkins endpoints + integration examples

**Deliverables:** 4 documentation files (~2000 words each)

**Quality Gate:** No clarity gaps; operators understand procedures

**Can run parallel with Phases D + E** (SMEs provide input during implementation)

---

## Implementation Order (Critical Path)

```
Week 1-2:   Phase A (scaffold) 
            ↓
Week 2-4:   Phase B (collections) + Phase C (terraform) in PARALLEL
            ↓
Week 4-5:   Phase D (bootstrap script) - waits for B+C
            ↓
Week 5-6:   Phase E (CI/CD) + Phase F (docs) in PARALLEL
            ↓
Week 6-9:   End-to-End Testing & QA (live deployment on Vernify)
            ↓
Week 10:    Sign-Off & Deployment Ready
```

**Key insight:** Phases B + C are parallelizable; saves ~3 days. Phase D acts as sequencing point (waits for both).

---

## Parallel Workstreams

Five independent workstreams can run concurrently:

| Workstream | Team | Effort | Duration | Notes |
|---|---|---|---|---|
| **Collections (B)** | 2–3 Ansible engineers | 26 points | 6 days | 5 collections parallelizable |
| **Terraform (C)** | 1–2 Infra engineers | 6 points | 3 days | 3 repos parallelizable |
| **Bootstrap (D)** | 1 Senior architect | 8 points | 4 days | Sequential (D-01 → D-02 → D-03) |
| **CI/CD (E)** | 1 DevOps engineer | 6 points | 3 days | Can start during Phase B |
| **Docs (F)** | 1 Tech writer + SMEs | 4 points | 2 days | Can start during Phase D |

**Peak concurrent effort:** Weeks 3–5 (all 5 workstreams + QA planning)

---

## Resource Requirements

**Recommended team composition:**
- **2–3 Ansible experts** (collections)
- **1–2 Infrastructure/Terraform engineers** (terraform repos)
- **1 DevOps/CI-CD engineer** (GitHub Actions)
- **1 Senior Ansible architect** (bootstrap script + orchestration)
- **1 Technical writer** (documentation)
- **1 QA engineer** (testing + sign-off)
- **1 Tech lead/PM** (coordination + reviews)

**Total:** ~8 people; peak concurrent: 5 (weeks 3–5)

**Time commitment per person:**
- Ansible experts: 4–5 weeks
- Infra engineers: 2 weeks
- DevOps engineer: 3 weeks
- Senior architect: 1 week (weeks 4–5, after B+C complete)
- Tech writer: 1.5 weeks (parallel with other phases)
- QA engineer: 2 weeks testing + 2 weeks sign-off
- Tech lead: Throughout (10 weeks)

---

## Quality Gates (Go/No-Go Criteria)

### Phase A Gate
- [ ] All 5 collections exist + are writable
- [ ] 12 roles identified + status documented
- [ ] terraform-proxmox-vm module verified complete
- [ ] No blockers identified

### Phase B Gate
- [ ] All 12 roles implemented + complete
- [ ] Molecule tests pass locally for all roles
- [ ] ansible-lint passes (0 errors)
- [ ] No hardcoded Vernify-specific values
- [ ] All sensitive data marked `no_log: true`

### Phase C Gate
- [ ] terraform validate passes for all 3 repos
- [ ] terraform plan shows expected resources (no errors)
- [ ] Variables documented + .tfvars.example provided
- [ ] GitHub Actions workflows running

### Phase D Gate
- [ ] bootstrap-phase-3-5.sh runs in dry-run (terraform plan + ansible --check)
- [ ] All playbooks syntax-check passes
- [ ] Operator gates implemented + functional
- [ ] Error handling + rollback instructions documented

### Phase E Gate
- [ ] All GitHub Actions workflows passing
- [ ] Molecule tests pass in CI for all collections
- [ ] E2E dry-run test succeeds
- [ ] Security scanning passes

### Phase F Gate
- [ ] LADRs 005–010 written + cross-linked
- [ ] Operator runbook complete + reviewed by SMEs
- [ ] Disaster recovery procedures validated (restore test)
- [ ] API reference + integration guide complete

### Live Testing Gate
- [ ] Phase 3–5 deployment succeeds on Vernify infrastructure
- [ ] Vault unsealed, secrets seeded, accessible
- [ ] Jenkins deployed, AppRole auth working, Job DSL pipelines seeded
- [ ] agent01 connected + toolchain installed + healthy
- [ ] All operator gates work as documented
- [ ] Rollback procedures tested + validated

---

## Risk Mitigation Highlights

### Top 3 Risks

| Risk | Mitigation | Contingency |
|---|---|---|
| **terraform-proxmox-vm incomplete** | Verify in A-03 immediately; escalate if issues | Use Proxmox provider directly (no module) |
| **Vault init JSON parsing fails** | Test in B-05 Molecule tests; handle version differences | Manual key extraction + operator unseals manually |
| **Operator skips unseal key backup** | Large warning text + clear runbook (F-02) | Recovery from external backup (if operator has one); Phase 6: implement KMS auto-seal |

### Mitigations by Phase

- **Phase A:** External dependency verification (terraform-proxmox-vm)
- **Phase B:** Molecule tests validate all role functionality locally
- **Phase C:** terraform validate catches configuration errors early
- **Phase D:** Dry-run test (E-05) catches orchestration issues before live deployment
- **Phase E:** GitHub Actions CI/CD ensures no regressions
- **Phase F:** Operator runbook + disaster recovery guide prevent operational surprises

---

## Definition of Done

**Delivery is complete when:**

1. ✅ All 47 work tickets implemented + merged to main
2. ✅ All GitHub Actions workflows passing on main branch
3. ✅ Molecule tests pass locally + in CI/CD for all collections
4. ✅ terraform validate passes for all 3 repos
5. ✅ bootstrap-phase-3-5.sh runs end-to-end in dry-run mode (0 errors)
6. ✅ Live Phase 3–5 deployment succeeds on Vernify infrastructure (all 3 VMs running)
7. ✅ Vault unsealed, secrets seeded, AppRole auth verified working
8. ✅ Jenkins deployed, Vault plugin configured, Job DSL pipelines seeded
9. ✅ agent01 appears in Jenkins UI as healthy connected node
10. ✅ Operator runbook tested + operators trained + sign-off given

**Success = Self-hosted infrastructure, bootstrap container retired, ready for Phase 6**

---

## Timeline & Milestones

| Milestone | Target Date | Status | Dependencies |
|---|---|---|---|
| Phase A complete (scaffold) | Week 2 (Day 7) | Ready | terraform-proxmox-vm verified |
| Phase B complete (collections) | Week 4 (Day 21) | Ready | Phase A complete |
| Phase C complete (terraform) | Week 4 (Day 21) | Ready | terraform-proxmox-vm verified |
| Phase D complete (bootstrap) | Week 5 (Day 28) | Ready | Phase B + C complete |
| Phase E complete (CI/CD) | Week 6 (Day 35) | Ready | Phase B complete |
| Phase F complete (docs) | Week 8 (Day 42) | Ready | All implementation phases |
| E2E dry-run test pass | Week 6 (Day 35) | Ready | Phase D complete |
| Live testing begin | Week 7 (Day 42) | Ready | Phase E + E2E dry-run complete |
| Live testing complete | Week 9 (Day 56) | Ready | All phases + live deployment |
| Sign-off & approval | Week 10 (Day 63) | Ready | Live testing complete |
| **Production ready** | **Week 10** | **Target** | **All gates passed** |

**Estimated project completion: 10 weeks** (including testing, QA, sign-off)

---

## Next Steps

1. **Assign resources** (8 people to 5 workstreams)
2. **Verify terraform-proxmox-vm module** (A-03, immediate)
3. **Kick-off Phase A** (collection scaffolding)
4. **Plan Phase B + C parallelization** (collections can start immediately after A)
5. **Schedule weekly sync** (coordination + blocker resolution)
6. **Track progress** against 47 tickets + quality gates

---

## Key Insights for Stakeholders

### What's Different from Initial Design

The delivery plan translates the 2800-line design document into **actionable work items** with:

✅ **Smaller scope:** Each ticket is ≤2 days (not multi-week epics)  
✅ **Explicit dependencies:** DAG showing which tickets block which  
✅ **Testable acceptance criteria:** Not "looks good," but specific, measurable outcomes  
✅ **Parallelization strategy:** 5 workstreams can run simultaneously (saves 3+ weeks)  
✅ **Quality gates:** Clear go/no-go criteria before moving to next phase  
✅ **Risk mitigation:** Identified 12+ risks with specific mitigations  
✅ **Resource plan:** Exactly who does what, when, with how much effort  

### Why 10 Weeks (vs. Theoretical 6 Days)

1. **QA + Testing (4 weeks):** Live deployment on Vernify + operator training + sign-off
2. **Sequencing:** Some phases must wait for others (Phase D waits for B+C)
3. **Realistic effort estimates:** Include code review, debugging, documentation, rework
4. **Buffer:** ~10% schedule buffer for unexpected issues (typical in infrastructure work)

### Go/No-Go Decision Points

**Three critical decision points before Phase 5:**

1. **After Phase A:** terraform-proxmox-vm module complete? → Go/No-Go Phase B
2. **After Phase D:** Dry-run test passes? → Go/No-Go live testing
3. **After Phase E2E test:** Live deployment succeeds? → Go/No-Go production deployment

If any gate fails, escalate immediately; do not proceed to next phase.

---

## Handoff to Implementation Team

**Everything needed to execute is documented in:**
- `/docs/delivery/PHASE_3_5_DELIVERY_PLAN.md` (47 tickets with full AC + deps)
- `/docs/design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md` (original design for reference)

**Immediate actions for tech lead:**
1. Assign resources to 5 workstreams
2. Verify terraform-proxmox-vm module (A-03) — blocker if incomplete
3. Schedule weekly sync + blocker resolution meeting
4. Create project board with 47 tickets
5. Set up GitHub issues + PR review process

**Expected outcome:** Self-hosted Vernify infrastructure (sec01 + docker01 + agent01) fully operational, ready for Phase 6 (monitoring, HA, scale).

---

**Prepared by:** Delivery Planning (Stage 6)  
**Date:** 28 June 2026  
**Status:** READY FOR HANDOFF  
**Next Phase:** Stage 7 Implementation (Week 1)
