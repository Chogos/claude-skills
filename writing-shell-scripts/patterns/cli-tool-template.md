# CLI Tool Template

Production-ready script template with argument parsing, logging, config, cleanup, and error handling.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

# --- Constants ---
readonly SCRIPT_NAME="$(basename "${BASH_SOURCE[0]}")"
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly VERSION="1.0.0"

# --- Defaults ---
VERBOSE=0
DRY_RUN=0
CONFIG_FILE=""
LOG_FILE=""

# --- Logging ---
log_info()  { printf '[INFO]  %s\n' "$*"; }
log_warn()  { printf '[WARN]  %s\n' "$*" >&2; }
log_error() { printf '[ERROR] %s\n' "$*" >&2; }
log_debug() { [[ "$VERBOSE" -eq 1 ]] && printf '[DEBUG] %s\n' "$*"; }
die()       { log_error "$@"; exit 1; }

# --- Cleanup ---
TEMP_DIR=""
cleanup() {
    local exit_code=$?
    if [[ -n "$TEMP_DIR" && -d "$TEMP_DIR" ]]; then
        rm -rf "$TEMP_DIR"
        log_debug "cleaned up temp dir: $TEMP_DIR"
    fi
    exit "$exit_code"
}
trap cleanup EXIT INT TERM

# --- Usage ---
usage() {
    cat <<EOF
Usage: $SCRIPT_NAME [OPTIONS] <target>

Description of what this tool does.

Arguments:
  target              The target to operate on

Options:
  -c, --config FILE   Path to config file
  -l, --log FILE      Log output to file
  -n, --dry-run       Show what would be done without doing it
  -v, --verbose       Enable debug output
  -h, --help          Show this help
  --version           Show version

Examples:
  $SCRIPT_NAME my-target
  $SCRIPT_NAME --config prod.conf --verbose my-target
  $SCRIPT_NAME --dry-run my-target
EOF
}

# --- Argument parsing (supports both short and long options) ---
parse_args() {
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -c|--config)
                [[ $# -ge 2 ]] || die "missing value for $1"
                CONFIG_FILE="$2"
                shift 2
                ;;
            -l|--log)
                [[ $# -ge 2 ]] || die "missing value for $1"
                LOG_FILE="$2"
                shift 2
                ;;
            -n|--dry-run)
                DRY_RUN=1
                shift
                ;;
            -v|--verbose)
                VERBOSE=1
                shift
                ;;
            -h|--help)
                usage
                exit 0
                ;;
            --version)
                printf '%s %s\n' "$SCRIPT_NAME" "$VERSION"
                exit 0
                ;;
            --)
                shift
                break
                ;;
            -*)
                die "unknown option: $1 (use --help for usage)"
                ;;
            *)
                break
                ;;
        esac
    done

    # Remaining positional args
    [[ $# -ge 1 ]] || { usage >&2; die "missing required argument: target"; }
    TARGET="$1"
    shift
    readonly TARGET
}

# --- Config loading ---
load_config() {
    if [[ -n "$CONFIG_FILE" ]]; then
        [[ -f "$CONFIG_FILE" ]] || die "config file not found: $CONFIG_FILE"
        log_debug "loading config: $CONFIG_FILE"
        # Source config (only key=value assignments)
        # shellcheck source=/dev/null
        source "$CONFIG_FILE"
    fi
}

# --- Prerequisite checks ---
check_prerequisites() {
    local missing=()
    for cmd in curl jq; do
        command -v "$cmd" >/dev/null || missing+=("$cmd")
    done
    [[ ${#missing[@]} -eq 0 ]] || die "missing required tools: ${missing[*]}"
}

# --- Main logic ---
run() {
    TEMP_DIR="$(mktemp -d)"
    log_debug "using temp dir: $TEMP_DIR"

    log_info "processing target: $TARGET"

    if [[ "$DRY_RUN" -eq 1 ]]; then
        log_info "[dry-run] would process $TARGET"
        return 0
    fi

    # actual work here
    log_info "done"
}

# --- Entry point ---
main() {
    parse_args "$@"
    load_config
    check_prerequisites

    if [[ -n "$LOG_FILE" ]]; then
        run 2>&1 | tee -a "$LOG_FILE"
    else
        run
    fi
}

main "$@"
```

## Key patterns used

- **`set -Eeuo pipefail`** at the top — non-negotiable
- **`readonly`** for constants set at script start
- **Long + short option parsing** without `getopt` (more portable)
- **`die()` helper** — logs error and exits in one call
- **`trap cleanup EXIT`** — runs on normal exit, Ctrl-C, and `kill`
- **Prerequisite checks** — fail early with a clear message listing all missing tools at once
- **Dry-run support** — `--dry-run` flag skips side effects
- **Temp directory** not temp file — `mktemp -d` gives a workspace; cleanup removes the whole tree
- **`source` for config** — simple but only safe for trusted config files; use `read`-based parsing for untrusted input
