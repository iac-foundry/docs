# LADR-007: Jenkins Containerized, Agent Systemd

**Status:** Accepted
**Date:** 28 June 2026
**Owner:** Platform Engineering
**Supersedes:** N/A
**Superseded By:** N/A
**Related:** [../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md](../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md) · [LADR-009-non-root-containers.md](LADR-009-non-root-containers.md)

---

## 1. Context

- **Deployment scope:** Vernify Phase 4 deploys Jenkins (CI/CD orchestrator) and Phase 5 deploys Jenkins agent (build executor).
- **Deployment options:** Both components could run as containers, both as systemd services, or hybrid.
- **Infrastructure:** docker01 is a container-optimized host; agent01 is a lightweight LXC container.
- **Workload characteristics:**
  - Jenkins: stateful, complex, many plugins, needs persistent storage. Benefits from isolation and standard container logging.
  - Jenkins agent: lightweight, stateless, runs build jobs. Can run as systemd service with minimal overhead.
- **Operational model:** Single operator team; wants minimal services to manage.

## 2. Decision

**Jenkins runs as a Docker container on docker01. Jenkins agent runs as a systemd service on agent01.**

**Jenkins (docker01):**
- **Deployment:** Docker container, pulled from `jenkinsci/jenkins:lts-jdk21`
- **Execution:** Runs as non-root user (UID 1000)
- **Storage:** `/var/lib/docker/volumes/jenkins-home` persisted across restarts
- **Networking:** Port 8080 (HTTP), 50000 (JNLP agent connection)
- **Image building:** Multi-stage Dockerfile in `ansible-collection-jenkins` with Vault auth plugin + Job DSL provisioning

**Jenkins Agent (agent01):**
- **Deployment:** systemd service unit, installed from Ansible playbook
- **Execution:** Runs as non-root user (UID 1001)
- **Process:** Java subprocess managed by systemd (automatic restart on failure)
- **Networking:** Connects to Jenkins on docker01 via JNLP protocol (port 50000)
- **Storage:** No persistent state (stateless)

## 3. Rationale

### Why Jenkins as container:
- **Isolation:** Docker container isolates Jenkins from host OS; plugin updates don't affect docker01 runtime.
- **Logging:** Standard Docker logging captures all Jenkins logs; easily shipped to central logging (Phase 6).
- **Standard pattern:** Container deployment is industry standard for Jenkins; operators expect this model.
- **Stateful workload:** Jenkins has complex initialization (plugin installation, Job DSL seed); container builder automates this (Dockerfile).
- **Reproducibility:** Container image is version-controlled; easy to rebuild or rollback to prior version.

### Why agent as systemd:
- **Lightweight:** Jenkins agent is a thin client (JVM + JNLP plugin); doesn't need container isolation overhead.
- **Simple:** agent01 is already an LXC container (Proxmox managed); adding Docker container adds nested virtualization and complexity.
- **Resource efficiency:** systemd service uses less memory/CPU than container; important on agent01 (limited resources for build execution).
- **Operational simplicity:** Single systemd unit to debug; no Docker daemon complexity.
- **Fault tolerance:** systemd auto-restart handles agent crashes; Vault agent renewal also systemd-managed.

### Trade-offs accepted:
- **Different deployment models:** Jenkins and agent use different deployment patterns (containers vs. systemd).
  - **Rationale:** Each model is optimal for its workload. Operators must learn both patterns, but both are standard.
  - **Mitigation:** Detailed runbooks document both. Phase 6 could standardize on Kubernetes if desired.
- **Agent-Jenkins communication:** JNLP protocol requires agent01 → docker01 network connectivity (port 50000).
  - **Mitigation:** Network connectivity verified during Phase 5 provisioning; Vault agent provides secure communication.

## 4. Alternatives Considered

### Option A — Both Jenkins and agent as containers
| | |
|---|---|
| **Pros** | Consistent deployment model; easier to version-control and reproduce |
| **Cons** | Agent container on agent01 (LXC) adds nested Docker overhead; resource-constrained agent01 cannot afford Docker daemon; Dockerfile for stateless agent is overkill |
| **Reason rejected** | Resource overhead on agent01 not justified for lightweight workload. Hybrid model is simpler. |

### Option B — Both Jenkins and agent as systemd services
| | |
|---|---|
| **Pros** | Simpler operational model (one deployment pattern); no Docker daemon to manage |
| **Cons** | Jenkins on docker01 loses isolation benefits; plugin updates affect host; not standard pattern for Jenkins |
| **Reason rejected** | Jenkins benefits from container isolation too much to sacrifice. Standard pattern is containers for stateful services. |

### Option C — Kubernetes cluster (Jenkins + agents)
| | |
|---|---|
| **Pros** | Modern orchestration; auto-scaling agents; declarative configuration |
| **Cons** | Over-engineered for Phase 3-5 scale; requires etcd + control plane + 3+ nodes; operational complexity high |
| **Reason rejected** | Out of scope for Phase 3-5. Consider in Phase 6+ if workload demands elastic scaling. |

## 5. Consequences

### Positive:
- **Jenkins portability:** Container image can be deployed to different docker01 hosts without reconfiguration.
- **Agent simplicity:** systemd service is straightforward to debug and monitor; fewer moving parts.
- **Clear separation of concerns:** Jenkins orchestrates jobs; agent executes them. Each component optimized for its role.

### Negative:
- **Operational diversity:** Operators must know Docker AND systemd. Training burden higher than single-model approach.
  - **Mitigation:** Detailed runbooks for each component; CI playbooks automated; operators rarely interact with raw Docker/systemd commands.
- **JNLP protocol complexity:** Agent-Jenkins communication via JNLP adds debugging complexity (not standard like SSH).
  - **Mitigation:** Vault agent provides secure tunnel; JNLP logging documented; Phase 6 could switch to SSH if issues arise.
- **Container-systemd debugging:** Troubleshooting spans two execution models; harder for new operators.
  - **Mitigation:** Runbook includes "Agent won't connect to Jenkins" and "Jenkins won't start" decision trees.

## 6. Implementation Notes

- **Jenkins Docker:** See `ansible-collection-jenkins` role `deploy_jenkins_docker`. Dockerfile at `roles/deploy_jenkins_docker/files/Dockerfile`.
- **Jenkins agent systemd:** See `ansible-collection-jenkins` role `deploy_jenkins_agent`. systemd unit at `roles/deploy_jenkins_agent/files/jenkins-agent.service`.
- **Networking:** Firewall rules on docker01 and agent01 open ports 50000 (JNLP) and 8080 (HTTP). Validated during Phase 4 provisioning.
- **Storage:** Jenkins home persisted via Docker volume; survives container restart. Agent has no persistent state.
- **Validation:** CI test: deploy Jenkins container → deploy agent systemd → verify agent connects to Jenkins UI → trigger test job → verify job execution on agent.

## 7. References

- [../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md](../design/BLUEPRINTS_PHASE_3_5_ARCHITECTURE.md) — Section 5: Jenkins & CI Design
- [LADR-009-non-root-containers.md](LADR-009-non-root-containers.md) — Non-root container security posture
- Jenkins JNLP Agent Protocol: https://www.jenkins.io/doc/book/managing/jenkins-agent/
