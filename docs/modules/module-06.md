# Module 06: Admission Control And Policy-As-Code

## Why This Matters

Every control in the previous modules — pod hardening, network isolation,
secret delivery — can be bypassed if someone deploys a workload that ignores
them. A developer who does not know the security requirements deploys a
container with `privileged: true`. A CI pipeline that has broad RBAC creates
a pod with a hostPath mount. A third-party Helm chart pulls images from Docker
Hub. None of this is malicious. All of it creates risk. Admission control is
the mechanism that catches these misconfigurations before they reach the
scheduler — and unlike documentation or code review, it is enforced on every
deployment, by every team, regardless of whether they read the guidelines.

---

## The Attack Narrative

A company has a strong security policy documented in a wiki. The policy
prohibits privileged containers, requires images from the internal registry,
and mandates resource limits. The policy was written by the security team
and announced to all engineering teams at a quarterly all-hands.

Six months later, a new engineer joins a product team. He is onboarding quickly
and following a tutorial he found online. The tutorial uses `privileged: true`
because it is simpler for the demo. He deploys the workload to the production
cluster. It runs without issue. No one notices.

A Kubernetes security audit eighteen months later finds eleven privileged
containers in production. Four are pulling images from Docker Hub. Three have
no resource limits. None of this was discovered because no one was looking.
The wiki policy had no enforcement mechanism.

This is the gap admission control closes. A policy that is only in a document
is not enforced on deployments that happen at 11pm, by engineers who are
onboarding, by CI pipelines that run without human review. A policy that is
enforced at the API server is enforced everywhere, always.

---

## How Admission Control Works

When the Kubernetes API server receives a request to create or modify a
resource, it passes the request through a chain of admission controllers
before writing to etcd. There are two types:

**Validating admission controllers** inspect the request and either allow it
or reject it with an error. They do not modify the resource. A policy that
blocks privileged containers is a validating control — if the container spec
includes `privileged: true`, the request is rejected with a message explaining
why.

**Mutating admission controllers** can modify the request before it is
validated or stored. They are used to inject defaults — adding resource
limits to pods that do not specify them, injecting sidecar containers,
rewriting image tags to digests. Mutation happens before validation in the
admission chain, so a mutating webhook can correct a missing field before the
validating webhook checks for it.

**Webhook-based admission** is the mechanism used by Kyverno, OPA Gatekeeper,
and similar tools. The API server forwards the request to an external webhook
service that evaluates it against configured policies and returns an allow or
deny decision. The webhook is a Kubernetes deployment — if it becomes
unavailable, the API server's behavior depends on the webhook's `failurePolicy`:
- `Fail`: deny all matching requests if the webhook is unreachable. Secure but
  operationally risky — a Kyverno outage blocks all deployments.
- `Ignore`: allow all requests if the webhook is unreachable. Operationally
  safe but creates a window where policies are not enforced.

The right answer depends on the policy's criticality. Security-critical policies
(no privileged containers, approved registry only) should use `Fail`. Developer-
experience policies (resource limits suggested but not required) can use `Ignore`.

---

## The Misconception: Policy Existence Equals Enforcement

The most dangerous state in admission control is having policies configured
in audit mode and believing they are enforced. Audit mode logs violations
without blocking them. It is useful for measuring the blast radius of
enforcement before enabling it. It is not enforcement.

A cluster with fifty Kyverno policies all set to `validationFailureAction: Audit`
has zero preventive admission control. It has a detailed log of all the
violations that are occurring. This is a common state in organizations that
rolled out a policy engine, ran into violations from existing workloads, set
everything to audit "temporarily," and never switched to enforce because there
was always something more urgent.

The path from audit to enforce requires: inventorying violations, remediating
or documenting exceptions for each one, and then switching to enforce. It
requires someone to own the process and a deadline. Without those, audit mode
is permanent.

---

## Exception Management: The Control That Decays

Every exception to a policy is a permanent weakening unless it is managed.
An exception granted for a valid operational reason in January may still be
in place when that reason has expired in July. Accumulated exceptions turn a
strong policy into a nominally enforced one with dozens of holes.

Good exception management requires three things:

**Scope**: the exception should apply to the minimum workload that needs it.
A `PolicyException` that names a specific Deployment in a specific namespace
is better than one that names a whole namespace, which is better than one that
names a label that happens to match forty workloads.

**Justification**: the exception record should explain why the standard control
cannot apply. "Legacy application cannot run as non-root" is a justification.
"Too much work to fix right now" is a reason to push back on the exception
request and require a remediation timeline instead.

**Expiration**: every exception should have a review date. Quarterly review
of all active exceptions is a reasonable cadence. Exceptions past their review
date should generate automated notifications to the workload owner. Exceptions
that have been open for more than a year without renewal should be escalated.

Without these properties, exceptions accumulate. After two years a policy engine
with good initial coverage but poor exception management may have more
workloads exempted than covered.

---

## Decision Framework: What Belongs In Admission vs Elsewhere

Not every security requirement belongs in admission control. Admission is the
right place for controls that:
- Must apply to all workloads, with no possibility of being forgotten
- Can be evaluated at deploy time from the resource spec alone
- Should fail the deployment if violated, not produce a warning after the fact

Controls that belong in CI/CD rather than admission:
- Secret scanning in code and manifests (before they reach the cluster)
- Dependency vulnerability scanning (cannot be evaluated from a pod spec)
- SAST and code quality checks

Controls that belong in both CI/CD and admission:
- Image scanning — CI catches known vulnerabilities before push, admission
  can block images that fail the scan gate or that come from unapproved registries

Controls that belong in documentation or training rather than admission:
- Naming conventions that have no security impact
- Annotation requirements for cost allocation (unless you have a policy engine
  that validates them and a process for exceptions)

The principle: admission control should be reserved for requirements where
violation creates genuine security risk. Overloading the admission webhook with
low-value checks slows deployments, generates noise, and trains engineers to
view policy failures as bureaucratic friction rather than security signals.

---

## What Good Looks Like vs What Compliant Looks Like

A compliant cluster has a policy engine installed and some policies in audit
mode. A secure cluster has policies in enforce mode for every security-critical
requirement, an exception process that scopes and expires exceptions, a violation
SLA that is tracked and reported, and a rollout history showing that audit mode
was used for measurement — not as a permanent state.

The question to ask about any policy-as-code implementation is: "If I deploy
a privileged container right now, does it run or does it get blocked?" If the
answer is "it runs, but there is a log entry about it," the policy is not
protecting anything.

---

## You Will Learn

- how the Kubernetes admission chain works and where policy engines sit in it
- the difference between validating and mutating admission webhooks
- how to roll out policies from audit to enforce without breaking production
- what makes an exception justified, scoped, and temporary
- which security controls belong in admission versus CI/CD or documentation

## Key Questions

- Which controls belong in admission rather than in CI/CD or documentation?
- How do you move from audit to enforce safely in a cluster with existing violations?
- What does a justified, properly scoped exception look like?
- What happens to your policies when the webhook is unavailable?

## Hands-On

- Lab: [Lab 06](../../labs/module-06/README.md)
- Asset: [kyverno-policies.yaml](../../labs/module-06/kyverno-policies.yaml)

## Output

Produce an enforceable policy set with verified test results and a documented
exception workflow with scope, justification, and review date requirements.

---

## Self-Assessment

**Question 1**
You deploy Kyverno with five policies in `Audit` mode. Six months later, every
policy is still in audit mode. An audit finds eleven privileged containers in
production. What failed organizationally, and what process would have prevented
this?

*A strong answer covers:* audit mode produces violations logs but does not
block deployments — the organization never completed the audit-to-enforce
transition. What failed: no owner was assigned to drive the transition, no
deadline was set, no metric tracked how many violations were outstanding, and
no SLA defined how long a violation could exist before escalation. The process
that prevents this: define an audit period with a fixed end date, inventory
violations, require remediation or documented exceptions for each one by the
end date, then flip to enforce. Without those steps, audit mode becomes the
permanent state.

---

**Question 2**
A Kyverno policy uses `failurePolicy: Ignore` for a rule that blocks
privileged containers. The Kyverno pod crashes during a deployment window.
What happens?

*A strong answer covers:* when the admission webhook is unreachable and
`failurePolicy: Ignore` is set, the API server allows the request through
without policy evaluation. A privileged container deployment submitted during
the Kyverno outage would succeed. This is appropriate for low-criticality
policies but not for security-critical ones. Privileged container blocking,
approved-registry enforcement, and other security-critical policies should
use `failurePolicy: Fail` — the deployment fails rather than bypassing policy.
Operationally, this means Kyverno availability becomes a prerequisite for
any deployment, so Kyverno must be highly available.

---

**Question 3**
A developer submits a `PolicyException` request for their workload because
the image signature verification policy blocks their deployment. They say
their image was built by the internal CI but something went wrong with signing.
How do you handle this?

*A strong answer covers:* do not grant a broad exception — fix the signing
issue in CI instead. The correct response is to investigate why the build
did not produce a signed image (broken signing step, missing key access,
wrong attestation format) and fix the pipeline. If a temporary exception
must be granted during the fix, scope it to the specific Deployment, in the
specific namespace, with an explicit review date within 48-72 hours. Document
that it is a temporary exception for a pipeline issue. A correctly scoped
exception is preferable to disabling the policy cluster-wide or for the whole
namespace. Note that a pattern of signing exceptions indicates CI pipeline
reliability issues, not a policy problem.

---

**Question 4**
Your admission policy requires images to come from `registry.internal.company.com`.
A developer deploys a Helm chart that uses an init container pulling from
`docker.io/bitnami/wait-for-it`. The policy blocks it. The developer argues
this init container is harmless. How do you respond?

*A strong answer covers:* the policy is working correctly — the init container
is pulling from an unapproved registry. "Harmless" is a judgment that cannot
be verified without reviewing the image, which is the point of the policy.
The correct remediation is to mirror the init container image to the internal
registry, verify its contents, and update the Helm chart to reference the
internal copy. Do not grant a registry exception for Docker Hub — that path
leads to an expanding set of approved-exception registries that defeats the
control. Note that Helm charts often pull from public registries and require
mirroring for air-gapped or registry-controlled environments.

---

**Question 5**
A security review finds that your policy engine has 47 active PolicyExceptions.
Many are over a year old. What process do you put in place, and which exceptions
do you prioritize reviewing first?

*A strong answer covers:* prioritize exceptions that apply to the most
security-critical policies (privileged containers, host namespaces, registry
allowlists) and those scoped too broadly (entire namespaces rather than specific
workloads). For each exception: verify the workload still exists, verify the
original justification is still valid, check if the underlying issue has been
fixed and the exception is no longer needed, and renew or close with a new
review date. Process going forward: all new exceptions require a review date
no more than 90 days out, automated reminders go to the workload owner two
weeks before expiry, and exceptions not renewed by the review date are
automatically closed by the platform team.

---

**Question 6**
You are designing admission policies for a new cluster. Which controls belong
in the admission policy engine versus in CI/CD, and why? Give at least two
examples in each category.

*A strong answer covers:* admission policy is the right place for controls
that must block a deployment before it reaches the scheduler and that can be
evaluated from the pod spec alone — e.g., privileged container blocking,
approved registry enforcement, required resource limits. These cannot be
left to CI/CD because not all deployments go through CI (manual deploys,
GitOps sync, operator-created pods). CI/CD is the right place for controls
that require code analysis or external data not available at admission time —
e.g., secret scanning in manifests and code, dependency vulnerability scanning
(Trivy/Grype in CI), SAST. Both CI/CD and admission should enforce image
registry allowlists — CI ensures only approved images are pushed, admission
ensures only approved images can run. Explains that admission is defense-in-
depth for what CI is supposed to catch.
