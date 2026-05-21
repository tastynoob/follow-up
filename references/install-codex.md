# Codex Installation

Install this folder as a Codex skill:

```bash
./scripts/install-codex
```

If `CODEX_HOME` is unset, Codex normally uses `~/.codex`.

The installer also registers one persistent managed `Stop` hook in `$CODEX_HOME/hooks.json`.

Restart Codex after installing the skill so its metadata and hook config are loaded.

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

Run the real command with Codex's normal shell engine. Call `follow-up "feedback text"` before launching the command, then call `follow-up --finished` after it exits:

```bash
FOLLOW_UP="${CODEX_HOME:-$HOME/.codex}/skills/follow-up/scripts/follow-up"
"$FOLLOW_UP" --timeout 3600 "feedback text" || exit $?
command args...; rc=$?
"$FOLLOW_UP" --finished --exit-code "$rc"; exit "$rc"
```

Use `;`, not `&&`, between the command and `--finished` so `follow-up` still records failed commands. The final `exit "$rc"` preserves the original command status.

```bash
FOLLOW_UP="${CODEX_HOME:-$HOME/.codex}/skills/follow-up/scripts/follow-up"; "$FOLLOW_UP" --timeout 3600 "feedback text" || exit $?; command args...; rc=$?; "$FOLLOW_UP" --finished --exit-code "$rc"; exit "$rc"
```

After launching this combined command, the agent must stop its turn immediately. If the shell tool reports a running session, do not poll it with `write_stdin`, `follow-up --status`, file inspection, or any other waiting command. The Stop hook waits for `--finished` and returns the feedback.

`--timeout SECONDS` is the expected maximum runtime before the hook asks Codex to inspect progress. It defaults to `3600` seconds and must be lower than the managed hook process timeout, which defaults to `7200` seconds. The script keeps a small safety margin so the hook can return feedback before Codex kills the hook process.

During normal use, `follow-up` does not edit:

```bash
$CODEX_HOME/hooks.json
```

Hook registration is done by the installer. The persistent hook entry is identified by the command marker `FOLLOW_UP_MANAGED=1`. The script removes only matching managed entries and preserves unrelated hook groups when uninstalling the hook.

Run state is written as a single overwriteable event in the current workspace root:

```bash
.follow-up/event.json
```

The event contains:

- feedback instruction, cwd, workspace root, host, and required session id
- event state: `pending`, `claimed`, or `finished`
- timeout gate and timeout count
- claim metadata from the Stop hook that claimed this event
- recorded exit status after `--finished`

`follow-up` does not execute the command and does not capture stdout or stderr. The hook prompt tells Codex to use the command output already visible in the Codex transcript.
The queued notification requires a Codex session id from `CODEX_THREAD_ID` or `CODEX_SESSION_ID`, and the persistent hook only claims an event whose session id and workspace root exactly match the Stop hook input.

The Stop hook does not deliver arbitrary old completed notifications. On each Stop event it claims only the fresh active event that was armed shortly before the Stop event, then polls it every 10 seconds until `--finished` records the result or the event timeout expires. If no fresh event exists, the hook returns immediately. After completion feedback is sent, `.follow-up/event.json` is deleted.

If the Stop hook is interrupted or killed before it finishes, it clears the claim on signal when possible. If it is killed uncleanly, the next Stop event discards the stale claim once the claimed hook process is no longer running, so old feedback is not replayed later.

If the event timeout expires before `--finished`, the hook returns a timeout prompt instead of clearing the event. That prompt asks Codex to inspect whether the command is making normal progress or appears stuck. If it is still progressing normally, Codex should stop again without creating a new follow-up event; the next Stop hook will reclaim the same event and wait another timeout interval.

When `--exit-code` is nonzero, the hook adds a default failure prompt asking Codex to inspect the output, diagnose the failure, and recover or report the blocker.

## Configuration

Environment variables:

- `CODEX_HOME`: Codex configuration home, default `~/.codex`
- `FOLLOW_UP_HOOKS_FILE`: override the hooks file path for install or uninstall
- `FOLLOW_UP_STATE_DIR`: override the runtime state directory; default is `.follow-up` in the current workspace root
- `FOLLOW_UP_HOOK_TIMEOUT`: managed hook process timeout in seconds, default `7200`
- `FOLLOW_UP_COMMAND_TIMEOUT_SECONDS`: default event timeout when `--timeout` is omitted, default `3600`
- `FOLLOW_UP_HOOK_POLL_SECONDS`: polling interval while waiting for `--finished`, default `10`
- `FOLLOW_UP_ARM_MAX_AGE_SECONDS`: maximum age for an unclaimed pending event to be claimed by a Stop hook, default `120`
- `FOLLOW_UP_EXIT_CODE`: optional exit code source when `--exit-code` is not used

## Troubleshooting

Show the current active event:

```bash
"${CODEX_HOME:-$HOME/.codex}/skills/follow-up/scripts/follow-up" --status
```

Remove the managed hook entry without touching run state:

```bash
"${CODEX_HOME:-$HOME/.codex}/skills/follow-up/scripts/install-codex" --uninstall-hook
```

If Codex stops without feedback, check whether `$CODEX_HOME/hooks.json` still contains a managed `Stop` hook and whether `.follow-up/event.json` was claimed before `--finished` recorded the result. An event that stayed unclaimed past `FOLLOW_UP_ARM_MAX_AGE_SECONDS` is cleared so later Stop events do not pick it up accidentally.

## Agent Adapter Contract

Future agent integrations should keep the same arm/finish contract:

```bash
follow-up --timeout 3600 "feedback text" || exit $?
command args...; rc=$?
follow-up --finished --exit-code "$rc"; exit "$rc"
```

An agent-specific adapter only needs to implement these operations:

1. Register a persistent stop/exit hook at install time.
2. Overwrite the active event before the command starts.
3. Have the stop/exit hook claim only the current-session active event and wait for completion.
4. Record the completed command result and optional exit code.
5. Build a feedback prompt from the event.
6. Clear the event after feedback.
7. Leave the persistent hook installed until explicit uninstall.

Codex uses `hooks.json` and `{"decision":"block","reason":"..."}` for step 1. Other agents can reuse the same notification layout with a different hook adapter.
