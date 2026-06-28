# LADR-009: Non-Root Containers (Security Hardening)

**Status:** Accepted
**Date:** 28 June 2026
**Owner:** Platform Engineering
**Supersedes:** N/A
**Superseded By:** N/A
**Related:** [../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md](../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md) · [LADR-007-jenkins-container-agent-systemd.md](LADR-007-jenkins-container-agent-systemd.md)

---

## 1. Context

- **Container security posture:** Industry standard practice is to run containers as non-root users to limit blast radius if container is compromised.
- **Vernify containers:** Phase 3-5 deploy step-ca, Vault, and Jenkins in containers. All are security-sensitive services.
- **Build process:** Ansible collection roles build container images via Dockerfile (multi-stage builds).
- **Challenge:** Some container operations traditionally require root (e.g., listening on privileged ports <1024; accessing host filesystem).
- **Organizational preference:** Vernify and Foundry favor defense-in-depth; non-root containers are a baseline security control.

## 2. Decision

**All containers in Vernify Phase 3-5 will run as non-root users. Container build process handles root operations (e.g., port binding via iptables) so runtime container does not need root privilege.**

**Application of non-root to each service:**
- **step-ca:** Runs as user `step-ca` (UID 999) in container. Dockerfile handles mTLS port binding (443) and permission setup.
- **Vault:** Runs as user `vault` (UID 999) in container. Filesystem permissions set during build; listener port 8200 mapped to non-privileged port.
- **Jenkins:** Runs as user `jenkins` (UID 1000) in container. Docker volume ownership handled during image build.

**Dockerfile pattern (multi-stage):**
```dockerfile
FROM alpine:latest AS builder
# Root operations: install, compile, bind ports, set permissions
RUN apk add --no-cache ...
RUN chmod 644 /etc/app.conf

FROM alpine:latest
# Copy artifacts from builder stage
COPY --from=builder --chown=app:app /app /app
USER app  # Switch to non-root
ENTRYPOINT ["/app/run.sh"]
```

## 3. Rationale

### Security benefits of non-root containers:
- **Blast radius reduction:** If container process is compromised (e.g., code injection), attacker cannot:
  - Access host root filesystem (only app-specific paths via volume mounts)
  - Modify host system files (can only write to container /tmp, /var/tmp)
  - Escape to host as root (kernel exploits still possible but less likely to escalate)
- **Defense in depth:** Non-root is one layer in a multi-layer security model (network isolation, SELinux, AppArmor).
- **Principle of least privilege:** Container only gets permissions it needs (e.g., read Vault config, write to TLS cert directory).
- **Compliance alignment:** Many security frameworks (CIS Docker Benchmark, Kubernetes PSP) mandate non-root containers.

### Build-time complexity vs. runtime safety:
- **Build complexity increases slightly:** Dockerfile must handle root operations (port binding, permission setup) before switching to non-root user.
- **Runtime safety increases significantly:** Container cannot escalate to root; reduces attack surface.
- **Trade-off accepted:** Small build burden for large security gain.

## 4. Alternatives Considered

### Option A — Run containers as root (UID 0)
| | |
|---|---|
| **Pros** | Simpler Dockerfile (no permission setup); fewer permission issues at runtime |
| **Cons** | Large blast radius if container compromised; attacker gains root on container host; violates org security preference |
| **Reason rejected** | Unacceptable security posture. Organizational policy mandates non-root containers. |

### Option B — Run containers with elevated privileges (Docker --privileged flag)
| | |
|---|---|
| **Pros** | Can access host devices, load kernel modules; simplifies some operations |
| **Cons** | Massive blast radius (full host access if container is compromised); defeats security purpose entirely |
| **Reason rejected** | Violates security principle. Reserved only for critical system services (e.g., kubelet, systemd); not needed for app containers. |

### Option C — Use rootless Docker
| | |
|---|---|
| **Pros** | Docker daemon itself runs as non-root; additional isolation layer |
| **Cons** | Requires systemd usernamespace setup; not all Docker features supported; adds complexity |
| **Reason rejected** | Non-root containers sufficient for Phase 3-5. Rootless Docker deferred to Phase 6+ if needed. |

## 5. Consequences

### Positive:
- **Reduced security risk:** Container compromise does not automatically grant host root access.
- **Portable images:** Non-root containers follow best practices; deployable to Kubernetes or other container orchestrators.
- **Audit compliance:** Non-root containers align with security frameworks (CIS, NIST); simplifies compliance audits.

### Negative:
- **Dockerfile complexity:** Build process must handle port binding, permission setup, user creation. Multi-stage builds required.
  - **Mitigation:** Template provided in `ansible-collection-*` roles; Dockerfile complexity is standard in industry.
- **Permission issues at runtime:** Volume mounts, socket access, or file permissions may require debugging.
  - **Example:** Jenkins container volume mounted from host must have correct ownership (jenkins:jenkins UID:GID).
  - **Mitigation:** Ansible playbooks handle permission setup; validation tests check ownership.
- **Capability restrictions:** Some operations may fail if container lacks required Linux capabilities.
  - **Example:** Container cannot load kernel modules, bind raw sockets, or change system time (acceptable for app containers).
  - **Mitigation:** Document required capabilities in Dockerfile comments; use `docker run --cap-add` only if necessary (flag in code review).

## 6. Implementation Notes

- **User creation:** Dockerfile creates non-root user with minimal groups (no sudo, no login shell).
  ```dockerfile
  RUN addgroup -S app && adduser -S app -G app
  ```
- **Volume mount ownership:** Ansible playbook sets correct UID:GID before starting container.
  ```ansible
  - name: Set Jenkins home ownership
    file:
      path: /var/lib/docker/volumes/jenkins-home/_data
      owner: 1000  # jenkins UID
      group: 1000
      recurse: yes
  ```
- **Capabilities:** Document required capabilities in Dockerfile (typically empty for app containers).
  ```dockerfile
  # No capabilities required; container runs read-only where possible
  ```
- **Validation:** CI test suite includes:
  - Container image scanned with Trivy (flags if running as root)
  - Unit test: verify container `USER` directive set to non-root
  - Integration test: start container, verify process UID is non-zero, verify no root processes

## 7. References

- [../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md](../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md) — Section 3-5: Service deployments
- [LADR-007-jenkins-container-agent-systemd.md](LADR-007-jenkins-container-agent-systemd.md) — Jenkins deployment model
- CIS Docker Benchmark: https://www.cisecurity.org/cis-benchmarks/
- Docker Security Best Practices: https://docs.docker.com/engine/security/
- Kubernetes Pod Security Policies: https://kubernetes.io/docs/concepts/security/pod-security-standards/
