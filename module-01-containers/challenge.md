# Module 01 · Challenge — Containerise a Real-World App

> **KCNA Domain:** Container Orchestration (22%)\n> **No step-by-step guide from here.** Use what you learned in Labs 01 and 02.  
> Hints are available but try without them first.

---

## The Mission

You've been handed a Node.js API. Your job:

1. Containerise it with a **production-grade** Dockerfile
2. Keep the final image **under 100MB**
3. The app must **not run as root**
4. Pass a vulnerability scan with **zero HIGH or CRITICAL CVEs**

---

## The App

Create this directory and files:

```bash
mkdir -p ~/kcna-labs/module-01/challenge && cd ~/kcna-labs/module-01/challenge
```

`package.json`:
```json
{
  "name": "kcna-challenge",
  "version": "1.0.0",
  "dependencies": {
    "express": "^4.18.0"
  }
}
```

`server.js`:
```javascript
const express = require('express');
const app = express();

app.get('/health', (req, res) => res.json({ status: 'ok' }));
app.get('/', (req, res) => res.json({
  message: 'KCNA Challenge App',
  node: process.version,
  user: process.env.USER || 'unknown'
}));

app.listen(3000, () => console.log('Listening on :3000'));
```

---

## Requirements

| Requirement | Verification Command |
|-------------|---------------------|
| Image < 100MB | `docker images challenge-app:final` |
| App responds on port 3000 | `curl localhost:3000` |
| App responds on /health | `curl localhost:3000/health` |
| Not running as root | `docker exec <id> whoami` → must NOT be `root` |
| Zero HIGH/CRITICAL CVEs | `trivy image challenge-app:final --severity HIGH,CRITICAL` |

---

## Hints (expand only if stuck)

<details>
<summary>Hint 1 — Which base image?</summary>

`node:20-alpine` is a good starting point for size. `node:20-slim` is another option.
For distroless: `gcr.io/distroless/nodejs20-debian12` — but you'll need to handle `node_modules` carefully.

</details>

<details>
<summary>Hint 2 — Running as non-root</summary>

Alpine has a built-in `node` user. After your setup steps:
```dockerfile
USER node
```
Make sure your `WORKDIR` is writable by the node user — set ownership with `chown` in a `RUN` step.

</details>

<details>
<summary>Hint 3 — Keeping image small with npm</summary>

```dockerfile
RUN npm ci --only=production
```
`npm ci` is cleaner than `npm install` in CI/CD contexts. `--only=production` skips devDependencies.

</details>

<details>
<summary>Hint 4 — Multi-stage for Node.js</summary>

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json .
RUN npm ci --only=production

FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY server.js .
USER node
EXPOSE 3000
CMD ["node", "server.js"]
```

</details>

---

## Bonus Challenge

1. Add a `.dockerignore` file — what belongs in it?
2. Make the app print the container's hostname in the response (use `os.hostname()`)
3. Build for two platforms: `linux/amd64` and `linux/arm64` using `docker buildx`

---

## Self-Assessment

After completing the challenge, answer these (no googling — test your understanding):

- Why does installing npm packages in a separate stage help with cache?
- What would happen if you ran the container as `root` and the app had a remote code execution vulnerability?
- What's the difference between `CMD` and `ENTRYPOINT`?
- Why is `COPY package*.json .` before `COPY . .` in most Dockerfiles?

---

## ➡️ Next Module: [Module 02 — Kubernetes Architecture](../module-02-kubernetes-architecture/lab-01-cluster-components.md)
