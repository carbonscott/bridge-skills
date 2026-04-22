# bridge CLI — Full Command Reference

All commands run as: `bridge <subcommand> ...`

All remote paths are relative to `--root-dir` set when the bridge-session was started.

---

## read

Read a remote file. Output is line-numbered (`     1\tline content`).

```bash
bridge read <path>
bridge read <path> --limit 50          # first 50 lines
bridge read <path> --offset 100        # start from line 100 (0-based)
bridge read <path> --offset 100 --limit 50  # lines 100-149
```

**When to use `--offset` / `--limit`:** large files consume context window space on read. Prefer narrow windows — locate with `bridge grep` or `bridge bash "rg ..."` first, then read just the relevant range (e.g. `--offset 500 --limit 100`). Use `bridge bash "wc -l <path>"` to size up an unknown file.

**To get raw content without line numbers**, use bash instead:
```bash
bridge bash "cat <path>"
```

---

## write

Upload content to a remote file. Overwrites the entire file.

```bash
bridge write <path> --file local.py         # from a local file
echo "content" | bridge write <path>        # from stdin
bridge bash "cat local.py" | bridge write <path>  # pipe through
```

**Recommended edit workflow** — use a local temp file to stage edits incrementally, then push once:

```
1. bridge read <remote_path>               # read remote content
2. Write tool → /tmp/<filename>            # save to local temp file
3. Edit tool → /tmp/<filename>             # apply targeted diffs locally
4. bridge write <remote_path> --file /tmp/<filename>  # push back
```

---

## bash

Run a shell command on the remote host. CWD is the `--root-dir`.

```bash
bridge bash "pwd"
bridge bash "ls -la"
bridge bash "python3 script.py"
bridge bash "command -v rg fd"                # check tool availability (once per session)
bridge bash "rg 'pattern' . -g '*.py' -n"     # always prefer rg over grep
bridge bash "rg 'pattern' src/ --type py"     # rg by file type
bridge bash "fd -e py"                         # always prefer fd over find
bridge bash "fd 'test_.*' tests/"              # fd with regex in subdir
bridge bash "grep -rn 'pattern' src/"         # fallback ONLY if rg unavailable
bridge bash "find . -name '*.py'"             # fallback ONLY if fd unavailable
bridge --timeout 600 bash "<cmd>"             # override timeout (global flag, default: 120s)
```

Output is capped at **1MB**. Scope `rg`/`fd` queries (use `-g`, `--type`, `-e`, or a subdirectory path) to stay under the limit.

---

## grep

Search file contents on the remote host (wraps ripgrep on the server).

`bridge grep` and `bridge bash "rg ..."` hit the same underlying tool — use whichever fits. Reach for `bridge bash "rg ..."` when you need `rg` flags not exposed by `bridge grep` (e.g. `--multiline`, `--json`, `-A`/`-B` asymmetric context).

```bash
bridge grep <pattern>                          # search all files
bridge grep <pattern> --path src/              # search in subdirectory
bridge grep <pattern> --glob '*.py'            # filter by glob
bridge grep <pattern> --type py                # filter by file type
bridge grep <pattern> --context 3              # show 3 lines of context
bridge grep <pattern> --mode content           # show matching lines (default)
bridge grep <pattern> --mode files             # show only file paths
bridge grep <pattern> --mode count             # show match counts per file
bridge --timeout 30 grep <pattern>             # override timeout (global flag, default: 120s)
```

---

## glob

Find files by glob pattern on the remote host.

```bash
bridge glob '**/*.py'                          # find all Python files
bridge glob '*.md' --path docs/                # find markdown files in docs/
bridge glob 'test_*.py' --path tests/          # find test files
bridge --timeout 30 glob '**/*.py'             # override timeout (global flag, default: 120s)
```

---

## status

Check if the bridge session is alive. Run this before any other command.

```bash
bridge status
```

Returns "Bridge session is active." or exits with an error.

---

## edit

Open a remote file in `$EDITOR` locally. Auto-syncs to remote on each save. Cleans up temp file on exit.

```bash
bridge edit <path>
bridge edit <path> --editor nano       # override $EDITOR
```

**How it works:**
1. Pulls raw file content to a local temp file (preserves file extension for syntax highlighting)
2. Opens `$EDITOR` (default: `vi`) on the temp file
3. Background thread polls for saves (mtime change) and syncs to remote
4. On editor exit: final sync + temp file cleanup

**Limitation:** Editors that fork (e.g. `code` without `--wait`, `subl`) will not work — the editor process exits immediately and the session ends before editing begins.

---

## Global Flags

Flags on the `bridge` command itself, placed **before** the subcommand. Both are optional.

```bash
bridge --session <name> <subcommand> ...    # target a named session (default: 'default')
bridge --timeout <N> <subcommand> ...       # subprocess timeout in seconds (default: 120)
```

`--timeout` applies to `bash`, `grep`, and `glob` (the subcommands that spawn subprocesses on the remote). `read`, `write`, `status`, and `edit` are unaffected.

Behavior on expiry:
- `bash` returns exit code `-1` with stderr `command timed out after Ns`
- `grep` / `glob` return an error response

`--session` and `--timeout` compose: `bridge --session myproj --timeout 600 bash "long_cmd"`.

---

## Session Management

```bash
# Start the default session
bridge-session start -- ssh <host> 'uv run ~/bridge-server --root-dir /path'

# Start a named session (multiple sessions simultaneously)
bridge-session start --name myproject -- ssh <host> 'uv run ~/bridge-server --root-dir /path'

# Run in foreground (useful for debugging)
bridge-session start --foreground -- ssh <host> 'uv run ~/bridge-server --root-dir /path'

# Target a named session with any bridge command
bridge --session myproject read <path>
bridge --session myproject bash "pwd"

# List all sessions
bridge-session list

# Stop a session (positional name arg, defaults to 'default')
bridge-session stop
bridge-session stop myproject

# Check status
bridge status
bridge --session myproject status
```

Each session socket lives at `~/.bridge/<name>/session.sock` (e.g. `~/.bridge/default/session.sock`).
