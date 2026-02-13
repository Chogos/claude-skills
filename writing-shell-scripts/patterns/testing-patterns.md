# Shell Script Testing Patterns

## Contents
- Bats framework basics
- Test structure
- Mocking commands
- Testing error paths
- CI integration

## Bats framework basics

[Bats](https://github.com/bats-core/bats-core) (Bash Automated Testing System) is the standard for testing shell scripts.

Install:
```bash
# macOS
brew install bats-core

# or clone
git clone https://github.com/bats-core/bats-core.git
./bats-core/install.sh /usr/local
```

Helper libraries (install alongside):
```bash
git clone https://github.com/bats-core/bats-support.git test/test_helper/bats-support
git clone https://github.com/bats-core/bats-assert.git test/test_helper/bats-assert
git clone https://github.com/bats-core/bats-file.git test/test_helper/bats-file
```

## Test structure

```
project/
├── scripts/
│   └── deploy.sh
└── test/
    ├── test_helper/
    │   ├── bats-support/
    │   ├── bats-assert/
    │   └── common.bash      # shared setup/helpers
    └── deploy.bats
```

### Basic test file

```bash
#!/usr/bin/env bats

# Load helpers
load 'test_helper/bats-support/load'
load 'test_helper/bats-assert/load'
load 'test_helper/common'

setup() {
    # Create temp workspace per test
    TEST_DIR="$(mktemp -d)"
    export TEST_DIR
}

teardown() {
    rm -rf "$TEST_DIR"
}

@test "deploy.sh prints usage with --help" {
    run scripts/deploy.sh --help
    assert_success
    assert_output --partial "Usage:"
}

@test "deploy.sh fails without required arguments" {
    run scripts/deploy.sh
    assert_failure
    assert_output --partial "missing required argument"
}

@test "deploy.sh validates config file exists" {
    run scripts/deploy.sh --config /nonexistent/file target
    assert_failure
    assert_output --partial "config file not found"
}
```

### Common helper file

```bash
# test/test_helper/common.bash

# Source the script's functions without executing main
source_script() {
    local script="$1"
    # Override main to prevent execution
    main() { :; }
    source "$script"
}

# Create a mock config file
create_test_config() {
    cat > "$TEST_DIR/config" <<'EOF'
SERVER_HOST=localhost
SERVER_PORT=8080
EOF
    echo "$TEST_DIR/config"
}
```

## Mocking commands

### Override with functions

```bash
@test "deploy fails when kubectl is missing" {
    # Shadow kubectl with a function that fails
    kubectl() { return 127; }
    export -f kubectl

    run scripts/deploy.sh target
    assert_failure
    assert_output --partial "kubectl not found"
}

@test "deploy calls helm with correct arguments" {
    # Capture helm invocations
    helm() {
        echo "helm $*" >> "$TEST_DIR/helm_calls"
        return 0
    }
    export -f helm

    run scripts/deploy.sh --namespace prod my-release
    assert_success

    # Verify helm was called correctly
    run cat "$TEST_DIR/helm_calls"
    assert_output --partial "upgrade --install my-release"
    assert_output --partial "-n prod"
}
```

### Mock with a script on PATH

For tools that can't be overridden with functions:

```bash
setup() {
    TEST_DIR="$(mktemp -d)"
    MOCK_BIN="$TEST_DIR/bin"
    mkdir -p "$MOCK_BIN"

    # Create mock curl that returns a canned response
    cat > "$MOCK_BIN/curl" <<'MOCK'
#!/usr/bin/env bash
echo '{"status": "healthy"}'
MOCK
    chmod +x "$MOCK_BIN/curl"

    # Prepend mock bin to PATH
    export PATH="$MOCK_BIN:$PATH"
}
```

## Testing error paths

```bash
@test "cleanup runs on script failure" {
    # Create a temp file the script should clean up
    local marker="$TEST_DIR/cleanup_ran"

    # Override cleanup to set a marker
    cleanup() { touch "$marker"; }
    export -f cleanup

    run scripts/deploy.sh --invalid-target
    assert_failure
    assert [ -f "$marker" ]
}

@test "script exits on undefined variable" {
    run bash -c 'set -Eeuo pipefail; echo "$UNDEFINED_VAR"'
    assert_failure
}

@test "pipeline failure is caught" {
    run bash -c 'set -Eeuo pipefail; false | true; echo "should not reach"'
    assert_failure
    refute_output --partial "should not reach"
}
```

## CI integration

### GitHub Actions

```yaml
name: Shell tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install bats
        run: |
          git clone --depth 1 https://github.com/bats-core/bats-core.git
          sudo ./bats-core/install.sh /usr/local
      - name: Install helpers
        run: |
          mkdir -p test/test_helper
          git clone --depth 1 https://github.com/bats-core/bats-support.git test/test_helper/bats-support
          git clone --depth 1 https://github.com/bats-core/bats-assert.git test/test_helper/bats-assert
      - name: Shellcheck
        run: shellcheck scripts/*.sh
      - name: Run tests
        run: bats test/*.bats
```

### TAP output for CI parsers

```bash
bats --formatter tap test/*.bats
```

### Parallel execution

```bash
bats --jobs 4 test/*.bats
```
