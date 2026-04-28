# Module 04 · Lab 01 — Services, DNS, and Network Policy

> **KCNA Domain:** Kubernetes Fundamentals (46%)  
> **Time:** ~75 minutes  
> **Cluster:** kind multi-node

---

## What You'll Build

A multi-tier application (frontend → backend → database) and connect the layers using Kubernetes Services, internal DNS, and NetworkPolicy. You'll see *why* Services exist and what breaks without them.

---

## Part 1 — Why Services Exist

### 1.1 The problem: pod IPs are ephemeral

```bash
kubectl create namespace module-04

# Deploy a simple backend
kubectl create deployment backend --image=nginx --replicas=3 -n module-04
kubectl get pods -n module-04 -o wide
```

Note the pod IPs. Now delete a pod:
```bash
POD=$(kubectl get pods -n module-04 -l app=backend -o name | head -1)
kubectl delete $POD -n module-04
kubectl get pods -n module-04 -o wide
```

The replacement pod has a **different IP**. If your frontend hardcoded the old IP, it would break. This is why Services exist — they provide a stable virtual IP and DNS name regardless of which pods are behind them.

---

## Part 2 — ClusterIP Service

### 2.1 Expose the backend

```bash
kubectl expose deployment backend --port=80 --target-port=80 -n module-04
kubectl get service backend -n module-04
```

Note the `CLUSTER-IP`. This IP is virtual — it's an iptables rule, not a real interface.

### 2.2 Test service routing

```bash
# Run a temporary pod to test from inside the cluster
kubectl run test-client --image=busybox --rm -it -n module-04 -- sh

# Inside the pod:
wget -O- http://backend        # Uses DNS!
wget -O- http://backend.module-04.svc.cluster.local  # Fully qualified
wget -O- http://<CLUSTER-IP>   # Direct IP also works
exit
```

### 2.3 How DNS works

```bash
kubectl get pods -n kube-system | grep coredns
```

CoreDNS is the cluster DNS. Every Service gets a DNS record:
```
<service-name>.<namespace>.svc.cluster.local
```

Short names (`backend`) work inside the same namespace. Cross-namespace requires the full name.

---

## Part 3 — NodePort Service

ClusterIP is only reachable inside the cluster. NodePort exposes a service on every node's IP at a static port.

```bash
kubectl expose deployment backend --type=NodePort --port=80 --name=backend-ext -n module-04
kubectl get service backend-ext -n module-04
```

Note the `NodePort` (30000-32767 range). With kind:
```bash
# Get the node IP
kubectl get nodes -o wide
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[1].status.addresses[0].address}')
NODE_PORT=$(kubectl get svc backend-ext -n module-04 -o jsonpath='{.spec.ports[0].nodePort}')
curl http://$NODE_IP:$NODE_PORT
```

**KCNA note:** NodePort is rarely used in production — it's more of a building block. LoadBalancer Services (which provision a cloud load balancer) are more common.

---

## Part 4 — Ingress

Ingress routes HTTP/S traffic to Services based on hostname and path rules — like a reverse proxy.

### 4.1 Install nginx ingress controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

### 4.2 Deploy two apps

```bash
# App v1
kubectl create deployment app-v1 --image=nginxdemos/hello --replicas=2 -n module-04
kubectl expose deployment app-v1 --port=80 -n module-04

# App v2
kubectl create deployment app-v2 --image=nginxdemos/hello --replicas=2 -n module-04
kubectl expose deployment app-v2 --port=80 -n module-04
```

### 4.3 Create an Ingress

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: apps-ingress
  namespace: module-04
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: app.local
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: app-v1
            port:
              number: 80
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: app-v2
            port:
              number: 80
EOF

# Test (add app.local to /etc/hosts pointing to 127.0.0.1)
echo "127.0.0.1 app.local" | sudo tee -a /etc/hosts
curl http://app.local/v1
curl http://app.local/v2
```

---

## Part 5 — NetworkPolicy

By default, all pods can talk to all other pods. NetworkPolicy lets you enforce "zero trust" between namespaces and pods.

### 5.1 Set up the scenario

```bash
kubectl create namespace frontend
kubectl create namespace backend-ns
kubectl create namespace monitoring

kubectl run web --image=nginx -n frontend --labels=tier=web
kubectl run api --image=nginx -n backend-ns --labels=tier=api
kubectl run monitor --image=nginx -n monitoring --labels=tier=monitor

# Expose the API
kubectl expose pod api --port=80 -n backend-ns
```

### 5.2 Verify open traffic (before policy)

```bash
kubectl exec -n frontend web -- wget -O- http://api.backend-ns.svc.cluster.local --timeout=3
# This works — no NetworkPolicy yet
```

### 5.3 Lock down the backend

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: backend-ns
spec:
  podSelector:
    matchLabels:
      tier: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: frontend
      podSelector:
        matchLabels:
          tier: web
    ports:
    - protocol: TCP
      port: 80
EOF
```

Label the frontend namespace (required for namespaceSelector):
```bash
kubectl label namespace frontend kubernetes.io/metadata.name=frontend
```

### 5.4 Test the policy

```bash
# Frontend web → backend API: ALLOWED
kubectl exec -n frontend web -- wget -O- http://api.backend-ns.svc.cluster.local --timeout=3

# Monitoring → backend API: BLOCKED
kubectl exec -n monitoring monitor -- wget -O- http://api.backend-ns.svc.cluster.local --timeout=3
# Timeout! Connection blocked.
```

**KCNA concept:** NetworkPolicy is implemented by the CNI plugin (Calico, Cilium, etc.), NOT by kube-proxy. Not all CNI plugins support NetworkPolicy — `kindnet` (kind's default) does NOT. For this lab, you may need to test with Calico or use Killercoda which has Calico pre-installed.

---

## Part 6 — Service Types Summary

```bash
cat <<EOF | kubectl apply -f -
# Headless service — no ClusterIP, DNS returns pod IPs directly
# Used by StatefulSets for peer-to-peer discovery
apiVersion: v1
kind: Service
metadata:
  name: headless-svc
  namespace: module-04
spec:
  clusterIP: None
  selector:
    app: backend
  ports:
  - port: 80
EOF

kubectl run dns-test --image=busybox --rm -it -n module-04 -- nslookup headless-svc
# Returns multiple A records — one per pod IP
```

| Service Type | Reachable From | Use Case |
|-------------|----------------|----------|
| ClusterIP | Inside cluster only | Internal service communication |
| NodePort | Outside cluster via node IP | Dev/test, on-prem |
| LoadBalancer | Outside cluster via LB IP | Cloud production |
| ExternalName | Inside cluster → external DNS | Integrating external services |
| Headless (clusterIP: None) | Inside cluster, per-pod | StatefulSets, peer discovery |

---

## Cleanup

```bash
kubectl delete namespace module-04 frontend backend-ns monitoring
```

## ➡️ Next: [Module 04 Challenge](challenge.md)
