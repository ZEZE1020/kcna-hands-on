# Final Challenge — Deploy a Production-Grade Cloud-Native App

> **Time:** 3–4 hours  
> **This challenge has no walkthrough.** Use everything from modules 01–07.

---

## Important Prerequisites

Before starting, ensure your cluster has the following **required components**:

1. **Ingress Controller** — An Ingress resource without a controller won't route traffic. Common options: NGINX Ingress Controller, Istio Ingress Gateway. Install one before attempting the Ingress requirement.

2. **NetworkPolicy-capable CNI** — NetworkPolicy resources may apply cleanly but won't actually block traffic unless your CNI enforces them. Verify with `kubectl get networkpolicy` after creation; it should show your policies. CNIs like Calico, Cilium, or Weave support NetworkPolicy.

3. **Metrics Server** — Required for `kubectl top` and HPA to work. Install it before the observability and autoscaling sections.

---

## Scenario

You've joined a team that needs to deploy **ShopCore** — a small e-commerce backend — to Kubernetes for the first time. You have the code and requirements. The rest is on you.

---

## What You Need to Deploy

ShopCore consists of three services:

| Service | Image | Port | Notes |
|---------|-------|------|-------|
| `frontend` | `nginxdemos/hello` | 80 | Serves static HTML |
| `api` | `kennethreitz/httpbin` | 80 | REST API mock |
| `redis` | `redis:7-alpine` | 6379 | Session store |

---

## Requirements

Complete all of the following. They're ordered loosely by module — but in this challenge, nothing tells you which module a requirement maps to.

### 1. Namespace and RBAC
- [ ] Deploy everything in a namespace called `shopcore`
- [ ] Create a `ServiceAccount` called `shopcore-sa` for the api pods
- [ ] Create a `Role` that allows the SA to `get`, `list` pods and services in `shopcore`
- [ ] Bind the role to the service account

### 2. Workloads
- [ ] Deploy `frontend` as a **Deployment** with 3 replicas
- [ ] Deploy `api` as a **Deployment** with 2 replicas, using the `shopcore-sa` service account
- [ ] Deploy `redis` as a **StatefulSet** with 1 replica
- [ ] All containers must have `requests` and `limits` set for both CPU and memory
- [ ] All containers must NOT run as root (`runAsNonRoot: true`)

### 3. Configuration
- [ ] Create a `ConfigMap` called `shopcore-config` with keys:
  - `ENVIRONMENT=production`
  - `LOG_LEVEL=info`
  - `MAX_CONNECTIONS=100`
- [ ] Mount these as environment variables in the `api` pods
- [ ] Create a `Secret` called `shopcore-secrets` with a key `REDIS_PASSWORD=supersecret`
- [ ] Mount it as an env var in the `api` pods

### 4. Health Probes
- [ ] `frontend` must have a `readinessProbe` on `GET /`
- [ ] `api` must have both a `livenessProbe` and `readinessProbe` on `GET /get`
- [ ] `redis` must have a `livenessProbe` using exec: `redis-cli ping`

### 5. Services and Networking
- [ ] `frontend` exposed as a `ClusterIP` Service on port 80
- [ ] `api` exposed as a `ClusterIP` Service on port 80
- [ ] `redis` exposed as a **Headless** Service on port 6379
- [ ] An `Ingress` that routes:
  - `/` → frontend
  - `/api/` → api

### 6. NetworkPolicy
- [ ] `frontend` can talk to `api` but NOT to `redis`
- [ ] `api` can talk to `redis`
- [ ] No external traffic should reach `redis` directly
- [ ] Deny all ingress by default, then allow explicitly

### 7. Autoscaling
- [ ] Create an `HorizontalPodAutoscaler` for `api`:
  - Min: 2, Max: 8
  - Scale up when CPU > 70%

### 8. Helm
- [ ] Package everything as a Helm chart named `shopcore`
- [ ] Support separate values files for `dev` and `prod`:
  - Dev: 1 replica, resource limits half of prod
  - Prod: original specs
- [ ] `helm install shopcore-prod ./shopcore -f values-prod.yaml -n shopcore` must work cleanly
  - **Cleanup note:** If rerunning, delete the namespace and release first: `helm uninstall shopcore-prod -n shopcore && kubectl delete ns shopcore`

### 9. Observability
- [ ] All pods must have the following labels: `app`, `version`, `tier`
- [ ] All pods must have annotations: `prometheus.io/scrape: "true"`, `prometheus.io/port`
- [ ] `kubectl top pods -n shopcore` must return real data (Metrics Server must be installed)
- [ ] Write a single PromQL query that shows the request rate for the `api` service (add to a file `queries.promql`)

### 10. GitOps (Bonus — Optional)

The core challenge is complete after requirement #9. The following is an **optional stretch task**:

- [ ] Create an ArgoCD `Application` manifest (YAML, not just CLI) that deploys the Helm chart
- [ ] The Application should be configured for **automated sync with self-heal and prune**

---

## Verification Script

Run this to check your work:

```bash
#!/bin/bash
set -e
NS=shopcore
PASS=0
FAIL=0

check() {
  if eval "$2" &>/dev/null; then
    echo "✅ $1"
    PASS=$((PASS+1))
  else
    echo "❌ $1"
    FAIL=$((FAIL+1))
  fi
}

echo "=== ShopCore Verification ==="

check "Namespace exists" "kubectl get ns $NS"
check "ServiceAccount shopcore-sa" "kubectl get sa shopcore-sa -n $NS"
check "Frontend deployment (3 replicas)" "kubectl get deploy frontend -n $NS -o jsonpath='{.spec.replicas}' | grep 3"
check "API deployment running" "kubectl get deploy api -n $NS"
check "Redis StatefulSet" "kubectl get sts redis -n $NS"
check "ConfigMap shopcore-config" "kubectl get cm shopcore-config -n $NS"
check "Secret shopcore-secrets" "kubectl get secret shopcore-secrets -n $NS"
check "Frontend Service" "kubectl get svc frontend -n $NS"
check "API Service" "kubectl get svc api -n $NS"
check "Redis Headless Service (no clusterIP)" "kubectl get svc redis -n $NS -o jsonpath='{.spec.clusterIP}' | grep None"
check "Ingress exists" "kubectl get ingress -n $NS"
check "HPA for api" "kubectl get hpa api -n $NS"
check "NetworkPolicy exists" "kubectl get networkpolicy -n $NS"
check "All pods Running" "kubectl get pods -n $NS --field-selector=status.phase!=Running 2>&1 | grep 'No resources found'"

echo ""
echo "Results: $PASS passed, $FAIL failed"
if [ $FAIL -eq 0 ]; then
  echo "🎉 All checks passed — you're ready for the KCNA!"
fi
```

Save as `verify.sh`, run with `bash verify.sh`.

---

## Hints

<details>
<summary>I'm stuck on NetworkPolicy</summary>

Start with a default-deny-all ingress policy for the namespace, then layer in allow rules:
```yaml
# 1. Default deny all
spec:
  podSelector: {}
  policyTypes: [Ingress]

# 2. Allow frontend → api
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
```

</details>

<details>
<summary>Redis StatefulSet headless service</summary>

A headless service has `clusterIP: None`. The StatefulSet must reference it in `spec.serviceName`.

</details>

<details>
<summary>HPA requires Metrics Server</summary>

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
# For kind, patch it to skip TLS verification:
kubectl patch deployment metrics-server -n kube-system \
  --type json \
  -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

</details>

---

## What the KCNA Tests (from this challenge)

After completing this, you've hands-on experience with every major KCNA topic:

- Container orchestration and workload types
- Kubernetes architecture and API
- Configuration (ConfigMap, Secret, env vars)
- Storage (StatefulSet + headless service)
- Networking (Services, Ingress, NetworkPolicy)
- Security (RBAC, non-root containers, Secrets)
- Autoscaling (HPA)
- Observability (Prometheus annotations, PromQL)
- Application delivery (Helm, GitOps)
- Cloud native patterns (12-factor, microservices)

---

## After the Challenge

- Review the [KCNA Exam Curriculum](https://github.com/cncf/curriculum/blob/master/KCNA_Curriculum.pdf)
- Take one practice exam to identify remaining gaps
- You don't need to memorise tool lists — if you did these labs you'll *know* what each tool does

**Good luck!** 🎉
