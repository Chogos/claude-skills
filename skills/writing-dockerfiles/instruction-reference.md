# Dockerfile Instruction Reference

Per-instruction cheatsheet focused on gotchas and non-obvious behavior.

## Contents

- [FROM](#from) — platform pinning, ARG scope, multi-stage
- [COPY vs ADD](#copy-vs-add) — feature matrix, when to use each
- [RUN](#run) — shell/exec/heredoc forms, set -eux
- [CMD vs ENTRYPOINT](#cmd-vs-entrypoint) — interaction matrix, override behavior
- [ARG](#arg) — scope rules, cache invalidation, visibility
- [ENV](#env) — build vs runtime, ARG override
- [USER](#user) — creation, placement, file ownership
- [WORKDIR](#workdir) — relative stacking, auto-creation
- [HEALTHCHECK](#healthcheck) — options, exit codes, NONE
- [EXPOSE](#expose) — documentation only
- [VOLUME](#volume) — silent discard gotcha
- [SHELL](#shell) — override default shell
- [STOPSIGNAL](#stopsignal) — graceful shutdown signals
- [LABEL](#label) — OCI annotations, formatting rules

## FROM

```dockerfile
FROM --platform=linux/amd64 node:24-alpine AS builder
```

- `--platform` pins the architecture. Required for consistent cross-platform builds.
- `ARG` before `FROM` is the only instruction allowed before `FROM`. Its scope ends at the `FROM` line:

  ```dockerfile
  ARG VERSION=3.14
  FROM python:${VERSION}-slim
  ARG VERSION  # must re-declare to use in this stage (value inherited)
  ```

- Multiple `FROM` instructions create separate build stages. Each starts with a fresh filesystem.

## COPY vs ADD

| Feature | COPY | ADD |
|---------|------|----|
| Local files | Yes | Yes |
| URL download | No | Yes (no checksum) |
| Auto-extract tar | No | Yes (.tar, .gz, .bz2, .xz) |
| `--chown` | Yes | Yes |
| `--chmod` | Yes | Yes |
| `--link` | Yes | Yes |
| `--from=stage` | Yes | No |

Rule: always `COPY`. Use `ADD` only for local tar extraction. For URLs, use `curl`/`wget` in `RUN` (supports checksum verification and cleanup in the same layer).

```dockerfile
# Bad: no checksum, creates a layer with the archive
ADD https://example.com/app.tar.gz /app/

# Good: verify checksum, extract, clean up in one layer
RUN curl -fsSL https://example.com/app.tar.gz -o /tmp/app.tar.gz && \
    echo "sha256sum  /tmp/app.tar.gz" | sha256sum -c && \
    tar -xzf /tmp/app.tar.gz -C /app && \
    rm /tmp/app.tar.gz
```

## RUN

Three forms:

```dockerfile
# Shell form: runs /bin/sh -c "command" — supports variable expansion, pipes
RUN echo "hello $NAME" | tee /app/log

# Exec form: no shell, no variable expansion
RUN ["executable", "param1", "param2"]

# Heredoc form: multiple commands, readable
RUN <<EOF
set -eux
apt-get update
apt-get install -y --no-install-recommends curl
rm -rf /var/lib/apt/lists/*
EOF
```

Gotcha: heredoc lines run independently. Without `set -e`, a failing line doesn't stop execution. Always start heredocs with `set -eux`.

## CMD vs ENTRYPOINT

| Setup | ENTRYPOINT | CMD | `docker run img arg1` runs |
|-------|-----------|-----|---------------------------|
| CMD only | — | `["node", "app.js"]` | `arg1` (CMD replaced) |
| ENTRYPOINT only | `["node"]` | — | `node arg1` |
| Both | `["node", "app.js"]` | `["--port=3000"]` | `node app.js arg1` (CMD replaced) |

- `ENTRYPOINT` sets the executable. `CMD` sets default arguments.
- `docker run img arg1` replaces `CMD` entirely, never `ENTRYPOINT`.
- `docker run --entrypoint sh img` overrides `ENTRYPOINT`.
- Shell form `ENTRYPOINT command` prevents CMD args from being appended — always use exec form.
- Only the last `CMD` and last `ENTRYPOINT` take effect.

## ARG

```dockerfile
ARG NODE_VERSION=22
FROM node:${NODE_VERSION}
ARG BUILD_DATE  # must re-declare after FROM
```

- **Scope**: ARG before `FROM` → only available in `FROM` line. ARG after `FROM` → scoped to that stage only.
- **Cache**: changing an ARG value invalidates the cache for all subsequent instructions.
- **Visibility**: ARG values appear in `docker history`. Never use for secrets.
- **Override**: `docker build --build-arg VERSION=22 .`
- **Undefined**: using an undefined ARG produces an empty string, not an error (unless `set -u` in a RUN).

## ENV

```dockerfile
ENV NODE_ENV=production
ENV PORT=3000
```

- Persists in the final image. Visible in `docker inspect` and to the running container.
- Available during build (after the `ENV` line) and at runtime.
- `ENV` overrides `ARG` of the same name.
- Override at runtime: `docker run -e PORT=8080 img`.
- Prefer `ARG` for build-only values (compilers, version numbers). Use `ENV` only for values needed at runtime.

## USER

```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser:appgroup
```

- Switches user for all subsequent `RUN`, `CMD`, `ENTRYPOINT`, `COPY` (ownership context).
- Must create the user first — `USER` does not create users.
- Files `COPY`'d before `USER` are owned by root unless `--chown` is specified.
- Place `USER` as late as possible — after all `RUN` commands that need root privileges.

## WORKDIR

```dockerfile
WORKDIR /app
```

- Creates the directory if it doesn't exist.
- Relative paths stack: `WORKDIR /a` then `WORKDIR b` → `/a/b`.
- Always use absolute paths to avoid confusion.
- Applies to `RUN`, `CMD`, `ENTRYPOINT`, `COPY`, `ADD`.

## HEALTHCHECK

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD ["curl", "-f", "http://localhost:8080/healthz"]
```

- Only the last `HEALTHCHECK` takes effect.
- `HEALTHCHECK NONE` disables a health check inherited from the parent image.
- Options: `--interval` (between checks), `--timeout` (per check), `--start-period` (grace period after start), `--retries` (consecutive failures before unhealthy).
- Exit codes: 0 = healthy, 1 = unhealthy.

## EXPOSE

```dockerfile
EXPOSE 8080/tcp
EXPOSE 9090/udp
```

- Documentation only. Does **not** publish ports.
- `docker run -p 8080:8080` is still required to map ports.
- Convention: always include to document which ports the container listens on.

## VOLUME

```dockerfile
VOLUME /data
```

- Creates an anonymous volume mount point.
- Gotcha: any `RUN` after `VOLUME /data` that modifies `/data` is **silently discarded**.
- Prefer explicit `-v` mounts at `docker run` instead of `VOLUME` in the Dockerfile.
- Multiple volumes: `VOLUME /data /logs /tmp`.

## SHELL

```dockerfile
SHELL ["/bin/bash", "-c"]
```

- Overrides the default shell (`/bin/sh -c`) for shell-form `RUN`, `CMD`, `ENTRYPOINT`.
- Useful for Windows containers: `SHELL ["powershell", "-Command"]`.
- On Linux, rarely needed — prefer exec form instead.

## STOPSIGNAL

```dockerfile
STOPSIGNAL SIGQUIT
```

- Sets the signal sent on `docker stop`. Default: `SIGTERM`.
- `SIGQUIT` for nginx graceful shutdown (finishes in-flight requests).
- `SIGINT` for applications that handle Ctrl-C.

## LABEL

Metadata-only — no layer cost. Labels from parent images are inherited; child labels override parent labels with the same key.

```dockerfile
# OCI image-spec annotations (https://github.com/opencontainers/image-spec/blob/main/annotations.md)
LABEL org.opencontainers.image.title="My Service" \
      org.opencontainers.image.description="Short human-readable description" \
      org.opencontainers.image.version="1.2.3" \
      org.opencontainers.image.revision="abc123def" \
      org.opencontainers.image.created="2026-02-23T12:00:00Z" \
      org.opencontainers.image.authors="team@example.com" \
      org.opencontainers.image.url="https://example.com/myapp" \
      org.opencontainers.image.documentation="https://docs.example.com" \
      org.opencontainers.image.source="https://github.com/org/repo" \
      org.opencontainers.image.vendor="Example Inc." \
      org.opencontainers.image.licenses="Apache-2.0" \
      org.opencontainers.image.base.name="node:24-alpine" \
      org.opencontainers.image.base.digest="sha256:abc123..."
```

| Key | Format / notes |
| --- | --- |
| `created` | RFC 3339 — `date -u +%Y-%m-%dT%H:%M:%SZ` |
| `licenses` | [SPDX expression](https://spdx.org/licenses/) — `Apache-2.0`, `MIT OR GPL-2.0` |
| `base.name` | Fully-qualified ref of the **final** base image only — not multi-stage intermediaries |
| `base.digest` | Digest of the immediate base; pair with `base.name` |
| `revision` | SCM commit SHA or tag |

Rules from the spec:

- Keys must be unique; values must be strings (empty string is valid).
- `org.opencontainers` prefix is reserved — never use it for custom keys.
- Custom keys: use reverse-domain notation (`com.example.mykey`).
- Consumers must not error on unknown annotation keys.

Inspect labels: `docker inspect --format='{{json .Config.Labels}}' img`.
