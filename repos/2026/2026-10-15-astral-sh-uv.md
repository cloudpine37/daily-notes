---
date: 2026-10-15
type: repo
subject: astral-sh/uv
tags: [python, rust, package-manager, cli, toolchain]
summary: uv 是用 Rust 写的极速 Python 包与项目管理器，单工具替代 pip、poetry、pyenv 等一整套工具链。
---

# uv：用 Rust 重写的 Python 包管理器

## 是什么

`uv`（念 "you-vee"）出自 Astral 团队（Ruff、ty 的作者），定位是「极快的 Python 包与项目管理器」。截至 2026-10-15，star 约 9.1 万，最新提交就在当天，最新 release 是 0.13.1（2026-10-14 发布），迭代节奏极快。README 直接宣称比 pip 快 10-100 倍。

## 代码里看到的架构

根 `Cargo.toml` 是标准 Rust workspace，`members = ["crates/*"]`，除 `scripts` 和需要 nightly 的 `crates/uv-trampoline` 外，拆出 80 多个内部 crate：`uv-resolver` 管依赖解析、`uv-python-managed` 管 Python 下载、`uv-publish` 管发布、`uv-windows`/`uv-trampoline-builder` 负责 Windows 特化。几个体现工程取向的细节：

- `rust-version = "1.97.0"`、`edition = "2024"`，工具链追得很新；
- 解析器用 PubGrub（依赖 `astral-pubgrub 0.6.1`），异步栈是 `tokio 1.40` + `reqwest 0.13`，CLI 用 `clap 4.5`；
- release 配置 `lto = "fat"`、`codegen-units = 1`、`strip = true`、`panic = "abort"`，把二进制压到最小最快。

主二进制在 `crates/uv`（`default-run = "uv"`），另有 Windows 专属的 `uvw`（feature `windows-gui-bin`）。`crates/uv-version/src/lib.rs` 里 `version()` 直接返回 `env!("CARGO_PKG_VERSION")`，与 crate 版本保持一致。

## 凭什么取代一整套工具

README 说它一个工具替代 pip、pip-tools、pipx、poetry、pyenv、twine、virtualenv。能力覆盖：

- 项目：`uv init` / `uv add` / `uv lock` / `uv sync`，带 universal lockfile 和 Cargo 式 workspaces；
- 脚本：`uv add --script example.py requests` 内联依赖，`uv run example.py` 隔离执行；
- 工具：`uvx pycowsay` 临时跑 CLI，`uv tool install ruff` 全局安装；
- Python 版本：`uv python install 3.12 3.13 3.14` 下载管理多版本，`uv python pin 3.11` 锁定；
- 兼容 pip：`uv pip compile/venv/sync` 平滑迁移旧流程。

## 判断

Windows 一条命令装好：

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

uv 把「管 Python 版本、建虚拟环境、解析锁定、跑脚本、发布」收敛进一个 Rust 单二进制，这是 pip/poetry/pyenv 各自为政给不了的体验。代价是构建门槛高（rust-version 已到 1.97）且功能面宽、迭代快，踩新特性的风险不低。许可证 MIT OR Apache-2.0，宽松友好。
