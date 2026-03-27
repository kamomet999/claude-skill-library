---
name: docker-optimize
description: Docker最適化スキル。イメージサイズ削減、ビルド高速化、セキュリティ強化、マルチステージビルド。Dockerfileの改善時に使用。
category: 開発ワークフロー
command: /docker-opt
version: 1.0.0
tags:
  - docker
  - optimization
  - container
  - security
---

# Docker Optimization

Source: [wshobson/commands](https://github.com/wshobson/commands) (2.2k stars) - 軽量版に要約

## Strategy Selection

| App Type | Base Image | Patterns |
|----------|-----------|----------|
| Web App | alpine or distroless | multi-stage, layer caching |
| Microservice | scratch or distroless | static compilation, minimal deps |
| Data Processing | slim or specific runtime | parallel processing, volume opt |
| ML | nvidia/cuda | model optimization, multi-stage |

## Multi-Stage Build Patterns

### Node.js / TypeScript
```dockerfile
# Build stage
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && cp -R node_modules prod_modules
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/prod_modules ./node_modules
COPY --from=builder /app/dist ./dist
USER node
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

### Go
```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server

FROM scratch
COPY --from=builder /app/server /server
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8080
ENTRYPOINT ["/server"]
```

### Python
```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /install /usr/local
COPY . .
USER nobody
CMD ["python", "main.py"]
```

## Image Size Optimization

### Layer Ordering (cache efficiency)
```dockerfile
# GOOD: Least-changing layers first
COPY package*.json ./
RUN npm ci
COPY . .

# BAD: Invalidates cache on any file change
COPY . .
RUN npm ci
```

### Minimize Layers
```dockerfile
# GOOD: Single RUN
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

# BAD: Multiple layers + leftover cache
RUN apt-get update
RUN apt-get install -y curl
```

### .dockerignore
```
node_modules
.git
*.md
.env*
dist
coverage
.next
```

## Security Hardening

### Non-Root User
```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

### Read-Only Filesystem
```dockerfile
# docker-compose.yml
services:
  app:
    read_only: true
    tmpfs:
      - /tmp
```

### Health Checks
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1
```

### Scan for Vulnerabilities
```bash
# Trivy
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image myapp:latest

# Hadolint (Dockerfile linting)
docker run --rm -i hadolint/hadolint < Dockerfile
```

## Quick Checklist

- [ ] Multi-stage build used
- [ ] Alpine or distroless base image
- [ ] Non-root user (`USER`)
- [ ] `.dockerignore` configured
- [ ] No secrets in image (use build args or runtime env)
- [ ] `HEALTHCHECK` defined
- [ ] Layer order optimized (deps before source)
- [ ] `--no-cache-dir` for pip, `npm ci` for node
- [ ] Single `RUN` for apt-get install + cleanup
- [ ] Production deps only (no devDependencies)

## Size Comparison

| Base Image | Size |
|-----------|------|
| ubuntu:22.04 | ~77MB |
| python:3.12 | ~1GB |
| python:3.12-slim | ~130MB |
| python:3.12-alpine | ~50MB |
| node:20 | ~1GB |
| node:20-alpine | ~130MB |
| golang:1.22 | ~800MB |
| scratch (Go binary) | ~10-20MB |
| gcr.io/distroless | ~20MB |
