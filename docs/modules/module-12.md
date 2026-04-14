# Module 12: Cloud Provider Mapping And Security Leadership

## Why This Matters

Kubernetes security knowledge that cannot be translated into cloud-specific
decisions and communicated to leadership has limited operational impact.
Every major cloud provider has a different model for identity federation,
node management, secrets integration, and audit logging. Getting these details
wrong creates gaps that generic Kubernetes security knowledge cannot fill.
And communicating security risk in terms that lead to resource allocation
and remediation requires a different skill set than knowing the technical
controls — it requires the ability to explain risk in business terms without
losing the precision that makes the technical analysis meaningful.

---

## The Attack Narrative — A Cloud-Specific Gap

A team migrates their workload from a self-managed cluster to EKS. They are
experienced Kubernetes operators. They correctly configure Pod Security
Standards, NetworkPolicy, Kyverno policies, and Falco. Their Kubernetes
security posture is strong.

What they miss: on their self-managed cluster, pods needed explicit credentials
to access any cloud API. On EKS, the EC2 nodes have an IAM instance role.
Every pod on a node can reach the instance metadata service at
`169.254.169.254` and obtain the node's IAM credentials — unless IMDSv2
with a hop limit of 1 is configured, or IRSA is used exclusively and metadata
API access is blocked by NetworkPolicy.

The node's IAM role has broad permissions — it was created by a script the
team found online that included `ec2:*` and `s3:*` for convenience. A
compromised pod, despite running in a hardened security context with all the
Kubernetes-level controls in place, can obtain cloud credentials with
administrator-equivalent access to S3 and can read from or write to any
bucket in the account.

The same security engineer who designed a strong Kubernetes posture missed
the EKS-specific identity model. This is the gap this module closes: the
controls that are Kubernetes-generic work the same everywhere, but the
controls that are cloud-specific require knowing the cloud.

---

## EKS, GKE, And AKS: The Security-Relevant Differences

Cloud providers all run Kubernetes, but their security models differ in ways
that matter operationally. These are the differences worth knowing, not the
feature comparison tables.

**Workload identity** is the most important cloud-specific security mechanism
in each platform, and each implements it differently.

EKS uses IRSA (IAM Roles for Service Accounts). The cluster has an OIDC
provider registered in AWS IAM. A Kubernetes service account is annotated
with an IAM role ARN. The kubelet injects a projected service account token
into the pod, which the AWS SDK exchanges for temporary STS credentials.
The IAM role's trust policy must explicitly name the OIDC issuer, namespace,
and service account. A misconfigured trust policy — particularly one that
uses a wildcard subject condition — can allow any service account in any
namespace to assume the role.

GKE uses Workload Identity, which binds a Kubernetes service account to a
GCP service account. The Kubernetes service account must be annotated with
the GCP service account's email, and the GCP service account must have the
Workload Identity User role granted to the Kubernetes service account in the
correct namespace. The security model is similar to IRSA but uses GCP's
IAM primitives. A key difference: GKE Autopilot enforces Workload Identity
and disables the metadata server for pods by default, which is a stronger
default than EKS.

AKS uses Azure AD Workload Identity, which uses OpenID Connect federation
similarly to IRSA. The managed identity and federated credential must both
be correctly configured. AKS also has managed identity options at the node
pool level, which creates the same risk as EKS instance roles if managed
identities are over-permissioned.

**etcd encryption** defaults differ. All three providers encrypt etcd data
at rest with provider-managed keys. Customer-managed key encryption (CMEK)
is opt-in on all three. On EKS, envelope encryption must be explicitly enabled
at cluster creation time with a KMS key ARN — it cannot be added after creation
without creating a new cluster. On GKE, CMEK is a cluster-level setting that
can be added after creation. On AKS, CMK is available but requires a specific
cluster configuration.

**Audit log delivery** varies in default configuration. GKE enables audit
logs by default and routes them to Cloud Logging. EKS requires explicitly
enabling each log type (API, audit, authenticator, scheduler, controller
manager) via the EKS API — they are not enabled by default. AKS requires
enabling the `kube-audit` and `kube-audit-admin` diagnostic setting on the
cluster to route audit logs to a Log Analytics workspace or storage account.
On all three, audit logs for control-plane events require explicit opt-in
or configuration to reach a destination you control.

**Managed add-on security** is where managed clusters create hidden risk.
Cloud providers maintain managed add-ons (CNI, kube-proxy, CoreDNS, metrics
server) and push updates on their own schedule. These add-ons run with
cluster-level RBAC and in some cases with hostNetwork access. Reviewing
the permissions and update model for every managed add-on is a security
responsibility that belongs to the customer, even though the provider manages
the add-on.

---

## The Misconception: Managed Means Secure

"We use EKS/GKE/AKS so they handle the security" is a statement that is
partially true and more often dangerously false. Managed Kubernetes providers
handle the security of the control plane — the API server, etcd, scheduler.
They do not handle the security of what runs on that control plane.

The shared responsibility model for managed Kubernetes is roughly:

The provider owns: control-plane availability and patching, etcd management,
physical hardware security, network connectivity between control-plane
components, and the security of the managed add-ons they deploy.

The customer owns: node OS patching (in managed node groups, the provider
updates the base AMI but the customer must trigger the rolling update),
kubelet configuration, RBAC model, workload security contexts, NetworkPolicy,
admission policies, audit log configuration and retention, secrets encryption
key management for customer-managed keys, IAM/workload identity configuration,
and the security of all workloads running in the cluster.

A useful test: could an attacker compromise your cluster through a weakness
in the things the provider owns? Almost never — the control plane is well
protected. Could they compromise it through a weakness in the things the
customer owns? Yes, routinely. The RBAC model, the workload identity
configuration, the node IAM permissions, the audit log absence — these are
customer responsibilities where real incidents happen.

---

## Communicating Security Risk To Leadership

The technical depth in this program is only valuable if it leads to decisions
that improve security. Those decisions — resourcing, prioritization, risk
acceptance — are made by engineering leadership, not security engineers. The
ability to communicate security risk in terms that lead to action is as
important as the technical knowledge.

The principles for effective security communication to leadership:

**Lead with business impact, not technical mechanism.** "Our cluster has no
audit logging" means nothing to an engineering director. "If we have a breach
today, we cannot determine what data was accessed, cannot prove scope to
regulators, and our forensic investigation would start from zero" gives them
the information to make a decision.

**Be specific about what "risk" means in this context.** Risk is not an
abstract score. For each gap, explain: who is the likely attacker, what
specific action would they take, and what is the worst-case outcome if this
gap is exploited? "A compromised workload could access cloud credentials and
exfiltrate customer data from S3" is actionable. "Our security posture has
room for improvement" is not.

**Separate findings by effort and impact.** Not every gap requires the same
investment to close. Present a prioritized list where quick wins are clearly
separated from architectural changes. Leadership can approve a quick-win
list in a week. An architectural change requires planning cycles.

**Quantify what you can, acknowledge what you cannot.** Some risks can be
quantified — the blast radius of a specific credential compromise, the number
of workloads that would be affected by a specific misconfiguration. Others
cannot — the probability of being targeted, the sophistication of the
attacker. Be honest about what is quantifiable and what is a judgment.

---

## What Good Looks Like vs What Compliant Looks Like

A compliant cloud Kubernetes deployment meets the provider's well-architected
framework checklist. A secure one has: workload identity configured for every
pod that needs cloud access (no instance-role credential sharing), node IAM
roles scoped to the minimum permissions nodes need for cluster operation,
etcd encryption with customer-managed keys, audit logs configured and retained
in a destination that is access-controlled and not on the cluster itself, and
a clear picture of what the provider owns versus what the customer owns —
with verification that the customer-owned controls are actually in place.

The executive security review is not a presentation of everything that is
correct. It is a honest account of the top risks, their business impact, and
a prioritized plan to address them. A security engineer who can produce that
review for their cluster — and who can update it as the cluster evolves —
is doing master-level work.

---

## You Will Learn

- the security-relevant differences between EKS, GKE, and AKS workload identity models
- how the shared responsibility model divides security ownership in managed Kubernetes
- which cloud-native integrations materially improve security versus which are convenience features
- how to communicate Kubernetes security risk to engineering leadership in decision-useful terms
- how to build a phased security roadmap that accounts for effort, impact, and organizational capacity

## Key Questions

- Which cloud-specific controls does your platform require that generic Kubernetes security does not cover?
- What does the provider own versus the customer own, and how do you verify the customer-owned controls?
- How would you explain the top three risks in your cluster to an engineering director in five minutes?
- What is the difference between a compliant cloud Kubernetes deployment and a secure one?

## Hands-On

- Lab: [Lab 12](../../labs/module-12/README.md)
- Assets:
  - [reference-architecture.md](../../labs/module-12/reference-architecture.md)
  - [eks-reference-architecture-example.md](../../labs/module-12/eks-reference-architecture-example.md)

## Output

Produce a secure managed-platform reference architecture for your chosen
cloud provider and a two-page executive security review covering current
risk, target state, and a phased remediation roadmap.

---

## Self-Assessment

**Question 1**
A team migrates from a self-managed cluster to EKS. Their Kubernetes security
posture is strong: Pod Security Standards enforced, NetworkPolicy in place,
Kyverno policies active. Two months after migration, a compromised pod accesses
S3 buckets containing customer backups. What did they miss, and how do you
explain this to the team?

*A strong answer covers:* they missed the EKS-specific identity model. On
EC2-backed EKS nodes, every pod can reach the instance metadata API at
`169.254.169.254` and obtain the node's IAM instance role credentials —
unless IMDSv2 with hop limit 1 is configured, or NetworkPolicy blocks access
to that address. The node's IAM role had `s3:GetObject` on the backup bucket.
The Kubernetes-layer controls (PSS, NetworkPolicy, Kyverno) are Kubernetes-
generic and do not address cloud identity. Explanation: moving to a managed
cloud platform adds a cloud-specific identity layer that requires cloud-specific
controls (IRSA instead of instance roles, IMDSv2 hop-limit enforcement,
NetworkPolicy blocking 169.254.169.254 for application pods).

---

**Question 2**
You are reviewing an IRSA trust policy for an EKS service account. The
policy condition is: `"StringLike": {"oidc.eks.us-east-1.amazonaws.com/id/EXAMPLE:sub": "system:serviceaccount:*:*"}`.
What is the security problem?

*A strong answer covers:* the wildcard condition `system:serviceaccount:*:*`
allows any service account in any namespace to assume this IAM role. The
intent of IRSA is to bind a specific Kubernetes service account in a specific
namespace to a specific IAM role. The correct condition is:
`"StringEquals": {"oidc.eks.../id/EXAMPLE:sub": "system:serviceaccount:NAMESPACE:SERVICE_ACCOUNT_NAME"}`.
Using `StringLike` with wildcards means that any compromised pod with any
service account can assume this role by exploiting the trust policy. This
is a critical misconfiguration that grants the IAM role's permissions to
the entire cluster effectively.

---

**Question 3**
A developer at your company says: "We use GKE so Google handles the security."
What specifically does Google handle, and what are three concrete examples of
security responsibilities that remain with your team?

*A strong answer covers:* Google owns the control-plane infrastructure —
API server, etcd, scheduler, controller manager. They patch and upgrade these
components and handle physical security. Customer owns: (1) workload identity
configuration — if GKE Workload Identity is not configured and pods use the
node's default service account, they can access GCP APIs with the node's
permissions; (2) RBAC model — Google does not prevent over-privileged service
accounts; (3) audit log configuration — GKE enables audit logs by default
to Cloud Logging, but the customer must configure retention, export to SIEM,
and write alert rules; (4) node OS patching — in standard node pools, the
customer must trigger rolling updates when Google releases new node images.

---

**Question 4**
An engineering director asks you to explain the top three security risks in
your EKS cluster in five minutes. Your last security review found: no etcd
encryption, overprivileged node IAM roles with `s3:*` and `ec2:*`, and audit
logging disabled. How do you present this?

*A strong answer covers:* lead with business impact, not technical mechanism.
(1) "If an attacker compromises any pod in the cluster, they can access every
S3 bucket in our AWS account because each node has administrator-level S3
permissions. Fix is scoping node IAM roles to what nodes actually need —
low effort, high impact." (2) "If anyone accesses our etcd backup or the
etcd storage directly, every secret in the cluster is readable as plaintext.
We can enable encryption at rest with a customer-managed key — note that on
EKS this requires cluster recreation if not done at creation time." (3)
"If there is an active compromise today, we cannot investigate it — audit
logging is not configured. We have no record of any API calls ever made to
this cluster. Enabling this is a same-day action." Structure: what is the
risk in plain language, who the likely attacker is, worst-case outcome,
and the effort to fix.

---

**Question 5**
After a security review you produce a remediation roadmap. Leadership approves
the quick wins but pushes back on the architectural changes (etcd encryption
requiring cluster recreation, IRSA migration across 40 workloads). How do you
respond?

*A strong answer covers:* acknowledge the effort and agree on compensating
controls for the interim period. For etcd encryption: verify that etcd backup
access controls are as strict as possible (no world-readable S3 buckets, CMK
on the S3 bucket itself), and add monitoring for unexpected etcd backup access.
For IRSA migration: prioritize the highest-risk workloads (those whose pods
have access to the most sensitive data or external systems), migrate those
first, and maintain the node IAM role migration as a phased project with a
deadline. Frame both as: "we accept the residual risk for now, these are the
compensating controls, and here is the date we commit to completing the
migration." Document the risk acceptance with an approver, a scope, and a
review date.

---

**Question 6**
You are onboarding a new workload that needs to write to an S3 bucket and
read from AWS Secrets Manager. Design the correct workload identity
configuration on EKS, and explain each component.

*A strong answer covers:* (1) create an IAM role with a trust policy that
names the specific EKS cluster's OIDC issuer, the specific namespace, and
the specific Kubernetes service account name — no wildcards in the subject
condition; (2) attach an IAM policy to the role with only the needed
permissions: `s3:PutObject` on the specific bucket ARN (not `s3:*`),
`secretsmanager:GetSecretValue` on the specific secret ARN; (3) create a
Kubernetes service account in the correct namespace annotated with
`eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/ROLE-NAME`; (4) set
`automountServiceAccountToken: true` on the pod (required for IRSA — the
projected service account token is the credential the AWS SDK exchanges for
STS credentials); (5) use the AWS SDK in the application, which will
automatically detect and use the projected token. Verify the trust policy
is not too broad and that the IAM policy is scoped to the specific resources
the workload needs.
