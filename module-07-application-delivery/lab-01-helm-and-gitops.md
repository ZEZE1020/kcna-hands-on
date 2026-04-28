# Module 07 · Lab 01 — Helm and GitOps with ArgoCD

> **KCNA Domain:** Cloud Native Application Delivery (8%)  
> **Time:** ~75 minutes  
> **Tools:** Helm 3, ArgoCD

---

## What You'll Build

First you'll package an application with Helm and understand why Helm exists. Then you'll deploy ArgoCD and experience GitOps — where your Git repo becomes the source of truth for cluster state.

---

## Part 1 — Why Helm Exists

### 1.1 The problem with raw YAML

Deploy an app manually:

```bash
kubectl create namespace module-07

# You need YAML for: Deployment, Service, Ingress, ConfigMap, Secret, HPA, PDB...
# And if you need to deploy it to dev, staging, prod with different values in each...
# You'd copy-paste and modify the same YAML files dozens of times
```

Helm solves this with **templates + values**.

### 1.2 Install Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

---

## Part 2 — Using Helm Charts

### 2.1 Add a Helm repo and install a chart

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# See what's configurable before installing
helm show values bitnami/nginx | head -50

# Install with custom values
helm install my-nginx bitnami/nginx \
  --namespace module-07 \
  --set replicaCount=2 \
  --set service.type=ClusterIP

helm list -n module-07
kubectl get all -n module-07
```

### 2.2 Upgrade and rollback

```bash
# Upgrade: scale to 3 replicas
helm upgrade my-nginx bitnami/nginx \
  --namespace module-07 \
  --set replicaCount=3 \
  --set service.type=ClusterIP

helm history my-nginx -n module-07

# Rollback to revision 1
helm rollback my-nginx 1 -n module-07
helm history my-nginx -n module-07
```

---

## Part 3 — Create Your Own Helm Chart

### 3.1 Scaffold a chart

```bash
mkdir -p ~/kcna-labs/module-07 && cd ~/kcna-labs/module-07
helm create webapp
ls webapp/
```

Helm creates:
- `Chart.yaml` — chart metadata
- `values.yaml` — default values
- `templates/` — Kubernetes manifests with Go template syntax

### 3.2 Understand the template syntax

Look at `webapp/templates/deployment.yaml`:
```bash
cat webapp/templates/deployment.yaml | head -30
```

You'll see things like:
```yaml
replicas: {{ .Values.replicaCount }}
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

### 3.3 Customise for dev and prod

Create `values-dev.yaml`:
```yaml
replicaCount: 1
image:
  repository: nginx
  tag: "1.24"
resources:
  limits:
    memory: "128Mi"
    cpu: "200m"
```

Create `values-prod.yaml`:
```yaml
replicaCount: 5
image:
  repository: nginx
  tag: "1.25"
resources:
  limits:
    memory: "512Mi"
    cpu: "1000m"
```

### 3.4 Dry run and diff

```bash
# Preview what Helm would deploy
helm template webapp ./webapp -f values-dev.yaml

# Install
helm install webapp-dev ./webapp -f values-dev.yaml -n module-07
helm install webapp-prod ./webapp -f values-prod.yaml -n module-07 --create-namespace

kubectl get pods -n module-07
```

### 3.5 Add a conditional resource

Edit `webapp/templates/hpa.yaml`:
```yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "webapp.fullname" . }}
  namespace: {{ .Release.Namespace }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "webapp.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
{{- end }}
```

In `values.yaml`:
```yaml
autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
```

Enable it for prod:
```bash
helm upgrade webapp-prod ./webapp \
  -f values-prod.yaml \
  --set autoscaling.enabled=true \
  -n module-07
```

---

## Part 4 — GitOps with ArgoCD

### 4.1 What is GitOps?

GitOps is an operating model where:
- Git is the **single source of truth** for desired state
- A **reconciliation loop** (ArgoCD, Flux) continuously syncs cluster state with Git
- All changes go through Git (PRs, not `kubectl apply`)

```
Developer pushes to Git
        ↓
ArgoCD detects diff between Git and cluster
        ↓
ArgoCD applies the diff automatically
        ↓
Cluster matches Git
```

### 4.2 Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=120s

# Get admin password
ARGOCD_PASS=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
echo "ArgoCD password: $ARGOCD_PASS"

# Port-forward UI
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
```

Open https://localhost:8080 — login with `admin` / the password above.

### 4.3 Deploy an app with ArgoCD

You'll use a public Git repo for this (or fork it):

```bash
# Install ArgoCD CLI
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/

argocd login localhost:8080 --username admin --password $ARGOCD_PASS --insecure

# Create an ArgoCD Application pointing to a public Helm chart repo
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path helm-guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace module-07 \
  --sync-policy automated \
  --auto-prune \
  --self-heal

argocd app get guestbook
argocd app sync guestbook
```

### 4.4 See GitOps in action

Try to manually change something the app manages:
```bash
kubectl scale deployment guestbook-helm-guestbook -n module-07 --replicas=1
```

Wait ~30 seconds:
```bash
kubectl get deployment guestbook-helm-guestbook -n module-07
# ArgoCD reverts it back to the desired state from Git
```

This is **self-healing** — the core value of GitOps.

### 4.5 ArgoCD Sync Strategies

In the ArgoCD UI, explore:
- **OutOfSync** — cluster doesn't match Git
- **Synced** — cluster matches Git
- **Degraded** — app is unhealthy
- **Auto-sync** vs **Manual sync**
- **Pruning** — ArgoCD deletes resources removed from Git

---

## Part 5 — Flux CD (Awareness)

Flux is the other major CNCF GitOps tool. The KCNA exam expects awareness of both.

| Feature | ArgoCD | Flux |
|---------|--------|------|
| UI | Yes (built-in) | No (use Weave GitOps) |
| Multi-tenancy | App Projects | Multi-tenancy via namespaces |
| OCI support | Yes | Yes (stronger) |
| Helm support | Yes | Yes (HelmRelease CRD) |
| CNCF status | Graduated | Graduated |

Both implement the GitOps spec. The choice is often about team preference and existing tooling.

---

## Part 6 — Helm vs Kustomize

The KCNA also tests awareness of **Kustomize**, which is bundled with `kubectl`.

```bash
mkdir -p ~/kcna-labs/module-07/kustomize/base

cat <<EOF > ~/kcna-labs/module-07/kustomize/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: nginx:1.24
EOF

cat <<EOF > ~/kcna-labs/module-07/kustomize/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
EOF

# Create a prod overlay
mkdir -p ~/kcna-labs/module-07/kustomize/overlays/prod
cat <<EOF > ~/kcna-labs/module-07/kustomize/overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
- ../../base
patches:
- patch: |-
    - op: replace
      path: /spec/replicas
      value: 5
  target:
    kind: Deployment
    name: app
images:
- name: nginx
  newTag: "1.25"
EOF

# Preview
kubectl kustomize ~/kcna-labs/module-07/kustomize/overlays/prod
```

| Approach | Templating | Learning Curve | Best for |
|----------|-----------|----------------|----------|
| Helm | Go templates + values | Steep | Packaged, reusable charts |
| Kustomize | Patch overlays, no templates | Gentle | Customising existing YAML |

---

## Cleanup

```bash
kubectl delete namespace module-07 argocd
helm uninstall my-nginx -n module-07 2>/dev/null
```

## ➡️ Next: [Final Challenge — Deploy a Full Cloud-Native App](../final-challenge/README.md)
