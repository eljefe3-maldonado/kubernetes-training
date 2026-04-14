# Lab 06: Admission Control And Policy-As-Code

## Objective

Build a policy set that blocks dangerous workload patterns before they reach the
scheduler. Apply the policies to a cluster, verify they block what they claim to
block, then design a justified exception workflow.

## Prerequisites

- Kyverno installed (`kubectl get po -n kyverno`)
  OR OPA Gatekeeper installed (`kubectl get po -n gatekeeper-system`)
- Policies in this lab are written for Kyverno — adapt to Gatekeeper if needed

Install Kyverno (latest):
```bash
kubectl create -f https://github.com/kyverno/kyverno/releases/latest/download/install.yaml
```

## Files

| File | Purpose |
| --- | --- |
| `kyverno-policies.yaml` | Four enforcement policies plus an exception pattern |

## Step 1 — Apply The Policies In Audit Mode

Apply with `validationFailureAction: Audit` first. Audit mode logs violations
without blocking, so you can see what would break before you enforce.

```bash
kubectl apply -f kyverno-policies.yaml
```

Audit mode produces PolicyReport objects:

```bash
kubectl get policyreport -A
kubectl describe policyreport -n <namespace> <name>
```

Review the results before proceeding to enforce mode.

## Step 2 — Switch To Enforce Mode

Edit each policy and change `validationFailureAction: Audit` to
`validationFailureAction: Enforce`, then reapply.

Alternatively, patch in place:

```bash
kubectl patch clusterpolicy disallow-privileged-containers \
  --type merge \
  -p '{"spec":{"validationFailureAction":"Enforce"}}'
```

## Step 3 — Verify Each Policy Blocks Its Target

Test each policy by attempting to apply a violating resource:

```bash
# Should be blocked by disallow-privileged-containers
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-privileged
  namespace: default
spec:
  containers:
  - name: c
    image: busybox
    securityContext:
      privileged: true
EOF

# Should be blocked by require-approved-registry
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-registry
  namespace: default
spec:
  containers:
  - name: c
    image: docker.io/library/busybox:latest
EOF

# Should be blocked by require-resource-limits
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-no-limits
  namespace: default
spec:
  containers:
  - name: c
    image: your-registry.example.com/busybox:1.36
EOF

# Should be blocked by disallow-host-namespaces
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-hostpid
  namespace: default
spec:
  hostPID: true
  containers:
  - name: c
    image: your-registry.example.com/busybox:1.36
EOF
```

## Step 4 — Create A Justified Exception

Exceptions should be:
- Scoped to a specific workload in a specific namespace
- Time-limited or tied to a tracked remediation
- Documented with a justification and an approver

The policy file includes a PolicyException example. Apply it and confirm
that only the excepted workload bypasses the policy while all others
remain blocked.

## Step 5 — Design An Adoption Plan

Write a one-page policy rollout plan for a team with 50 existing Deployments
that were deployed before these policies existed. Cover:

1. How you audit without breaking anything (hint: start with Audit mode)
2. How you prioritize which violations to fix first
3. How you handle legacy workloads that cannot be immediately fixed
4. How you prevent new violations while remediating old ones
5. How you measure success

## Deliverable

The applied policy set, the exception workflow, the verified test results
for each policy, and your adoption plan draft.

## Assessment Question

A developer submits a PR with an exception request for a container that needs
`privileged: true` because it runs a CNI agent. How do you evaluate the
request? What questions do you ask, and what compensating controls would
you require if you approve it?
