# Lab 08: Cluster Hardening And Platform Baselines

## Objective

Produce a cluster hardening checklist for one managed Kubernetes platform and
one self-managed pattern. Distinguish controls owned by the cloud provider from
controls owned by the customer. Run a gap analysis and prioritize remediation
by risk and implementation difficulty.

## Prerequisites

- Access to a cluster (managed or self-managed)
- kubectl and cloud provider CLI (aws/az/gcloud)
- Optional: kube-bench for automated CIS scanning

## Files

| File | Purpose |
| --- | --- |
| `audit-policy.yaml` | CIS-aligned API server audit policy |
| `hardening-checklist.md` | Structured checklist covering control-plane, node, and workload layers |

## Step 1 — Run CIS Benchmark Scan (Optional)

If you have kube-bench available:

```bash
# Run on a node
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml
kubectl logs -f job/kube-bench
kubectl delete job kube-bench
```

Record the FAIL and WARN findings for use in Step 3.

## Step 2 — Apply The Audit Policy

```bash
# On a self-managed cluster, copy the audit policy to the control plane node
# and configure the API server to use it:
#   --audit-policy-file=/etc/kubernetes/audit-policy.yaml
#   --audit-log-path=/var/log/kubernetes/audit.log
#   --audit-log-maxage=30
#   --audit-log-maxbackup=10
#   --audit-log-maxsize=100

# On managed clusters, audit logging is configured via the provider:
#   EKS: aws eks update-cluster-config --logging '{"clusterLogging":[{"types":["audit"],"enabled":true}]}'
#   GKE: enabled by default; configure log sink in Cloud Logging
#   AKS: az aks enable-addons --addons monitoring
```

After enabling audit logging, generate some events and review the log format:

```bash
kubectl get secrets -n kube-system
kubectl create namespace test-audit
kubectl delete namespace test-audit
```

Review the audit log entries and identify:
- Which events are captured at the `RequestResponse` level
- Which events are captured at `Metadata` level only
- Which events are omitted (health checks, watch keep-alives)

## Step 3 — Complete The Hardening Checklist

Work through `hardening-checklist.md`. For each control, mark:

- **Pass**: the control is in place and verified
- **Fail**: the control is missing or misconfigured
- **N/A**: not applicable for this cluster type
- **Provider-owned**: the cloud provider implements this; verify their documentation

Prioritize failures by:

| Priority | Criteria |
| --- | --- |
| Critical | Direct path to cluster-admin or node compromise |
| High | Enables privilege escalation or data exfiltration |
| Medium | Increases attacker capability but requires other conditions |
| Low | Defense-in-depth; reduces noise or limits blast radius |

## Step 4 — Managed vs Self-Managed Comparison

Fill in the table below for the platform you have access to.
If you have access to both, compare them.

| Control Area | EKS / GKE / AKS | Self-Managed (kubeadm) | Who Owns It |
| --- | --- | --- | --- |
| Control-plane OS patching | | | |
| etcd encryption at rest | | | |
| etcd access control | | | |
| API server TLS configuration | | | |
| Audit log delivery | | | |
| Node OS hardening | | | |
| Kubelet configuration | | | |
| Secrets encryption key management | | | |
| Network policy enforcement | | | |
| Add-on security updates | | | |

## Step 5 — Produce A Prioritized Gap Analysis

List the top five gaps you found. For each:

- Control name
- Current state
- Risk if not addressed
- Remediation action
- Effort to fix (Low / Medium / High)
- Whether it requires a maintenance window

## Deliverable

A completed hardening checklist with your gap analysis and a prioritized
remediation table.

## Assessment Question

A managed Kubernetes cluster has etcd encryption enabled by the cloud provider.
A security engineer argues this means secrets are protected. Explain two scenarios
where secrets could still be accessed despite etcd encryption at rest.
