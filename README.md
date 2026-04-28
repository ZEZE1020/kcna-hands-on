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
- `minikube` or `kind` installed (labs specify which)
- Basic comfort with the terminal

> **Quick environment check:**
> ```bash
> docker version && kubectl version --client && minikube version
> ```

---

## Repository Structure

```
kcna-labs/
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
│   ├── lab-01-cluster-components.md
│   ├── lab-02-api-server-and-kubectl.md
│   └── challenge.md
│
├── module-03-workloads/             ← Pods, Deployments, Jobs, DaemonSets
│   ├── lab-01-pods-deep-dive.md
│   ├── lab-02-deployments-and-rollouts.md
│   ├── lab-03-jobs-and-cronjobs.md
│   ├── lab-04-daemonsets-and-statefulsets.md
│   └── challenge.md
│
├── module-04-networking-and-services/  ← Services, DNS, Ingress, NetworkPolicy
│   ├── lab-01-services-clusterip-nodeport.md
│   ├── lab-02-dns-and-service-discovery.md
│   ├── lab-03-ingress.md
│   ├── lab-04-network-policy.md
│   └── challenge.md
│
├── module-05-cloud-native-architecture/  ← Microservices, service mesh, CRDs
│   ├── lab-01-microservices-decomposition.md
│   ├── lab-02-service-mesh-linkerd.md
│   ├── lab-03-crds-and-operators.md
│   └── challenge.md
│
├── module-06-observability/         ← Logs, metrics, tracing
│   ├── lab-01-structured-logging.md
│   ├── lab-02-prometheus-and-grafana.md
│   ├── lab-03-distributed-tracing-jaeger.md
│   └── challenge.md
│
├── module-07-application-delivery/  ← Helm, GitOps, ArgoCD
│   ├── lab-01-helm-basics.md
│   ├── lab-02-helm-advanced.md
│   ├── lab-03-gitops-argocd.md
│   └── challenge.md
│
└── final-challenge/
    └── README.md                    ← Capstone: deploy a full cloud-native app
```

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

- [ ] Module 01 – Containers
- [ ] Module 02 – Kubernetes Architecture
- [ ] Module 03 – Workloads
- [ ] Module 04 – Networking & Services
- [ ] Module 05 – Cloud Native Architecture
- [ ] Module 06 – Observability
- [ ] Module 07 – Application Delivery
- [ ] Final Challenge

---

## KCNA Exam Tips (from the labs themselves)

> These aren't tips to memorise — they're things you'll *understand* after completing the labs.

- The KCNA is 90 minutes, 60 questions — it tests breadth across the CNCF landscape
- You do NOT need to write code in the exam — but hands-on practice builds the conceptual clarity that multiple-choice questions test
- The exam expects you to know *what* tools in the CNCF ecosystem do, not just Kubernetes itself
- After module 06, you'll understand why observability is its own domain — it's not an afterthought

---

## Contributing

Found something outdated? Labs are intentionally kept tool-version-agnostic where possible. PRs that add a `challenge.md` for any module are especially welcome.

---

*Based on the official [KCNA Exam Curriculum](https://github.com/cncf/curriculum) — last verified against the 2024/2025 exam guide.*
