# Module 01: Architecture And Attack Surface

## Why This Matters

Security engineering in Kubernetes starts with knowing which components hold
authority, which paths carry trust, and how a single foothold becomes broader
cluster impact. You cannot defend a system you cannot draw. Before you write
a single policy or review a single RBAC binding, you need to be able to walk
through a cluster's architecture and explain, component by component, what an
attacker gains by owning each one.

---

## The Attack Narrative

A developer at a mid-sized company pushes a config change late on a Friday.
She copies a kubeconfig from her workstation to a temporary directory, runs
a script, and forgets to clean up. Three weeks later a routine secret scanning
job finds the file committed to a public GitHub repository. The kubeconfig has
cluster-admin privileges on the production cluster.

By the time the security team is notified, the attacker has already used it.
The first thing they did was not deploy malware. They ran `kubectl get pods -A`
and `kubectl get secrets -A`. In sixty seconds they had a complete inventory
of every running workload and every secret in the cluster. They picked one
secret — a database credential — and extracted it. Then they created a new
service account with cluster-admin and deleted the old ClusterRoleBinding so
their activity would look like a cleanup rather than an intrusion.

What made this possible was not the leaked kubeconfig alone. It was the
architecture: a single API server that holds authority over everything, a
ClusterRoleBinding that granted unlimited access, and no detection on unusual
secret reads. The attacker needed one credential and found one credential.
They never needed to exploit a vulnerability.

This is the most common shape of a Kubernetes incident. Not a zero-day.
Not a sophisticated exploit chain. A stolen credential, an over-privileged
binding, and no one watching.

---

## How The Cluster Architecture Creates Risk

Every Kubernetes cluster has the same fundamental trust hierarchy. The API
server is the central authority. Every action — creating a pod, reading a
secret, modifying RBAC — goes through it. This centralization is a feature
for operations and a significant risk concentration for security.

**The API server** is the highest-value target in any cluster. Whoever can
authenticate to it with sufficient privileges can do anything: read all
secrets, create privileged pods, modify RBAC, delete workloads, or exfiltrate
data. The API server trusts credentials from several sources: x509 certificates
(used by kubelets and control-plane components), bearer tokens (service accounts
and OIDC), and in some configurations static passwords or bootstrap tokens.
Each of these authentication mechanisms is a potential attack path.

**etcd** is the second-highest-value target. It stores the entire cluster state,
including every secret in base64-encoded plaintext unless envelope encryption
is configured. An attacker who compromises etcd directly — for example, by
exploiting a misconfigured backup job that writes etcd snapshots to a public
S3 bucket — can extract every secret without ever authenticating to the API
server. Many organizations configure the API server endpoint carefully but
forget that etcd backups are stored with weaker access controls.

**The kubelet** runs on every node and is the component that actually starts
and stops containers. It has an API of its own (port 10250) that allows
reading pod logs, executing commands in running containers, and in older
configurations proxying to the API server. A kubelet API that accepts
unauthenticated requests is equivalent to `kubectl exec` access to every pod
on that node. Even with authentication required, a compromised kubelet
credential gives an attacker access to all pods running on that node.

**Namespaces** are the boundary that learners most often overestimate. A
namespace is an organizational boundary, not a security boundary. Pods in
different namespaces run on the same nodes, share the same network unless
NetworkPolicy explicitly isolates them, and can communicate freely unless
restricted. A service account in namespace A with a ClusterRoleBinding can
act cluster-wide regardless of which namespace it lives in. Understanding
this is foundational: when you see "we put it in a separate namespace for
security," you should immediately ask what RBAC, NetworkPolicy, and Pod
Security controls back that claim up.

---

## The Components Worth Mapping

When you draw a trust map of a cluster, every communication path has three
properties that matter for security: the authentication mechanism, whether
the channel is encrypted in transit, and the blast radius if the credential
is stolen.

The path from the API server to etcd uses client certificates. If an attacker
obtains the API server's etcd client certificate, they have unrestricted read
and write access to the entire cluster state. This certificate typically lives
on the control-plane node. Node access is how this credential is stolen.

The path from the kubelet to the API server uses a bootstrap credential that
is rotated to a node-specific certificate. If an attacker compromises a node,
they get a credential that allows the API server's Node authorizer to approve
requests, but only on behalf of that specific node. This is more limited than
cluster-admin, but it is enough to read secrets mounted into pods on that node.

The path from a pod to the API server uses a service account token mounted as
a projected volume. This token is namespaced and has whatever RBAC the service
account has been granted. The risk here is not the mechanism — projected tokens
with short TTLs are reasonably safe — it is the RBAC attached to the token.
A default service account with no bindings can do almost nothing. A CI service
account with cluster-admin can do everything.

---

## The Misconception: Private Clusters Are Safe

The most common mistake when mapping Kubernetes architecture is to focus on
the perimeter and stop there. Teams that run private clusters — no public API
endpoint, nodes in private subnets — often believe the hard work is done.
It is not.

Perimeter controls protect against external attackers who have no presence in
the environment. They do not protect against an attacker who compromises a
workload inside the cluster, steals a service account token from a pod, finds
a leaked credential in a CI log, or gains access through an employee's machine.
Most real incidents start from inside the network boundary, not outside it.

A private cluster with no NetworkPolicy, no RBAC review, and no audit logging
is less secure than a public cluster that has those controls. The perimeter
reduces the attack surface from external threats. It does nothing for the
threat paths that matter most.

---

## Decision Framework: Classifying Components By Security Impact

When you encounter a new cluster, classify every component before you evaluate
controls. Use three questions:

**What does an attacker gain by owning this component?**
For the API server: everything. For a single pod: the pod's service account
privileges and whatever it can reach on the network. For a node: all pods on
that node and the kubelet credential. The answer tells you how hard to defend
and monitor each component.

**What credential authenticates to this component?**
If the credential is a long-lived static token, a certificate with no revocation
path, or a kubeconfig that has never been rotated, the risk is higher. If the
credential is short-lived, scoped to a specific principal, and produces audit
log entries when used, the risk is lower.

**What would an attacker do immediately after gaining access?**
After owning the API server, they read all secrets and look for IAM credentials.
After owning a node, they look for service account tokens in mounted volumes.
After owning a pod, they look for what the service account token allows and
what the network lets them reach. Walking this chain is how you identify
which controls would break the attacker's path earliest.

---

## What Good Looks Like vs What Compliant Looks Like

A compliant cluster might pass a CIS Kubernetes Benchmark scan. It has TLS
everywhere, audit logging enabled, and RBAC in place. A secure cluster has
all of that and also: a reviewed RBAC model where every binding is justified,
a network topology that limits blast radius if any workload is compromised,
detection on the events that would indicate credential theft or privilege
escalation, and an incident response plan that does not start with "first,
we delete the suspicious pod."

The benchmark tells you what settings to check. It does not tell you whether
the RBAC model is actually least-privilege, whether the audit logs are being
reviewed, or whether the team knows what to do when an alert fires. Those
require judgment, not configuration.

---

## You Will Learn

- control-plane trust boundaries and why the API server is the central risk point
- high-value components — API server, etcd, kubelet, ingress — and what losing each means
- how namespace isolation fails when not backed by RBAC and NetworkPolicy
- the difference between perimeter security and defense-in-depth
- common workload-to-cluster abuse paths and where to break them

## Key Questions

- What component is the attacker trying to reach next, and what credential gets them there?
- Which communication paths are the strongest detection opportunities?
- Which compromises change impact from namespace scope to cluster scope?
- Where does your cluster assume trust that an attacker could exploit?

## Hands-On

- Lab: [Lab 01](../../labs/module-01/README.md)
- Asset: [trust-map-template.md](../../labs/module-01/trust-map-template.md)

## Output

Produce a trust map with annotated abuse paths and explain the shortest route
from a compromised workload to cluster-wide impact.

---

## Self-Assessment

Write your answer before reading the guidance. A strong answer is specific —
it names components, explains the mechanism, and identifies the attacker's
next step. A weak answer stays general.

**Question 1**
You have read access to an unfamiliar production cluster. You have 20 minutes
to find the single highest-risk issue. What do you look at first, in what
order, and why?

*A strong answer covers:* starting with `kubectl get clusterrolebindings` to
find over-privileged principals, then `kubectl get pods -A` for unexpected
workloads in `kube-system`, then checking whether the API server endpoint is
public. Explains that RBAC is the highest-leverage finding because it determines
blast radius for every other issue. Does not start with CVE scanners or workload
configuration.

---

**Question 2**
A developer says: "We run on a private cluster with no public API endpoint,
so external attackers cannot reach us." What is incomplete about this reasoning,
and what risks does it not address?

*A strong answer covers:* internal threat paths (compromised workload, stolen
developer credential, CI pipeline access), the fact that most real incidents
start from inside the perimeter, and at least two specific risks a private
cluster does not mitigate — for example, a pod exploiting another pod via
the flat network, or a developer with cluster-admin whose laptop is compromised.

---

**Question 3**
You discover a ClusterRoleBinding granting `cluster-admin` to a service account
named `monitoring-agent` in the `default` namespace. The binding is two years
old. What do you do, and in what order?

*A strong answer covers:* not deleting immediately — first determine whether
any running workload depends on this binding, then check audit logs to see
what API calls have been made with this service account recently, then identify
the legitimate permission requirement and replace the binding with a least-
privilege alternative, then remove the cluster-admin binding. Explains why
order matters: deleting first could break a production workload with no warning.

---

**Question 4**
An attacker compromises a pod that has the default service account with no
explicit RBAC bindings. Describe what the attacker can and cannot do from
inside that pod.

*A strong answer covers:* can make network connections to other pods (unless
NetworkPolicy restricts it), can reach the metadata API (in cloud environments),
can call the Kubernetes API with the mounted token but with very limited
permissions in a default setup. Cannot read secrets, cannot list cluster
resources, cannot create pods. Next step the attacker would try: scan the
network for reachable services, attempt to reach the metadata API for cloud
credentials. Notes that "very limited" depends on cluster configuration —
some clusters bind roles to the default service account explicitly.

---

**Question 5**
Explain why etcd is a higher-value target than the API server in the context
of an attacker who already has network access to the control-plane subnet but
has no Kubernetes credentials.

*A strong answer covers:* the API server requires valid credentials (certificate,
token, or OIDC) to return data — unauthenticated access is rejected. etcd
without client authentication returns the full cluster state including all
secrets. If etcd is accessible from the node subnet without mutual TLS, an
attacker on any node can read everything. Also notes that etcd backups
often have weaker access controls than etcd itself and represent the same risk.

---

**Question 6**
A security scan shows that a cluster's kubelet on every node has
`--anonymous-auth=true`. The platform team says "we've had this configuration
for two years and nothing bad has happened." How do you respond?

*A strong answer covers:* the absence of a detected incident does not mean
the configuration is safe — it means either no one with network access to
the node subnet has exploited it yet, or they have and it was not detected.
Explains specifically what `--anonymous-auth=true` enables (unauthenticated
API calls to port 10250, which allows exec into pods, log reads, and kubelet
queries). Proposes a remediation path that does not require cluster downtime.
