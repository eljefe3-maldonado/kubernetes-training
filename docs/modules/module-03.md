# Module 03: Pod Hardening And Runtime Controls

## Why This Matters

Poor workload defaults give attackers a much easier post-compromise path.
Pod-level restrictions do not solve everything, but they shape blast radius.

## You Will Learn

- how `securityContext` changes attacker capability
- where restricted pod settings meaningfully reduce risk
- which dangerous workload patterns should be blocked by default

## Key Questions

- Which workload settings block escalation versus just improve hygiene?
- What cannot be solved by pod configuration alone?
- Which runtime controls should be enforced centrally?

## Hands-On

- Lab: [Lab 03](../../labs/module-03/README.md)
- Assets:
  - [insecure-deployment.yaml](../../labs/module-03/insecure-deployment.yaml)
  - [hardened-deployment.yaml](../../labs/module-03/hardened-deployment.yaml)

## Output

Produce a restricted-baseline deployment and a residual-risk note.
