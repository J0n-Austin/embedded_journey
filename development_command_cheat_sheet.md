# C/C++ Development Command Cheat Sheet

Quick reference for AddressSanitizer, Valgrind, Linux, Git, and `jq` on Ubuntu.

## 1. AddressSanitizer (ASan) & UBSan

Detect buffer overflows, use-after-free, memory errors, and undefined behavior.

| Command | Purpose |
|---|---|
| `clang -g -O1 -fsanitize=address main.c -o app` | Build with ASan |
| `clang -g -O1 -fsanitize=address,undefined main.c -o app` | Build with ASan + UBSan |
| `./app` | Run instrumented application |
| `ASAN_OPTIONS=detect_leaks=1 ./app` | Check for leaks (supported Linux builds) |
| `ASAN_OPTIONS=halt_on_error=1 ./app` | Stop on first reported ASan error |

Useful build flags: `-g` (debug symbols), `-O0` (no optimization), `-O1` (light optimization), `-Wall -Wextra -Wpedantic` (warnings), `-fno-omit-frame-pointer` (better stack traces).

```bash
clang -g -O1 -Wall -Wextra -Wpedantic \
  -fsanitize=address,undefined \
  -fno-omit-frame-pointer main.c -o app
```

## 2. Valgrind

Diagnose leaks, invalid accesses, and use of uninitialized memory.

| Command | Purpose |
|---|---|
| `sudo apt install valgrind` | Install |
| `gcc -g -O0 main.c -o app` | Make an ordinary debug build |
| `valgrind ./app` | Run basic Memcheck |
| `valgrind --leak-check=full ./app` | Detailed leak reports |
| `valgrind --track-origins=yes ./app` | Track origins of uninitialized values |
| `valgrind --show-leak-kinds=all ./app` | Display all leak categories |
| `valgrind --error-exitcode=1 ./app` | Nonzero exit status on Valgrind-detected errors |

```bash
valgrind --leak-check=full \
  --show-leak-kinds=all \
  --track-origins=yes ./app
```

**Tip:** Use a separate, non-ASan build for Valgrind.

## 3. Linux

### Navigation and files

| Command | Purpose |
|---|---|
| `pwd` | Show current directory |
| `ls -lah` | Detailed file listing |
| `cd /path/to/dir` | Change directory |
| `mkdir -p project/src` | Create nested directories |
| `cp -r source/ dest/` | Copy directory recursively |
| `mv old new` | Move or rename |
| `rm -i file` | Delete with confirmation |
| `find . -name '*.c'` | Find C files |
| `grep -Rn 'text' .` | Search content recursively |
| `head -n 20 file` | Show first 20 lines |
| `tail -f logfile` | Follow a growing log |

### Processes, networking, and permissions

| Command | Purpose |
|---|---|
| `ps aux` | List processes |
| `top` | Monitor processes |
| `kill PID` | Request process termination |
| `chmod +x script.sh` | Mark file executable |
| `sudo command` | Run with elevated privileges |
| `ip addr` | List interfaces and addresses |
| `ip route` | Show routing table |
| `ss -tulnp` | Show listening TCP/UDP sockets |
| `ping -c 4 HOST` | Send four ICMP probes |
| `ssh user@host` | Connect via SSH |

### Builds and debugging

```bash
# Compile multiple C files
gcc -g -Wall -Wextra main.c utils.c -o app

# CMake configure/build
cmake -S . -B build
cmake --build build

# Debug / identify / inspect dependencies
gdb ./app
file ./app
ldd ./app
```

**Security note:** Avoid `ldd` on untrusted executables; inspect their ELF metadata using tools such as `readelf -d` instead.

## 4. Git and GitHub CLI

### Repository workflow

| Command | Purpose |
|---|---|
| `git init` | Initialize repository |
| `git clone URL` | Clone repository |
| `git status` | Inspect changes |
| `git add file.c` | Stage one file |
| `git add .` | Stage changes under current directory |
| `git commit -m 'message'` | Create commit |
| `git push -u origin main` | First push and track upstream |
| `git pull` | Fetch and integrate remote changes |

### Branches and history

| Command | Purpose |
|---|---|
| `git branch` | List local branches |
| `git switch -c feature` | Create and enter branch |
| `git switch main` | Switch to main |
| `git merge feature` | Merge into current branch |
| `git log --oneline -10` | Show recent commits |
| `git diff` | Show unstaged changes |
| `git diff --staged` | Show staged changes |
| `git restore file.c` | Discard unstaged changes (destructive) |
| `git restore --staged file.c` | Unstage file |
| `git stash` | Stash working changes |
| `git stash pop` | Reapply latest stash |

```bash
# GitHub CLI
gh auth login
gh repo create my-project --private --source=. --push
gh repo view --web
gh pr create
```

## 5. jq — JSON Inspection and Filtering

`jq` reads JSON from files or standard input. **Use single quotes around filters in Bash** to prevent shell interpretation.

```bash
sudo apt install jq
jq '.' data.json                  # Pretty-print JSON
jq -c '.' data.json               # Compact JSON (one result per line)
jq -r '.name' data.json           # Print raw strings (no JSON quotes)
jq 'keys' data.json               # List top-level object keys
jq 'type' data.json               # Show JSON value type
jq '.items | length' data.json    # Count items in array
```

### Select fields and filter arrays

| Command | Purpose |
|---|---|
| `jq '.name' data.json` | Read a field |
| `jq '.network.domain // "unknown"' data.json` | Fallback when null/missing |
| `jq '.items[]' data.json` | Iterate over array entries |
| `jq '.items[] | {id, name}' data.json` | Extract chosen fields |
| `jq '.items[] | select(.status == "active")' data.json` | Filter by exact value |
| `jq '.items[] | select(.score > 50)' data.json` | Filter numeric values |
| `jq '[.items[] | .name]' data.json` | Collect values as an array |
| `jq '[.items[].name] | unique' data.json` | Deduplicate and sort values |

### Logs, large JSON, and resilient filters

```bash
# JSON Lines (one independent JSON object per line):
jq -c 'select(.event == "mal_lead")' events.jsonl

# Print only selected fields from matching objects:
jq -c 'select(.event == "mal_lead") | {timestamp, event, payload}' events.jsonl

# Print the entire original object for matching events:
jq -c 'select(.event == "mal_lead")' events.jsonl

# Avoid errors if a record is a number/string rather than an object:
jq -c 'select(type == "object" and .event == "mal_lead")' events.jsonl

# Optional array: iterate even when items is absent/null:
jq '.items[]?' data.json

# Search any string-valued field recursively for a term (case-insensitive):
jq 'select([.. | strings | ascii_downcase | contains("xmrig")] | any)' data.json

# From standard input:
cat data.json | jq '.items[] | .id'
```

**Large-file tip:** For JSONL, `jq -c` processes a stream of records rather than printing one giant formatted result. For a *single huge JSON document*, `jq --stream` can reduce memory usage, but its path/value output requires different filters. Avoid assuming a multi-gigabyte JSON array is JSONL.

## 6. Quick Memory-Debugging Workflow

```bash
# 1. Build and test with ASan + UBSan
clang -g -O1 -Wall -Wextra \
  -fsanitize=address,undefined \
  -fno-omit-frame-pointer main.c -o app
./app

# 2. Rebuild without sanitizers for Valgrind
gcc -g -O0 main.c -o app
valgrind --leak-check=full --track-origins=yes ./app

# 3. Inspect changes and commit once corrected
git diff
git add .
git commit -m 'Fix memory issues'
git push
```

**Rule of thumb:** ASan/UBSan for day-to-day C/C++ testing; Valgrind for additional memory diagnostics; GDB for stepping through execution; `jq` for structured JSON logs and reports.
