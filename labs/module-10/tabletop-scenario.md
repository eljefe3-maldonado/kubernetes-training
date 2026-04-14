# Tabletop Scenario: Compromised Namespace Escalation

## Scenario Overview

**Incident ID**: MOCK-001  
**Severity**: P1 — Suspected cluster-admin compromise  
**Environment**: Production Kubernetes cluster, multi-tenant platform  
**Duration**: 100 minutes across four phases  

**Facilitator note**: Read each inject aloud at the indicated time. Pause after
each inject and ask the group for their immediate response before revealing the
next one. Record decisions and rationale as you go.

---

## Background

The affected workload is `api-server`, a Python REST API running in the
`incident-lab` namespace. It processes customer orders and has read access
to a single database credential secret. It has been running in production
for three months with no known issues.

The application's service account (`compromised-app`) has the following RBAC:
- `get` on `secrets/db-credentials` in `incident-lab`
- `get`, `list` on pods in `incident-lab`

---

## Phase 1 — Triage (30 minutes)

### T+0: Initial Alert

```
ALERT [HIGH]: Interactive exec session opened
User:      system:serviceaccount:incident-lab:compromised-app
Source IP: 198.51.100.45
Pod:       api-server-7d9f8b-xkj2p
Namespace: incident-lab
Time:      <scenario start time>
```

**Questions for the group:**

1. What is your first action?
2. Who do you notify immediately?
3. Do you kill the pod or preserve it? Why?
4. What information do you need before you can make that decision?

---

### T+5: First Responder Queries

The responder runs `kubectl get all -n incident-lab` and finds:

```
NAME                              READY   STATUS    RESTARTS
pod/api-server-7d9f8b-xkj2p       1/1     Running   0
pod/data-collector-d4c9f-lmpq8    1/1     Running   0   ← new, not in usual inventory
pod/debug-util-559f6-zr8vn        1/1     Running   0   ← new, not in usual inventory
```

**Questions:**

1. What do you do about the two unexpected pods?
2. How do you determine when they were created and by whom?
3. What do you look at next without deleting anything?

---

### T+10: Audit Log Review

Audit logs for `system:serviceaccount:incident-lab:compromised-app` in the
past 2 hours show:

```
09:14:22  GET     secrets/db-credentials          (normal)
09:47:33  GET     secrets/db-credentials          (normal)
10:02:11  LIST    secrets                          ← unusual: list, not get on named secret
10:02:14  GET     secrets/ci-token                ← unusual: accessed a secret it should not see
10:03:01  CREATE  pods                             ← created data-collector pod
10:03:45  CREATE  pods                             ← created debug-util pod
10:04:22  CREATE  clusterrolebindings              ← CRITICAL
```

**Questions:**

1. How did the service account list secrets when its RBAC only allows `get` on one named secret?
2. What was the CI token doing in this namespace?
3. What ClusterRoleBinding did it create? How do you find out?

---

### T+15: ClusterRoleBinding Discovery

```bash
kubectl get clusterrolebindings -o json | jq '.items[] |
  select(.subjects[]?.name == "attacker-backdoor") |
  {name, role: .roleRef.name, subjects}'
```

Output:
```json
{
  "name": "kube-controller-health",
  "role": "cluster-admin",
  "subjects": [{"kind": "ServiceAccount", "name": "attacker-backdoor", "namespace": "kube-system"}]
}
```

The binding is named `kube-controller-health` — designed to look like a
legitimate system component.

**Questions:**

1. Does the service account `attacker-backdoor` in `kube-system` exist?
2. If it exists, what has it done? If it does not exist, why?
3. What is your containment priority right now?

---

## Phase 2 — Containment (30 minutes)

### T+30: Scope Confirmation

The `attacker-backdoor` service account exists in `kube-system`. Audit logs
show it has performed the following actions in the past 45 minutes:

- `LIST secrets` (cluster-wide) — 3 calls
- `GET secret/kube-system/kube-controller-manager-token` — 1 call
- `CREATE clusterrolebinding/kube-controller-health` — 1 call (itself)
- `GET nodes` (cluster-wide) — 1 call
- `LIST pods` (cluster-wide) — 1 call

No deployments or further workloads created yet.

**Decisions required:**

1. Do you delete `kube-controller-health` ClusterRoleBinding now? What are the risks?
2. Do you delete the `attacker-backdoor` service account? In what order?
3. How do you isolate the original compromised pod?
4. Do you take the cluster offline? Who makes that call?

---

### T+35: Isolation Steps

Perform containment in this order (for the group to execute or discuss):

```bash
# Step 1: Revoke the escalated access first
kubectl delete clusterrolebinding kube-controller-health
kubectl delete serviceaccount attacker-backdoor -n kube-system

# Step 2: Isolate the original compromised pod (block all network, do not delete)
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-compromised
  namespace: incident-lab
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes: [Ingress, Egress]
EOF

# Step 3: Capture the pod state before any further action
kubectl get pod api-server-7d9f8b-xkj2p -n incident-lab -o yaml > /tmp/pod-evidence.yaml
kubectl logs api-server-7d9f8b-xkj2p -n incident-lab > /tmp/pod-logs.txt

# Step 4: Isolate the two unexpected pods
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-data-collector
  namespace: incident-lab
spec:
  podSelector:
    matchLabels:
      app: data-collector
  policyTypes: [Ingress, Egress]
EOF
```

**Questions:**

1. In what order did you perform these steps, and why does the order matter?
2. What evidence did you collect before taking containment actions?
3. What do you tell the application team who is seeing traffic failures?

---

## Phase 3 — Eradication (20 minutes)

### T+60: Evidence Review Complete

You have captured pod specs and logs. You have confirmed:

- The `data-collector` pod was running a cryptocurrency miner
- The `debug-util` pod had mounted the host filesystem (privileged, hostPath: /)
- The RBAC gap that allowed the service account to LIST secrets came from a
  recently merged PR that changed the Role from `get` on a named secret to
  `get, list` on all secrets — it was not reviewed by security

**Eradication checklist — work through these:**

- [ ] Delete all three attacker-created pods
- [ ] Delete the two attacker-created service accounts
- [ ] Confirm the malicious ClusterRoleBinding is gone
- [ ] Rotate the `db-credentials` secret (it was accessed)
- [ ] Rotate the `ci-token` secret (it was accessed)
- [ ] Revert the RBAC Role to `get` on the named secret only
- [ ] Verify no other namespaces have the same RBAC misconfiguration
- [ ] Confirm no new ClusterRoleBindings were created after your containment

---

## Phase 4 — Recovery And Post-Incident Review (20 minutes)

### T+80: Recovery

**Questions for the group:**

1. Before you restore the `api-server` deployment, what do you verify?
2. How do you confirm the `api-server` image itself was not tampered with?
3. What monitoring do you put in place for the next 48 hours?

### T+90: Post-Incident Improvements

The group must agree on exactly three control improvements. Rank them by:
- Impact on preventing recurrence
- Speed of implementation
- Risk of introducing new issues

Document your three choices and the rationale here:

1. Improvement:
   Rationale:
   Owner:
   Target date:

2. Improvement:
   Rationale:
   Owner:
   Target date:

3. Improvement:
   Rationale:
   Owner:
   Target date:

---

## Facilitator Debrief Questions

Use these to close the exercise:

1. Which decision was hardest and why?
2. Where did you have the least information at the time you needed to decide?
3. What would have caught this 30 minutes earlier?
4. What single control, if it had been in place, would have changed the outcome most?
