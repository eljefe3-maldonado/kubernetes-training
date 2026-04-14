# Lab 10: Incident Response And Forensics In Kubernetes

## Objective

Run a tabletop exercise that starts with a detection alert and ends with cluster
recovery and post-incident control improvements. Practice isolating workloads
without destroying evidence, rotating credentials after a suspected compromise,
and documenting persistence mechanisms before removing them.

## Prerequisites

- kubectl access with ability to apply and delete resources
- The attacker scenario manifests in this directory
- The IR runbook in this directory

## Files

| File | Purpose |
| --- | --- |
| `tabletop-scenario.md` | Structured incident scenario with timed injects |
| `attacker-pod.yaml` | Simulated attacker persistence artifact for analysis |

## Incident Overview

The scenario involves a suspected compromise of a workload namespace that
may have escalated to cluster-admin privileges. The exercise runs in four phases:

1. **Triage** (30 min): scope the incident, preserve evidence
2. **Containment** (30 min): isolate without destroying
3. **Eradication** (20 min): remove attacker presence
4. **Recovery and improvement** (20 min): restore service, close gaps

## Step 1 — Set Up The Scenario

```bash
kubectl create namespace incident-lab
kubectl apply -f attacker-pod.yaml -n incident-lab
```

## Step 2 — Receive The Alert

Your detection engineering alert fires:

```
ALERT: kubectl exec session opened in production pod
User: system:serviceaccount:incident-lab:compromised-app
Source IP: 198.51.100.45 (not in known-good IP list)
Target pod: api-server-7d9f8b-xkj2p (namespace: incident-lab)
Time: <now>
```

Work through `tabletop-scenario.md` from this point.

## Step 3 — Triage Commands Reference

Use these commands during the tabletop. Do not memorize them — practice
looking them up under simulated time pressure.

```bash
# What is running in the affected namespace?
kubectl get all -n incident-lab

# Who has access to the namespace?
kubectl get rolebindings,clusterrolebindings -A | \
  grep incident-lab

# What service account tokens exist?
kubectl get serviceaccounts -n incident-lab
kubectl get secrets -n incident-lab

# What has the pod been doing? (requires audit log access)
# Search for: user=system:serviceaccount:incident-lab:compromised-app

# Is there a privileged pod or unusual DaemonSet?
kubectl get pods -A -o json | jq '.items[] |
  select(.spec.containers[].securityContext.privileged == true) |
  .metadata | {name, namespace}'

# Check for unexpected ClusterRoleBindings
kubectl get clusterrolebindings -o json | jq '.items[] |
  select(.subjects[]?.namespace == "incident-lab") |
  {name: .metadata.name, role: .roleRef.name}'

# Inspect the suspicious pod without exec (read-only)
kubectl describe pod <pod-name> -n incident-lab
kubectl get pod <pod-name> -n incident-lab -o yaml

# Use an ephemeral debug container to inspect the pod without exec
kubectl debug -it <pod-name> -n incident-lab \
  --image=busybox:1.36 --target=<container-name>
```

## Step 4 — Isolation Without Evidence Destruction

To isolate a pod without deleting it:

```bash
# Apply a NetworkPolicy that blocks all traffic to/from the pod
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-compromised-pod
  namespace: incident-lab
spec:
  podSelector:
    matchLabels:
      app: compromised-app
  policyTypes:
  - Ingress
  - Egress
  # No rules = deny all ingress and egress
EOF
```

Do NOT delete the pod before you have:
- Captured the pod spec (`kubectl get pod -o yaml`)
- Reviewed audit logs for all API calls made by its service account
- Noted all mounted volumes, secrets, and environment variables
- Captured any relevant container logs

## Step 5 — Credential Rotation Checklist

After a service account compromise, rotate in this order:

1. Delete and recreate the service account (invalidates all existing tokens)
2. Update any Deployments or CronJobs referencing the old service account
3. Rotate any secrets the service account had `get` access to
4. Audit which other principals can read those same secrets
5. Review whether the compromised token was used from any external system

```bash
# Delete the service account (all issued tokens are immediately revoked)
kubectl delete serviceaccount compromised-app -n incident-lab

# Recreate with the same name for workload continuity
kubectl create serviceaccount compromised-app -n incident-lab
```

## Step 6 — Post-Incident Control Review

After the scenario, document:

1. Which detection fired and how quickly
2. Which controls slowed the attacker (even if they did not stop the attack)
3. Which controls were missing or bypassed
4. The three highest-priority improvements to prevent recurrence

## Deliverable

A completed incident response runbook entry covering this scenario:
scope, timeline, evidence collected, containment actions, eradication steps,
credential rotation performed, and three post-incident improvements.

## Assessment Question

An attacker has compromised a pod and used the mounted service account token
to create a new ClusterRoleBinding granting cluster-admin to a new service
account they created. They then deleted the original ClusterRoleBinding from
their audit trail. Describe your investigation and recovery steps, including
how you would identify what the attacker did even after they attempted to
cover their tracks.
