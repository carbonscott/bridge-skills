# BRIDGE.md — per-project session anchor

A `BRIDGE.md` lives in the **local project root** and records how that project
maps to a bridge session: the remote **host**, the absolute remote `--root-dir`
**path**, and the **session** name. It also carries a "reuse-first,
create-if-missing" recipe so a fresh agent can reconnect without re-deriving any
of it.

Produce one **only when asked to**. Fill in the placeholders below, keep the
reuse-first `status` check before the create command, and make sure the
`bridge` / `bridge-session` invocations match those documented in
[commands.md](commands.md).

## Template

```markdown
# BRIDGE — <project> remote session

Work on the <project> codebase on remote host **<host>** via the
`<session>` bridge session.

- **Host:** <host>
- **Path:** <absolute remote --root-dir path>
- **Session:** <session>

Reuse first — check it's alive; only create if missing:

```bash
bridge --session <session> status   # if active, you're done
```

If not active, create it:

```bash
bridge-session start --name <session> -- \
  ssh <host> uv run ~/bridge-server \
  --root-dir <absolute remote --root-dir path>
```
```

## Worked example

A filled-in `BRIDGE.md` for a project named `deploy-opencode` on host
`sdfiana025`:

```markdown
# BRIDGE — deploy-opencode remote session

Work on the deploy-opencode codebase on remote host **sdfiana025** via the
`deploy-opencode` bridge session.

- **Host:** sdfiana025
- **Path:** /sdf/data/lcls/ds/prj/prjdat21/results/cwang31/deploy-opencode
- **Session:** deploy-opencode

Reuse first — check it's alive; only create if missing:

```bash
bridge --session deploy-opencode status   # if active, you're done
```

If not active, create it:

```bash
bridge-session start --name deploy-opencode -- \
  ssh sdfiana025 uv run ~/bridge-server \
  --root-dir /sdf/data/lcls/ds/prj/prjdat21/results/cwang31/deploy-opencode
```
```
