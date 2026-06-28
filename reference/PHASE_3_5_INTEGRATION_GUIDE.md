# Vault, step-ca, and Jenkins Integration Guide

**Audience:** Developers, platform engineers, external consumers
**Last Updated:** 28 June 2026
**Status:** Phase F Documentation (Stage 7)

---

## Overview

This guide provides integration documentation for applications and platforms consuming Vernify Phase 3-5 services:
- **Vault** for secret management
- **step-ca** for certificate issuance
- **Jenkins** for CI/CD orchestration

Each section covers authentication, example API calls, and integration patterns.

---

## Part 1: Vault Integration

### 1.1 Service Information

**Vault API Endpoint:** `https://vault.sec01.vernify.internal:8200`

**Version:** HashiCorp Vault v1.15+ (Community Edition)

**Authentication methods available:**
- AppRole (for applications, Jenkins, agents)
- Userpass (for operators, break-glass)
- Kubernetes auth (future; not in Phase 3-5)

**TLS certificate:** Signed by Vernify root CA (trust anchor in `/etc/ssl/certs/`)

### 1.2 Authentication

#### AppRole Authentication (Recommended for applications)

**Use case:** Applications, microservices, Jenkins agent jobs

**Prerequisites:**
- AppRole role created in Vault (e.g., `role/app-name`)
- `role_id` and `secret_id` generated
- `secret_id` stored securely (rotated regularly)

**Authentication flow:**
```bash
# Retrieve role_id (stable identifier)
ROLE_ID="$(vault read -field=role_id auth/approle/role/app-name)"

# Retrieve secret_id (should be rotated periodically)
SECRET_ID="$(vault write -field=secret_id -f auth/approle/role/app-name/secret-id)"

# Authenticate and get Vault token
curl -s https://vault.sec01.vernify.internal:8200/v1/auth/approle/login \
  -d "{\"role_id\":\"${ROLE_ID}\",\"secret_id\":\"${SECRET_ID}\"}" | jq '.auth.client_token'
```

**Example: Application authentication script**
```bash
#!/bin/bash
# Authenticate to Vault using AppRole

VAULT_ADDR="https://vault.sec01.vernify.internal:8200"
ROLE_ID="app-example"
SECRET_ID="$(cat /etc/app/vault-secret-id)"

# Login to Vault
VAULT_TOKEN=$(curl -s ${VAULT_ADDR}/v1/auth/approle/login \
  -d "{\"role_id\":\"${ROLE_ID}\",\"secret_id\":\"${SECRET_ID}\"}" \
  | jq -r '.auth.client_token')

export VAULT_TOKEN
export VAULT_ADDR
echo "Authenticated to Vault; token expires in $(vault token lookup -field=ttl)"
```

#### Userpass Authentication (For operators, break-glass only)

**Use case:** Operator break-glass access, manual secret retrieval

**Prerequisites:**
- Operator username and password configured in Vault
- MFA enabled (recommended)

**Authentication flow:**
```bash
# Interactive login
vault login -method=userpass username=operator
# Prompts for password

# OR non-interactive (not recommended for production)
vault login -method=userpass username=operator password="${OPERATOR_PASSWORD}"
```

### 1.3 Secret Paths and Structure

**Canonical secret path model:**

```
kv/platform/              # Vernify infrastructure secrets
├── vault/                # Vault bootstrap secrets
│   ├── unseal-keys       # 3 Shamir unseal keys (JSON array)
│   └── root-token        # Vault root token (break-glass only)
├── pki/                  # PKI and certificate secrets
│   ├── root-key          # step-ca root CA private key
│   ├── root-cert         # step-ca root CA certificate
│   └── provisioner-creds # step-ca provisioner password
├── jenkins/              # Jenkins bootstrap secrets
│   ├── admin-token       # Jenkins admin API token
│   ├── admin-password    # Jenkins admin password (if needed)
│   └── approle-secret-id # Jenkins AppRole secret_id
└── apps/                 # Application-specific secrets (for future apps)
    └── {app-name}/       # Organized by application
        ├── db-password
        ├── api-key
        └── certificates

kv/applications/         # Third-party application secrets (Phase 6+)
├── iotel/
└── saicom/
```

**Example secret structure:**
```bash
# Retrieve all platform secrets
vault kv list kv/platform
# Output:
# Keys
# ----
# jenkins/
# pki/
# vault/

# Retrieve Jenkins admin token
vault kv get -field=admin-token kv/platform/jenkins
# Output: ghp_1234567890abcdefghijklmnopqrstuvwxyz

# Retrieve Vault unseal keys
vault kv get kv/platform/vault/unseal-keys
# Output:
# ===== Data =====
# Key      Value
# ---      -----
# keys     [key1, key2, key3]
```

### 1.4 Reading Secrets (API)

#### GET: Retrieve secret
```bash
# Retrieve secret using Vault token
curl -s -H "X-Vault-Token: ${VAULT_TOKEN}" \
  https://vault.sec01.vernify.internal:8200/v1/kv/data/platform/jenkins

# Response:
# {
#   "auth": {...},
#   "data": {
#     "data": {
#       "admin-token": "ghp_...",
#       "admin-password": "..."
#     },
#     "metadata": {...}
#   }
# }
```

#### Example: Python client
```python
import hvac
import json

# Create Vault client
client = hvac.Client(url='https://vault.sec01.vernify.internal:8200')

# Authenticate using AppRole
response = client.auth.approle.login(
    role_id='app-example',
    secret_id='...'
)
client.token = response['auth']['client_token']

# Retrieve secret
secret = client.secrets.kv.v2.read_secret_version(
    path='platform/jenkins'
)

admin_token = secret['data']['data']['admin-token']
print(f"Jenkins admin token: {admin_token}")
```

#### Example: Bash script with curl
```bash
#!/bin/bash
# Retrieve secret from Vault

VAULT_ADDR="https://vault.sec01.vernify.internal:8200"
SECRET_PATH="kv/data/platform/jenkins"

# Authenticate (assume AppRole auth done first)
VAULT_TOKEN="${VAULT_TOKEN:-}"

# Retrieve secret
curl -s -H "X-Vault-Token: ${VAULT_TOKEN}" \
  ${VAULT_ADDR}/v1/${SECRET_PATH} | \
  jq '.data.data.admin-token'
```

### 1.5 Vault Agent Integration (For agent01)

**Service:** `vault-agent` systemd service on agent01

**Configuration:** `/etc/vault/agent.hcl`

**Secrets delivery:** Rendered to `/var/run/secrets/` directory

**Usage in application:**
```bash
# Read secret delivered by Vault agent
cat /var/run/secrets/jenkins-token

# Secret is automatically renewed by vault-agent service
# No explicit token refresh needed by application

# Read metadata (for debugging)
ls -la /var/run/secrets/
# Output:
# jenkins-token
# db-password
# api-key
# .updated  # timestamp of last renewal
```

**Renewal**: Vault agent automatically renews secrets before expiry (configurable renewal window)

---

## Part 2: step-ca Certificate Issuance

### 2.1 Service Information

**step-ca HTTP API endpoint:** `https://step-ca.sec01.vernify.internal/ca`

**step-ca provisioner name:** `blueprints`

**Provisioner type:** JWK (JSON Web Key)

**Certificate validity defaults:** 1 year (for leaf certs); 10 years (root CA)

**Supported certificate types:** X.509 v3 (mTLS, TLS)

**TLS verification:** Signed by Vernify root CA

### 2.2 Provisioner Authentication

**step-ca uses OIDC-like token exchange:** Provisioner presents proof-of-possession via provisioner password

**Provisioner credential location:** Stored in Vault KV at `kv/platform/pki/provisioner-creds`

**Provisioner password:** Known to step-ca container and stored in Vault

**Example retrieval:**
```bash
# Retrieve provisioner password from Vault (operator only)
vault kv get -field=provisioner_password kv/platform/pki/provisioner-creds
# Output: <base64-encoded password>

# Store in secure location for certificate operations
export STEP_CA_PROVISIONER_PASSWORD="<password>"
```

### 2.3 Certificate Issuance API

#### Using `step` CLI tool

**Prerequisite:** Install `step` CLI on operator's workstation

```bash
# Install step CLI (macOS)
brew install smallstep/step/step

# Or download from https://smallstep.com/cli/
```

**Issue a certificate:**
```bash
# Syntax: step certificate create <subject> <output-crt> <output-key> [options]

step certificate create \
  --provisioner blueprints \
  --provisioner-password-file=<(echo -n "${STEP_CA_PROVISIONER_PASSWORD}") \
  --ca-url https://step-ca.sec01.vernify.internal \
  --ca-cert <(curl -s https://step-ca.sec01.vernify.internal/ca.crt) \
  "example.vernify.internal" \
  example.crt \
  example.key

# Output: Certificates written to example.crt and example.key
```

**Issue a certificate with SAN (Subject Alternative Name):**
```bash
step certificate create \
  --provisioner blueprints \
  --provisioner-password-file=<(echo -n "${STEP_CA_PROVISIONER_PASSWORD}") \
  --ca-url https://step-ca.sec01.vernify.internal \
  --ca-cert <(curl -s https://step-ca.sec01.vernify.internal/ca.crt) \
  --san example1.vernify.internal \
  --san example2.vernify.internal \
  "example.vernify.internal" \
  example.crt \
  example.key
```

#### Using cURL API

**Endpoint:** `https://step-ca.sec01.vernify.internal/ca/sign`

**Request body:** CSR (Certificate Signing Request) in base64 format

**Example: Generate CSR and request certificate**
```bash
#!/bin/bash
# Generate private key and CSR
openssl req -new -newkey rsa:2048 -keyout example.key \
  -out example.csr -nodes \
  -subj "/CN=example.vernify.internal"

# Encode CSR
CSR=$(base64 -w0 < example.csr)

# Request certificate from step-ca
curl -s -X POST https://step-ca.sec01.vernify.internal/ca/sign \
  -H "Content-Type: application/json" \
  -d "{
    \"csr\": \"$(cat example.csr | base64 -w0)\",
    \"provisioner\": \"blueprints\",
    \"provisioner_password\": \"${STEP_CA_PROVISIONER_PASSWORD}\"
  }" | jq '.crt' -r | base64 -d > example.crt

# Verify certificate
openssl x509 -in example.crt -noout -text
```

### 2.4 Certificate Verification

**Verify certificate chain:**
```bash
# Download root certificate
curl -s https://step-ca.sec01.vernify.internal/ca.crt > root.pem

# Verify certificate is signed by root
openssl verify -CAfile root.pem example.crt
# Output: example.crt: OK

# Inspect certificate
openssl x509 -in example.crt -noout -subject -dates -issuer
# Output:
# subject=CN = example.vernify.internal
# issuer=CN = Vernify Root CA
# notBefore=Jun 28 12:00:00 2026 GMT
# notAfter=Jun 28 12:00:00 2027 GMT
```

### 2.5 Certificate Renewal

Certificates can be renewed before expiry using Vault's Certificate Auth method:

```bash
# Prerequisites: Vault Cert Auth method enabled (configured during Phase 3)
# Issuing certificate signed by step-ca root

# Use Vault Cert Auth to renew
vault write -format=json auth/cert/login \
  name=example \
  certificate=@example.crt \
  key=@example.key | jq '.auth.client_token'

# Renewed certificate has fresh TTL (1 year)
```

---

## Part 3: Jenkins Integration

### 3.1 Service Information

**Jenkins URL:** `https://jenkins.docker01.vernify.internal:8443`

**Jenkins port:** 8443 (HTTPS), 8080 (HTTP, internal only)

**JNLP agent port:** 50000 (for agent01 connections)

**Admin console:** `https://jenkins.docker01.vernify.internal:8443/`

**API endpoint:** `https://jenkins.docker01.vernify.internal:8443/api/`

**Version:** Jenkins LTS v2.426+ (with plugins)

**Pre-installed plugins:**
- HashiCorp Vault (for secret integration)
- Job DSL (for programmatic job definition)
- Pipeline (for declarative/scripted pipelines)
- Git (for SCM integration)

### 3.2 Authentication

#### API Token Authentication

**Jenkins admin token location:** Stored in Vault KV at `kv/platform/jenkins/admin-token`

**Retrieve admin token:**
```bash
vault kv get -field=admin-token kv/platform/jenkins
# Output: ghp_1234567890abcdefghijklmnopqrstuvwxyz
```

**Use token in API calls:**
```bash
# Basic auth: username:token
curl -s -k --user admin:${JENKINS_ADMIN_TOKEN} \
  https://jenkins.docker01.vernify.internal:8443/api/json
```

#### Vault-based Credential Provider

**Use case:** Jobs retrieve credentials from Vault at runtime

**Configuration:** Pre-configured in Jenkins; jobs use Vault plugin

**Example Jenkinsfile:**
```groovy
pipeline {
  agent any

  stages {
    stage('Retrieve Secrets') {
      steps {
        withVault([
          vaultSecrets: [
            [path: 'kv/platform/jenkins', keys: ['admin-token']]
          ]
        ]) {
          sh '''
            echo "Jenkins token: ${admin-token}"
            # Use token in subsequent steps
          '''
        }
      }
    }
  }
}
```

### 3.3 Job and Pipeline Management

#### List Jobs (API)

```bash
# Retrieve all jobs
curl -s -k --user admin:${JENKINS_ADMIN_TOKEN} \
  https://jenkins.docker01.vernify.internal:8443/api/json | jq '.jobs'

# Output:
# [
#   {
#     "name": "dryrun-test",
#     "url": "https://jenkins.docker01.vernify.internal:8443/job/dryrun-test/",
#     ...
#   },
#   ...
# ]
```

#### Trigger Job

```bash
# Trigger a job build
curl -s -k --user admin:${JENKINS_ADMIN_TOKEN} \
  -X POST https://jenkins.docker01.vernify.internal:8443/job/dryrun-test/build

# Monitor build queue
curl -s -k --user admin:${JENKINS_ADMIN_TOKEN} \
  https://jenkins.docker01.vernify.internal:8443/api/json | jq '.queue'
```

#### Job DSL: Programmatic Job Definition

**Job DSL files location:** `/vernify/bootstrap-container/terraform-jenkins/jobs/*.groovy`

**Example Job DSL file:**
```groovy
// dryrun-test.groovy
pipelineJob('dryrun-test') {
  description('Dry-run test for infrastructure changes')

  triggers {
    // Trigger on manual invocation
    githubPush()
  }

  definition {
    cps {
      script('''
@Library('vernify-shared-lib') _

pipeline {
  agent any

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Dry-run') {
      steps {
        sh '''
          terraform init
          terraform plan -out=tfplan
        '''
      }
    }

    stage('Scan') {
      steps {
        sh 'tfsec . --format json > scan-results.json'
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'tfplan, scan-results.json'
    }
  }
}
      '''.stripIndent())
      sandbox(true)
    }
  }
}
```

**Deploy Job DSL:**
```bash
# Job DSL is seeded during Phase 4 deployment
# To manually re-seed:
curl -s -k --user admin:${JENKINS_ADMIN_TOKEN} \
  -X POST https://jenkins.docker01.vernify.internal:8443/job/seed-job-dsl/build
```

### 3.4 Agent Integration

**Agent nodes:** agent01 (LXC container) registered as Jenkins agent

**Agent status:** Visible in Jenkins UI at `/computer/`

```bash
# Retrieve agent status
curl -s -k --user admin:${JENKINS_ADMIN_TOKEN} \
  https://jenkins.docker01.vernify.internal:8443/computer/agent01/api/json | \
  jq '{displayName, offline, executors}'

# Output:
# {
#   "displayName": "agent01",
#   "offline": false,
#   "executors": 2
# }
```

**Restrict jobs to specific agent:**
```groovy
// In Jenkinsfile or Job DSL
pipeline {
  agent {
    node {
      label 'agent01'  // Run on agent01 only
    }
  }

  stages {
    stage('Build') {
      steps {
        sh 'packer build template.hcl'
      }
    }
  }
}
```

### 3.5 Example Integrations

#### Pipeline: Build and Deploy

**Example: CI/CD pipeline integrating step-ca, Vault, and Jenkins**

```groovy
// Jenkinsfile
@Library('vernify-shared-lib') _

pipeline {
  agent any

  options {
    buildDiscarder(logRotator(numToKeepStr: '10'))
    timeout(time: 1, unit: 'HOURS')
  }

  parameters {
    string(name: 'REPO', defaultValue: 'vernify-core', description: 'Repository to build')
    choice(name: 'ENV', choices: ['dev', 'staging'], description: 'Target environment')
  }

  environment {
    VAULT_ADDR = 'https://vault.sec01.vernify.internal:8200'
    STEP_CA_ENDPOINT = 'https://step-ca.sec01.vernify.internal'
  }

  stages {
    stage('Authenticate') {
      steps {
        withVault([
          vaultSecrets: [
            [path: 'kv/platform/vault', keys: ['root-token']],
            [path: 'kv/platform/pki', keys: ['provisioner-creds']],
            [path: 'kv/applications/${params.ENV}', keys: ['*']]
          ]
        ]) {
          sh '''
            echo "✓ Authenticated to Vault"
            export VAULT_TOKEN="${root-token}"
            vault kv list "kv/applications/${ENV}"
          '''
        }
      }
    }

    stage('Checkout') {
      steps {
        checkout([$class: 'GitSCM', branches: [[name: '*/main']], userRemoteConfigs: [[url: "https://github.com/vernify/${REPO}.git"]]])
      }
    }

    stage('Build') {
      steps {
        sh '''
          set -e
          echo "[Build] Building ${REPO} for ${ENV}..."
          
          # Example: Use Packer to build VM image
          packer build -var "environment=${ENV}" template.hcl
        '''
      }
    }

    stage('Generate Certificate') {
      steps {
        withVault([vaultSecrets: [[path: 'kv/platform/pki', keys: ['provisioner-creds']]]]) {
          sh '''
            # Issue certificate for built artifact
            step certificate create \
              --provisioner blueprints \
              --provisioner-password-file=<(echo -n "${provisioner-creds}") \
              --ca-url ${STEP_CA_ENDPOINT} \
              "artifact-${BUILD_NUMBER}.vernify.internal" \
              artifact.crt artifact.key

            echo "✓ Certificate issued"
          '''
        }
      }
    }

    stage('Deploy') {
      when { expression { params.ENV == 'staging' } }
      steps {
        sh '''
          echo "[Deploy] Deploying to ${ENV}..."
          terraform apply -auto-approve -var "environment=${ENV}"
        '''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'artifact.crt, artifact.key, build-output.log', allowEmptyArchive: true
      cleanWs()
    }
    success {
      echo "✓ Pipeline succeeded"
    }
    failure {
      echo "✗ Pipeline failed; check logs for details"
    }
  }
}
```

---

## Part 4: Common Integration Patterns

### 4.1 Application: Read secrets from Vault

```bash
#!/bin/bash
# Generic application secret retrieval

VAULT_ADDR="https://vault.sec01.vernify.internal:8200"
APP_ROLE="app-example"
SECRET_PATH="kv/platform/apps/example"

# Authenticate using AppRole
AUTH_RESPONSE=$(curl -s -X POST ${VAULT_ADDR}/v1/auth/approle/login \
  -d "{\"role_id\":\"${APP_ROLE}\",\"secret_id\":\"$(cat /etc/app/secret-id)\"}")

VAULT_TOKEN=$(echo "${AUTH_RESPONSE}" | jq -r '.auth.client_token')

# Retrieve secret
curl -s -H "X-Vault-Token: ${VAULT_TOKEN}" \
  ${VAULT_ADDR}/v1/${SECRET_PATH} | jq '.data.data'

# Use secrets in application
export DB_PASSWORD=$(...)
export API_KEY=$(...)
/usr/local/bin/start-app.sh
```

### 4.2 Kubernetes Pod: Retrieve certificates from step-ca

```yaml
# Example: Kubernetes Job using step-ca for certificate provisioning
apiVersion: batch/v1
kind: Job
metadata:
  name: cert-provisioner
spec:
  template:
    spec:
      serviceAccountName: cert-provisioner
      containers:
      - name: provisioner
        image: smallstep/step-cli:latest
        env:
        - name: VAULT_ADDR
          value: "https://vault.sec01.vernify.internal:8200"
        - name: STEP_CA_ENDPOINT
          value: "https://step-ca.sec01.vernify.internal"
        command:
        - sh
        - -c
        - |
          step certificate create \
            --provisioner blueprints \
            --provisioner-password-file=/etc/pki/provisioner-password \
            --ca-url ${STEP_CA_ENDPOINT} \
            "pod-${POD_NAME}.default.svc.cluster.local" \
            /etc/pki/cert.crt \
            /etc/pki/cert.key
        volumeMounts:
        - name: provisioner-password
          mountPath: /etc/pki
      volumes:
      - name: provisioner-password
        secret:
          secretName: step-ca-provisioner
      restartPolicy: Never
```

### 4.3 Terraform: Request certificate during infrastructure provisioning

```hcl
# Example: Terraform requesting certificate from step-ca

resource "null_resource" "request_certificate" {
  provisioner "local-exec" {
    command = <<-EOT
      step certificate create \
        --provisioner blueprints \
        --provisioner-password-file=<(echo -n "${var.step_ca_provisioner_password}") \
        --ca-url https://step-ca.sec01.vernify.internal \
        "${local.fqdn}" \
        ${local_file.cert.filename} \
        ${local_file.key.filename}
    EOT
    interpreter = ["bash", "-c"]
  }

  depends_on = [
    local_file.provisioner_password
  ]
}

output "certificate_path" {
  value = local_file.cert.filename
}
```

---

## Part 5: Troubleshooting

### 5.1 Cannot connect to Vault

**Symptom:** `curl: (60) SSL certificate problem`

**Diagnosis:**
```bash
# Check Vault is running
curl -k https://vault.sec01.vernify.internal:8200/v1/sys/health
# If connection refused: Vault not running

# Check certificate
curl -k https://vault.sec01.vernify.internal:8200/v1/sys/health \
  -v 2>&1 | grep "subject="
```

**Resolution:**
```bash
# Option 1: Trust Vault's self-signed cert (development only)
export VAULT_SKIP_VERIFY=true

# Option 2: Download and trust root certificate (production)
curl -s https://step-ca.sec01.vernify.internal/ca.crt | \
  sudo tee /usr/local/share/ca-certificates/vernify-root.crt
sudo update-ca-certificates
```

### 5.2 AppRole authentication fails

**Symptom:** `permission denied` or `invalid role_id/secret_id`

**Diagnosis:**
```bash
# Verify role exists
vault read auth/approle/role/app-example

# Verify secret_id is valid
vault write -f auth/approle/role/app-example/secret-id
# Should return new secret_id
```

**Resolution:**
```bash
# Rotate secret_id if expired
vault write -f auth/approle/role/app-example/secret-id

# Or recreate role if corrupted
vault delete auth/approle/role/app-example
# Re-run bootstrap script to recreate
```

### 5.3 Cannot issue certificate from step-ca

**Symptom:** `step certificate create` returns error

**Diagnosis:**
```bash
# Verify step-ca health
curl -s https://step-ca.sec01.vernify.internal/health | jq '.status'

# Verify provisioner password
echo "${STEP_CA_PROVISIONER_PASSWORD}" | wc -c
```

**Resolution:**
```bash
# Restart step-ca container
docker restart step-ca
sleep 10

# Re-issue certificate
step certificate create ...
```

### 5.4 Jenkins API authentication fails

**Symptom:** `401 Unauthorized` or `403 Forbidden`

**Diagnosis:**
```bash
# Verify Jenkins is running
curl -s -k https://jenkins.docker01.vernify.internal:8443/login | grep -q "Jenkins" && echo "✓ Jenkins up"

# Verify token
echo "${JENKINS_ADMIN_TOKEN}" | wc -c
# Should be >20 characters
```

**Resolution:**
```bash
# Retrieve fresh token from Vault
vault kv get -field=admin-token kv/platform/jenkins

# Use correct API endpoint format
curl -s -k --user admin:${NEW_TOKEN} \
  https://jenkins.docker01.vernify.internal:8443/api/json | jq '.version'
```

---

## Appendix: Environment Variables

Standard environment variables for integration with Vernify Phase 3-5:

```bash
# Vault
export VAULT_ADDR="https://vault.sec01.vernify.internal:8200"
export VAULT_TOKEN="${VAULT_TOKEN}"  # Set after authentication
export VAULT_SKIP_VERIFY="false"     # Trust root CA (set to true for dev)

# step-ca
export STEP_CA_ENDPOINT="https://step-ca.sec01.vernify.internal"
export STEP_CA_PROVISIONER="blueprints"
export STEP_CA_PROVISIONER_PASSWORD="${STEP_CA_PROVISIONER_PASSWORD}"

# Jenkins
export JENKINS_URL="https://jenkins.docker01.vernify.internal:8443"
export JENKINS_ADMIN_TOKEN="${JENKINS_ADMIN_TOKEN}"

# Secret paths (standard)
export SECRET_PATH_VAULT="kv/platform/vault"
export SECRET_PATH_JENKINS="kv/platform/jenkins"
export SECRET_PATH_PKI="kv/platform/pki"
export SECRET_PATH_APPS="kv/applications"
```

## Appendix: Useful Commands

```bash
# Vault: List all secrets
vault kv list kv/platform

# Vault: Get secret value
vault kv get -field=admin-token kv/platform/jenkins

# step-ca: Verify certificate
openssl x509 -in cert.crt -noout -text

# Jenkins: Get job status
curl -s -k --user admin:${JENKINS_ADMIN_TOKEN} \
  https://jenkins.docker01.vernify.internal:8443/job/dryrun-test/api/json | jq '.lastBuild.result'

# Jenkins: Trigger job
curl -s -k --user admin:${JENKINS_ADMIN_TOKEN} \
  -X POST https://jenkins.docker01.vernify.internal:8443/job/dryrun-test/build
```
