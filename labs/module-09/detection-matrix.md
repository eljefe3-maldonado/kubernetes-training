# Kubernetes Detection Matrix

A catalog of detection content for Kubernetes environments.
Each row is a complete detection: data source, analytic logic, expected false
positive rate, triage steps, and response playbook linkage.

Build on this template. The goal is detections that are worth operationalizing —
high signal-to-noise, actionable on alert, and mapped to a real attacker technique.

---

## Detection Template

Use this structure for each new detection you add.

| Field | Description |
| --- | --- |
| ID | Sequential identifier (K8S-DET-NNN) |
| Name | Short descriptive name |
| MITRE ATT&CK Technique | Tactic and technique ID if applicable |
| Data Source | Where the signal comes from |
| Analytic Logic | The query or rule condition |
| Alert Severity | Critical / High / Medium / Low |
| Expected FP Rate | Low / Medium / High + one sentence on FP cause |
| Triage Steps | First three things the responder checks |
| Response Playbook | Link or inline steps |
| Notes | Tuning guidance, known gaps |

---

## Detections

### K8S-DET-001: Interactive Shell Opened In Production Container

| Field | Value |
| --- | --- |
| MITRE ATT&CK | Execution: T1609 Container Administration Command |
| Data Source | Kubernetes API server audit log |
| Analytic Logic | `verb=create AND resource="pods/exec" AND namespace IN (production, prod, app)` |
| Alert Severity | High |
| Expected FP Rate | Medium — legitimate debug sessions by oncall engineers during incidents |
| Triage Steps | 1. Identify the user and source IP. 2. Check if there is a linked incident ticket. 3. Review what commands were run (if runtime logging is available). |
| Response Playbook | Confirm with the engineer whether the session was authorized. If not, isolate the pod (see IR runbook). Rotate service account tokens for the namespace. |
| Notes | Suppress for known oncall IP ranges during active incidents. Alert on ALL exec sessions outside business hours regardless of suppression. |

---

### K8S-DET-002: Secret Read By Non-Platform Principal

| Field | Value |
| --- | --- |
| MITRE ATT&CK | Credential Access: T1552.007 Container API |
| Data Source | Kubernetes API server audit log |
| Analytic Logic | `verb=get AND resource=secrets AND NOT user.username=~"system:.*"` |
| Alert Severity | High |
| Expected FP Rate | Medium — legitimate operators reading secrets for debugging |
| Triage Steps | 1. Identify the user, service account, or token used. 2. Check if the user has a documented reason to access that specific secret. 3. Look for correlated activity (new pods, RBAC changes) in the same time window. |
| Response Playbook | If unauthorized, rotate the accessed secret immediately. Audit what else the same principal accessed. Review how the principal obtained access (RBAC gap, token theft). |
| Notes | Tune by adding known-good service accounts to a suppression list. Escalate immediately if the reading principal is a service account from a workload namespace (not a platform namespace). |

---

### K8S-DET-003: ClusterRoleBinding Created Or Modified By Non-Platform Principal

| Field | Value |
| --- | --- |
| MITRE ATT&CK | Privilege Escalation: T1078 Valid Accounts |
| Data Source | Kubernetes API server audit log |
| Analytic Logic | `verb IN (create, update, patch) AND resource=clusterrolebindings AND NOT user.groups=~"system:masters"` |
| Alert Severity | Critical |
| Expected FP Rate | Low — legitimate RBAC changes are infrequent and should be approved in advance |
| Triage Steps | 1. Identify the subject and the role being bound. 2. Check whether there is a change management ticket for this action. 3. Determine whether the new binding grants cluster-admin or equivalent. |
| Response Playbook | If unauthorized, delete the binding immediately. Suspend the principal that created it. Audit all cluster activity by that principal in the past 24 hours. Escalate to P1 if cluster-admin was granted. |
| Notes | This should almost never fire in a well-governed cluster. Any alert here deserves immediate review. |

---

### K8S-DET-004: Privileged Pod Created

| Field | Value |
| --- | --- |
| MITRE ATT&CK | Privilege Escalation: T1611 Escape to Host |
| Data Source | Kubernetes API server audit log |
| Analytic Logic | `verb=create AND resource=pods AND requestObject.spec.containers[*].securityContext.privileged=true` |
| Alert Severity | Critical |
| Expected FP Rate | Low — admission policies should block this; if it fires, a policy was bypassed |
| Triage Steps | 1. Identify the namespace and the creating principal. 2. Check whether a PolicyException exists for this workload. 3. Check whether admission control was active and why it did not block the pod. |
| Response Playbook | Delete the privileged pod. Investigate how it bypassed admission control (policy gap, webhook failure, exception abuse). Review node-level access from the pod before deletion if forensics are needed. |
| Notes | If admission control is working correctly, this should only fire for infrastructure DaemonSets with approved exceptions. Investigate any other case immediately. |

---

### K8S-DET-005: Service Account Token Requested From Workload Namespace

| Field | Value |
| --- | --- |
| MITRE ATT&CK | Credential Access: T1528 Steal Application Access Token |
| Data Source | Kubernetes API server audit log |
| Analytic Logic | `verb=create AND resource=serviceaccounts/token AND namespace NOT IN (kube-system, monitoring, platform)` |
| Alert Severity | High |
| Expected FP Rate | Medium — legitimate workloads that use projected tokens will generate these |
| Triage Steps | 1. Identify which service account token was requested and by whom. 2. Check whether the requesting workload should be calling the token API. 3. Look for downstream API calls using the issued token from unexpected IPs. |
| Response Playbook | If the request is unauthorized, revoke the issued token (delete the TokenRequest or rotate the service account). Audit subsequent API calls made with the token. |
| Notes | Tune by allowing known workloads (CI agents, platform controllers) that legitimately request tokens. Alert on any workload-namespace service account requesting tokens for service accounts in other namespaces. |

---

### K8S-DET-006: Cloud Metadata API Access From Pod

| Field | Value |
| --- | --- |
| MITRE ATT&CK | Discovery: T1552.005 Cloud Instance Metadata API |
| Data Source | VPC flow logs or cloud network logs |
| Analytic Logic | Outbound connection from pod CIDR to 169.254.169.254 on port 80 or 443 |
| Alert Severity | High |
| Expected FP Rate | Low — most workloads should not need direct metadata API access |
| Triage Steps | 1. Identify the pod and workload. 2. Determine whether the workload legitimately uses the metadata API (e.g., AWS SDK implicit credential chain). 3. Check what data was retrieved (if accessible via cloud logs). |
| Response Playbook | If unexpected, isolate the pod. Check whether a node IAM role has been used to obtain credentials. Rotate affected cloud credentials immediately. |
| Notes | Some cloud SDKs hit the metadata API for instance identity documents and region discovery even when not using IAM roles. Tune by allowing known SDK patterns while alerting on unusual access patterns. |

---

### K8S-DET-007: Namespace Created And Immediately Used

| Field | Value |
| --- | --- |
| MITRE ATT&CK | Defense Evasion: T1036 Masquerading |
| Data Source | Kubernetes API server audit log |
| Analytic Logic | Namespace created by a non-platform principal, followed within 5 minutes by workload creation in that namespace, from the same principal |
| Alert Severity | Medium |
| Expected FP Rate | Medium — developers with broad access may legitimately create test namespaces |
| Triage Steps | 1. Identify the principal and the new namespace name. 2. Review what was deployed into the namespace. 3. Determine whether this follows the organization's namespace provisioning process. |
| Response Playbook | If unauthorized, delete the namespace and its contents. Audit the creating principal's full activity. Review RBAC to determine how namespace creation was permitted. |
| Notes | Most organizations should not allow non-platform principals to create namespaces in production clusters. Tune by restricting namespace creation via admission policy and alerting on any policy bypass. |

---

## Threat Hunting Leads

Use these as starting points for periodic threat hunting exercises.

| Hunt | Data Source | Query Idea | What You Are Looking For |
| --- | --- | --- | --- |
| Lateral movement via service account token reuse | Audit log | Same token, different source IPs within short window | Token exfiltrated and used from attacker infrastructure |
| Persistence via CronJob | Audit log | CronJob created in non-platform namespace | Attacker establishing recurring execution |
| Data exfiltration via log flooding | Container logs | Unusually high log volume from a single container | Application logging secret values or config |
| Registry bypass | Audit log | Pod created with image from unapproved registry | Developer bypassing image trust policy |
| RBAC reconnaissance | Audit log | High volume of SelfSubjectAccessReview requests | Attacker enumerating their own permissions |
| DNS tunneling | CoreDNS logs | Unusually long or high-entropy DNS query names | Data exfiltration via DNS |
