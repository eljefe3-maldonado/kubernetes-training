# Module 08: Cluster Hardening And Platform Baselines

## Why This Matters

The workload controls in previous modules — pod security, NetworkPolicy,
admission policies — all run on top of a platform. If the platform itself
is misconfigured, every workload control built on it is undermined. A kubelet
that accepts unauthenticated requests makes pod security contexts irrelevant.
An etcd backup with world-readable storage makes RBAC irrelevant for secrets.
An API server with no audit logging makes detection and forensics blind. This
module is about the layer below the workloads — the cluster infrastructure
that everything else depends on.

---

## The Attack Narrative

A cloud security engineer joins a company that has been running Kubernetes
for two years. The clusters were set up by a contractor who is no longer
available. The security engineer starts with a review using kube-bench, the
CIS Kubernetes Benchmark scanner.

The results show four critical findings. The kubelet on every node has
`--anonymous-auth=true` and `--authorization-mode=AlwaysAllow`. This means
any unauthenticated request to port 10250 on any node is accepted and
authorized. An attacker on the cluster network can run commands in any pod
on any node, read logs for any container, and query kubelet endpoints — all
without any credential.

The second finding: etcd is accessible from the node subnet without
authentication. An attacker who compromises a node can connect to etcd
directly and read the entire cluster state.

Third: audit logging is not configured. There is no record of any API
server request, ever. If an attacker has been accessing the cluster, there
is no log to investigate.

Fourth: the API server has a public endpoint with no authorized IP
restrictions. It is protected only by authentication, which is fine in
theory, but there are several service account tokens with long TTLs in
the cluster, and one of them was visible in a public GitHub repository
for an indeterminate time.

None of these findings require a software vulnerability to exploit.
They are configuration choices — or rather, absent configuration choices,
where the defaults were left in place. The cluster had been running for
two years with an open kubelet API, direct etcd access from nodes, and no
audit log. The security team had no idea what had happened in that cluster.

---

## The Shared Responsibility Model In Practice

The most important concept in cluster hardening for managed Kubernetes is
understanding what the cloud provider owns and what the customer owns. This
boundary is often misunderstood, and that misunderstanding creates gaps.

For managed services like EKS, GKE, and AKS:

**The provider owns** the control-plane infrastructure: API server, etcd,
scheduler, controller manager. They patch and upgrade these components. They
handle the physical security of the underlying hardware. They manage the TLS
certificates for control-plane communication. For etcd specifically, the
provider manages the etcd instances, network isolation, and peer authentication.

**The customer owns** everything else: node OS configuration and patching,
kubelet configuration, network policy enforcement, RBAC model, workload
security contexts, audit log configuration and retention, secrets encryption
key management (if using customer-managed KMS keys), and add-on security.

The gap that catches teams: they see "managed Kubernetes" and assume "managed
security." It is not. The provider manages the control plane. The customer
is responsible for every security decision about what runs on that control
plane, how the nodes are configured, and what access controls govern the
workloads.

A concrete example: on EKS, etcd encryption at rest is available but must
be explicitly enabled with a customer-managed KMS key at cluster creation
time. If it is not enabled, secrets are stored in etcd without encryption.
This is a customer responsibility. The provider does not enable it by default.
Many EKS clusters do not have it configured.

---

## What The CIS Benchmark Measures And What It Misses

The CIS Kubernetes Benchmark is a useful baseline. Running kube-bench against
a cluster and addressing every FAIL finding produces a cluster with reasonable
hygiene. It is not a complete security posture.

The benchmark checks configuration settings: flags on the API server, kubelet
configuration, file permissions on certificate files. It does not check:
- Whether the RBAC model is least-privilege (Module 2)
- Whether secrets are handled safely (Module 5)
- Whether admission policies are enforced (Module 6)
- Whether anyone is monitoring the audit logs (Module 9)

A cluster that passes every CIS Kubernetes Benchmark check can still be
trivially compromised if a service account has cluster-admin and its token
is in a public repository. Benchmark compliance is a floor, not a ceiling.

The value of the benchmark is that it catches the configuration-level issues
that are easy to overlook and that collectively form the foundation the other
controls depend on. Audit logging, kubelet authentication, etcd access controls,
API server flag hardening — these are unglamorous but foundational. They are
worth doing systematically.

---

## Audit Logging: The Foundation Of Everything After

Audit logging deserves special emphasis because it is the prerequisite for
every detection and forensics capability in Modules 9 and 10. Without audit
logs, you cannot detect privilege escalation, cannot investigate an incident,
and cannot determine the blast radius of a credential compromise.

Kubernetes audit logs record every request to the API server: who made it,
what they requested, whether it was allowed, and — depending on the audit
level — the full request and response bodies. The audit policy controls what
is logged at what verbosity. The choices involve a tradeoff between storage
cost and forensic completeness.

A minimal audit policy that only logs `Metadata` level for everything produces
a usable log for most detection use cases. The request method, resource type,
resource name, user identity, and response code are usually sufficient to
detect anomalous access patterns.

The events that should always be logged at `RequestResponse` level — capturing
the full request and response bodies — are: secret access, RBAC changes,
service account token creation, pod exec and attach, and namespace deletion.
These are the events where having the full body matters for forensics. The
storage cost for these specific event types is manageable even for large clusters.

The audit policy in `labs/module-08/audit-policy.yaml` implements this
approach. Read it not just as a configuration file but as a set of decisions
about what information is worth paying to retain.

---

## Node Hardening: The Often-Skipped Layer

Nodes are the physical substrate the cluster runs on. A compromised node
gives an attacker access to every pod on that node — their processes, their
memory, their mounted secrets, their network traffic. Node security is not
glamorous, but it is the foundation that pod security contexts and NetworkPolicy
are built on.

Key node hardening practices:

**Immutable OS images**: use an OS designed for container workloads that
cannot be modified after boot. Amazon Linux 2023, Google Container-Optimized
OS, and Azure's AKS node image all fall into this category. They have minimal
attack surface and no package manager running at runtime. Modifications require
replacing the node, not patching it in place.

**No direct SSH**: engineers should not SSH to nodes to debug production issues.
Use `kubectl exec` for pod debugging and cloud-native session management
(AWS SSM Session Manager, GCP IAP Tunneling) for node access that requires
an audit trail without SSH key management.

**Node patching cadence**: node OS updates contain kernel patches that close
vulnerabilities. A node running a kernel that is twelve months old may have
dozens of published CVEs. Automated node group rolling updates that replace
nodes with new AMIs on a monthly or bi-weekly schedule are the practical
solution for managed clusters.

**Kubelet hardening**: at minimum, disable anonymous authentication
(`--anonymous-auth=false`), enable Webhook authorization (`--authorization-mode=Webhook`),
disable the read-only port (`--read-only-port=0`), and enable node restriction
(the NodeRestriction admission plugin limits what the kubelet can modify in
the API server to only objects related to its own node).

---

## What Good Looks Like vs What Compliant Looks Like

A compliant cluster passes the CIS Kubernetes Benchmark. A secure cluster
passes the benchmark, has audit logging enabled and shipped to a retained
destination, has kubelet authentication and authorization configured correctly,
has etcd encryption at rest with a customer-managed key, has nodes patched
within 30 days of OS releases, and has a clear picture of which controls are
provider-owned and which are customer-owned with verification that the
customer-owned ones are actually in place.

The telling question: if your cluster were compromised yesterday, could you
reconstruct what the attacker did? If the answer is no — because audit logs
are not configured or not retained — then the cluster is not secure regardless
of what other controls are in place.

---

## You Will Learn

- the control-plane components that underpin all other cluster security
- where managed Kubernetes ends and customer responsibility begins
- what the CIS Kubernetes Benchmark measures and what it misses
- how to configure audit logging to support detection and forensics
- node hardening practices that reduce the blast radius of node compromise

## Key Questions

- Which hardening actions reduce the most risk with the lowest operational cost?
- Which controls are owned by the cloud provider, and how do you verify they are in place?
- If your cluster were compromised yesterday, what could you reconstruct from your logs?
- What is the difference between a cluster that passes the CIS benchmark and one that is secure?

## Hands-On

- Lab: [Lab 08](../../labs/module-08/README.md)
- Assets:
  - [hardening-checklist.md](../../labs/module-08/hardening-checklist.md)
  - [audit-policy.yaml](../../labs/module-08/audit-policy.yaml)

## Output

Produce a hardening checklist with a provider/customer ownership split and a
prioritized gap analysis ranked by risk and remediation effort.

---

## Self-Assessment

**Question 1**
A cluster has been running for two years. The CIS Kubernetes Benchmark scan
shows no FAIL findings. A security engineer runs a manual review and finds
the service account has cluster-admin, secrets are not encrypted at rest,
and audit logging is sending to a bucket that is world-readable. What does
this tell you about the benchmark?

*A strong answer covers:* the CIS Kubernetes Benchmark checks configuration
settings — API server flags, kubelet configuration, certificate file permissions.
It does not check RBAC models, secrets encryption key management, or audit
log access controls. A clean benchmark pass means the platform configuration
is correct. It says nothing about the RBAC model, the handling of secrets, or
whether detection is functional. The benchmark is a floor, not a complete
security assessment. The finding illustrates that benchmark compliance must
be combined with RBAC review (Module 2), secrets posture review (Module 5),
and detection capability review (Module 9).

---

**Question 2**
You are told that your EKS cluster uses "managed Kubernetes so the provider
handles etcd security." What exactly does AWS handle, and what do you as the
customer need to verify?

*A strong answer covers:* AWS manages the etcd infrastructure — the physical
nodes, network isolation, peer authentication between etcd members, and TLS
for etcd communication. The customer is responsible for whether envelope
encryption is enabled (it must be explicitly configured at cluster creation
with a KMS key ARN — it cannot be added after the fact). The customer must
verify: (1) envelope encryption is enabled (`aws eks describe-cluster` shows
`encryptionConfig`), (2) the KMS key is customer-managed and not marked for
deletion, (3) key rotation is enabled. If none of these are configured,
secrets are stored in etcd without encryption — a customer responsibility
gap, not a provider gap.

---

**Question 3**
A node's kubelet is configured with `--anonymous-auth=true` and
`--authorization-mode=AlwaysAllow`. Describe the specific impact an attacker
with network access to the node subnet could have.

*A strong answer covers:* with these settings, any unauthenticated request to
port 10250 is accepted and authorized. The attacker can: list all pods on the
node (`/pods` endpoint), read logs for any container (`/containerLogs/namespace/pod/container`),
execute commands in any running container (`/exec/namespace/pod/container`),
and access the kubelet's API for health and stats queries. This is equivalent
to having `kubectl exec` and `kubectl logs` access to every pod on that node
without any credential. Remediation: `--anonymous-auth=false`, `--authorization-mode=Webhook`
(which delegates authorization to the API server NodeAuthorizer). This is a
critical finding requiring immediate remediation.

---

**Question 4**
Your audit policy logs everything at `Metadata` level. During an incident
investigation, you need the full request body for a secret that was read
four days ago. Can you get it?

*A strong answer covers:* no. `Metadata` level logs the request metadata —
who made it, what resource they accessed, the response code — but not the
request or response body. To capture what a secret read returned, the audit
policy must log secret access at `RequestResponse` level. The fix going
forward: add a specific rule in the audit policy that matches `get secrets`
and sets level `RequestResponse`. The lesson: audit policy decisions have
forensic consequences. The events that should always be logged at
`RequestResponse` include secret access, RBAC changes, service account token
creation, and pod exec/attach. Storage cost for these specific events is
manageable and is worth the forensic completeness.

---

**Question 5**
A platform team member SSHes into a production node to debug a scheduling
issue. This is the current practice for node-level debugging. What risks
does this create, and what is the alternative?

*A strong answer covers:* SSH to nodes requires managing SSH keys (rotated
how often?), creates a privileged access path without Kubernetes audit logging,
and the session is not recorded in any centralized audit trail. If the
engineer's laptop is compromised, the attacker has an SSH key with node access.
The alternative: use cloud-native session management — AWS SSM Session Manager,
GCP IAP Tunneling — which requires no open SSH port, authenticates through
cloud IAM, creates an auditable session record in CloudTrail, and can be
scoped with IAM policies. For pod-level debugging, `kubectl debug` with
ephemeral containers is the correct tool. Direct SSH to production nodes
should be a last resort, not standard practice.

---

**Question 6**
You inherit a cluster where audit logging was never configured. A potential
incident is reported — a service account may have read secrets it was not
supposed to access three weeks ago. What can you reconstruct, and what cannot
be recovered?

*A strong answer covers:* without audit logs, the API server requests from
three weeks ago are unrecoverable — they were never written to disk. You can
check current RBAC to see if the service account still has the permission it
allegedly used. You can check any application logs from the pods using that
service account for behavioral indicators. You may be able to check cloud
provider logs (CloudTrail) if the service account used IRSA credentials to
call AWS APIs outside the cluster. But the Kubernetes API call record is gone.
This is the case for enabling audit logging before you need it. The answer
should include steps to enable audit logging immediately and what to configure
going forward.
