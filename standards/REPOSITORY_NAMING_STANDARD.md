# Repository Naming Standard

Status: Draft
Date: 2026-06-01
Owner: ICS Platform Engineering

## Purpose

Define one intuitive naming convention for all shared platform repositories created and curated by ICS.

This standard is designed to:

- Keep names predictable and searchable.
- Align Terraform module repositories with expected provider/module structure.
- Keep Ansible and Packer repository names equally clear.
- Reduce ambiguity during migrations across IOTel and Saicom.

## Scope

Applies to:

- Shared Ansible collection repositories.
- Shared Terraform module repositories.
- Shared Packer template repositories.
- Shared CI/CD, policy, catalog, and documentation repositories.
- Consumer platform repositories for each company.

## Naming Rules

1. Use lowercase letters, digits, and hyphens only.
2. Do not use underscores, spaces, or mixed case.
3. Use singular nouns unless plural is semantically required.
4. Keep names concise but specific.
5. Prefix by tool or platform role to make intent obvious.

## Canonical Patterns

## 1. Ansible Collections

Pattern:

- ansible-collection-<name>

Examples:

- ansible-collection-common
- ansible-collection-vault
- ansible-collection-graylog

Notes:

- Repository name is for source control.
- Collection namespace remains blueprint.<name> unless explicitly changed by architecture decision.

## 2. Terraform Modules

Pattern:

- terraform-<provider>-<module>

Examples:

- terraform-vsphere-vm
- terraform-vsphere-networking
- terraform-tfe-workspace

Notes:

- This pattern intentionally follows Terraform module naming expectations.
- Provider token should match the primary provider used by the module.

## 3. Packer Templates

Pattern:

- packer-template-<name>

Examples:

- packer-template-ubuntu-base
- packer-template-ubuntu-platform
- packer-template-image-manifest

## 4. Shared Platform Support Repositories

Pattern:

- platform-<capability>

Examples:

- platform-jenkins-shared-lib
- platform-ci-policy
- platform-release-catalog
- platform-standards
- platform-runbooks

## 5. Consumer Platform Repositories

Pattern:

- <company>-platform

Examples:

- iotel-platform
- saicom-platform

## Prohibited Patterns

1. Mixed naming styles in the same domain (example: blueprint-... and ansible-collection-... for equivalent artifact types).
2. Tool suffixes that hide artifact type (example: ansible-graylog-blueprints).
3. Ambiguous generic names with no domain qualifier (example: platform-common only).
4. Environment-specific names in shared source repos (example: ...-prod).

## Change Control

1. New repos must follow this standard by default.
2. Any exception requires documented rationale and approval from ICS Platform Engineering.
3. If a name is changed after creation, update all references in roadmap, CI, and consumer docs in the same change set.

## Quick Decision Guide

When creating a new repo:

1. Is it an Ansible collection? Use ansible-collection-<name>.
2. Is it a Terraform module? Use terraform-<provider>-<module>.
3. Is it a Packer template? Use packer-template-<name>.
4. Is it shared support tooling/docs? Use platform-<capability>.
5. Is it a company consumer repo? Use <company>-platform.
