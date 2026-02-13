---
name: writing-shell-scripts
description: Shell script development best practices. Use when writing, modifying, or reviewing bash/sh/zsh scripts, .sh files, .bash files, .bashrc, .bash_profile, or shell script templates (.sh.tftpl).
---

# Shell Script Development Best Practices

## Shebang

- `#!/usr/bin/env bash` for portability. `#!/bin/sh` for POSIX-only scripts.
- Executables: no extension or `.sh`. Libraries: always `.sh`, not executable.

## Error Handling

**Always** use `set -Eeuo pipefail` after the shebang:
- `-E`: ERR trap inherited in functions
- `-e`: exit on failure
- `-u`: exit on undefined variables
- `-o pipefail`: exit if any pipeline command fails

Use `|| true` only when explicitly ignoring failures.

## Formatting

- 2-space indent, never tabs. Max 80 characters per line.
- Place `; then` and `; do` on the same line as `if`/`for`/`while`.
- Split long pipelines across lines with `|` at the start of continuation lines.

## Variables

- **Always** quote: `"$variable"`, `"${prefix}_${suffix}"`.
- `readonly` or `declare -r` for constants. `UPPER_CASE` for constants/environment variables.
- `local` for function variables. `lower_case` for local/function-scoped variables.
- Avoid globals; prefer function parameters and return values.

## Functions

- Define before use. `local` for all variables. Return meaningful exit codes.
- `lower_case` with underscores. Use `::` for library namespacing (`mylib::helper`).
- Document purpose, globals, arguments, outputs, and return values:
  ```bash
  # Downloads and verifies a file
  # Globals: DOWNLOAD_TIMEOUT
  # Args: $1=URL, $2=destination
  # Outputs: writes to stdout on success
  # Returns: 0 on success, 1 on failure
  download_file() {
      local url="${1:?URL required}"
      local dest="${2:?destination required}"
  }
  ```
- Use a `main` function in scripts with multiple functions; call it last:
  ```bash
  main() {
      parse_args "$@"
      run
  }
  main "$@"
  ```

## Safe file iteration

```bash
while IFS= read -r -d '' file; do
    process "$file"
done < <(find . -name "*.txt" -print0)
```

## Cleanup with trap

```bash
cleanup() {
    local exit_code=$?
    rm -rf "$TEMP_DIR"
    exit "$exit_code"
}
trap cleanup EXIT INT TERM
```

## Logging

```bash
log_info()  { printf '[INFO]  %s\n' "$*"; }
log_warn()  { printf '[WARN]  %s\n' "$*" >&2; }
log_error() { printf '[ERROR] %s\n' "$*" >&2; }
die()       { log_error "$@"; exit 1; }
```

## Input/Output

- `printf '%s\n' "$msg"` over `echo` for formatted output.
- Errors to stderr: `>&2`.
- Use heredocs for multi-line strings.

## Arithmetic

- Use `(( ))` or `$(( ))` for math. Never `let`, `$[ ]`, or `expr`.

## Security

- Validate all user inputs. Use `--` to separate options from arguments.
- Avoid `eval`. Secure permissions (`chmod 600`) for sensitive files.
- SUID/SGID forbidden on shell scripts — use `sudo` instead.

## Portability

- Check required tools: `command -v git >/dev/null || die "git required"`.
- Avoid GNU-specific options when possible (see [portability-matrix.md](portability-matrix.md)).

## New script workflow

```
- [ ] Start with shebang and set -Eeuo pipefail
- [ ] Add argument parsing with usage/help
- [ ] Add prerequisite checks (required tools)
- [ ] Implement main logic with cleanup trap
- [ ] Add logging and error handling
- [ ] Run validation loop (below)
```

## Validation loop

1. `shellcheck script.sh` — fix all warnings (SC1xxx=syntax, SC2xxx=best practices, SC3xxx=POSIX)
2. `bash -n script.sh` — fix syntax errors
3. Test with valid input — verify expected behavior
4. Test with invalid input — verify error messages and exit codes
5. Test on macOS if targeting both platforms
6. Repeat until shellcheck is clean and all cases pass

## Deep-dive references

**CLI tool template**: See [patterns/cli-tool-template.md](patterns/cli-tool-template.md) for a production-ready script template
**Deployment scripts**: See [patterns/deployment-scripts.md](patterns/deployment-scripts.md) for deploy, rollback, health check patterns
**Testing**: See [patterns/testing-patterns.md](patterns/testing-patterns.md) for bats framework, mocking, CI integration
**GNU vs BSD**: See [portability-matrix.md](portability-matrix.md) for cross-platform command differences

## Official references

- [Google Shell Style Guide](https://google.github.io/styleguide/shellguide.html) — naming, formatting, features, best practices
- [ShellCheck Wiki](https://www.shellcheck.net/wiki/) — rule-by-rule documentation (SC1xxx syntax, SC2xxx quality, SC3xxx POSIX)
