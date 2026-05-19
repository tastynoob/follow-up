---
name: follow-up
description: Continue after long Codex shell commands without agent-side polling. Use when a command should run through Codex normally, then a small follow-up suffix should register a temporary Stop hook that sends follow-up instructions after the command has completed.
---

# Follow Up

Use this skill when a command should be executed by Codex's normal shell engine, but Codex should not poll the command after it has been launched.

## Command

Use the installed script at the stable skill path:

```bash
./longtime-run; "${CODEX_HOME:-$HOME/.codex}/skills/follow-up/scripts/follow-up" --exit-code "$?" "inspect the output and continue"
```

Use `;`, not `&&`. The `;` separator still runs `follow-up` when the previous command fails. Pass the previous status with `--exit-code "$?"` so a nonzero exit code adds the default failure self-check prompt.

For shells where the final command status must remain the original command status, save it first:

```bash
./longtime-run; rc=$?; "${CODEX_HOME:-$HOME/.codex}/skills/follow-up/scripts/follow-up" --exit-code "$rc" "inspect the output and continue"; exit "$rc"
```

The suffix command:

1. Records the feedback instruction, current directory, session id when available, and optional exit code under `$CODEX_HOME/follow-up/runs/`.
2. Registers one managed Codex `Stop` hook in `$CODEX_HOME/hooks.json`.
3. On the next Stop event, sends a continuation prompt that points Codex back to the shell output already produced by Codex.
4. Adds a default failure-handling prompt when `--exit-code` is nonzero.
5. Marks the notification delivered and unregisters the hook when no undelivered notifications remain.

After the suffix runs, stop normally. Do not start a polling loop. The Stop hook owns feedback delivery.

## Agent Rules

- Use exactly one user-facing command: `${CODEX_HOME:-$HOME/.codex}/skills/follow-up/scripts/follow-up`.
- Put `follow-up` after the real command with `;`, not `&&`, so failed commands still arm the feedback hook.
- Use `--exit-code "$?"` or save `rc=$?` before invoking `follow-up` so failure handling is accurate.
- Do not pass the command to `follow-up`; it does not execute commands and does not capture output.
- Treat the Stop-hook feedback as the next user instruction. Use the shell output already visible in the Codex transcript.

## Installation Docs

For Codex installation and operational notes, read `references/install-codex.md`.

The agent adapter contract for future Claude or other agent support is included at the end of that reference.
