# Multi-stage Dockerfile Templates

Production-ready templates for common languages. Each uses named stages, dependency caching, minimal runtime images, and non-root users.

## Go

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.23-alpine AS deps
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download

FROM deps AS build
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /app /app
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/app"]
```

Key choices:
- `CGO_ENABLED=0` produces a fully static binary — no libc dependency, runs on scratch/distroless.
- `-ldflags="-s -w"` strips debug info and DWARF symbols (~30% smaller binary).
- `distroless/static` has no shell, no package manager — smallest attack surface.
- `go.mod`/`go.sum` copied before source for dependency cache.

## Node.js

```dockerfile
# syntax=docker/dockerfile:1
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM deps AS build
COPY tsconfig.json ./
COPY src/ src/
RUN npm run build

FROM node:22-alpine AS runtime
RUN apk add --no-cache tini
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist/ dist/
USER node
EXPOSE 3000
ENTRYPOINT ["tini", "--"]
CMD ["node", "dist/server.js"]
```

Key choices:
- `npm ci` (not `npm install`) for deterministic installs from lockfile.
- Separate `npm ci --omit=dev` in runtime — devDependencies never enter the final image.
- `tini` as PID 1 — Node.js doesn't handle signals well as PID 1 (orphaned child processes, no SIGTERM forwarding).
- `USER node` — built into the official Node.js image, no need to create.

### Without TypeScript (plain JS)

```dockerfile
# syntax=docker/dockerfile:1
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

FROM node:22-alpine AS runtime
RUN apk add --no-cache tini
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
USER node
EXPOSE 3000
ENTRYPOINT ["tini", "--"]
CMD ["node", "src/server.js"]
```

## Python

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.13-slim AS deps
WORKDIR /app
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM deps AS build
COPY . .
RUN pip install --no-cache-dir .

FROM python:3.13-slim AS runtime
COPY --from=build /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
WORKDIR /app
COPY --from=build /app/src/ src/
USER nobody
EXPOSE 8000
CMD ["python", "-m", "myapp"]
```

Key choices:
- Virtual env (`/opt/venv`) isolates deps and makes them easy to copy as a single directory.
- `--no-cache-dir` skips pip's download cache — no point caching inside a build layer.
- `slim` over `alpine` — many Python packages require glibc and fail to build on alpine without workarounds.

### With uv (faster alternative to pip)

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.13-slim AS build
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-install-project
COPY . .
RUN uv sync --frozen --no-dev

FROM python:3.13-slim AS runtime
COPY --from=build /app/.venv /app/.venv
ENV PATH="/app/.venv/bin:$PATH"
WORKDIR /app
COPY --from=build /app/src/ src/
USER nobody
EXPOSE 8000
CMD ["python", "-m", "myapp"]
```

## Java

```dockerfile
# syntax=docker/dockerfile:1
FROM eclipse-temurin:21-jdk-alpine AS deps
WORKDIR /app
COPY pom.xml .
COPY .mvn/ .mvn/
COPY mvnw .
RUN chmod +x mvnw && ./mvnw dependency:go-offline -q

FROM deps AS build
COPY src/ src/
RUN ./mvnw package -DskipTests -q

FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser:appgroup
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Key choices:
- `jdk` for build, `jre` for runtime — JDK adds ~200MB of compiler tools.
- `dependency:go-offline` caches all Maven deps before source copy.
- `-DskipTests` in build stage — tests should run in a separate `test` target or CI step.

### With Gradle

```dockerfile
# syntax=docker/dockerfile:1
FROM eclipse-temurin:21-jdk-alpine AS deps
WORKDIR /app
COPY build.gradle.kts settings.gradle.kts gradle.properties ./
COPY gradle/ gradle/
COPY gradlew .
RUN chmod +x gradlew && ./gradlew dependencies --no-daemon -q

FROM deps AS build
COPY src/ src/
RUN ./gradlew build -x test --no-daemon -q

FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser:appgroup
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## Rust

```dockerfile
# syntax=docker/dockerfile:1
FROM rust:1.84-alpine AS build
RUN apk add --no-cache musl-dev
WORKDIR /app

# Cache dependency build with dummy main
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs && \
    cargo build --release && \
    rm -rf src

# Build real application
COPY src/ src/
RUN touch src/main.rs && cargo build --release && \
    strip target/release/myapp

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /app/target/release/myapp /myapp
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/myapp"]
```

Key choices:
- Dummy `main.rs` trick — `cargo build` compiles and caches all dependencies. Replacing `src/` and touching `main.rs` triggers only the application recompile.
- `musl-dev` for static linking on alpine. Binary runs on distroless/scratch without libc.
- `strip` removes symbol tables (~50-70% smaller binary).
- `touch src/main.rs` updates the timestamp so cargo knows to recompile it.