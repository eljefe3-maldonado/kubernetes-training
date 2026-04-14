# Cluster Trust Map — Exercise Template

Fill in every table below based on your cluster. Leave nothing blank.
If a cell is unknown, write "unknown — investigate" so gaps are visible.

---

## Component Inventory

| Component | Namespace / Location | Port(s) | Auth Method | TLS? |
| --- | --- | --- | --- | --- |
| API server | kube-system | | | |
| etcd | | | | |
| kube-scheduler | kube-system | | | |
| kube-controller-manager | kube-system | | | |
| kubelet | node-local | | | |
| kube-proxy | node-local | | | |
| CoreDNS | kube-system | | | |
| Ingress controller | | | | |
| Container runtime | node-local | | | |
| Cloud controller manager | kube-system | | | |

---

## Communication Paths

| Source | Destination | Protocol | Credential Type | Attacker Impact If Path Is Compromised |
| --- | --- | --- | --- | --- |
| kubectl (admin) | API server | | | |
| kubectl (service account) | API server | | | |
| API server | etcd | | | |
| API server | kubelet | | | |
| API server | webhook (admission) | | | |
| kubelet | API server | | | |
| kubelet | container runtime | | | |
| Pod (default SA) | API server | | | |
| Pod (named SA) | API server | | | |
| Ingress controller | upstream pod | | | |
| Cloud controller | cloud API | | | |
| CI/CD system | API server | | | |

---

## Abuse Path Analysis

For each scenario, trace the attack step by step.

### Scenario 1 — Stolen Cluster-Admin Kubeconfig

- Starting point:
- First action:
- Escalation path:
- Maximum blast radius:
- Log entry that would appear (if any):
- Detection gap:

---

### Scenario 2 — Exposed Dashboard Without Authentication

- Starting point:
- First action:
- Escalation path:
- Maximum blast radius:
- Log entry that would appear (if any):
- Detection gap:

---

### Scenario 3 — Overly Privileged Service Account Token In A Pod

- Starting point:
- First action:
- Escalation path:
- Maximum blast radius:
- Log entry that would appear (if any):
- Detection gap:

---

### Scenario 4 — Compromised Node (Kubelet Credential Access)

- Starting point:
- First action:
- Escalation path:
- Maximum blast radius:
- Log entry that would appear (if any):
- Detection gap:

---

### Scenario 5 — Poisoned Image Via CI/CD Pipeline

- Starting point:
- First action:
- Escalation path:
- Maximum blast radius:
- Log entry that would appear (if any):
- Detection gap:

---

## Component Classification

Revisit your component inventory and mark each:

| Component | High-Value Target | Detection Opportunity | Hardening Candidate | Notes |
| --- | --- | --- | --- | --- |
| API server | | | | |
| etcd | | | | |
| kube-scheduler | | | | |
| kube-controller-manager | | | | |
| kubelet | | | | |
| CoreDNS | | | | |
| Ingress controller | | | | |
| Container runtime | | | | |
| Cloud controller manager | | | | |

---

## Summary

After completing the map, answer:

1. Which single component, if compromised, gives the broadest blast radius?
2. Which communication path has the weakest authentication today?
3. Where is your best detection opportunity if an attacker already has a foothold
   in a workload pod?
4. What is the one hardening change that would most reduce your attack surface?
