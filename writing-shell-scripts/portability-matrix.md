# Portability Matrix: GNU vs BSD

Common commands that behave differently on Linux (GNU) and macOS (BSD).

## sed — in-place editing

| GNU (Linux) | BSD (macOS) | Portable |
|-------------|-------------|----------|
| `sed -i 's/a/b/' file` | `sed -i '' 's/a/b/' file` | `sed -i.bak 's/a/b/' file && rm file.bak` |

The difference: GNU `-i` takes an optional suffix, BSD `-i` requires a suffix argument (even if empty string).

## date — formatting and parsing

| GNU (Linux) | BSD (macOS) | Portable |
|-------------|-------------|----------|
| `date -d '2024-01-15' +%s` | `date -j -f '%Y-%m-%d' '2024-01-15' +%s` | Use `python3 -c` or `gdate` |
| `date -d @1705276800` | `date -r 1705276800` | Use `python3 -c` |
| `date -d '+1 hour'` | `date -v +1H` | Use `python3 -c` |

If date manipulation is complex, prefer:
```bash
python3 -c "from datetime import datetime; print(datetime.now().isoformat())"
```

## readlink — resolve symlinks

| GNU (Linux) | BSD (macOS) | Portable |
|-------------|-------------|----------|
| `readlink -f path` | `readlink path` (no `-f`) | See below |

BSD `readlink` doesn't resolve recursively. Portable alternative:
```bash
resolve_path() {
    cd "$(dirname "$1")" && pwd -P
}
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd -P)"
```

Or install `coreutils` on macOS for `greadlink -f`.

## mktemp — temp file creation

| GNU (Linux) | BSD (macOS) |
|-------------|-------------|
| `mktemp /tmp/foo.XXXXXX` | `mktemp /tmp/foo.XXXXXX` |
| `mktemp -d` | `mktemp -d` |
| `mktemp --suffix=.log` | Not supported |

Portable: always use `mktemp` without `--suffix`. If you need a specific extension:
```bash
tmp="$(mktemp)"
mv "$tmp" "${tmp}.log"
```

## grep — Perl regex and options

| GNU (Linux) | BSD (macOS) | Portable |
|-------------|-------------|----------|
| `grep -P '\d+'` | Not supported | `grep -E '[0-9]+'` |
| `grep -oP '(?<=key=)\w+'` | Not supported | `grep -oE 'key=[a-zA-Z0-9_]+' \| cut -d= -f2` |
| `grep --include='*.sh'` | `grep --include='*.sh'` | Same (both support) |

Avoid `-P` (Perl regex). Use `-E` (extended regex) for portability.

## xargs — null-delimited input

| GNU (Linux) | BSD (macOS) |
|-------------|-------------|
| `xargs -r` (no-run-if-empty) | Not supported (default behavior) |
| `xargs -d '\n'` | Not supported |
| `xargs -0` | `xargs -0` |

Portable: always pair `find -print0` with `xargs -0`:
```bash
find . -name "*.sh" -print0 | xargs -0 shellcheck
```

## stat — file info

| GNU (Linux) | BSD (macOS) | Portable |
|-------------|-------------|----------|
| `stat -c '%Y' file` | `stat -f '%m' file` | See below |
| `stat -c '%s' file` | `stat -f '%z' file` | See below |

Portable file size:
```bash
wc -c < file | tr -d ' '
```

Portable modification time:
```bash
python3 -c "import os; print(int(os.path.getmtime('file')))"
```

## cp — archive mode

| GNU (Linux) | BSD (macOS) |
|-------------|-------------|
| `cp --archive` | `cp -a` (same) |
| `cp --parents` | Not supported |
| `cp -t dir/` | Not supported |

## General portability rules

1. **Check before using GNU-specific flags**: `man command` differs between platforms
2. **Use `command -v`** to check tool availability, not `which`
3. **Install GNU coreutils on macOS** for consistency: `brew install coreutils` (prefixed with `g`: `gdate`, `gsed`, `greadlink`)
4. **Test on both platforms** in CI — use a matrix build with `ubuntu-latest` and `macos-latest`
5. **When in doubt, use Python/Perl one-liners** for complex text processing — they're consistent everywhere
