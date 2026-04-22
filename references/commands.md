# bridge CLI — Full Command Reference

All commands run as: `bridge <subcommand> ...`

All remote paths are relative to `--root-dir` set when the bridge-session was started.

---

## read

Read a remote file. Output is line-numbered (`     1\tline content`) by default.

```bash
bridge read <path>
bridge read <path> --limit 50          # first 50 lines
bridge read <path> --offset 100        # start from line 100 (0-based)
bridge read <path> --offset 100 --limit 50  # lines 100-149
bridge read <path> --raw               # raw content, no line numbers
bridge read <path> --raw > local.py    # download — safe to redirect (byte-identical)
bridge read <path> --raw --offset 500 --limit 100  # raw slice of lines 501-600
```

**When to use `--offset` / `--limit`:** large files consume context window space on read. Prefer narrow windows — locate with `bridge bash "rg ..."` first, then read just the relevant range (e.g. `--offset 500 --limit 100`). Use `bridge bash "wc -l <path>"` to size up an unknown file.

**When to use `--raw`:**
- Downloading a file to local disk via `> local.py` — redirects the raw content without polluting your context window (the tool result only shows the short command outcome).
- Staging a file for local editing in the pull-stage-edit-push pattern.
- Pulling a byte-accurate slice by pairing with `--offset`/`--limit` — useful when you need an excerpt without line-number prefixes (e.g. piping into a hash check or another tool).

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
1. bridge read <remote_path> --raw > /tmp/<filename>   # pull raw content
2. Edit tool → /tmp/<filename>                         # apply targeted diffs locally
3. bridge write <remote_path> --file /tmp/<filename>   # push back
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
bridge --max-output 5000000 bash "<cmd>"      # raise output cap (global flag, default: 1MB)
```

Output is capped at **1MB** by default. Scope `rg`/`fd` queries (use `-g`, `--type`, `-e`, or a subdirectory path) to stay under the limit, or raise the cap with `--max-output`.

---

## status

Check if the bridge session is alive. Run this before any other command.

```bash
bridge status
```

Returns "Bridge session is active." or exits with an error.

---

## Global Flags

Flags on the `bridge` command itself, placed **before** the subcommand. All optional.

```bash
bridge --session <name> <subcommand> ...     # target a named session (default: 'default')
bridge --timeout <N> <subcommand> ...        # subprocess timeout in seconds (default: 120)
bridge --max-output <N> <subcommand> ...     # max output bytes for bash (default: 1000000)
```

`--timeout` and `--max-output` only affect `bash` (the only subcommand that spawns a remote subprocess). `read`, `write`, and `status` ignore them.

Behavior on expiry:
- `bash` returns exit code `-1` with stderr `command timed out after Ns`

Flags compose: `bridge --session myproj --timeout 600 --max-output 5000000 bash "long_cmd"`.

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
