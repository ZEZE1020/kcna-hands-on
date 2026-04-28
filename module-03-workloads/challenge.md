# Module 03 · Challenge — Fix a Broken Deployment

> **KCNA Domain:** Kubernetes Fundamentals (46%)\n> **No walkthroughs.** Debug and fix a production incident using your workload knowledge.

---

## The Incident

You've been paged at 2am. The `payments` service is down. Your job is to diagnose and fix it.

---

## Setup (Deploy the Broken Environment)

> **Important:** If you've run this challenge before, delete the existing namespace first: `kubectl delete namespace module-03-challenge`

```bash
kubectl create namespace module-03-challenge

kubectl apply -f - <<'EOF'
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments
  namespace: module-03-challenge
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payments
  template:
    metadata:
      labels:
        app: payments
    spec:
      containers:
      - name: app
        image: nginx:BROKEN-TAG        # Bug 1
        resources:
          requests:
            memory: "900Gi"            # Bug 2: Requesting more RAM than exists
          limits:
            memory: "900Gi"
        readinessProbe:
          httpGet:
            path: /health
            port: 9999                 # Bug 3: nginx listens on 80, not 9999
          initialDelaySeconds: 2
          periodSeconds: 3
        env:
        - name: DB_PASSWORD
          value: "hardcoded-secret"    # Bug 4: not from a Secret
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: payment-reconciler
  namespace: module-03-challenge
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Always        # Bug 5: invalid for Job/CronJob
          containers:
          - name: reconciler
            image: busybox
            command: ["sh", "-c", "echo reconciling"]
EOF
```

---

## Your Tasks

1. Identify and fix all 5 bugs
2. After fixing, all 3 pods should be `Running` and `Ready`
3. The CronJob should be valid and schedulable
4. The DB_PASSWORD must come from a `Secret`, not hardcoded

---

## Pass Criteria (Explicit)

Your fixes are complete when ALL of the following are true:

1. **All payment pods are Ready:**
   ```bash
   kubectl get pods -n module-03-challenge -l app=payments
   ```
   Should show: 3 pods with `Status: Running` and `Ready: 1/1`

2. **CronJob is accepted by the API:**
   ```bash
   kubectl get cronjob payment-reconciler -n module-03-challenge
   ```
   Should show: Status `Active` (or blank if not yet triggered)

3. **A Secret exists for the password:**
   ```bash
   kubectl get secret -n module-03-challenge
   ```
   Should show a secret (e.g., `db-secret`)

4. **DB_PASSWORD is wired from the Secret (not hardcoded):**
   ```bash
   kubectl get deployment payments -n module-03-challenge -o yaml | grep -A5 "env:"
   ```
   Should show `valueFrom.secretKeyRef`, not a literal `value:`

---

## Verification

---

## Hints

<details>
<summary>Bug 1</summary>
`nginx:BROKEN-TAG` doesn't exist. Use `nginx:1.25`.
</details>

<details>
<summary>Bug 2</summary>
900Gi of RAM doesn't exist on any node. Use `memory: "64Mi"` — a much smaller request that fits within node allocatable memory.
</details>

<details>
<summary>Bug 3</summary>
nginx listens on port 80 and serves `/` by default (no `/health` route exists). Change the readinessProbe to:
- `port: 80` (not 9999)
- `path: "/"` (not `/health`)
</details>

<details>
<summary>Bug 4</summary>
Create a Secret: `kubectl create secret generic db-secret --from-literal=DB_PASSWORD=real-password -n module-03-challenge`
Then reference it with `secretKeyRef`.
</details>

<details>
<summary>Bug 5</summary>
`restartPolicy: Always` is not valid for Job pods. Use `restartPolicy: OnFailure` or `Never`.
</details>

---

## ➡️ Next Module: [Module 04 — Services, DNS, and Network Policy](../module-04-networking-and-services/lab-01-services-dns-networkpolicy.md)
