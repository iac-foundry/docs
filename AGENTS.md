# New Agent Orientation — Where to Look for What

---

## Agent Working Protocol (read before anything else)

These rules apply to every AI agent working in this repository. They are not optional.

### 1. Conflict surfacing

If a user instruction contradicts anything in this AGENTS.md or any collection-level
AGENTS.md, **stop and surface the conflict before proceeding**.

> "This instruction conflicts with [rule/section] in AGENTS.md. Specifically: [what
> the doc says] vs [what you asked]. Please confirm how you'd like to proceed — and
> if the doc is wrong, I'll update it."

Do not silently override documented rules. Do not assume the user is right and the
doc is outdated. Surface it, resolve it, then act.

### 2. Living document — prompt to expand

If any instruction given, decision made, or clarification reached during a session
would make future interactions clearer or prevent repeated questions, **prompt the
user to add it to the relevant AGENTS.md**:

> "This decision [summary] isn't currently in AGENTS.md. Should I add it so future
> agents don't have to re-derive it?"

Use judgement — not every conversation needs a memory write. Prompt when the insight
is non-obvious, when it prevents a class of future mistakes, or when it resolves an
ambiguity that would recur.

### 3. Agent responsibility for maintenance

AGENTS.md is a living document. Agents are responsible for keeping it current:

- **When updating:** Add to the relevant section; don't append orphaned notes at the
  bottom. Keep entries concise — one clear rule per point.
- **When contradicted by new decisions:** Update the doc, don't leave stale rules.
- **When scope expands:** Add new sections as needed; structure beats completeness.
- **Do not add:** Ephemeral task state, PR-specific notes, or anything that belongs
  in a commit message instead.

---

**TL;DR:** Bookmark this page. It's your navigation guide to find the right docs fast based on what you're working on.

---

## Standing Decisions & Conventions

Decisions made during development that are non-obvious and must not be re-derived.
When a decision here is superseded, update the entry — don't leave stale rules.

### Namespace: `blueprints.*` plural is canonical

The correct namespace is **`blueprints`** (plural). The old singular `blueprint.*`
namespace is retired. Old `blueprint.*` docs live in `docs/legacy/` and are preserved
as historical record only — they must not be edited, referenced as current truth, or
used as the basis for new work.

- New collections: `blueprints.jenkins`, `blueprints.vault`, `blueprints.common`, etc.
- New repos: `ansible-collection-jenkins`, `ansible-collection-vault`, etc.
- Any instruction to use `blueprint.*` (singular) for new work should be surfaced as a
  conflict before proceeding.

### Consumer repos hold config only — logic lives in the collection

Consumer repos (org-specific platforms like `jenkins-agent`, `iotel-platform-jenkins`)
must contain **inventory, variables, environment config, and pipelines only**.
Service logic — how to install Java, create a system user, manage a systemd unit —
belongs in the collection role, not in the consumer.

If a consumer repo is doing work that the collection role should be doing, that is a
signal the collection role is incomplete. The fix is to implement the role, not to
expand the consumer.

### Reference consumers live in `iac-foundry/reference-consumers/`

Each collection that has a deployable role ships a reference consumer under
`reference-consumers/<capability>/`. A reference consumer:

- Is org-agnostic — no real IPs, hostnames, or org-specific paths
- Is runnable — not docs-only; actually exercises the collection roles
- Resolves secrets from environment variables (swappable for Vault, CI env, etc.)
- Serves as the proof-of-org-agnosticity: if it requires changes to the collection
  to run, the collection has hidden dependencies

When a new role is implemented, its reference consumer should be created alongside it.

---

## Quick Answers by Task

### "I'm implementing a new shared collection (blueprints.*)"

1. **Start with design principles** — [design/BLUEPRINTS_DESIGN_PRINCIPLES.md](design/BLUEPRINTS_DESIGN_PRINCIPLES.md)
   - 8 enforceable rules; these are non-negotiable
   - Use the conformance checklist in your PR

2. **Implement a role** — [standards/BLUEPRINTS_ROLE_LAYOUT.md](standards/BLUEPRINTS_ROLE_LAYOUT.md)
   - Directory structure, argument_specs.yml, molecule defaults, README contract

3. **Name your variables** — [standards/BLUEPRINTS_VARIABLE_STANDARDS.md](standards/BLUEPRINTS_VARIABLE_STANDARDS.md)
   - Namespace pattern, secret naming, generic safe defaults

4. **Handle secrets correctly** — [standards/BLUEPRINTS_SECRET_CONSUMPTION.md](standards/BLUEPRINTS_SECRET_CONSUMPTION.md)
   - Secrets come from caller (platform layer), never from role
   - Use `community.hashi_vault` at platform boundary, never inside role tasks

5. **Scaffold the collection** — [standards/BLUEPRINTS_COLLECTION_STRUCTURE.md](standards/BLUEPRINTS_COLLECTION_STRUCTURE.md)
   - galaxy.yml, README, meta/runtime.yml, changelogs, .ansible-lint, roles/

6. **Write tests** — [standards/BLUEPRINTS_TESTING_STANDARDS.md](standards/BLUEPRINTS_TESTING_STANDARDS.md)
   - molecule default scenario (converge + idempotence + no hidden deps)
   - CI guard script (enforces Principles 1 & 2)

7. **Release & publish** — [standards/BLUEPRINTS_RELEASE_STRATEGY.md](standards/BLUEPRINTS_RELEASE_STRATEGY.md)
   - Semantic versioning, changelog fragments, Galaxy tarball

**Quick summary:** Design principles → role layout → variable naming → secret handling → collection structure → tests → release.

---

### "I'm adding a new integration (blueprints.*_integrations)"

1. **Read the integration standard** — [standards/BLUEPRINTS_INTEGRATION_STANDARDS.md](standards/BLUEPRINTS_INTEGRATION_STANDARDS.md)
   - Integrations are separate collections, not embedded in core
   - One integration role = configures one side of one connection
   - Never deploy/upgrade the base product (that's not integration logic)

2. **Follow LADR-002** — [decisions/LADR-002-integrations-separated-from-core.md](decisions/LADR-002-integrations-separated-from-core.md)
   - Why integrations are separate; what goes where

3. **Example to copy from** — [examples/platform-ci/](examples/platform-ci/)
   - Shows how platform layer wires integrations: vault_auth, ldap_auth, authentik_sso

---

### "I need to understand the secret retrieval contract"

**Read in this order:**

1. **LADR-003** — [decisions/LADR-003-secret-retrieval-community-hashi-vault.md](decisions/LADR-003-secret-retrieval-community-hashi-vault.md)
   - The decision & rationale
   - Why collections ship no secret-retrieval code

2. **Secret consumption standard** — [standards/BLUEPRINTS_SECRET_CONSUMPTION.md](standards/BLUEPRINTS_SECRET_CONSUMPTION.md)
   - The rule: platform layer resolves secrets, passes as ordinary variables

3. **Example pattern** — [examples/platform-ci/](examples/platform-ci/)
   - Shows how to call `lookup('community.hashi_vault.vault_kv2_get', ...)` at platform layer

---

### "I'm composing a platform from collections"

1. **Framework architecture** — [design/BLUEPRINTS_FRAMEWORK_ARCHITECTURE.md](design/BLUEPRINTS_FRAMEWORK_ARCHITECTURE.md)
   - Three layers: component / integration / platform
   - Platform is where composition happens

2. **Example platforms** — [examples/](examples/)
   - [platform-ci](examples/platform-ci/) — Jenkins + Vault + Authentik
   - [platform-monitoring](examples/platform-monitoring/) — Graylog
   - [platform-identity](examples/platform-identity/) — Vault + OIDC

3. **Copy the pattern** — [examples/README.md](examples/README.md)
   - How to adapt examples for your org

---

### "I'm creating a new repository in iac-foundry"

1. **Repository naming** — [standards/REPOSITORY_NAMING_STANDARD.md](standards/REPOSITORY_NAMING_STANDARD.md)
   - `ansible-collection-<name>`, `terraform-<provider>-<module>`, `packer-template-<name>`

2. **Repo visibility & protection** — [standards/REPO_VISIBILITY_AND_PROTECTION.md](standards/REPO_VISIBILITY_AND_PROTECTION.md)
   - Branch protection baseline, CODEOWNERS, Dependabot, secret scanning

3. **Collection structure** — [standards/BLUEPRINTS_COLLECTION_STRUCTURE.md](standards/BLUEPRINTS_COLLECTION_STRUCTURE.md)
   - What files go in a new collection repo

---

### "I'm reviewing a PR on a blueprint collection"

Use the conformance checklist in [design/BLUEPRINTS_DESIGN_PRINCIPLES.md](design/BLUEPRINTS_DESIGN_PRINCIPLES.md) — it's designed for exactly this:

- [ ] `meta/main.yml` → `dependencies: []`
- [ ] No `include_role`/`import_role` of another `blueprints.*` component (§1)
- [ ] No Vault/LDAP/cloud/secret retrieval inside component `tasks/` (§2)
- [ ] Role configures only its own product (§3)
- [ ] Any cross-product wiring lives in `*_integrations` (§4)
- [ ] No org-specific values in `defaults/` (§7)
- [ ] Secret tasks set `no_log: true` (§7/§8)
- [ ] `meta/argument_specs.yml` present and complete (§8)
- [ ] molecule scenario present; idempotent; ansible-lint clean (§8)
- [ ] README states inputs + deployment model + non-goals (§8)

---

### "I want to understand the framework from first principles"

**Read in this order:**

1. [design/BLUEPRINTS_DESIGN_PRINCIPLES.md](design/BLUEPRINTS_DESIGN_PRINCIPLES.md) — the 8 rules
2. [design/BLUEPRINTS_FRAMEWORK_ARCHITECTURE.md](design/BLUEPRINTS_FRAMEWORK_ARCHITECTURE.md) — three-layer model
3. [decisions/LADR-001-blueprints-composable-collections.md](decisions/LADR-001-blueprints-composable-collections.md) — why this design
4. [decisions/LADR-002-integrations-separated-from-core.md](decisions/LADR-002-integrations-separated-from-core.md) — integration separation
5. [decisions/LADR-003-secret-retrieval-community-hashi-vault.md](decisions/LADR-003-secret-retrieval-community-hashi-vault.md) — secret retrieval
6. Any `standards/` doc relevant to your task

---

## Doc Directory Organization

### `/decisions/` — LADRs (Lightweight Architecture Decision Records)

Why we made certain choices. Reference these when proposing changes.

- **LADR-001** — composable collections framework (the foundation)
- **LADR-002** — integrations separated from core collections
- **LADR-003** — secret retrieval via `community.hashi_vault` at platform layer

### `/design/` — Design & principles

Conceptual foundations and the big picture.

- **BLUEPRINTS_DESIGN_PRINCIPLES.md** — the 8 enforceable rules (use in PRs)
- **BLUEPRINTS_FRAMEWORK_ARCHITECTURE.md** — three-layer model
- **BLUEPRINTS_EXAMPLE_COLLECTIONS.md** — gallery of what good collections look like
- **BLUEPRINTS_EXAMPLE_PLATFORMS.md** — gallery of what good platforms look like

### `/standards/` — Mandatory practices

How to implement things consistently. These are the working docs you reference while coding.

- **BLUEPRINTS_COLLECTION_STRUCTURE.md** — collection directory layout, galaxy.yml
- **BLUEPRINTS_ROLE_LAYOUT.md** — role directory layout, argument_specs.yml, molecule
- **BLUEPRINTS_VARIABLE_STANDARDS.md** — variable naming conventions
- **BLUEPRINTS_INTEGRATION_STANDARDS.md** — how to write integration collections
- **BLUEPRINTS_SECRET_CONSUMPTION.md** — secrets from caller, never from role
- **BLUEPRINTS_TESTING_STANDARDS.md** — ansible-lint, molecule, CI guards
- **BLUEPRINTS_DOCUMENTATION_STANDARDS.md** — how to write collection & role docs
- **BLUEPRINTS_RELEASE_STRATEGY.md** — versioning, changelog, tagging
- **BLUEPRINTS_GALAXY_PUBLISHING.md** — publishing to Galaxy/Automation Hub
- **REPOSITORY_NAMING_STANDARD.md** — repo naming conventions
- **REPO_VISIBILITY_AND_PROTECTION.md** — branch protection baseline, CODEOWNERS, Dependabot

### `/examples/` — Working reference implementations

Copy these patterns when building your own platforms or collections.

- **platform-ci** — Jenkins controller/agent + Vault auth + Authentik SSO
- **platform-monitoring** — Graylog log management
- **platform-identity** — Vault server + OIDC auth backend

### `/roadmap/` — Transition & deprecation

- **BLUEPRINTS_LEGACY_RETIREMENT.md** — how old `blueprint.*` (singular) is being retired in favor of `blueprints.*` (plural)

### `/legacy/` — Superseded documents

Historical record of old `blueprint.*` decisions and migration order. **Read for context only; do not use as current truth.**

- Preserved for archaeology only; see BLUEPRINTS_LEGACY_RETIREMENT.md for what changed

### `/templates/` — Document templates

Templates for writing new decisions, designs, and standards. Copy these if adding new docs.

---

## Common Scenarios & Paths

| Scenario | Read | Then | Then |
|---|---|---|---|
| **Implement new collection** | Principles (8 rules) | Role layout | Collection structure |
| **Implement new integration** | LADR-002 | Integration standard | Test + release |
| **Review a PR** | Principles checklist | (apply to PR) | — |
| **Compose a platform** | Framework architecture | Examples (platform-ci, etc.) | Copy pattern |
| **Create new repo** | Repo naming | Repo protection | Collection structure |
| **Understand secrets** | LADR-003 | Secret consumption | Example platform |
| **Learn from scratch** | Principles | LADR-001 | Examples |

---

## Red Flags & Guardrails

If you see these in a PR or collection, consult the design principles:

| Red Flag | See | Rule |
|---|---|---|
| Collection importing another `blueprints.*` | Principles | §1 — No hidden dependencies |
| Role calling `lookup('iotel.common.vault_kv', ...)` | Principles + LADR-003 | §2 — No secret retrieval in roles |
| Collection depends on another in `meta/dependencies` | Principles | §1 — Empty dependencies |
| Integration role deploys/upgrades the base product | Integration standard | Not an integration, wrong layer |
| Org-specific IPs, domains, vault paths in `defaults/` | Variable standards | §7 — No org-specific defaults |
| Secret variables not `no_log: true` | Testing standards | §8 — Secure by default |

---

## Quick Links

- **Anchor document:** [design/BLUEPRINTS_DESIGN_PRINCIPLES.md](design/BLUEPRINTS_DESIGN_PRINCIPLES.md)
- **Framework explained:** [design/BLUEPRINTS_FRAMEWORK_ARCHITECTURE.md](design/BLUEPRINTS_FRAMEWORK_ARCHITECTURE.md)
- **All LADRs:** [decisions/](decisions/)
- **All standards:** [standards/](standards/)
- **Working examples:** [examples/](examples/)
- **Legacy docs (reference only):** [legacy/](legacy/)

---

## Questions?

- **"Where do I find X?"** — Check the scenario table above, or search this doc for keywords.
- **"Can I do Y?"** — Check the Red Flags table, then the design principles.
- **"How should I structure Z?"** — Find the matching standard in `/standards/`.
- **"Is there an example?"** — Check `/examples/`.
