# Module 07: Supply Chain Security And Image Trust

## Why This Matters

Strong runtime policy is not enough if the image or build pipeline was already
compromised before deployment.

## You Will Learn

- provenance and trusted build concepts
- signing, registry promotion, and admission checks
- what SBOM and vulnerability tooling can and cannot prove

## Key Questions

- Where can trust be broken between source and running workload?
- Which controls prevent bad images from deploying?
- Which controls merely improve visibility after the fact?

## Hands-On

- Lab: [Lab 07](../../labs/module-07/README.md)
- Asset: [image-signing-policy.yaml](../../labs/module-07/image-signing-policy.yaml)

## Output

Produce an image trust policy spanning build, registry, and deployment.
