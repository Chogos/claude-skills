---
name: writing-dockerfiles
description: Dockerfile development best practices. Use when creating, modifying, or reviewing Dockerfiles, .dockerignore files, or container image build configurations.
---

# Dockerfile Development Best Practices

## Base Images

- Pin to specific version: `FROM node:22.14-alpine3.21`. Never `latest` or untagged.
- Pin by digest for reproducibility: `FROM node:22.14-alpine3.21@sha256:abc123...`.
- Prefer minimal bases: distroless > alpine/slim > full.
  - `distroless`: no shell, no package manager — smallest attack surface, production only.
  - `alpine`: ~5MB, has shell and apk — good default for Go/Rust static binaries. Can cause issues with Python/Node/Java native modules due to musl libc.
  - `slim`: ~70MB Debian with minimal packages — preferred for Python, Node.js, Java (glibc compatibility).
- Cross-architecture: `FROM --platform=linux/amd64 image:tag`.

## Layer Ordering & Caching

Order instructions from least-changed to most-changed:

```dockerfile
# 1. System deps (rarely change)
RUN apt-get update && apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

# 2. App dependency files (change occasionally)
COPY package.json package-lock.json ./
RUN npm ci

# 3. Source code (changes frequently)
COPY src/ src/
```

- Each `RUN`, `COPY`, `ADD` creates a layer. `ENV`, `LABEL`, `EXPOSE`, `WORKDIR` are metadata-only.
- Combine install + cleanup in the same `RUN` — separate layers retain deleted files.
- Use `COPY --link` for independent layers that build in parallel.

### CI/CD Build Caching

Local layer cache doesn't persist between CI runs. Use external cache backends:

- **GitHub Actions**:
  ```yaml
  - uses: docker/build-push-action@v6
    with:
      context: .
      cache-from: type=gha
      cache-to: type=gha,mode=max
  ```
  `mode=max` caches all layers (including intermediate stages), not just the final image.
- **Registry cache**: push cache layers to a registry, pull on next build.
  ```bash
  docker buildx build \
    --cache-from type=registry,ref=ghcr.io/org/myapp:cache \
    --cache-to type=registry,ref=ghcr.io/org/myapp:cache,mode=max \
    -t ghcr.io/org/myapp:latest .
  ```
- **Local cache** (self-hosted runners):
  ```bash
  docker buildx build \
    --cache-from type=local,src=/tmp/.buildx-cache \
    --cache-to type=local,dest=/tmp/.buildx-cache-new,mode=max .
  ```
  Rotate cache directories to prevent unbounded growth.

## Multi-stage Builds

Always multi-stage. Name every stage — never reference by index.

```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM deps AS build
COPY tsconfig.json ./
COPY src/ src/
RUN npm run build

FROM node:22-alpine AS runtime
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
USER node
CMD ["node", "dist/server.js"]
```

- Build stage: compilers, dev deps. Runtime stage: only artifacts.
- `COPY --from=builder /path /path` to copy between stages.
- Separate test target for CI: `docker build --target test .`.
- Full language-specific templates: see [patterns/multi-stage-templates.md](patterns/multi-stage-templates.md).

## Instructions

- **COPY over ADD** always. `ADD` only for local tar auto-extraction. For URLs, use `curl`/`wget` in `RUN`.
- **Exec form** for `CMD` and `ENTRYPOINT`:
  ```dockerfile
  CMD ["node", "server.js"]        # correct: receives signals
  # CMD node server.js             # wrong: wraps in /bin/sh -c, breaks signals
  ```
- **ENTRYPOINT + CMD combo**: ENTRYPOINT for the binary, CMD for default args (overridable at `docker run`):
  ```dockerfile
  ENTRYPOINT ["python", "-m", "myapp"]
  CMD ["--port=8080"]
  ```
- **ARG vs ENV**: ARG is build-time only (and visible in `docker history`). ENV persists in the image. ARG before FROM is scoped to the FROM line only.
- **WORKDIR /app** over `RUN mkdir -p /app && cd /app`. Creates the dir and sets it for all subsequent instructions.
- **EXPOSE** is documentation only — does not publish ports.

## Security

- Run as non-root. Add `USER` after all `RUN` instructions that need root:
  ```dockerfile
  RUN addgroup -S appgroup && adduser -S appuser -G appgroup
  USER appuser:appgroup
  ```
- Never bake secrets into images. No `ARG SECRET`, no `COPY .env`, no `ENV API_KEY=...`. Use BuildKit `--mount=type=secret` (see [patterns/security-hardening.md](patterns/security-hardening.md)).
- **Sign images**: `cosign sign --key cosign.key ghcr.io/org/myapp@sha256:abc...`. Verify in CI or Kubernetes admission controller (Kyverno, Connaisseur) before deploy.
- **Verify before pull**: `cosign verify --key cosign.pub ghcr.io/org/myapp@sha256:abc...`. Reject unsigned images in the deploy pipeline.
- `COPY --chown=appuser:appgroup` to set ownership without extra layers.
- Clean package caches in the same `RUN`:
  ```dockerfile
  RUN apt-get update && apt-get install -y --no-install-recommends curl && \
      rm -rf /var/lib/apt/lists/*
  ```

## .dockerignore

Always create one. Without it, the entire directory (`.git`, `node_modules`, local env) goes into the build context — on large repos this can send GBs to the daemon and dramatically slow builds.

```
.git
.github
.vscode
.idea
*.md
docker-compose*.yml
Dockerfile*
.dockerignore
.env*
**/node_modules/
**/__pycache__/
**/test/
**/tests/
```

## Labels & Metadata

Use OCI image-spec labels:
```dockerfile
LABEL org.opencontainers.image.source="https://github.com/org/repo" \
      org.opencontainers.image.version="1.0.0" \
      org.opencontainers.image.description="My service"
```

## Health Checks

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD ["curl", "-f", "http://localhost:8080/healthz"]
```

For distroless (no curl): compile a static healthcheck binary or use the runtime's built-in health endpoint.

## Signal Handling

- Exec form `CMD ["binary"]` runs as PID 1 and receives signals directly.
- Shell form `CMD binary` runs under `/bin/sh -c` — SIGTERM goes to shell, not the app.
- For scripts as entrypoints: `exec` to replace the shell, or use `tini`:
  ```dockerfile
  RUN apk add --no-cache tini
  ENTRYPOINT ["tini", "--"]
  CMD ["node", "server.js"]
  ```

## New Dockerfile workflow

```
- [ ] Choose minimal base image with pinned version
- [ ] Create .dockerignore
- [ ] Design multi-stage build (deps -> build -> runtime)
- [ ] Order layers for cache efficiency
- [ ] Add non-root USER
- [ ] Add HEALTHCHECK
- [ ] Add OCI labels
- [ ] Run validation loop (below)
```

## Validation loop

1. `hadolint Dockerfile` — fix all warnings (DL=Dockerfile rules, SC=ShellCheck rules)
2. `docker build --no-cache -t test .` — fix build errors
3. `trivy image test` or `grype test` — fix CVEs in base image or deps
4. `docker run --rm test` — verify the container starts and works
5. Repeat until hadolint is clean, scan is acceptable, container runs correctly

## Deep-dive references

**Multi-stage templates**: See [patterns/multi-stage-templates.md](patterns/multi-stage-templates.md) for Go, Node.js, Python, Java, Rust
**Security hardening**: See [patterns/security-hardening.md](patterns/security-hardening.md) for distroless, secrets, scanning
**Optimization**: See [patterns/optimization-patterns.md](patterns/optimization-patterns.md) for cache mounts, BuildKit, image size
**Instruction gotchas**: See [instruction-reference.md](instruction-reference.md) for per-instruction cheatsheet

## Official references

- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/) — all instructions, syntax, escape directives
- [Docker Build best practices](https://docs.docker.com/build/building/best-practices/) — official guidance on layers, caching, multi-stage
- [hadolint rules](https://github.com/hadolint/hadolint#rules) — DL/SC rule reference