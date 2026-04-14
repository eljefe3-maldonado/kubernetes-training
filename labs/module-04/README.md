# Lab 04: Network Isolation With NetworkPolicy

## Module Alignment

- Module doc: `docs/modules/module-04.md`
- Curriculum: Module 4
- Estimated time: 75 to 90 minutes

## What You Will Produce

- a default-deny posture
- scoped allow rules for application traffic
- a short analysis of what NetworkPolicy still does not cover

## Objective

Implement layered network defenses using Kubernetes NetworkPolicy. Start from
an open-by-default cluster posture, enforce namespace isolation, restrict
database access to a single workload, then add egress controls and document
the bypass paths that remain.

## Prerequisites

- A cluster with a CNI plugin that enforces NetworkPolicy
  (Calico, Cilium, or Weave — NOT Flannel alone)
- kubectl access with ability to create NetworkPolicy and Deployment resources

## Files

| File | Purpose |
| --- | --- |
| `default-deny.yaml` | Baseline default-deny posture for a namespace |
| `allow-app-to-db.yaml` | Narrow allow rule: app tier to database only |
| `egress-restrict.yaml` | Egress control: DNS + specific external CIDR only |

## Architecture Under Test

```
internet → ingress-controller (namespace: ingress)
                ↓
           frontend (namespace: web)
                ↓
           api-server (namespace: app)
                ↓
           postgres (namespace: data)
```

The lab works at the `app` and `data` namespace boundary.

## Step 1 — Observe The Open Posture

```bash
kubectl create namespace app
kubectl create namespace data

# Deploy a simple test workload in each namespace
kubectl run api-server --image=busybox:1.36 -n app \
  --command -- sleep 3600
kubectl run postgres --image=busybox:1.36 -n data \
  --command -- sleep 3600

# Verify unrestricted connectivity
kubectl exec -n app api-server -- wget -qO- http://postgres.data:5432 || true
kubectl exec -n data postgres -- wget -qO- http://api-server.app:8080 || true
```

Record what succeeds before any policy is applied.

## Step 2 — Apply Default Deny

```bash
kubectl apply -f default-deny.yaml
```

Verify that all traffic is now blocked:

```bash
kubectl exec -n app api-server -- wget -qO- http://postgres.data:5432 --timeout=3 || true
```

Test that the pods themselves are still running (policy blocks traffic, not pods).

## Step 3 — Allow App To Database

```bash
kubectl apply -f allow-app-to-db.yaml
```

Verify:

```bash
# Should succeed
kubectl exec -n app api-server -- nc -zv postgres.data 5432

# Should still fail — data namespace cannot initiate to app
kubectl exec -n data postgres -- nc -zv api-server.app 8080
```

## Step 4 — Restrict Egress

```bash
kubectl apply -f egress-restrict.yaml
```

Test that DNS still works (it must, or all service discovery breaks):

```bash
kubectl exec -n app api-server -- nslookup postgres.data
```

Test that arbitrary egress is blocked:

```bash
# Should be blocked
kubectl exec -n app api-server -- wget -qO- http://169.254.169.254 --timeout=3 || true
kubectl exec -n app api-server -- wget -qO- http://8.8.8.8 --timeout=3 || true
```

## Step 5 — Document Remaining Bypass Paths

NetworkPolicy has real limits. After applying all three policies, document
at least three things a compromised `api-server` pod can still do:

1.
2.
3.

Consider: node-level access, CNI bypasses, service mesh gaps, DNS tunneling,
and cloud metadata services that are not on a standard CIDR.

## Step 6 — Threat Model The Architecture

Draw the network flows for the four-tier architecture above. For each allowed
flow, mark the blast radius if the source workload is compromised. For each
blocked flow, describe the attack it prevents.

## Deliverable

A completed network threat model with your allow/deny policy set applied.
Include the bypass paths you identified and a recommendation for each.

## Assessment Question

A teammate proposes replacing all NetworkPolicy with a service mesh for
mutual TLS between pods. Describe two scenarios where NetworkPolicy is still
necessary even with a service mesh in place.
