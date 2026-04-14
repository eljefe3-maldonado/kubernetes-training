# Module 07: Supply Chain Security And Image Trust

## Why This Matters

Everything built in the previous modules assumes the container image running
in your cluster is the one you intended to run. Supply chain attacks break
that assumption before the workload ever starts. A compromised dependency,
a typosquatted base image, a build pipeline with write access to the registry
— any of these can deliver malicious code that passes every runtime control
you have configured, because the malicious code is the application. This
module is about building a chain of custody from source code to running
container that an attacker cannot silently break.

---

## The Attack Narrative

A team maintains a Python API service. Their `requirements.txt` references a
popular data parsing library. The library's maintainer account on PyPI is
compromised, and the attacker publishes a new patch version of the library
that includes a crypto miner that activates only in cloud environments.

The team's CI pipeline runs `pip install -r requirements.txt` without pinning
versions. The malicious version is installed. The image is built, tagged as
`v2.4.1`, pushed to the internal registry, and deployed to production. The
deployment passes every admission check: approved registry, valid tag, resource
limits set, non-root user, no privileged access.

The crypto miner runs. It uses outbound TCP on port 4444 to reach a mining
pool. The NetworkPolicy in this namespace does not block outbound TCP on that
port because the team had not anticipated restricting egress to specific ports
beyond port 443.

The incident is detected forty-three days later when a cloud cost anomaly
alert fires on unusually high CPU utilization.

What would have caught this? Not the admission policy — the image came from
the approved registry. Not the pod security context — the miner ran as the
application process. The controls that could have caught it:

- **Dependency pinning with hash verification** would have blocked the
  installation of the new version without explicit approval.
- **SBOM generation and comparison** at build time would have flagged an
  unexpected new dependency in the image.
- **Egress NetworkPolicy scoped to port 443 only** would have blocked the
  mining pool connection.
- **Falco rule detecting outbound connections to port 4444** would have
  alerted within minutes of the first connection.

No single control catches this. Defense-in-depth does.

---

## The Trust Chain From Source To Running Container

A secure supply chain is a chain of trust. Each link must be verifiable,
and the chain must be continuous — no gaps where an unverified artifact can
be substituted. The links in the chain are:

**Source code**: the starting point. Trust here means the code that was
reviewed and merged is the code that entered the build. Version pinning,
dependency lock files with hash verification, and protected branches that
require code review are the controls.

**Build environment**: the CI system that builds the image must be trusted.
An attacker who can modify the build environment — inject code during the
build step, modify the Dockerfile, alter the base image pull — can produce
a malicious image from legitimate source code. Build environment integrity
means: isolated build runners, no persistent state between builds, build
steps that are defined in version-controlled configuration, and build logs
that are retained for audit.

**Base image**: every application image is built on a base image. If the
base image is pulled from a public registry without verification, the build
trusts whatever happens to be at that tag today. Tags on public registries
are mutable — the same tag can point to a different image tomorrow. Pinning
by digest (`FROM ubuntu@sha256:...`) makes the base image reference immutable.
Pulling base images from an internal mirror that has been scanned and promoted
ensures that the base image was verified before it entered your supply chain.

**Image registry**: the registry is where built images are stored before
deployment. An attacker with write access to the registry can push a modified
image with the same tag. Registry immutable tags (a feature on ECR, Artifact
Registry, and Azure Container Registry) prevent overwriting an existing tag.
Registry access controls (who can push, to which repository) limit the attack
surface.

**Signing and attestation**: image signing with Cosign creates a
cryptographically verifiable record that a specific image digest was
produced by a specific signing key at a specific time. An admission policy
that verifies signatures before allowing a pod to run ensures that only images
produced by trusted build systems can be deployed. Attestations extend this
to include SBOMs and vulnerability scan results — the admission policy can
check not just that the image was signed, but that it was signed by the CI
system and has a clean vulnerability scan attached.

**Registry promotion**: rather than deploying images directly from the build
registry, a promotion workflow moves images from a staging registry (where
CI pushes) to a production registry (from which deployments pull) only after
passing defined quality gates: scan results, attestation verification, and
approval. This creates a clear checkpoint where security review occurs.

**Admission**: the final gate before the image runs. An admission policy
verifying the Cosign signature ensures that even if someone bypasses the
promotion workflow, they cannot deploy an unsigned image.

---

## The Misconception: CVE Scanning Means The Supply Chain Is Secure

Vulnerability scanning is valuable and necessary. It is not sufficient on its
own, and conflating "no critical CVEs" with "supply chain is secure" is one
of the most common misconceptions in this domain.

CVE scanning detects known vulnerabilities in known packages. It does not
detect:
- Malicious code injected into a package that has no CVE assigned to it
- Compromised dependencies that have not yet been identified as compromised
- Build pipeline modifications that inject code after the scan runs
- Malicious base images that have no vulnerabilities but contain backdoors

The SolarWinds attack used a legitimate build process to insert malicious
code. The resulting binary would have passed a CVE scan. The event log
manipulation was not a CVE — it was novel malicious behavior.

Signing and attestation provide a different guarantee: not "this image has
no known vulnerabilities" but "this image was built by our CI system, from
this specific commit, at this specific time, and has not been modified since."
These are complementary guarantees, not competing ones.

---

## Decision Framework: Preventive vs Detective Supply Chain Controls

When evaluating supply chain controls, classify each as preventive (stops
the bad artifact from running) or detective (identifies it after the fact).
A mature supply chain needs both.

**Preventive controls** (stop deployment of bad artifacts):
- Image signature verification at admission — unsigned images do not run
- Registry allowlist at admission — images from unapproved registries do not run
- Immutable tags — once pushed, a tag cannot be overwritten
- Dependency pinning with hash verification — unexpected dependency versions
  do not install silently

**Detective controls** (identify problems after they occur):
- CVE scanning at build time and continuously on deployed images
- SBOM comparison between builds — unexpected new packages trigger review
- Runtime behavioral detection (Falco) — unusual process execution or
  network connections from the application process
- Egress NetworkPolicy scoped to known-good destinations

For each control, ask: if an attacker compromises this stage of the pipeline,
does this control prevent the attack, detect it, or only reduce its impact?
A comprehensive supply chain policy answers this question for every stage.

---

## What Good Looks Like vs What Compliant Looks Like

A compliant supply chain might have image vulnerability scanning in CI and
an approved registry policy in admission. A secure supply chain has:
dependency pinning with lock files, base images pulled from an internal mirror
pinned by digest, image signing at build time with a KMS-backed key, signature
verification at admission, SBOM generation and attestation, a registry
promotion workflow with defined quality gates, and continuous scanning of
deployed images with alerts on newly discovered vulnerabilities.

The test for a secure supply chain is not "do we scan our images?" but "if
our build pipeline is compromised tomorrow and produces a malicious image,
which control would catch it, and how quickly?"

---

## You Will Learn

- the stages of the supply chain and where each can be attacked
- how image signing and attestation create verifiable chain of custody
- what SBOM generation and vulnerability scanning can and cannot detect
- how registry promotion creates a security checkpoint before deployment
- which supply chain controls are preventive versus detective

## Key Questions

- Where can the chain of trust be broken between source code and running container?
- Which controls actually prevent a bad image from running versus which only detect it?
- If your build pipeline is compromised today, which control catches it?
- What does a compromised-dependency scenario look like in your current environment?

## Hands-On

- Lab: [Lab 07](../../labs/module-07/README.md)
- Asset: [image-signing-policy.yaml](../../labs/module-07/image-signing-policy.yaml)

## Output

Produce an image trust policy covering build, registry, and deployment stages,
and analyze a compromised-image scenario to identify which controls would have
prevented, detected, or only reduced the impact.
