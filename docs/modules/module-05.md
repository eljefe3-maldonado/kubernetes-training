# Module 05: Secrets And Sensitive Data Exposure

## Why This Matters

Secret compromise often happens through ordinary operational choices such as
over-broad RBAC, environment variables, logs, or mounted data.

## You Will Learn

- native secret limitations
- encryption and external secret-manager patterns
- common runtime and operational leak paths

## Key Questions

- Where do secrets leak first in real clusters?
- Which controls are preventive, and which are only compensating?
- How should regulated workloads handle secret delivery?

## Hands-On

- Lab: [Lab 05](../../labs/module-05/README.md)
- Assets:
  - [leaky-deployment.yaml](../../labs/module-05/leaky-deployment.yaml)
  - [safer-secret-delivery.yaml](../../labs/module-05/safer-secret-delivery.yaml)

## Output

Produce a safer secret-handling design and explain why the original approach was risky.
