# Optimization Patterns

Cache mounts, BuildKit features, image size reduction, and .dockerignore templates.

## Cache mounts

Persist package manager caches across builds. The cache directory is mounted during `RUN` and never stored in a layer.

Requires `# syntax=docker/dockerfile:1` at the top of the Dockerfile.

```dockerfile
# apt (Debian/Ubuntu)
RUN --mount=type=cache,target=/var/cache/apt \
    --mount=type=cache,target=/var/lib/apt/lists \
    apt-get update && apt-get install -y --no-install-recommends curl

# apk (Alpine)
RUN --mount=type=cache,target=/var/cache/apk \
    apk add --no-cache curl

# pip
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# npm
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# yarn
RUN --mount=type=cache,target=/usr/local/share/.cache/yarn \
    yarn install --frozen-lockfile

# Go modules
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -o /app ./cmd/server

# Maven
RUN --mount=type=cache,target=/root/.m2/repository \
    ./mvnw package -DskipTests

# Gradle
RUN --mount=type=cache,target=/root/.gradle \
    ./gradlew build -x test --no-daemon

# Cargo (Rust)
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/app/target \
    cargo build --release
```

Cache mounts are not invalidated by layer changes — a modified `requirements.txt` still benefits from the cached packages for unchanged deps.

## BuildKit features

### Syntax directive

Always add as the first line:
```dockerfile
# syntax=docker/dockerfile:1
```

Enables all modern features: cache mounts, secret mounts, SSH mounts, heredocs, `COPY --link`.

### COPY --link

Creates an independent layer not dependent on previous layers. Enables parallel layer building.

```dockerfile
# Without --link: layer depends on all previous layers
COPY dist/ /app/dist/

# With --link: independent layer, builds in parallel
COPY --link dist/ /app/dist/
```

Use on all `COPY` instructions unless you depend on files from a previous layer in the same path.

### Bind mounts

Read-only access to files during `RUN` without creating a layer:

```dockerfile
# Mount requirements.txt from build context without COPY
RUN --mount=type=bind,source=requirements.txt,target=/tmp/requirements.txt \
    pip install -r /tmp/requirements.txt
```

### Heredocs

Inline files without creating separate files in the build context:

```dockerfile
COPY <<EOF /app/config.json
{"port": 8080, "debug": false}
EOF
```

Multi-command RUN with proper error handling:
```dockerfile
RUN <<EOF
set -eux
apt-get update
apt-get install -y --no-install-recommends curl git
rm -rf /var/lib/apt/lists/*
EOF
```

`set -eux` in heredocs — each line runs as a separate command, so without `-e` failures are silently ignored.

### Multi-platform builds

```bash
# Build for multiple architectures
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest .

# Build for a specific platform (cross-compile)
docker buildx build --platform linux/arm64 -t myapp:arm64 .
```

In the Dockerfile, use `TARGETPLATFORM`, `TARGETOS`, `TARGETARCH` build args (automatically set by BuildKit):
```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.23 AS build
ARG TARGETOS TARGETARCH
RUN CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o /app
```

## Reducing image size

Techniques ranked by impact:

1. **Multi-stage builds** — don't ship compilers and build tools (~500MB+ savings)
2. **Minimal base image** — alpine ~5MB vs debian ~120MB vs ubuntu ~70MB
3. **`--no-install-recommends`** for apt-get — skips suggested packages
4. **Clean caches in the same RUN** — separate layers retain deleted files
5. **`.dockerignore`** — reduces build context transfer time
6. **Combine related RUN instructions** — fewer layers, smaller image

Bad — 3 layers, cache persists in layer 1:
```dockerfile
RUN apt-get update
RUN apt-get install -y curl git
RUN rm -rf /var/lib/apt/lists/*
```

Good — 1 layer, cache cleaned:
```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl git && \
    rm -rf /var/lib/apt/lists/*
```

Best — cache mount, no cleanup needed:
```dockerfile
RUN --mount=type=cache,target=/var/cache/apt \
    --mount=type=cache,target=/var/lib/apt/lists \
    apt-get update && apt-get install -y --no-install-recommends curl git
```

## .dockerignore patterns

### Universal starter

```
.git
.github
.vscode
.idea
*.md
!README.md
docker-compose*.yml
Dockerfile*
.dockerignore
.env*
LICENSE
Makefile
```

### Node.js additions

```
node_modules
npm-debug.log*
coverage/
.nyc_output/
dist/
```

### Python additions

```
__pycache__/
*.py[cod]
*.egg-info/
.venv/
venv/
.pytest_cache/
.mypy_cache/
htmlcov/
```

### Go additions

```
vendor/
*.test
*.out
bin/
```

### Java additions

```
target/
build/
.gradle/
*.class
*.jar
```

## Layer analysis

### Check build context size

```bash
docker build --progress=plain . 2>&1 | grep "transferring context"
```

Large context (>50MB) usually means missing `.dockerignore` entries.

### Inspect layer sizes

```bash
docker history --no-trunc myapp:latest
```

### Interactive exploration with dive

```bash
dive myapp:latest
```

`dive` shows each layer's contents and highlights wasted space (files added then removed in later layers).

Red flags:
- Layers over 100MB — likely includes dev tools or uncleaned caches
- Duplicate files across layers — copy-on-write overhead
- `.git` directory in any layer — missing `.dockerignore`
- `node_modules` or `__pycache__` in runtime image — missing multi-stage