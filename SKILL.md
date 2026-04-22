---
name: bridge-skills
description: "Guide for using the cc-bridge CLI to access remote filesystems over SSH. This skill should be used when a bridge session is running (or CLAUDE.md references bridge commands), the user asks to read, write, search, or edit files on a remote host, or any task requires operating on remote files that native tools (Read, Write, Grep, Glob) cannot reach."
---

# cc-bridge CLI

To access remote files, use the `bridge` CLI instead of native tools (Read, Write, Grep, Glob). The bridge communicates with a `bridge-server` running on the remote host over a persistent SSH connection managed by `bridge-session`.

## Prerequisites

Always verify the session is alive before use:

```bash
bridge status
```

If it fails, start a session first:

```bash
bridge-session start -- ssh <host> 'uv run ~/bridge-server --root-dir /path/to/project'
```

## Command Reference

| Command | Purpose |
|---------|---------|
| `bridge read <path>` | Read a remote file (supports `--offset` / `--limit`; line-numbered output) |
| `bridge write <path> --file <local>` | Upload a local file to remote |
| `bridge bash "<cmd>"` | Run a shell command on the remote |
| `bridge grep <pattern>` | Search file contents (like ripgrep) |
| `bridge glob <pattern>` | Find files by glob pattern |
| `bridge status` | Check session is alive |
| `bridge edit <path>` | Open remote file in `$EDITOR`, auto-syncs on save |

For full flags and examples, see [references/commands.md](references/commands.md).

## Reading Large Files

Remote reads land in your context window — a multi-thousand-line file consumed whole wastes context and crowds out reasoning space. Treat `bridge read` like the native Read tool:

- **Locate first, then read narrowly.** Use `bridge grep` or `bridge bash "rg ..."` to find the relevant file and line range, then `bridge read <path> --offset N --limit M` to pull just that window.
- **Default to `--limit`** when you don't yet know the file's size. `bridge bash "wc -l <path>"` is a cheap pre-check.
- **Page through, don't slurp.** For exploration, read in chunks (e.g. `--limit 200`) and advance `--offset` as needed rather than re-reading the full file.

Full reads are fine for small files (under ~500 lines) and for the pull-stage-edit-push pattern below, where you need the complete content to write back.

## Editing Remote Files (Recommended Pattern)

For any non-trivial edit, use a local temp file as a staging area:

1. **Pull** — `bridge read <path>` to get the current content
2. **Stage locally** — Write it to `/tmp/<filename>` using the Write tool
3. **Edit locally** — Apply changes with the Edit tool (normal incremental diffs)
4. **Push back** — `bridge write <path> --file /tmp/<filename>`

This avoids full rewrites in your head and gives you the full Edit tool experience. Only use `bridge write` with generated content for brand-new files. *(The pull-stage-edit-push pattern intentionally reads the whole file — that's what you're writing back.)*

## Multiple Sessions

Run multiple bridge sessions simultaneously by giving each a name:

```bash
# Start named sessions
bridge-session start --name project-a -- ssh host 'uv run ~/bridge-server --root-dir /path/a'
bridge-session start --name project-b -- ssh host 'uv run ~/bridge-server --root-dir /path/b'

# Target a specific session
bridge --session project-a read <path>
bridge --session project-b bash "pwd"

# List all sessions
bridge-session list

# Stop a named session
bridge-session stop project-a
```

The default session (no `--name` / `--session`) is called `default`. Each session socket lives at `~/.bridge/<name>/session.sock`.

## When to Use Bridge vs Native Tools

- **Remote files** → always use bridge commands. **Local files** → use native Read/Write/Grep/Glob tools.
- **Always prefer `rg` over `grep` and `fd` over `find`** — they're faster, respect `.gitignore`, produce smaller output (important given the 1MB `bash` cap), and have better defaults. This applies whether you use them via `bridge bash "rg ..."` / `bridge bash "fd ..."` or via the built-in wrappers.
- `bridge grep` is a thin wrapper around ripgrep on the server — use it for convenience, but `bridge bash "rg ..."` is equivalent and gives you full `rg` flags when you need them.
- `bridge glob` handles glob patterns (e.g. `**/*.py`). For anything beyond simple globs (regex names, type filters, gitignore-aware walks), use `bridge bash "fd ..."`.
- **Check availability once per session:** `bridge bash "command -v rg fd"`. If either is missing on the remote, fall back to `grep -rn` / `find` and scope the query tightly (subdirectory, `--include`, `-name`).
- All paths are **relative to `--root-dir`** set when the session was started.

## Key Gotchas

- `bridge read` output includes **line numbers** (`     1\tline content`) — strip them if you need raw content, or use `bridge bash "cat <path>"`
- **Text files only** — content passes through JSON; binary files will be mangled
- `bridge edit` launches `$EDITOR` (default: `vi`) and syncs on each save; editors that fork (e.g. `code` without `--wait`) won't work correctly
- `bridge bash` output is capped at **1MB**; scope `rg`/`fd` queries (use `-g`, `--type`, `-e`, or a subdirectory path) to avoid hitting the limit
- **120s subprocess timeout** — `bash`/`grep`/`glob` fail with `command timed out after 120s` by default. Override with `bridge --timeout N <subcommand>` (flag goes before the subcommand, e.g. `bridge --timeout 600 bash "long_cmd"`).
