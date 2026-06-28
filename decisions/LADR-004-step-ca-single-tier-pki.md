# LADR-004: Step-CA Single-Tier (Root-Only) Internal PKI

**Status:** Accepted
**Date:** 27 June 2026
**Owner:** Platform Engineering
**Supersedes:** N/A
**Superseded By:** N/A
**Related:** [../design/BLUEPRINTS_STEP_CA_PKI_DESIGN.md](../design/BLUEPRINTS_STEP_CA_PKI_DESIGN.md) · [LADR-003-secret-retrieval-community-hashi-vault.md](LADR-003-secret-retrieval-community-hashi-vault.md)

---

## 1. Context

- **Existing situation:** `iac-foundry` has no internal-PKI component collection. Phase 3 (`sec01`)
  of the Vernify homelab roadmap is blocked: `blueprints.vault.container_server` cannot be filled in
  because Vault's own TLS certificate must be issued by something, and nothing in the
  `blueprints.*` catalogue issues internal TLS certificates yet
  (`HOMELAB_GREENFIELD_BOOTSTRAP.md` Phase 3 lists this as a flat **GAP**).
- **Constraints:**
  - The framework serves multiple unrelated organisations (Design Principles, "the litmus test") —
    a new `blueprints.step_ca` collection must converge on a clean host with no other
    `blueprints.*` collection present and no hidden dependency on Vault, Jenkins, or any specific
    org's topology.
  - The org context for the only named concrete consumer (Vernify) is explicitly confirmed as a
    **small environment** where additional PKI hierarchy layering is considered unneeded overhead,
    not a missing capability.
  - No secret-retrieval code may ship in this collection (LADR-003 applies to step-ca with no
    exception — the provisioner password and any caller-supplied root key arrive as caller-supplied,
    `no_log` variables, never fetched at runtime).
  - The organisation's standing working agreement on blast radius requires any change with a
    global/host-wide effect to be either contained or escalated to a human risk call — this bears
    directly on how the CA's trust is allowed to interact with the host (see §5 and the companion
    design doc's Risk R-1).
- **Drivers:** Stage 1 requirements (FR-1) state the single-tier model as **confirmed and
  non-negotiable** — this LADR is not deciding *whether* to go single-tier, it is recording *why*,
  what is explicitly rejected as a result, and the key-protection posture that follows from that
  choice, so the decision and its trade-offs are auditable rather than assumed.
- **Risks of inaction:** Without this decision recorded, future engineers (or future orgs adopting
  this framework) may reasonably ask "why no intermediate CA?" and, absent a documented answer,
  either re-litigate the (already-closed) question or assume the omission was an oversight rather
  than a deliberate, scoped trade-off.

## 2. Decision

**We will implement `blueprints.step_ca` as a single-tier (root-only) internal PKI: the
`container_server` role initializes and runs exactly one root CA with no intermediate CA, and no
code path in this collection creates, configures, or chains to an intermediate. The root key may be
generated on the same host the CA runs on (standard Ansible-converge model), or the caller may
supply an existing root key/cert pair instead — both paths remain single-tier. Key protection is
achieved through host-level controls (dedicated service user, `0600` permissions on the key file, no
broad sudo) rather than through CA hierarchy isolation.**

## 3. Rationale

- **Benefits:**
  - Matches the confirmed org context: small environments where the operational overhead of
    minting, rotating, and protecting a separate intermediate CA exceeds the security benefit it
    would provide at this scale.
  - Drastically simpler implementation and operational model — one root, one provisioner config, one
    set of key material to protect, one thing to back up.
  - Faster to ship, faster to test (molecule converges a single CA, not a two-tier chain), faster to
    reason about in review — directly serves the goal of unblocking Phase 3 without over-building.
  - Keeps the collection's surface area small, consistent with this framework's stated philosophy of
    composable, demand-driven building blocks rather than building ahead of a named need.
- **Tradeoffs accepted:**
  - The root CA's private key is the single point of trust for the entire PKI, with **no
    isolation boundary** between "the thing that signs leaf certs day-to-day" and "the thing whose
    compromise is catastrophic" — an intermediate CA is the conventional mitigation for exactly this,
    and we are deliberately not building it.
  - Single point of failure for issuance: if the one CA instance is unavailable, no new or renewed
    certificates can be issued anywhere in that trust domain until it returns (already-issued leaf
    certs remain valid until they expire).
  - No path to revoke and re-issue under an intermediate while keeping the root offline/protected —
    any root key event (suspected compromise, planned rotation) means re-keying the actual trust
    anchor, which is a more disruptive operation than rotating an intermediate would be.
- **Security considerations:**
  - The root key lives on the host that runs `container_server`, protected by file permissions
    (`0600`), a dedicated service user/group, and no broad sudo — not by air-gapping or hierarchy
    isolation.
  - This is a deliberate, scoped security trade-off, not a global/host-wide blast-radius change: the
    blast radius of a compromised root key is bounded to *this CA's trust domain* (the certificates
    it has issued and will issue), not to every process or every host in the estate by default. It
    therefore does not trigger this organisation's mandatory human-risk-call gate for global changes
    — that gate is reserved for changes injected into every process/host by default (e.g. the
    `/etc/ld.so.preload` incident the working agreement is named after), which this is not.
  - The collection still does not retrieve secrets at runtime (LADR-003 applies unmodified): the
    provisioner password is caller-supplied and `no_log`; a caller-supplied root key/cert pair (the
    FR-1 alternative to generation) is likewise an input, never fetched from any store by the role.
- **Operational considerations:**
  - Backup/DR of the data directory containing the root key is an operational/platform-layer
    concern, not collection code — but because there is no intermediate, that one directory's
    durability is now the *entire* PKI's durability, raising the operational stakes of getting that
    backup story right (flagged in the companion design doc, §6 Deployment Model / NFR-Rel-2).
  - Re-keying (if ever needed) is a single, all-at-once event affecting every certificate the CA has
    issued — there is no graceful "rotate the intermediate, leaves transition transparently" path.

## 4. Alternatives Considered

### Option A — Multi-tier (root + intermediate) CA hierarchy
| | |
|---|---|
| Pros | Standard PKI best practice; root key can be kept offline/cold after minting the intermediate; intermediate compromise is recoverable without disturbing trust in the root; granular revocation/rotation boundary |
| Cons | Meaningfully more implementation and operational complexity (two CA configs, an intermediate-signing step, certificate chain handling in every consumer); more moving parts to test, back up, and reason about; protects against a threat model (root key isolation) the org has explicitly said is unneeded overhead at its current scale |
| Reason rejected | Stage 1 requirements confirm single-tier as non-negotiable for this org context; building multi-tier anyway would be scope creep against an explicit, validated business decision, not a technical necessity |

### Option B — Single-tier, but require an externally-generated root key with no on-host-generation path
| | |
|---|---|
| Pros | Forces every adopter toward an offline-ceremony-equivalent posture; removes the on-host-key-generation risk entirely |
| Cons | Contradicts Stage 1's confirmed assumption that no air-gapped/offline root-CA ceremony is required for v1; adds friction for the only named concrete consumer (Vernify/`sec01`), which has no existing offline-ceremony tooling or process; FR-1 already requires supporting a caller-supplied pair as an *alternative*, not as the only path |
| Reason rejected | Over-constrains v1 relative to the validated requirement; the caller-supplied-pair path already gives any org that wants this posture a way to get it without forcing it on everyone |

### Option C — Single-tier with a shipped opt-in trust-distribution role bundled into this same decision
| | |
|---|---|
| Pros | Would close the "how does anything actually trust this root" gap in the same release |
| Cons | A trust-store mutation capability is structurally a host-wide blast-radius operation (affects every process consulting the system trust store), which is the exact risk class this organisation's working agreement says must be contained or escalated to a human risk call before shipping — bundling it into this LADR would pre-empt that required human decision |
| Reason rejected | Kept as a separate, explicitly flagged risk (R-1 in the companion design doc) requiring its own human sign-off, rather than folded into the single-tier decision where it doesn't belong architecturally anyway (trust distribution is orthogonal to root-vs-intermediate hierarchy) |

## 5. Consequences

- **Positive:** Unblocks Phase 3 with the smallest viable PKI implementation; consistent with the
  org's stated risk appetite for its scale; fast to build, test, and review; leaves a documented,
  intentional seam (FR-1's caller-supplied-root path) for any future org that wants stronger
  isolation without forking the collection.
- **Negative / tradeoffs:** No isolation boundary around the root key; single point of failure for
  issuance; any root-key event is maximally disruptive (re-key everything, no intermediate buffer).
  These are accepted, not mitigated away — see Risks R-3 and R-4 in the companion design document.
  Separately, no opt-in role to *install* the root cert into a host's system-wide trust store ships
  in v1 either (Risk R-1, companion design doc §6 Decision 6/§7) — that gap has been reviewed and
  accepted by human sign-off. It is narrower than it sounds: it covers only trust-store
  *installation*, not trust-cert *retrieval* — retrieval is already solved natively by step-ca's own
  CA API (see Implementation Notes below) and required no design change.
- **Migration impact:** None at adoption time — this is a greenfield collection, so choosing
  single-tier today migrates nothing. The impact that matters is the **outgrow path**, and it is more
  disruptive than "stand up a new collection" makes it sound, so it is stated plainly here rather than
  understated: if a future org outgrows the single-tier model, migrating to multi-tier means standing
  up a *new* intermediate-capable collection (or a major version of this one) and then executing a
  **synchronized flag-day re-trust across every consumer that TOFU-pinned the old root's fingerprint**
  (the same `step ca bootstrap --ca-url <url> --fingerprint <fp>` flow this design relies on for
  trust bootstrap, §7 Decision 6/Risk R-1 resolution). Every consumer that pinned the old root's
  fingerprint must be updated to trust the new hierarchy's root/intermediate at effectively the same
  time — there is no cross-signing or root-overlap mechanism proposed anywhere in this design or
  LADR-004 that would let old- and new-root trust coexist during a gradual transition. This is the
  same all-at-once characteristic already named for root-key compromise (R-3) and data-dir loss (R-5)
  in the companion design doc — outgrowing single-tier is not a graceful, incremental migration, it is
  another flag-day re-trust event, just planned instead of forced by an incident. This LADR does not
  commit to ever building that path, and the lack of a cross-sign/overlap mechanism is correctly out
  of scope to build now — no named consumer needs it — but it must be described honestly here rather
  than implied to be simple. FR-1's escape hatch (bringing in an externally-managed root) remains
  available and does not change this conclusion, since it addresses *where the root comes from*, not
  *how every existing consumer learns to trust a new one*.

## 6. Implementation Notes

- **Rollout:** Implemented in Stage 7 as `ansible-collection-step-ca`
  (`blueprints.step_ca.container_server`); no intermediate-CA task, template, or variable exists
  anywhere in the role — the absence is structural (no dead code path to disable), not just a
  documented policy.
- **Validation:**
  - Molecule's `default` scenario converges a single root CA and issues a leaf cert that
    independently validates against that root (Stage 1 Success Metric 2) — there is no second tier
    to validate.
  - Root key file permissions (`0600`, dedicated service user/group ownership) are asserted/verified
    in the role's tasks and exercised by molecule.
  - The CI no-hidden-dependency guard (`ci/no_hidden_deps_guard.sh`) confirms no secret-retrieval
    code and no cross-collection dependency, consistent with LADR-003.
- **Root-cert retrieval for consumer bootstrap (Risk R-1 resolution):** The human reviewer accepted
  the companion design doc's Risk R-1 gap — no opt-in role ships in v1 to *install* the root cert
  into a host's system-wide trust store — but asked whether the root cert could still be made easy
  to *pull*, since it is public material, not a secret. It already can be, with zero new code in
  this collection: step-ca's CA API natively exposes the root certificate set at unauthenticated,
  well-known endpoints (`GET /roots`, `GET /roots.pem`, `GET /root/:fingerprint`) on the same HTTPS
  port `container_server` already runs for certificate issuance (FR-3). The standard
  `step ca bootstrap --ca-url <url> --fingerprint <fp>` client flow fetches the root over that port
  using fingerprint pinning (TOFU) rather than requiring the client to already trust the CA's
  serving certificate. Consumers must obtain the CA fingerprint out-of-band (e.g. a platform-layer
  output variable) to use this safely — the fingerprint, like the root cert itself, is not sensitive
  and needs no secret handling. This resolves "how does a consumer obtain the root" without touching
  the single-tier decision, the key-protection posture, or FR-5's "no silent host-wide trust
  *installation*" guarantee above: fetching a public cert over HTTPS is not a trust-store mutation,
  and this collection still installs nothing into any host's system trust store by default.
- **Review trigger:** Revisit this decision if:
  - Issuance volume, environment count, or availability requirements grow past what a single
    instance comfortably serves (see the companion design doc §6 "Scaling Strategy" and Appendix C);
  - An adopting org has a compliance or threat-model requirement that specifically mandates root/
    intermediate isolation (at which point Option A's analysis above should be re-run against that
    org's actual constraint, not assumed away);
  - A root-key compromise or near-miss occurs in production, which would be direct evidence that the
    accepted SPOF/no-isolation trade-off needs re-examination ahead of schedule.

## 7. References

- [../design/BLUEPRINTS_STEP_CA_PKI_DESIGN.md](../design/BLUEPRINTS_STEP_CA_PKI_DESIGN.md)
- [LADR-003-secret-retrieval-community-hashi-vault.md](LADR-003-secret-retrieval-community-hashi-vault.md)
- [../design/BLUEPRINTS_DESIGN_PRINCIPLES.md](../design/BLUEPRINTS_DESIGN_PRINCIPLES.md)
- [../standards/BLUEPRINTS_SECRET_CONSUMPTION.md](../standards/BLUEPRINTS_SECRET_CONSUMPTION.md)
- Smallstep `step-ca` documentation — https://smallstep.com/docs/step-ca/
- `vernify/roadmap/HOMELAB_GREENFIELD_BOOTSTRAP.md` — Phase 3 (`sec01`), the concrete consumer
  context that confirms the small-environment, single-tier-appropriate scale (reference only; this
  framework repo does not depend on that roadmap document)
