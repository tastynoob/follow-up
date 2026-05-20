# Follow Up

Return to Codex after a long shell command finishes.

## Install

```bash
./scripts/install-codex
```

Installation registers one persistent Codex Stop hook. Restart Codex after installing.

## Use

```bash
FOLLOW_UP="${CODEX_HOME:-$HOME/.codex}/skills/follow-up/scripts/follow-up"
"$FOLLOW_UP" --timeout 3600 "inspect the output and continue" || exit $?
./longtime-run; rc=$?
"$FOLLOW_UP" --finished --exit-code "$rc"; exit "$rc"
```

Use `;`, not `&&`, between the real command and `--finished` so failures are still recorded. A nonzero exit code adds a default failure prompt for Codex to inspect the output and recover.

Normal use only writes `.follow-up/` in the current workspace root; it does not edit Codex hook config.
Notifications are session-scoped; `follow-up` requires Codex's session id and fails closed without it. The Stop hook only waits for the fresh active event it claims during the current Stop event, then checks every 10 seconds until `--finished` records the result. After feedback, the event is cleared.

`--timeout` is the expected maximum runtime before the hook asks Codex to inspect progress. The managed hook process timeout defaults to `7200` seconds, and `--timeout` must be lower than that. If the command is still progressing normally after a timeout prompt, stop again and the same event will wait for another interval.

After starting the combined command, the agent must stop its turn immediately. Do not poll the running shell session; the Stop hook is the polling mechanism.
