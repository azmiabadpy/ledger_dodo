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
