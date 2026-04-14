# Module 09: Detection Engineering And Threat Hunting

## Why This Matters

An attacker in a Kubernetes cluster leaves traces. Every API call is logged.
Every process execution can be recorded. Every network connection can be
captured. The question is not whether the evidence exists — it almost always
does — but whether anyone is looking at it, and whether they have turned raw
log data into signal that is actionable within a response time that matters.
Most Kubernetes environments fail this test not because they have no logs but
because they have logs without analytics, or analytics that are so noisy that
real signals are buried.

---

## The Attack Narrative

An attacker compromises a workload pod through a dependency vulnerability.
She has a shell inside the container. The attack unfolds over four days:

**Day 1**: She reads the mounted service account token. She calls the
Kubernetes API: `kubectl auth can-i --list`. The service account can list
secrets cluster-wide. She runs `kubectl get secrets -A` and identifies a
CI deployment credential in the `ci-system` namespace. She reads it.

**Day 2**: Using the CI credential, she accesses the build pipeline. She
modifies a build step to exfiltrate environment variables from future builds.

**Day 3**: She creates a new service account in `kube-system`, names it
`metrics-controller` to blend in, and creates a ClusterRoleBinding to
`cluster-admin`. She then creates a pod in `kube-system` using the new
service account, named `kube-metrics-aggregator`.

**Day 4**: She uses the cluster-admin service account to read every secret
in the cluster. She has database credentials, API keys, and cloud provider
access keys. She exfiltrates them and begins lateral movement into the
broader cloud environment.

The audit logs contain evidence of every step. The API calls to list secrets,
the ClusterRoleBinding creation, the new pod in kube-system, the token
requests — all logged. But no one is querying the logs. There is no alert
on ClusterRoleBinding creation, no alert on pod creation in kube-system, no
alert on secret enumeration by a workload namespace service account. The
attacker has forty-seven days before a cloud cost anomaly triggers an
investigation.

When the investigation begins, the forensics team can reconstruct every step
from the audit logs. The logs were there. The detection was not.

---

## The Telemetry Sources That Matter

Not all log sources carry equal security signal. Understanding which sources
are worth investing in — and which produce noise without value — is the
foundation of detection engineering.

**Kubernetes API server audit logs** are the highest-value source for Kubernetes-
specific attacker behavior. Every API call — creating pods, reading secrets,
modifying RBAC, requesting service account tokens — is logged here. The
attacker in the narrative above left a complete record in the audit logs.
The limit of audit logs is that they record API calls, not what happens
inside containers. An attacker who compromises a pod and reads the service
account token from the filesystem appears in audit logs only when they use
that token to call the API.

**Container runtime telemetry and Falco** fills the gap that audit logs leave.
Falco uses eBPF or kernel modules to capture system calls made by container
processes — file opens, process executions, network connections, memory maps.
An attacker who reads `/var/run/secrets/kubernetes.io/serviceaccount/token`
from the filesystem appears in Falco as a `open_read` on that path. An
attacker who runs `wget` to download a tool appears as a `execve` of a process
that is not in the container's normal process tree. These signals are invisible
in audit logs and visible in runtime telemetry.

**Cloud control-plane logs** (AWS CloudTrail, GCP Cloud Audit Logs, Azure
Activity Log) capture API calls to the cloud provider. When an attacker uses
a stolen IRSA credential to call `s3:GetObject` from an IP address that is
not in the cluster's CIDR, the call appears in CloudTrail. Correlating
Kubernetes audit logs with cloud provider logs is how you detect credential
theft that leads to movement outside the cluster.

**VPC flow logs and DNS logs** are useful for detecting network-based attacker
behavior: unusual outbound connections, DNS tunneling, connections to known
malicious IPs. They produce high volume with low signal density and require
good baselining and anomaly detection to be actionable.

---

## Writing Detections That Are Worth Operationalizing

The test for any detection is: "If this alert fires, can a responder take
a meaningful action within fifteen minutes?" If the answer is no — because
the alert is too vague, fires too frequently, or does not include enough
context — the detection is not worth adding to an on-call rotation.

**High-confidence detections** are those where the signal is inherently
suspicious, regardless of context. A ClusterRoleBinding created by a non-
platform principal is always worth investigating. A `kubectl exec` session
from a source IP not in the known-good list is always worth investigating.
A new pod in `kube-system` created by a workload namespace service account
should never happen. These detections can be P1/P2 and page on-call.

**Medium-confidence detections** require context to distinguish malicious
from benign activity. A service account reading a secret is suspicious
in some contexts and normal in others. An outbound connection on port 4444
might be a crypto miner or might be a legitimate application. These detections
are useful for threat hunting but should not automatically page on-call until
they have been tuned with enough context to distinguish the cases.

**Low-confidence detections** are those that fire so frequently on benign
activity that they produce no actionable signal. "Any pod creation in a
production namespace" is technically an interesting event, but in a cluster
with continuous deployment it fires hundreds of times per day. Without
further filtering — pod creation by a service account that does not normally
create pods, at an unusual time, with an image not in the approved list —
it is noise.

The detection-matrix.md in Lab 09 provides the structure for documenting
each detection: data source, analytic logic, expected false positive rate,
triage steps, and response playbook. The last two are as important as the
logic itself. A detection that fires and has no documented triage steps
generates responder confusion rather than security value.

---

## The Misconception: Audit Logs Equal Detection

"We have audit logging configured" is a common answer to questions about
Kubernetes detection capability. Audit logging is a prerequisite for detection,
not detection itself.

Raw audit logs contain millions of events. Finding the three events that
indicate an attacker in those millions requires either real-time queries
with alert rules, a SIEM with correlation logic, or someone proactively
hunting through the logs. None of these happen automatically when you enable
audit logging.

The gap between "we have audit logs" and "we have detections" is the gap
between evidence existing and evidence being found. Both matter. An attacker
who knows they will not be detected can move slowly, quietly, and thoroughly.
An attacker who knows they will trigger an alert within minutes moves fast
or not at all.

---

## Threat Hunting vs Reactive Detection

Detection engineering produces alerts that fire automatically. Threat hunting
is the proactive investigation of logs looking for evidence of compromise that
existing detections did not catch.

Hunting complements detection. It finds attacker techniques that are too low-
signal for automated alerts — things that are individually normal but together
suggest a reconnaissance or lateral movement pattern. It also finds gaps in
detection coverage: if a hunt finds evidence of activity that should have
triggered an existing alert but did not, the detection needs to be fixed.

A basic Kubernetes threat hunt starts with three questions: Which service
accounts have made unusual API calls in the last 30 days? Which pods were
created in the last 30 days that were not created by a known controller?
Which ClusterRoleBindings or RoleBindings were created or modified in the
last 90 days? The answers rarely produce clear incidents, but they regularly
surface the kind of drift — overprivileged accounts, unexpected workloads,
undocumented RBAC changes — that attackers exploit.

---

## What Good Looks Like vs What Compliant Looks Like

A compliant environment has audit logging enabled. A mature detection program
has audit logs shipped to a SIEM with retention, specific alert rules for
high-confidence Kubernetes attacker behaviors, a runtime detection tool
(Falco) with custom rules for container-level signals, a documented detection
matrix with response playbooks for every alert, and a regular threat hunting
cadence that tests detection coverage and surfaces gaps.

The number that matters is not "how many detections do we have?" but "if an
attacker runs `kubectl get secrets -A` in our cluster right now, how long
before someone knows?" In a mature program, the answer is minutes. In a
typical environment, it is never.

---

## You Will Learn

- which telemetry sources carry security signal and which produce noise
- how to write detections tied to specific attacker behaviors rather than general anomalies
- the difference between audit-log-based and runtime (syscall-level) detection
- how to balance signal strength against false positive rate
- the difference between automated detection and proactive threat hunting

## Key Questions

- Which events in your cluster today would indicate an active attacker?
- What is the difference between having audit logs and having detection capability?
- Which detections justify paging someone at 2am?
- If an attacker has been in your cluster for 30 days, what evidence would they have left?

## Hands-On

- Lab: [Lab 09](../../labs/module-09/README.md)
- Asset: [detection-matrix.md](../../labs/module-09/detection-matrix.md)

## Output

Produce five high-confidence detections with data source, analytic logic, false
positive rate, triage steps, and response playbook linkage.

---

## Self-Assessment

**Question 1**
A team says: "We have audit logging configured, so we have detection coverage."
What is the gap between having audit logs and having detection capability,
and what would you need to add to close it?

*A strong answer covers:* audit logs contain millions of events. Without
analytics, the evidence of an attack exists in the logs but is never found.
Detection capability requires: (1) logs shipped to a queryable destination
(SIEM, CloudWatch Logs with Insights, Splunk), (2) specific alert rules that
match high-confidence attacker behaviors rather than generic anomaly, (3)
documented triage steps for each alert so responders know what to do when
it fires, (4) regular testing to verify alerts actually fire on the behaviors
they target. Audit logging is a prerequisite; the detection is the layer above
it. "We have logs" means the evidence exists. "We have detection" means someone
or something is looking at it in time to respond.

---

**Question 2**
Write the detection logic for: "A service account from a workload namespace
reads secrets in `kube-system`." What is the data source, what makes this
high-confidence, and what are the triage steps?

*A strong answer covers:* data source — Kubernetes API server audit logs.
Logic: filter audit log records where `verb=get`, `resource=secrets`,
`objectRef.namespace=kube-system`, and `user.username` starts with
`system:serviceaccount:` followed by a non-system namespace. This is
high-confidence because workload service accounts should never need access
to `kube-system` secrets — those secrets contain cluster infrastructure
credentials. Triage: identify the pod using the service account, check
whether the service account has any legitimate reason for this access (it
should not), check whether the service account was recently granted this
RBAC (indicates potential escalation), and check the pod for indicators of
compromise. Expected false positive rate: near-zero in a properly governed
cluster.

---

**Question 3**
A Falco rule fires: a `wget` binary was executed inside the `payments-api`
container. The development team says this is impossible because their container
has a read-only filesystem. How do you triage this?

*A strong answer covers:* Falco detects syscall activity — it sees the
`execve` system call regardless of whether the filesystem is read-only
or what the container spec says. The read-only filesystem prevents writing
new binaries, but `wget` may already exist in the base image. Triage steps:
check the container image for whether `wget` is present (`docker run --rm
image which wget`), check the Falco event for the full process tree (what
parent process spawned `wget`), check network connections made around the
time of the alert for the destination URL. If the application should never
run `wget`, this is a high-confidence indicator of compromise regardless
of how the attacker is doing it.

---

**Question 4**
You want to detect privilege escalation via ClusterRoleBinding creation.
An alert fires for every ClusterRoleBinding creation. Within one week the
on-call team has disabled the alert because it fires dozens of times daily
from the GitOps operator. What went wrong, and how do you fix the detection?

*A strong answer covers:* the alert fires on all ClusterRoleBinding creation
without filtering out expected principals. GitOps operators (ArgoCD, Flux)
legitimately create and update RBAC objects as part of normal operation.
The fix: filter the alert to fire only when the creating principal is not
in the known-good list of principals allowed to create ClusterRoleBindings
(the GitOps service account, the platform team's CI, specific controllers).
Alert on ClusterRoleBinding creation by `user.username` values that are not
in the allowlist. This eliminates the GitOps noise while preserving the
signal for unexpected principals. The broader lesson: alerts must be tuned
to the specific environment, not generic event type matches.

---

**Question 5**
An attacker has been in your cluster for 30 days with cluster-admin access
before being discovered. If you have configured audit logging at `RequestResponse`
for secrets and RBAC events with 365-day retention, what can you reconstruct
about their activity?

*A strong answer covers:* with audit logging at `RequestResponse` for secrets,
you can reconstruct: every secret they accessed by name and when, every
RBAC binding they created or modified (with the full binding spec), every
pod they created (pod spec captured), every namespace they accessed. You can
build a timeline of their escalation path. What you cannot reconstruct from
audit logs alone: what happened inside containers (process execution, file
reads, memory contents) — that requires Falco or runtime telemetry if it
was configured. Explains that this is why `RequestResponse` level is worth
the storage cost for the specific high-value event types.

---

**Question 6**
Explain the difference between a detection that should page on-call at 2am
versus a detection that should feed a threat hunting dashboard. Give a specific
example of each and explain your reasoning.

*A strong answer covers:* a 2am page is justified when: the signal has near-
zero legitimate false positive rate, the scenario it detects is a critical
security event that requires immediate human response, and delay makes the
situation materially worse. Example: ClusterRoleBinding created by a workload
namespace service account — this should never happen legitimately and indicates
an active escalation attempt requiring immediate response. A threat hunting
dashboard is appropriate when: the signal requires context to distinguish
malicious from benign, the base rate of legitimate occurrences is non-zero,
or the scenario allows for investigation over hours rather than minutes.
Example: a service account reading a secret it is permitted to read but does
not normally access — legitimate 90% of the time, worth hunting monthly.
The principle: page when false inaction is costly; hunt when false positives
are costly.
