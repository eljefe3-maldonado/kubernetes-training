# Module 03: Pod Hardening And Runtime Controls

## Why This Matters

When an attacker lands inside a container — through a web application exploit,
a compromised dependency, or a stolen image — the pod's security configuration
determines what happens next. A well-hardened pod forces the attacker to work
much harder to escape. A poorly configured pod is one command away from node
access. Pod hardening does not prevent initial compromise. It controls blast
radius after compromise, and at master level, understanding the difference
between settings that actually block attacker capability and settings that only
improve hygiene is what separates a security engineer from someone who just
ran a scanner.

---

## The Attack Narrative

A team deploys a Node.js API that has a known prototype pollution vulnerability.
An attacker exploits it and gains remote code execution inside the container.
The first thing they do is check what they can access.

In a default, unhardened deployment, here is what they find: the container is
running as root (UID 0). The root filesystem is writable, so they can install
tools — `curl`, `ncat`, a static binary of their choice. `allowPrivilegeEscalation`
is not set, so any setuid binary in the image can be used to escalate further.
There are no capability restrictions, so `CAP_NET_RAW` is available, which means
raw socket access and ARP spoofing on the pod network. The service account token
is mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token` and the
service account has broad RBAC.

Within five minutes the attacker has downloaded a `kubectl` binary, authenticated
to the API server with the mounted token, listed all pods cluster-wide, and
identified a pod with a `hostPath` volume. They create a privileged pod of their
own — the existing RBAC allows `create pods` — mount the node filesystem, and
read the kubelet's credentials from `/var/lib/kubelet/config.yaml`. They now
have node-level access.

None of this required a container runtime exploit. The application had a
vulnerability, yes. But every subsequent step was enabled by the pod configuration:
running as root, writable filesystem, no capability restrictions, auto-mounted
service account token with excess RBAC, and no admission policy blocking the
privileged pod creation.

Now replay the same scenario in a hardened pod: non-root UID, `readOnlyRootFilesystem`,
all capabilities dropped, `allowPrivilegeEscalation: false`, `automountServiceAccountToken: false`,
and admission policy blocking privileged pod creation. The attacker still has RCE.
But they cannot install tools (read-only filesystem), cannot use raw sockets
(capabilities dropped), cannot escalate via setuid (privilege escalation blocked),
and cannot call the Kubernetes API (no token). Their blast radius is the
application process. Not the node. Not the cluster.

That is the value of pod hardening.

---

## What Each Setting Actually Does

Pod security settings are not equally valuable. Some block specific attacker
capabilities. Others mainly improve hygiene or satisfy compliance requirements
without materially changing what an attacker can do. Knowing the difference
is essential for prioritizing remediation and for making the case to engineering
teams when they push back.

**`runAsNonRoot: true` and `runAsUser`** prevent the container from running
as UID 0. This matters because UID 0 inside the container maps to root on the
host if the container runtime has a vulnerability. It also matters because
many sensitive files on the node and in mounted volumes are owned by root and
readable only by root. A non-root container cannot read them even if it can
see the path.

**`readOnlyRootFilesystem: true`** prevents the attacker from writing tools,
backdoors, or scripts to the container filesystem. This is one of the highest-
value settings because it directly limits tool staging. An attacker with RCE
but no writable filesystem has to work entirely in memory, which is harder and
more constrained. The tradeoff is that legitimate applications often need to
write to certain paths — but those can be satisfied with `emptyDir` volumes
for specific directories rather than a writable root.

**`allowPrivilegeEscalation: false`** prevents the container process from
gaining more privileges than its parent through setuid binaries or system
calls like `setuid()`. Combined with dropping capabilities, this closes the
escalation path through any setuid binary that may exist in the image.

**`capabilities: drop: [ALL]`** removes every Linux capability from the
container. The default capability set granted to containers includes `NET_RAW`
(raw sockets and ICMP), `NET_BIND_SERVICE` (binding to ports below 1024),
`CHOWN`, `DAC_OVERRIDE`, `KILL`, `SETUID`, `SETGID`, and others. Each of
these has a specific attacker use. `NET_RAW` enables ARP spoofing and ICMP-
based scanning on the pod network. Dropping all capabilities and adding back
only what the application genuinely needs is the right default.

**`seccompProfile: RuntimeDefault`** applies the container runtime's built-in
seccomp filter, which blocks approximately 300 system calls that a typical
container does not need. Many container escape techniques rely on specific
system calls — `unshare`, `clone` with certain flags, `mount`. Seccomp filters
these at the kernel level. `RuntimeDefault` is a safe starting point. Fine-
grained profiles require profiling the application first.

**`hostPID`, `hostNetwork`, `hostIPC`** are all disabled by default and should
stay that way for application workloads. `hostPID` shares the node's process
namespace, allowing the container to see and signal every process on the node.
`hostNetwork` puts the container on the node's network stack, bypassing all
Kubernetes NetworkPolicy. `hostIPC` shares inter-process communication
facilities. Each of these effectively breaks container isolation.

**`hostPath` volumes** are the most direct path to node compromise available
to a workload. A container that can mount `/` from the host can read kubelet
credentials, cloud instance metadata, SSH keys, and anything else on the node
filesystem. A `hostPath` mount of `/var/run/docker.sock` or the containerd
socket is equivalent to root on the node. These should never appear in
application workloads.

---

## The Misconception: Non-Root Means Safe

"We run all our containers as non-root" is one of the most common claims made
by teams that believe their workload security is solid. It addresses one risk
factor — UID 0 privilege — and leaves the others untouched.

A container running as UID 1000 with `privileged: true` is far more dangerous
than a container running as UID 0 with no special privileges. `privileged: true`
disables most seccomp filtering, apparmor profiles, and capability restrictions
entirely. It gives the container near-complete access to the host kernel. The
UID the container runs as does not change this.

Similarly, a non-root container with `hostPath: /` still has full access to
the node filesystem — just as the UID of the process that mounted it, not
as root. If the application runs as UID 1000 and the files it cares about
are world-readable (which many are on a default Linux system), the hostPath
access is just as useful to an attacker.

Non-root is one layer. It is not a substitute for the full restricted baseline.

---

## What Pod Hardening Cannot Solve

Master-level understanding requires knowing the limits of a control, not just
how to apply it. Pod hardening reduces blast radius after compromise. It does
not prevent all compromises, and it does not address several real attack paths.

**Kernel vulnerabilities** that allow container escapes do not require any of
the settings above to be misconfigured. If the kernel has a vulnerability in
a system call path that a restricted container can reach, seccomp with the
right profile can block it, but RuntimeDefault is not guaranteed to cover
novel exploit techniques. Defense-in-depth at the node layer (immutable OS,
timely patching) is necessary alongside pod hardening.

**Stolen service account tokens** are not affected by pod security context
settings. Once an attacker has the token — whether they read it from the
filesystem or stole it through a memory read — the token's RBAC is what
determines the blast radius. Pod hardening does not help here. That is
Module 2's domain.

**Application-layer attacks** — SQL injection, SSRF, authorization bypasses
— are not affected by seccomp or capability settings. An attacker exploiting
an application vulnerability to read database records or call internal APIs
does not need any of the pod settings to be misconfigured.

**Inter-pod network access** is not controlled by pod security context. A
container with all capabilities dropped and a read-only filesystem can still
make TCP connections to other pods on the cluster network unless NetworkPolicy
restricts it. That is Module 4.

---

## Decision Framework: Restricted Baseline Or Justified Exception

The Kubernetes Pod Security Standards Restricted profile is the correct
default for application workloads. The question is not "should I apply
Restricted?" but "why would I not apply Restricted, and what compensating
controls exist for the exception?"

When a team claims they cannot meet the Restricted profile, the first response
is to question the claim. The most common objections:

**"We need to run as root because we bind to port 80."** This is not true.
Binding to ports below 1024 requires `CAP_NET_BIND_SERVICE`, not UID 0. And
even that capability is unnecessary — applications should bind to high ports
(8080, 3000) and Kubernetes Services or the ingress controller handle the
port translation.

**"We need a writable filesystem for logs."** Log to stdout/stderr and let
the container runtime collect them. If the application requires a writable
path, mount an `emptyDir` volume at that path only. Do not make the entire
root filesystem writable.

**"We need privileged for our monitoring agent."** Infrastructure agents
(Falco, node exporters, CNI plugins) often legitimately need elevated access.
These belong in infrastructure DaemonSets with documented exceptions, not in
application namespaces. The exception is the rare case, not the default.

For every exception: document the business justification, the compensating
controls in place, and a review date. Exceptions that exist permanently
without review become permanent vulnerabilities.

---

## What Good Looks Like vs What Compliant Looks Like

A compliant deployment might pass a Pod Security Admission check at the
Baseline level. That means it does not use `privileged: true` and does not
use the most dangerous host namespace settings. The Baseline profile prevents
the most obvious misconfigurations.

A secure deployment meets the Restricted profile: non-root, read-only
filesystem, all capabilities dropped, seccomp enabled, no host namespaces,
no hostPath volumes, resource limits set. More importantly, the team knows
*why* each setting is there — which specific attack it raises the cost of —
and they can explain the residual risks that pod hardening cannot address.

Compliance is meeting the profile. Security is understanding why the profile
exists and what it does not cover.

---

## You Will Learn

- which `securityContext` settings block attacker capability versus improve hygiene
- the Pod Security Standards tiers and what each one prevents
- how capabilities, seccomp, and read-only filesystems interact as a defense
- what pod hardening cannot protect against
- how to handle legitimate exceptions without creating permanent holes

## Key Questions

- Which runtime restrictions materially reduce attacker capability, and which are hygiene?
- What cannot be solved at the pod layer that requires other controls?
- When a team says they need an exception, what do you investigate first?
- What does a compromised container look like in a hardened pod versus an unhardened one?

## Hands-On

- Lab: [Lab 03](../../labs/module-03/README.md)
- Assets:
  - [insecure-deployment.yaml](../../labs/module-03/insecure-deployment.yaml)
  - [hardened-deployment.yaml](../../labs/module-03/hardened-deployment.yaml)

## Output

Produce a restricted-baseline deployment and a residual-risk note explaining
what the hardening addresses and what it does not.
