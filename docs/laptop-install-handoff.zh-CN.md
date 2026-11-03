# 笔记本 Codex 安装交接

## 给接手 Codex 的任务

请在这台 Windows 笔记本上安装下面指定的 CodexStatusbar 修复版，并启用当前用户登录后自启动。先检查本机环境和已有安装，再完成构建、自检、安装与运行验证。不要改动或重启 Codex，不要修改 Codex 的启动快捷方式，也不要让 Codex 等待状态条启动成功。此任务已包含安装和自启动授权。

- 仓库：https://github.com/zhumj1231/codex-statusbar
- 分支：`codex/fix_usage_timing`
- 必须包含的修复提交：`2711df23d5cabd357716fc38738c6858eeb07952`
- 预期产品版本：`1.0.0-rc3.usage-fix1`
- 修复原理：[usage-timing-fix.md](usage-timing-fix.md)

请从此分支构建，不能使用上游旧版 Release 的 EXE。当前交接交付的是源码，没有发布新的预编译 Release。不要复制另一台机器的会话日志、账号配置或整个 `.codex` 目录。

## 环境与源码

已验证的环境是 Windows x64、.NET SDK 10.0.403。先检查笔记本操作系统和架构；下面的 `win-x64` 命令针对 x64，ARM64 环境需要单独确认运行兼容性，不能声称已验证。最终 EXE 是自包含程序，运行它不需要另外安装 .NET Runtime。

优先使用现有 Git 和 .NET 10 SDK。若缺少 SDK，从 Microsoft 官方渠道安装，或下载官方便携 SDK 并核对发布的校验值；便携安装不需要改系统 PATH。不要使用这台交接来源电脑的绝对路径。选择本机合适且尚不存在的源码目录；如已有克隆，先检查远程和未提交内容，保留已有工作。

```powershell
git clone --branch codex/fix_usage_timing --single-branch https://github.com/zhumj1231/codex-statusbar.git codex-statusbar
if ($LASTEXITCODE -ne 0) { throw 'clone failed' }
Set-Location codex-statusbar
git merge-base --is-ancestor 2711df23d5cabd357716fc38738c6858eeb07952 HEAD
if ($LASTEXITCODE -ne 0) { throw 'required fix commit is missing' }
git status --short
git rev-parse HEAD
dotnet --version
```

## 构建和自检

在源码目录执行；如果使用便携 SDK，把 `dotnet` 换成其完整路径。以下步骤不会启动状态条 UI。

```powershell
$ErrorActionPreference = 'Stop'
dotnet publish src/CodexStatusbar/CodexStatusbar.csproj -c Release -r win-x64 --self-contained true -o dist
if ($LASTEXITCODE -ne 0) { throw 'publish failed' }
$builtExe = (Resolve-Path -LiteralPath 'dist/CodexStatusbar.exe').Path
$fixturesPath = (Resolve-Path -LiteralPath 'fixtures').Path
$testLog = Join-Path $env:TEMP ('codex-statusbar-self-test-' + [guid]::NewGuid() + '.txt')
$testProcess = Start-Process -FilePath $builtExe -ArgumentList ('--self-test "' + $fixturesPath + '"') -WindowStyle Hidden -Wait -PassThru -RedirectStandardOutput $testLog
if ($testProcess.ExitCode -ne 0) { throw "self-test failed; inspect $testLog" }
Get-Content -LiteralPath $testLog -Tail 6
(Get-Item -LiteralPath $builtExe).VersionInfo.ProductVersion
Get-FileHash -LiteralPath $builtExe -Algorithm SHA256
```

本修复提交在来源电脑通过 **342 checks / 0 failures**。笔记本需要实际执行并记录自己的结果，不能把来源电脑的结果当成本机验证。不同 SDK/构建环境可能产生不同的 EXE 哈希；记录本机构建值，并在复制安装后比较两者，不要拿来源电脑哈希硬性判断重构建结果。

## 安装与自启动

推荐安装位置：`%LOCALAPPDATA%\Programs\CodexStatusbar\CodexStatusbar.exe`。
配置位置：`%LOCALAPPDATA%\CodexStatusbar\settings.json`。

1. 如果已有安装，先备份旧 EXE、配置和当前 `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` 下的 `CodexStatusbar` 值。备份使用新目录，不覆盖旧备份。
2. 仅停止 `ExecutablePath` 与目标安装路径完全相同的状态条进程；不能按模糊名称停止 Codex/ChatGPT。确认进程退出后再替换 EXE。没有已有安装时创建安装目录。
3. 复制自检通过的 `dist\CodexStatusbar.exe` 到上述安装路径，保留配置。核对安装文件与本机构建文件的 SHA-256 一致。
4. 从最终安装路径调用程序自己的自启动入口；不要手工改 JSON 与注册表造成状态不一致：

```powershell
$installedExe = Join-Path $env:LOCALAPPDATA 'Programs\CodexStatusbar\CodexStatusbar.exe'
$startupProcess = Start-Process -FilePath $installedExe -ArgumentList '--install-startup' -WindowStyle Hidden -Wait -PassThru
if ($startupProcess.ExitCode -ne 0) { throw 'startup registration failed' }
Get-ItemPropertyValue -LiteralPath 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Run' -Name CodexStatusbar
```

应得到带引号的最终安装路径加 `--background`。配置中 `startWithWindows` 和 `startupConfigured` 应为 `true`。这是当前用户登录后启动，不是系统服务；不需要管理员级 HKLM 注册或计划任务。

5. 当前会话也立即启动。为避免被 Codex 启动的子进程生命周期牵连，可以通过 WMI 创建独立进程。来源电脑已经验证此方式；笔记本仍须检查返回值和实际进程：

```powershell
$launchResult = Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{
    CommandLine = ('"' + $installedExe + '" --background')
    CurrentDirectory = (Split-Path -Parent $installedExe)
}
if ($launchResult.ReturnValue -ne 0) { throw "statusbar launch failed: $($launchResult.ReturnValue)" }
Get-CimInstance Win32_Process -Filter "ProcessId=$($launchResult.ProcessId)" |
    Select-Object ProcessId, ParentProcessId, ExecutablePath, SessionId
```

若本机不允许 WMI 创建进程，保留正确的登录自启动配置，并说明即时启动受限；可让用户从资源管理器启动已安装的 EXE。不要为了状态条修改 Codex 的启动流程，不要用无限重试或高频守护进程兜底。

## 验收、行为和撤销

- 确认安装版本、哈希、自检结果、进程持续存在，以及上述启动项和配置一致。状态条应跟随当前 Codex 窗口；等待当前响应完成后再观察速度。
- 若需要程序级读取验证，可运行 `--probe <本地输出JSON路径> <本机线程ID>`；线程 ID 必须从本机取得。探针可能包含本地会话元数据，留在本机，不要提交到 GitHub。
- 自启动项验证不等于已经验证重启登录。不要为验收擅自关闭用户正在工作的 Codex 或注销系统；在交接结果里明确哪些生命周期操作尚未实际执行。
- 修复后的速度仍是**响应级日志估算**：响应完成才更新；生成中使用平滑值，完成后使用匹配样本的加权平均。缺少计时的响应保留 Token 总量，但排除出速度分子和分母；`~` 表示保留值或不完整覆盖，`--` 表示尚无有效测速。它不是逐 Token 的实时服务器解码速度。
- 这里修复的是同一会话混用日志来源导致的回退；切换会话后显示另一个会话的总量是正常行为。
- 默认不启用持续调试日志，不上传会话数据。状态条是独立辅助进程，其启动失败不能阻止 Codex 正常启动。
- 撤销自启动使用最终安装 EXE 的 `--uninstall-startup`，并检查退出码。若需回滚，停止准确匹配的状态条进程后恢复备份的 EXE/配置/启动设置，仅操作本次安装相关文件。

完成后向用户报告安装位置、实际提交与版本、本机验证结果、自启动状态及未验证项。安装本身无需改源码，也不需要创建空提交。
