# Vernify Phase 3-5 Delivery Plan

This directory contains the complete delivery plan for Phases 3–5 of the Vernify bootstrap project (sec01 security core, docker01 CI core, agent01 build capacity).

## Documents

### 1. **DELIVERY_PLAN_EXECUTIVE_SUMMARY.md**
**For stakeholders & decision-makers**

Quick overview of:
- What gets built (6 implementation phases)
- Who works on what (team allocation)
- Timeline & milestones (10 weeks)
- Quality gates & go/no-go criteria
- Top 3 risks & mitigations

**Start here if you need:** High-level overview, timeline, resource allocation

**Read time:** 15 minutes

---

### 2. **PHASE_3_5_DELIVERY_PLAN.md** 
**Detailed work breakdown for implementation teams**

Complete delivery plan containing:
- **Executive Summary:** Metrics, phases, dependencies
- **47 work tickets:** Each with story points, acceptance criteria, dependencies
- **Dependency graph:** Visual DAG showing what blocks what
- **Effort summary table:** Points by phase, duration estimates
- **Critical path analysis:** Longest chain of dependencies
- **Parallel workstreams:** 5 independent tracks (collections, terraform, bootstrap, CI/CD, docs)
- **Resource allocation:** Team composition, skill mix, parallel tracks
- **Gantt timeline:** Visual schedule
- **Quality gates:** Go/no-go criteria at each phase
- **Hand-off brief:** Everything implementation team needs to know
- **Risk & mitigation:** 12+ identified risks with specific mitigations

**Start here if you need:** Detailed work breakdown, dependencies, effort estimates, quality gates

**Read time:** 60 minutes

---

## Quick Facts

| Metric | Value |
|---|---|
| **Total Tickets** | 47 |
| **Total Story Points** | 68 |
| **Estimated Duration (with QA)** | 10 weeks |
| **Critical Path (sequential only)** | ~21 days |
| **With Parallelization** | ~14-16 days execution |
| **Parallel Workstreams** | 5 |
| **Recommended Team Size** | 8 people |
| **External Blockers** | 1 (terraform-proxmox-vm module) |

---

## Implementation Phases

| Phase | Name | Tickets | Points | Days | Duration |
|---|---|---|---|---|---|
| **A** | Collection Scaffolding | A-01 to A-04 | 4 | 2 | Weeks 1–2 |
| **B** | Ansible Collections | B-01 to B-12 | 26 | 13 (6 critical path) | Weeks 2–4 |
| **C** | Terraform Repos | C-01 to C-06 | 6 | 3 | Weeks 3–4 |
| **D** | Bootstrap & Orchestration | D-01 to D-03 | 8 | 4 | Week 5 |
| **E** | CI/CD & Testing | E-01 to E-06 | 6 | 3 | Weeks 6–7 |
| **F** | Documentation | F-01 to F-04 | 4 | 2 | Weeks 7–8 |
| **QA** | End-to-End Testing | QA-01 to QA-02 | 6 | 4 | Weeks 7–10 |
| **TOTAL** | — | **47** | **68** | **34** | **~10 weeks** |

---

## Critical Path

```
terraform-proxmox-vm (ext. dep.)
  → Phase A (scaffolding) [2d]
  → Phase B (collections) [6d critical path, 13d total]
  + Phase C (terraform) [3d, parallel]
  → Phase D (bootstrap) [4d, waits for B+C]
  → Phase E (CI/CD) [3d, can overlap with D]
  → Phase F (Docs) [2d, can overlap]
  → QA/Testing [4 weeks live deployment + sign-off]
  
Total: ~10 weeks (including testing + QA)
```

---

## Parallel Workstreams

Five independent workstreams can run concurrently:

1. **Collections (B):** 2–3 Ansible engineers, 6 days critical path (13 days total)
2. **Terraform (C):** 1–2 Infra engineers, 3 days
3. **Bootstrap (D):** 1 Senior architect, 4 days (sequential internally)
4. **CI/CD (E):** 1 DevOps engineer, 3 days (can start during B)
5. **Docs (F):** 1 Tech writer, 2 days (can start during D)

**Peak concurrent effort:** Weeks 3–5 (all workstreams + QA planning)

---

## Quality Gates (Go/No-Go)

| Gate | Phase | Criteria | Pass = ? |
|---|---|---|---|
| **A** | Scaffolding | Collections exist, terraform-proxmox-vm verified | No blockers |
| **B** | Collections | Molecule tests pass, ansible-lint passes, no hardcoded secrets | All roles tested |
| **C** | Terraform | terraform validate passes for all 3 repos | No errors |
| **D** | Bootstrap | Dry-run test succeeds end-to-end | terraform plan + ansible --check OK |
| **E** | CI/CD | All GitHub Actions workflows passing | Green builds |
| **F** | Docs | Runbook reviewed, no clarity gaps | SMEs sign-off |
| **Live Test** | QA | Phase 3–5 deployment succeeds on Vernify | All services running + connected |

**Any gate fails → Escalate immediately; do not proceed to next phase**

---

## Resource Allocation

**Recommended team:**
- 2–3 Ansible experts (collections)
- 1–2 Infrastructure engineers (terraform)
- 1 DevOps engineer (CI/CD)
- 1 Senior architect (bootstrap script)
- 1 Technical writer (documentation)
- 1 QA engineer (testing + sign-off)
- 1 Tech lead / PM (coordination)

**Total:** ~8 people (peak concurrent: 5 in weeks 3–5)

---

## Key Risks & Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| **terraform-proxmox-vm incomplete** | Blocks all terraform repos | Verify in A-03 immediately; escalate if incomplete |
| **Vault init output parsing fails** | Unseal keys not captured | Test in B-05; handle version differences |
| **Operator skips unseal key backup** | Vault unrecoverable | Clear warning + operator runbook (F-02) |

See full plan for 12+ additional risks with mitigations.

---

## Next Steps (For Tech Lead)

1. **Assign resources** to 5 workstreams
2. **Verify terraform-proxmox-vm module** (A-03) — CRITICAL BLOCKER
3. **Create project board** with 47 tickets in GitHub Projects
4. **Schedule kick-off** (brief team on phases, workstreams, timeline)
5. **Schedule weekly sync** (progress tracking + blocker resolution)

---

## Definition of Done

**Delivery is complete when:**

- ✅ All 47 tickets implemented + merged
- ✅ All GitHub Actions workflows passing
- ✅ Molecule tests pass locally + in CI
- ✅ bootstrap-phase-3-5.sh runs dry-run successfully
- ✅ Live Phase 3–5 deployment succeeds on Vernify
- ✅ Vault unsealed, secrets seeded, Jenkins AppRole working
- ✅ agent01 healthy in Jenkins UI with toolchain installed
- ✅ Operator runbook tested + team trained
- ✅ Sign-off from tech lead + QA engineer + operators

**Success = Self-hosted infrastructure, ready for Phase 6**

---

## Document Locations

- **Architecture Design:** `/docs/design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md`
- **Delivery Plan (Full):** `/docs/delivery/PHASE_3_5_DELIVERY_PLAN.md`
- **Executive Summary:** `/docs/delivery/DELIVERY_PLAN_EXECUTIVE_SUMMARY.md`
- **This README:** `/docs/delivery/README.md`

---

**Prepared by:** Delivery Planning (Stage 6)  
**Date:** 28 June 2026  
**Status:** READY FOR HANDOFF  
**Next Phase:** Stage 7 Implementation

For questions or clarifications, refer to the full delivery plan. For high-level overview, start with the executive summary.
