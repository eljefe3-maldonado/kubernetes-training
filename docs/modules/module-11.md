# Module 11: Governance And Multi-Tenancy

## Why This Matters

Technical controls work when they are applied consistently. They break down
when the organizational model does not support consistent application —
when teams bypass controls because they do not understand them, when
exceptions accumulate without review, when responsibilities are unclear,
or when the platform and security teams operate in isolation from the
engineering teams they are supposed to protect. This module is about building
a governance model that makes security the path of least resistance for
engineering teams, not an obstacle they route around.

---

## The Attack Narrative

Two product teams share a Kubernetes cluster. Team Alpha owns a `payments`
namespace. Team Beta owns a `reporting` namespace. The company has documented
security policies but no enforcement: namespaces are created by filling out
a Jira ticket, and the platform team creates them with no baseline controls.

Team Beta deploys a new reporting service. The engineer follows a tutorial
that uses a broad ClusterRole for the service account — "it was easier than
figuring out the minimum permissions." She deploys to the `reporting` namespace
with a ClusterRoleBinding.

Two months later, a supply chain compromise hits Team Beta's reporting service
through a malicious npm package. The attacker has RCE in the reporting pod.
Because of the ClusterRoleBinding, the service account can list secrets
cluster-wide. Within minutes the attacker reads the payments namespace secrets,
which include the payment processor API key and the database encryption key.

Team Alpha's payments system, which had no vulnerabilities, is compromised
through Team Beta's security gap.

This is the multi-tenant problem: in a shared cluster, one team's security
posture affects every other team. The attack surface of the payments system
includes every other namespace in the cluster, not just the payments code itself.

---

## Why Namespaces Are Not Isolation Boundaries

This is the most important concept in multi-tenancy and the one that is most
frequently misunderstood. A namespace is an organizational and resource-
scoping boundary. It is not a security boundary in the way that a VM boundary
or a network boundary is.

What namespaces provide:
- Scope for RBAC — a Role in namespace A does not apply in namespace B
- Scope for resource quotas and limit ranges
- A unit for NetworkPolicy rules
- A naming scope — resources with the same name can coexist in different
  namespaces

What namespaces do not provide:
- Network isolation — pods in different namespaces can communicate freely
  unless NetworkPolicy prevents it
- RBAC isolation — a ClusterRoleBinding applies regardless of namespace
- Node isolation — pods from different namespaces run on the same nodes
- Kernel isolation — pods in different namespaces share the same kernel

A namespace boundary prevents accidental interference. It does not prevent
intentional lateral movement by an attacker who has compromised one workload.
The attacker's constraints are determined by what the compromised pod's
service account can do (RBAC) and what the compromised pod can reach on the
network (NetworkPolicy) — not by the namespace it runs in.

This has a direct implication for tenancy decisions: if two tenants have
materially different data sensitivity or compliance requirements, putting them
in separate namespaces on the same cluster is not sufficient isolation on
its own. Separate namespaces with strict NetworkPolicy and least-privilege
RBAC reviewed to prevent cross-namespace access can be sufficient for
moderate-sensitivity cases. For high-sensitivity workloads — PCI scope,
regulated data, cross-customer data — separate clusters or separate cloud
accounts are the correct answer.

---

## The Governance Gap: Standards Without Enforcement

Most organizations have Kubernetes security standards. The standards are
written down somewhere — a wiki, a security policy document, a README in
the platform repository. The gap is between the standard existing and the
standard being applied to every new namespace, every new workload, every
Helm chart deployed by every team.

Documentation-only governance relies on every engineer reading the docs,
understanding them, and applying them correctly under time pressure, while
onboarding, and for every deployment they ever make. This does not work.
Not because engineers are careless, but because humans are not reliable
enforcement mechanisms for rules that apply to every action in a complex
system.

Enforceable governance works differently. The namespace provisioning process
automatically applies a security baseline: ResourceQuotas, LimitRanges,
default-deny NetworkPolicy, Pod Security Admission labels. The admission
policy engine ensures that no workload can be deployed in a production
namespace without meeting the minimum security requirements. The RBAC model
gives each team access to their namespace and nothing else. These controls
apply whether or not the engineer knows they exist, whether or not the team
has read the security wiki, and whether or not the deployment happens at 2am.

The principle: make the secure path the default path. The engineer who
follows the normal deployment process should automatically land on a secure
configuration. The insecure configuration should require extra steps and
explicit exceptions.

---

## The Tenancy Decision Framework

When deciding whether two workloads can safely share a cluster, three
questions determine the answer:

**Can a compromise of workload A expose workload B's data or credentials?**
If yes and the consequence is unacceptable, they cannot share a cluster.
No combination of namespace isolation, NetworkPolicy, and RBAC is sufficient
if the consequence of cross-tenant compromise is severe enough — for example,
if workload B is PCI-scoped and a compromise would require breach notification.

**Do the two workloads have materially different compliance requirements?**
A PCI-scoped workload has specific requirements about network segmentation
that are difficult to satisfy with namespace isolation on a shared cluster
where a control plane and nodes are shared with non-PCI workloads. The
compliance cost of proving isolation on a shared cluster often exceeds
the operational cost of a dedicated cluster.

**Can both workloads be governed by the same security baseline?**
If one workload needs a privileged DaemonSet, special node access, or
admission policy exceptions that would not be acceptable for the other
workload's data sensitivity, they should not share a cluster. Exceptions
applied at the cluster level apply to all tenants.

The rule of thumb: start with separate namespaces and enforce hard controls
(NetworkPolicy, RBAC, PSS Restricted). Move to separate clusters when
compliance requires it or when cross-tenant blast radius is unacceptable.
Move to separate cloud accounts when the regulatory requirement is for
network-level isolation that cannot be achieved within a single VPC.

---

## Platform Security Operating Model

Governance requires clear ownership. The three parties in most platform
organizations have different security responsibilities:

**The platform team** owns: cluster provisioning and hardening, admission
policy baseline, namespace provisioning automation, add-on management,
Kubernetes version upgrades, node patching. They create and maintain the
guardrails. They do not own application security decisions within a namespace.

**The security team** owns: the security baseline requirements, exception
review and approval, detection and response, compliance evidence, and
escalation paths. They define what the guardrails must be. They do not
provision namespaces or manage cluster infrastructure day-to-day.

**The product team** owns: workload design within the namespace, application
RBAC (service accounts for their workloads), secret management patterns
their applications use, and ensuring their workloads comply with the baseline.
They do not own cluster-level controls or cross-namespace access.

The clearest sign of a poorly governed platform is when these boundaries
are unclear: when product teams are creating ClusterRoleBindings because
the platform team has not provided a self-service RBAC model, or when the
security team is reviewing every Deployment because the admission policy
engine is not in place to automate that check.

---

## What Good Looks Like vs What Compliant Looks Like

A compliant platform has namespaces created on request and some documentation
about security requirements. A governed platform has an automated namespace
provisioning workflow that applies a security baseline on creation, admission
policies that enforce the baseline on every deployment, RBAC models that scope
each team to their namespace, and a clear responsibility model so that every
security requirement has an owner who can be held accountable for it.

The test for a well-governed platform: if a new engineer joins a product team
today and follows the standard deployment process without reading the security
documentation, does their workload meet the security baseline? If yes, the
governance is working. If no, the governance exists on paper.

---

## You Will Learn

- why namespaces are organizational boundaries, not security boundaries
- the tenancy decision framework for choosing between shared and dedicated clusters
- how to design a namespace provisioning process that enforces a security baseline
- what the platform team, security team, and product team each own
- how to build governance that works when engineers do not read the documentation

## Key Questions

- When is a shared cluster acceptable and when is dedicated isolation required?
- What controls need to be applied consistently to every namespace, and who owns enforcing them?
- How do you prevent one team's security gap from becoming another team's compromise?
- What does an enforceable governance model look like versus a documented one?

## Hands-On

- Lab: [Lab 11](../../labs/module-11/README.md)
- Asset: [secure-namespace.yaml](../../labs/module-11/secure-namespace.yaml)

## Output

Produce a governance model for a multi-team Kubernetes platform, including a
tenancy recommendation for three teams with different sensitivity levels and
a responsibility matrix for security controls.
