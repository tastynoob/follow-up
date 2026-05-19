# Codex Installation

Install this folder as a Codex skill:

```bash
./scripts/install-codex
```

If `CODEX_HOME` is unset, Codex normally uses `~/.codex`.

Restart Codex after installing the skill so its metadata is loaded.

## Installed Layout

The installer creates one Codex skill location:

```bash
$CODEX_HOME/skills/follow-up/
```

Use the script inside that skill directory as the stable entrypoint:

```bash
$CODEX_HOME/skills/follow-up/scripts/follow-up
```

Do not reference the source folder in skill instructions or agent prompts after installation.

## Runtime Behavior

No permanent hook installation is required. Run the real command with Codex's normal shell engine, then add `follow-up` after it with `;`:

```bash
command args...; "${CODEX_HOME:-$HOME/.codex}/skills/follow-up/scripts/follow-up" --exit-code "$?" "feedback text"
```

Use `;`, not `&&`, so `follow-up` still runs when the command fails.

If the final shell status must remain the original command status, save and restore it:

```bash
command args...; rc=$?; "${CODEX_HOME:-$HOME/.codex}/skills/follow-up/scripts/follow-up" --exit-code "$rc" "feedback text"; exit "$rc"
```

`follow-up` temporarily merges one managed `Stop` hook into:

```bash
$CODEX_HOME/hooks.json
```

The hook entry is identified by the command marker `FOLLOW_UP_MANAGED=1`. The script removes only matching managed entries and preserves unrelated hook groups.

Run state is written under:

```bash
$CODEX_HOME/follow-up/runs/<run-id>/
```

Each run directory contains:

- `request.json`: feedback instruction, cwd, host, and session id when available
- `state.json`: notification state
- `result.json`: recorded exit status
- `delivered.json`: marker written when the hook has sent feedback back to Codex

`follow-up` does not execute the command and does not capture stdout or stderr. The hook prompt tells Codex to use the command output already visible in the Codex transcript.

When `--exit-code` is nonzero, the hook adds a default failure prompt asking Codex to inspect the output, diagnose the failure, and recover or report the blocker.

## Configuration

Environment variables:

- `CODEX_HOME`: Codex configuration home, default `~/.codex`
- `FOLLOW_UP_HOOKS_FILE`: override the hooks file path
- `FOLLOW_UP_STATE_DIR`: override the runtime state directory
- `FOLLOW_UP_HOOK_TIMEOUT`: managed hook timeout in seconds, default `300`
- `FOLLOW_UP_EXIT_CODE`: optional exit code source when `--exit-code` is not used

## Troubleshooting

Show current managed notifications:

```bash
"${CODEX_HOME:-$HOME/.codex}/skills/follow-up/scripts/follow-up" --status
```

Remove the managed hook entry without touching run state:

```bash
"${CODEX_HOME:-$HOME/.codex}/skills/follow-up/scripts/follow-up" --unregister
```

If Codex stops without feedback, check whether `$CODEX_HOME/hooks.json` still contains a managed `Stop` hook and whether the matching run directory is missing `delivered.json`.

## Agent Adapter Contract

Future agent integrations should keep the user-facing suffix stable:

```bash
command args...; follow-up --exit-code "$?" "feedback text"
```

An agent-specific adapter only needs to implement these operations:

1. Record the completed command notification and optional exit code.
2. Register a temporary stop/exit hook that can continue or block the agent with text feedback.
3. Build a feedback prompt from `request.json` and `result.json`.
4. Mark the notification delivered.
5. Unregister the temporary hook when no undelivered notifications remain.

Codex uses `hooks.json` and `{"decision":"block","reason":"..."}` for step 2. Other agents can reuse the same notification layout with a different hook adapter.
