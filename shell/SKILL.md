---
name: shell
description: Shell script development best practices. Use when writing, modifying, or reviewing bash/sh/zsh scripts, .sh files, .bash files, .bashrc, .bash_profile, or shell script templates (.sh.tftpl).
---

# Shell Script Development Best Practices

## Shebang & Shell
- Always start with `#!/usr/bin/env bash` for portability (never `#!/bin/bash`).
- For POSIX compliance, use `#!/bin/sh` when features are basic.
- For zsh-specific scripts, use `#!/usr/bin/env zsh`.

## Error Handling
- **Always** use `set -Eeuo pipefail` at the top after shebang:
  - `-E`: Inherit ERR trap in functions
  - `-e`: Exit on any command failure
  - `-u`: Exit on undefined variables
  - `-o pipefail`: Exit if any pipeline command fails
- Use `|| true` only when explicitly ignoring failures.
- Informative error messages to stderr:
  ```bash
  if ! command_that_might_fail; then
      echo "Error: Command failed - check your configuration" >&2
      exit 1
  fi
  ```

## Variables & Quoting
- **Always** quote variables: `"$variable"`, not `$variable`.
- Use `${variable}` when concatenating: `"${prefix}_${suffix}"`.
- `readonly` for constants: `readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"`.
- `local` for function variables: `local temp_file="/tmp/script.$$"`.
- Avoid global variables; prefer function parameters and return values.

## Functions
- Define before use.
- `local` for all function variables.
- Return meaningful exit codes (0 success, non-zero failure).
- Document with purpose, parameters, return values:
  ```bash
  # Description: Downloads and verifies a file
  # Arguments:
  #   $1 - URL to download from
  #   $2 - destination path
  # Returns: 0 on success, 1 on failure
  download_file() {
      local url="${1}"
      local dest="${2}"
      [[ -n "$url" ]] || { echo "Error: URL required" >&2; return 1; }
      [[ -n "$dest" ]] || { echo "Error: Destination required" >&2; return 1; }
  }
  ```

## Command Substitution
- Use `$()` not backticks: `result="$(command)"`.
- Quote command substitutions: `file_count="$(find . -type f | wc -l)"`.
- Avoid unnecessary subshells.

## File Operations
- Check existence: `[[ -f "$file" ]]` for files, `[[ -d "$dir" ]]` for directories.
- Use `mktemp` for temp files: `temp_file="$(mktemp)"`.
- Always cleanup with trap: `trap 'rm -f "$temp_file"' EXIT`.
- `chmod 600` for sensitive files.

## Loops & Conditionals
- Use `[[ ]]` not `[ ]`.
- Quote variables in conditionals: `[[ "$var" == "value" ]]`.
- Safe file iteration:
  ```bash
  while IFS= read -r -d '' file; do
      echo "$file"
  done < <(find . -name "*.txt" -print0)
  ```

## Arrays
- Declare explicitly: `declare -a my_array=()`.
- Access: `"${array[@]}"` for all, `"${#array[@]}"` for length.
- Associative arrays: `declare -A config=()`.

## Input/Output
- Use `printf` over `echo` for formatted output: `printf '%s\n' "$message"`.
- Errors to stderr: `echo "Error message" >&2`.
- Heredocs for multi-line strings:
  ```bash
  cat << 'EOF' > config.txt
  multi-line content
  EOF
  ```

## Security & Safety
- Validate all user inputs.
- Use `--` to separate options from arguments: `rm -- "$user_file"`.
- Avoid `eval`; prefer arrays or functions.
- Secure permissions for created files.

## Portability
- Avoid GNU-specific options when possible.
- Test macOS vs Linux compatibility.
- Check required tools: `command -v git >/dev/null || { echo "git required" >&2; exit 1; }`.

## Logging
```bash
log_info() { printf '[INFO] %s\n' "$*"; }
log_warn() { printf '[WARN] %s\n' "$*" >&2; }
log_error() { printf '[ERROR] %s\n' "$*" >&2; }
```

## Performance
- Avoid excessive subprocess calls in loops.
- Use built-in string operations over external tools when possible.

## Testing & Validation
- Use `shellcheck` for static analysis.
- Include usage info: `usage() { echo "Usage: $0 [options] arguments"; }`.
- Validate required arguments with helpful error messages.

## Signal Handling
```bash
cleanup() {
    local exit_code=$?
    # cleanup logic
    exit $exit_code
}
trap cleanup EXIT INT TERM
```

## Common Patterns
- Root check: `[[ $EUID -eq 0 ]]`.
- Script directory: `SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"`.
- Option parsing:
  ```bash
  while getopts "hv" opt; do
      case $opt in
          h) usage; exit 0 ;;
          v) verbose=1 ;;
          *) usage; exit 1 ;;
      esac
  done
  ```
