# Lab 09: Detection Engineering And Threat Hunting

## Module Alignment

- Module doc: `docs/modules/module-09.md`
- Curriculum: Module 9
- Estimated time: 75 to 90 minutes

## What You Will Produce

- a detection matrix
- five candidate detections
- response linkage for each analytic

## Objective

Build practical Kubernetes detection content. Learn which telemetry sources
matter and why. Separate noisy events from high-confidence attacker signals
and produce five detections that are worth operationalizing in production.

## Prerequisites

- API server audit logging enabled (Lab 08)
- A SIEM or log query tool (Splunk, Elastic, CloudWatch Logs Insights, or
  BigQuery if using GKE)
- Optional: Falco deployed for runtime detection

## Files

| File | Purpose |
| --- | --- |
| `detection-matrix.md` | Detection content catalog with data source, logic, FP rate, and response |

## Step 1 — Inventory Your Telemetry Sources

For your cluster, confirm which of these sources are available and where
the logs are stored:

| Source | Available? | Log Destination | Latency |
| --- | --- | --- | --- |
| Kubernetes API server audit log | | | |
| Kubelet logs | | | |
| Container stdout/stderr | | | |
| Container runtime events (containerd/cri-o) | | | |
| Cloud control-plane logs (CloudTrail/Cloud Audit Logs) | | | |
| DNS query logs (CoreDNS or cloud DNS) | | | |
| VPC flow logs / cloud network logs | | | |
| Falco rule alerts | | | |
| Node syslog | | | |

## Step 2 — Query Your Audit Logs

Run the following queries against your audit log source. Adapt the syntax to
your query tool.

```
# Find all kubectl exec sessions in the last 24 hours
verb=create AND resource=pods/exec

# Find secret reads outside normal working hours
resource=secrets AND verb=get AND NOT user.username="system:*"

# Find RBAC changes
resource IN (clusterroles, clusterrolebindings, roles, rolebindings)
AND verb IN (create, update, patch, delete)

# Find new service account token requests
resource=serviceaccounts/token AND verb=create

# Find pod creations with privileged=true that bypassed policy
resource=pods AND verb=create AND requestObject.spec.containers.securityContext.privileged=true
```

For each query, record:
- How many results did you get?
- Were any results surprising?
- What is the false positive rate in your environment?

## Step 3 — Build Your Detection Matrix

Fill in `detection-matrix.md`. Complete at least five detections. Use the
following guidance for each field:

- **Analytic logic**: the query or rule condition, not just the concept
- **Expected FP rate**: Low / Medium / High, with one sentence on what causes FPs
- **Response playbook**: what the responder should do first when this fires

## Step 4 — Threat Hunt Exercise

Using your audit logs, hunt for evidence of these attacker techniques.
If you do not find real evidence, generate it deliberately in a test namespace
and then find it.

1. **Token theft**: a pod reads a service account token from a mounted secret
   then uses it from a different source IP
2. **Privilege escalation via RBAC**: a non-admin principal creates a
   ClusterRoleBinding to cluster-admin
3. **Suspicious exec**: a `kubectl exec` session into a production pod from
   an IP address not in your normal access range
4. **Crypto mining detection**: a pod creates outbound connections to known
   mining pool ports (3333, 4444, 14444)
5. **Namespace escape attempt**: a pod tries to access the metadata API at
   169.254.169.254

## Step 5 — Tune For Production

For each detection in your matrix, answer:

1. What benign behavior most commonly triggers this alert?
2. What is the minimum context you need to make the alert actionable?
3. What suppression or enrichment would reduce noise without hiding real attacks?

## Deliverable

A completed detection matrix with five or more detections, your query results
from Step 2, and a brief tuning plan for each detection.

## Assessment Question

A detection engineer proposes alerting on every `kubectl exec` command as a
high-severity incident. Describe how you would tune this detection to be
operationally useful — what additional context would you require, and what
conditions would you use to suppress known-good activity while preserving
sensitivity for real attacker behavior?
