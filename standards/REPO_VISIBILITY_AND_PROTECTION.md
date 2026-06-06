# Repository Visibility and Protection — iac-foundry

Summary
- Repositories: packer-template-proxmox, packer-template-vmware
- Visibility: private (owned by the `iac-foundry` GitHub org)
- Write access: restricted to `@Wernervdmerwe` (pushes to protected `main` branch are blocked for others)
- Protection: `main` branch protected with required PR reviews and CODEOWNERS approval
- Security policy file: intentionally omitted for now; to be added once ICS security reporting workflow is finalized
- Security: Dependabot vulnerability alerts and automated security fixes enabled; secret scanning attempted (may require org/enterprise level enablement)

Rationale
- These repositories contain platform templates and configuration intended for internal operational use by ICS customers and approved third parties only. Keeping them private avoids accidental public disclosure while still allowing controlled sharing.

What we configured
- `CODEOWNERS` points to `@Wernervdmerwe` so you will be the default approver for changes.
- Branch protection on `main` requires:
  - at least 1 approving review
  - dismissal of stale reviews
  - CODEOWNERS approval
  - enforcement for admins
  - restrictions so only `@Wernervdmerwe` can push directly
- Dependabot:
  - `.github/dependabot.yml` added to each repo (weekly updates for GitHub Actions and Terraform)
  - Vulnerability alerts and automated security fixes enabled via GitHub API
- Secret scanning:
  - Attempted to enable via API; if the org has GitHub Advanced Security or Enterprise settings, enable secret scanning at the org level for full coverage.

Operational notes
- To make a change:
  1. Work in a feature branch.
  2. Open a PR; request review from `@Wernervdmerwe` or appropriate team.
  3. After passing CI and receiving required approvals, merge into `main`.

- If you need to grant others push access, use org teams (recommended) rather than adding individual collaborators. Update the `restrictions` and `CODEOWNERS` accordingly.

Useful commands (what I ran):

```bash
# set remote to org and push local branch
git remote remove origin || true
git remote add origin https://github.com/iac-foundry/packer-template-proxmox.git
git push -u origin main

# apply branch protection (requires an auth token)
curl -H "Authorization: token $(gh auth token)" \
  -H "Accept: application/vnd.github+json" \
  -X PUT \
  https://api.github.com/repos/iac-foundry/packer-template-proxmox/branches/main/protection \
  -d '{"required_status_checks":{"strict":true,"contexts":[]},"enforce_admins":true,"required_pull_request_reviews":{"dismiss_stale_reviews":true,"require_code_owner_reviews":true,"required_approving_review_count":1},"restrictions":{"users":["Wernervdmerwe"],"teams":[]}}'
```

Next steps / recommendations
- Verify org-level secret scanning and SCA policies in the GitHub org security settings.
- Add `SECURITY.md` only after internal security response and contact process are finalized.
- Consider adding CODEOWNERS entries for teams if you want team-based approvals.
