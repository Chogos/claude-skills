# Security Hardening

Distroless builds, secret management, image scanning, and runtime constraints.

## Distroless builds

Google's distroless images contain only the application runtime — no shell, no package manager, no OS utilities.

| Variant | Use case | Base size |
|---------|----------|-----------|
| `static-debian12` | Go, Rust, any static binary | ~2MB |
| `base-debian12` | Dynamically linked binaries (needs glibc) | ~20MB |
| `cc-debian12` | Needs libstdc++ (C++ apps) | ~25MB |
| `java21-debian12` | Java applications | ~90MB |
| `python3-debian12` | Python applications | ~50MB |
| `nodejs22-debian12` | Node.js applications | ~60MB |

All variants available with `:nonroot` tag (runs as uid 65532) and `:debug` tag (includes busybox shell for troubleshooting).

```dockerfile
# Production
FROM gcr.io/distroless/static-debian12:nonroot

# Debugging (has a shell for troubleshooting)
# FROM gcr.io/distroless/static-debian12:debug
```

Size comparison for a Go binary:

| Base | Image size |
|------|-----------|
| `ubuntu:24.04` | ~780MB |
| `debian:12-slim` | ~200MB |
| `alpine:3.21` | ~50MB |
| `distroless/static` | ~15MB |
| `scratch` | ~binary size |

`scratch` is even smaller but has no CA certificates, timezone data, or `/etc/passwd`. Distroless includes these.

## BuildKit secrets

Mount secrets at build time without baking them into image layers.

```dockerfile
# syntax=docker/dockerfile:1
FROM node:22-alpine

# Secret available only during this RUN, never stored in a layer
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) && \
    echo "//registry.npmjs.org/:_authToken=${NPM_TOKEN}" > .npmrc && \
    npm ci && \
    rm -f .npmrc
```

Build command:
```bash
docker build --secret id=npm_token,src=.npm_token .
```

### SSH for private repos

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.23-alpine AS build
RUN apk add --no-cache openssh-client git
RUN mkdir -p -m 0600 ~/.ssh && ssh-keyscan github.com >> ~/.ssh/known_hosts
RUN --mount=type=ssh go mod download
```

Build command:
```bash
docker build --ssh default .
```

### Patterns that leak secrets

Bad — all visible in `docker history` or image layers:

```dockerfile
# Baked into image metadata
ARG DB_PASSWORD=secret123
ENV API_KEY=abc

# Baked into a layer (even if deleted in a later layer)
COPY .env /app/.env
RUN rm /app/.env  # still recoverable from earlier layer
```

Good — secrets never touch image layers:

```dockerfile
# BuildKit secret mount
RUN --mount=type=secret,id=db_password \
    DB_PASS=$(cat /run/secrets/db_password) ./migrate.sh

# Or pass at runtime
# docker run -e DB_PASSWORD="$DB_PASSWORD" myapp
```

## Non-root users

### Alpine

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser:appgroup
```

### Debian/Ubuntu

```dockerfile
RUN groupadd --system appgroup && \
    useradd --system --gid appgroup --no-create-home appuser
USER appuser:appgroup
```

### File ownership

Files copied before `USER` are owned by root. Use `--chown` to fix:

```dockerfile
COPY --chown=appuser:appgroup ./dist /app/dist
```

Or set permissions explicitly:
```dockerfile
RUN chown -R appuser:appgroup /app && \
    chmod -R 555 /app
```

`555` = read+execute for everyone, no write. Appropriate for application binaries. Use `444` for config files (read-only).

## Image scanning

### Trivy

```bash
# Scan image for CVEs
trivy image --severity HIGH,CRITICAL myapp:latest

# Scan Dockerfile for misconfigurations
trivy config Dockerfile

# Fail CI on findings
trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:latest
```

### Grype

```bash
grype myapp:latest
grype myapp:latest --fail-on high
```

### GitHub Actions integration

```yaml
- name: Build image
  run: docker build -t myapp:${{ github.sha }} .

- name: Scan image
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    severity: HIGH,CRITICAL
    exit-code: 1
```

### Accepting known CVEs

Create `.trivyignore` with justification:
```
# CVE in base image, no fix available, low exploitability
CVE-2024-12345

# Fixed in next base image release, tracking in JIRA-456
CVE-2024-67890
```

## Runtime security

Apply these flags at `docker run` or in compose/orchestration config:

```bash
docker run \
  --read-only \                          # read-only root filesystem
  --tmpfs /tmp \                         # writable /tmp only
  --cap-drop=ALL \                       # drop all Linux capabilities
  --security-opt=no-new-privileges \     # prevent privilege escalation
  --memory=256m \                        # memory limit
  --cpus=0.5 \                           # CPU limit
  --pids-limit=100 \                     # prevent fork bombs
  myapp:latest
```

Key flags:
- `--read-only`: prevents writes to the container filesystem. Mount `--tmpfs` for directories that need writes.
- `--cap-drop=ALL`: removes all Linux capabilities. Add back specific ones with `--cap-add` only if needed.
- `--security-opt=no-new-privileges`: blocks setuid/setgid binaries from gaining permissions.
- `--network=bridge` (default): use custom bridge networks for isolation. Never `--network=host` in production.