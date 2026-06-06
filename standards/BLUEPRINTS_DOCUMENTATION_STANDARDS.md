# Blueprints Documentation Standards

**Status:** Active
**Date:** 06 June 2026
**Owner:** Platform Engineering

The `blueprints` knowledge base is **org-neutral and self-contained**. It deliberately does **not**
inherit any single organisation's documentation plumbing (e.g. Confluence parent pages or
organisation-specific "generated page" notices). Any org may mirror this content into its own portal
without edits.

---

## 1. Directory model

```
docs/
├── design/        # WHAT — architecture, principles, target state
├── standards/     # HOW — mandatory practices (this file lives here)
├── decisions/     # WHY — LADRs (lightweight architecture decision records)
├── templates/     # org-neutral templates for new design/standards/decisions
├── roadmap/       # future work, migration/retirement plans
└── legacy/        # superseded docs, preserved with a supersession note
```

---

## 2. File naming

Design and standards docs use `BLUEPRINTS_<CONTEXT>.md` (uppercase, underscore), e.g.
`BLUEPRINTS_ROLE_LAYOUT.md`, `BLUEPRINTS_GALAXY_PUBLISHING.md`.

Decisions use `LADR-NNN-kebab-title.md` with an iac-foundry-local sequence starting at `001`.

---

## 3. Document header (org-neutral)

Every doc starts with a minimal header — **no** organisation-specific frontmatter:

```markdown
# <Title>

**Status:** Active | Draft | Superseded
**Date:** DD Month YYYY
**Owner:** Platform Engineering
```

Do **not** add `<!-- Parent: ... -->` Confluence comments or any "generated from <org> repository"
notice in this knowledge base; those couple the docs to one organisation.

---

## 4. Normative language

Use RFC-2119 style: **MUST / MUST NOT / SHOULD / SHOULD NOT / MAY**. Standards are mandatory; design
docs explain intent; decisions record rationale and alternatives.

---

## 5. Per-collection and per-role docs

- Each collection ships a `README.md`: scope, the reuse contract, and an explicit *no hidden
  dependencies* statement.
- Each role ships a `README.md`: what it does, deployment model, required/optional inputs (pointing
  at `argument_specs.yml`), and **non-goals**.

---

## 6. Diagrams

Use Mermaid in fenced ```mermaid blocks so diagrams render in any Markdown portal without external
image assets.

---

## Related

- [../templates/decisions.md](../templates/decisions.md)
- [../templates/design.md](../templates/design.md)
- [../templates/standards.md](../templates/standards.md)
