# Lab 01: Kubernetes Architecture Trust Map

## Objective

Build a component-level trust map of your cluster. Annotate each communication
path with the protocol used, the credential type, and the attacker impact if
that path is compromised. This exercise trains you to see the cluster the way an
attacker does before you ever touch a security control.

## Prerequisites

- A running cluster: kind, k3s, minikube, or a managed cloud cluster
- kubectl configured with at least read access to kube-system
- A copy of `trust-map-template.md` from this directory to fill in

## Step 1 — Enumerate Control-Plane Components

Run the commands below and record what you find for each component.

```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl get pods -n kube-system -o wide
kubectl get endpoints -n default kubernetes
```

For each kube-system pod ask:
- What role does it play in the cluster?
- What port does it listen on?
- What authenticates to it and how?
- What can an attacker do if they fully control it?

## Step 2 — Fill In The Trust Map Template

Open `trust-map-template.md` and complete every row in the communication paths
table. Minimum paths to document:

- kubectl → API server
- API server → etcd
- API server → kubelet
- kubelet → container runtime
- Pod service account token → API server
- Cloud controller manager → cloud provider API
- Ingress controller → upstream pod

## Step 3 — Trace Abuse Paths

For each scenario below, trace the attacker's path through your trust map.
Record: starting point, first action, how far the attacker can reach, and what
log entry (if any) would appear.

1. Stolen kubeconfig with cluster-admin privileges
2. Kubernetes dashboard exposed without authentication
3. Pod running with an overly privileged service account token
4. Compromised node — attacker has kubelet credentials
5. Poisoned image deployed through the CI/CD pipeline

## Step 4 — Classify Each Component

Mark every component in your map with one or more labels:

- **High-value target** — compromise gives broad cluster access
- **Detection opportunity** — anomalous access would be visible in audit logs
- **Hardening candidate** — a configuration change meaningfully reduces blast radius

## Deliverable

A completed trust map with abuse paths annotated. Be ready to explain the
shortest path from a compromised workload to cluster-admin.

## Assessment Question

A pod runs in the `default` namespace with the default service account and no
explicit RBAC bindings. Describe what that pod can do by default, then describe
what a motivated attacker would attempt next and why.
