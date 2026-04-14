# Lab 05: Secrets, Exposure Vectors, And Safer Delivery

## Module Alignment

- Module doc: `docs/modules/module-05.md`
- Curriculum: Module 5
- Estimated time: 60 to 75 minutes

## What You Will Produce

- a safer secret-delivery design
- a summary of exposure paths in the original deployment
- a recommendation note for production use

## Objective

Identify every path through which a workload leaks secret material, then
refactor it to use safer delivery and narrower access controls. Understand
why Kubernetes native secrets are not encrypted by default and what the
operational tradeoffs are between native secrets, sealed secrets, and
external secret managers.

## Prerequisites

- kubectl access with ability to create Deployments, Secrets, and RoleBindings
- Optional: External Secrets Operator installed, or a Vault/AWS SSM environment

## Files

| File | Purpose |
| --- | --- |
| `leaky-deployment.yaml` | Workload that leaks secrets through multiple vectors |
| `safer-secret-delivery.yaml` | Refactored version with tighter controls |

## Step 1 — Apply The Leaky Workload

```bash
kubectl create namespace secrets-lab
kubectl apply -f leaky-deployment.yaml -n secrets-lab
```

## Step 2 — Find Every Leak Vector

For the leaky deployment, identify and document every path through which the
secret value could be exposed. Use the table below as a starting framework.

| Leak Vector | Present? | How An Attacker Exploits It |
| --- | --- | --- |
| Secret value in environment variable | | |
| `env` vars visible in `kubectl describe pod` | | |
| Application logs the env var at startup | | |
| RBAC allows listing all secrets in namespace | | |
| RBAC allows `watch` which streams future updates | | |
| Secret mounted at a predictable path without mode restriction | | |
| `automountServiceAccountToken: true` with no RBAC guard | | |
| Secret value visible in Deployment spec via `kubectl get deploy -o yaml` | | |
| etcd stores the secret as base64 (not encrypted at rest) | | |

Check each one:

```bash
# Can you read the secret via RBAC?
kubectl auth can-i list secrets -n secrets-lab \
  --as=system:serviceaccount:secrets-lab:leaky-app

# Is the secret value visible in pod env output?
kubectl exec -n secrets-lab deploy/leaky-app -- env | grep -i db

# Is the Deployment spec itself the source of truth for the secret value?
kubectl get deploy leaky-app -n secrets-lab -o yaml | grep -A5 env
```

## Step 3 — Refactor For Safer Delivery

Before reading `safer-secret-delivery.yaml`, write your own refactored version:

- Remove environment variable delivery; use a volume-mounted secret file instead
- Set `defaultMode: 0400` on the volume so only the container user can read it
- Restrict RBAC to `get` on one named secret — no `list`, no `watch`
- Set `automountServiceAccountToken: false` unless the app needs API access
- Add a liveness probe or startup check that does not log the secret value

## Step 4 — Understand Etcd Exposure

Run the following to check whether your cluster has encryption at rest enabled:

```bash
# On a self-managed cluster, check the API server flag
kubectl get pod kube-apiserver-<node> -n kube-system -o yaml | \
  grep encryption-provider-config

# On a managed cluster, check provider documentation
# EKS: envelope encryption via KMS is opt-in per cluster
# GKE: default encryption at rest, CMEK is opt-in
# AKS: server-side encryption at rest by default, CMK is opt-in
```

If encryption is not enabled, any backup of etcd contains all secret values
in base64-encoded plaintext.

## Step 5 — Compare Delivery Patterns

Write a one-paragraph comparison of each pattern for a regulated workload:

| Pattern | Pros | Cons | Best For |
| --- | --- | --- | --- |
| Native Kubernetes Secret | | | |
| Sealed Secrets (Bitnami) | | | |
| External Secrets Operator + AWS SSM | | | |
| CSI Secrets Store Driver + Vault | | | |

## Deliverable

The refactored deployment manifest plus a completed leak-vector table that
explains each vector and how your refactor addressed it.

## Assessment Question

Your application reads a database password from an environment variable. A
security audit flags this as a finding. The developer says "the secret is
already in Kubernetes so reading it as an env var is no different from reading
it as a file." Explain the specific ways in which env var delivery is riskier
than volume-mounted file delivery.
