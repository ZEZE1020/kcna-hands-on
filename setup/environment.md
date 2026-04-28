# Environment Setup

Pick one of the following setups. **Option A (kind)** is recommended — it's lightweight, fast, and closest to real multi-node clusters.

---

## Option A — `kind` (Kubernetes in Docker) — Recommended

### Install kind
```bash
# macOS
brew install kind

# Linux
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind
```

### Create a multi-node cluster (used in most labs)
```bash
cat <<EOF | kind create cluster --name kcna-labs --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF
```

### Verify
```bash
kubectl cluster-info --context kind-kcna-labs
kubectl get nodes
```

Expected output:
```text
NAME                      STATUS   ROLES           AGE
kcna-labs-control-plane   Ready    control-plane   1m
kcna-labs-worker          Ready    <none>          1m
kcna-labs-worker2         Ready    <none>          1m
```

### Teardown
```bash
kind delete cluster --name kcna-labs
```

---

## Option B — `minikube`

```bash
minikube start --nodes 2 --cpus 2 --memory 4096 --driver docker
kubectl get nodes
```

---

## Option C — Cloud Playground (No Install Required)

| Platform | URL | Notes |
|----------|-----|-------|
| Killercoda | https://killercoda.com | Free, browser-based, pre-built K8s environments |
| Play with Kubernetes | https://labs.play-with-k8s.com | 4-hour sessions, multi-node |
| KodeKloud Playground | https://kodekloud.com/playgrounds | Good for longer sessions |

> All labs in this guide work on any of the above. Killercoda is the fastest to get started.

---

> **🚨 Real-World Engineering Tip: The CNCF Landscape Moves Fast**

> Cloud-native tools release notoriously fast. If an installation command (like a `kubectl apply -f https://.../v1.x/install.yaml`) fails or times out in the future, don’t panic! That is a core engineering lesson.

> 1. Check the official docs or the **Artifact Hub** for the updated chart or manifest link.

> 2. Ensure your Kubernetes cluster version is compatible with the latest tool version.


## Required Tools

Install these before starting:

```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Helm (needed for Module 07)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# k9s (optional but great for visualising cluster state)
brew install derailed/k9s/k9s  # macOS
# or: https://github.com/derailed/k9s/releases
```

### Verify everything
```bash
kubectl version --client
helm version
docker version
kind version
```

---

## Resource Requirements Per Module

| Module | Min RAM | Notes |
|--------|---------|-------|
| 01 – Containers | 1GB | Docker only, no cluster needed |
| 02 – Architecture | 2GB | Single-node kind cluster |
| 03 – Workloads | 2GB | Single-node kind cluster |
| 04 – Networking | 2GB | Multi-node kind cluster |
| 05 – Cloud Native Arch | 3GB | Linkerd service mesh |
| 06 – Observability | 4GB | Prometheus + Grafana + Jaeger |
| 07 – App Delivery | 2GB | Helm + ArgoCD |

---

## Aliases (Quality of Life)

Add these to your `~/.bashrc` or `~/.zshrc`:

```bash
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgn='kubectl get nodes'
alias kgs='kubectl get svc'
alias kdp='kubectl describe pod'
alias kaf='kubectl apply -f'
alias kdelf='kubectl delete -f'
export do='--dry-run=client -o yaml'   # k run nginx --image=nginx $do
export now='--force --grace-period 0'  # k delete pod nginx $now
```

Source them: `source ~/.bashrc`

---

## Namespace Convention for Labs

Every module uses its own namespace so labs don't interfere:

```bash
kubectl create namespace module-01
kubectl create namespace module-02
# ... etc
```

Each lab file specifies its namespace at the top.
