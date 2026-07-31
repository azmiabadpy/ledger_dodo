# Task 1 – Secure Kubernetes Deployment

## Dodo Payments DevSecOps Technical Assessment

---

# Objective

The objective of this task was to securely deploy the **ledger-api** application along with a neighbouring service on Kubernetes while implementing production-grade security controls. The deployment demonstrates secure workload configuration, secret management, least-privilege access control, and Kubernetes admission policy enforcement.

The implementation satisfies the following requirements:

- Deploy ledger-api and a neighbouring service using Kubernetes Deployments and Services.
- Configure ConfigMaps and Ingress.
- Harden container security using Kubernetes Security Contexts.
- Configure readiness and liveness probes.
- Define CPU and memory resource requests/limits.
- Replace plaintext secrets with Sealed Secrets.
- Implement dedicated Service Accounts and least-privilege RBAC.
- Enforce admission policies using Kyverno.
- Demonstrate rejection of insecure deployments.

---

# Environment

| Component | Value |
|------------|-------|
| Kubernetes Distribution | Minikube |
| Container Runtime | Docker |
| Ingress Controller | NGINX Ingress Controller |
| Admission Controller | Kyverno |
| Secret Management | Bitnami Sealed Secrets |
| Namespace | payments |

---

# Repository Structure

```text
ledger-api-assignment/
│
├── app/
│   ├── app.py
│   ├── Dockerfile
│   └── requirements.txt
│
├── deploy/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── neighbour.yaml
│   ├── serviceaccount.yaml
│   ├── role.yaml
│   ├── rolebinding.yaml
│   ├── sealed-secret.yaml
│   │
│   ├── policies/
│   │   ├── require-non-root.yaml
│   │   ├── disallow-latest-tag.yaml
│   │   └── verify-image.yaml
│   │
│   └── network-policies/
│
└── README.md
```

---

# Architecture

```
                        Internet
                             │
                      NGINX Ingress
                             │
                ┌────────────────────────┐
                │      ledger-api        │
                │      Deployment        │
                │      (3 Replicas)      │
                └────────────────────────┘
                             │
                     ClusterIP Service
                             │
                ┌────────────────────────┐
                │   reporting service    │
                │      Deployment        │
                └────────────────────────┘

                 Namespace: payments

Security Layers

• Kyverno Admission Controller
• Service Accounts
• RBAC
• Sealed Secrets
• Security Context
• Liveness & Readiness Probes
```

---

# Kubernetes Resources Created

The following Kubernetes resources were created as part of the deployment.

| Resource | Purpose |
|-----------|---------|
| Namespace | Isolates application resources |
| Deployment | Deploys ledger-api |
| Deployment | Deploys reporting service |
| Service | Internal service communication |
| ConfigMap | Stores application configuration |
| Ingress | External HTTP access |
| ServiceAccount | Dedicated workload identity |
| Role | Least privilege permissions |
| RoleBinding | Associates Role with ServiceAccount |
| SealedSecret | Secure storage of secrets |

---

# Deployment Steps

## 1. Create Namespace

```bash
kubectl apply -f deploy/namespace.yaml
```

---

## 2. Create ConfigMap

```bash
kubectl apply -f deploy/configmap.yaml
```

---

## 3. Create Dedicated Service Accounts

```bash
kubectl apply -f deploy/serviceaccount.yaml
```

Dedicated Service Accounts were created instead of using the default Kubernetes ServiceAccount.

Service Accounts used:

- ledger-api-sa
- reporting

---

## 4. Configure RBAC

```bash
kubectl apply -f deploy/role.yaml
kubectl apply -f deploy/rolebinding.yaml
```

Least-privilege Role and RoleBinding resources were configured to grant only the permissions required by the workloads.

---

## 5. Deploy Sealed Secrets

Sensitive configuration values were removed from Git and replaced with encrypted Sealed Secrets.

```bash
kubectl apply -f deploy/sealed-secret.yaml
```

The Sealed Secrets controller decrypts secrets only inside the Kubernetes cluster.

---

## 6. Deploy Applications

```bash
kubectl apply -f deploy/deployment.yaml
kubectl apply -f deploy/neighbour.yaml
```

The deployment creates:

- ledger-api
- reporting service

Both workloads are deployed inside the **payments** namespace.

---

## 7. Expose the Application

```bash
kubectl apply -f deploy/service.yaml
kubectl apply -f deploy/ingress.yaml
```

The application is exposed externally through the Kubernetes Ingress resource.

---

# Security Hardening

The deployment was hardened using Kubernetes security best practices to minimise the attack surface and comply with the assessment requirements.

## Security Context

Every workload was configured with the following security settings.

### Run as Non-root

Containers are prevented from running as the root user.

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
```

---

### Read-only Root Filesystem

The root filesystem is mounted as read-only to prevent runtime modifications.

```yaml
securityContext:
  readOnlyRootFilesystem: true
```

---

### Drop Linux Capabilities

All Linux capabilities were removed from the containers.

```yaml
securityContext:
  capabilities:
    drop:
      - ALL
```

---

### Disable Privilege Escalation

Containers are prevented from gaining additional privileges during execution.

```yaml
securityContext:
  allowPrivilegeEscalation: false
```

---

### Runtime Default Seccomp Profile

The default Linux seccomp profile was enabled.

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
```

---

# Resource Management

Resource requests and limits were configured for every application container.

Example configuration:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi

  limits:
    cpu: 500m
    memory: 512Mi
```

Benefits:

- Prevents noisy neighbour problems.
- Ensures predictable scheduling.
- Protects cluster resources.

---

# Health Checks

Each deployment includes Kubernetes health probes.

## Readiness Probe

The readiness probe ensures traffic is only routed to healthy Pods.

Example:

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
```

---

## Liveness Probe

The liveness probe automatically restarts unhealthy containers.

Example:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
```

---

# Service Accounts

The default Kubernetes ServiceAccount was not used.

Dedicated ServiceAccounts were created for each workload.

| Workload | ServiceAccount |
|----------|----------------|
| ledger-api | ledger-api-sa |
| reporting | reporting |

Using dedicated ServiceAccounts provides:

- Better workload identity
- Reduced attack surface
- Least privilege access

---

# Role-Based Access Control (RBAC)

RBAC was configured using Kubernetes Roles and RoleBindings.

Resources created:

- Role
- RoleBinding

Permissions were scoped only to the Kubernetes resources required by the application.

The default ServiceAccount was intentionally not used.

---

# Secret Management

Initially, application secrets were stored as plaintext environment variables.

To improve security, plaintext secrets were removed from Git and replaced with **Bitnami Sealed Secrets**.

Benefits:

- Secrets remain encrypted inside the Git repository.
- Only the Sealed Secrets controller can decrypt them.
- No sensitive credentials are committed to source control.

Deployment:

```bash
kubectl apply -f deploy/sealed-secret.yaml
```

---

# Kyverno Admission Policies

Kyverno was installed as the admission controller to enforce security guardrails before workloads are admitted into the cluster.

The following ClusterPolicies were implemented.

---

## Policy 1 – Require Non-root Containers

Purpose:

Reject any Pod that attempts to run as the root user.

Validation:

```yaml
runAsNonRoot: true
```

---

## Policy 2 – Disallow Latest Image Tag

Purpose:

Prevent deployments using mutable image tags.

Rejected Example:

```
nginx:latest
```

Recommended:

```
nginx:1.27.0
```

---

## Policy 3 – Image Verification

Purpose:

Prevent deployment of unsigned or unverified container images.

This policy ensures only trusted images are deployed.

---

# Admission Policy Validation

The original insecure deployment was intentionally tested.

Kyverno successfully rejected workloads that:

- executed as root
- used the `latest` image tag
- violated required security policies

Example validation error:

```
validation error:

Container must run as non-root user.
```

Another validation example:

```
Using latest image tag is not allowed.
```

These tests confirmed that the admission controller correctly prevented insecure workloads from entering the cluster.

---

# Verification

The deployment was verified after all resources were successfully applied.

## Verify Namespace

```bash
kubectl get namespace payments
```

Expected Result:

```
payments   Active
```

---

## Verify Deployments

```bash
kubectl get deployments -n payments
```

Expected Deployments:

- ledger-api
- reporting

---

## Verify Pods

```bash
kubectl get pods -n payments
```

Expected:

- All Pods in **Running** state
- All containers ready

Example:

```
NAME                          READY   STATUS    RESTARTS
ledger-api-xxxxx              1/1     Running   0
ledger-api-xxxxx              1/1     Running   0
ledger-api-xxxxx              1/1     Running   0
reporting-xxxxx               1/1     Running   0
```

---

## Verify Services

```bash
kubectl get svc -n payments
```

Expected Services:

- ledger-api
- reporting

---

## Verify Ingress

```bash
kubectl get ingress -n payments
```

Confirm that the application is exposed through the configured Ingress resource.

---

## Verify ConfigMap

```bash
kubectl get configmap -n payments
```

Verify that application configuration has been successfully loaded.

---

## Verify Service Accounts

```bash
kubectl get sa -n payments
```

Expected:

```
ledger-api-sa
reporting
```

The workloads should not use the default ServiceAccount.

---

## Verify RBAC

```bash
kubectl get role -n payments
kubectl get rolebinding -n payments
```

Confirm that the Roles and RoleBindings have been successfully created.

---

## Verify Sealed Secrets

```bash
kubectl get sealedsecrets -n payments
kubectl get secrets -n payments
```

Confirm that the encrypted secret has been successfully decrypted inside the cluster.

---

## Verify Kyverno Policies

```bash
kubectl get cpol
```

Expected policies:

- require-non-root
- disallow-latest-tag
- verify-image

---

## Verify Policy Enforcement

The original insecure deployment was intentionally tested to validate admission control.

### Test 1 – Root Container

Attempted to deploy a container running as root.

Result:

```
Rejected by Kyverno
```

Example:

```
Container must run as non-root user.
```

---

### Test 2 – Latest Image Tag

Attempted deployment using:

```
nginx:latest
```

Result:

```
Rejected by Kyverno
```

Example:

```
Using latest image tag is not allowed.
```

These tests confirmed that insecure workloads are blocked before being admitted into the cluster.

---

# Security Features Implemented

The following security controls were successfully implemented.

| Security Control | Status |
|------------------|--------|
| Dedicated Namespace | ✅ |
| Deployments | ✅ |
| Services | ✅ |
| ConfigMaps | ✅ |
| Ingress | ✅ |
| Non-root Containers | ✅ |
| Read-only Root Filesystem | ✅ |
| Drop Linux Capabilities | ✅ |
| Disable Privilege Escalation | ✅ |
| RuntimeDefault Seccomp | ✅ |
| CPU Requests/Limits | ✅ |
| Memory Requests/Limits | ✅ |
| Liveness Probe | ✅ |
| Readiness Probe | ✅ |
| Dedicated Service Accounts | ✅ |
| Least-Privilege RBAC | ✅ |
| Sealed Secrets | ✅ |
| Kyverno Admission Policies | ✅ |
| Rejected Insecure Deployments | ✅ |

---

# Assignment Requirement Mapping

| Requirement | Status |
|-------------|--------|
| Deploy ledger-api | ✅ Completed |
| Deploy neighbour service | ✅ Completed |
| Deployments | ✅ Completed |
| Services | ✅ Completed |
| ConfigMaps | ✅ Completed |
| Ingress | ✅ Completed |
| Secure securityContext | ✅ Completed |
| Resource requests/limits | ✅ Completed |
| Liveness probes | ✅ Completed |
| Readiness probes | ✅ Completed |
| Dedicated ServiceAccount | ✅ Completed |
| Least-Privilege RBAC | ✅ Completed |
| Remove plaintext secrets | ✅ Completed |
| Sealed Secrets | ✅ Completed |
| Kyverno admission controller | ✅ Completed |
| Reject root containers | ✅ Completed |
| Reject latest image tag | ✅ Completed |
| Image verification policy | ✅ Implemented |

---

# Bonus Features

The following additional security improvements were implemented beyond the minimum requirements.

- Dedicated ServiceAccounts for individual workloads.
- Least-Privilege RBAC configuration.
- Admission policy validation using intentionally insecure deployments.
- Production-style hardened Pod Security Contexts.
- Encrypted secret management using Sealed Secrets.

---

# Challenges Encountered

During the implementation the following challenges were encountered:

- Migrating application secrets from plaintext manifests to Sealed Secrets.
- Updating workloads to satisfy Kyverno admission policies.
- Ensuring all Pods complied with hardened security settings.
- Validating policy enforcement using intentionally insecure deployments.

These issues were resolved by iteratively updating the Kubernetes manifests and verifying deployment behaviour.

---

# Conclusion

The objective of this task was successfully achieved by securely deploying the **ledger-api** application and its neighbouring service on Kubernetes while applying multiple layers of security.

The final deployment demonstrates:

- Secure Kubernetes workload configuration.
- Least-privilege access using RBAC and dedicated ServiceAccounts.
- Secure secret management using Sealed Secrets.
- Admission control using Kyverno.
- Hardened container runtime configuration.
- Health monitoring through readiness and liveness probes.
- Resource management through CPU and memory limits.

The deployment follows Kubernetes security best practices and satisfies the Task 1 assessment requirements by implementing defence-in-depth across workload configuration, identity, secret management, and admission control.


# Task 2 – Secure CI/CD Pipeline & Software Supply Chain

## Dodo Payments DevSecOps Assessment

---

# Objective

The objective of this task was to build a secure end-to-end Continuous Integration and Continuous Deployment (CI/CD) pipeline that automatically validates, builds, signs, and deploys container images while enforcing software supply chain security.

Instead of relying on manual validation, multiple security gates were integrated directly into the CI/CD workflow to prevent vulnerable code, leaked secrets, insecure dependencies, or unsigned container images from reaching production.

The completed implementation includes:

- GitHub Actions based CI/CD
- Docker image build
- GitHub Container Registry (GHCR)
- Static Application Security Testing (Semgrep)
- Secret Detection (Gitleaks)
- Filesystem Vulnerability Scan (Trivy)
- Container Image Scan (Trivy)
- Cosign Keyless Image Signing
- Signature Verification
- SLSA Build Provenance
- GitOps deployment using ArgoCD
- Drift Detection
- Automatic Self Healing

---

# Environment

| Component | Technology |
|------------|------------|
| Source Control | GitHub |
| CI Platform | GitHub Actions |
| Container Registry | GitHub Container Registry (GHCR) |
| SAST | Semgrep |
| Secret Scan | Gitleaks |
| Vulnerability Scanner | Trivy |
| Image Signing | Cosign (Keyless) |
| Provenance | SLSA |
| GitOps | ArgoCD |
| Kubernetes | Minikube |

---

# CI/CD Architecture

```

```
Developer
    │
    ▼
GitHub Repository
    │
    ▼
GitHub Actions Workflow
    │
    ├──────────────► Checkout Repository
    │
    ├──────────────► Build Docker Image
    │
    ├──────────────► Gitleaks Secret Scan
    │
    ├──────────────► Semgrep SAST
    │
    ├──────────────► Trivy Filesystem Scan
    │
    ├──────────────► Trivy Image Scan
    │
    ├──────────────► Push Image to GHCR
    │
    ├──────────────► Cosign Keyless Signing
    │
    ├──────────────► Verify Signature
    │
    ├──────────────► Generate SLSA Provenance
    │
    ▼
Git Repository (GitOps)
    │
    ▼
ArgoCD
    │
    ▼
Kubernetes Cluster
```

---

# GitHub Actions Workflow

The complete CI/CD workflow was implemented in:

```

.github/workflows/secure-pipeline.yaml

```

The workflow is automatically triggered on:

```yaml
on:
  push:
    branches:
      - main

  pull_request:
    branches:
      - main
```

---

# Workflow Permissions

GitHub Actions permissions were configured as follows:

```yaml
permissions:
  contents: read
  packages: write
  security-events: write
  id-token: write
  attestations: write
```

These permissions are required for:

| Permission | Purpose |
|------------|----------|
| contents | Read repository |
| packages | Push Docker images to GHCR |
| security-events | Upload security results |
| id-token | OIDC authentication for Cosign |
| attestations | Publish SLSA provenance |

---

# Pipeline Stages

The GitHub Actions workflow performs the following operations.

```
Checkout Repository
        │
        ▼
Docker Buildx
        │
        ▼
Login to GHCR
        │
        ▼
Gitleaks Scan
        │
        ▼
Semgrep Scan
        │
        ▼
Trivy Filesystem Scan
        │
        ▼
Build Docker Image
        │
        ▼
Trivy Image Scan
        │
        ▼
Push Image to GHCR
        │
        ▼
Cosign Sign
        │
        ▼
Cosign Verify
        │
        ▼
Generate SLSA Provenance
```

---

# Stage 1 – Checkout Repository

The latest version of the repository is checked out.

```yaml
uses: actions/checkout@v4
```

---

# Stage 2 – Docker Buildx

Docker Buildx is configured for multi-platform image builds.

```yaml
uses: docker/setup-buildx-action@v3
```

---

# Stage 3 – Login to GitHub Container Registry

Authentication to GitHub Container Registry is performed using the automatically generated GitHub token.

```yaml
uses: docker/login-action@v3
```

Container images are published to:

```
ghcr.io/<github-username>/ledger-api
```

---

# Stage 4 – Secret Detection

Gitleaks scans the repository for accidentally committed secrets.

Examples include:

- AWS Keys
- API Tokens
- Passwords
- Private Keys
- Database Credentials

Workflow:

```yaml
uses: gitleaks/gitleaks-action@v2
```

If secrets are detected, the pipeline immediately fails.

---

# Stage 5 – Static Application Security Testing

Semgrep performs Static Application Security Testing.

Workflow:

```yaml
uses: semgrep/semgrep-action@v1
```

The scan automatically checks for:

- SQL Injection
- Command Injection
- Hardcoded Secrets
- Unsafe Python Code
- Insecure Functions

The build is blocked if critical findings are detected.

---
# Vulnerability Management and Supply Chain Security

The pipeline implements multiple security gates before a container image is released.

The objective is to ensure that only:

- Secure source code
- Clean dependencies
- Verified container images
- Signed artifacts

are promoted through the delivery pipeline.

---

# Trivy Filesystem Security Scan

Before building the Docker image, Trivy scans the repository filesystem.

The purpose of this stage is to detect vulnerabilities in:

- Application dependencies
- Package files
- Configuration files
- Known CVEs

Implementation:

```yaml
- name: Run Trivy Filesystem Scan
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: fs
    scan-ref: .
    format: table
    severity: HIGH,CRITICAL
    ignore-unfixed: true
    exit-code: 1
```

---

## Trivy Scan Policy

The pipeline follows the following rules:

| Finding | Action |
|---------|--------|
| LOW vulnerability | Allowed |
| MEDIUM vulnerability | Allowed / reviewed |
| HIGH vulnerability with fix | Pipeline blocked |
| CRITICAL vulnerability with fix | Pipeline blocked |
| HIGH/CRITICAL without fix | Reported and monitored |

---

# Docker Image Build

After passing source security checks, the Docker image is built.

The application Dockerfile is located at:

```
app/Dockerfile
```

Build step:

```yaml
- name: Build Docker Image
  uses: docker/build-push-action@v6
  with:
    context: ./app
    file: ./app/Dockerfile
    load: true
    push: false
    tags: ledger-api:test
```

---

# Build Issue Resolution – requirements.txt

During the initial pipeline execution, the Docker build failed because the Python dependencies were not correctly defined.

The failure occurred during:

```
pip install -r requirements.txt
```

The issue was resolved by:

- Reviewing application imports
- Updating `requirements.txt`
- Adding missing Python dependencies
- Rebuilding the Docker image

After updating dependencies:

```
Docker build completed successfully
```

This demonstrated the importance of validating application dependencies before image creation.

---

# Container Image Vulnerability Scan

After the image is created, Trivy performs a second security scan against the built container image.

Implementation:

```yaml
- name: Run Trivy Image Scan
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ledger-api:test
    format: table
    severity: HIGH,CRITICAL
    ignore-unfixed: true
    exit-code: 1
```

This scan validates:

- Base image vulnerabilities
- Installed OS packages
- Application libraries
- Container dependencies

The image cannot be published if blocking vulnerabilities are detected.

---

# Push Container Image to GHCR

After successful security validation, the image is pushed to GitHub Container Registry.

Implementation:

```yaml
- name: Push Docker Image to GHCR
  uses: docker/build-push-action@v6
```

Published image format:

```
ghcr.io/<github-user>/ledger-api:<commit-sha>
```

Example:

```
ghcr.io/azmiabadpy/ledger-api:a83f91c
```

Using commit SHA tags provides:

- Immutable versions
- Traceability
- Rollback capability

---

# Container Image Signing with Cosign

After pushing the image, Cosign is used to sign the container image.

Cosign provides:

- Artifact authenticity
- Supply chain security
- Verification of image origin

Installation:

```yaml
- name: Install Cosign
  uses: sigstore/cosign-installer@v3
```

---

# Cosign SHA Digest Fix

During implementation, the initial signing attempt failed because the image was being referenced incorrectly.

The issue:

- Signing by mutable tag is unreliable.
- Cosign requires the immutable image digest.

The workflow was fixed by using:

```yaml
steps.push.outputs.digest
```

instead of signing by tag.

Final signing command:

```bash
cosign sign --yes \
ghcr.io/${{ github.repository_owner }}/ledger-api@${{ steps.push.outputs.digest }}
```

The digest guarantees that the exact image artifact produced by the pipeline is signed.

---

# Keyless Signing using GitHub OIDC

The pipeline uses Cosign keyless signing.

No private signing key is stored.

Authentication flow:

```
GitHub Actions
        |
        |
        ▼
GitHub OIDC Token
        |
        |
        ▼
Sigstore Fulcio CA
        |
        |
        ▼
Temporary Signing Certificate
        |
        |
        ▼
Container Image Signature
```

Benefits:

- No secret key management
- Short-lived certificates
- Identity-based verification

---

# Signature Verification

After signing, the image signature is verified.

Implementation:

```yaml
- name: Verify Signature
  run: |

    cosign verify \
    --certificate-identity-regexp="https://github.com/${{ github.repository }}/.github/workflows/.*" \
    --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
    ghcr.io/${{ github.repository_owner }}/ledger-api@${{ steps.push.outputs.digest }}
```

Verification confirms:

- Image was signed by GitHub Actions
- Certificate issuer is trusted
- Artifact integrity is maintained

---

# SLSA Build Provenance

The workflow generates an attestation containing build information.

Implementation:

```yaml
- name: Generate SLSA Provenance Attestation
  uses: actions/attest-build-provenance@v2
```

Configuration:

```yaml
with:
  subject-name: ghcr.io/${{ github.repository_owner }}/ledger-api
  subject-digest: ${{ steps.push.outputs.digest }}
  push-to-registry: true
```

---

# Provenance Contains

The generated provenance records:

- Source repository
- Commit SHA
- Workflow identity
- Builder information
- Build timestamp
- Image digest

This enables verification of:

"Where did this image come from?"

and:

"How was this artifact created?"

---

# Complete Supply Chain Flow

```
Source Code
    |
    ▼
Security Scans
    |
    ▼
Docker Build
    |
    ▼
Dependency Scan
    |
    ▼
Image Scan
    |
    ▼
GHCR Push
    |
    ▼
Cosign Signature
    |
    ▼
SLSA Provenance
    |
    ▼
Trusted Artifact
```

---

# Security Gate Summary

| Security Gate | Tool | Failure Action |
|---------------|------|----------------|
| Secret Detection | Gitleaks | Block pipeline |
| SAST | Semgrep | Block critical findings |
| Dependency Scan | Trivy FS | Block HIGH/CRITICAL fixable CVEs |
| Image Scan | Trivy Image | Block HIGH/CRITICAL fixable CVEs |
| Image Signing | Cosign | Block release |
| Signature Verification | Cosign | Block deployment |
| Provenance | SLSA | Block artifact promotion |

---
# GitOps Deployment with ArgoCD

After the container image passed all security gates, the deployment process was implemented using the GitOps approach with ArgoCD.

In GitOps:

- Git repository becomes the single source of truth.
- Kubernetes state is managed declaratively.
- ArgoCD continuously compares Git state with cluster state.
- Any manual drift is detected and automatically corrected.

---

# GitOps Architecture

```
Developer
    |
    |
    ▼
GitHub Repository
(Application + Kubernetes Manifests)
    |
    |
    ▼
ArgoCD
    |
    |
    ├────────────── Compare Desired State
    |
    ├────────────── Detect Drift
    |
    └────────────── Self Heal
             |
             ▼
     Kubernetes Cluster
```

---

# ArgoCD Application

The Kubernetes deployment manifests were connected with ArgoCD.

The ArgoCD Application defines:

- Source Git repository
- Kubernetes namespace
- Deployment manifests
- Synchronization policy

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: ledger-api

spec:

  source:
    repoURL: <github-repository>
    path: deploy

  destination:
    server: https://kubernetes.default.svc
    namespace: payments

  syncPolicy:

    automated:
      prune: true
      selfHeal: true
```

---

# ArgoCD Sync Policy

Automatic synchronization was enabled.

```yaml
syncPolicy:

  automated:
    prune: true
    selfHeal: true
```

This provides:

| Feature | Purpose |
|---------|---------|
| Automated Sync | Automatically deploy Git changes |
| Prune | Remove resources deleted from Git |
| Self Heal | Restore manual cluster changes |

---

# Git as Source of Truth

The desired Kubernetes configuration exists inside Git.

Example:

```
deploy/
│
├── deployment.yaml
├── service.yaml
├── ingress.yaml
├── configmap.yaml
└── sealed-secret.yaml
```

Any changes to application configuration are performed through Git commits.

The Kubernetes cluster should always match the Git repository state.

---

# Drift Detection Demonstration

To validate GitOps behaviour, a manual change was performed directly inside Kubernetes.

Example:

```bash
kubectl edit deployment ledger-api -n payments
```

A replica value was manually modified:

Before:

```yaml
replicas: 3
```

Changed manually to:

```yaml
replicas: 1
```

This created a difference between:

```
Git Desired State
        |
        |
        ▼
replicas: 3


Kubernetes Live State
        |
        |
        ▼
replicas: 1
```

---

# ArgoCD Drift Detection

ArgoCD continuously monitors the cluster.

After the manual modification, ArgoCD detected:

```
Application Status:

OutOfSync
```

The difference was displayed between:

- Git manifest
- Live Kubernetes resource

This proved that ArgoCD successfully detected configuration drift.

---

# Self-Healing Demonstration

Because self-healing was enabled:

```yaml
selfHeal: true
```

ArgoCD automatically restored the Kubernetes resource.

After synchronization:

```
Application Status:

Synced
Healthy
```

The deployment returned to the Git-defined state.

Final verification:

```bash
kubectl get deployment ledger-api -n payments
```

Result:

```
NAME          READY
ledger-api    3/3
```

---

# End-to-End Secure Delivery Flow

The complete secure software supply chain implementation:

```
Developer Commit
        |
        ▼
GitHub Repository
        |
        ▼
GitHub Actions
        |
        |
        ├── Gitleaks
        |
        ├── Semgrep
        |
        ├── Trivy Filesystem Scan
        |
        ├── Docker Build
        |
        ├── Trivy Image Scan
        |
        ├── Push to GHCR
        |
        ├── Cosign Signing
        |
        └── SLSA Provenance
                |
                ▼
          Trusted Image
                |
                ▼
             ArgoCD
                |
                ▼
        Kubernetes Deployment
```

---

# Verification Commands

## Check GitHub Actions

Verify successful workflow execution:

```
GitHub Repository
        |
        ▼
Actions Tab
        |
        ▼
Secure CI/CD Pipeline
```

Expected:

```
Workflow completed successfully
```

---

## Verify Container Image

```bash
docker images
```

Verify image exists:

```
ledger-api
```

---

## Verify GHCR Image

Check published package:

```
ghcr.io/<username>/ledger-api
```

---

## Verify Cosign Signature

```bash
cosign verify \
ghcr.io/<username>/ledger-api@<digest>
```

Expected:

```
Verified OK
```

---

## Verify ArgoCD Application

```bash
kubectl get applications -n argocd
```

Expected:

```
ledger-api    Synced    Healthy
```

---

# Assignment Requirement Mapping

| Requirement | Implementation | Status |
|-------------|---------------|--------|
| GitHub Actions Pipeline | secure-pipeline.yaml | ✅ |
| Docker Build | Docker Buildx | ✅ |
| GHCR Publishing | docker/build-push-action | ✅ |
| SAST | Semgrep | ✅ |
| Dependency Scan | Trivy | ✅ |
| Image Scan | Trivy | ✅ |
| Secret Scan | Gitleaks | ✅ |
| Image Signing | Cosign Keyless | ✅ |
| Provenance | SLSA Attestation | ✅ |
| Fail Security Gates | Pipeline Policies | ✅ |
| GitOps | ArgoCD | ✅ |
| Drift Detection | ArgoCD Comparison | ✅ |
| Self Healing | Automated Sync | ✅ |

---

# Challenges Encountered

## 1. Python Dependency Build Failure

During the initial Docker build, the image creation failed because required Python packages were missing.

Resolution:

- Reviewed application imports.
- Updated `requirements.txt`.
- Rebuilt the image successfully.

---

## 2. Cosign Image Signing Failure

Initial signing failed because the image was referenced using a mutable tag.

Problem:

```
ledger-api:latest
```

Solution:

Used immutable image digest:

```
ledger-api@sha256:<digest>
```

using:

```yaml
steps.push.outputs.digest
```

This ensured the exact artifact produced by the pipeline was signed.

---

## 3. GitOps Drift Testing

Manual Kubernetes changes caused intentional drift.

ArgoCD successfully:

- Detected the difference.
- Marked application OutOfSync.
- Restored the desired Git state.

---

# Conclusion

The secure CI/CD pipeline successfully implements a complete DevSecOps software supply chain.

The final solution provides:

- Automated security validation.
- Secure container publishing.
- Cryptographic image verification.
- Build provenance tracking.
- GitOps-based deployment.
- Continuous drift detection.
- Automatic cluster recovery.

The implementation ensures that only trusted, verified, and traceable software artifacts are deployed into Kubernetes.


# Task 3 — Zero Trust Kubernetes Service Mesh Security

## Dodo Payments DevSecOps Assessment

---

# Objective

The objective of this task was to implement a Zero Trust security model inside Kubernetes using Istio Service Mesh.

The goal was to secure service-to-service communication by enforcing:

- Encrypted communication
- Workload identity verification
- Least privilege authorization
- Default deny security policies
- Defence-in-depth network controls

Security decisions were based on workload identity instead of:

- IP addresses
- Network location
- Namespace trust

---

# Zero Trust Security Model

Traditional security model:

```
Inside Network
      |
      ▼
Trusted
```

Zero Trust model:

```
Every Request
      |
      ▼
Authenticate Identity
      |
      ▼
Verify Certificate
      |
      ▼
Authorize Access
      |
      ▼
Allow or Deny
```

Every workload must prove:

- Who it is
- What it is allowed to access
- Whether communication is encrypted

---

# Environment

| Component | Technology |
|-----------|------------|
| Kubernetes Platform | Minikube |
| Service Mesh | Istio 1.30.3 |
| Proxy | Envoy Sidecar |
| Authentication | Istio mTLS |
| Authorization | Istio AuthorizationPolicy |
| Identity | SPIFFE |
| Admission Security | Kyverno |
| Network Security | Kubernetes NetworkPolicy |

---

# Application Architecture

The deployed application consisted of:

```
payments namespace


        ┌─────────────────┐
        │   reporting     │
        │ ServiceAccount  │
        └────────┬────────┘
                 |
                 |
              mTLS
                 |
                 ▼

        ┌─────────────────┐
        │   ledger-api    │
        │ ServiceAccount  │
        └─────────────────┘

```

Communication flow:

```
Application
     |
     ▼
Envoy Sidecar
     |
     ▼
Istio Security Layer
     |
     ▼
Destination Service
```

---

# Installing Istio

Istio version used:

```
Istio 1.30.3
```

Istio was downloaded and installed.

Installation:

```bash
curl -L https://istio.io/downloadIstio | sh -
```

Navigate:

```bash
cd istio-1.30.3
```

Configure PATH:

```bash
export PATH=$HOME/ledger-api-assignment/istio-1.30.3/bin:$PATH
```

---

# Istio CLI Verification

Command:

```bash
istioctl version
```

Output:

```
client version: 1.30.3
control plane version: 1.30.3
data plane version: 1.30.3
```

This confirmed:

- Istio CLI installation
- Istiod availability
- Data plane compatibility

---

# Installing Istio Control Plane

Istio was installed using:

```bash
istioctl install
```

Verification:

```bash
kubectl get pods -n istio-system
```

Expected components:

```
istiod
istio-ingressgateway
istio-egressgateway
```

Example:

```
NAME                         READY
istiod                       1/1
istio-ingressgateway         1/1
istio-egressgateway          1/1
```

---

# Enabling Istio Sidecar Injection

The application namespace was enabled for automatic Envoy injection.

Namespace:

```
payments
```

Command:

```bash
kubectl label namespace payments \
istio-injection=enabled
```

After enabling injection, every new pod receives:

```
Application Container
        +
Envoy Proxy Container
```

---

# Deployments Added to Service Mesh

The following workloads were added:

```
payments namespace

ledger-api
reporting
```

Verification:

```bash
kubectl get pods -n payments
```

Output:

```
ledger-api-xxxxx     2/2 Running
reporting-xxxxx      2/2 Running
```

The `2/2` status confirms:

```
1 Application Container
+
1 Envoy Sidecar Proxy
```

---

# Istio Proxy Verification

Command:

```bash
istioctl proxy-status
```

Example output:

```
NAME

ledger-api.payments
reporting.payments
istio-ingressgateway
istiod
```

Connected resources:

```
CDS
LDS
EDS
RDS
```

Meaning:

| Resource | Purpose |
|-|-|
| CDS | Cluster Discovery |
| LDS | Listener Discovery |
| EDS | Endpoint Discovery |
| RDS | Route Discovery |

This confirmed Envoy proxies were successfully connected with Istiod.

---

# Workload Identity Setup

Zero Trust requires identity-based access instead of IP-based access.

Dedicated ServiceAccounts were created.

---

## Ledger API Identity

Verification:

```bash
kubectl get deployment ledger-api \
-n payments \
-o=jsonpath='{.spec.template.spec.serviceAccountName}'
```

Output:

```
ledger-api-sa
```

---

## Reporting Identity

Verification:

```bash
kubectl get deployment reporting \
-n payments \
-o=jsonpath='{.spec.template.spec.serviceAccountName}'
```

Output:

```
reporting
```

---

# SPIFFE Workload Identity

Istio converts Kubernetes ServiceAccounts into SPIFFE identities.

Identity format:

```
spiffe://cluster.local/ns/<namespace>/sa/<service-account>
```

Example:

```
spiffe://cluster.local/ns/payments/sa/ledger-api-sa
```

The identity is used by Istio AuthorizationPolicy.

---

# Result

Completed:

✅ Istio installed  
✅ Workloads added to mesh  
✅ Envoy sidecars injected  
✅ Workload identities configured  
✅ ServiceAccounts separated  

Next section:

- STRICT mTLS enforcement
- Certificate issuance
- Plaintext traffic rejection
- AuthorizationPolicy Zero Trust access control


# STRICT Mutual TLS (mTLS) Enforcement

After workloads were added into the Istio mesh, mutual TLS was enforced to ensure that all service-to-service communication was:

- Encrypted
- Authenticated
- Identity verified

Without mTLS, any workload that can reach the service network could attempt communication.

With mTLS:

```
Client Workload
        |
        |
        ▼
Client Envoy
        |
        |
   mTLS Encryption
        |
        |
        ▼
Server Envoy
        |
        |
        ▼
Application
```

---

# PeerAuthentication Policy

Istio `PeerAuthentication` was used to enforce STRICT mTLS mode.

Configuration:

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication

metadata:
  name: default
  namespace: payments

spec:

  mtls:
    mode: STRICT
```

Applied:

```bash
kubectl apply -f peer-authentication.yaml
```

---

# STRICT Mode Behaviour

The following communication is allowed:

```
Workload A
    |
    |
    ▼
Envoy Sidecar
    |
    |
    ▼
mTLS Connection
    |
    |
    ▼
Workload B
```

The following is rejected:

```
External Client
       |
       |
       X
       |
       ▼
Service
```

because no valid Istio certificate exists.

---

# Verifying mTLS Status

Istio TLS verification command:

```bash
istioctl authn tls-check
```

Example:

```
HOST
SOURCE
STATUS

ledger-api.payments
reporting.payments

CONNECTION:
mTLS
```

This confirms workload communication is protected by mutual TLS.

---

# Plaintext Traffic Test

A plaintext request was tested against the service.

Command:

```bash
curl -v http://ledger-api.payments:8080/health
```

Initial result:

```
Connection reset by peer
```

The request was rejected because:

- No client certificate
- No mTLS handshake
- STRICT mode enabled

This proved that plaintext communication was not accepted.

---

# Workload Certificate Architecture

Istio automatically manages workload certificates.

Certificate flow:

```
Application Pod
        |
        |
        ▼
Kubernetes ServiceAccount
        |
        |
        ▼
Istio Agent
        |
        |
        ▼
Istiod CA
        |
        |
        ▼
Workload Certificate
        |
        |
        ▼
Envoy Sidecar
```

The application does not manage certificates manually.

Envoy handles:

- Certificate presentation
- TLS handshake
- Certificate rotation

---

# Certificate Issuing Process

When a pod starts:

1. Envoy sidecar starts.
2. Istio agent requests certificate.
3. Istiod validates workload identity.
4. Istiod CA issues certificate.
5. Certificate is stored inside Envoy.
6. Envoy uses certificate for mTLS communication.

---

# Certificate Rotation

Istio automatically rotates certificates before expiry.

Benefits:

- No manual renewal
- Reduced key exposure
- Short-lived workload credentials

---

# Trust Root

The trust root is:

```
Istio Certificate Authority (CA)
```

The trust chain:

```
Istio Root CA
       |
       |
       ▼
Istiod CA
       |
       |
       ▼
Workload Certificate
       |
       |
       ▼
Envoy Sidecar
```

All workloads inside the mesh trust certificates signed by the same Istio CA.

---

# AuthorizationPolicy - Zero Trust Authorization

Authentication proves:

```
Who are you?
```

Authorization decides:

```
What are you allowed to access?
```

Istio AuthorizationPolicy was used to enforce least privilege access.

---

# Default Deny Policy

A default deny policy was created.

Concept:

```
No workload is trusted by default
```

Example:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy

metadata:
  name: default-deny
  namespace: payments

spec:

  {}
```

This blocks all requests unless explicitly allowed.

---

# Identity Based Allow Policy

Access was granted based on workload identity.

NOT based on:

❌ IP Address  
❌ Namespace only  
❌ Network location  

Allowed identity:

```
ServiceAccount:
reporting
```

Example:

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy

metadata:
  name: allow-reporting
  namespace: payments

spec:

  selector:

    matchLabels:
      app: ledger-api


  rules:

  - from:

    - source:

        principals:

        - cluster.local/ns/payments/sa/reporting
```

---

# Unauthorized Workload Test

An unauthorized workload attempted access.

Example:

```
Unknown workload
        |
        ▼
ledger-api
```

Request:

```bash
curl http://ledger-api.payments:8080/health
```

Response:

```
HTTP/1.1 403 Forbidden

RBAC: access denied
```

This confirmed:

- Authentication succeeded
- Authorization failed

---

# Authorized Workload Test

The reporting workload was allowed.

Verification:

```bash
kubectl exec deployment/reporting \
-n payments -- sh
```

Request:

```bash
curl http://ledger-api.payments:8080/health
```

Successful response:

```json
{
 "status":"ok"
}
```

This proved:

```
reporting
      |
      |
      ▼
Istio AuthorizationPolicy
      |
      |
      ▼
ledger-api
```

was working correctly.

---

# Security Model Achieved

The final request flow:

```
Request
   |
   ▼
mTLS Authentication
   |
   ▼
SPIFFE Identity Verification
   |
   ▼
AuthorizationPolicy Check
   |
   ▼
Application Access
```

---

# Result

Completed:

✅ STRICT mTLS enabled  
✅ Plaintext traffic rejected  
✅ Workload certificates issued automatically  
✅ Istio CA trust chain verified  
✅ Default deny AuthorizationPolicy  
✅ Identity-based allow rules  
✅ Unauthorized workload blocked  
✅ Authorized workload allowed  

Next section:

- Kubernetes NetworkPolicy defence-in-depth
- CNI limitation documentation
- Testing results
- Bonus items status
- Final assignment mapping

  # Kubernetes NetworkPolicy Defence-in-Depth

Istio provides Layer 7 security controls:

- Identity authentication
- mTLS encryption
- Application-level authorization

However, Kubernetes NetworkPolicy provides an additional Layer 3/Layer 4 security boundary.

The objective was to create defence-in-depth:

```
                Request

                   |
                   ▼

        Istio Service Mesh Layer
        -------------------------
        mTLS Authentication
        SPIFFE Identity
        AuthorizationPolicy

                   |
                   ▼

        Kubernetes Network Layer
        -------------------------
        NetworkPolicy
        Pod Connectivity Control

                   |
                   ▼

             Application
```

---

# Why NetworkPolicy Was Added

Istio AuthorizationPolicy protects:

```
Who can call the service?
```

NetworkPolicy protects:

```
Who can reach the pod network?
```

Both layers solve different security problems.

---

# NetworkPolicy Implementation

NetworkPolicy files were created:

```
deploy/
 |
 └── network-policies/
        |
        ├── default-deny-ledger.yaml
        |
        └── allow-reporting-to-ledger.yaml
```

---

# Default Deny Policy

A default deny ingress policy was created for ledger-api.

Purpose:

- Block all incoming traffic by default
- Remove implicit trust
- Require explicit allow rules

Example:

```yaml
apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: default-deny-ledger-ingress

  namespace: payments


spec:

  podSelector:

    matchLabels:

      app: ledger-api


  policyTypes:

  - Ingress
```

---

# Explicit Allow Policy

Only the reporting workload was allowed to communicate with ledger-api.

Example:

```yaml
apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: allow-reporting-to-ledger

  namespace: payments


spec:

  podSelector:

    matchLabels:

      app: ledger-api


  ingress:

  - from:

    - podSelector:

        matchLabels:

          app: reporting
```

---

# Applying NetworkPolicies

Command:

```bash
kubectl apply -f default-deny-ledger.yaml

kubectl apply -f allow-reporting-to-ledger.yaml
```

Verification:

```bash
kubectl get networkpolicy -n payments
```

Output:

```
NAME

default-deny-ledger-ingress

allow-reporting-to-ledger
```

---

# NetworkPolicy Testing

## Allowed Communication

The reporting service was tested.

Command:

```bash
kubectl exec -it deployment/reporting \
-n payments -- sh
```

Inside the pod:

```bash
curl http://ledger-api.payments:8080/health
```

Response:

```json
{
 "status":"ok"
}
```

Result:

```
reporting
     |
     |
     ▼
NetworkPolicy Allow
     |
     |
     ▼
ledger-api
```

---

# CNI Validation Limitation

During NetworkPolicy testing, deeper CNI-level verification was attempted.

The Kubernetes environment was running on:

```
Minikube
```

CNI verification command:

```bash
kubectl get nodes -o yaml | grep -i cni
```

However, the expected CNI information was not available.

Because Kubernetes NetworkPolicy enforcement depends on the underlying CNI implementation, complete packet-level validation could not be performed.

---

# Impact of CNI Limitation

Because the CNI enforcement layer could not be fully validated, the following test was not completed:

```
Unauthorized Pod

       |
       |
       X

ledger-api
```

Expected behaviour:

```
Connection blocked by NetworkPolicy
```

We did not claim this test as completed because the enforcement engine could not be verified.

---

# Future Improvement

For production-grade Kubernetes security testing, the cluster should use a CNI with advanced NetworkPolicy visibility.

Recommended:

## Calico

Provides:

- NetworkPolicy enforcement
- Flow visibility
- Policy logs

## Cilium

Provides:

- eBPF networking
- Network observability
- Security monitoring

Future testing would include:

- Packet-level deny verification
- Network flow logs
- CNI enforcement events
- Policy debugging

---

# Security Layer Comparison

| Security Layer | Protects Against | Example |
|---|---|---|
| Istio mTLS | Traffic interception | Prevents plaintext communication |
| SPIFFE Identity | Fake workloads | Verifies workload identity |
| AuthorizationPolicy | Unauthorized service calls | Blocks unwanted API access |
| NetworkPolicy | Network reachability | Blocks pod communication |

---

# Bonus Features Status

## Istio Ingress Gateway TLS Termination

Status:

```
Not implemented
```

Reason:

Due to time limitations, production ingress TLS termination was documented as future enhancement.

Future implementation:

```
External Client
       |
       ▼
Istio Gateway
       |
       ▼
TLS Termination
       |
       ▼
Service Mesh
```

---

# Canary Deployment

Status:

```
Not implemented
```

Future implementation would use:

- VirtualService
- DestinationRule

Example:

```
90% Traffic
     |
     ▼
ledger-api-v1


10% Traffic
     |
     ▼
ledger-api-v2
```

Benefits:

- Safe releases
- Gradual rollout
- Easy rollback

---

# PCI Cardholder Data Environment (CDE) Mapping

Although the application was a demonstration environment, the Zero Trust design maps directly to PCI DSS requirements.

---

## Requirement Mapping

| PCI Security Need | Implemented Control |
|-|-|
| Protect data in transit | Istio mTLS |
| Restrict access | AuthorizationPolicy |
| Least privilege | ServiceAccount identity |
| Network segmentation | NetworkPolicy |
| Continuous security enforcement | Istio policies |

---

# Final Task 3 Verification

| Requirement | Status |
|---|---|
| Install Istio | ✅ Completed |
| Add workloads into mesh | ✅ Completed |
| STRICT mTLS | ✅ Completed |
| Plaintext blocked | ✅ Completed |
| TLS verification | ✅ Completed |
| Workload certificates | ✅ Completed |
| SPIFFE identity | ✅ Completed |
| Default deny AuthorizationPolicy | ✅ Completed |
| Identity-based allow rules | ✅ Completed |
| Unauthorized workload blocked | ✅ Completed |
| Authorized workload allowed | ✅ Completed |
| NetworkPolicy created | ✅ Completed |
| CNI packet validation | ⚠️ Not completed |
| Istio Gateway TLS | ❌ Future enhancement |
| Canary deployment | ❌ Future enhancement |

---

# Conclusion

The Zero Trust security model was successfully implemented using Istio.

The final security architecture provides:

- Encrypted service communication
- Strong workload identity
- Least privilege authorization
- Defence-in-depth network controls
- Continuous security enforcement

The remaining improvements are production-level enhancements:

- Advanced CNI validation
- Gateway TLS management
- Canary deployment strategy

The implementation demonstrates how Kubernetes workloads can be secured using modern Zero Trust principles.


