# Blueprints Role Layout Standard

**Status:** Active
**Date:** 06 June 2026
**Owner:** Platform Engineering

Every role in every `blueprints.*` collection MUST follow this layout.

---

## 1. Directory layout

```
roles/<role>/
├── defaults/main.yml          # ALL tunables, namespaced; safe + generic defaults
├── vars/main.yml              # internal constants only (never org-specific, not meant for override)
├── meta/main.yml              # galaxy_info + dependencies: []
├── meta/argument_specs.yml    # typed interface — required vars fail fast, self-documenting
├── tasks/main.yml             # entrypoint; tagged; check-mode aware; no_log on secrets
├── handlers/main.yml          # service restarts etc.
├── templates/                 # *.j2
├── files/                     # static files (optional)
├── molecule/default/          # molecule.yml + converge.yml (+ verify.yml where useful)
└── README.md                  # contract: inputs, deployment model, NON-GOALS
```

`vars/`, `handlers/`, `templates/`, `files/` are present only when needed. `defaults/`, `meta/`,
`tasks/`, `molecule/`, `README.md`, and `meta/argument_specs.yml` are mandatory.

---

## 2. Role naming

Roles are named for **capability / deployment model**, not for the product (the collection already
names the product):

| Role name | Meaning |
|---|---|
| `container_server` | run the product as a container |
| `systemd_server` | run the product as a systemd-managed binary |
| `kubernetes_server` | deploy the product to Kubernetes |
| `agent` | a worker/agent that registers with a server |
| `client` | client-side configuration of the product |
| `install` | install a CLI/toolchain |
| `<capability>` | e.g. `inputs`, `pipelines`, `job_dsl`, `template_build` |

Deployment-model siblings (`container_server`, `systemd_server`, `kubernetes_server`) MUST share a
common variable interface (see Principle 6 and the variable standard).

---

## 3. meta/main.yml

```yaml
galaxy_info:
  role_name: container_server
  author: Platform Engineering
  description: Run <product> as a container.
  license: MIT
  min_ansible_version: "2.16"
  platforms:
    - name: Ubuntu
      versions: [jammy, noble]
    - name: Debian
      versions: [bookworm]
    - name: EL
      versions: ["9"]
dependencies: []        # MUST be empty — composition is a platform concern
```

---

## 4. meta/argument_specs.yml

Declares the typed interface. Required inputs (no safe default) MUST be `required: true`; this is how
"variable-driven, fail-fast" (Principle 2/8) is enforced and self-documented.

```yaml
argument_specs:
  main:
    short_description: Run <product> as a container.
    options:
      <component>_server_image:
        type: str
        required: true
        description: Container image reference (caller-supplied; no org default).
      <component>_server_http_port:
        type: int
        default: 8080
      <component>_server_tls_cert:
        type: str
        default: ""
        description: PEM cert; when set, TLS is enabled automatically.
```

---

## 5. tasks/main.yml conventions

- Begin with `assert` guards for required vars that aren't covered by argument_specs flow (belt and
  braces, with helpful `fail_msg`).
- Every task or block carries **tags** (at least the role name).
- Tasks that touch secrets set `no_log: true`.
- Support check mode: avoid `command`/`shell` for things modules can do; set `check_mode:` and
  `changed_when:` appropriately on unavoidable commands.
- A component role MUST NOT `include_role`/`import_role` another `blueprints` component, and MUST NOT
  query external systems (Principle 1/2).

---

## 6. README contract

Each role README MUST state:

1. **What it does** (one paragraph).
2. **Deployment model** (container / systemd / k8s / client / agent).
3. **Required inputs** and key optional inputs (point at `argument_specs.yml`).
4. **Non-goals** — explicitly, what this role does *not* do (e.g. "does not configure SSO; see
   `blueprints.<component>_integrations`").

---

## Related

- [BLUEPRINTS_VARIABLE_STANDARDS.md](BLUEPRINTS_VARIABLE_STANDARDS.md)
- [BLUEPRINTS_TESTING_STANDARDS.md](BLUEPRINTS_TESTING_STANDARDS.md)
- [../design/BLUEPRINTS_DESIGN_PRINCIPLES.md](../design/BLUEPRINTS_DESIGN_PRINCIPLES.md)
