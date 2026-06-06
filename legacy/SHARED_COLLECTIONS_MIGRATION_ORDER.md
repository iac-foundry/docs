# Shared Collections Migration Order (Safest Sequence)

Status: Draft
Date: 2026-06-01
Owner: ICS Platform Engineering

## Purpose

Define the safest migration order for moving IOTel and Saicom to ICS-curated shared `blueprint.*` collections.

- This is a sequencing document, not a standard definition document.
- The standard is defined in `SHARED_PLATFORM_COLLECTIONS_STRATEGY.md`.
- Planned repository inventory is defined in [SHARED_PLATFORM_REPOSITORIES.md](./SHARED_PLATFORM_REPOSITORIES.md).
- This plan is executed technology-by-technology, with both companies migrated per technology wave.

## Core Migration Model

ICS executes each technology wave in this order:

1. Build and validate in ICS (source-of-build and canary).
2. Migrate IOTel to the new shared collection release.
3. Migrate Saicom to the same release.
4. Freeze and retire legacy per-company implementation for that technology.

This keeps both companies close to latest curated code and avoids long divergence windows.

## Safety Gates For Every Wave

Do not promote a wave unless all gates pass:

1. Automated tests pass for the shared collection.
2. Security checks pass (secret handling, auth model, least privilege).
3. Rollback path is tested and documented.
4. Both companies complete smoke tests in lower environments.
5. Change-window approval is in place for production cutover.

## Safest Technology Order

## Wave 0: Foundation Guardrails

Goal: prevent drift during migration.

Includes:

- Pinned versions for shared collections.
- CI lint for prohibited patterns.
- Canonical variable and inventory contracts.

Reason for position:

- Reduces migration risk before high-impact technology moves.

## Wave 1: Vault (First Priority)

Goal: migrate secret-consumption and Vault lifecycle first.

Includes:

- `blueprint.vault` baseline implementation.
- Standardized consumer secret-read path per layer.
- Removal of ad-hoc secret read patterns.

Reason for first position:

- Vault is a current blocker for multiple IOTel workstreams.
- Secret handling inconsistency has the highest cross-platform risk.
- Clearing Vault first unblocks downstream Jenkins, Terraform, and app migrations.

Minimum exit criteria:

- No technology stack in either company uses prohibited Vault access patterns.
- Shared Vault consumption contract is proven in both IOTel and Saicom.
- Incident rollback drills pass for secret-path and auth changes.

## Wave 2: Jenkins And Pipeline Secret Interfaces

Goal: align CI orchestration to consume shared collections safely.

Includes:

- Shared Jenkins integration patterns.
- Standard pipeline-level secret injection for non-Ansible stages.
- Removal of repo-local one-off secret helper logic.

Reason for position:

- Depends on Vault model being stable.
- Controls blast radius for all downstream delivery workflows.

## Wave 3: Terraform Cloud And Terraform Agents

Goal: standardize provisioning runtime contracts after CI and Vault are stable.

Includes:

- `blueprint.terraform` and `blueprint.terraform_agents` adoption.
- Workspace/policy/agent contract normalization.
- Removal of company-local Terraform orchestration drift where shared behavior is expected.

Reason for position:

- Provisioning is high impact; safer after secret and CI controls are stable.

## Wave 4: Common Host Baseline

Goal: migrate reusable host-level capabilities.

Includes:

- `blueprint.common` baseline roles (linux baseline, logging client, monitoring client, cert/trust helpers).

Reason for position:

- Broad host footprint; safer after core platform control-plane tooling is normalized.

## Wave 5: Monitoring Stack Products

Goal: migrate observability services and agents.

Includes:

- `blueprint.graylog`
- `blueprint.grafana`
- `blueprint.graphite`

Reason for position:

- Monitoring migration needs stable secret and baseline host contracts.

## Wave 6: Access And Edge Integrations

Goal: migrate network-edge and remote-access platform products.

Includes:

- `blueprint.twingate`
- associated connector lifecycle and registration patterns

Reason for position:

- Integration-heavy; lower priority than core secrets, CI, and provisioning paths.

## Wave 7: Security Scanner Products

Goal: complete scanner and compliance product migrations.

Includes:

- `blueprint.openvas`
- `blueprint.openscap`

Reason for position:

- Important but not the primary blocker for immediate platform delivery.
- Better migrated after core dependencies are stable.

## Per-Wave Backlog Template

Use this template for each technology wave:

| ID | Item | Owner | Effort | Exit Criteria |
|---|---|---|---:|---|
| Wn-01 | Build or refactor shared collection implementation | ICS | TBD h | Shared collection feature complete and tested in ICS |
| Wn-02 | Migrate IOTel consumer repo(s) for this technology | ICS + IOTel | TBD h | IOTel lower-env and production smoke checks pass |
| Wn-03 | Migrate Saicom consumer repo(s) for this technology | ICS + Saicom | TBD h | Saicom lower-env and production smoke checks pass |
| Wn-04 | Remove legacy per-company implementation | ICS | TBD h | Legacy code paths retired with rollback notes |

## Current Recommendation

Immediate focus should be:

1. Wave 0 completion.
2. Wave 1 Vault migration start.
3. Strictly no parallel start of Wave 2 until Wave 1 exit criteria pass.

This is the safest path to unblock IOTel while keeping Saicom and IOTel aligned to ICS-curated latest code per technology.
