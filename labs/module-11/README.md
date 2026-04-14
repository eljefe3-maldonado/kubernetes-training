# Lab 11: Governance, Multi-Tenancy, And Platform Security Operating Model

## Objective

Design a security operating model for an internal platform that serves multiple
product teams with different risk profiles. Understand where namespace-based
tenancy is sufficient and where it breaks down. Produce a governance model
that scales without blocking delivery.

## Prerequisites

- kubectl access with ability to create Namespaces, ResourceQuotas, and LimitRanges
- Kyverno installed (from Lab 06)

## Files

| File | Purpose |
| --- | --- |
| `secure-namespace.yaml` | Template for a production-grade tenant namespace |

## Tenancy Models — Core Concepts

Before the exercise, align on the definitions you will use:

| Model | Isolation Mechanism | Blast Radius | Use When |
| --- | --- | --- | --- |
| Namespace per team | RBAC + NetworkPolicy | Pod-to-pod across namespaces | Low-sensitivity internal apps |
| Namespace per environment | RBAC + NetworkPolicy + PSS | Pod escape affects one env | Separated dev/staging/prod |
| Node pool per team | Node affinity + taints/tolerations | Pod escape limited to node pool | Different compliance requirements |
| Separate cluster per team | Full cluster isolation | No shared blast radius | High-sensitivity, regulated workloads |
| Separate account per team | Cloud account + cluster isolation | Maximum isolation | PCI, HIPAA, financial data |

## Step 1 — Define Your Tenant Model

You are the platform security engineer for a company with three product teams:

- **Team Alpha**: internal tool, no customer data, low sensitivity
- **Team Beta**: processes customer PII, subject to regional data residency requirements
- **Team Gamma**: financial transaction processing, PCI-DSS scope

**Questions to answer before designing the namespace template:**

1. Can Team Beta and Team Gamma share a cluster? What controls would be required?
2. What is the minimum isolation boundary that satisfies PCI-DSS?
3. Who should be able to create namespaces — the product team, or the platform?
4. What must the platform enforce consistently across all tenants?

## Step 2 — Apply The Namespace Template

```bash
# Create Team Alpha's namespace using the template
sed 's/TEAM_NAME/alpha/g; s/COST_CENTER/cc-1001/g; s/SENSITIVITY/low/g' \
  secure-namespace.yaml | kubectl apply -f -

# Verify the namespace was created with the correct labels and annotations
kubectl get namespace alpha -o yaml

# Verify the ResourceQuota
kubectl describe resourcequota -n alpha

# Verify the LimitRange
kubectl describe limitrange -n alpha
```

## Step 3 — Test Tenant Isolation

Verify that Team Alpha's workloads cannot reach Team Beta's workloads:

```bash
kubectl create namespace beta
kubectl run nginx --image=nginx:1.27 --port=80 -n beta
kubectl expose pod nginx --port=80 -n beta

# From Team Alpha's namespace, try to reach Team Beta's service
kubectl run test --image=busybox:1.36 -n alpha --rm -it \
  --command -- wget -qO- http://nginx.beta.svc.cluster.local --timeout=3
# Expected: connection timeout (blocked by NetworkPolicy)
```

## Step 4 — Design The Platform Security Model

Document your platform security model using these dimensions:

### 4.1 Namespace Provisioning

- Who can request a new namespace?
- What information is required (team name, cost center, data sensitivity, compliance scope)?
- What is the automated provisioning flow?
- What manual review steps exist for high-sensitivity requests?

### 4.2 Policy Tiers

Define three policy tiers applied based on the `sensitivity` label on the namespace:

| Tier | Label | Additional Controls Beyond Baseline |
| --- | --- | --- |
| Standard | sensitivity=low | |
| Regulated | sensitivity=medium | |
| High-Security | sensitivity=high | |

### 4.3 Team Responsibilities vs Platform Responsibilities

| Responsibility | Platform Team | Security Team | Product Team |
| --- | --- | --- | --- |
| Namespace provisioning | | | |
| RBAC within the namespace | | | |
| NetworkPolicy baseline | | | |
| Application RBAC design | | | |
| Image scanning | | | |
| Secret rotation | | | |
| Incident response | | | |
| Policy exceptions | | | |

### 4.4 Evidence Collection For Audits

For a PCI-DSS audit, document what evidence you can produce automatically:

- [ ] Network segmentation proof
- [ ] Access control logs
- [ ] Encryption at rest verification
- [ ] Patch currency for nodes and images
- [ ] Security scan results
- [ ] Audit log retention proof

## Step 5 — Recommend Cluster Architecture

Based on the three teams described in Step 1, make a formal recommendation:

- Which teams can share a cluster and under what conditions?
- Which teams require dedicated clusters or dedicated cloud accounts?
- What is the cost and operational complexity tradeoff?
- What compensating controls allow a shared cluster to satisfy compliance
  requirements that would otherwise require separation?

## Deliverable

Your completed platform security model document covering namespace provisioning,
policy tiers, team responsibilities, evidence collection, and your cluster
architecture recommendation with justification.

## Assessment Question

A product team argues that they should be allowed to deploy a privileged DaemonSet
because their observability tool requires host access. The platform's policy
blocks all privileged workloads. Walk through your exception decision process:
what you verify, what compensating controls you require, how you document the
exception, and when you would revisit it.
