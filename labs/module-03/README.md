# Lab 03: Pod Hardening To Restricted Baseline

## Objective

Take a deliberately insecure deployment and harden it until it meets the
Kubernetes Pod Security Standards Restricted profile. Document what each
insecure setting enables and what the hardened replacement actually blocks.

## Prerequisites

- A cluster with Pod Security Admission enabled (Kubernetes 1.25+)
- kubectl access with ability to apply Deployments

## Files

| File | Purpose |
| --- | --- |
| `insecure-deployment.yaml` | Starting point — intentionally broken |
| `hardened-deployment.yaml` | Reference solution — attempt your own first |

## Background

The Pod Security Standards define three tiers:

- **Privileged**: no restrictions — allows everything
- **Baseline**: prevents known privilege escalation vectors
- **Restricted**: heavily locked down, follows current hardening best practices

Your goal is Restricted. A namespace enforcing Restricted will reject any pod
that does not meet all of its requirements.

## Step 1 — Apply The Insecure Deployment

```bash
# Create a test namespace with no enforcement so the broken pod runs
kubectl create namespace lab-insecure
kubectl apply -f insecure-deployment.yaml -n lab-insecure
```

## Step 2 — Audit The Security Context

For each field in the insecure deployment, answer:

| Field | Value | What Can An Attacker Do With This? |
| --- | --- | --- |
| `privileged: true` | | |
| `hostPID: true` | | |
| `hostNetwork: true` | | |
| `runAsUser: 0` | | |
| `allowPrivilegeEscalation: true` | | |
| `hostPath` volume to `/` | | |
| No `seccompProfile` | | |
| No `capabilities` drop | | |
| No resource limits | | |

## Step 3 — Attempt To Enforce Restricted

```bash
kubectl create namespace lab-restricted
kubectl label namespace lab-restricted \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest

# This should be rejected
kubectl apply -f insecure-deployment.yaml -n lab-restricted
```

Read the admission error carefully. It tells you exactly which fields violate
the Restricted profile.

## Step 4 — Harden The Deployment

Before reading `hardened-deployment.yaml`, write your own hardened version.
Your deployment must:

- Run as a non-root user (UID >= 1000)
- Set `allowPrivilegeEscalation: false`
- Drop all capabilities
- Use a `RuntimeDefault` seccomp profile
- Set `readOnlyRootFilesystem: true`
- Remove all host namespace flags (hostPID, hostNetwork, hostIPC)
- Remove all hostPath volumes
- Set CPU and memory requests and limits
- Disable service account token automounting if the app does not call the API

## Step 5 — Validate In The Restricted Namespace

```bash
kubectl apply -f your-hardened-deployment.yaml -n lab-restricted
kubectl get pods -n lab-restricted
kubectl describe pod -n lab-restricted <pod-name>
```

If the pod is Running, it passed. If it is not, read the events and fix
each violation.

## Step 6 — What Restricted Still Cannot Block

Hardening at the pod layer reduces risk but does not eliminate it. Document
at least three attack techniques that a Restricted pod does not prevent:

1.
2.
3.

## Deliverable

Your hardened deployment manifest plus a completed audit table that explains
what each original insecure setting enabled and what the replacement blocks.

## Assessment Question

A developer argues that their application must run as root because it binds to
port 80. Explain why this is not true and what the correct solution is, both for
the container configuration and for the Kubernetes service layer.
