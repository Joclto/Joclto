# 原始修复教程（存档）

> 出处：2026-11-14 Hermes Agent 更新失败，用户与 DeepSeek 逐步排查后由 DeepSeek 整理的完整教程原文。
> 本文件为存档保留；SKILL.md 是其执行版。冲突时以实际命令输出为准。

## Hermes Agent 更新失败修复教程（Windows / 网络受限环境）

适用症状：`hermes update` 报 `Network error — cannot reach the remote repository`、`Recv failure: Connection was reset`、`fatal: unable to access 'https://github.com/...'`，或卡在 Electron 下载报 `TypeError: fetch failed`。

根因：本地网络对 github.com 的 TCP 443 端口 HTTPS 流量被干扰（ping 正常但 curl 超时），而 Hermes Agent 的源码仓库走 HTTPS、桌面构建从 GitHub Releases 下载 Electron 二进制，两处都会被阻断。

解决思路：把 Git 远程地址从 HTTPS 换成 SSH（走 22 端口），把 Electron 下载源换成国内镜像（npmmirror）。

### 前置条件确认

在开始前，先确认你的环境属于下面这种情况（这是本教程的适用场景）：

```powershell
# 1. hermes 命令存在
Get-Command hermes | Select-Object Source
# 典型输出: C:\Users\<用户名>\AppData\Local\hermes\bin\hermes.exe

# 2. 安装方式是源码克隆（有 hermes-agent 子目录且含 .git）
Test-Path "$env:LOCALAPPDATA\hermes\hermes-agent\.git"
# 应返回 True

# 3. 确认网络症状：ping 通但 HTTPS 不通
ping github.com
curl.exe -I https://github.com/NousResearch/hermes-agent.git
# ping 正常（有 Reply），curl 报 (28) Failed to connect ... port 443 after ... ms
```

⚠️ 如果 Test-Path 返回 False，说明你是 Installer/ZIP 安装而非源码克隆，本教程的 Git 部分不适用，需要走 config.yaml 里 updates.proxy 配置代理或手动下载 ZIP 覆盖。本文主要针对源码克隆安装。

### 第一步：验证 SSH 通道可用

Hermes 的 SSH 方案依赖本机已有的 GitHub SSH Key。先确认：

```powershell
ssh -T git@github.com
```

期望输出：`Hi <你的GitHub用户名>! You've successfully authenticated, but GitHub does not provide shell access.`

✅ 看到这行 → SSH 可用，继续第二步

❌ 报 Permission denied (publickey) → 先配置 SSH Key：

```powershell
ssh-keygen -t ed25519 -C "你的邮箱"
Get-Content "$env:USERPROFILE\.ssh\id_ed25519.pub"
# 把输出内容添加到 https://github.com/settings/keys
ssh -T git@github.com   # 再测试一次
```

❌ 报 Connection refused 或超时 → 22 端口也不通，需要改用 SSH over 443（见附录 A）

### 第二步：把 Hermes 仓库的 Git 远程从 HTTPS 改为 SSH

这是核心修复步骤。

```powershell
# 进入 Hermes 源码目录
cd "$env:LOCALAPPDATA\hermes\hermes-agent"

# 查看当前 remote（应该是 https://github.com/...）
git remote -v

# 改为 SSH
git remote set-url origin git@github.com:NousResearch/hermes-agent.git

# 确认已生效（应显示 git@github.com:...）
git remote -v
```

验证 SSH 拉取确实能通（关键验证，不要跳过）：

```powershell
git fetch origin
```

期望输出：出现 `From github.com:NousResearch/hermes-agent` 以及若干 `-> origin/main`、`[new tag] ...` 行。

✅ 成功 → 继续第三步

❌ Permission denied (publickey) → 检查 `~/.ssh/config` 是否有针对 github.com 的特殊配置，或显式指定 key：

```powershell
$env:GIT_SSH_COMMAND = "ssh -i $env:USERPROFILE\.ssh\id_ed25519"
git fetch origin
```

❌ Could not resolve hostname / 超时 → 22 端口不通，走附录 A 的 SSH over 443

💡 如果 remote 后面有 `[blob:none]` 标记，这是部分克隆（partial clone），属于正常特性，`git remote set-url` 不会破坏它，不用管。

### 第三步：运行 hermes update（首次会报 Electron 下载失败，属正常）

```powershell
hermes update
```

首次运行大概率会在 Desktop 应用构建阶段报错：

```text
electron-runtime: preparing locked Electron 44.10.2 (win32-x64)...
TypeError: fetch failed
node-deps: install failed; restored the previously installed dependencies
```

这是预期的——代码本身已经通过 SSH 更新成功，卡住的是 Electron 二进制从 GitHub Releases 下载。CLI/TUI/Web UI 通常已经构建完成，只有 Desktop 没完成。

不要慌，继续第四步。

📌 如果首次 hermes update 在 Git 拉取阶段就报错（而不是 Electron 阶段），说明 hermes update 内部硬编码了 HTTPS URL 而没使用 remote 名。执行下面的兜底命令后重跑：

```powershell
git config --global url."git@github.com:NousResearch/hermes-agent.git".insteadOf "https://github.com/NousResearch/hermes-agent.git"
hermes update
```

### 第四步：配置 Electron 国内镜像并重新构建 Desktop

Hermes Agent 从 v0.21.6 起不再自动回退到 npmmirror 镜像，必须手动设置 ELECTRON_MIRROR 环境变量。

4.1 设置镜像（当前会话临时生效，先测试）：

```powershell
$env:ELECTRON_MIRROR = "https://npmmirror.com/mirrors/electron/"
```

4.2 清除可能损坏的 Electron 缓存（之前失败的下载可能留下不完整文件，必须清掉，否则构建脚本会复用坏缓存）：

```powershell
Remove-Item -Path "$env:LOCALAPPDATA\electron\Cache" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path "$env:LOCALAPPDATA\hermes\hermes-agent\apps\desktop\.electron-cache" -Recurse -Force -ErrorAction SilentlyContinue
```

4.3 重新运行更新：

```powershell
hermes update
```

期望看到：

```text
✓ Building desktop packaged app
✓ Packaging the desktop app
✓ Code updated!
✓ Update complete! (v0.23.1+XXXX.gXXXXXXX) [main @ XXXXXXXX]
```

此时会自动再跑一轮 hermes update 完成收尾，最终显示 `Already up to date!` 并重启 gateway，就成功了。

### 第五步：持久化 Electron 镜像（推荐）

第四步的 `$env:` 设置只在当前 PowerShell 窗口有效。持久化到用户环境变量，以后新开的窗口自动继承：

```powershell
[Environment]::SetEnvironmentVariable("ELECTRON_MIRROR", "https://npmmirror.com/mirrors/electron/", "User")
```

验证（新开一个 PowerShell 窗口后运行 `$env:ELECTRON_MIRROR`，应输出 `https://npmmirror.com/mirrors/electron/`）。

### 第六步：收尾检查

6.1 重启仍跑旧代码的 dashboard 进程（如提示 `⚠ These serve/dashboard processes still run pre-update code: pid XXXXX — dashboard (profile default, manual-serve)`）：

```powershell
Stop-Process -Id XXXXX -Force
hermes serve   # 或 hermes dashboard，按你平时用的命令
```

不处理的话下次重启会自动加载新代码。

6.2 升级 cua-driver（可选，需要管理员权限）。如果提示 `Windows cua-driver refresh deferred (autostart registration requires UAC)`，新开一个「以管理员身份运行」的 PowerShell：

```powershell
hermes computer-use install --upgrade
```

看到 `✓ cua-driver ready: X.XX.X` 即完成。若中间出现 `Not granting the current user access to ...` 的提示，这是 Hermes 的安全保护机制，不影响安装，可忽略。

6.3 验证最终版本：

```powershell
cd "$env:LOCALAPPDATA\hermes\hermes-agent"
git log -1 --oneline
git remote -v   # 确认是 git@github.com:...
```

### 附录 A：SSH over 443（当 22 端口也不通时使用）

A.1 创建/编辑 SSH 配置文件：

```powershell
$sshDir = "$env:USERPROFILE\.ssh"
if (!(Test-Path $sshDir)) { New-Item -ItemType Directory $sshDir | Out-Null }

$configPath = "$sshDir\config"
$existing = if (Test-Path $configPath) { Get-Content $configPath -Raw } else { "" }

if ($existing -notmatch "Host github\.com") {
    $block = @"

Host github.com
    HostName ssh.github.com
    Port 443
    User git
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 30
"@
    Add-Content -Path $configPath -Value $block
}
```

A.2 验证：`ssh -T git@github.com` 应返回 `Hi <用户名>! You've successfully authenticated...`，之后回到第二步继续即可。

### 附录 B：一键检查清单（快速照做版）

```powershell
# ① 验证 SSH
ssh -T git@github.com

# ② 切换 remote
cd "$env:LOCALAPPDATA\hermes\hermes-agent"
git remote set-url origin git@github.com:NousResearch/hermes-agent.git
git fetch origin

# ③ 设置 Electron 镜像并清缓存
$env:ELECTRON_MIRROR = "https://npmmirror.com/mirrors/electron/"
Remove-Item -Path "$env:LOCALAPPDATA\electron\Cache" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path "$env:LOCALAPPDATA\hermes\hermes-agent\apps\desktop\.electron-cache" -Recurse -Force -ErrorAction SilentlyContinue

# ④ 运行更新
hermes update

# ⑤ 持久化镜像（成功后再执行）
[Environment]::SetEnvironmentVariable("ELECTRON_MIRROR", "https://npmmirror.com/mirrors/electron/", "User")
```

### 附录 C：故障对照表

| 报错 | 原因 | 处理 |
|---|---|---|
| `Recv failure: Connection was reset` | HTTPS 443 被干扰 | 走第二步（换 SSH remote） |
| `curl: (28) Failed to connect ... port 443` | 同上 | 同上 |
| `Permission denied (publickey)` | SSH Key 未配置或不匹配 | ssh-keygen / 附录 A.1 |
| `Could not resolve hostname` | 22 端口被封 | 附录 A（SSH over 443） |
| `TypeError: fetch failed`（Electron 阶段） | GitHub Releases 下载受阻 | 第四步（ELECTRON_MIRROR） |
| `[WinError 5] 拒绝访问: mask.cp314-...pyd` | 文件被运行中进程占用 | 非致命，重启 dashboard 后下次更新自动消失 |
| `Not granting the current user access...`（cua-driver 安装时） | Hermes 的安全保护机制 | 忽略，安装本身已成功 |
| hermes update 又用 HTTPS 拉取 | 内部硬编码 URL | 第三步中的 insteadOf 兜底命令 |

### 附录 D：本次修复的关键认知（存档备注）

1. **ping 通 ≠ HTTPS 通**。ping 测的是 ICMP，而 HTTPS 走的是 TCP 443，两者被干扰的策略可能不同。判断网络是否被阻断，要用 `curl.exe -I <url>` 测实际协议。
2. Hermes 是「源码克隆 + 部分克隆」安装，`.git` 在 `%LOCALAPPDATA%\hermes\hermes-agent` 子目录，而不是 `%LOCALAPPDATA%\hermes` 根目录。用 `Test-Path "$env:LOCALAPPDATA\hermes\hermes-agent\.git"` 确认。
3. hermes update 分多个阶段：Git 拉取 → Python 依赖 → Node 依赖 → TUI/Web/Desktop 构建 → 技能同步 → 重启 gateway。Git 拉取成功后即使后续阶段失败，代码也已经更新了，修复后续阶段（如 Electron 镜像）后重跑 hermes update 会从断点继续，不需要重来。
4. ELECTRON_MIRROR 从 v0.21.6 起必须手动设置，官方不再自动回退镜像。这是很多老教程失效的原因。
5. SSH 方案一次配置，长期有效。以后所有 hermes update 都会自动走 SSH，不需要每次手动处理。

### 2026-11-14 补充实测（深圳网络）

- github.com:443 报 `curl: (52) Empty reply from server`（同为 443 干扰症状，与 timeout/reset 并列）；
- 同网同刻 api.github.com 返回 200、raw.githubusercontent.com 返回 301 —— 干扰是按主机分化的，检测必须打 github.com 本身；
- cdn.jsdelivr.net 被同网 reset —— 不可依赖单一 CDN 镜像兜底。
