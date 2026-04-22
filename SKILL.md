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
| `bridge read <path>` | Read a remote file (supports `--offset` / `--limit`; line-numbered output; `--raw` for redirectable plain content) |
| `bridge write <path> --file <local>` | Upload a local file to remote |
| `bridge bash "<cmd>"` | Run a shell command on the remote (use `rg` / `fd` for search) |
| `bridge status` | Check session is alive |

For full flags and examples, see [references/commands.md](references/commands.md).

## Reading Large Files

Remote reads land in your context window — a multi-thousand-line file consumed whole wastes context and crowds out reasoning space. Treat `bridge read` like the native Read tool:

- **Locate first, then read narrowly.** Use `bridge bash "rg ..."` to find the relevant file and line range, then `bridge read <path> --offset N --limit M` to pull just that window.
- **Default to `--limit`** when you don't yet know the file's size. `bridge bash "wc -l <path>"` is a cheap pre-check.
- **Page through, don't slurp.** For exploration, read in chunks (e.g. `--limit 200`) and advance `--offset` as needed rather than re-reading the full file.
- **Combine `--raw` with `--offset`/`--limit`** when you need a byte-accurate slice (no line-number prefixes) — e.g. `bridge read <path> --raw --offset 500 --limit 100 > /tmp/slice.txt`. Without `--offset`/`--limit`, `--raw` returns the full file verbatim.

Full reads are fine for small files (under ~500 lines) and for the pull-stage-edit-push pattern below, where you need the complete content to write back.

## Downloading a File (Avoiding Context Pollution)

To copy a remote file to local disk **without** putting its content into your context window, redirect `bridge read --raw` to a local file:

```bash
bridge read <remote-path> --raw > /tmp/local-copy.py
```

The `--raw` flag strips line numbers so the output is usable. The shell redirect (`>`) captures stdout to disk, so the tool result only shows the short command outcome — not the file body. Use this for any download where you don't need to inspect the content inline.

## Editing Remote Files (Recommended Pattern)

For any non-trivial edit, use a local temp file as a staging area:

1. **Pull** — `bridge read <path> --raw > /tmp/<filename>` (raw content, no line numbers, doesn't fill context)
2. **Edit locally** — Apply changes with the Edit tool (normal incremental diffs)
3. **Push back** — `bridge write <path> --file /tmp/<filename>`

Only use `bridge write` with generated content for brand-new files.

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
- For searching/listing remote files, use `bridge bash "rg ..."` and `bridge bash "fd ..."`. Prefer `rg` over `grep` and `fd` over `find` — they're faster, respect `.gitignore`, and produce smaller output (important given the 1MB `bash` cap).
- **Check availability once per session:** `bridge bash "command -v rg fd"`. If either is missing on the remote, fall back to `grep -rn` / `find` and scope the query tightly (subdirectory, `--include`, `-name`).
- All paths are **relative to `--root-dir`** set when the session was started.

## Key Gotchas

- `bridge read` output includes **line numbers** (`     1\tline content`). Use `bridge read <path> --raw` for raw content (safe to redirect to a file).
- **Text files only** — content passes through JSON; binary files will be mangled.
- `bridge bash` output is capped at **1MB** by default; scope `rg`/`fd` queries (use `-g`, `--type`, `-e`, or a subdirectory path) to avoid hitting the limit, or raise it with `bridge --max-output N bash "..."`.
- **120s subprocess timeout** on `bash` — override with `bridge --timeout N bash "..."` (flag goes before the subcommand, e.g. `bridge --timeout 600 bash "long_cmd"`).
