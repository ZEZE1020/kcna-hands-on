# 🧭 KCNA Hands-On Lab Guide

> **Progressive, challenge-driven prep for the Kubernetes and Cloud Native Associate (KCNA) exam.**  
> No practice questions. No passive reading. Every concept earned through hands-on work.

---

## What This Is

This guide maps directly to the **CNCF KCNA exam domains** and translates each topic into a lab you can run yourself. Labs are designed to reflect the *type of thinking* the KCNA exam tests — not memorisation, but understanding *why* things work the way they do.

Each module follows a pattern:
1. **Setup** – spin up the environment
2. **Guided Lab** – step-by-step with explanation
3. **Challenge** – you solve it with hints only
4. **Teardown** – clean up, understand what you built

---

## Exam Domain Coverage

| Domain | Weight | Modules |
|--------|--------|---------|
| Kubernetes Fundamentals | 46% | 02, 03, 04 |
| Container Orchestration | 22% | 01, 03 |
| Cloud Native Architecture | 16% | 05 |
| Cloud Native Observability | 8% | 06 |
| Cloud Native Application Delivery | 8% | 07 |

---

## Prerequisites

- A Linux/macOS machine or cloud VM (2 CPU, 4GB RAM minimum)
- Docker installed
- `kubectl` installed
- `minikube` or `kind` installed (see [setup/environment.md](setup/environment.md))
- Basic comfort with the terminal

> **Quick environment check:**
> ```bash
> docker version && kubectl version --client && kind version
> ```

---

## Repository Structure

```
kcna-hands-on/
├── README.md                        ← You are here
├── setup/
│   └── environment.md               ← Full environment setup guide
│
├── module-01-containers/            ← Container fundamentals
│   ├── lab-01-build-and-run.md
│   ├── lab-02-layers-and-cache.md
│   └── challenge.md
│
├── module-02-kubernetes-architecture/  ← Control plane & node internals
│   └── lab-01-cluster-components.md
│
├── module-03-workloads/             ← Pods, Deployments, Jobs, DaemonSets
│   ├── lab-01-pods-deep-dive.md
│   ├── lab-02-deployments-and-rollouts.md
│   └── challenge.md
│
├── module-04-networking-and-services/  ← Services, DNS, Ingress, NetworkPolicy
│   └── lab-01-services-dns-networkpolicy.md
│
├── module-05-cloud-native-architecture/  ← Microservices, service mesh, CRDs
│   └── lab-01-architecture.md
│
├── module-06-observability/         ← Logs, metrics, tracing
│   └── lab-01-logging-metrics-tracing.md
│
├── module-07-application-delivery/  ← Helm, GitOps, ArgoCD
│   └── lab-01-helm-and-gitops.md
│
└── final-challenge/
    └── README.md                    ← Capstone: deploy a full cloud-native app
```

> **Note:** This is a partial curriculum with core labs. Additional modules and advanced topics may be added in future updates.

---

## How to Use This Guide

### Option A — Linear (Recommended for Beginners)
Work through modules 01 → 07 in order. Each builds on the previous.

### Option B — Domain-Targeted
If you're weak in a specific KCNA domain, jump directly to the relevant module.

### Option C — Challenge-First
Attempt each module's `challenge.md` first. If you get stuck, go back through the guided labs.

---

## Environment Setup

See [`setup/environment.md`](setup/environment.md) for:
- Local setup with `minikube` or `kind`
- Cloud playground options (Killercoda, Play with Kubernetes)
- Resource requirements per module

---

## Progress Tracker

Use this checklist as you work through the labs:

- [ ] Module 01 – Containers (3 labs + challenge)
- [ ] Module 02 – Kubernetes Architecture (1 lab)
- [ ] Module 03 – Workloads (2 labs + challenge)
- [ ] Module 04 – Networking & Services (1 lab)
- [ ] Module 05 – Cloud Native Architecture (1 lab)
- [ ] Module 06 – Observability (1 lab)
- [ ] Module 07 – Application Delivery (1 lab)
- [ ] Final Challenge

---

## KCNA Exam Tips

> **Disclaimer:** These insights emerge from the labs themselves. Always cross-reference with the current [CNCF KCNA exam guide](https://training.linuxfoundation.org/certification/kubernetes-cloud-native-associate/) for the latest exam details.

- The KCNA tests breadth across the CNCF landscape, not deep Kubernetes API knowledge
- You do NOT need to write code in the exam — but hands-on practice builds the conceptual clarity that multiple-choice questions test
- The exam expects you to know *what* tools in the CNCF ecosystem do, not just Kubernetes itself
- After module 06, you'll understand why observability is its own domain — it's not an afterthought
- Focus on understanding *why* patterns exist (e.g., 12-factor apps, declarative infrastructure) rather than memorising commands

---

## Contributing

Found something outdated? Labs are intentionally kept tool-version-agnostic where possible. PRs that add a `challenge.md` for any module are especially welcome.

---

*Based on the official [KCNA Exam Curriculum](https://github.com/cncf/curriculum) — last verified against the 2024/2025 exam guide.*
