# QwenAlpha-SOD：Harness 环境下的 GitHub 正确同步方法

版本：v1.1  
日期：2026-10-29  
适用环境：Windows PowerShell 5.1、DSH/Harness 便携 Git、系统 Git 未安装  
唯一允许的远端：`https://github.com/bixenokow830-sudo/QwenAlpha-SOD.git`

---

## 1. 结论

Harness 环境不得直接依赖便携 Git 的默认 `credential.helper=helper-selector`。该助手需要 MSYS `sh.exe`，而当前宿主禁止其创建 signal pipe，最终会出现：

```text
sh.exe: fatal error - couldn't create signal pipe, Win32 error 5
fatal: could not read Username for 'https://github.com': terminal prompts disabled
```

正确做法是分成两段：

1. 把便携 `git.exe` 临时加入当前 PowerShell 的 `PATH`，使用项目的一次性同步脚本和 `-NoPush` 完成 allowlist 筛选、凭据内容检查、复制及本地提交；
2. 直接调用原生 `git-credential-wincred.exe` 读取 Windows 凭据管理器中的现有 GitHub 凭据，在内存中构造临时 HTTP Authorization 头，再用便携 `git.exe` 完成 `fetch`、分支安全检查、`push` 和远端验证。

此方法已经通过以下只读验证：

```text
便携 Git：2.56.0.windows.2
原生 wincred：成功读取 username 和 secret，未输出其值
GitHub ls-remote：成功
git push --dry-run：成功
待推送范围：e5c7fa0..d80cba8 main -> main
```

`push --dry-run` 未修改 GitHub 远端。

---

## 2. 不可违反的同步约束

- 只允许同步到 `https://github.com/bixenokow830-sudo/QwenAlpha-SOD.git`；
- 必须保留 `sync-qwenalpha-artifacts-once.ps1` 的 QwenAlpha allowlist 和凭据扫描；白名单包含根目录 `AGENTS.md`、QwenAlpha-SOD 论文文档及 PDF；
- 不得提交数据集、权重、checkpoint、`runs/`、`tmp/`、缓存、密钥、Token 或第三方论文；
- 不得把失败内容改投其他仓库；
- 不得创建 scheduler、watcher、轮询循环或后台持续同步；
- 不得把 Token 写入脚本、Git config、日志、命令输出或 Markdown；
- 不得使用 `git config --global --unset-all credential.helper` 修改用户全局配置；
- 不得调用 `helper-selector`、shell 形式的凭据助手或 Git hooks；
- 远端分支领先本地时必须停止，不能 force push、rebase 或自动合并；
- 同步失败时保留本地提交，不删除或重写历史。

---

## 3. 一次性执行脚本

将下面完整代码保存到 Harness 当前 PowerShell 会话中执行，或逐段粘贴执行。不要把其中的内存变量输出到终端。

```powershell
$ErrorActionPreference = 'Stop'

$workspace = 'D:\桌面\CSCD\Qwenalpha-SOD'
$syncScript = Join-Path $workspace 'workspace-tools\sync-qwenalpha-artifacts-once.ps1'
$repo = $workspace
$remoteUrl = 'https://github.com/bixenokow830-sudo/QwenAlpha-SOD.git'
$proxy = 'http://127.0.0.1:7890'

# 1. 动态寻找当前 Harness/DSH 会话附带的便携 Git。
# bash-ipc 的会话哈希会变化，因此不得永久硬编码旧哈希目录。
$gitFile = Get-ChildItem -Path 'D:\DSH-home\dsh-ops\bash-ipc\*\cmd\git.exe' -File -ErrorAction SilentlyContinue |
    Sort-Object LastWriteTime -Descending |
    Select-Object -First 1

if (-not $gitFile) {
    throw '未找到 Harness 便携 git.exe；停止同步，不要绕过 allowlist。'
}

$gitExe = $gitFile.FullName
$portableRoot = Split-Path -Parent (Split-Path -Parent $gitExe)
$wincredExe = Join-Path $portableRoot 'ucrt64\libexec\git-core\git-credential-wincred.exe'

if (-not (Test-Path -LiteralPath $wincredExe -PathType Leaf)) {
    throw "未找到原生 wincred helper：$wincredExe"
}

if (-not (Test-Path -LiteralPath $syncScript -PathType Leaf)) {
    throw "同步脚本不存在：$syncScript"
}

# 仅修改当前 PowerShell 进程及其子进程，不修改系统 PATH。
$env:PATH = "$(Split-Path -Parent $gitExe);$env:PATH"

# 2. 先让受控同步脚本完成 allowlist、秘密扫描和本地提交。
# -NoPush 很重要：避免脚本进入当前环境中不可用的默认 credential helper。
& powershell -NoProfile -ExecutionPolicy Bypass -File $syncScript -NoPush
if ($LASTEXITCODE -ne 0) {
    throw "QwenAlpha 本地同步阶段失败，退出码：$LASTEXITCODE"
}

# 3. 再次确认 dedicated checkout 和唯一远端。
if (-not (Test-Path -LiteralPath (Join-Path $repo '.git'))) {
    throw "Git checkout 不存在：$repo"
}

$origin = (& $gitExe -C $repo remote get-url origin | Select-Object -First 1).Trim()
if ($LASTEXITCODE -ne 0 -or $origin -ne $remoteUrl) {
    throw "origin 不符合唯一允许目标：$origin"
}

# 4. 直接调用原生 .exe 读取 Windows Credential Manager。
# 不经过 helper-selector，不经过 sh.exe。
$credentialRequest = @(
    'protocol=https'
    'host=github.com'
    ''
)

$credentialLines = $credentialRequest | & $wincredExe get
if ($LASTEXITCODE -ne 0) {
    throw '原生 wincred 无法读取 GitHub 凭据。'
}

$githubUser = (($credentialLines | Where-Object { $_ -like 'username=*' } | Select-Object -First 1) -replace '^username=', '')
$githubToken = (($credentialLines | Where-Object { $_ -like 'password=*' } | Select-Object -First 1) -replace '^password=', '')

if ([string]::IsNullOrWhiteSpace($githubUser) -or [string]::IsNullOrWhiteSpace($githubToken)) {
    throw 'Windows Credential Manager 中缺少 github.com 的 username 或 secret。'
}

try {
    # 凭据只存在于当前进程内存；不要 Write-Host 这些变量。
    $basicBytes = [Text.Encoding]::UTF8.GetBytes($githubUser + ':' + $githubToken)
    $basicValue = [Convert]::ToBase64String($basicBytes)
    $authHeader = 'Authorization: Basic ' + $basicValue

    # 空 credential.helper 必须排在前面，彻底清除继承的 helper-selector 列表。
    # 显式代理解决 Git 不读取 Windows WinINET 代理的问题。
    # openssl 避免当前便携 Git 的 schannel 握手问题。
    $networkGitArgs = @(
        '-C', $repo,
        '-c', 'credential.helper=',
        '-c', "http.proxy=$proxy",
        '-c', 'http.sslBackend=openssl',
        '-c', "http.extraHeader=$authHeader"
    )

    # 5. 先 fetch，再拒绝任何远端领先情况。
    & $gitExe @networkGitArgs fetch --no-tags origin main
    if ($LASTEXITCODE -ne 0) {
        throw "fetch origin/main 失败，退出码：$LASTEXITCODE"
    }

    $behindText = (& $gitExe -C $repo rev-list --count 'HEAD..origin/main' | Select-Object -First 1).Trim()
    if ($LASTEXITCODE -ne 0) {
        throw '无法计算本地分支落后数量。'
    }

    $behind = [int]$behindText
    if ($behind -gt 0) {
        throw "origin/main 领先本地 $behind 个提交；停止同步，禁止自动合并或 force push。"
    }

    $aheadText = (& $gitExe -C $repo rev-list --count 'origin/main..HEAD' | Select-Object -First 1).Trim()
    if ($LASTEXITCODE -ne 0) {
        throw '无法计算本地待推送提交数量。'
    }

    $ahead = [int]$aheadText
    if ($ahead -eq 0) {
        Write-Host '本地与 origin/main 已一致，无需推送。' -ForegroundColor Green
    }
    else {
        Write-Host "准备推送 $ahead 个本地提交到唯一允许的 origin/main。"

        # --no-verify 防止便携 Git 尝试运行依赖 sh.exe 的 hooks。
        # allowlist 和凭据扫描已经由同步脚本完成。
        & $gitExe @networkGitArgs push --no-verify -u origin main
        if ($LASTEXITCODE -ne 0) {
            throw "push 失败，退出码：$LASTEXITCODE；本地提交保持不变。"
        }
    }

    # 6. 读取远端 main 并与本地 HEAD 对比。
    $localHead = (& $gitExe -C $repo rev-parse HEAD | Select-Object -First 1).Trim()
    $remoteLine = (& $gitExe @networkGitArgs ls-remote --heads origin refs/heads/main | Select-Object -First 1)
    if ($LASTEXITCODE -ne 0 -or -not $remoteLine) {
        throw '推送后无法读取远端 main。'
    }

    $remoteHead = ([string]$remoteLine -split '\s+')[0].Trim()
    if ($localHead -ne $remoteHead) {
        throw "远端校验失败：local=$localHead remote=$remoteHead"
    }

    Write-Host "QwenAlpha-SOD 同步成功：$remoteHead" -ForegroundColor Green
}
finally {
    # 尽最大努力缩短敏感信息在当前 PowerShell 会话内存中的存活时间。
    $githubUser = $null
    $githubToken = $null
    $credentialLines = $null
    $basicBytes = $null
    $basicValue = $null
    $authHeader = $null
    $networkGitArgs = $null
}
```

---

## 4. 推荐的 Harness 执行方式

Harness 应在一次前台 PowerShell 会话中执行第 3 节完整脚本，并等待其返回；不要把同步放到后台，不要轮询，不要建立 watcher。

运行前先确认：

```powershell
Test-Path -LiteralPath 'D:\桌面\CSCD\Qwenalpha-SOD\workspace-tools\sync-qwenalpha-artifacts-once.ps1'
Test-Path -LiteralPath 'D:\桌面\CSCD\Qwenalpha-SOD\.git'
```

运行结束只报告以下非敏感信息：

```text
便携 Git 路径
allowlist candidate/approved 数量
本地提交哈希
推送前 ahead/behind 数量
远端 main 最终哈希
同步成功或错误阶段
```

禁止报告：

```text
username
password/token
Authorization header
Base64 credential
Windows Credential Manager 原始输出
```

---

## 5. 仅验证、不推送

若用户只要求诊断，应在构造 `$networkGitArgs` 后执行：

```powershell
& $gitExe @networkGitArgs push --dry-run --no-verify origin main
```

退出码为 0 且输出类似下列内容，表示认证、代理和 ref 更新路径均可用，但远端没有改变：

```text
To https://github.com/bixenokow830-sudo/QwenAlpha-SOD.git
   <old>..<new>  main -> main
```

不要把 `--dry-run` 结果描述为“已经同步成功”。

---

## 6. 常见失败处理

### 6.1 找不到便携 Git

停止。不要下载未知 Git、不要修改系统软件。报告 `D:\DSH-home\dsh-ops\bash-ipc\*\cmd\git.exe` 无匹配，等待 Codex 或用户处理。

### 6.2 找不到 wincred

停止。不要要求用户在聊天中粘贴 Token。报告预期的 helper 路径和便携 Git 路径。

### 6.3 Windows 凭据不存在

停止。不得输出凭据请求内容，不得把 Token 写进远端 URL。由用户通过受信任方式恢复 Windows Credential Manager 的 `git:https://github.com` 条目。

### 6.4 fetch 失败

检查 `127.0.0.1:7890` 的本地代理是否正在运行。不要自动改用其他代理，也不要循环重试。

### 6.5 origin/main 领先本地

停止并报告 ahead/behind 与提交哈希。禁止 `push --force`、`reset --hard` 或自动 rebase。由用户或 Codex审查差异后决定。

### 6.6 push 失败

保留本地提交。不得重新运行 allowlist 脚本制造重复提交。修复认证或网络后，仅重新执行 fetch、安全检查和 push 部分。

### 6.7 本地无新增文件但存在未推送提交

同步脚本可能显示 `No approved artifact changes to commit.`。这不代表无需 push；继续检查 `origin/main..HEAD`。只要本地 ahead 大于 0，仍应执行安全推送阶段。

---

## 7. 为什么此前的修复没有成功

此前使用：

```powershell
git -c credential.helper=wincred push origin main
```

仍会失败，因为 `credential.helper` 是追加语义。系统级的 `helper-selector` 仍排在 `wincred` 前面，并率先触发崩溃。即使写成：

```powershell
git -c credential.helper= -c credential.helper=wincred ...
```

本环境中通过 Git 间接调用 helper 仍可能进入受限启动路径。已经验证可行的稳定办法是：直接运行原生 `git-credential-wincred.exe`，再用临时 `http.extraHeader` 调用 Git，同时保留空 `credential.helper=` 以禁止后续回退。

代理也必须显式指定：

```text
http.proxy=http://127.0.0.1:7890
http.sslBackend=openssl
```

否则 Git 不会自动继承 Windows WinINET 代理，或会遇到 schannel TLS 握手失败。

---

## 8. 当前已知现场

根据 `GitHub同步失败诊断.md` 以及 2026-10-27 的只读复核：

```text
远端 main：e5c7fa0fe4f49c58ab48f19429a3592c15158f3b
本地 main：d80cba84...
本地领先：1 个提交
本地工作树：诊断时为干净
未推送内容：S1.2 报告提交
```

正式同步前仍需重新执行 fetch 和 ahead/behind 检查，不能仅依赖此快照。

---

## 9. 最短原则

```text
受控脚本 -NoPush
        ↓
原生 wincred.exe 直接读取（不输出）
        ↓
临时 Authorization header + 显式代理
        ↓
fetch → behind=0 → push --no-verify
        ↓
远端 HEAD == 本地 HEAD
```

只有最后一项成立，才能报告 GitHub 同步成功。
