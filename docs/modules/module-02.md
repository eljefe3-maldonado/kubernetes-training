# Module 02: Identity, RBAC, And Privilege Escalation

## Why This Matters

Many Kubernetes incidents are not about software exploits. They are about
excess authority, token misuse, and under-reviewed RBAC.

## You Will Learn

- service account and user authentication patterns
- RBAC review and least-privilege design
- escalation paths through verbs, bindings, and secret access

## Key Questions

- Which principals can mint more privilege than they appear to have?
- Which permissions create indirect cluster-admin paths?
- Where should cloud IAM federation replace static credential patterns?

## Hands-On

- Lab: [Lab 02](../../labs/module-02/README.md)
- Assets:
  - [over-privileged-rbac.yaml](../../labs/module-02/over-privileged-rbac.yaml)
  - [hardened-rbac.yaml](../../labs/module-02/hardened-rbac.yaml)

## Output

Produce a hardened RBAC model and a review memo documenting the original
escalation paths.
