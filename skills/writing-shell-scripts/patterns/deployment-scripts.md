# Deployment Script Patterns

## Contents
- Pre-flight checks
- Deploy with rollback
- Health verification
- Lock files
- Blue-green pattern

## Pre-flight checks

Run all checks before any mutation. Fail fast and report all issues at once.

```bash
preflight_checks() {
    local errors=()

    command -v kubectl >/dev/null || errors+=("kubectl not found")
    command -v helm >/dev/null    || errors+=("helm not found")

    [[ -f "$MANIFEST" ]] || errors+=("manifest not found: $MANIFEST")
    [[ -n "$NAMESPACE" ]] || errors+=("NAMESPACE not set")

    kubectl auth can-i create deployments -n "$NAMESPACE" >/dev/null 2>&1 \
        || errors+=("no permission to create deployments in $NAMESPACE")

    if [[ ${#errors[@]} -gt 0 ]]; then
        log_error "pre-flight checks failed:"
        for err in "${errors[@]}"; do
            log_error "  - $err"
        done
        exit 1
    fi
    log_info "pre-flight checks passed"
}
```

## Deploy with rollback

Track the previous state so you can revert on failure.

```bash
deploy() {
    local release="$1"
    local chart="$2"
    local values="$3"
    local namespace="$4"

    # Capture current revision for rollback
    local prev_revision
    prev_revision="$(helm history "$release" -n "$namespace" --max 1 -o json 2>/dev/null \
        | jq -r '.[0].revision // empty')" || true

    log_info "deploying $release from $chart"
    if ! helm upgrade --install "$release" "$chart" \
        -n "$namespace" \
        -f "$values" \
        --wait \
        --timeout 5m; then

        log_error "deploy failed"
        if [[ -n "${prev_revision:-}" ]]; then
            log_warn "rolling back to revision $prev_revision"
            helm rollback "$release" "$prev_revision" -n "$namespace" --wait
            die "rolled back to revision $prev_revision after failed deploy"
        else
            die "deploy failed and no previous revision to rollback to"
        fi
    fi

    log_info "deploy succeeded"
}
```

## Health verification

Poll until healthy or timeout. Exponential backoff avoids hammering the endpoint.

```bash
wait_for_healthy() {
    local url="$1"
    local max_attempts="${2:-30}"
    local interval="${3:-5}"
    local attempt=1

    log_info "waiting for $url to become healthy"
    while [[ "$attempt" -le "$max_attempts" ]]; do
        local status
        status="$(curl -s -o /dev/null -w '%{http_code}' "$url" 2>/dev/null)" || true

        if [[ "$status" == "200" ]]; then
            log_info "healthy after $attempt attempts"
            return 0
        fi

        log_debug "attempt $attempt/$max_attempts: status=$status"
        sleep "$interval"
        ((attempt++))
    done

    die "health check timed out after $((max_attempts * interval))s"
}
```

### Kubernetes-native health check

```bash
wait_for_rollout() {
    local deployment="$1"
    local namespace="$2"
    local timeout="${3:-300s}"

    log_info "waiting for rollout: $deployment"
    if ! kubectl rollout status "deployment/$deployment" \
        -n "$namespace" \
        --timeout="$timeout"; then
        die "rollout timed out for $deployment"
    fi
    log_info "rollout complete: $deployment"
}
```

## Lock files

Prevent concurrent deployments. Use `flock` when available, fall back to mkdir-based locking.

```bash
LOCK_FILE="/tmp/deploy-${RELEASE_NAME}.lock"

acquire_lock() {
    if ! mkdir "$LOCK_FILE" 2>/dev/null; then
        local pid
        pid="$(cat "$LOCK_FILE/pid" 2>/dev/null)" || true
        if [[ -n "$pid" ]] && kill -0 "$pid" 2>/dev/null; then
            die "another deployment is running (pid: $pid)"
        fi
        # Stale lock — remove and retry
        log_warn "removing stale lock"
        rm -rf "$LOCK_FILE"
        mkdir "$LOCK_FILE" || die "failed to acquire lock"
    fi
    echo $$ > "$LOCK_FILE/pid"
    log_debug "acquired lock: $LOCK_FILE"
}

release_lock() {
    rm -rf "$LOCK_FILE"
    log_debug "released lock: $LOCK_FILE"
}

# Add to cleanup trap
cleanup() {
    local exit_code=$?
    release_lock
    exit "$exit_code"
}
trap cleanup EXIT INT TERM
```

## Blue-green pattern

Deploy to inactive slot, verify, then switch traffic.

```bash
blue_green_deploy() {
    local active
    active="$(get_active_slot)"  # returns "blue" or "green"
    local inactive
    [[ "$active" == "blue" ]] && inactive="green" || inactive="blue"

    log_info "active=$active, deploying to $inactive"

    # Deploy to inactive slot
    helm upgrade --install "myapp-${inactive}" ./chart \
        -f "values-${inactive}.yaml" \
        --wait --timeout 5m

    # Verify inactive slot
    wait_for_healthy "http://myapp-${inactive}.internal/healthz"

    # Switch traffic
    log_info "switching traffic: $active -> $inactive"
    kubectl patch service myapp \
        -p "{\"spec\":{\"selector\":{\"slot\":\"${inactive}\"}}}"

    # Verify after switch
    wait_for_healthy "http://myapp.example.com/healthz"
    log_info "blue-green deploy complete, active=$inactive"
}

get_active_slot() {
    kubectl get service myapp -o jsonpath='{.spec.selector.slot}'
}
```
