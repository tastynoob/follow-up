# Follow Up

Return to Codex after a long shell command finishes.

## Install

```bash
./scripts/install-codex
```

## Use

```bash
./longtime-run; "${CODEX_HOME:-$HOME/.codex}/skills/follow-up/scripts/follow-up" --exit-code "$?" "inspect the output and continue"
```

Use `;`, not `&&`, so `follow-up` also runs after failures. A nonzero exit code adds a default failure prompt for Codex to inspect the output and recover.
