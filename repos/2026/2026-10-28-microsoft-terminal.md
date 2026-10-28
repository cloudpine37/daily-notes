---
date: 2026-10-28
type: repo
subject: microsoft/terminal
tags: [windows, terminal, cpp, cli]
summary: 微软把新 Windows Terminal 与老控制台 conhost 同仓维护，核心终端能力已抽成可复用组件。
---

# Windows Terminal：一个仓库装下新终端与老控制台

微软把「新的 Windows Terminal」和「老控制台 conhost.exe」的源码放在同一个仓库里。截至 2026-10-27 它仍是活跃项目：约 105k star，MIT 许可，最新 release 是 **v1.25.2733.0**（10 月 2 日发布）。我这次读的是 README.md、vcpkg.json、Directory.Build.props、doc/ORGANIZATION.md 和 src/cascadia/WindowsTerminal/main.cpp。

安装方式 README 给了一堆，最省事的是 winget：`winget install --id Microsoft.WindowsTerminal -e`，或 `choco install microsoft-windows-terminal`。运行门槛是 Windows 10 2004（build 19041）以上。

构建环境要求不低：PowerShell 7+、Windows 11 SDK 10.0.26100.8249 以上、VS 2026 18.6 以上，还要开开发者模式。README 给了两条构建路径——PowerShell 下：

```powershell
Import-Module .\tools\OpenConsole.psm1
Set-MsBuildDevEnvironment
Invoke-OpenConsoleBuild
```

或 Cmd 下 `.\tools\razzle.cmd` 再执行 `bcz`。

依赖写在根目录 vcpkg.json 里，很干净：基础依赖是 fmt 12.1.0 与 ms-gsl 4.2.2；`terminal` 这个 feature 额外引入 jsoncpp 1.9.6、cli11 2.6.1、cmark 0.31.1。

代码组织（doc/ORGANIZATION.md）把「引擎」和「壳」分得很清：`src/cascadia/TerminalCore` 定义核心类 `Terminal`（buffer、色表、VT 解析、输入，但不含任何 UI）；`TerminalControl` 用 UWP-XAML + DX renderer 画字；`TerminalApp` 负责设置、标签页与分栏；`WindowsTerminal` 是真正的 Win32 入口，用 XAML islands 承载；`CascadiaPackage` 负责打包成 .msix。老 conhost 集中在 `src/host`，VT 序列解析在 `src/terminal/parser`（状态机）与 `src/terminal/adapter`。

我还读了 WindowsTerminal/main.cpp 的入口 `wWinMain`：它先 `SetConsoleCtrlHandler(NULL, FALSE)` 规避「用 cmd 以怪异方式启动时 ^C 在终端里失效」的继承问题，再用 `IsWow64Process2` 校验架构匹配（conhost 必须与 condrv.sys 架构一致，PTY 基础设施才能工作），最后 `winrt::init_apartment(single_threaded)` 并构造 `WindowEmperor`。注释里那句 "LOAD BEARING"——STA 必须在构造任何 Xaml 对象之前初始化——挺能体现 Windows UI 开发的坑。

判断：这个仓库最值钱的地方，是把终端「内核」做成了可复用组件（TerminalCore / TerminalControl 可被第三方嵌入），而不只是一个闭源应用本身。
