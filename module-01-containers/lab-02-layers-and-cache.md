# Module 01 · Lab 02 — Layers, Multi-Stage Builds, and Image Hygiene

> **KCNA Domain:** Container Orchestration (22%)  
> **Time:** ~30 minutes  
> **Prerequisite:** Lab 01 complete  
> **Tools:** Docker (Part 1–3), Trivy (Part 4 — optional but recommended)

---

## What You'll Learn

- How multi-stage builds dramatically reduce image size
- Why small images matter in cloud-native environments (faster pulls, smaller attack surface)
- The OCI image spec and what "distroless" means
- How `docker scout` / `trivy` finds vulnerabilities in layers

---

## Part 1 — The Problem with Fat Images

### 1.1 Build a Go app the naive way

```bash
mkdir -p ~/kcna-labs/module-01/goapp && cd ~/kcna-labs/module-01/goapp
```

Create `main.go`:
```go
package main

import (
    "fmt"
    "net/http"
    "os"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello from %s!\n", os.Getenv("APP_NAME"))
    })
    http.ListenAndServe(":8080", nil)
}
```

Single-stage Dockerfile (the wrong way):
```dockerfile
FROM golang:1.22
WORKDIR /app
COPY main.go .
RUN go build -o server main.go
EXPOSE 8080
CMD ["./server"]
```

```bash
docker build -t goapp:fat .
docker images goapp:fat
```

> **Port cleanup note:** If you have a container still running on port 8080 from Lab 01, stop it first with `docker stop my-app`. Otherwise the `docker run` command below will fail with "port already in use".

Note the image size. It will be **~800MB+** — because the entire Go toolchain is included.

---

## Part 2 — Multi-Stage Build

### 2.1 Rewrite the Dockerfile

```dockerfile
# Stage 1: Build
FROM golang:1.22 AS builder
WORKDIR /app
COPY main.go .
RUN CGO_ENABLED=0 go build -o server main.go

# Stage 2: Run
FROM scratch
COPY --from=builder /app/server /server
EXPOSE 8080
CMD ["/server"]
```

```bash
docker build -t goapp:lean .
docker images | grep goapp
```

Compare the sizes. `goapp:lean` should be **~7MB**. The build tools never made it into the final image.

**Why this matters for KCNA:** Smaller images pull faster in Kubernetes — critical during pod scheduling, autoscaling, and node failures. The exam expects you to understand *why* multi-stage builds exist.

---

## Part 3 — Distroless Images

`scratch` is truly empty — not even a shell. `distroless` images include minimal OS libs without a shell or package manager.

```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY main.go .
RUN CGO_ENABLED=0 go build -o server main.go

FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/server /server
EXPOSE 8080
CMD ["/server"]
```

```bash
docker build -t goapp:distroless .
docker run -d -p 8080:8080 -e APP_NAME="Distroless" goapp:distroless
curl localhost:8080

# Get the container ID
CONTAINER_ID=$(docker ps -q -f "ancestor=goapp:distroless")
echo "Container ID: $CONTAINER_ID"

# Try to exec — you'll fail (no shell)
docker exec -it $CONTAINER_ID sh   # Error: no such file
```

**Security implication:** No shell = attacker can't easily explore the container if it's compromised.

---

## Part 4 — Scanning for Vulnerabilities (Optional)

**Prerequisites:** Trivy requires host-level tools (`wget`, `tar`, `curl`) and optionally `sudo` or `brew`. This section is optional if you prefer to skip vulnerability scanning.

### 4.1 Install Trivy (lightweight vulnerability scanner)

```bash
# Linux
wget https://github.com/aquasecurity/trivy/releases/download/v0.50.0/trivy_0.50.0_Linux-64bit.tar.gz
tar xzf trivy_0.50.0_Linux-64bit.tar.gz
sudo mv trivy /usr/local/bin/

# macOS
brew install trivy
```

### 4.2 Scan your images

```bash
trivy image goapp:fat
trivy image goapp:lean
trivy image goapp:distroless
```

Notice how the fat image has **many more CVEs** — Go build tools, system libs, etc. The distroless image has close to zero.

---

## Part 5 — Image Tagging Strategy

```bash
# Bad: always re-tag latest — you lose history
docker tag goapp:lean myrepo/goapp:latest

# Good: use semantic versioning + git SHA
GIT_SHA=$(git rev-parse --short HEAD 2>/dev/null || echo "abc123")
docker tag goapp:lean myrepo/goapp:1.0.0-${GIT_SHA}
```

> **Note:** The `git rev-parse --short HEAD` command gets the current commit SHA. If your lab folder is not in a git repository, it falls back to a placeholder like `abc123`. For real projects, this would tie each image to a specific commit.

**KCNA concept:** Immutable tags (`1.0.0-abc123`) are a cloud-native best practice. Using `latest` in production is an antipattern because you lose traceability.

---

## Cleanup

```bash
# Stop and remove any running containers
docker stop $(docker ps -a -q -f "ancestor=goapp:distroless" 2>/dev/null) 2>/dev/null

# Remove images
docker rmi goapp:fat goapp:lean goapp:distroless

# If you installed Trivy, optionally remove it (not necessary)
# rm /usr/local/bin/trivy
```

**Note:** Always clean up containers before removing images. If a container is still running or stopped but not removed, `docker rmi` will fail with "image is being used".

---

## Key Concepts Covered

- Multi-stage builds = build tools stay out of the final image
- `scratch` vs `distroless` vs `alpine` — tradeoffs between size, debuggability, security
- CVE surface area is proportional to image size
- Image tagging: `latest` is a lie in production

---

## ➡️ Next: [Module 01 Challenge](challenge.md)
