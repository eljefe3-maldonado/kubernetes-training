# Reference Architecture: Secure Managed Kubernetes Platform

Complete each section for your chosen cloud provider (EKS / GKE / AKS).
Replace placeholder values and delete any sections that do not apply.

---

## Platform Overview

| Field | Value |
| --- | --- |
| Cloud provider | |
| Kubernetes service | EKS / GKE / AKS |
| Cluster version | |
| Region | |
| Environment | Production |
| Tenancy model | Shared / Dedicated (circle one) |
| Teams served | |

---

## Identity And Access

### Human Access

| Access Pattern | Mechanism | MFA Required | Notes |
| --- | --- | --- | --- |
| Cluster admin (break-glass) | | | |
| Platform engineer (daily ops) | | | |
| Application developer (namespace-scoped) | | | |
| Security engineer (read + audit) | | | |
| CI/CD pipeline | | | |

**OIDC / Cloud IAM integration:**
- Identity provider:
- Token issuer URL:
- How RBAC subjects map to IdP groups:

### Workload Identity (Pod-Level IAM)

| Provider | Mechanism | Notes |
| --- | --- | --- |
| EKS | IRSA (IAM Roles for Service Accounts) via OIDC federation | |
| GKE | Workload Identity (GCP Service Account bound to KSA) | |
| AKS | Azure AD Workload Identity (federated credential on managed identity) | |

Describe your workload identity design:

- Which workloads use cloud IAM credentials?
- How are the IAM role-to-namespace bindings governed?
- How are unused roles audited?

---

## Networking

### VPC And Subnet Layout

```
[ Describe or diagram your VPC layout ]

Example:
VPC: 10.0.0.0/16
  Private subnets (nodes):    10.0.1.0/24, 10.0.2.0/24, 10.0.3.0/24
  Private subnets (pods):     10.0.16.0/20, 10.0.32.0/20, 10.0.48.0/20
  Private subnets (data):     10.0.64.0/24, 10.0.65.0/24
  No public subnets for nodes or pods
```

### Cluster Access

- API server endpoint:  [ Private only / Public with CIDR restriction ]
- Authorized IP ranges for API server access:
- Bastion or VPN solution for engineer access:

### In-Cluster Networking

- CNI plugin:
- NetworkPolicy enforcement: [ Enabled / Disabled ]
- Default-deny applied to all production namespaces: [ Yes / No ]
- Ingress controller:
- Ingress TLS termination:

### Egress Controls

- Default egress posture: [ Default allow / Default deny with allowlist ]
- Cloud metadata API (169.254.169.254) access restricted to authorized workloads: [ Yes / No ]
- Mechanism for egress control (NetworkPolicy / NAT gateway allowlist / egress gateway):

---

## Secrets And Key Management

### Etcd Encryption

| Control | Status | Notes |
| --- | --- | --- |
| Encryption at rest enabled | | |
| Encryption key managed by: | Provider-managed / Customer KMS | |
| KMS key rotation schedule | | |
| Who can access the KMS key | | |

### Secret Delivery

| Pattern | Used For | Notes |
| --- | --- | --- |
| Native Kubernetes secrets | | |
| External Secrets Operator | | |
| CSI Secrets Store Driver | | |
| Direct SDK calls from workload (e.g., AWS SDK) | | |

- External secrets manager:
- Secret rotation schedule:
- How leaked secrets are detected:

---

## Policy And Admission Control

### Pod Security

- Pod Security Admission: [ Enabled / Disabled ]
- Default enforcement level for production namespaces:
- Namespaces enforcing Privileged (and why):

### Policy Engine

- Engine: [ Kyverno / OPA Gatekeeper / Both / None ]
- Policies enforcing:
  - [ ] Approved registry only
  - [ ] Resource limits required
  - [ ] No privileged containers
  - [ ] No host namespace sharing
  - [ ] Image signature verification
  - [ ] Other:

- Exception process:
- Policy violation SLA (how quickly must a violation be remediated):

---

## Observability And Detection

### Log Sources

| Source | Enabled | Destination | Retention |
| --- | --- | --- | --- |
| API server audit log | | | |
| Node / kubelet logs | | | |
| Container stdout/stderr | | | |
| Cloud control-plane logs | | | |
| VPC flow logs | | | |
| DNS logs | | | |

### Alerting

| Alert | Severity | On-Call? | Runbook |
| --- | --- | --- | --- |
| kubectl exec in production | | | |
| Secret read by unexpected principal | | | |
| ClusterRoleBinding created | | | |
| Privileged pod created | | | |
| New workload in kube-system | | | |
| API server auth failure spike | | | |

### Runtime Detection

- Tool: [ Falco / Cloud-native runtime / None ]
- Ruleset version:
- Alert destination:
- On-call integration:

---

## Incident Response

### Isolation Procedure

1. Identify the affected pod or namespace
2. Apply default-deny NetworkPolicy to isolate network
3. Capture pod spec and logs before any deletion
4. Review audit logs for all actions by the compromised service account
5. Rotate service account and any accessed secrets

### Forensics Access

- How do responders access nodes without direct SSH?
- Ephemeral debug container policy:
- Log retention for forensics (minimum 90 days recommended):

### Credential Rotation Runbook

| Credential Type | Rotation Procedure | Time To Rotate |
| --- | --- | --- |
| Service account token | Delete and recreate SA | |
| Database secret | Update in external secrets manager | |
| Cloud IAM role | Rotate or delete bound credentials | |
| TLS certificate | Reissue via cert-manager or provider | |

---

## Platform Upgrade And Patch Management

| Item | Current State | Target State | Owner |
| --- | --- | --- | --- |
| Kubernetes minor version | | | |
| Node OS image | | | |
| Container runtime | | | |
| Add-on versions | | | |
| Base images in use | | | |

- Version support window for current Kubernetes version:
- Upgrade cadence:
- Testing procedure before production upgrade:

---

## Open Risks And Assumptions

List any controls described in this architecture that are not yet implemented,
and the residual risk each represents.

| Control | Status | Residual Risk | Remediation Timeline |
| --- | --- | --- | --- |
| | Planned | | |
| | In progress | | |
| | Not started | | |
