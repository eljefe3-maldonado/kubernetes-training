# Kubernetes Security Master Class

[![License: MIT](https://img.shields.io/badge/license-MIT-yellow.svg)](./LICENSE)
[![Curriculum](https://img.shields.io/badge/modules-12-blue.svg)](./MASTER_CLASS_CURRICULUM.md)
[![Labs](https://img.shields.io/badge/labs-hands--on-green.svg)](./labs/README.md)
[![Study Guide](https://img.shields.io/badge/track-12_week-orange.svg)](./STUDY_GUIDE.md)

## Overview

This repository is a portable Kubernetes security training program built for
serious practitioner growth rather than lightweight certification prep. It is
designed for Cloud Security Engineers who need to understand how Kubernetes
works, how it fails, how it is attacked, and how to build practical controls
that hold up in real environments.

The material combines:

- a master-level curriculum with 12 modules and a capstone
- delivery schedules for part-time and intensive formats
- hands-on labs with manifests, policy examples, worksheets, and scenarios
- security-focused outcomes across prevention, detection, response, and
  governance

## Start Here

Choose the path that matches how you want to use the repository:

| If You Want To... | Start Here |
| --- | --- |
| Follow the full course in sequence | [STUDY_GUIDE.md](./STUDY_GUIDE.md) |
| Review the complete program design | [MASTER_CLASS_CURRICULUM.md](./MASTER_CLASS_CURRICULUM.md) |
| Jump into hands-on work | [labs/README.md](./labs/README.md) |
| Browse module-by-module docs | [docs/modules/README.md](./docs/modules/README.md) |

## Learning Experience

This repo is organized around four learning layers:

1. Curriculum: what to learn and why it matters
2. Modules: focused teaching pages for each topic
3. Labs: hands-on exercises with manifests and artifacts
4. Study guide: a week-by-week execution plan

## Tracks

| Track | Pace | Best For | Entry Point |
| --- | --- | --- | --- |
| Self-study | 12 weeks | Individual learning with steady weekly progress | [STUDY_GUIDE.md](./STUDY_GUIDE.md) |
| Intensive | 12 business days | Boot camp or team sprint delivery | [MASTER_CLASS_CURRICULUM.md#delivery-schedules](./MASTER_CLASS_CURRICULUM.md#delivery-schedules) |
| Module review | Flexible | Targeted topic review or interview prep | [docs/modules/README.md](./docs/modules/README.md) |

## Audience

- Cloud Security Engineers building or reviewing Kubernetes platforms
- Security Architects responsible for container and platform controls
- DevSecOps engineers embedding preventive and detective controls into delivery
  pipelines
- Senior engineers preparing to lead Kubernetes security programs

## Program Goals

By the end of this program, the learner should be able to:

- explain Kubernetes internals well enough to reason about attack paths and
  failure domains
- design secure multi-tenant cluster patterns and hardened workload baselines
- enforce least privilege across Kubernetes RBAC, IAM integrations, secrets,
  network flows, and admission control
- build policy-as-code controls for prevention, compliance, and secure delivery
- detect, investigate, and contain Kubernetes-focused incidents
- evaluate managed Kubernetes and self-managed cluster security tradeoffs
- lead security reviews for platform, workload, and supply chain risk

## How To Use This Program

1. Start with the curriculum in `MASTER_CLASS_CURRICULUM.md`.
2. Treat each module as a teachable block with lecture, reading, lab, and
   assessment expectations.
3. Use `labs/README.md` to move from theory into hands-on execution.
4. Use the capstone and operating model sections to convert the training into a
   job-relevant portfolio.

## Program Structure

- Level: Master / advanced practitioner
- Format: 12 modules plus capstone
- Suggested pace: 8 to 12 weeks part-time, or 2 to 3 weeks full-time intensive
- Output: security architecture fluency, hands-on hardening skills, and
  incident-response capability in Kubernetes environments

## Module Map

| Module | Theme | Module Doc | Lab |
| --- | --- | --- | --- |
| 01 | Architecture and attack surface | [Module 01](./docs/modules/module-01.md) | [Lab 01](./labs/module-01/README.md) |
| 02 | Identity, RBAC, and escalation | [Module 02](./docs/modules/module-02.md) | [Lab 02](./labs/module-02/README.md) |
| 03 | Pod hardening and runtime controls | [Module 03](./docs/modules/module-03.md) | [Lab 03](./labs/module-03/README.md) |
| 04 | Network isolation and exposure control | [Module 04](./docs/modules/module-04.md) | [Lab 04](./labs/module-04/README.md) |
| 05 | Secrets and sensitive data handling | [Module 05](./docs/modules/module-05.md) | [Lab 05](./labs/module-05/README.md) |
| 06 | Admission control and policy-as-code | [Module 06](./docs/modules/module-06.md) | [Lab 06](./labs/module-06/README.md) |
| 07 | Supply chain and image assurance | [Module 07](./docs/modules/module-07.md) | [Lab 07](./labs/module-07/README.md) |
| 08 | Cluster hardening and baselines | [Module 08](./docs/modules/module-08.md) | [Lab 08](./labs/module-08/README.md) |
| 09 | Detection engineering and hunting | [Module 09](./docs/modules/module-09.md) | [Lab 09](./labs/module-09/README.md) |
| 10 | Incident response and forensics | [Module 10](./docs/modules/module-10.md) | [Lab 10](./labs/module-10/README.md) |
| 11 | Governance and multi-tenancy | [Module 11](./docs/modules/module-11.md) | [Lab 11](./labs/module-11/README.md) |
| 12 | Cloud mappings and leadership | [Module 12](./docs/modules/module-12.md) | [Lab 12](./labs/module-12/README.md) |

## Repository Layout

```text
kubernetes-training/
├── docs/
│   └── modules/
├── README.md
├── MASTER_CLASS_CURRICULUM.md
├── STUDY_GUIDE.md
├── AGENTS.md
├── AGENT_CONTEXT.md
└── labs/
    ├── README.md
    ├── module-01/
    ├── module-02/
    ├── module-03/
    ├── module-04/
    ├── module-05/
    ├── module-06/
    ├── module-07/
    ├── module-08/
    ├── module-09/
    ├── module-10/
    ├── module-11/
    └── module-12/
```

## Primary Document

- `MASTER_CLASS_CURRICULUM.md`: complete course outline, labs, deliverables, and
  evaluation criteria
- `STUDY_GUIDE.md`: week-by-week self-study plan aligned to the curriculum and labs
- `docs/modules/README.md`: module navigation and focused topic pages
- `labs/README.md`: index of all hands-on labs and supporting assets
- `LICENSE`: permissive license for publishing and reuse

## Publish To GitHub

If you want this directory available from anywhere, publish this folder as its
own repository named `kubernetes-training`.

```bash
cd /home/grasshopper/Projects/kubernetes-training
git init
git add .
git commit -m "Initial commit: Kubernetes security master class"
git branch -M main
git remote add origin git@github.com:eljefe3-maldonado/kubernetes-training.git
git push -u origin main
```

## Suggested Next Steps

- publish the repository to GitHub
- add screenshots or diagrams for the lab flow if you want a stronger landing page
- add issues or project boards if you want to teach from this as a living program
