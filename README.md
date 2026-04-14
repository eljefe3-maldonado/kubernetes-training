# Kubernetes Security Master Class

This directory contains a master-level Kubernetes training program tailored for
Cloud Security Engineers who need to secure, operate, assess, and respond in
modern container platforms.

[![License: MIT](https://img.shields.io/badge/license-MIT-yellow.svg)](./LICENSE)
[![Curriculum](https://img.shields.io/badge/modules-12-blue.svg)](./MASTER_CLASS_CURRICULUM.md)
[![Labs](https://img.shields.io/badge/labs-hands--on-green.svg)](./labs/README.md)

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

## Repository Layout

```text
kubernetes-training/
├── README.md
├── MASTER_CLASS_CURRICULUM.md
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
