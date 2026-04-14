# Study Guide

This guide turns the curriculum into a practical self-study path. It is aligned
to the module sequence in `MASTER_CLASS_CURRICULUM.md` and the lab assets under
`labs/`.

## How To Use This Guide

For each week:

1. Read the matching module in `MASTER_CLASS_CURRICULUM.md`.
2. Complete the linked lab in `labs/`.
3. Write short notes for:
   - key risks
   - preventive controls
   - detections
   - response actions
4. Do the reflection prompt before moving to the next week.

## Recommended Tracks

- Part-time: follow the 12-week plan below
- Intensive: combine multiple weeks using the `Delivery Schedules` section in
  `MASTER_CLASS_CURRICULUM.md`

## 12-Week Self-Study Plan

### Week 1: Architecture And Attack Surface

- Read: Module 1 in `MASTER_CLASS_CURRICULUM.md`
- Lab: `labs/module-01/README.md`
- Supporting file: `labs/module-01/trust-map-template.md`
- Focus:
  - control plane trust boundaries
  - kubelet, etcd, ingress, and node risk
  - workload-to-cluster attack paths
- Deliverable: a completed trust map with three abuse paths
- Reflection prompt: If one pod is compromised, what conditions would let that
  become a cluster-wide incident?

### Week 2: Identity, RBAC, And Privilege Escalation

- Read: Module 2 in `MASTER_CLASS_CURRICULUM.md`
- Lab: `labs/module-02/README.md`
- Supporting files:
  - `labs/module-02/over-privileged-rbac.yaml`
  - `labs/module-02/hardened-rbac.yaml`
- Focus:
  - service accounts
  - least privilege
  - escalation through RBAC verbs and bindings
- Deliverable: a rewritten RBAC model and a memo explaining the escalation paths
- Reflection prompt: Which permissions are most dangerous because they look
  harmless at first glance?

### Week 3: Pod Hardening And Runtime Controls

- Read: Module 3 in `MASTER_CLASS_CURRICULUM.md`
- Lab: `labs/module-03/README.md`
- Supporting files:
  - `labs/module-03/insecure-deployment.yaml`
  - `labs/module-03/hardened-deployment.yaml`
- Focus:
  - `securityContext`
  - capabilities
  - restricted workloads
  - what pod hardening cannot solve alone
- Deliverable: a hardened deployment and a short list of residual risks
- Reflection prompt: Which runtime restrictions materially reduce attacker
  capability, and which mainly improve hygiene?

### Week 4: Network Isolation And Exposure Reduction

- Read: Module 4 in `MASTER_CLASS_CURRICULUM.md`
- Lab: `labs/module-04/README.md`
- Supporting files:
  - `labs/module-04/default-deny.yaml`
  - `labs/module-04/allow-app-to-db.yaml`
  - `labs/module-04/egress-restrict.yaml`
- Focus:
  - namespace isolation
  - application-to-database access control
  - egress limits and bypasses
- Deliverable: a network policy set with a short gap analysis
- Reflection prompt: What attack paths still exist after NetworkPolicy is in
  place?

### Week 5: Secrets And Sensitive Data Exposure

- Read: Module 5 in `MASTER_CLASS_CURRICULUM.md`
- Lab: `labs/module-05/README.md`
- Supporting files:
  - `labs/module-05/leaky-deployment.yaml`
  - `labs/module-05/safer-secret-delivery.yaml`
- Focus:
  - secret exposure patterns
  - etcd encryption
  - secret managers and safer delivery methods
- Deliverable: an architecture note on safer secret handling
- Reflection prompt: Where do secrets leak most often in real environments:
  storage, transit, runtime, or people/process?

### Week 6: Admission Control And Policy-As-Code

- Read: Module 6 in `MASTER_CLASS_CURRICULUM.md`
- Lab: `labs/module-06/README.md`
- Supporting file: `labs/module-06/kyverno-policies.yaml`
- Focus:
  - preventive control points
  - policy rollout strategy
  - handling exceptions without gutting enforcement
- Deliverable: a policy set plus one documented exception workflow
- Reflection prompt: Which controls belong in admission policy versus platform
  standards or CI/CD?

### Week 7: Supply Chain Security And Image Trust

- Read: Module 7 in `MASTER_CLASS_CURRICULUM.md`
- Lab: `labs/module-07/README.md`
- Supporting file: `labs/module-07/image-signing-policy.yaml`
- Focus:
  - provenance
  - signing
  - registry trust
  - SBOM and vulnerability management tradeoffs
- Deliverable: an image trust policy covering build, registry, and deploy stages
- Reflection prompt: Which supply chain controls actually stop deployment of
  bad artifacts, and which only create visibility?

### Week 8: Cluster Hardening And Baselines

- Read: Module 8 in `MASTER_CLASS_CURRICULUM.md`
- Lab: `labs/module-08/README.md`
- Supporting files:
  - `labs/module-08/hardening-checklist.md`
  - `labs/module-08/audit-policy.yaml`
- Focus:
  - node and control-plane baselines
  - CIS benchmark themes
  - audit configuration
  - managed vs self-managed responsibilities
- Deliverable: a hardening checklist with prioritized gaps
- Reflection prompt: Which hardening actions have the highest risk reduction for
  the least operational cost?

### Week 9: Detection Engineering And Threat Hunting

- Read: Module 9 in `MASTER_CLASS_CURRICULUM.md`
- Lab: `labs/module-09/README.md`
- Supporting file: `labs/module-09/detection-matrix.md`
- Focus:
  - audit logs
  - runtime telemetry
  - suspicious exec, token abuse, and lateral movement signals
- Deliverable: five detections with logic, data source, and response linkage
- Reflection prompt: Which Kubernetes detections are high-signal enough to page
  on?

### Week 10: Incident Response And Forensics

- Read: Module 10 in `MASTER_CLASS_CURRICULUM.md`
- Lab: `labs/module-10/README.md`
- Supporting files:
  - `labs/module-10/tabletop-scenario.md`
  - `labs/module-10/attacker-pod.yaml`
- Focus:
  - triage sequence
  - evidence preservation
  - containment and recovery
  - persistence paths
- Deliverable: a namespace-compromise runbook
- Reflection prompt: What should you avoid doing first in a Kubernetes incident
  if you want to preserve evidence?

### Week 11: Governance, Multi-Tenancy, And Platform Security

- Read: Module 11 in `MASTER_CLASS_CURRICULUM.md`
- Lab: `labs/module-11/README.md`
- Supporting file: `labs/module-11/secure-namespace.yaml`
- Focus:
  - tenancy models
  - platform guardrails
  - team boundaries
  - security operating model design
- Deliverable: a governance model for a multi-team Kubernetes platform
- Reflection prompt: When is a shared cluster acceptable, and when is it the
  wrong security boundary?

### Week 12: Cloud Provider Mapping, Leadership, And Capstone

- Read:
  - Module 12 in `MASTER_CLASS_CURRICULUM.md`
  - Capstone section in `MASTER_CLASS_CURRICULUM.md`
- Lab: `labs/module-12/README.md`
- Supporting file: `labs/module-12/reference-architecture.md`
- Focus:
  - EKS, AKS, and GKE control mapping
  - leadership communication
  - target-state architecture
  - capstone synthesis
- Deliverable:
  - reference architecture
  - threat model
  - hardening baseline
  - detection and response matrix
  - 90-day roadmap
- Reflection prompt: Can you explain your recommended security roadmap to both a
  platform engineer and an engineering director?

## Weekly Cadence

Use this cadence if you are studying part-time:

- Day 1: read the module and annotate unknown concepts
- Day 2: review the lab guide and supporting assets
- Day 3: perform the lab
- Day 4: write your deliverable
- Day 5: do the reflection prompt and capture follow-up questions

## Completion Standard

You are on track if, by the end of the program, you can:

- explain how a Kubernetes attack path works from initial foothold to broader
  impact
- choose preventive controls based on realistic threat paths
- write or review policy, RBAC, and workload restrictions with confidence
- define meaningful detections instead of generic monitoring
- respond to a cluster security event with a disciplined workflow
- present a Kubernetes security roadmap in operational terms
