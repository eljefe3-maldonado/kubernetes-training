# Lab 12: Cloud Provider Mappings And Reference Architecture

## Module Alignment

- Module doc: `docs/modules/module-12.md`
- Curriculum: Module 12 and capstone prep
- Estimated time: 75 to 120 minutes

## What You Will Produce

- a cloud-specific reference architecture
- control mappings for identity, networking, secrets, and logging
- inputs for the final capstone roadmap

## Objective

Produce a reference architecture for a secure managed Kubernetes platform in
one cloud provider (EKS, GKE, or AKS), covering identity, networking, secrets,
policy, observability, and incident response controls. Then prepare a
presentation-ready security review that explains current-state risk, target
state, top control gaps, and a phased remediation roadmap.

## Prerequisites

- Access to a cloud provider account and its managed Kubernetes service
- Cloud CLI configured (aws/az/gcloud)
- kubectl configured against the target cluster

## Files

| File | Purpose |
| --- | --- |
| `reference-architecture.md` | Blank template — complete this first |
| `eks-reference-architecture-example.md` | Fully worked EKS example — review this after completing your own |

**Important**: complete `reference-architecture.md` before opening the example.
The value of this lab is in making your own decisions first, then comparing
your reasoning against a worked model. Reading the example first turns a design
exercise into a copying exercise.

## Step 1 — Provider Comparison

Before designing your architecture, complete the comparison table below for
the three major providers. Focus on security-relevant differences, not feature
parity lists.

| Control Area | EKS | GKE | AKS |
| --- | --- | --- | --- |
| Default etcd encryption | | | |
| Customer-managed encryption keys | | | |
| Control-plane log types available | | | |
| Node OS update model | | | |
| Built-in network policy engine | | | |
| Workload identity (pod IAM) mechanism | | | |
| Secrets integration (native CSI) | | | |
| Admission control (built-in policy) | | | |
| Private endpoint support | | | |
| Kubernetes version support window | | | |

## Step 2 — Complete The Reference Architecture

Open `reference-architecture.md` and fill in each section for the provider
you have access to. Use the hardening checklist from Lab 08 as a guide for
what controls to include.

Minimum coverage for each section:

- **Identity**: how humans and workloads authenticate; OIDC or cloud IAM integration
- **Networking**: VPC layout, private cluster, NetworkPolicy CNI, ingress design
- **Secrets**: etcd encryption at rest, external secrets manager integration
- **Policy**: admission control, Pod Security Standards, Kyverno/Gatekeeper
- **Observability**: audit log destination, runtime detection tool, alerting
- **Incident response**: isolation procedure, credential rotation, forensics access

## Step 3 — Evaluate Managed Add-On Risk

For each managed add-on your cluster uses (metrics-server, cluster-autoscaler,
AWS Load Balancer Controller, etc.), assess:

| Add-On | RBAC Permissions Required | Network Access Required | Update Model | Risk Assessment |
| --- | --- | --- | --- | --- |
| | | | | |
| | | | | |
| | | | | |

Key questions:
- Does the add-on require cluster-admin? Is that justified?
- Does the add-on run with hostNetwork?
- How are add-on updates applied, and does that require a maintenance window?

## Step 4 — Platform Upgrade Strategy

Document your approach to Kubernetes version upgrades.

1. What is the provider's version support window for your current minor version?
2. What is your current version skew between control plane and nodes?
3. What is your upgrade cadence target (e.g., always within one minor version of latest)?
4. What must be tested before upgrading a production cluster?
5. How do you handle deprecated API versions in workload manifests?

## Step 5 — Executive Security Review

Prepare a two-page briefing for engineering leadership. Use this structure:

### Current State

- What managed Kubernetes platform are you running?
- What is the overall security maturity level? (Initial / Developing / Defined / Managed)
- What are the top three risks today?

### Target State

- What does good look like in 6 months?
- What controls need to be implemented or improved?

### Top Control Gaps

List the five highest-priority gaps, each with:
- Gap description
- Business risk if not addressed
- Remediation effort (low/medium/high)
- Owner

### Phased Remediation Roadmap

| Phase | Timeline | Focus | Key Deliverables |
| --- | --- | --- | --- |
| 1 — Quick wins | Weeks 1–4 | Highest-risk, lowest-effort controls | |
| 2 — Foundation | Weeks 5–12 | Core hardening baseline | |
| 3 — Maturity | Months 4–6 | Detection, policy-as-code, governance | |

## Step 6 — Compare Against The Model Answer

Open `eks-reference-architecture-example.md` and compare it to your completed
architecture. For each section, answer:

1. What choices did you make differently, and why?
2. What controls are in the example that you did not include — are they missing
   from your environment or did you consciously omit them?
3. What is in your architecture that is not in the example — what risk does
   yours address that the EKS example does not?
4. Review the **Open Risks** section of the example. Does your architecture have
   an equivalent honest accounting of what is not yet in place?

The goal is not to match the example. The goal is to be able to defend every
difference with a sentence that starts with "I chose X instead of Y because..."

## Deliverable

Completed reference architecture document, your comparison notes against the
EKS example, and your executive security review.

## Assessment Question

An engineering leader asks why the team cannot simply use `cluster-admin` for
the CI/CD pipeline — "it's easier and we trust our pipeline." Write a one-page
response that explains the risk in operational terms (not abstract security
principles) and proposes the least-disruptive path to a properly scoped
alternative.
