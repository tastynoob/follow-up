# 自动跟进

用于让 Codex 在长时间 shell 命令结束后自动收到后续提示。

## 安装

```bash
./scripts/install-codex
```

## 使用

```bash
./longtime-run; "${CODEX_HOME:-$HOME/.codex}/skills/follow-up/scripts/follow-up" --exit-code "$?" "检查输出并继续"
```

使用 `;`，不要使用 `&&`，这样前一个命令失败时也会执行 `follow-up`。非零退出码会附加默认失败处理提示，让 Codex 检查输出并恢复。
