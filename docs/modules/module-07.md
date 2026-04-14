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

---

## Self-Assessment

**Question 1**
Your CI pipeline scans images with Trivy and produces a clean report with no
critical CVEs. A week later, one of those images is found to contain a crypto
miner. How is this possible, and what control would have caught it?

*A strong answer covers:* CVE scanners detect known vulnerabilities in known
packages. Malicious code injected by a compromised dependency maintainer has
no CVE — it is novel behavior, not a known vulnerability. Trivy would report
clean because no CVE is assigned to intentionally injected malicious code.
Controls that could catch it: SBOM comparison between builds (an unexpected
new package or behavior change in an existing package triggers review), runtime
behavioral detection via Falco (unusual process execution or outbound connections),
or egress NetworkPolicy that blocks the miner's C2 connection. Signing and
attestation prove the image was built by the trusted pipeline but do not
verify the content of what was built.

---

**Question 2**
A developer pins their base image as `FROM python:3.11-slim`. Another developer
says this is sufficient for supply chain security. What is wrong, and what is
the correct practice?

*A strong answer covers:* image tags are mutable — `python:3.11-slim` can
point to a different image tomorrow if the upstream maintainer updates it.
Pinning by tag does not guarantee the same image digest on every build. The
correct practice is pinning by digest: `FROM python@sha256:abc123...`. The
SHA256 digest is cryptographically tied to a specific image content — if the
content changes, the digest changes. The team should pull the pinned digest
from an internal mirror that has been scanned and approved, not from Docker
Hub directly on every build. This closes both the mutable-tag and the public-
registry-dependency attack surfaces.

---

**Question 3**
You implement Cosign image signing in your CI pipeline with a policy in
Kyverno that verifies signatures at admission. An attacker compromises the
container registry and overwrites the `v2.1.0` tag with a malicious image.
Does your signature verification catch this?

*A strong answer covers:* yes — this is the core guarantee of image signing.
The malicious image the attacker pushed has a different content digest than
the original. The Cosign signature is attached to the original digest, not
the tag. When Kyverno verifies the signature, it checks whether the current
image digest has a valid signature from the trusted key. The overwritten image
has no valid signature and the admission policy blocks it. This is why signing
by digest (not tag) is essential and why registry immutable tags are
defense-in-depth but not a substitute for signature verification.

---

**Question 4**
Your team uses a registry promotion workflow: CI pushes to `staging-registry`,
images are scanned and promoted to `prod-registry` only if they pass the
quality gate. An attacker compromises a developer's CI credentials and pushes
a malicious image to `staging-registry` with a clean scan result. Can the
malicious image reach production?

*A strong answer covers:* it depends on the quality gate. If the gate is
only a CVE scan, a malicious image with no known CVEs passes and gets
promoted. Controls that would stop it: SBOM comparison against the previous
build (unexpected package additions trigger review), behavioral analysis at
build time, or manual sign-off on promotion. Signing does not help here
because the attacker has CI credentials and can sign the malicious image
with the trusted key. This is why detection controls (SBOM comparison, runtime
detection) are necessary alongside preventive ones — a compromised build
system can produce valid-looking artifacts.

---

**Question 5**
A developer argues that requiring version-pinned dependencies with lock file
hash verification "slows down development" and asks for an exception. Construct
the security case for why this control is non-negotiable for production
workloads.

*A strong answer covers:* unpinned dependencies (`>=1.2.0` or `latest`) allow
CI to install any package version published after the lock file was created.
A compromised maintainer account can publish a malicious patch version that
matches the unpinned specification and is installed automatically. This is
the exact mechanism in the attack narrative (PyPI compromise). The developer
workflow cost is real but bounded — pin once, update on explicit decision.
The security benefit is that no new dependency version can enter the build
without explicit developer action and CI pipeline review. Frame it as: you
cannot secure the supply chain if you don't control what enters it.

---

**Question 6**
After a supply chain incident, leadership asks: "What controls would have
prevented this?" Given a scenario where a malicious npm package was installed
via an unpinned dependency, mapped through a clean scan, and deployed as
a signed image, identify the earliest point in the chain where a control
could have stopped it, and which control that is.

*A strong answer covers:* the earliest prevention point is dependency version
pinning with lock file hash verification — if the dependency version is pinned
to a specific hash, the malicious version would not install even if published.
The next detection point is SBOM generation and comparison: the malicious
package would appear as an unexpected change in the SBOM between builds. The
latest detection point is runtime behavioral analysis: Falco detecting unusual
process execution or outbound connections. Notes that signing would not have
caught this because the attacker's code was built by the legitimate CI system
and signed with the trusted key. The lesson: supply chain security requires
controls at the dependency layer, not just at the image layer.
