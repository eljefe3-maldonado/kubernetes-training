# Lab 02: RBAC Review And Least-Privilege Redesign

## Objective

Audit a deliberately over-privileged RBAC model, identify every privilege
escalation path it enables, and produce a replacement model that enforces
least privilege for each persona.

## Prerequisites

- kubectl access to a non-production cluster
- Ability to apply and delete RBAC resources

## Files

| File | Purpose |
| --- | --- |
| `over-privileged-rbac.yaml` | The broken starting state — review this first |
| `hardened-rbac.yaml` | Reference solution — attempt your own before reading |

## Step 1 — Apply The Broken Model

```bash
kubectl apply -f over-privileged-rbac.yaml
```

## Step 2 — Audit Each Service Account

For each service account in the file, answer:

1. What namespaces can it operate in?
2. What verbs can it use on which resources?
3. Can it read secrets? Which ones?
4. Can it create or modify pods?
5. Can it escalate to a higher-privileged role using `bind`, `escalate`,
   or `impersonate`?
6. Can it use `create pods` or `create pods/exec` to run arbitrary code?

Use these commands to verify:

```bash
kubectl auth can-i --list --as=system:serviceaccount:default:ci-deployer
kubectl auth can-i --list --as=system:serviceaccount:default:app-api
kubectl auth can-i --list --as=system:serviceaccount:monitoring:logging-agent
```

## Step 3 — Map Escalation Paths

For each service account, document the shortest privilege escalation path.
Consider:

- Can the service account read secrets and find a more privileged token?
- Can it create a pod that mounts the node filesystem or runs as root?
- Can it modify its own role or binding?
- Can it impersonate another user or service account?

## Step 4 — Redesign For Least Privilege

Before looking at `hardened-rbac.yaml`, write your own replacement model:

- `ci-deployer`: needs to deploy applications to the `production` namespace only
- `app-api`: needs to read one named secret in its own namespace
- `logging-agent`: needs to read pod metadata and logs cluster-wide, nothing else

Constraints:
- Use namespace-scoped Roles and RoleBindings where possible
- No wildcards on verbs or resources
- No ClusterRoleBinding unless a cluster-scoped resource genuinely requires it

## Step 5 — Validate Your Model

```bash
kubectl apply -f your-hardened-rbac.yaml

# ci-deployer should be able to deploy in production
kubectl auth can-i create deployments -n production \
  --as=system:serviceaccount:production:ci-deployer

# ci-deployer should NOT be able to read secrets
kubectl auth can-i get secrets -n production \
  --as=system:serviceaccount:production:ci-deployer

# app-api should only read its own secret
kubectl auth can-i get secret db-credentials -n app \
  --as=system:serviceaccount:app:app-api
kubectl auth can-i list secrets -n app \
  --as=system:serviceaccount:app:app-api   # should be no
```

## Step 6 — Compare And Document

Review `hardened-rbac.yaml`. For any difference between your model and the
reference, write one sentence explaining which is better and why.

## Deliverable

An RBAC review memo containing:
- A table of each service account, its original permissions, and the
  escalation path it enables
- Your replacement model with justification for each binding
- Any permissions you could not remove because the application requires them,
  with a note on the residual risk

## Assessment Question

Explain the difference between a ClusterRole with namespace-scoped bindings
(RoleBinding referencing a ClusterRole) versus a ClusterRoleBinding. When would
you choose one over the other from a security perspective?
