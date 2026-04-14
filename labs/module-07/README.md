# Lab 07: Software Supply Chain And Image Trust

## Objective

Design an image trust policy that combines build provenance, image signing,
registry promotion, and deployment admission checks. Trace a compromised-image
scenario and identify which controls would have prevented it, detected it, or
only reduced its impact.

## Prerequisites

- Cosign installed (`cosign version`)
- A registry you can push to (or use a local registry via `docker run -d -p 5000:5000 registry:2`)
- Kyverno installed (reuse from Lab 06)

## Files

| File | Purpose |
| --- | --- |
| `image-signing-policy.yaml` | Kyverno policy requiring signed images with verified attestations |

## Step 1 — Understand The Trust Chain

Map the stages from source code to running container. For each stage, identify
the trust assumptions and the attack opportunities.

| Stage | What Happens | Trust Assumption | Attack Opportunity |
| --- | --- | --- | --- |
| Developer commits code | | | |
| CI pipeline triggers | | | |
| CI builds the image | | | |
| CI pushes to staging registry | | | |
| Promotion job runs | | | |
| Image pushed to production registry | | | |
| Admission controller checks image | | | |
| Kubelet pulls and runs image | | | |

## Step 2 — Sign An Image With Cosign

```bash
# Generate a signing key pair (in production, use KMS-backed keys)
cosign generate-key-pair

# Build a test image and push it to your registry
docker build -t localhost:5000/test-app:v1.0.0 .
docker push localhost:5000/test-app:v1.0.0

# Sign the image
cosign sign --key cosign.key localhost:5000/test-app:v1.0.0

# Verify the signature
cosign verify --key cosign.pub localhost:5000/test-app:v1.0.0
```

## Step 3 — Generate And Attach An SBOM

```bash
# Generate an SBOM in SPDX format
syft localhost:5000/test-app:v1.0.0 -o spdx-json > sbom.spdx.json

# Attach the SBOM as a Cosign attestation
cosign attest --predicate sbom.spdx.json \
  --type spdxjson \
  --key cosign.key \
  localhost:5000/test-app:v1.0.0

# Verify the attestation
cosign verify-attestation \
  --type spdxjson \
  --key cosign.pub \
  localhost:5000/test-app:v1.0.0
```

## Step 4 — Apply The Admission Policy

```bash
kubectl apply -f image-signing-policy.yaml
```

Test that unsigned images are rejected:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: unsigned-test
spec:
  containers:
  - name: c
    image: localhost:5000/unsigned-app:latest
EOF
# Expected: admission webhook denied
```

Test that a properly signed image is accepted:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: signed-test
spec:
  containers:
  - name: c
    image: localhost:5000/test-app:v1.0.0
EOF
# Expected: pod created
```

## Step 5 — Compromised-Image Scenario Analysis

A production incident occurs: a compromised dependency was introduced into
`test-app:v1.0.1` between the build and the registry push. The image was
pushed to the production registry and deployed before anyone noticed.

For each control below, determine whether it would have **prevented** the
incident, **detected** it, or **only reduced impact** — and why.

| Control | Prevented / Detected / Reduced Impact | Explanation |
| --- | --- | --- |
| Image signing at build time | | |
| Signature verification at admission | | |
| Registry tag immutability | | |
| SBOM generation and attestation | | |
| Vulnerability scanning in CI | | |
| Vulnerability scanning at admission | | |
| Runtime anomaly detection (unexpected syscall) | | |
| Immutable container filesystem (readOnlyRootFilesystem) | | |
| NetworkPolicy restricting egress | | |
| Registry promotion from staging to production | | |

## Step 6 — Design The Full Trust Policy

Document an end-to-end image trust policy for your organization. Cover:

1. What must happen in CI before an image is eligible for production
2. How the production registry enforces these requirements
3. What the admission controller verifies at deploy time
4. How you handle emergency patches that need to skip the normal process
5. What your rollback procedure is if a signed image is later found to be compromised

## Deliverable

The completed trust chain map, scenario analysis table, and your organization's
draft image trust policy document.

## Assessment Question

A team argues that vulnerability scanning in CI is sufficient and image signing
adds friction without proportional security benefit. Construct a specific attack
scenario where vulnerability scanning alone would not have caught the compromise
but image signing and attestation verification would have.
