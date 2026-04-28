# Module 01 · Lab 02 — Layers, Multi-Stage Builds, and Image Hygiene

> **KCNA Domain:** Container Orchestration (22%)  
> **Time:** ~30 minutes  
> **Prerequisite:** Lab 01 complete

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

# Try to exec — you'll fail (no shell)
docker exec -it <container-id> sh   # Error: no such file
```

**Security implication:** No shell = attacker can't easily explore the container if it's compromised.

---

## Part 4 — Scanning for Vulnerabilities

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

**KCNA concept:** Immutable tags (`1.0.0-abc123`) are a cloud-native best practice. Using `latest` in production is an antipattern because you lose traceability.

---

## Cleanup

```bash
docker rmi goapp:fat goapp:lean goapp:distroless
```

---

## Key Concepts Covered

- Multi-stage builds = build tools stay out of the final image
- `scratch` vs `distroless` vs `alpine` — tradeoffs between size, debuggability, security
- CVE surface area is proportional to image size
- Image tagging: `latest` is a lie in production

---

## ➡️ Next: [Module 01 Challenge](challenge.md)
