# Lab 12: Cloud Provider Mappings And Reference Architecture

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
| `reference-architecture.md` | Reference architecture template to complete |

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

## Deliverable

Completed reference architecture document plus your executive security review.

## Assessment Question

An engineering leader asks why the team cannot simply use `cluster-admin` for
the CI/CD pipeline — "it's easier and we trust our pipeline." Write a one-page
response that explains the risk in operational terms (not abstract security
principles) and proposes the least-disruptive path to a properly scoped
alternative.
