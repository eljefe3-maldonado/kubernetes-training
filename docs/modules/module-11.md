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

---

## Self-Assessment

**Question 1**
Two teams share a cluster. Team A owns `payments` (PCI-scoped). Team B owns
`analytics` (no compliance requirements). A compromise of Team B's pod leads
to exfiltration of Team A's payment processor API key. What architectural
decision enabled this, and what is the correct fix?

*A strong answer covers:* the enabling decision was running PCI-scoped and
non-PCI workloads in the same cluster without effective isolation between
namespaces. Namespace separation alone does not prevent cross-namespace access
if Team B's service account has a ClusterRoleBinding, if NetworkPolicy is
absent, or if secrets in both namespaces are accessible through a shared
controller. The correct fix for PCI-scoped workloads is a dedicated cluster
or dedicated cloud account — not namespace isolation. The compliance cost
of proving isolation on a shared cluster typically exceeds the operational
cost of a dedicated cluster. Notes that this is the tenancy decision framework
question: can a compromise of A expose B's data, and is that consequence
acceptable?

---

**Question 2**
Your platform has documented security requirements in a wiki. A new engineer
joins and deploys a workload with a broad ClusterRole because she followed
a tutorial. How did your governance model fail, and what would prevent this?

*A strong answer covers:* documentation-only governance fails because it
relies on every engineer reading, understanding, and correctly applying
requirements under time pressure. The new engineer did not know about the
wiki, or did not understand why the ClusterRole was problematic. The fix:
enforceable governance. The namespace provisioning process should not allow
engineers to create ClusterRoleBindings — that verb should be reserved for
the platform team. Admission policy should block ClusterRoleBindings that
reference overly broad roles. The RBAC model should give engineers access
to create namespace-scoped Roles only. The goal: the secure path is the
default path; the insecure action requires extra steps and platform review.

---

**Question 3**
A product team requests a dedicated namespace for their new service. What
baseline controls should be applied automatically to every new namespace,
and who is responsible for applying them?

*A strong answer covers:* the platform team is responsible for automated
namespace provisioning that applies: (1) ResourceQuota — prevents one team
from consuming all cluster resources; (2) LimitRange — sets default resource
limits on pods that don't specify them; (3) default-deny NetworkPolicy —
blocks all ingress and egress until the team adds explicit allow rules for
their traffic flows; (4) Pod Security Admission label — enforces the Restricted
profile for workloads in the namespace; (5) RBAC — namespace-scoped Role
granting the team admin access to their namespace only, no ClusterRoleBindings.
These should be applied via automation (Helm chart, Terraform, Kyverno
generate policy) rather than manual steps that may be forgotten.

---

**Question 4**
A compliance audit asks you to demonstrate that the `payments` namespace has
network isolation from all other namespaces. You have NetworkPolicy applied.
What else must you verify before you can make this claim?

*A strong answer covers:* NetworkPolicy enforcement requires a CNI that
supports it — verify Calico/Cilium/AWS VPC CNI with network policy add-on
is in use (not Flannel). Verify no pods in `payments` run with `hostNetwork:
true` (which bypasses NetworkPolicy). Verify there are no ClusterRoleBindings
that give other namespace service accounts access to `payments` secrets
(RBAC isolation). Verify the admission policy blocks privileged pods and
hostNetwork pods from being deployed in `payments`. The claim "we have
NetworkPolicy" is not sufficient — network isolation requires all four:
CNI enforcement, no hostNetwork exceptions, RBAC scoping, and admission
policy preventing bypasses.

---

**Question 5**
The platform team is reviewing a request from a product engineer who wants
to create a CronJob that runs `kubectl` to list pods across all namespaces
as a "health check." How do you evaluate this, and what is the correct
response?

*A strong answer covers:* this request is for a ClusterRole with `list pods`
cluster-wide, bound to a service account running a CronJob. Evaluate: (1) is
this a genuine operational need or can the health check be done with metrics/
probes instead? (2) if it is legitimate, scope it to `list pods` only with
a ClusterRoleBinding rather than broader permissions — do not grant `list
secrets` or other resources. (3) verify the CronJob image is from the approved
registry and signed. (4) document the exception with a review date. The broader
concern: a CronJob running `kubectl` is a credential and process that can be
exploited. If the job's RBAC or the cluster credential it uses is compromised,
it is a cluster-wide reconnaissance tool. Platform team should provide a
supported health check mechanism rather than individual teams running kubectl.

---

**Question 6**
You are building the security responsibility matrix for a new multi-team
platform. List at least five security controls and assign each to platform
team, security team, or product team. Explain why each assignment is correct.

*A strong answer covers:* Platform team owns: admission policy baseline
(they configure the enforcement mechanism), namespace provisioning automation
(they create the security baseline), add-on management and upgrades (they
control what runs at cluster level), Kubernetes version and node OS upgrades
(cluster infrastructure). Security team owns: the security baseline
requirements (what the platform team must enforce), exception review and
approval process, detection rules and alert thresholds, compliance evidence
collection, incident response escalation paths. Product team owns: service
account design for their workloads, secret usage within their namespace,
NetworkPolicy allow rules for their specific traffic flows, ensuring their
application code does not log secrets. The clarity test: can every control
be traced to a single owner who can be held accountable?
