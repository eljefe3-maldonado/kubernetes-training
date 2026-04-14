# Module 01: Architecture And Attack Surface

## Why This Matters

Security engineering in Kubernetes starts with knowing which components hold
authority, which paths carry trust, and how a single foothold becomes broader
cluster impact.

## You Will Learn

- control-plane trust boundaries
- high-value components such as the API server, etcd, kubelet, and ingress
- common workload-to-cluster abuse paths

## Key Questions

- What component is the attacker trying to reach next?
- Which communication paths are strongest detection opportunities?
- Which compromises change impact from namespace to cluster scope?

## Hands-On

- Lab: [Lab 01](../../labs/module-01/README.md)
- Asset: [trust-map-template.md](../../labs/module-01/trust-map-template.md)

## Output

Produce a trust map with annotated abuse paths and explain the shortest route
from compromised workload to cluster-wide impact.
