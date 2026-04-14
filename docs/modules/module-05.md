# Module 05: Secrets And Sensitive Data Exposure

## Why This Matters

Secrets are the keys to everything else. A leaked database password gives an
attacker access to the data tier without needing to touch a pod. A leaked
cloud credential gives them access to infrastructure outside the cluster.
A leaked CI token gives them the ability to inject code into the build
pipeline. The paradox of secrets management is that the controls that make
secrets easy to use — environment variables, broad RBAC, centralized storage —
are also the controls that make them easy to leak. This module is about
understanding every path a secret travels and eliminating the paths that are
unnecessarily risky.

---

## The Attack Narrative

A development team uses a Deployment manifest to configure their application.
A junior engineer, following an internal wiki tutorial, adds the database
password as an environment variable directly in the Deployment spec. The
manifest goes into git. The git repository is private, but six months later
the company's GitHub organization is audited and it turns out that one team
member had accidentally made the repo public for two hours during a demo.
GitHub's secret scanning found the credential and notified the company.

But someone else found it first. A threat actor's automated scanner had
indexed the repository during those two hours. The database password was
already being tested against exposed endpoints by the time the security team
received the notification.

This scenario is one source. Here are four others that happen without any
git exposure:

An attacker compromises a pod with the service account token. The service
account has `list secrets` permission on the namespace. They call the
Kubernetes API, list all secrets, and decode the base64 values. The application
developer never intended to grant this access — it came from a ClusterRole
that was "simplified" to list everything.

A developer is debugging a production issue and runs `kubectl describe pod
api-server-abc`. The output includes the environment variables, including
`DB_PASSWORD: supersecret`. The terminal output is automatically logged to
a SIEM that indexes it.

An application starts up and logs its configuration for observability. The
logger iterates over `os.environ()` and prints every variable. The database
password goes into CloudWatch Logs where it is indexed, retained, and
searchable indefinitely.

An etcd backup job writes a snapshot to an S3 bucket. The bucket has a
misconfigured bucket policy that allows public reads. Every secret in the
cluster, base64-encoded in plaintext, is now publicly accessible.

All five of these paths are real. All five are preventable with controls that
are covered in this module.

---

## Why "It's In A Kubernetes Secret" Is Not Enough

The most common misconception about Kubernetes secrets is that storing data
in a `Secret` object means it is encrypted. It is not. By default, Kubernetes
secrets are stored in etcd as base64-encoded plaintext. Base64 is encoding,
not encryption — any process that can read the etcd data can decode the values
with a single command.

This has two practical implications. First, anyone with `get` or `list` access
to the Secret resource via the Kubernetes API can read the value. RBAC controls
who can read secrets through the API. Second, anyone with access to the etcd
data directly — through a backup, a misconfigured etcd endpoint, or access to
the control-plane node — can read all secrets without going through the API or
RBAC at all.

Encryption at rest (etcd envelope encryption via KMS) addresses the second
path. It protects secrets in etcd backups and in etcd storage on disk from
an attacker who obtains the raw data. It does not protect secrets from an
attacker who reads them through the Kubernetes API with a valid credential.
The API server decrypts secrets before returning them to authorized callers.
Encryption at rest is one layer of a multi-layer control, not a solution.

---

## The Seven Leak Vectors You Must Know

**Environment variables** expose secrets to every process in the container,
to crash dumps and core files, to `/proc/<pid>/environ` on the node (readable
by root or privileged processes), and to `kubectl describe pod` output which
appears in audit logs and developer terminals. They are the highest-risk
delivery mechanism for secrets that does not involve a full security bypass.

**List and watch permissions on secrets** allow an attacker with the service
account token to enumerate and stream every secret in scope. `get` with
`resourceNames` restriction is the correct permission — it allows reading
one specific named secret and nothing else. `list` reveals all secret names.
`watch` streams real-time updates to all secrets including future ones created
after the attacker obtains the token.

**Application startup logging** is a silent risk. Many frameworks log their
configuration on startup for observability. If the configuration includes
secrets — whether from environment variables, config files, or SDK calls —
those values end up in logs that may be indexed, retained, and searched by
anyone with log access.

**Mounted secrets with default permissions** use `defaultMode: 0644`, making
the file world-readable inside the container. Any process running in the same
container can read it. Setting `defaultMode: 0400` restricts reads to the
file owner only.

**etcd backups** are often forgotten in the secret security model. A
well-configured cluster with strict RBAC on secrets and etcd encryption
at rest can still expose every secret if the etcd backup job writes to an
S3 bucket with a permissive policy, or if the backup is downloaded and
decrypted using the encryption key stored alongside it.

**Hardcoded values in manifests and code** are committed to source control
and included in container images. They persist in git history even after
the line is removed. They appear in audit logs when the Deployment is
created. They survive secret rotation because the code still references the
old value.

**Projected volume automounting** — the default `automountServiceAccountToken: true`
behavior — means every pod receives a service account token whether it needs
one or not. That token can be used to call the Kubernetes API. If the service
account has secret read permissions, the mounted token is effectively a secret
reader for any process that runs in the pod.

---

## The Decision Framework: Which Secret Pattern Fits Which Requirement

**Native Kubernetes secrets** are appropriate for secrets that are managed by
Kubernetes tooling (cert-manager writing TLS certificates, operators managing
credentials they create). They work well when etcd encryption at rest is
enabled, access is controlled by tight `resourceNames`-scoped RBAC, and the
secret values are rotated through the Kubernetes API. They are not appropriate
for application secrets in regulated environments where audit trails, rotation
automation, and centralized governance are required.

**External Secrets Operator with a cloud secret manager** (AWS Secrets Manager,
GCP Secret Manager, Azure Key Vault) is the right pattern for most application
secrets. The secret value lives outside the cluster in a system designed for
secret management: versioning, rotation, audit logging, and fine-grained access
control are native features. The ESO syncs the value into a Kubernetes Secret
at a configurable refresh interval. Rotation in the external system propagates
to the cluster automatically without modifying any Kubernetes resources. This
is the pattern to default to for regulated workloads.

**CSI Secrets Store Driver** delivers secrets directly from a secrets manager
into a pod as a volume mount, without creating a Kubernetes Secret object at
all. The secret is never written to etcd. For the most sensitive values —
signing keys, high-value credentials — this eliminates the etcd exposure vector
entirely. The tradeoff is operational complexity: the secret is available only
while the pod is running, and rotation requires a pod restart to pick up the
new value.

**Direct SDK calls using workload identity** are appropriate for cloud
credentials. A pod with an IRSA role or GKE Workload Identity binding can
call the AWS or GCP APIs directly using short-lived, automatically rotated
credentials without ever materializing a credential into a Kubernetes Secret
or a file. This is the correct pattern for accessing S3, Secrets Manager,
or other cloud services from within a workload.

The choosing principle: the shorter the path from secret store to application
process, with the fewest intermediate copies, the smaller the attack surface.
An environment variable is a copy. A Kubernetes Secret is a copy in etcd. A
mounted file is a copy on disk in the container. Direct API calls with
workload identity are not copies at all.

---

## What Good Looks Like vs What Compliant Looks Like

A compliant cluster might use Kubernetes Secrets, have etcd encryption enabled,
and have RBAC that includes secret permissions. A secure cluster has all of
that, plus: secrets delivered via volume mounts with `0400` permissions, not
environment variables; RBAC scoped to `get` on named secrets only; application
code that does not log secret values; an external secret manager for application
credentials with automated rotation; and etcd backups stored in encrypted
storage with access controls that are as strict as the cluster's RBAC.

The test for a secure secrets posture is not "are our secrets in Kubernetes
Secrets?" — it is "can I enumerate the paths a secret takes from its origin
to the application process, and have I eliminated the unnecessary ones?"

---

## You Will Learn

- why Kubernetes secrets are not encrypted by default and what encryption at rest actually protects
- the seven paths through which secrets leak in real clusters
- RBAC scoping for secret access using `resourceNames`
- how to choose between native secrets, ESO, CSI, and direct SDK calls
- how to deliver secrets safely using volume mounts with correct permissions

## Key Questions

- Where do secrets leak first in real clusters, and which of those paths does your current setup leave open?
- Which secret delivery mechanism creates the fewest intermediate copies of the secret value?
- What does etcd encryption at rest protect against, and what does it not protect?
- How do you detect a secret that has already leaked?

## Hands-On

- Lab: [Lab 05](../../labs/module-05/README.md)
- Assets:
  - [leaky-deployment.yaml](../../labs/module-05/leaky-deployment.yaml)
  - [safer-secret-delivery.yaml](../../labs/module-05/safer-secret-delivery.yaml)

## Output

Produce a safer secret-delivery design and document every leak vector in the
original approach with a specific explanation of how each one was closed.
