# Module 03 · Lab 02 — Deployments, Rollouts, and Self-Healing

> **KCNA Domain:** Kubernetes Fundamentals (46%)  
> **Time:** ~60 minutes  
> **Namespace:** `kubectl create namespace module-03`

---

## What You'll Build

A deployment pipeline that demonstrates Kubernetes' self-healing, rolling updates, and rollback capabilities — the features that make Kubernetes useful in production.

---

## Part 1 — Deployment Internals

### 1.1 Create a Deployment

```bash
kubectl create namespace module-03

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: module-03
  annotations:
    deployment.kubernetes.io/revision: "1"
spec:
  replicas: 4
  selector:
    matchLabels:
      app: webapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: webapp
        version: "v1"
    spec:
      containers:
      - name: app
        image: nginx:1.24
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 3
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
EOF
```

```bash
kubectl rollout status deployment/webapp -n module-03
kubectl get pods -n module-03 -o wide
```

### 1.2 Understand the Deployment → ReplicaSet → Pod hierarchy

```bash
kubectl get deployment,replicaset,pod -n module-03
```

Notice the ReplicaSet name is `webapp-<hash>`. The hash is derived from the pod template spec. This will become important during rolling updates.

---

## Part 2 — Rolling Update

### 2.1 Trigger a rolling update

Open a second terminal and watch the rollout:
```bash
# Terminal 2
kubectl get pods -n module-03 -w
```

Update the image in Terminal 1:
```bash
kubectl set image deployment/webapp app=nginx:1.25 -n module-03
```

Watch Terminal 2. You'll see:
- New pods created (maxSurge=1 allows 5 total pods temporarily)
- Old pods terminated (maxUnavailable=1 means only 1 can be down at once)
- The Deployment never goes below 3 ready replicas

```bash
kubectl rollout status deployment/webapp -n module-03
kubectl get replicaset -n module-03   # Now there are TWO ReplicaSets
```

The old ReplicaSet still exists (with 0 replicas) — this is how rollback works.

### 2.2 Check rollout history

```bash
kubectl rollout history deployment/webapp -n module-03
kubectl rollout history deployment/webapp -n module-03 --revision=1
```

---

## Part 3 — Simulate a Bad Deployment

### 3.1 Push a broken image

```bash
kubectl set image deployment/webapp app=nginx:DOES-NOT-EXIST -n module-03
```

Watch what happens:
```bash
kubectl rollout status deployment/webapp -n module-03
# It will hang — waiting for pods to become ready
kubectl get pods -n module-03
# You'll see new pods in ErrImagePull or ImagePullBackOff
```

The readinessProbe saves you — the broken pods never become ready, so the rolling update stalls. The old pods are still running (maxUnavailable=1 means at least 3 out of 4 are still serving traffic).

### 3.2 Rollback

```bash
kubectl rollout undo deployment/webapp -n module-03
kubectl rollout status deployment/webapp -n module-03
kubectl get pods -n module-03  # All healthy again
```

Rollback to a specific version:
```bash
kubectl rollout undo deployment/webapp -n module-03 --to-revision=1
```

---

## Part 4 — Jobs and CronJobs

### 4.1 A one-time Job

```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  namespace: module-03
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: migrate
        image: busybox
        command: ["sh", "-c", "echo 'Running migration...'; sleep 5; echo 'Done!'"]
EOF

kubectl get job db-migration -n module-03 -w
kubectl logs -l job-name=db-migration -n module-03
```

### 4.2 A parallel Job

```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-work
  namespace: module-03
spec:
  completions: 6      # Run 6 total
  parallelism: 2      # 2 at a time
  backoffLimit: 2
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: worker
        image: busybox
        command: ["sh", "-c", "echo Worker $JOB_COMPLETION_INDEX starting; sleep 3; echo Done"]
EOF

kubectl get pods -n module-03 -w
kubectl get job parallel-work -n module-03
```

### 4.3 CronJob

```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup
  namespace: module-03
spec:
  schedule: "*/1 * * * *"   # every minute
  concurrencyPolicy: Forbid  # don't run overlapping jobs
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: cleanup
            image: busybox
            command: ["sh", "-c", "date && echo 'Cleanup ran'"]
EOF

# Wait ~1 minute, then:
kubectl get cronjob cleanup -n module-03
kubectl get jobs -n module-03
```

---

## Part 5 — DaemonSet

A DaemonSet ensures one pod per node. Used for log collectors, monitoring agents, network plugins.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-logger
  namespace: module-03
spec:
  selector:
    matchLabels:
      app: node-logger
  template:
    metadata:
      labels:
        app: node-logger
    spec:
      containers:
      - name: logger
        image: busybox
        command: ["sh", "-c", "while true; do echo [$(hostname)] $(date); sleep 10; done"]
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
EOF

kubectl get daemonset -n module-03
kubectl get pods -n module-03 -l app=node-logger -o wide
# One pod per worker node
```

Scale your cluster and watch:
```bash
# kind doesn't easily allow adding nodes, but note the concept:
# If you added a node, the DaemonSet controller would automatically schedule a pod on it
```

---

## Part 6 — StatefulSet (Overview)

StatefulSets give pods **stable identities** — predictable names (`app-0`, `app-1`, `app-2`) and persistent storage per pod. Used for databases, message queues.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
  namespace: module-03
spec:
  serviceName: "db"
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: db
        image: nginx   # pretend this is a DB
        ports:
        - containerPort: 80
EOF

kubectl get pods -n module-03 -l app=db
# Note: db-0, db-1, db-2 — always in order, always those names
```

Delete `db-1` and watch:
```bash
kubectl delete pod db-1 -n module-03
kubectl get pods -n module-03 -w
# db-1 comes back as db-1, not some random hash name
```

---

## Key Concept Summary

| Workload | Use Case | Restart Policy |
|----------|----------|---------------|
| Deployment | Stateless apps | Always |
| StatefulSet | Stateful apps needing stable IDs | Always |
| DaemonSet | Per-node agents | Always |
| Job | Batch, run-to-completion | OnFailure / Never |
| CronJob | Scheduled batch | OnFailure / Never |

---

## Cleanup

```bash
kubectl delete namespace module-03
```

## ➡️ Next: [Module 03 Challenge](challenge.md)
