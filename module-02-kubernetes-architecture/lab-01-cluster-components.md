# Module 02 · Lab 01 — Kubernetes Architecture From the Inside

> **KCNA Domain:** Kubernetes Fundamentals (46%)  
> **Time:** ~60 minutes  
> **Cluster:** kind multi-node (see setup/environment.md)

---

## What You'll Explore

This lab makes the Kubernetes control plane *tangible*. Rather than reading about what `etcd` does, you'll watch it store data. Rather than reading about the scheduler, you'll create an unschedulable pod and watch it get stuck — then fix it.

---

## Cluster Setup

```bash
cat <<EOF | kind create cluster --name kcna-arch --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF

kubectl get nodes
```

---

## Part 1 — The Control Plane Components

### 1.1 See the control plane pods

```bash
kubectl get pods -n kube-system
```

You'll see:
- `kube-apiserver` — the front door; everything goes through here
- `etcd` — the database; the *only* stateful component
- `kube-scheduler` — decides which node a pod runs on
- `kube-controller-manager` — runs control loops (ReplicaSet, Node, Job controllers, etc.)

### 1.2 Watch the API server process a request

Open two terminals.

**Terminal 1** — watch API server audit logs:
```bash
kubectl get pods -n kube-system -w
```

**Terminal 2** — create something:
```bash
kubectl create namespace lab-02
kubectl run nginx --image=nginx -n lab-02
```

Watch Terminal 1 react in real time.

### 1.3 Understand etcd as the source of truth

Everything in Kubernetes is stored in etcd as key-value pairs. The API server is the *only* component that talks to etcd directly.

```bash
# Get into the etcd pod
kubectl exec -it -n kube-system etcd-kcna-arch-control-plane -- sh

# Inside the etcd pod — list all keys (use etcdctl)
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get / --prefix --keys-only | head -30
```

You'll see paths like `/registry/pods/lab-02/nginx`. This is your nginx pod, stored as a protobuf blob.

```bash
# Read your nginx pod from etcd directly
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/pods/lab-02/nginx | strings | grep -E "nginx|image|status"
```

**KCNA key insight:** If etcd goes down, the cluster can't write new state — but *existing running workloads keep running* because kubelet doesn't need etcd to manage local containers.

---

## Part 2 — The Scheduler

### 2.1 Watch a pod get scheduled

```bash
kubectl run watcher --image=nginx -n lab-02
kubectl get events -n lab-02 --sort-by='.lastTimestamp'
```

Look for `Scheduled` events — they show which node the scheduler chose and why.

### 2.2 Create an Unschedulable Pod

Label nodes:
```bash
kubectl get nodes --show-labels
kubectl label node kcna-arch-worker gpu=true
```

Create a pod that requires a label that doesn't exist on any node:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: needs-special-gpu
  namespace: lab-02
spec:
  containers:
  - name: app
    image: nginx
  nodeSelector:
    gpu: "false"    # no node has gpu=false
EOF
```

```bash
kubectl get pod needs-special-gpu -n lab-02   # Status: Pending
kubectl describe pod needs-special-gpu -n lab-02 | grep -A5 Events
```

You'll see: `0/3 nodes are available: 1 node has unacceptable pod spec, 2 didn't match node selector`

Fix it:
```bash
kubectl patch pod needs-special-gpu -n lab-02 -p '{"spec":{"nodeSelector":{"gpu":"true"}}}'
# Note: you can't patch a running pod's spec — delete and recreate
kubectl delete pod needs-special-gpu -n lab-02
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: needs-special-gpu
  namespace: lab-02
spec:
  containers:
  - name: app
    image: nginx
  nodeSelector:
    gpu: "true"
EOF
kubectl get pod needs-special-gpu -n lab-02  # Now Scheduled
```

---

## Part 3 — The Controller Manager

### 3.1 Watch the ReplicaSet controller react

```bash
kubectl create deployment demo --image=nginx --replicas=3 -n lab-02
kubectl get pods -n lab-02
```

Now delete a pod manually:
```bash
POD=$(kubectl get pods -n lab-02 -l app=demo -o name | head -1)
kubectl delete $POD -n lab-02
```

Immediately watch:
```bash
kubectl get pods -n lab-02 -w
```

The controller manager detects the pod count dropped below desired (3) and creates a replacement within seconds. This is a **reconciliation loop** — the core concept behind all Kubernetes controllers.

### 3.2 The reconciliation pattern

```
Desired State (stored in etcd)  ←──────────┐
        ↓                                   │
  Controller observes                       │
  actual state                              │
        ↓                                   │
  If actual ≠ desired →  take action  ──────┘
```

This is also why Kubernetes is *declarative*, not imperative — you say what you want, not how to get there.

---

## Part 4 — kubelet and kube-proxy (Worker Node Components)

### 4.1 kubelet

The kubelet runs on every node. It's responsible for:
- Pulling images
- Starting/stopping containers via the container runtime
- Reporting node and pod status back to the API server
- Running pod liveness/readiness probes

```bash
# SSH into a worker node (kind-specific)
docker exec -it kcna-arch-worker bash

# Inside the worker node
systemctl status kubelet
cat /var/lib/kubelet/config.yaml | head -20
exit
```

### 4.2 kube-proxy

kube-proxy runs on every node and implements `Service` routing by managing iptables/IPVS rules.

```bash
kubectl get pods -n kube-system | grep kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy | tail -20
```

---

## Part 5 — The API Server is Everything

Every `kubectl` command is an API call. You can see this:

```bash
kubectl get pods -n lab-02 -v=8 2>&1 | grep -E "GET|POST|PATCH" | head -10
```

The `-v=8` flag shows the raw HTTP calls kubectl is making. Everything goes through `https://<apiserver>/api/v1/...`

---

## Cleanup

```bash
kubectl delete namespace lab-02
# Keep the cluster for Module 03
```

---

## Key Concepts Covered

| Component | Role | Where it runs |
|-----------|------|---------------|
| kube-apiserver | REST API, auth, admission | Control plane |
| etcd | Key-value store, source of truth | Control plane |
| kube-scheduler | Pod-to-node assignment | Control plane |
| kube-controller-manager | Reconciliation loops | Control plane |
| kubelet | Node agent, container lifecycle | Every node |
| kube-proxy | Service networking (iptables/IPVS) | Every node |

---

## ➡️ Next: [Lab 02 — API Server, kubectl, and RBAC](lab-02-api-server-and-kubectl.md)
