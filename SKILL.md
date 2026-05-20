---
name: follow-up
description: Continue after long Codex shell commands without agent-side polling. Use when a command should run through Codex normally, with a follow-up message armed before the command and marked finished after the command so the installed persistent Stop hook can wait and return the requested instruction.
---

# Follow Up

Use this skill when a command should be executed by Codex's normal shell engine, but Codex should not poll the command after it has been launched.

## Command

Use the installed script at the stable skill path. Arm follow-up before the long command, then mark it finished after the command exits:

```bash
FOLLOW_UP="${CODEX_HOME:-$HOME/.codex}/skills/follow-up/scripts/follow-up"
"$FOLLOW_UP" --timeout 3600 "inspect the output and continue" || exit $?
./longtime-run; rc=$?
"$FOLLOW_UP" --finished --exit-code "$rc"; exit "$rc"
```

Use `;`, not `&&`, between the real command and `--finished`. The `;` separator still records the result when the command fails. Pass the previous status with `--exit-code "$rc"` so a nonzero exit code adds the default failure self-check prompt.

Use `--timeout SECONDS` to set the expected maximum runtime before the Stop hook asks Codex to inspect progress. The managed hook process timeout is `7200` seconds by default; `--timeout` must be lower than that and leaves a small margin for the hook to return feedback. If omitted, the command timeout defaults to `3600` seconds.

One-line form:

```bash
FOLLOW_UP="${CODEX_HOME:-$HOME/.codex}/skills/follow-up/scripts/follow-up"; "$FOLLOW_UP" --timeout 3600 "inspect the output and continue" || exit $?; ./longtime-run; rc=$?; "$FOLLOW_UP" --finished --exit-code "$rc"; exit "$rc"
```

The follow-up event:

1. `follow-up "message"` overwrites the active event in `.follow-up/event.json` with the feedback instruction, current directory, workspace root, host, and required session id.
2. Requires a Codex session id from `CODEX_THREAD_ID` or `CODEX_SESSION_ID`; without it, the command fails closed to avoid cross-session delivery.
3. Leaves `$CODEX_HOME/hooks.json` untouched during normal use.
4. On the next Stop event, the installed persistent hook claims only the fresh active event from the same session and workspace, then checks every 10 seconds until `--finished` records the result or the event timeout expires.
5. `follow-up --finished --exit-code "$rc"` records the exit code after the command completes, including failed commands.
6. If the event timeout expires before `--finished`, the hook returns a timeout prompt asking Codex to inspect whether the command is making progress or stuck. If it is still normal, Codex should stop again so the next Stop hook waits for another timeout interval.
7. The hook sends completion feedback only for the event it claimed, then clears `.follow-up/event.json`; old completed events cannot accumulate.
8. Adds a default failure-handling prompt when `--exit-code` is nonzero.

After launching the command, stop immediately. Do not start an agent-side polling loop. The persistent Stop hook owns completion polling.

## Stop Contract

This skill only works if the agent stops after the shell command has been launched.

When the shell tool returns `Process running with session ID` for a command that already contains both `follow-up "message"` and `follow-up --finished`, the next agent action must be to end the turn and let the Codex Stop hook run.

Forbidden after the command is launched:

- Do not call `write_stdin` or otherwise poll the running shell session.
- Do not run `follow-up --status`.
- Do not inspect `.follow-up/event.json`.
- Do not run `sleep`, `ps`, `tail`, `find`, or any other waiting/checking command.
- Do not explain that you will wait. End the turn so the Stop hook can wait.

The running shell command itself is responsible for calling `follow-up --finished` when the real command exits. The agent must not wait for that point directly.

If the Stop hook returns `# Follow Up Timeout`, inspect progress only enough to decide whether the command is stuck. If it is still making normal progress, end the turn again without creating a new follow-up event; the existing event remains armed for the next Stop hook wait interval.

## Agent Rules

- Use exactly one user-facing command: `${CODEX_HOME:-$HOME/.codex}/skills/follow-up/scripts/follow-up`.
- Use this command only after the skill has been installed and Codex has been restarted, because Codex loads hooks at startup.
- Run `follow-up --timeout SECONDS "message"` before the real command so the Stop hook knows this Stop event should wait. Omit `--timeout` only when the default `3600` seconds is appropriate.
- Put `follow-up --finished` after the real command with `;`, not `&&`, so failed commands still record feedback.
- Save `rc=$?` before invoking `follow-up --finished --exit-code "$rc"` so failure handling is accurate.
- After launching the combined shell command, stop the turn immediately. If the shell tool reports a running session, do not poll it.
- Do not pass the command to `follow-up`; it does not execute commands and does not capture output.
- Do not use `follow-up` to install hooks; `scripts/install-codex` owns persistent hook registration.
- Treat the Stop-hook feedback as the next user instruction. Use the shell output already visible in the Codex transcript.

## Installation Docs

For Codex installation and operational notes, read `references/install-codex.md`.

The agent adapter contract for future Claude or other agent support is included at the end of that reference.
