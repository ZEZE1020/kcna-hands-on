# Module 05 · Lab — Cloud Native Architecture

> **KCNA Domain:** Cloud Native Architecture (16%)  
> **Time:** ~90 minutes  
> **Tools:** Linkerd, custom CRDs

---

## What You'll Build

A microservices-based app with a service mesh, and your first Custom Resource Definition. This module covers the CNCF landscape concepts that appear heavily in KCNA: 12-factor apps, service meshes, serverless, CRDs, and operators.

---

## Part 1 — Microservices Decomposition

### 1.1 The 12-Factor App

The KCNA exam tests awareness of the 12-Factor App methodology. Deploy an app that demonstrates the key factors:

```bash
kubectl create namespace module-05

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: twelve-factor-demo
  namespace: module-05
spec:
  replicas: 3
  selector:
    matchLabels:
      app: twelve-factor
  template:
    metadata:
      labels:
        app: twelve-factor
    spec:
      containers:
      - name: app
        image: nginx:1.25
        # Factor III: Config in environment, not code
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: db-url
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: log-level
        # Factor XI: Treat logs as event streams (stdout)
        # Factor IX: Disposability — fast startup, graceful shutdown
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"]
        # Factor VIII: Concurrency — scale out via replicas
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: module-05
data:
  log-level: "info"
  max-connections: "100"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: module-05
stringData:
  db-url: "postgresql://db:5432/myapp"
EOF
```

**KCNA key factors to know:**
1. Codebase — one codebase per service, tracked in version control
2. Dependencies — explicitly declared (Dockerfile, package.json)
3. Config — in environment variables, not code
4. Backing services — databases, queues treated as attached resources
5. Build/Release/Run — strictly separate stages
6. Processes — stateless, share-nothing
7. Port binding — self-contained HTTP server
8. Concurrency — scale via process model (horizontal)
9. Disposability — fast startup, graceful shutdown
10. Dev/prod parity — same tools, same dependencies everywhere
11. Logs — event streams to stdout
12. Admin processes — one-off tasks as one-time processes

---

## Part 2 — Service Mesh with Linkerd

### 2.1 Why a service mesh?

Without a service mesh:
- Each app must implement mTLS itself
- Retry logic lives in every service
- You can't see traffic between services

A service mesh (Linkerd, Istio, Consul) handles these at the **infrastructure layer** via sidecar proxies.

```text
Without mesh:    Frontend → Backend
With mesh:       Frontend [proxy] → [proxy] Backend
                                ↑ transparent, automatic
```

### 2.2 Install Linkerd

```bash
# Install Linkerd CLI
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
export PATH=$PATH:$HOME/.linkerd2/bin
linkerd version

# Validate cluster compatibility
linkerd check --pre

# Install Linkerd control plane
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -
linkerd check   # Wait for all checks to pass
```

### 2.3 Deploy a sample app and mesh it

```bash
# Deploy the official Linkerd demo app
curl -fsL https://run.linkerd.io/emojivoto.yml | kubectl apply -f -

# Inject Linkerd sidecars into the namespace
kubectl get -n emojivoto deploy -o yaml \
  | linkerd inject - \
  | kubectl apply -f -

# Verify injection
linkerd -n emojivoto check --proxy
kubectl get pods -n emojivoto
# Each pod now has 2 containers: the app + the linkerd-proxy sidecar
```

### 2.4 Observe traffic

```bash
# Install Linkerd Viz extension
linkerd viz install | kubectl apply -f -
linkerd viz check

# Open the dashboard
linkerd viz dashboard &
```

In the dashboard you'll see:
- **Golden metrics** (success rate, RPS, latency) per service — automatically!
- Live traffic graph between services
- HTTP request/response metadata

No code changes were needed. The proxy handles it all.

### 2.5 mTLS verification

```bash
# Every service-to-service call is now encrypted
linkerd -n emojivoto viz edges deployment
# "SECURED" in the mTLS column means mutual TLS is active
```

---

## Part 3 — Custom Resource Definitions (CRDs)

CRDs let you extend the Kubernetes API with your own object types. They're foundational to operators.

### 3.1 Create a CRD

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databaseclusters.kcna.labs
spec:
  group: kcna.labs
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              engine:
                type: string
                enum: [postgres, mysql, redis]
              replicas:
                type: integer
                minimum: 1
                maximum: 5
              storageGB:
                type: integer
            required: [engine, replicas]
  scope: Namespaced
  names:
    plural: databaseclusters
    singular: databasecluster
    kind: DatabaseCluster
    shortNames: [dbc]
EOF

kubectl get crd databaseclusters.kcna.labs
```

### 3.2 Create instances of your custom resource

```bash
cat <<EOF | kubectl apply -f -
apiVersion: kcna.labs/v1
kind: DatabaseCluster
metadata:
  name: production-db
  namespace: module-05
spec:
  engine: postgres
  replicas: 3
  storageGB: 100
---
apiVersion: kcna.labs/v1
kind: DatabaseCluster
metadata:
  name: dev-db
  namespace: module-05
spec:
  engine: redis
  replicas: 1
  storageGB: 10
EOF

kubectl get dbc -n module-05
kubectl describe dbc production-db -n module-05
```

Nothing happens yet — there's no controller watching these resources. That's what an **operator** adds.

### 3.3 What is an Operator?

An operator = CRD + a controller that acts on instances of that CRD.

```text
User creates DatabaseCluster CR (spec: postgres, 3 replicas)
         ↓
Operator's controller detects new DatabaseCluster
         ↓
Operator creates: StatefulSet + Service + PVC + ConfigMap
         ↓
Operator monitors: handles scaling, backup, upgrades
```

Real-world examples from CNCF ecosystem:
- **Prometheus Operator** — manages Prometheus instances as `PrometheusRule`, `ServiceMonitor` CRDs
- **ArgoCD** — manages `Application` CRDs
- **cert-manager** — manages `Certificate`, `Issuer` CRDs
- **Strimzi** — manages Kafka clusters as CRDs

---

## Part 4 — Serverless on Kubernetes

### 4.1 Knative (awareness)

The KCNA tests awareness of serverless platforms in the CNCF ecosystem.

```bash
# Knative Serving allows:
# - Scale to zero (pods spin up only when traffic arrives)
# - Automatic scaling based on concurrent requests
# - Traffic splitting for canary deployments

# Install Knative CRDs (installs even without the serving backend)
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.13.0/serving-crds.yaml
kubectl get crd | grep knative
```

**KCNA key distinction:**

| Approach | Scale to zero? | Cold start? | Use case |
|----------|---------------|-------------|----------|
| Deployment | No | No | Long-running services |
| Knative Serving | Yes | Yes (~1-2s) | Event-driven, spiky workloads |
| Knative Eventing | Yes | Yes | Event-driven microservices |

---

## Part 5 — CNCF Landscape Awareness

The KCNA exam tests broad awareness of tools. Know the *category* each tool falls into:

### Scheduling & Orchestration
- Kubernetes — container orchestration (graduated)
- Nomad — polyglot scheduler (not CNCF)

### Service Mesh
- Linkerd — lightweight (graduated)
- Istio — feature-rich (graduated)
- Consul Connect — HashiCorp

### Serverless
- Knative — Kubernetes-native serverless (incubating)
- OpenFaaS — function-as-a-service on Kubernetes

### Storage
- Rook/Ceph — cloud-native storage (graduated)
- Longhorn — distributed block storage (incubating)
- CSI — Container Storage Interface standard

### Security
- Falco — runtime security (graduated)
- OPA/Gatekeeper — policy engine (graduated)
- cert-manager — certificate management (incubating)

---

## Cleanup

```bash
kubectl delete namespace module-05 emojivoto
linkerd uninstall | kubectl delete -f -
kubectl delete crd databaseclusters.kcna.labs
```

## ➡️ Next: [Module 06 — Observability](../module-06-observability/lab-01-logging-metrics-tracing.md)
