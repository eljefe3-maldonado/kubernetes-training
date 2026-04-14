# Module 04: Network Isolation And Exposure Reduction

## Why This Matters

Kubernetes networking is one of the most misunderstood layers in cluster
security. Good design reduces lateral movement and accidental exposure.

## You Will Learn

- namespace isolation patterns
- application-to-database restriction
- egress control and its limitations

## Key Questions

- What remains reachable after NetworkPolicy is applied?
- Where do ingress, DNS, and egress still create bypass paths?
- Which traffic deserves hard denial by default?

## Hands-On

- Lab: [Lab 04](../../labs/module-04/README.md)
- Assets:
  - [default-deny.yaml](../../labs/module-04/default-deny.yaml)
  - [allow-app-to-db.yaml](../../labs/module-04/allow-app-to-db.yaml)
  - [egress-restrict.yaml](../../labs/module-04/egress-restrict.yaml)

## Output

Produce a network policy set and a short gap analysis of remaining exposure.
