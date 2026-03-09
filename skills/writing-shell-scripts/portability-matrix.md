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

BSD `readlink` on macOS 12.3+ supports `-f`. For older macOS versions:
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

## sort — version sorting and flags

| GNU (Linux) | BSD (macOS) | Portable |
|---|---|---|
| `sort --version-sort` | Not supported | `sort -t. -k1,1n -k2,2n -k3,3n` |
| `sort -h` (human-numeric) | Not supported | Parse units manually |
| `sort -V` | Not supported | Same as `--version-sort` |

## tar — archive flags

| GNU (Linux) | BSD (macOS) |
|---|---|
| `tar --exclude='*.log'` | `tar --exclude '*.log'` (same, but no `=`) |
| `tar -czf out.tar.gz dir/` | `tar -czf out.tar.gz dir/` |
| `tar --wildcards '*.txt'` | Not supported (wildcards work by default) |

Portable: always use short flags. Avoid `--wildcards` and `--transform`.

## find — regex support

| GNU (Linux) | BSD (macOS) | Portable |
|---|---|---|
| `find . -regex '.*\.sh'` | `find . -regex '.*\.sh'` (different default regex type) | `find . -name '*.sh'` |
| `find . -regextype posix-extended -regex ...` | `find -E . -regex ...` | Use `-name`/`-path` when possible |

## cut — delimiter handling

| GNU (Linux) | BSD (macOS) |
|---|---|
| `cut -d$'\t' -f1` | `cut -d'	' -f1` (literal tab) |
| `cut --complement -f2` | `cut -f1,3-` (manual complement) |

Portable: use `awk -F'\t' '{print $1}'` for complex field extraction.

## base64 — encoding/decoding

| GNU (Linux) | BSD (macOS) | Portable |
|-------------|-------------|----------|
| `base64 -d` (decode) | `base64 -D` (decode) | `base64 -d 2>/dev/null \|\| base64 -D` |
| `base64 -w0` (no wrap) | `base64` (no wrap by default) | `base64 \| tr -d '\n'` |

## realpath — resolve full path

| GNU (Linux) | BSD (macOS) | Portable |
|-------------|-------------|----------|
| `realpath path` | Not available (pre-Ventura) | `python3 -c "import os; print(os.path.realpath('path'))"` |

macOS 13+ (Ventura) includes `realpath`. For older macOS, use `readlink -f` (macOS 12.3+) or the `resolve_path` function above.

## timeout — command timeout

| GNU (Linux) | BSD (macOS) | Portable |
|-------------|-------------|----------|
| `timeout 30 cmd` | Not available | `gtimeout 30 cmd` (coreutils) |

Fallback without coreutils:

```bash
( cmd ) & pid=$!
( sleep 30; kill "$pid" 2>/dev/null ) & watcher=$!
wait "$pid" 2>/dev/null; result=$?
kill "$watcher" 2>/dev/null
```

## General portability rules

1. **Check before using GNU-specific flags**: `man command` differs between platforms
2. **Use `command -v`** to check tool availability, not `which`
3. **Install GNU coreutils on macOS** for consistency: `brew install coreutils` (prefixed with `g`: `gdate`, `gsed`, `greadlink`)
4. **Test on both platforms** in CI — use a matrix build with `ubuntu-latest` and `macos-latest`
5. **When in doubt, use Python/Perl one-liners** for complex text processing — they're consistent everywhere
