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
