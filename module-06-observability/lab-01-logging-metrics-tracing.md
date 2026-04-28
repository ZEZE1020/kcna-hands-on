# Module 06 · Lab 01 — Observability: Logs, Metrics, and Tracing

> **KCNA Domain:** Cloud Native Observability (8%)  
> **Time:** ~90 minutes  
> **Tools:** Prometheus, Grafana, Loki, Jaeger

---

## The Three Pillars of Observability

The KCNA exam expects you to understand the CNCF observability landscape. This lab gives you hands-on time with the real tools.

| Pillar | What it answers | CNCF tools |
|--------|----------------|------------|
| **Logs** | What happened? | Loki, Fluentd, Fluentbit |
| **Metrics** | How much / how often? | Prometheus, Thanos |
| **Traces** | Why is it slow? | Jaeger, Zipkin, Tempo |

---

## Setup: Monitoring Stack

```bash
kubectl create namespace monitoring
kubectl create namespace module-06

# Add Helm repos
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack (Prometheus + Grafana + Alertmanager)
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=kcna-labs \
  --set prometheus.prometheusSpec.retention=1h \
  --set alertmanager.enabled=false \
  --wait --timeout 5m

kubectl get pods -n monitoring
```

---

## Part 1 — Logs

### 1.1 Container logs in Kubernetes

```bash
# Deploy an app that generates structured logs
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: log-generator
  namespace: module-06
spec:
  replicas: 2
  selector:
    matchLabels:
      app: log-generator
  template:
    metadata:
      labels:
        app: log-generator
    spec:
      containers:
      - name: app
        image: busybox
        command: ["sh", "-c", "i=0; while true; do echo \"{\\\"timestamp\\\": \\\"$(date -Iseconds)\\\", \\\"level\\\": \\\"INFO\\\", \\\"msg\\\": \\\"request processed\\\", \\\"req_id\\\": $i, \\\"duration_ms\\\": $((RANDOM % 200))}\"; i=$((i+1)); sleep 2; done"]
EOF
```

### 1.2 Basic log commands

```bash
# Get logs from a specific pod
kubectl logs -n module-06 -l app=log-generator

# Stream logs in real time
kubectl logs -n module-06 -l app=log-generator -f

# Logs from the previous container crash (invaluable for debugging CrashLoopBackOff)
kubectl logs -n module-06 <pod-name> --previous

# All containers in a pod (multi-container pods)
kubectl logs -n module-06 <pod-name> --all-containers

# Last 50 lines
kubectl logs -n module-06 <pod-name> --tail=50

# Logs since a time
kubectl logs -n module-06 <pod-name> --since=5m
```

### 1.3 Why structured logging matters

Notice the JSON format above. Structured logs (JSON) can be parsed and queried by log aggregators. Unstructured logs (plain text) are hard to search at scale.

**KCNA concept:** In cloud-native apps, logs go to `stdout`/`stderr` — never to files. The container runtime captures them. Tools like Fluentd or Fluent Bit collect and forward them.

### 1.4 Sidecar log shipping pattern

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-with-log-sidecar
  namespace: module-06
spec:
  volumes:
  - name: log-volume
    emptyDir: {}
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "while true; do echo $(date) request >> /var/log/app.log; sleep 2; done"]
    volumeMounts:
    - name: log-volume
      mountPath: /var/log
  - name: log-shipper
    image: busybox
    command: ["sh", "-c", "tail -f /var/log/app.log"]  # Simulate forwarding to log aggregator
    volumeMounts:
    - name: log-volume
      mountPath: /var/log
EOF

kubectl logs app-with-log-sidecar -c log-shipper -n module-06 -f
```

---

## Part 2 — Metrics with Prometheus

### 2.1 Access Prometheus

```bash
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090 &
```

Open http://localhost:9090 in your browser.

### 2.2 Query the data with PromQL

In the Prometheus UI, try these queries:

```promql
# CPU usage per container
rate(container_cpu_usage_seconds_total{namespace="module-06"}[5m])

# Memory usage
container_memory_usage_bytes{namespace="module-06"}

# Pod restarts (critical for detecting CrashLoopBackOff)
kube_pod_container_status_restarts_total{namespace="module-06"}

# Number of running pods per deployment
kube_deployment_status_replicas_available

# Node CPU usage
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### 2.3 Create a pod that generates custom metrics

```bash
# Expose a /metrics endpoint that Prometheus can scrape
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: metrics-app
  namespace: module-06
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
spec:
  containers:
  - name: app
    image: prom/node-exporter   # This exposes real node metrics
    ports:
    - containerPort: 8080
EOF
```

### 2.4 Access Grafana

```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80 &
# Login: admin / kcna-labs
```

Browse the pre-built dashboards: **Kubernetes / Pods**, **Kubernetes / Nodes**, **Kubernetes / Deployments**

---

## Part 3 — Distributed Tracing with Jaeger

### 3.1 Deploy Jaeger

```bash
# Jaeger all-in-one for dev/testing
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
      - name: jaeger
        image: jaegertracing/all-in-one:latest
        env:
        - name: COLLECTOR_OTLP_ENABLED
          value: "true"
        ports:
        - containerPort: 16686  # UI
        - containerPort: 4317   # OTLP gRPC
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger
  namespace: monitoring
spec:
  selector:
    app: jaeger
  ports:
  - name: ui
    port: 16686
  - name: otlp-grpc
    port: 4317
EOF

kubectl wait --for=condition=ready pod -l app=jaeger -n monitoring --timeout=60s
kubectl port-forward -n monitoring svc/jaeger 16686:16686 &
```

Open http://localhost:16686 — this is the Jaeger UI.

### 3.2 Understand trace concepts

**KCNA key concepts:**

- **Trace** — the full journey of one request through multiple services
- **Span** — one operation within a trace (a single service call, a DB query, etc.)
- **TraceID** — a UUID that links all spans in one request
- **OpenTelemetry (OTel)** — the CNCF standard for instrumentation; vendor-neutral

```
Request: User → Frontend → API → Database

Trace ID: abc-123
  Span 1: Frontend processes request          (10ms)
    Span 2: API receives and handles           (45ms)
      Span 3: Database query                   (30ms)
```

Without tracing, you'd see high latency but not *which service* caused it.

---

## Part 4 — The Golden Signals

Google's SRE book defines 4 metrics every service should expose. The KCNA exam references these:

| Signal | What it measures | PromQL example |
|--------|-----------------|----------------|
| **Latency** | How long requests take | `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))` |
| **Traffic** | How many requests/sec | `rate(http_requests_total[5m])` |
| **Errors** | Rate of failed requests | `rate(http_requests_total{status=~"5.."}[5m])` |
| **Saturation** | How full is the system | `container_memory_usage_bytes / container_spec_memory_limit_bytes` |

### 4.1 Exercise: Set up an alert

```bash
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: pod-restart-alert
  namespace: monitoring
  labels:
    release: monitoring
spec:
  groups:
  - name: pod-alerts
    rules:
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"
        description: "Pod {{ $labels.pod }} in namespace {{ $labels.namespace }} has restarted more than once in 15 minutes"
EOF
```

Trigger it:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: crasher
  namespace: module-06
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "exit 1"]
  restartPolicy: Always
EOF

kubectl get pod crasher -n module-06 -w
# Watch it crash and restart — CrashLoopBackOff
```

---

## Part 5 — Liveness vs Readiness vs Startup Probes

These feed directly into observability — they're how Kubernetes knows if your app is healthy.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
  namespace: module-06
spec:
  containers:
  - name: app
    image: nginx
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 5
      failureThreshold: 3
      # If this fails 3 times in a row → container is RESTARTED
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 3
      # If this fails → pod removed from Service endpoints (no traffic)
    startupProbe:
      httpGet:
        path: /
        port: 80
      failureThreshold: 30
      periodSeconds: 10
      # Gives slow-starting apps 5 minutes before liveness kicks in
EOF
```

| Probe | Failure action | Use case |
|-------|---------------|----------|
| Liveness | Restart container | Detect deadlocks |
| Readiness | Remove from Service | App still starting, or overloaded |
| Startup | Delay liveness/readiness | Slow-start apps (JVM, ML models) |

---

## Cleanup

```bash
kubectl delete namespace module-06
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring
```

## ➡️ Next: [Module 07 — Application Delivery: Helm and GitOps](../module-07-application-delivery/lab-01-helm-basics.md)
