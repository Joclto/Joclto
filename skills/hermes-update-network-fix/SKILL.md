---
name: hermes-update-network-fix
description: Fix hermes update failures when GitHub HTTPS is blocked.
version: 0.1.0
author: "刘飞天 (Joclto), Hermes Agent"
license: MIT
platforms: [windows]
metadata:
  hermes:
    tags: [hermes, update, network, github, ssh, electron-mirror]
    category: software-development
---

# Hermes Update Network Fix Skill

修复网络受限环境（对 github.com 的 TCP 443 HTTPS 流量干扰）导致的 `hermes update` 失败：Git remote 由 HTTPS 切 SSH（22 端口），Electron 下载源切 npmmirror 镜像，一次配置长期免疫。检测步骤幂等只读；修复步骤会改动 git remote 与用户环境变量，执行前先征得用户同意。完整原始教程（含 PowerShell 全量命令与收尾细节）见 `references/original-walkthrough.md`。

## When to Use

- `hermes update` 报 `Network error — cannot reach the remote repository`、`Recv failure: Connection was reset`、`fatal: unable to access 'https://github.com/...'`，或 Electron 阶段报 `TypeError: fetch failed`。
- `ping github.com` 通，但 `curl -I https://github.com` 超时 / 被重置 / 空回复。
- **主动巡检**：会话开始或任何依赖 GitHub 连通的任务前，跑「第 0 步」幂等检测；若 443 被干扰且尚未免疫，向用户报告并提议提前处理。

Don't use for:

- Installer/ZIP 安装（无源码克隆目录）——改走 config.yaml `updates.proxy` 配代理，或手动下载 ZIP 覆盖。
- 用户明确不愿配置 SSH key——同样走 `updates.proxy`。

## Prerequisites

- Windows + 源码克隆安装：`%LOCALAPPDATA%\hermes\hermes-agent\.git` 存在。
- 一枚可用 GitHub SSH key（无则第 1 步生成登记）；npmmirror 可达（第 3 步先验证再切换）。

## Quick Reference

经 `terminal` 工具（Windows 上为 git-bash）按序执行：

```bash
ssh -T git@github.com                                                              # ① SSH 通道
HA="$LOCALAPPDATA/hermes/hermes-agent"
git -C "$HA" remote set-url origin git@github.com:NousResearch/hermes-agent.git   # ② 切 SSH
git -C "$HA" fetch origin                                                          # ② 验证拉取
curl -sS -I -m 8 https://npmmirror.com/mirrors/electron/                           # ③ 镜像可达
export ELECTRON_MIRROR="https://npmmirror.com/mirrors/electron/"                   # ③ 会话内镜像
rm -rf "$LOCALAPPDATA/electron/Cache" "$HA/apps/desktop/.electron-cache"           # ③ 清坏缓存
hermes update                                                                      # ④ 更新
setx ELECTRON_MIRROR "https://npmmirror.com/mirrors/electron/"                     # ⑤ 持久化
```

## Procedure

### 第 0 步 · 检测（幂等只读，先跑这个）

`terminal(command="curl -sS -I -m 8 https://github.com -o /dev/null -w \"%{http_code}\"; echo \" exit=$?\"")`

判定：

- 输出 `200` / `301` → 443 通，本 skill 不适用，另找原因。
- `(28) timeout` / `(52) Empty reply` / `(35) reset` → 443 被干扰，继续。
- 完成标准：得出明确「阻断 / 不阻断」结论后才进入后续步骤。

再核对免疫状态（三项都已就位则报告「已免疫」并结束）：

```bash
test -d "$LOCALAPPDATA/hermes/hermes-agent/.git" && echo "source-clone: yes"
git -C "$LOCALAPPDATA/hermes/hermes-agent" remote -v        # fetch 行应为 git@github.com:...
powershell -NoProfile -Command '[Environment]::GetEnvironmentVariable("ELECTRON_MIRROR","User")'
```

**必须测 github.com 本身**：干扰可以是按主机分化的（实测案例：github.com 443 被掐，同网 api.github.com 却返回 200）——其它 GitHub 域名通不代表 github.com 通。

### 第 1 步 · 验证 SSH 通道

`terminal`: `ssh -T git@github.com` → 期望 `Hi <user>! You've successfully authenticated...`

- `Permission denied (publickey)` → 生成并登记 key：`ssh-keygen -t ed25519 -C "邮箱"`，把 `~/.ssh/id_ed25519.pub` 内容加入 github.com/settings/keys 后重试。
- 连接超时 / refused → 22 端口也不通，直接跳「兜底 B」（SSH over 443）。
- 完成标准：看到 `Hi <用户名>!` 才进第 2 步。

### 第 2 步 · remote 切 SSH 并验证 fetch

```bash
git -C "$LOCALAPPDATA/hermes/hermes-agent" remote set-url origin git@github.com:NousResearch/hermes-agent.git
git -C "$LOCALAPPDATA/hermes/hermes-agent" fetch origin
```

- fetch 出现 `From github.com ...` 及 `[new tag]` 行即成功。remote 带 `[blob:none]`（部分克隆）属正常特性，`set-url` 不破坏它。
- 仍 `Permission denied` → 显式指定 key：`GIT_SSH_COMMAND="ssh -i ~/.ssh/id_ed25519" git -C "$LOCALAPPDATA/hermes/hermes-agent" fetch origin`。

### 第 3 步 · Electron 镜像 + 清缓存

```bash
curl -sS -I -m 8 https://npmmirror.com/mirrors/electron/     # 先确认镜像本身可达
export ELECTRON_MIRROR="https://npmmirror.com/mirrors/electron/"
rm -rf "$LOCALAPPDATA/electron/Cache" "$LOCALAPPDATA/hermes/hermes-agent/apps/desktop/.electron-cache"
```

v0.21.6 起 Hermes 不再自动回退 npmmirror，必须手动设置。失败残留的坏缓存不清掉，重跑必复现失败。

### 第 4 步 · 更新并收尾

`terminal`: `hermes update` → 期望 `✓ Update complete!`，随后自动再跑一轮显示 `Already up to date!` 并重启 gateway。

持久化镜像（新开窗口自动继承）：`setx ELECTRON_MIRROR "https://npmmirror.com/mirrors/electron/"`（PowerShell 等价与收尾检查——dashboard 旧进程、cua-driver 升级——见 `references/original-walkthrough.md` 第六步）。

### 兜底 A · hermes update 仍走 HTTPS 拉取（更新器硬编码 URL）

```bash
git config --global url."git@github.com:NousResearch/hermes-agent.git".insteadOf "https://github.com/NousResearch/hermes-agent.git"
```

重跑 `hermes update`。

### 兜底 B · 22 端口也不通（SSH over 443）

`~/.ssh/config` 追加（PowerShell 自动化版本见 references 附录 A）：

```text
Host github.com
    HostName ssh.github.com
    Port 443
    User git
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 30
```

`ssh -T git@github.com` 验证通过后回第 2 步。

## Pitfalls

- **ping 通 ≠ HTTPS 通**：ICMP 与 TCP 443 的干扰策略不同，判断阻断永远用 curl 测真实协议，别拿 ping 当证据。
- **干扰按主机分化**：github.com 死时 api.github.com / raw.githubusercontent.com 可能照常（2026-11-14 实测）；反向同理，检测必须打 github.com 本身。CDN 镜像（如 jsdelivr）可能同网被 reset——换任何镜像前先 curl 验证其可达。
- **update 是分阶段流水线**（git 拉取 → Python 依赖 → Node 依赖 → 构建 → 技能同步 → gateway 重启）：git 成功后中途失败，代码其实已更新，修好后续阶段重跑会断点续接，不必回滚重来。
- **hermes update 会重启 gateway**，杀掉运行中的 agent 会话与 cron 调度，之后自动恢复——别在巡逻 / 长任务中间触发。
- `[WinError 5] 拒绝访问: mask.cp314-...pyd` = 文件被运行中进程占用，非致命，重启相关进程后下次更新自愈。
- macOS / Linux 同症可用同思路（remote 切 SSH、ELECTRON_MIRROR 同值、Electron 缓存路径不同）；本 skill 命令按 Windows 实测校验，未在其它平台验证。

## Verification

1. `git -C "$LOCALAPPDATA/hermes/hermes-agent" remote -v` → fetch/push 两行均为 `git@github.com:NousResearch/hermes-agent.git`。
2. `git -C "$LOCALAPPDATA/hermes/hermes-agent" fetch origin` → 出现 `From github.com` 且无报错。
3. `hermes update` 收尾出现 `Update complete!` 或 `Already up to date!`。
4. 新开 shell 运行 `powershell -NoProfile -Command '[Environment]::GetEnvironmentVariable("ELECTRON_MIRROR","User")'` → 返回镜像 URL。
