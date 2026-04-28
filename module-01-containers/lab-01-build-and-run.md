# Module 01 · Lab 01 — Build, Run, and Understand Containers

> **KCNA Domain:** Container Orchestration (22%)  
> **Time:** ~45 minutes  
> **Tools:** Docker only (no cluster needed yet)

---

## What You'll Build

A small Python web app, containerised from scratch. Along the way you'll understand image layers, the difference between images and containers, and why containers are isolated but not VMs.

---

## Part 1 — Your First Dockerfile

### 1.1 Create the app

```bash
mkdir -p ~/kcna-labs/module-01/app && cd ~/kcna-labs/module-01/app
```

Create `app.py`:
```python
from http.server import HTTPServer, BaseHTTPRequestHandler
import os

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        name = os.environ.get("APP_NAME", "KCNA Lab")
        self.send_response(200)
        self.end_headers()
        self.wfile.write(f"Hello from {name}!\n".encode('utf-8'))

HTTPServer(( "localhost", 8080), Handler).serve_forever()

```

### 1.2 Write the Dockerfile

```dockerfile
# Dockerfile
FROM python:3.12-slim

WORKDIR /app
COPY app.py .

EXPOSE 8080
CMD ["python", "app.py"]
```

### 1.3 Build and run

```bash
docker build -t kcna-app:v1 .
docker run -d -p 8080:8080 --name my-app kcna-app:v1
curl localhost:8080
```

Expected: `Hello from KCNA Lab!`

> **Note:** The app binds to `0.0.0.0:8080` (not `localhost`), which makes it reachable through Docker port mapping. If the app used `localhost` (127.0.0.1), the container would not be reachable from the host.

---

## Part 2 — Understanding Layers

### 2.1 Inspect the image layers

```bash
docker history kcna-app:v1
```

You'll see each instruction in the Dockerfile created a layer. The `FROM python:3.12-slim` layer is the biggest — it's pulled from a registry, not rebuilt.

### 2.2 See layer caching in action

Add a `requirements.txt` (even empty):
```bash
touch requirements.txt
```

Update the Dockerfile to copy requirements *before* the app code:
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .          # ← copy deps first
RUN pip install -r requirements.txt
COPY app.py .                    # ← app code last
EXPOSE 8080
CMD ["python", "app.py"]
```

Build twice:
```bash
docker build -t kcna-app:v2 .   # first build — all layers run
docker build -t kcna-app:v2 .   # second build — watch "CACHED"
```

**Why this matters for KCNA:** The exam tests whether you understand that images are immutable, layered, and that layers are shared between images. `COPY requirements.txt` before `COPY app.py` is a real-world best practice that reflects this understanding.

---

## Part 3 — Environment Variables and Runtime Config

### 3.1 Pass an environment variable

> **Tip:** If you're rerunning this lab, remove the existing `my-app-2` container first: `docker rm my-app-2` (stop it if running: `docker stop my-app-2`). Container names must be unique.

```bash
docker run -d -p 8081:8080 -e APP_NAME="Module 01" --name my-app-2 kcna-app:v1
curl localhost:8081
```

Expected: `Hello from Module 01!`

### 3.2 Inspect the running container

```bash
docker inspect my-app-2 | grep -A5 '"Env"'
docker exec my-app-2 env
docker logs my-app-2
```

**Concept check:** The container image didn't change — only the runtime configuration did. This is the foundation of Kubernetes `ConfigMaps` and `Secrets` (Module 03).

---

## Part 4 — Container Isolation

### 4.1 Network isolation

```bash
# Run two containers
docker run -d --name alpha kcna-app:v1
docker run -d --name beta kcna-app:v1

# What IP does alpha have?
docker inspect alpha | grep IPAddress

# Can alpha reach beta? (Note: ping may not be available in slim images)
docker exec alpha ping -c3 <beta-ip>
```

Try to curl `beta` from your host — you can't unless you published the port.

### 4.2 Process isolation

> **Environment note:** The `ps` command may not be available in minimal images like `python:3.12-slim`. The following checks the isolation concept; adapt based on your image.

```bash
docker exec alpha ps aux 2>/dev/null || echo "(ps not available in this image)"
ps aux | grep python         # host sees them — they're just Linux processes
```

**Concept check:** Containers are NOT VMs. They're isolated Linux processes using namespaces and cgroups. The host kernel is shared.

---

## Part 5 — Push to a Registry (Optional)

This section is **optional**. It requires a Docker Hub account or a local registry, but the core lab works without it.

### 5.1 Tag and push (optional, needs Docker Hub account)

```bash
docker tag kcna-app:v1 <your-dockerhub-username>/kcna-app:v1
docker push <your-dockerhub-username>/kcna-app:v1
```

If you don't have Docker Hub, use a local registry:
```bash
docker run -d -p 5000:5000 --name registry registry:2
docker tag kcna-app:v1 localhost:5000/kcna-app:v1
docker push localhost:5000/kcna-app:v1
```

---

## Cleanup

```bash
docker stop my-app my-app-2 alpha beta
docker rm my-app my-app-2 alpha beta
docker rmi kcna-app:v1 kcna-app:v2
```

---

## Key Concepts Covered

- **OCI image format** — layers, manifests, tags
- **Docker image ≠ container** — an image is a blueprint; a container is a running instance
- **Layer caching** — why ordering in a Dockerfile matters
- **Runtime config** — environment variables vs baked-in config
- **Container isolation** — namespaces, not VMs

---

## ➡️ Next: [Lab 02 — Layers, Multi-Stage Builds, and Image Size](lab-02-layers-and-cache.md)
