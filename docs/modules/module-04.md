# Module 04: Network Isolation And Exposure Reduction

## Why This Matters

A Kubernetes cluster without NetworkPolicy is a flat network. Every pod can
reach every other pod, every service, the cloud metadata API, and in many
configurations the API server itself — unless something at the host or cloud
layer prevents it. Most clusters do not have those host-layer controls.
The result is that a single compromised pod has network access to the entire
cluster, and lateral movement costs the attacker almost nothing. NetworkPolicy
changes this, but it also has real limits. Understanding those limits is as
important as knowing how to write policies.

---

## The Attack Narrative

A security researcher is testing a financial services company's cluster. She
has compromised a low-privilege frontend pod through a JavaScript injection
vulnerability. The pod has a hardened security context — non-root, no
privileged access, read-only filesystem. Pod hardening worked. She cannot
escape to the node.

But she can make network connections.

She runs a port scan from inside the pod. In a flat cluster with no
NetworkPolicy, she reaches: the database pods in the `data` namespace on
port 5432, an internal admin panel in the `ops` namespace on port 8080,
and the AWS instance metadata API at `169.254.169.254`.

The admin panel was designed as "internal only," meaning the VPC perimeter
was the assumed protection. It accepts requests from inside the cluster
without authentication. She lists all users and exports a dataset. The
metadata API provides temporary IAM credentials tied to the EC2 node's
instance role. The role has `s3:GetObject` on a bucket containing customer
backups. She downloads three months of data.

She never escalated privileges. She never left the frontend pod. The flat
network gave her access to everything that pod could reach, which turned
out to be a significant portion of the company's infrastructure.

This scenario plays out in real clusters regularly, and it does not require
NetworkPolicy to be absent everywhere — just absent for the paths that matter.

---

## How Kubernetes Networking Creates Risk

Kubernetes assigns every pod an IP address. By default, every pod IP is
reachable from every other pod IP on the cluster network. There are no
implicit firewall rules between namespaces or between pods. A pod in the
`frontend` namespace can initiate a TCP connection to a pod in the `data`
namespace on any port, and the connection succeeds unless NetworkPolicy or
a host-level rule blocks it.

Services add a stable DNS name and virtual IP as an abstraction layer, but
they do not add a security boundary. If a pod can reach another pod's IP,
it can also reach it via its service name. NetworkPolicy enforces at the
pod IP layer, which means it controls both direct pod-to-pod traffic and
traffic through service VIPs.

**DNS is a special case** that breaks every new NetworkPolicy implementation.
Kubernetes uses CoreDNS for internal name resolution. If you apply a
default-deny egress policy without explicitly allowing UDP and TCP port 53
to kube-dns, every DNS lookup in that namespace fails. Service names stop
resolving. Applications break immediately. DNS must always be explicitly
allowed — even in the most restrictive egress policy — or nothing works.

This is the most common first mistake when rolling out NetworkPolicy in an
existing environment. Test DNS resolution before declaring success.

---

## What NetworkPolicy Controls And What It Does Not

NetworkPolicy is enforced by the CNI plugin, not by Kubernetes itself. This
is the most important thing to verify before writing a single policy: does
your CNI enforce NetworkPolicy? Flannel does not. Calico, Cilium, and the
AWS VPC CNI with the network policy add-on do. A cluster running Flannel with
NetworkPolicy resources applied has the illusion of network controls with none
of the enforcement.

**What NetworkPolicy controls:**
- Inbound (ingress) and outbound (egress) traffic to pods matched by label
- Traffic from specific namespaces (matched by namespace label)
- Traffic from or to specific pod labels
- Traffic to specific CIDR ranges
- Traffic on specific ports and protocols

**What NetworkPolicy does not control:**
- Traffic from pods running with `hostNetwork: true`. These pods use the
  node's network stack directly, bypassing the pod network and all NetworkPolicy
  rules. A `hostNetwork` pod can reach everything the node can reach.
- Traffic from privileged pods that manipulate iptables or eBPF rules directly.
- Node-to-node traffic, kubelet traffic, and control-plane communication.
- TLS certificate validity — NetworkPolicy controls whether a connection can
  be established, not whether the certificate presented is valid.
- DNS query content — NetworkPolicy can block DNS entirely, but it cannot
  inspect or filter specific DNS queries. DNS tunneling moves data inside
  DNS queries that NetworkPolicy permits.

The most important implication: NetworkPolicy is only as effective as your
pod security baseline. A cluster with strong NetworkPolicy but no admission
policy blocking `hostNetwork: true` can have its entire network isolation
bypassed by any workload that can run with that setting. Modules 3 and 6
are prerequisites for NetworkPolicy to be a meaningful control.

---

## The Metadata API: The Most Overlooked Risk

In cloud environments, the cloud metadata API at `169.254.169.254` is one of
the highest-value targets reachable from a compromised pod. This address is
link-local — reachable from any host on the same network segment without
routing through a gateway. On cloud VMs it provides instance metadata and,
critically, temporary IAM credentials tied to the node's instance role.

If the node's IAM role has permissions beyond what nodes strictly need for
cluster operation — and it often does — a compromised pod with metadata API
access can obtain those credentials and use them outside the cluster entirely.
NetworkPolicy egress rules to `169.254.169.254` can block this from most pods.

IMDSv2 provides defense-in-depth at the hypervisor level for AWS. It requires
a PUT request to obtain a session token before any metadata request, and
setting the hop limit to 1 means pods cannot obtain that session token — the
PUT must arrive within one network hop, but pod traffic takes two hops (pod
to node, node to metadata service). This blocks pod access to instance
credentials even without NetworkPolicy. It should be configured at cluster
creation and verified: IMDSv2 with hop limit 1, not IMDSv1 or IMDSv2 with
higher hop limits.

---

## Decision Framework: Default Deny Then Allow

When writing NetworkPolicy for a production namespace, start with default-deny
and add explicit allow rules for each required flow. The question for every
allow rule is not "will this break anything?" but "does this workload genuinely
need this traffic path, and what is the blast radius if the source is
compromised?"

**Define ingress first:** What should be able to send traffic to pods in this
namespace? Usually: the ingress controller (for external-facing services) and
specific namespaces that are legitimate clients. Nothing else.

**Define egress second:** What should pods in this namespace be able to
initiate connections to? Always: kube-dns on 53 UDP/TCP. Then: specific
databases or services the workload calls, the external APIs it legitimately
uses. Never: the metadata API for application workloads, arbitrary internet
CIDR ranges.

**For each allowed flow, ask:** If the source is compromised, what does this
flow enable? A direct database connection from a compromised frontend pod is
a much higher-risk allow rule than a connection from an API server to a
read-only cache. Risk varies by traffic path, not just by namespace.

**Scope by port:** Allow TCP 5432 to the database pod, not all TCP to the
database namespace. Narrowing the port reduces what an attacker can do even
if they compromise the source.

---

## The Decay Problem

NetworkPolicy does not maintain itself. A policy set that is correct today
will be wrong in six months as the application architecture changes. A new
service gets deployed, someone adds a broad allow rule to unblock it quickly,
the exception is never reviewed, and the effective policy is now weaker than
intended.

Master-level practice treats NetworkPolicy as a living document. Every
architecture change that adds a new service or a new communication flow should
include a NetworkPolicy review. Every exception should have a justification and
a review date. The policy set should be version-controlled alongside the
application manifests, not managed by hand in the cluster.

---

## What Good Looks Like vs What Compliant Looks Like

A compliant cluster has a CNI that supports NetworkPolicy and at least some
policies applied. A secure cluster has default-deny applied to every production
namespace, explicit allow rules for every required traffic flow reviewed against
the current architecture, metadata API access blocked for all application
workloads, and the policy set maintained as the application evolves.

The difference is coverage and maintenance. Compliance is having the feature
enabled. Security is having the feature enforced everywhere it matters, with
policies that accurately reflect the current threat model.

---

## You Will Learn

- why default-allow networking enables lateral movement and what default-deny requires
- how to write ingress and egress NetworkPolicy that matches real traffic flows
- why DNS requires an explicit allow in every restrictive egress policy
- what NetworkPolicy cannot control and where other layers must compensate
- how the cloud metadata API creates credential theft risk and how to mitigate it

## Key Questions

- What attack paths remain after NetworkPolicy is fully applied?
- Which traffic flows are the highest-risk if the source workload is compromised?
- How does a `hostNetwork` pod bypass all NetworkPolicy controls?
- What is IMDSv2 hop-limit enforcement, and why does it block pod metadata access?

## Hands-On

- Lab: [Lab 04](../../labs/module-04/README.md)
- Assets:
  - [default-deny.yaml](../../labs/module-04/default-deny.yaml)
  - [allow-app-to-db.yaml](../../labs/module-04/allow-app-to-db.yaml)
  - [egress-restrict.yaml](../../labs/module-04/egress-restrict.yaml)

## Output

Produce a network policy set with default-deny and scoped allow rules, and a
gap analysis of the attack paths that remain after the policies are applied.

---

## Self-Assessment

**Question 1**
You apply a default-deny egress policy to the `payments` namespace. Immediately
after applying it, every application in the namespace fails to resolve service
names. No pods can connect to anything. What did you forget, and why does this
happen?

*A strong answer covers:* DNS resolution uses UDP and TCP port 53 to
kube-dns (`kube-system` namespace). A default-deny egress policy blocks all
outbound traffic including DNS queries. The fix is an explicit egress allow rule
permitting UDP/TCP on port 53 to the kube-dns pod or its service IP. Explains
that this is the most common first mistake when rolling out NetworkPolicy —
DNS must always be explicitly allowed in any restrictive egress policy.

---

**Question 2**
A developer argues: "We have NetworkPolicy on all application namespaces so
lateral movement between namespaces is blocked." A pod in `frontend` is
compromised. What can the attacker still do, and what does NetworkPolicy not
prevent?

*A strong answer covers:* NetworkPolicy controls pod-to-pod traffic but does
not prevent the compromised pod from reaching its explicitly allowed traffic
paths (which may include a database or cache). If the CNI does not enforce
NetworkPolicy, all policies are silently ignored. `hostNetwork: true` pods
bypass NetworkPolicy entirely. The metadata API at `169.254.169.254` is
reachable unless explicitly blocked. The attacker can still call any allowed
external destination. Correct answer: NetworkPolicy is one layer; its
effectiveness depends on CNI support, pod security baseline, and policy
completeness.

---

**Question 3**
You discover that your cluster is running Flannel as the CNI. You have 30
NetworkPolicy resources applied across production namespaces. What is the
security implication, and what do you do?

*A strong answer covers:* Flannel does not enforce NetworkPolicy. All 30
policies are ignored — the cluster is operating with a flat network regardless
of what the policies say. Immediate action: evaluate CNI migration to Calico,
Cilium, or the AWS VPC CNI with network policy add-on. Until migration, all
other controls that assume network isolation are undermined. Notes that CNI
enforcement verification is the first step before writing any NetworkPolicy.

---

**Question 4**
Explain why blocking the metadata API at `169.254.169.254` with NetworkPolicy
is a defense-in-depth control rather than a complete solution, and what
the complementary control is.

*A strong answer covers:* NetworkPolicy can block pod access to the metadata
API, but it depends on CNI enforcement and policy completeness. IMDSv2 with
a hop limit of 1 blocks pod access at the hypervisor/network layer regardless
of Kubernetes configuration — the PUT request for a session token cannot reach
the metadata service from a pod because it takes two hops (pod to node, node
to IMDS). Both controls should be in place: NetworkPolicy egress block for
defense-in-depth, IMDSv2 hop-limit-1 as the authoritative control. Neither
alone is sufficient.

---

**Question 5**
A team wants to allow their API pods to connect to an external payment
processor on `api.payments-provider.com`. How do you write the egress
NetworkPolicy, and what gaps remain after applying it?

*A strong answer covers:* NetworkPolicy does not support DNS-based egress
rules — policies reference IP CIDRs, not hostnames. To allow traffic to a
domain, you must allow the CIDR range of that domain's IPs, which can change.
The correct approach is to allow port 443 to the specific CIDR block of the
payment processor, or use a service mesh or egress gateway that can enforce
FQDN-based egress. Gaps that remain: the allowed CIDR may include other hosts
at the same IP range, and IP changes can break the policy without notice.
A policy must be maintained as architecture changes.

---

**Question 6**
During a network policy review you find an allow rule that permits all egress
traffic from the `reporting` namespace to `0.0.0.0/0` on port 443. A developer
says this is needed for "HTTPS to external APIs." How do you evaluate this?

*A strong answer covers:* `0.0.0.0/0` on 443 allows the reporting namespace
to make HTTPS connections to any IP on the internet, including attacker-
controlled infrastructure. The correct approach is to enumerate the specific
external endpoints the reporting service needs and restrict to those CIDRs,
or use an egress gateway/proxy to enforce FQDN-based allowlists. The vague
"external APIs" justification requires a concrete list of destinations — any
destination not on that list should not be reachable. Explains that port-443-
to-anywhere is a common data exfiltration path because HTTPS is rarely blocked.
