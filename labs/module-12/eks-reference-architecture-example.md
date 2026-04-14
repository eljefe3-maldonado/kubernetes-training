# Completed Reference Architecture: Secure EKS Platform (Model Answer)

This is a fully worked example for a production EKS cluster. Use it to
validate your own design after you have completed `reference-architecture.md`.
Compare each section against your choices and note where you made different
decisions — the reasoning matters more than matching this exactly.

Scenario: a multi-tenant internal platform serving three product teams with
mixed data sensitivity. One team processes customer PII. No PCI scope.

---

## Platform Overview

| Field | Value |
| --- | --- |
| Cloud provider | AWS |
| Kubernetes service | Amazon EKS |
| Kubernetes version | 1.30 (one minor version behind latest 1.31) |
| Region | us-east-1 (primary), us-west-2 (DR) |
| Environment | Production |
| Tenancy model | Shared cluster, namespace-per-team |
| Teams served | platform-eng, app-alpha (internal), app-beta (PII), app-gamma (internal) |

**Why shared cluster**: three teams, moderate sensitivity, consistent policy
enforcement is easier on one cluster. app-beta (PII) gets a dedicated node
group with taints and stricter NetworkPolicy. If app-beta becomes PCI-scoped
or handles financial data, it moves to a dedicated cluster.

---

## Identity And Access

### Human Access

| Access Pattern | Mechanism | MFA Required | Notes |
| --- | --- | --- | --- |
| Cluster admin (break-glass) | IAM user with `system:masters` group in aws-auth ConfigMap; stored in Secrets Manager | Yes — hardware MFA | Used only during outages when SSO is unavailable. Access is logged via CloudTrail. Reviewed quarterly. |
| Platform engineer (daily ops) | AWS IAM Identity Center (SSO) → IAM role → EKS access entry mapped to `platform-eng` ClusterRole | Yes — SSO enforces MFA | Role allows namespace management, RBAC review, add-on updates. Does not include `cluster-admin`. |
| Application developer (namespace-scoped) | AWS IAM Identity Center → IAM role per team → EKS access entry mapped to namespace-scoped `developer` Role | Yes | Scoped to their namespace only. Can exec into pods, view logs, manage workloads. Cannot read secrets or modify RBAC. |
| Security engineer (read + audit) | AWS IAM Identity Center → IAM role → EKS access entry mapped to `security-auditor` ClusterRole | Yes | Read-only cluster-wide. Can list all resources, read audit logs in CloudWatch. Cannot exec or modify. |
| CI/CD pipeline (GitHub Actions) | GitHub OIDC federation → IAM role assumed via `sts:AssumeRoleWithWebIdentity` → EKS access entry mapped to `ci-deployer` Role per namespace | N/A — non-human | Role is scoped to deployment verbs in the target namespace only. No secret access. Token TTL: 1 hour. |

**OIDC integration:**
- Identity provider: AWS IAM Identity Center connected to corporate Okta
- EKS authenticates humans via `aws eks get-token` using IAM Identity Center session credentials
- RBAC subjects use IAM role ARNs mapped via EKS access entries (not aws-auth ConfigMap, which is deprecated)
- Groups in Okta map to IAM Identity Center permission sets, which map to IAM roles, which map to Kubernetes RBAC

**Why EKS access entries instead of aws-auth ConfigMap:**
- ConfigMap is a single point of failure: corrupt it and you lose cluster access
- Access entries are an AWS API resource — changes are CloudTrail-auditable and cannot be accidentally kubectl-deleted

### Workload Identity — IRSA

All pods that need AWS API access use IRSA (IAM Roles for Service Accounts).

**How it works:**
1. EKS cluster has an OIDC provider registered in IAM
2. Each workload gets a Kubernetes service account annotated with an IAM role ARN
3. The IAM role trust policy allows `sts:AssumeRoleWithWebIdentity` from the cluster's OIDC issuer for that specific service account in that specific namespace
4. The pod receives a projected service account token (TTL: 1 hour) that the AWS SDK exchanges for temporary credentials

**Example annotation:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-secrets
  namespace: platform
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/eks-external-secrets
    eks.amazonaws.com/token-expiration: "3600"
```

**Example IAM trust policy (locks to namespace + service account):**
```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:sub": "system:serviceaccount:platform:external-secrets",
      "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:aud": "sts.amazonaws.com"
    }
  }
}
```

**Common IRSA failure modes and how to debug them:**

| Symptom | Root Cause | Fix |
| --- | --- | --- |
| `WebIdentityErr: failed to retrieve credentials` | OIDC provider not registered in IAM, or URL mismatch | Run `aws iam list-open-id-connect-providers` and compare to cluster OIDC issuer |
| `AccessDenied` when calling AWS API | Trust policy condition does not match the actual namespace or SA name | Check `sub` claim: `aws sts get-caller-identity` from inside the pod should show the role |
| Pod gets no credentials at all | `automountServiceAccountToken: false` or token projection not injected | Verify `eks.amazonaws.com/role-arn` annotation is present and pod has projected volume |
| Credentials expire mid-operation | Default token TTL (86400s) not set; tokens are not refreshed | Set `eks.amazonaws.com/token-expiration: "3600"` and ensure SDK version supports token refresh |

**Governance:**
- All IRSA roles are defined in Terraform; no manual IAM changes are accepted
- Unused IRSA roles are flagged by a weekly Lambda that checks for roles with zero `AssumeRole` events in CloudTrail in 30 days
- IAM role permissions use `Condition` blocks to restrict to specific S3 buckets, Secrets Manager paths, or KMS key ARNs — never `Resource: "*"`

---

## Networking

### VPC And Subnet Layout

```
VPC: 10.0.0.0/16  (us-east-1)

  Private subnets — EKS nodes (one per AZ, no public route):
    10.0.1.0/24  (us-east-1a)
    10.0.2.0/24  (us-east-1b)
    10.0.3.0/24  (us-east-1c)

  Private subnets — EKS pods (VPC CNI custom networking):
    10.0.16.0/20  (us-east-1a)   ~4,000 pod IPs
    10.0.32.0/20  (us-east-1b)
    10.0.48.0/20  (us-east-1c)

  Private subnets — data tier (RDS, ElastiCache):
    10.0.64.0/24  (us-east-1a)
    10.0.65.0/24  (us-east-1b)
    10.0.66.0/24  (us-east-1c)

  No public subnets for nodes or pods.
  NAT gateways in each AZ for outbound-only internet access.
  Internet gateway exists but has no routes to node or pod subnets.

  VPC endpoints (removes internet path for AWS API calls):
    s3, ecr.api, ecr.dkr, sts, ec2, kms, secretsmanager,
    logs, ssm, ssmmessages, ec2messages
```

**Why custom networking for pods:** VPC CNI default mode assigns pod IPs from
the node subnet, consuming IP space that conflicts with large node counts.
Custom networking puts pod IPs in a dedicated subnet, preventing exhaustion.

### Cluster Access

- **API server endpoint**: Private only — no public endpoint
- **Engineer access**: AWS Client VPN (certificate + AWS SSO MFA) to a VPN endpoint in the VPC. Engineers reach the private API server through the VPN tunnel.
- **CI/CD access**: GitHub Actions runners are self-hosted on EC2 instances in the private subnet. They call the private API server directly.
- **Authorized API server security group**: allows 443/TCP from VPN client CIDR and CI runner security group only

### In-Cluster Networking

- **CNI plugin**: Amazon VPC CNI with network policy add-on enabled (uses eBPF via AWS Network Policy Agent)
  - Alternative: Calico in policy-only mode for richer egress controls
- **NetworkPolicy enforcement**: Enabled
- **Default-deny**: Applied to all production namespaces via the namespace template from Lab 11
- **Ingress controller**: AWS Load Balancer Controller (ALB ingress) for external traffic; internal services use Kubernetes Service type ClusterIP only
- **TLS**: cert-manager with AWS ACM Private CA for internal certificates; ACM public certificates for external ALB listeners
- **Service mesh**: Not deployed — NetworkPolicy + mTLS via ALB covers current requirements. Re-evaluate when east-west mTLS between internal services is required.

### Egress Controls

- **Default posture**: Default-deny per namespace (NetworkPolicy), NAT gateway for approved outbound
- **Metadata API**: Restricted — IMDSv2 enforced at node launch (hop limit = 1 blocks pods from reaching instance metadata via the node's IMDS). Pods that legitimately need AWS credentials use IRSA, not IMDS.
- **Egress to AWS services**: Routes through VPC endpoints, never the NAT gateway
- **Egress to internet**: Only the External Secrets Operator (to Secrets Manager endpoint), approved third-party APIs
- **DNS filtering**: CoreDNS with a block list for known C2 domains; all external DNS resolves through Route 53 Resolver with query logging enabled

---

## Secrets And Key Management

### Etcd Encryption

| Control | Status | Notes |
| --- | --- | --- |
| Encryption at rest enabled | Yes | EKS envelope encryption via AWS KMS |
| Encryption key | Customer-managed KMS key (CMK) | Key ARN stored in Terraform state; key policy allows EKS service principal only |
| KMS key rotation | Annual automatic rotation | AWS KMS handles re-wrapping; no manual procedure needed |
| KMS key access | EKS service role + break-glass IAM user only | Separate key policy statement for audit (CloudTrail logs all `Decrypt` calls) |

**What this protects**: an attacker who obtains an etcd snapshot (backup, misconfigured S3 bucket) cannot read secret values without the KMS key. An attacker who compromises a node but not the KMS key gets encrypted blobs only.

**What this does NOT protect**: secrets in memory on the API server, secrets accessible via valid RBAC, or secrets in application logs. Encryption at rest is one layer, not a complete secret security solution.

### Secret Delivery

| Pattern | Used For | Deployed? |
| --- | --- | --- |
| Native Kubernetes secrets | TLS certificates managed by cert-manager, pull secrets for ECR | Yes — cert-manager writes to native secrets |
| External Secrets Operator (ESO) + AWS Secrets Manager | Application secrets (DB passwords, API keys, tokens) | Yes — primary pattern for application secrets |
| CSI Secrets Store Driver + AWS Secrets Manager | Not in use currently | No — ESO covers requirements without CSI complexity |
| Direct SDK calls using IRSA | Read-only access to Secrets Manager from application code | Yes — used by legacy apps not yet migrated to ESO |

**External Secrets Operator setup:**
```yaml
# ESO ClusterSecretStore — one per cluster, scoped to AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: platform
            # IRSA handles the AWS authentication — no static credentials
```

```yaml
# ExternalSecret — one per application secret, lives in the app namespace
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: app-beta
spec:
  refreshInterval: 1h          # ESO syncs from Secrets Manager every hour
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials       # creates a native Kubernetes Secret with this name
    creationPolicy: Owner
  data:
  - secretKey: password
    remoteRef:
      key: prod/app-beta/db    # Secrets Manager path
      property: password
```

- **Rotation**: Secrets Manager rotation is configured on a 30-day schedule for database credentials using a Lambda rotation function. ESO's 1-hour refresh picks up the new value automatically — zero-downtime rotation with no manual steps.
- **Leak detection**: CloudTrail alerts on `GetSecretValue` calls outside the expected IAM principals. Macie scans S3 backups and log archives for secret-shaped strings.

---

## Policy And Admission Control

### Pod Security

- **Pod Security Admission**: Enabled
- **Production namespaces** (`app-alpha`, `app-beta`, `app-gamma`): enforce=restricted, warn=restricted, audit=restricted
- **Platform namespace** (`platform`): enforce=baseline, warn=restricted, audit=restricted — some platform tools need capabilities that Restricted blocks; each exception is documented
- **Infrastructure namespace** (`kube-system`): enforce=privileged — CNI, kube-proxy, and EBS CSI driver require privileged access. These are AWS-managed components; validate on version updates.

### Policy Engine — Kyverno

- **Version**: 1.12
- **Mode**: Enforce for all policies below (audit mode ran for 30 days during rollout)

**Policies in force:**

| Policy | Action | Scope |
| --- | --- | --- |
| disallow-privileged-containers | Enforce | All namespaces except kube-system |
| require-approved-registry | Enforce | All namespaces (approved: `123456789012.dkr.ecr.us-east-1.amazonaws.com`) |
| require-resource-limits | Enforce | All namespaces |
| disallow-host-namespaces | Enforce | All namespaces except kube-system |
| require-signed-images | Enforce | production namespaces only (app-*) |
| restrict-node-port | Enforce | All namespaces (no NodePort services allowed) |
| require-pod-disruption-budget | Warn | Deployments with replicas > 1 |

**Exception process:**
1. Engineer submits a PR to the platform-config repo with a `PolicyException` resource
2. PR requires two approvals: one from the platform team, one from the security team
3. Exception includes: workload name, namespace, justification, approver, review date
4. Exceptions are reviewed quarterly; stale exceptions (past review date) trigger an automated GitHub issue

**Policy violation SLA**: Critical policies (privileged, host namespaces) — remediate within 24 hours. Standard policies (resource limits) — remediate within 7 days.

---

## Observability And Detection

### Log Sources

| Source | Enabled | Destination | Retention |
| --- | --- | --- | --- |
| EKS API server audit log | Yes | CloudWatch Logs: `/aws/eks/prod-cluster/cluster` | 365 days |
| EKS control-plane logs (authenticator, scheduler, controller-manager) | Yes | CloudWatch Logs: `/aws/eks/prod-cluster/cluster` | 365 days |
| Node / kubelet logs | Yes | CloudWatch agent on each node → `/eks/prod-cluster/nodes` | 90 days |
| Container stdout/stderr | Yes | Fluent Bit DaemonSet → CloudWatch Logs per namespace | 90 days |
| VPC flow logs | Yes | S3 bucket → Athena for ad-hoc queries | 365 days |
| Route 53 Resolver query logs | Yes | CloudWatch Logs | 90 days |
| AWS CloudTrail (API calls) | Yes | S3 + CloudWatch Events for real-time alerting | 365 days |

### Alerting

All alerts route to PagerDuty (P1/P2) or Slack (P3/P4) via CloudWatch Alarms + SNS.

| Alert | Severity | On-Call | Metric Filter / Query |
| --- | --- | --- | --- |
| `kubectl exec` in production namespace | P2 | Yes | `filter @message like /pods/exec/ \| filter verb = "create"` |
| Secret read by non-platform principal | P2 | Yes | `filter objectRef.resource = "secrets" AND verb = "get" AND NOT user.username like /system:/` |
| ClusterRoleBinding created or modified | P1 | Yes | `filter objectRef.resource = "clusterrolebindings" AND verb in ["create","update","patch"]` |
| Privileged pod admitted | P1 | Yes | Kyverno audit event in CloudWatch + policy admission log |
| New pod in kube-system not from EKS managed add-ons | P1 | Yes | `filter objectRef.namespace = "kube-system" AND verb = "create" AND objectRef.resource = "pods"` |
| API server auth failure > 10 in 5 min | P2 | Yes | CloudWatch metric math on `403` response codes |
| IMDSv1 call from pod IP | P2 | Yes | VPC flow log: source in pod CIDR, dest 169.254.169.254 |
| IRSA `AssumeRoleWithWebIdentity` from unexpected IP | P2 | Yes | CloudTrail: `AssumeRoleWithWebIdentity` where sourceIPAddress not in [VPC CIDR, GitHub runner CIDR] |

### Runtime Detection — Falco

- **Version**: 0.38 (eBPF driver, not kernel module — no node reboot required)
- **Deployment**: DaemonSet in `falco` namespace, one pod per node
- **Ruleset**: Falco default rules + custom rules below
- **Alert destination**: Falco sidekick → SNS → PagerDuty

**Custom rules in production:**

```yaml
# Alert when a process writes to a binary directory after container startup.
# Indicates post-start modification, which is a common persistence technique.
- rule: Write Binary Dir After Startup
  desc: Detects writes to /usr/bin, /usr/sbin, /bin, /sbin after container startup
  condition: >
    open_write
    and container
    and not container.image.repository in (trusted_images)
    and (fd.name startswith /bin/ or fd.name startswith /usr/bin/
         or fd.name startswith /sbin/ or fd.name startswith /usr/sbin/)
    and proc.aname[2] != "package-manager"
  output: >
    Binary dir write detected (user=%user.name container=%container.name
    image=%container.image.repository:%container.image.tag
    file=%fd.name proc=%proc.name parent=%proc.pname)
  priority: WARNING
  tags: [persistence, container-escape]

# Alert on outbound connections to known crypto mining pool ports.
- rule: Crypto Mining Pool Connection
  desc: Detects outbound TCP connections to common mining pool ports
  condition: >
    outbound
    and container
    and fd.sport in (3333, 4444, 5555, 7777, 8888, 14444, 45560)
  output: >
    Potential crypto mining connection (container=%container.name
    image=%container.image.repository connection=%fd.name)
  priority: CRITICAL
  tags: [cryptomining, impact]

# Alert on execve of shell interpreters inside containers that should not have them.
- rule: Shell Spawned In Container
  desc: Detects a shell being spawned inside a non-debug container
  condition: >
    spawned_process
    and container
    and shell_procs
    and not container.image.repository in (known_shell_images)
    and not proc.pname in (known_shell_parents)
  output: >
    Shell spawned in container (user=%user.name container=%container.name
    image=%container.image.repository shell=%proc.name parent=%proc.pname
    cmdline=%proc.cmdline)
  priority: WARNING
  tags: [execution, t1059]

# Alert on reads of sensitive file paths from within a container.
- rule: Read Sensitive File In Container
  desc: Detects reads of credential files, shadow files, or cloud config inside a container
  condition: >
    open_read
    and container
    and (fd.name in (sensitive_files)
         or fd.name startswith /root/.aws/
         or fd.name startswith /home/.aws/
         or fd.name = /.aws/credentials)
    and not proc.name in (trusted_credential_readers)
  output: >
    Sensitive file read in container (user=%user.name container=%container.name
    image=%container.image.repository file=%fd.name)
  priority: WARNING
  tags: [credential-access, t1552]
```

---

## Add-On Risk Assessment

| Add-On | RBAC Permissions | hostNetwork | Update Model | Risk Notes |
| --- | --- | --- | --- | --- |
| AWS VPC CNI | ClusterRole: nodes/status, pods, namespaces (read); daemonsets (read) | Yes — required for network programming | EKS managed add-on — update via API or Terraform | High privilege DaemonSet; validate each update against release notes for CVEs |
| AWS Load Balancer Controller | ClusterRole with broad read on services, ingresses, nodes; write on ingress status | No | Helm chart in platform-config repo | Watches all ingress objects cluster-wide — RBAC scope is justified by function |
| EBS CSI Driver | ClusterRole: persistentvolumes, persistentvolumeclaims, storageclasses (CRUD) | No | EKS managed add-on | IRSA role scoped to `ec2:CreateVolume`, `ec2:DeleteVolume` in the account; not `ec2:*` |
| Cluster Autoscaler | ClusterRole: nodes (read/write), pods (read/delete) | No | Helm chart, updated with minor Kubernetes upgrades | Can delete pods — verify PodDisruptionBudgets protect critical workloads |
| Kyverno | ClusterRole: all resources (admission webhook must inspect everything) | No | Helm chart | Webhook failure mode is set to `Ignore` for non-critical policies; `Fail` for security policies — prevents a Kyverno outage from blocking all deployments |
| Falco | Privileged DaemonSet, hostPID: true, hostNetwork: true, kernel driver access | Yes | Helm chart | Required for syscall inspection. Scoped via PolicyException. Runs on dedicated security node group. |
| External Secrets Operator | ClusterRole: secrets, externalsecrets, clustersecretstores (CRUD) | No | Helm chart | IRSA role scoped to specific Secrets Manager paths (`prod/*`), not entire account |
| cert-manager | ClusterRole: certificates, certificaterequests, issuers (CRUD), secrets (CRUD in its namespace) | No | Helm chart | Secret write access is necessary for certificate storage; scoped to `cert-manager` namespace |

---

## Incident Response

### Isolation Procedure

```bash
# Step 1: Identify scope
kubectl get all -n <namespace>
kubectl get rolebindings,clusterrolebindings -A | grep <namespace>

# Step 2: Preserve evidence before any deletion
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
kubectl get pod <pod-name> -n <namespace> -o yaml > /tmp/pod-${TIMESTAMP}.yaml
kubectl logs <pod-name> -n <namespace> --all-containers > /tmp/logs-${TIMESTAMP}.txt
kubectl describe pod <pod-name> -n <namespace> > /tmp/describe-${TIMESTAMP}.txt

# Step 3: Isolate network (do not delete pod yet)
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-$(date +%s)
  namespace: <namespace>
spec:
  podSelector:
    matchLabels:
      app: <app-label>
  policyTypes: [Ingress, Egress]
EOF

# Step 4: Query audit logs in CloudWatch
# Replace PRINCIPAL with the service account or user under investigation
aws logs filter-log-events \
  --log-group-name /aws/eks/prod-cluster/cluster \
  --filter-pattern '{ $.user.username = "PRINCIPAL" }' \
  --start-time $(date -d '24 hours ago' +%s000) \
  --output json | jq '.events[].message | fromjson | {time: .requestReceivedTimestamp, verb, resource: .objectRef.resource, name: .objectRef.name}'
```

### Forensics Access

- **Node access**: AWS Systems Manager Session Manager (no SSH, no key management). Logs all commands to CloudWatch. Access requires the `ssm:StartSession` permission on the node's IAM instance profile. Does not require any Kubernetes RBAC.
- **Ephemeral debug containers**: Permitted for security engineers via their ClusterRole. Requires `pods/ephemeralcontainers` verb. Debug images must come from the approved registry.
- **Log retention**: 365 days for audit logs; 90 days for application logs. Both stored in CloudWatch with KMS encryption and an S3 export for long-term archival.

### Credential Rotation Runbook

| Credential Type | Rotation Procedure | Time To Rotate | Automated? |
| --- | --- | --- | --- |
| Kubernetes service account token | Delete and recreate the ServiceAccount; restart affected pods | 10 min | No — manual action |
| Application secret via ESO | Update value in Secrets Manager; ESO syncs within `refreshInterval` (1 hour max) | Up to 1 hour | Partially — ESO automates sync |
| DB password with Secrets Manager rotation | Trigger Lambda rotation function; new value is live in Secrets Manager immediately | 5 min | Yes — Lambda rotation function |
| IRSA temporary credentials | Credentials expire naturally (TTL: 1 hour). To revoke before expiry: add a deny condition to the IAM role trust policy | 5 min for deny; credentials expire within 1 hour | No |
| Break-glass IAM user credentials | Rotate key via `aws iam create-access-key` and `delete-access-key`; update Secrets Manager entry | 15 min | No |
| TLS certificate (internal CA) | Delete cert-manager `Certificate` resource and recreate; cert-manager re-issues immediately | 5 min | No (cert-manager handles renewal automatically before expiry) |

---

## Platform Upgrade And Patch Management

| Item | Current State | Target State | Owner |
| --- | --- | --- | --- |
| Kubernetes minor version | 1.30 | Always within 1 minor version of current EKS latest (1.31 target within 60 days) | Platform team |
| Node OS (Amazon Linux 2023) | AMI updated monthly via node group rolling update | Zero nodes older than 30 days from AMI release | Platform team (automated via Terraform + scheduled Lambda) |
| Container runtime (containerd) | 1.7.x — ships with AL2023 AMI | Current with AL2023 AMI updates | Platform team (comes with OS updates) |
| EKS managed add-on versions | Pinned in Terraform | Updated within 14 days of EKS releasing a new minor version | Platform team |
| Kyverno, Falco, ESO, cert-manager | Pinned Helm chart versions in platform-config repo | Updated within 30 days of upstream release; critical CVE patches within 7 days | Platform team |
| Application base images | Scanned weekly by ECR image scanning | No critical CVEs older than 14 days | Application teams (platform provides alerting) |

**EKS version support window:** AWS supports each EKS minor version for approximately 14 months. We target upgrading before the 12-month mark to avoid emergency upgrades.

**Upgrade procedure:**
1. Terraform plan with new Kubernetes version — review API deprecation warnings
2. Test on a non-production cluster running the same add-on versions
3. Run `kubectl convert` or update manifests for any deprecated APIs
4. Upgrade control plane via EKS API (managed by AWS — control plane nodes are not customer-accessible)
5. Upgrade managed node groups one at a time using a rolling update (max unavailable: 1)
6. Upgrade EKS managed add-ons to versions compatible with the new control plane
7. Validate: run the full policy suite, check Falco alerts, verify all workloads are healthy

---

## Open Risks And Assumptions

| Control | Status | Residual Risk | Remediation Timeline |
| --- | --- | --- | --- |
| Service mesh (east-west mTLS) | Not started | Internal service-to-service traffic is unencrypted inside the cluster; NetworkPolicy provides segmentation but not encryption or per-request identity | Q3 — evaluate Istio ambient mode or Cilium mTLS as lighter alternatives to sidecar injection |
| GitOps (ArgoCD) security hardening | In progress | ArgoCD currently has broad read access to all namespaces; RBAC is not scoped per team | Q2 — implement ArgoCD AppProject isolation per team namespace |
| Runtime detection coverage (Falco) | Partial | Custom Falco rules cover high-priority patterns; kernel module loading and cgroupv2 manipulation are not yet covered | Q2 — add detection rules from the Falco community rules library |
| PII namespace (app-beta) node isolation | Planned | app-beta pods run on shared node groups; a node-level exploit from any pod could affect app-beta data | Q3 — dedicated node group with taint `privacy=pii:NoSchedule`; only app-beta tolerates the taint |
| Break-glass account rotation | Overdue | Break-glass IAM user credentials have not been rotated in 8 months (target: quarterly) | Immediate — rotate this week, set a calendar reminder |

---

## How To Use This Example

When reviewing your own completed architecture against this example:

1. **For every blank in your template**, ask: did you make a deliberate choice, or did you leave it blank because you were unsure?
2. **For every difference from this example**, write one sentence explaining why your choice is better or equivalent for your environment.
3. **Focus on the reasoning**, not the values — there is no single correct reference architecture.
4. **Use the open risks table** as a model for your own: it is more honest and more useful to document what you have NOT done than to pretend the architecture is complete.

The strongest signal of master-level thinking is the ability to explain tradeoffs — not just implement controls, but explain why you chose one approach over an alternative, what it does not protect against, and what you would add next.
