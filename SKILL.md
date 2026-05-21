---
name: follow-up
description: Continue after long Codex shell commands without agent-side polling. Use before launching a long command when the installed persistent Stop hook should wait for completion and return a follow-up instruction.
---

# Follow Up

Use when Codex should launch a long shell command normally, then stop. The persistent Stop hook waits for the command to finish and returns the follow-up message.

## Command Template

```bash
FOLLOW_UP="${CODEX_HOME:-$HOME/.codex}/skills/follow-up/scripts/follow-up"
"$FOLLOW_UP" --timeout 3600 "inspect the output and continue" || exit $?
./longtime-run; rc=$?
"$FOLLOW_UP" --finished --exit-code "$rc"; exit "$rc"
```

Rules:

- Use the installed script path above; `scripts/install-codex` owns hook installation.
- Arm first with `follow-up --timeout SECONDS "message"`, then run the real command.
- Use `;`, not `&&`, before `--finished` so failures still report back.
- Save `rc=$?` and pass `--exit-code "$rc"`; nonzero exits add a default failure prompt.
- `--timeout` defaults to `3600` seconds and must be lower than the installed hook timeout.
- `follow-up` does not execute commands and does not capture output.

## Stop Contract

After launching the combined command, end the turn immediately. If the shell reports a running session, do not poll it.

Do not call `write_stdin`, run `follow-up --status`, inspect `.follow-up/`, or run waiting/checking commands such as `sleep`, `ps`, or `tail`. The shell command itself must call `follow-up --finished`.

If the hook returns `# Follow Up Timeout`, inspect progress only enough to decide whether the command is stuck. If it is still normal, end the turn again without creating a new follow-up event.

Treat Stop-hook feedback as the next user instruction and use the shell output already visible in the Codex transcript.

For installation and detailed behavior, read `references/install-codex.md`.
