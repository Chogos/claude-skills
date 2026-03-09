# Twelve-Factor App Cheatsheet

Backend-specific guidance for each factor. Based on [12factor.net](https://12factor.net/).

## I. Codebase

One codebase tracked in version control, many deploys.

- One repo per service. Shared code goes in libraries, not copy-paste.
- Every environment (dev, staging, production) deploys from the same codebase — different configs, same code.
- If two services need different release cycles, they belong in separate repos.
- Monorepo is acceptable if build tooling supports per-service builds and deploys.
- Tag or SHA-pin every production deploy for traceability.

## II. Dependencies

Explicitly declare and isolate dependencies.

- Use a lockfile: `package-lock.json`, `go.sum`, `Cargo.lock`, `poetry.lock`, `requirements.txt` with pinned versions.
- Never rely on system-level packages being pre-installed. Declare everything.
- Vendor or cache dependencies in CI to avoid relying on registry availability during builds.
- Pin transitive dependencies — a lockfile handles this. Commit the lockfile.
- Run dependency vulnerability scanning (`trivy`, `snyk`, `dependabot`) in CI. Block merges on high/critical CVEs.

## III. Config

Store config in the environment.

- Read all configuration from environment variables. No config files in the image.
- Validate every config value at startup. Fail fast with a clear error message.
- Map env vars to a typed config struct — one place to see all configuration.
- Secrets (DB passwords, API keys) go in a secret manager (Vault, AWS Secrets Manager), not plain env vars. Inject them at runtime.
- Use sensible defaults for non-secret values: `PORT=8080`, `LOG_LEVEL=info`.

## IV. Backing Services

Treat backing services as attached resources.

- Database, cache, queue, email service, object storage — all accessed via URL/connection string from config.
- Swapping a local PostgreSQL for RDS requires only a config change, no code change.
- Health checks must verify backing service connectivity (`/health/ready`).
- Use connection pooling with timeouts for every backing service.
- Circuit breakers protect against failing backing services cascading into your service.

## V. Build, Release, Run

Strictly separate build, release, and run stages.

- **Build**: compile code, install dependencies, produce an artifact (binary, container image). No environment-specific config.
- **Release**: combine build artifact + environment config. Tag every release with a unique ID (git SHA, semver, timestamp).
- **Run**: execute the release in the target environment.
- Never patch running code. Build a new release and deploy it.
- Store build artifacts in a registry (container registry, artifact repo). Every deploy pulls a specific, immutable artifact.

## VI. Processes

Execute the app as one or more stateless processes.

- Store no state in-process: no in-memory sessions, no local file caches, no sticky sessions.
- All persistent state goes in a backing service (database, Redis, object storage).
- Any data that must survive a process restart lives in a database or cache.
- Processes can be killed and restarted at any time without data loss.
- Use Redis or a database for session storage, not local memory. Horizontal scaling requires shared state.

## VII. Port Binding

Export services via port binding.

- The service binds to a port and listens for requests. No separate web server required.
- Expose HTTP, gRPC, or other protocols by binding directly — no Apache/Nginx in front (reverse proxy is an infrastructure concern, not an application one).
- Read the port from config: `PORT=8080`. Don't hardcode.
- One service per port. Side processes (metrics, admin) can bind additional ports (`METRICS_PORT=9090`).
- The service is fully self-contained — start it, it listens, it serves.

## VIII. Concurrency

Scale out via the process model.

- Scale horizontally by running more instances, not by threading a single giant process.
- Different process types handle different workloads: web processes for HTTP, worker processes for background jobs, scheduler processes for cron.
- Each process type scales independently. 10 web processes + 3 workers, adjusted by load.
- Never rely on running exactly N instances. The service must work with 1 instance or 100.
- Use the orchestrator (Kubernetes, ECS) for process management — not supervisor scripts inside the container.

## IX. Disposability

Maximize robustness with fast startup and graceful shutdown.

- Start fast — seconds, not minutes. Precompute or cache-warm asynchronously after the server starts listening.
- Handle SIGTERM gracefully: stop accepting new work, finish in-flight requests (with a timeout), close connections, exit.
- Workers must handle job interruption: use idempotent jobs so interrupted work can be retried safely.
- Crash-only design: if the process can die at any moment, recovery is just "start a new one."
- Set readiness probes to prevent traffic before the process is ready. Set startup probes to give slow-starting services time to initialize.

## X. Dev/Prod Parity

Keep development, staging, and production as similar as possible.

- Same backing services: if production uses PostgreSQL, development uses PostgreSQL — not SQLite.
- Same container image across environments. Only config changes between dev/staging/prod.
- Deploy frequently — small diffs between dev and prod reduce the "works on my machine" gap.
- Use Docker Compose or similar to run the full dependency stack locally (DB, cache, queue).
- Run the same CI checks locally that run in the pipeline. No environment-specific test suites.

## XI. Logs

Treat logs as event streams.

- Write all logs to stdout. Never write to log files from application code.
- The platform (Docker, Kubernetes, CloudWatch) collects and routes stdout to a log aggregator.
- Use structured logging (JSON) with standard fields: `timestamp`, `level`, `message`, `service`, `request_id`.
- Log level is configuration, not code: set via `LOG_LEVEL` env var, change without redeploying.
- Never log secrets, tokens, PII, or full request/response bodies containing sensitive data.

## XII. Admin Processes

Run admin/management tasks as one-off processes.

- Database migrations, data backups, console access, data fixes — run as one-off processes in the same environment with the same codebase and config.
- Use `kubectl exec`, `ecs run-task`, or equivalent. Don't SSH into production servers.
- Ship admin scripts in the same repo and image as the application. They share the same dependencies and ORM.
- One-off processes must be idempotent — safe to run multiple times if interrupted.
- Audit-log every admin action. No undocumented manual changes to production data.