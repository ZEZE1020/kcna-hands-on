# Module 03 · Lab 01 — Pods Deep Dive

> **KCNA Domain:** Kubernetes Fundamentals (46%)  
> **Time:** ~50 minutes

---

## Part 1 — Pod Anatomy

A Pod is the smallest deployable unit in Kubernetes — one or more containers that share:
- Network namespace (same IP, same `localhost`)
- Storage volumes
- Lifecycle

### 1.1 Create a multi-container pod

```bash
kubectl create namespace module-03

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
  namespace: module-03
  labels:
    app: demo
    tier: web
spec:
  initContainers:
  - name: init-db-check
    image: busybox
    command: ["sh", "-c", "echo 'Checking DB connectivity...'; sleep 3; echo 'DB ready'"]

  containers:
  - name: app
    image: nginx:1.25
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html

  - name: content-updater
    image: busybox
    command: ["sh", "-c", "while true; do echo \"<h1>Updated at $(date)</h1>\" > /html/index.html; sleep 10; done"]
    volumeMounts:
    - name: shared-data
      mountPath: /html

  volumes:
  - name: shared-data
    emptyDir: {}
EOF

kubectl wait --for=condition=Ready pod/multi-container -n module-03 --timeout=60s
```

### 1.2 Observe the shared volume

```bash
# Content written by content-updater is served by nginx
kubectl port-forward pod/multi-container 8080:80 -n module-03 &
curl localhost:8080
# See "Updated at <time>" — content-updater wrote it, nginx serves it
```

### 1.3 Explore init containers

```bash
kubectl describe pod multi-container -n module-03 | grep -A10 "Init Containers"
# Init container ran first, completed, then app containers started
```

**KCNA concept:** Init containers are used for:
- Database migration checks
- Config download before main app starts
- Secrets population
- Waiting for dependencies

---

## Part 2 — Resource Management

### 2.1 Requests vs Limits

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
  namespace: module-03
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:          # Guaranteed resources (scheduler uses this)
        memory: "64Mi"
        cpu: "250m"
      limits:            # Maximum allowed (container killed if exceeded)
        memory: "128Mi"
        cpu: "500m"
EOF
```

| | Requests | Limits |
|-|----------|--------|
| **Used by** | Scheduler (decides which node) | Kubelet (enforces at runtime) |
| **If exceeded** | Nothing — it's just a hint | CPU throttled; Memory → OOMKilled |
| **Best practice** | Always set | Always set |

### 2.2 QoS classes

Kubernetes assigns Quality of Service based on requests/limits:

```bash
# BestEffort: no requests or limits
cat <<EOF | kubectl apply -f - 
apiVersion: v1
kind: Pod
metadata:
  name: besteffort
  namespace: module-03
spec:
  containers:
  - name: app
    image: nginx
EOF

# Guaranteed: requests == limits for all containers
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed
  namespace: module-03
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF

kubectl get pod besteffort guaranteed -n module-03 -o custom-columns="NAME:.metadata.name,QOS:.status.qosClass"
```

**Eviction order under pressure:** BestEffort → Burstable → Guaranteed

---

## Part 3 — ConfigMaps and Secrets

### 3.1 ConfigMap as environment variables

```bash
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=debug \
  --from-literal=APP_PORT=8080 \
  --from-literal=MAX_RETRIES=3 \
  -n module-03

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: config-demo
  namespace: module-03
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "env | grep -E 'LOG|APP|MAX'; sleep 3600"]
    envFrom:
    - configMapRef:
        name: app-config
EOF

kubectl logs config-demo -n module-03
```

### 3.2 ConfigMap as a mounted file

```bash
kubectl create configmap nginx-config \
  --from-literal=nginx.conf="server { listen 8080; location /health { return 200 'ok'; } }" \
  -n module-03

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: file-config-demo
  namespace: module-03
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "cat /etc/config/nginx.conf; sleep 3600"]
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
  volumes:
  - name: config-vol
    configMap:
      name: nginx-config
EOF

kubectl logs file-config-demo -n module-03
```

### 3.3 Secrets

```bash
# Secrets are base64-encoded (not encrypted by default)
kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=s3cret123 \
  --from-literal=DB_USER=admin \
  -n module-03

# See the base64 encoding
kubectl get secret db-secret -n module-03 -o yaml

# Decode it
kubectl get secret db-secret -n module-03 -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

**KCNA security note:** Base64 is encoding, not encryption. For real secret management, use:
- Sealed Secrets (encrypt before storing in Git)
- External Secrets Operator (sync from Vault, AWS Secrets Manager, etc.)
- HashiCorp Vault Agent Injector

---

## Part 4 — Pod Scheduling Controls

### 4.1 Node Selector

```bash
# Label a node
kubectl label node kcna-arch-worker region=eu-west

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: eu-pod
  namespace: module-03
spec:
  nodeSelector:
    region: eu-west
  containers:
  - name: app
    image: nginx
EOF

kubectl get pod eu-pod -n module-03 -o wide
# Should be on kcna-arch-worker
```

### 4.2 Tolerations and Taints

```bash
# Taint a node — no pods will schedule here unless they tolerate it
kubectl taint node kcna-arch-worker2 special=true:NoSchedule

# This pod has NO toleration — will NOT schedule on worker2
kubectl run no-toleration --image=nginx -n module-03
kubectl get pod no-toleration -n module-03 -o wide   # On worker1

# This pod tolerates the taint — CAN schedule on worker2
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
  namespace: module-03
spec:
  tolerations:
  - key: "special"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  containers:
  - name: app
    image: nginx
EOF

# Remove the taint when done
kubectl taint node kcna-arch-worker2 special=true:NoSchedule-
```

**Common use case:** Control plane nodes have `node-role.kubernetes.io/control-plane:NoSchedule` by default — that's why user pods don't run on them.

---

## Cleanup

```bash
kubectl delete namespace module-03
```

## ➡️ Next: [Lab 02 — Deployments and Rollouts](lab-02-deployments-and-rollouts.md)
