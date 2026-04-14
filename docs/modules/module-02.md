# Module 02: Identity, RBAC, And Privilege Escalation

## Why This Matters

Most Kubernetes incidents do not require a software exploit. They require
finding a credential that has more authority than it should, or finding a
path from a limited credential to an unlimited one. RBAC is the control that
determines what every principal in the cluster can do. When it is wrong —
and it is almost always wrong in ways that are not obvious — it becomes the
attacker's primary tool rather than the defender's.

---

## The Attack Narrative

A security consultant is hired to assess a production Kubernetes cluster.
She has no special access — just a service account token she found in the
company's internal wiki, left there by a developer who was trying to document
how to deploy a monitoring agent. The wiki entry is six months old and no one
has thought about the token since.

She starts by asking the API server what her token can do:

```bash
kubectl auth can-i --list
```

The token belongs to a service account called `monitoring-agent`. It was given
a ClusterRole because the original developer thought it was simpler than
managing per-namespace roles. The ClusterRole has `get`, `list`, and `watch`
on `secrets`. Cluster-wide. That means every secret in every namespace.

She lists the secrets in the `ci-system` namespace. One of them is a
`github-token` with `repo:write` scope and a second is an AWS access key
for the CI pipeline's deployment role. The AWS role has `AdministratorAccess`
on the production account.

The monitoring agent token was never intended to give access to CI credentials.
No one decided it should. It happened because the ClusterRole was written with
broad verbs and broad resources, and no one reviewed what that meant across
all namespaces.

This is not a sophisticated attack. It is a permissions inventory. The
consultant did not exploit a vulnerability — she asked the cluster what she
was allowed to do, and the cluster told her.

---

## How RBAC Works And Where It Breaks

Kubernetes RBAC has four objects: Role, ClusterRole, RoleBinding, and
ClusterRoleBinding. A Role is namespaced. A ClusterRole is cluster-wide.
A RoleBinding grants a Role or ClusterRole within a specific namespace.
A ClusterRoleBinding grants a ClusterRole across the entire cluster.

The most important thing to understand about this model is what it does not
provide: there is no automatic least-privilege default. Nothing in Kubernetes
grants the minimum permissions required for a workload to function. That
judgment has to come from the engineer who writes the RBAC. When that judgment
is absent or rushed, the result is always over-privilege.

**The verbs that matter most** are not always the obvious ones. `create pods`
is dangerous because someone who can create pods can run arbitrary container
images, potentially with `privileged: true`, giving them node-level access.
`get secrets` without `resourceNames` restriction means all secrets in scope,
not just the one the workload needs. `bind` allows the principal to attach
existing roles to new subjects — a path to cluster-admin that does not require
creating new RBAC objects. `escalate` allows modifying a role to include
permissions the granting principal does not themselves hold. `impersonate`
allows acting as any user or service account. These three verbs — `bind`,
`escalate`, `impersonate` — should never appear in production RBAC outside
of the platform team, and even then they should be scrutinized.

**The path from namespaced to cluster-scoped** is the most common escalation
route. A service account in namespace A has `get` on pods in namespace A.
That seems fine. But if that same service account also has `list` on secrets
cluster-wide via a ClusterRoleBinding, the namespace scope of the pod access
is irrelevant. The effective blast radius is the entire cluster.

**Service account token handling** is where theory meets practice. Every pod
gets a service account token mounted by default unless `automountServiceAccountToken`
is set to `false`. In a default cluster configuration with no explicit RBAC
bindings, the default service account token cannot do much. But in a cluster
where someone has bound a ClusterRole to the default service account "to make
things easier," every pod in that namespace can do everything that ClusterRole
allows — including pods created by attackers.

---

## Privilege Escalation Paths You Must Know

**Path 1: Token theft from a running pod.** An attacker compromises a pod.
The pod's service account token is mounted at
`/var/run/secrets/kubernetes.io/serviceaccount/token`. The attacker reads it,
uses it from their own infrastructure, and now has whatever RBAC that service
account has. This is why `automountServiceAccountToken: false` matters for
any workload that does not call the Kubernetes API.

**Path 2: Secret read leads to more privileged credential.** A service account
can read secrets. Among those secrets is a token for a different, more
privileged service account — perhaps a CI deployment account or a platform
tool. The attacker reads that token and uses it to escalate. The RBAC control
that prevents this is `resourceNames`: restrict secret access to only the
specific named secrets the workload genuinely needs.

**Path 3: `create pods` leads to node access.** A service account can create
pods. The attacker creates a pod with `privileged: true`, `hostPID: true`, and
a `hostPath` volume mounting `/`. They now have full access to the node's
filesystem. This is why admission control (Module 6) and pod security policy
must work together with RBAC — RBAC alone cannot prevent this if `create pods`
is allowed.

**Path 4: ClusterRoleBinding to cluster-admin.** A service account has the
`bind` verb on ClusterRoles. The attacker creates a ClusterRoleBinding that
assigns `cluster-admin` to a service account they control. This is a two-step
escalation that looks like routine RBAC administration in logs that are not
specifically monitored for it.

---

## The Misconception: Namespacing Provides Isolation

A team creates a dedicated namespace for a sensitive workload and binds a Role
rather than a ClusterRole. They believe the workload is isolated. It may not
be.

If the workload's service account token is stolen, the attacker has
namespace-scoped access. That is genuinely more limited than cluster-admin.
But if that namespace's Role includes `get secrets` — even just in that
namespace — and that namespace happens to contain a CI token, a database
credential, or a cloud provider access key, the attacker has everything they
need for lateral movement outside the cluster entirely.

Namespace isolation is meaningful. It is not a substitute for reviewing what
specific permissions exist and what resources they expose. A well-scoped Role
in a badly organized namespace is still dangerous.

---

## Decision Framework: Writing RBAC From Scratch

Start from zero and add only what you can justify. The question for every rule
is not "will this break anything?" but "does this workload genuinely need this
permission to perform its function?"

**Step 1: Identify the workload's actual API calls.** Most application
workloads make zero Kubernetes API calls. They talk to databases, external
APIs, and other services. They do not need a service account with any RBAC
bindings at all. Set `automountServiceAccountToken: false` and move on.

**Step 2: For workloads that do call the API, name every call.** A controller
that watches Deployments and reads ConfigMaps needs `get`, `list`, `watch` on
Deployments and ConfigMaps — in the specific namespaces it operates in, not
cluster-wide unless it genuinely monitors all namespaces.

**Step 3: Use `resourceNames` for secrets.** Any time a workload reads a
secret, scope it to the specific secret name. `resourceNames: [db-credentials]`
means the service account can only access that one secret. It cannot enumerate
other secrets even if it knows their names.

**Step 4: Prefer RoleBinding over ClusterRoleBinding.** A ClusterRoleBinding
means the permissions apply everywhere, forever, regardless of what new
namespaces are created. A RoleBinding means the permissions are scoped to one
namespace. Even when a workload needs to operate in multiple namespaces,
creating one RoleBinding per namespace is safer than one ClusterRoleBinding.

**Step 5: Review for escalation verbs before applying.** Before any RBAC is
applied to production, check for `bind`, `escalate`, `impersonate`, `create pods`,
`create clusterrolebindings`, and wildcard verbs. Any of these requires
explicit sign-off.

---

## What Good Looks Like vs What Compliant Looks Like

A compliant cluster has RBAC enabled — that is the default, and it is a low
bar. A secure cluster has an RBAC model where every binding can be traced to
a specific workload need, has been reviewed by someone who understands the
escalation paths, and is tested against `kubectl auth can-i --list` to confirm
that the effective permissions match the intended permissions.

Compliance says "we use RBAC." Security says "we know what every service
account can do, why it needs those permissions, and what an attacker could do
with that token." The difference is legibility. If you cannot explain in one
sentence why a binding exists, it should not exist.

---

## You Will Learn

- how Kubernetes authenticates users, service accounts, and controllers
- RBAC objects and how they compose into effective permissions
- privilege escalation paths through verbs, bindings, and secret access
- why namespacing does not substitute for least-privilege RBAC
- cloud IAM federation as a replacement for static service account tokens

## Key Questions

- Which principals can mint more privilege than they appear to have?
- Which permissions create indirect cluster-admin paths?
- What is the blast radius if this service account token is stolen?
- Where should cloud IAM federation replace long-lived static credentials?

## Hands-On

- Lab: [Lab 02](../../labs/module-02/README.md)
- Assets:
  - [over-privileged-rbac.yaml](../../labs/module-02/over-privileged-rbac.yaml)
  - [hardened-rbac.yaml](../../labs/module-02/hardened-rbac.yaml)

## Output

Produce a hardened RBAC model and a review memo documenting the original
escalation paths and how each was closed.

---

## Self-Assessment

**Question 1**
A service account needs to read a database credential secret named
`db-credentials` in the `app` namespace. A developer writes a Role with
`get, list` on `secrets` in that namespace. What is wrong, and what is the
correct replacement?

*A strong answer covers:* `list` on secrets reveals the names and metadata
of every secret in the namespace, even if values cannot be read. Correct
replacement: `get` only, with `resourceNames: ["db-credentials"]` to restrict
to the specific named secret. Explains why `resourceNames` matters — it turns
a namespace-wide permission into a single-resource permission.

---

**Question 2**
A ClusterRole contains the `bind` verb on `clusterroles`. Explain the specific
privilege escalation path this enables and why it is dangerous even if no
other permission looks problematic.

*A strong answer covers:* `bind` allows the principal to create a
ClusterRoleBinding that assigns any existing ClusterRole — including
`cluster-admin` — to any subject. The principal does not need `create
clusterrolebindings` explicitly if `bind` is present. This is a two-step
path to cluster-admin: create a service account, bind cluster-admin to it,
use the new service account. Emphasizes that this is often present in CI/CD
service accounts "for deployment flexibility" without understanding what it
enables.

---

**Question 3**
An engineering lead argues that using `cluster-admin` for the CI/CD pipeline
service account is acceptable because "we control the pipeline and trust it
completely." Construct a specific scenario where this reasoning leads to a
real incident.

*A strong answer covers:* a supply chain attack on the pipeline (malicious
dependency or build tool modification), a compromised pipeline runner host,
a developer with write access to the pipeline config who is socially
engineered, or a leaked pipeline service account token in a log or repository.
In any of these cases, `cluster-admin` means the incident is automatically
a full cluster compromise — every secret, every workload, every namespace.
A least-privilege alternative (deploy verbs in specific namespaces only)
limits the blast radius to those namespaces.

---

**Question 4**
A service account has only `create pods` in the `app` namespace and no other
permissions. Is this dangerous? Explain the specific attack path.

*A strong answer covers:* yes — `create pods` allows creating a pod with
`privileged: true`, `hostPath: /`, and `hostNetwork: true` unless admission
policy blocks it. A privileged pod with host filesystem access is equivalent
to root on the node. If admission policy is not in place (Module 6), `create
pods` alone is a path to node compromise. Notes the dependency: RBAC and
admission control must both be correct for either to be effective.

---

**Question 5**
You need to give a Prometheus monitoring agent read access to pod metrics and
logs across all namespaces. Design the minimum viable RBAC model and explain
each choice.

*A strong answer covers:* a ClusterRole (not Role) because the agent needs
cluster-wide access, with `get, list, watch` on `pods` and `pods/log` only —
not `secrets`, not `configmaps`, not wildcard resources. A ClusterRoleBinding
to the monitoring service account. Sets `automountServiceAccountToken: false`
on all other service accounts in the monitoring namespace to prevent token
misuse. Explains why ClusterRole with ClusterRoleBinding is justified here
(cluster-wide pod access is the legitimate requirement) versus a per-namespace
RoleBinding pattern.

---

**Question 6**
During an RBAC audit you find a RoleBinding in the `production` namespace
that references a ClusterRole named `cluster-admin`. A developer explains it
was set up for a temporary debugging session six months ago. What are the
security implications and what do you do?

*A strong answer covers:* a RoleBinding referencing `cluster-admin` grants
cluster-admin permissions scoped to that namespace — not cluster-wide, but
all permissions on all resources within `production`. This is still excessive.
Immediate action: identify whether anything still depends on it (audit logs,
running workloads), replace with a scoped Role for the legitimate need, and
remove the binding. Notes that "temporary" RBAC bindings without an expiration
or review mechanism become permanent.
