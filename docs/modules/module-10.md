# Module 10: Incident Response And Forensics

## Why This Matters

Kubernetes incidents are easy to make significantly worse with the wrong first
response. The instinct to "delete the bad thing" destroys evidence, prevents
accurate scope determination, and can leave attacker persistence mechanisms
in place that are discovered weeks later. Response discipline — the ability
to sequence actions correctly, preserve what matters, contain without
destroying, and rotate credentials in the right order — is what separates
an effective incident response from one that closes the visible symptom while
the underlying compromise continues.

---

## The Attack Narrative — From The Responder's Perspective

An alert fires at 11:47 PM: `kubectl exec` session opened in a production pod
from an IP address that is not in the known-good list. The on-call engineer
gets the page. Here is what happens in two parallel universes.

**Universe A — Wrong first response:**

The engineer is not sure what to do. The obvious move is to remove the threat,
so she deletes the suspicious pod. The pod is gone. The alert stops. She
marks the incident as resolved and goes back to sleep.

What she did not know: the attacker had been in the cluster for 72 hours
before the exec session triggered the alert. In those 72 hours, they had
created a new service account in `kube-system`, created a ClusterRoleBinding
granting cluster-admin, and created a second pod — named to blend in — that
they still control. Deleting the first pod did nothing. The attacker's
persistence mechanism is intact. The cluster is still compromised.

Three weeks later, the attacker uses their persistent access to exfiltrate
the customer database. When forensics is requested, the key evidence — the
original pod's logs, its network connections, the service account token it
used — is gone. The deleted pod's log was on the node and was rotated out
after seven days. The blast radius of the original compromise cannot be
determined.

**Universe B — Right first response:**

The engineer receives the alert and opens the playbook. First: preserve
evidence before taking any action. She captures the pod spec, current logs,
and a list of all resources in the namespace. She queries the audit logs for
every API call made by the pod's service account in the last 24 hours. She
does not delete anything yet.

The audit log query shows the service account read secrets, created a new
service account in `kube-system`, and created a ClusterRoleBinding. She now
knows the scope includes a cluster-admin privilege escalation. She escalates
to P1 and wakes the security lead.

They apply a default-deny NetworkPolicy to isolate the original pod without
deleting it. They revoke the malicious ClusterRoleBinding. They identify and
isolate the second pod in `kube-system`. Only after all persistence mechanisms
are identified and all evidence is captured do they begin deletion and credential
rotation — in a specific order: RBAC first, then service accounts, then secrets,
then workloads.

The outcome is different not because the responder was smarter. It was because
she had a playbook that told her what to do in what order, and she followed it.

---

## The Sequencing Problem

Incident response in Kubernetes has a sequencing problem that does not exist
in the same form in traditional environments. The actions you take interact
with each other in ways that can destroy evidence or leave persistence intact
if done in the wrong order.

**Evidence before action.** Before deleting, isolating, or rotating anything,
capture the state of the environment. This means: `kubectl get pod -o yaml`,
`kubectl logs`, a list of all resources in affected namespaces, and audit log
queries for the principals involved. Container logs exist only as long as the
pod exists and the node hasn't rotated them. A pod deleted in the heat of
response takes its evidence with it.

**Isolation before deletion.** Applying a NetworkPolicy that blocks all
traffic to and from a suspicious pod preserves the pod for forensics while
removing its ability to communicate. This is the correct intermediate state
during investigation. Deletion is irreversible; isolation is not.

**RBAC before credentials.** When a service account has been compromised,
the first action is to revoke the elevated access it has acquired — deleting
malicious ClusterRoleBindings and removing from groups — before rotating the
service account itself. Rotating the service account while malicious RBAC
remains leaves the attacker's escalation path available to any other principal
they may have compromised.

**Credentials before workloads.** After RBAC is cleaned up and before
workloads are restarted, rotate every credential that the compromised service
account could have read. This includes secrets it had `get` access to, any
cloud credentials available through the mounted token, and any secrets in
namespaces it had access to. Restarting workloads before rotating their
credentials means they restart with the same credentials the attacker has.

---

## What Evidence Disappears Fastest

Knowing what to preserve first requires knowing what disappears fastest.

**Container logs** are ephemeral. They are stored on the node in
`/var/log/containers/` (or the runtime equivalent) and are subject to log
rotation. Once a pod is deleted, logs are typically gone within hours to days
depending on the node's log rotation configuration. If you have a centralized
log aggregator (Fluent Bit to CloudWatch, for example), logs may already be
preserved there — but verify before deleting the pod.

**Process state and memory** are gone the moment the container is killed.
If you need to capture a memory dump, network connections, or open file
handles, you must do it while the pod is running. An ephemeral debug container
(`kubectl debug`) attached to the running pod is the way to do this without
exec-ing into the compromised process.

**Kubernetes events** are retained for about one hour by default. Events
record scheduling decisions, image pull operations, and probe failures —
useful context for reconstruction. Query them early.

**Audit logs** are retained according to your audit log policy and retention
configuration. If you have configured 365-day retention in CloudWatch or a
SIEM, they are available indefinitely. If audit logging is not configured
(Module 8 failure), there is nothing to reconstruct from.

---

## Persistence Mechanisms To Search For

An attacker who has had cluster-admin for any length of time may have
established multiple persistence mechanisms. Closing the visible entry point
without searching for others leaves the compromise ongoing.

**ClusterRoleBindings with unusual subjects**: run
`kubectl get clusterrolebindings -o json | jq '.items[] | select(.subjects[]?.namespace != null) | {name, role: .roleRef.name, subjects}'`
to find cluster-admin bindings that reference workload namespace service accounts.

**New service accounts in kube-system**: platform components in kube-system
are stable. A service account created in the last 30 days that was not
created by a known platform tool is suspicious.

**CronJobs or Jobs in unexpected namespaces**: attackers use CronJobs to
maintain periodic access even if their pod is deleted. `kubectl get cronjobs -A`
with particular attention to namespaces that should not have them.

**Modified admission webhook configurations**: a `MutatingWebhookConfiguration`
or `ValidatingWebhookConfiguration` that the attacker has modified can disable
policy controls or redirect admission requests to an attacker-controlled endpoint.

**GitOps repository modifications**: if the cluster is managed by ArgoCD
or Flux, the attacker may have committed changes to the GitOps repository.
Check recent commits to the repository for unexpected changes.

---

## What Good Looks Like vs What Compliant Looks Like

A compliant organization might have an incident response policy document.
A mature Kubernetes incident response capability has: documented playbooks
that sequence actions correctly for specific scenarios (namespace compromise,
credential theft, privilege escalation), on-call engineers who have run
the tabletop and know what to do without reading from scratch at midnight,
centralized log retention that makes audit log queries possible during an
incident, and a post-incident review process that improves controls rather
than just closing tickets.

The distinguishing characteristic of master-level incident response is not
speed — it is discipline. Moving slowly in the right sequence produces better
outcomes than moving fast in the wrong one. The attacker has usually had hours
or days. You have time to think before you act. Use it.

---

## You Will Learn

- why the first response action in a Kubernetes incident is often evidence preservation, not remediation
- how to isolate workloads without destroying forensic state
- the correct sequencing of RBAC revocation, credential rotation, and workload deletion
- which evidence disappears fastest and must be captured first
- how to search for persistence mechanisms before declaring an incident closed

## Key Questions

- What is the first action when a suspicious pod alert fires, and why?
- Which evidence disappears if you delete the pod immediately?
- In what order do you revoke access, rotate credentials, and restart workloads?
- What persistence mechanisms would you search for in a cluster that may have had a cluster-admin compromise?

## Hands-On

- Lab: [Lab 10](../../labs/module-10/README.md)
- Assets:
  - [tabletop-scenario.md](../../labs/module-10/tabletop-scenario.md)
  - [attacker-pod.yaml](../../labs/module-10/attacker-pod.yaml)

## Output

Produce a namespace-compromise runbook that sequences evidence preservation,
isolation, RBAC revocation, credential rotation, and workload remediation
with explicit justification for the order of operations.

---

## Self-Assessment

**Question 1**
An alert fires at 11:47 PM indicating a `kubectl exec` session in a production
pod from an unknown IP. Your first instinct is to delete the pod immediately
to stop the attacker. What is wrong with this action, and what should you do
first?

*A strong answer covers:* deleting the pod immediately destroys the evidence
needed to determine scope, removes the only record of what the attacker did
inside the container (process execution, file access, network connections),
and does nothing about persistence mechanisms the attacker may have already
established. The correct first action: preserve evidence — capture the pod
spec (`kubectl get pod -o yaml`), current logs (`kubectl logs`), and a list
of all resources in the namespace. Then query audit logs for every API call
made by the pod's service account. Only after understanding scope should you
move to isolation (NetworkPolicy default-deny on the pod) rather than deletion.
Isolation stops the attacker's communication while preserving forensic state.

---

**Question 2**
During a cluster-admin compromise investigation, you find a ClusterRoleBinding
granting `cluster-admin` to `system:serviceaccount:reporting:analytics`. You
delete the binding. Thirty minutes later, the attacker still has cluster-admin
access. What did you miss?

*A strong answer covers:* deleting one persistence mechanism does not mean
all are gone. A sophisticated attacker with cluster-admin will create multiple
persistence paths. The investigation should search for: other ClusterRoleBindings
to cluster-admin or high-privilege roles, new service accounts in `kube-system`
created in the last N days, CronJobs or Jobs in unexpected namespaces, modified
admission webhook configurations, and changes to the GitOps repository if
applicable. The attacker likely created a second service account and binding
while they had access. The correct process is to search for all persistence
mechanisms before removing any, so that the removal is comprehensive.

---

**Question 3**
You are rotating credentials after a service account compromise. You rotate
the service account token first, then rotate the database password the service
account had access to, then restart the workloads. What is the problem with
this sequence?

*A strong answer covers:* rotating the service account token first invalidates
the stolen token but leaves the database password unrotated for the duration
of the credential rotation window. If the attacker already copied the database
password (which they would have done immediately upon gaining access), they
can use it to connect to the database directly while you are rotating other
credentials. The correct sequence: revoke elevated RBAC first (any malicious
bindings), then rotate the database password (and all secrets the compromised
account could access), then rotate the service account itself, then restart
workloads with new credentials. Rotate the highest-value secrets first, not
the stolen credential first.

---

**Question 4**
A pod was deleted by the on-call engineer immediately after an alert fired.
Twelve hours later, you are asked to determine whether the attacker read
the database password. What sources can you still query, and what is
unrecoverable?

*A strong answer covers:* recoverable from centralized log aggregation (if
configured): container logs shipped to CloudWatch or a SIEM before the pod
was deleted. Recoverable from Kubernetes audit logs: API calls made by the
pod's service account — if the service account called `get secrets` on the
secret containing the database password, that is recorded in the audit log.
Unrecoverable: what happened inside the container (file reads, process
execution, memory) if no runtime telemetry (Falco) was in place and container
logs were not shipped before deletion. If the attacker read the mounted secret
file directly from the filesystem without making an API call, there is no
audit log entry for that access — only Falco syscall monitoring would have
captured it.

---

**Question 5**
After closing an incident, your post-incident review identifies that the
attacker had cluster-admin for six days before detection. You are asked to
determine the full blast radius. What is your process?

*A strong answer covers:* with six days of cluster-admin, assume the attacker
has read every secret in every namespace. Query audit logs for every `get
secrets` call by the compromised principal over those six days to confirm.
For each secret they accessed, identify the system it provides access to and
initiate rotation. Check CloudTrail (or equivalent cloud logs) for API calls
made using cloud credentials that may have been obtained through IRSA or
mounted cloud credentials — the blast radius may extend outside the cluster.
Check for persistence mechanisms (other service accounts, CronJobs, modified
webhooks). Check the GitOps repository for changes. The blast radius is
everything the attacker could have read plus everything those credentials
could have accessed outside the cluster.

---

**Question 6**
A junior engineer on your team says: "We should automate incident response
so pods are automatically deleted when a suspicious alert fires." What is
your response?

*A strong answer covers:* automated remediation that deletes pods destroys
evidence, may delete legitimate workloads during false positives, and does
not address persistence mechanisms. A false positive that auto-deletes a
production pod causes an availability incident worse than the original security
alert. Better automated responses: (1) apply a default-deny NetworkPolicy to
isolate the pod — stops attacker communication while preserving evidence; (2)
capture pod logs and spec automatically to a retention store before any human
action; (3) revoke elevated RBAC bindings if the automation can determine
with high confidence which ones are malicious. Deletion should remain a human
decision made after scope is understood. Automation should assist evidence
preservation and isolation, not remediation.
