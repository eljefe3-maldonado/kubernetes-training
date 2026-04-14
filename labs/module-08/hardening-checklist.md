# Cluster Hardening Checklist

Mark each control: **Pass**, **Fail**, **N/A**, or **Provider-owned**.
Add notes in the Notes column for anything that needs follow-up.

---

## Control Plane

| # | Control | Status | Notes |
| --- | --- | --- | --- |
| CP-01 | API server has no anonymous authentication (`--anonymous-auth=false`) | | |
| CP-02 | API server uses TLS with a trusted CA | | |
| CP-03 | API server authorization mode includes RBAC and Node (`--authorization-mode=Node,RBAC`) | | |
| CP-04 | API server admission plugins include NodeRestriction | | |
| CP-05 | API server does not use the AlwaysAllow authorization mode | | |
| CP-06 | API server audit logging is enabled with an appropriate policy | | |
| CP-07 | API server audit log max age is at least 30 days | | |
| CP-08 | API server is not exposed on a public IP address | | |
| CP-09 | etcd has peer and client TLS authentication enabled | | |
| CP-10 | etcd is only accessible from the API server (network-level restriction) | | |
| CP-11 | etcd data is encrypted at rest (envelope encryption or provider KMS) | | |
| CP-12 | Kubernetes secrets encryption uses an external KMS key (not a local key) | | |
| CP-13 | Controller manager uses a service account credential for each controller | | |
| CP-14 | Scheduler binds only to localhost or a private interface | | |
| CP-15 | Control-plane components run with profiling disabled | | |

---

## Nodes And Kubelet

| # | Control | Status | Notes |
| --- | --- | --- | --- |
| ND-01 | Kubelet authentication uses x509 certificates or API bearer token (not anonymous) | | |
| ND-02 | Kubelet authorization mode is Webhook (delegates to the API server) | | |
| ND-03 | Kubelet read-only port is disabled (`--read-only-port=0`) | | |
| ND-04 | Kubelet streaming connection idle timeout is set | | |
| ND-05 | Kubelet protects kernel defaults (`--protect-kernel-defaults=true`) | | |
| ND-06 | Node OS is a minimal or container-optimized distribution | | |
| ND-07 | Node OS is patched to current minor version within 30 days of release | | |
| ND-08 | Container runtime socket is not mounted into workload containers | | |
| ND-09 | Nodes use immutable OS images or enforce image integrity | | |
| ND-10 | SSH access to nodes requires MFA or bastion and is logged | | |
| ND-11 | Node instance metadata API (169.254.169.254) is restricted to authorized workloads | | |
| ND-12 | Nodes are in private subnets with no direct internet ingress | | |

---

## Authentication And Authorization

| # | Control | Status | Notes |
| --- | --- | --- | --- |
| AA-01 | No ClusterRoleBinding to cluster-admin exists for non-system subjects | | |
| AA-02 | No wildcard verbs or resources in production-facing roles | | |
| AA-03 | Service accounts have `automountServiceAccountToken: false` unless required | | |
| AA-04 | All service account tokens are projected tokens with short TTLs | | |
| AA-05 | OIDC or cloud IAM is used for human user authentication (not static tokens) | | |
| AA-06 | kubeconfig files for admin access are stored in a secrets manager, not on disk | | |
| AA-07 | `kubectl` access to production is gated by MFA | | |
| AA-08 | Service accounts do not have permissions to create or modify RBAC resources | | |
| AA-09 | The `default` service account in each namespace has no RBAC bindings | | |

---

## Network

| # | Control | Status | Notes |
| --- | --- | --- | --- |
| NW-01 | A CNI plugin that enforces NetworkPolicy is installed | | |
| NW-02 | Default-deny NetworkPolicy is applied to all production namespaces | | |
| NW-03 | No namespace has unrestricted egress to 0.0.0.0/0 | | |
| NW-04 | Access to the cloud metadata API is restricted by NetworkPolicy or instance policy | | |
| NW-05 | Ingress controllers do not run with cluster-admin or host-network access | | |
| NW-06 | All ingress is TLS-terminated; no plaintext HTTP in production | | |
| NW-07 | Internal services not requiring external access have no NodePort or LoadBalancer | | |

---

## Workload Security

| # | Control | Status | Notes |
| --- | --- | --- | --- |
| WL-01 | Pod Security Admission is enabled and configured for all namespaces | | |
| WL-02 | All production namespaces enforce the Restricted Pod Security Standard | | |
| WL-03 | Kyverno or OPA Gatekeeper is installed and enforcing policies | | |
| WL-04 | Resource requests and limits are required on all workloads | | |
| WL-05 | LimitRanges define default limits for each namespace | | |
| WL-06 | ResourceQuotas exist for all production namespaces | | |
| WL-07 | All container images come from approved registries only | | |
| WL-08 | Images are pinned by digest in production Deployments | | |
| WL-09 | Image vulnerability scanning runs in CI and at admission | | |

---

## Logging And Detection

| # | Control | Status | Notes |
| --- | --- | --- | --- |
| LD-01 | API server audit logs are enabled and shipped to a central SIEM | | |
| LD-02 | Container runtime logs are collected and retained for at least 30 days | | |
| LD-03 | Cloud control-plane logs (CloudTrail, Cloud Audit Logs) are enabled | | |
| LD-04 | Alerts exist for: `kubectl exec`, secret access, RBAC changes, privileged pod creation | | |
| LD-05 | A runtime threat detection tool (Falco, cloud-native equivalent) is deployed | | |
| LD-06 | Log integrity is protected (logs shipped before they can be deleted from the node) | | |

---

## Gap Analysis Summary

Complete this table after running the checklist.

| Control # | Control Name | Current State | Risk Level | Remediation Action | Effort | Maintenance Window? |
| --- | --- | --- | --- | --- | --- | --- |
| | | | | | | |
| | | | | | | |
| | | | | | | |
| | | | | | | |
| | | | | | | |

**Risk levels**: Critical / High / Medium / Low  
**Effort**: Low (hours) / Medium (days) / High (weeks or architectural change)
