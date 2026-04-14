# Master Class Curriculum: Kubernetes For Cloud Security Engineers

## Course Positioning

This class is designed for security professionals who need more than
administrator-level Kubernetes familiarity. The target outcome is operating
judgment: the ability to assess platform risk, design compensating controls,
guide engineering teams, and respond effectively when a cluster, workload, or
software supply chain is compromised.

## Competency Model

The program builds mastery across six dimensions:

- Kubernetes architecture and control-plane trust boundaries
- Workload and runtime security
- Identity, access, and secret management
- Policy, governance, and compliance engineering
- Detection engineering and incident response
- Secure platform design and organizational operating models

## Prerequisites

The learner should already understand:

- Linux fundamentals and process/network basics
- cloud IAM concepts in at least one major provider
- containers, images, and registries
- infrastructure as code and CI/CD fundamentals
- basic security concepts such as least privilege, segmentation, logging, and
  threat modeling

## Learning Outcomes

After completing the course, the learner will be able to:

- map Kubernetes components to security responsibilities and attack surfaces
- distinguish cluster-admin concerns from security engineering concerns
- secure workloads through pod design, image controls, policy gates, and
  runtime restrictions
- implement layered control strategies using RBAC, network policy, admission
  control, secrets controls, and supply chain verification
- build practical detection content using audit logs, runtime telemetry, and
  cloud-native observability sources
- run a Kubernetes-centric incident response workflow from triage to recovery
- evaluate organizational maturity and recommend a platform security roadmap

## Recommended Delivery Model

Each module should include:

- 60 to 90 minutes of lecture or guided walkthrough
- 60 to 120 minutes of hands-on lab
- 30 minutes of review, threat discussion, or assessment

## Module 1: Kubernetes Architecture Through A Security Lens

### Objectives

- Understand the control plane, kubelet, etcd, CNI, CSI, and ingress paths as
  trust boundaries.
- Explain where authentication, authorization, admission, scheduling, and
  runtime isolation each occur.
- Identify common attacker goals in cluster environments.

### Must Know

- API server as the central policy and control choke point
- etcd sensitivity and why it is a high-value target
- kubelet and node compromise implications
- namespace isolation limits
- differences between container isolation and VM isolation

### Lab

Build a cluster component trust map and annotate likely abuse paths:

- stolen kubeconfig
- exposed dashboard or API endpoint
- overly privileged service account
- compromised node
- poisoned image in the deployment pipeline

### Assessment

Describe the shortest path from a compromised workload to broader cluster impact
under three different misconfiguration scenarios.

## Module 2: Identity, Authentication, And Authorization

### Objectives

- Master how users, service accounts, controllers, and external identities
  authenticate to Kubernetes.
- Design least-privilege authorization patterns.
- Analyze RBAC escalation paths.

### Must Know

- RBAC verbs, resources, aggregation, and wildcard risk
- service account token handling and projected tokens
- impersonation and delegated access
- common privilege-escalation primitives such as `create pods`, `bind`, and
  secret access
- cloud identity federation patterns for EKS, GKE, and AKS

### Lab

- Review a deliberately over-privileged RBAC model.
- Reduce privileges for platform operators, application teams, CI/CD, and
  security tooling.
- Document how the original model enabled escalation.

### Assessment

Produce an RBAC review memo that identifies high-risk permissions and proposes a
replacement access model.

## Module 3: Workload Security And Pod Hardening

### Objectives

- Harden workloads to reduce breakout and privilege escalation risk.
- Distinguish safe defaults from risky workload patterns.
- Translate Pod Security requirements into engineering guardrails.

### Must Know

- `securityContext` fields and their security impact
- root vs non-root execution
- capabilities and privilege escalation controls
- seccomp, AppArmor, SELinux, and read-only filesystem controls
- host namespaces, hostPath, privileged mode, and dangerous DaemonSet patterns
- Pod Security Standards: Privileged, Baseline, Restricted

### Lab

Take an insecure deployment and harden it until it meets a restricted baseline.
Document what still cannot be enforced at the pod layer alone.

### Assessment

Explain which workload settings actually block attacker actions versus which only
improve hygiene.

## Module 4: Network Security In Real Clusters

### Objectives

- Apply layered network defenses in Kubernetes.
- Understand where Kubernetes NetworkPolicy is sufficient and where it is not.
- Design service exposure patterns that minimize blast radius.

### Must Know

- east-west vs north-south traffic
- default allow vs default deny network posture
- ingress controllers, gateways, and service mesh tradeoffs
- DNS abuse, metadata service access, and egress control
- node-to-pod, pod-to-pod, and control-plane communication paths

### Lab

- Implement namespace isolation with NetworkPolicy.
- Restrict database access to specific workloads.
- Add egress restrictions and document any bypass paths that remain.

### Assessment

Threat-model an internet-facing application on Kubernetes and propose a layered
network defense architecture.

## Module 5: Secrets, Key Management, And Sensitive Data Exposure

### Objectives

- Handle secret material safely in Kubernetes platforms.
- Understand the limits of native secrets.
- Integrate external secret stores and encryption controls.

### Must Know

- why base64 is not security
- etcd encryption at rest and key management dependencies
- secret access via RBAC, mounted volumes, env vars, and application logs
- external secret managers and CSI-based secret delivery
- certificate lifecycle management and workload identity patterns

### Lab

Review a workload that leaks secrets through configuration, logging, and RBAC.
Refactor it to use safer delivery and narrower access controls.

### Assessment

Write a short architecture note comparing native secrets, sealed secrets, and
external secret managers for regulated workloads.

## Module 6: Admission Control, Policy-As-Code, And Guardrails

### Objectives

- Build preventive controls that stop bad workloads before deployment.
- Compare built-in and external admission strategies.
- Turn security requirements into enforceable policy.

### Must Know

- validating vs mutating admission
- built-in admission plugins
- ValidatingAdmissionPolicy concepts
- OPA Gatekeeper and Kyverno use cases
- exceptions management and policy rollout strategy
- balancing developer experience with enforcement depth

### Lab

Create a policy set that blocks:

- privileged containers
- images without pinned tags or approved registries
- workloads missing resource limits
- risky host namespace usage

Then define one justified exception workflow.

### Assessment

Design a policy adoption plan for an engineering organization with legacy
workloads and weak standards.

## Module 7: Software Supply Chain And Image Assurance

### Objectives

- Secure the path from source code to running container.
- Identify supply chain compromise opportunities.
- Apply attestation, scanning, and provenance controls in practical ways.

### Must Know

- image provenance and trusted build systems
- SBOM generation and practical uses
- vulnerability scanning limitations
- signature verification and image admission
- registry segmentation and promotion workflows
- dependency freshness vs reproducibility tradeoffs

### Lab

Design an image trust policy that combines build provenance, image signing,
registry promotion, and deployment admission checks.

### Assessment

Review a compromised-image scenario and identify which controls would have
prevented it, detected it, or only reduced impact.

## Module 8: Cluster Hardening And Platform Baselines

### Objectives

- Secure cluster infrastructure, nodes, and control-plane dependencies.
- Build a hardened baseline for managed or self-managed clusters.
- Understand where provider-managed services reduce effort and where they hide
  risk.

### Must Know

- CIS Kubernetes Benchmark themes
- node OS hardening and immutable node patterns
- kubelet configuration risk
- API server exposure and private endpoint strategies
- audit logging configuration
- managed Kubernetes security differences across major cloud platforms

### Lab

Create a cluster hardening checklist for one managed platform and one
self-managed pattern. Call out controls owned by the cloud provider versus the
customer.

### Assessment

Present a gap analysis for a cluster baseline and prioritize remediation by risk
and implementation difficulty.

## Module 9: Observability, Detection Engineering, And Threat Hunting

### Objectives

- Build practical Kubernetes detections.
- Know which telemetry sources matter and why.
- Separate noisy events from strong attacker signals.

### Must Know

- Kubernetes audit logs
- container runtime telemetry
- cloud control-plane logs
- DNS, egress, and service-account anomaly signals
- detection content for crypto mining, token theft, privilege escalation, and
  suspicious exec activity

### Lab

Create a detection matrix with:

- data source
- analytic idea
- expected false positives
- response playbook linkage

### Assessment

Draft five high-confidence detections for a production Kubernetes environment
and justify why each is worth operationalizing.

## Module 10: Incident Response And Forensics In Kubernetes

### Objectives

- Respond to Kubernetes incidents without destroying useful evidence.
- Coordinate containment across workload, node, cluster, and cloud layers.
- Recover safely after attacker persistence attempts.

### Must Know

- triage sequencing for suspected cluster compromise
- ephemeral container use for response
- isolating workloads without erasing state
- node quarantine patterns
- credential rotation after service account or secret theft
- persistence mechanisms through controllers, cron jobs, images, and GitOps

### Lab

Run a tabletop exercise covering:

- alert on suspicious `kubectl exec`
- discovery of a privileged pod
- evidence of secret exfiltration
- cluster recovery and post-incident control improvements

### Assessment

Write an incident response runbook for a compromised namespace that may have
reached cluster-admin privileges.

## Module 11: Governance, Compliance, And Multi-Tenant Platform Security

### Objectives

- Build a governance model that scales across teams.
- Translate security requirements into platform standards.
- Secure shared clusters without relying on soft boundaries.

### Must Know

- multi-tenancy models and their security tradeoffs
- namespace-based tenancy limitations
- policy tiers, golden templates, and paved-road approaches
- evidence collection for audits and control attestations
- separation of duties between platform, security, and application teams

### Lab

Design a security operating model for an internal platform that serves multiple
product teams with different risk profiles.

### Assessment

Recommend when an organization should use shared clusters, dedicated clusters,
or separate accounts/projects/subscriptions for stronger isolation.

## Module 12: Cloud Provider Mappings And Real-World Security Leadership

### Objectives

- Translate Kubernetes security knowledge into cloud-provider decisions.
- Prepare to lead reviews, standards, and roadmaps.
- Make pragmatic decisions under business constraints.

### Must Know

- EKS, AKS, and GKE shared patterns and meaningful differences
- cloud-native integrations for identity, secrets, logging, and policy
- managed add-on risk evaluation
- platform upgrade strategy and version skew concerns
- how to brief engineering leadership on Kubernetes risk and maturity

### Lab

Produce a reference architecture for a secure managed Kubernetes platform in one
cloud provider, including identity, networking, secrets, policy, observability,
and incident response controls.

### Assessment

Deliver an executive security review that explains current-state risk, target
state, top control gaps, and a phased remediation roadmap.

## Capstone

The learner should complete a capstone that simulates acting as the lead Cloud
Security Engineer for a Kubernetes platform.

### Capstone Requirements

- assess an example cluster architecture
- identify top risks and likely abuse paths
- define preventive, detective, and responsive controls
- create a prioritized remediation roadmap
- present tradeoffs to both engineering and leadership audiences

### Capstone Deliverables

- threat model
- cluster hardening baseline
- policy-as-code strategy
- detection and response matrix
- 90-day improvement roadmap

## Delivery Schedules

Two pacing models are provided. Both cover the same 12 modules and capstone.
Choose based on available time and cohort intensity.

---

### Part-Time Track — 10 Weeks

Designed for engineers who are carrying full workloads alongside the course.
Each week requires roughly 4 to 6 hours of study, lab, and review time.

| Week | Module(s) | Focus | Time Budget |
| --- | --- | --- | --- |
| 1 | Module 1 | Kubernetes architecture, control-plane trust boundaries, attack surface mapping | 5 h |
| 2 | Module 2 | Identity, RBAC, service accounts, privilege escalation paths | 5 h |
| 3 | Module 3 | Pod hardening, security contexts, Pod Security Standards | 5 h |
| 4 | Module 4 | Network security, east-west isolation, ingress and egress control | 5 h |
| 5 | Module 5 | Secrets, etcd encryption, external secret managers | 4 h |
| 6 | Module 6 | Admission control, ValidatingAdmissionPolicy, OPA Gatekeeper, Kyverno | 5 h |
| 7 | Module 7 | Software supply chain, image signing, SBOM, provenance | 5 h |
| 8 | Module 8 | Cluster hardening, CIS Benchmark, node and control-plane baselines | 5 h |
| 9 | Module 9 | Observability, audit logs, runtime telemetry, detection content | 5 h |
| 10 | Module 10 | Incident response, forensics, node quarantine, credential rotation | 5 h |
| 11 | Modules 11–12 | Governance, multi-tenancy, cloud-provider mappings, leadership | 6 h |
| 12 | Capstone | Full cluster security review, hardening baseline, detection matrix, roadmap | 6 h |

**Week structure (suggested):**

- Day 1: lecture or guided walkthrough (60–90 min)
- Day 2–3: hands-on lab (60–120 min)
- Day 4: review, threat discussion, or written assessment (30–45 min)
- Day 5: optional reading, practice cluster time, or carry-over

---

### Intensive Track — 2.5 Weeks (12 Business Days)

Designed for full-time immersive delivery such as a boot camp, team sprint, or
dedicated training block. Each day runs roughly 6 to 7 hours.

| Day | Module | Morning (3 h) | Afternoon (3–4 h) |
| --- | --- | --- | --- |
| 1 | Module 1 | Architecture walkthrough, trust boundary annotation | Lab: cluster component map and abuse path exercise |
| 2 | Module 2 | RBAC deep dive, service account token lifecycle | Lab: RBAC review and least-privilege redesign |
| 3 | Module 3 | Pod security contexts, capability controls, PSS tiers | Lab: harden an insecure deployment to restricted baseline |
| 4 | Module 4 | Network policy, ingress, service mesh, DNS and egress | Lab: namespace isolation, database access controls, egress restriction |
| 5 | Module 5 | Secrets architecture, etcd risk, CSI secret delivery | Lab: secret leak review and refactor to safer delivery |
| 6 | Module 6 | Admission control design, Gatekeeper vs Kyverno | Lab: build a policy set, define exception workflow |
| 7 | Module 7 | Supply chain threat model, image trust, signing | Lab: design image trust policy across build, registry, and deploy stages |
| 8 | Module 8 | CIS Benchmark review, node hardening, audit log setup | Lab: cluster hardening checklist and gap analysis |
| 9 | Module 9 | Detection strategy, telemetry sources, analytic development | Lab: detection matrix with data source, analytic, FP rate, and playbook |
| 10 | Module 10 | IR workflow, ephemeral containers, persistence patterns | Lab: tabletop — suspicious exec to cluster-admin escalation |
| 11 | Module 11 | Multi-tenancy models, policy tiers, governance design | Lab: security operating model for a multi-team platform |
| 12 | Module 12 | Cloud provider tradeoffs, managed service risk, leadership | Lab: reference architecture for one cloud provider |
| 13–15 | Capstone | Three days: assess → design → present | Deliver threat model, baseline, detection matrix, and 90-day roadmap |

**Day structure (intensive):**

- 09:00–12:00: lecture, guided walkthrough, threat discussion
- 12:00–13:00: break
- 13:00–16:00: hands-on lab
- 16:00–17:00: debrief, assessment, Q&A

---

### Milestone Checkpoints

Use these to evaluate learner readiness before advancing.

| Checkpoint | After | Gate Criteria |
| --- | --- | --- |
| Architecture fluency | Module 1 | Can draw the control plane, annotate trust boundaries, and name three high-value attacker targets |
| Access model | Module 2 | Can audit an RBAC model and identify privilege escalation paths |
| Workload baseline | Module 3 | Can harden a deployment to the Restricted Pod Security Standard |
| Network posture | Module 4 | Can write NetworkPolicy for namespace isolation and document remaining gaps |
| Secrets hygiene | Module 5 | Can identify secret exposure vectors and propose safer delivery |
| Policy gate | Module 6 | Can write and deploy a policy that blocks at least two dangerous workload patterns |
| Supply chain | Module 7 | Can describe the trust chain from source to running container and name two break points |
| Platform baseline | Module 8 | Can produce a cluster hardening checklist and prioritize by risk |
| Detection capability | Module 9 | Can author five detections with data source, logic, and response linkage |
| IR readiness | Module 10 | Can run a tabletop from initial alert to cluster recovery |
| Mid-program review | Module 11 | Cohort debrief on modules 1–11; revisit any milestone not yet met |
| Capstone complete | Module 12 + capstone | Delivers all five capstone artifacts at the final standard defined at the end of this curriculum |

---

## Study Guidance

To reach mastery, the learner should repeatedly connect every topic to four
questions:

1. What can be attacked here?
2. What preventive control is strongest?
3. What detection would reveal abuse early?
4. What recovery action contains damage fastest?

## Suggested External Practice Areas

- deploy and harden a small multi-namespace cluster
- write RBAC from scratch for multiple personas
- enforce policies with Kyverno or Gatekeeper
- review Kubernetes audit logs and build detections
- simulate incident response for token theft or privileged pod deployment
- compare the same control objective across EKS, AKS, and GKE

## Final Standard

Someone who completes this program at the intended level should be able to:

- review a Kubernetes platform and identify meaningful security gaps quickly
- explain why each gap matters in operational terms
- implement or guide controls without blocking delivery unnecessarily
- respond to incidents with discipline across infrastructure, identity, and
  workload layers
