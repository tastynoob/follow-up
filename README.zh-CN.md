# 自动跟进

用于让 Codex 在长时间 shell 命令结束后自动收到后续提示。

## 安装

```bash
./scripts/install-codex
```

安装时会注册一个常驻 Codex Stop hook。安装后需要重启 Codex。

## 使用

```bash
FOLLOW_UP="${CODEX_HOME:-$HOME/.codex}/skills/follow-up/scripts/follow-up"
"$FOLLOW_UP" --timeout 3600 "检查输出并继续" || exit $?
./longtime-run; rc=$?
"$FOLLOW_UP" --finished --exit-code "$rc"; exit "$rc"
```

在真实命令和 `--finished` 之间使用 `;`，不要使用 `&&`，这样前一个命令失败时也会记录结果。非零退出码会附加默认失败处理提示，让 Codex 检查输出并恢复。

日常使用只会在当前工作区根目录写入 `.follow-up/`，不会修改 Codex hook 配置。
通知按 session 隔离；`follow-up` 需要 Codex 的 session id，拿不到就直接失败。Stop hook 只等待它在当前 Stop 事件中 claim 到的新事件，然后每 10 秒检查一次，直到 `--finished` 写入结果。反馈后事件会被清空。

`--timeout` 是命令预期最长运行时间，超过后 hook 会先让 Codex 检查命令是在正常推进还是卡死。managed hook 进程超时默认是 `7200` 秒，`--timeout` 必须小于它。若判断命令仍正常运行，直接再次停止，本事件会继续等待下一轮超时。

启动这条组合命令后，agent 必须立刻结束本轮回复。不要轮询正在运行的 shell session；Stop hook 才是轮询机制。
