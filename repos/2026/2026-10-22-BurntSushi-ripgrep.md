---
date: 2026-10-22
type: repo
subject: BurntSushi/ripgrep
tags: [rust, cli, search, grep, 工具链]
summary: Rust 写的极速递归搜索工具 rg，默认尊重 gitignore，当前 15.2.0。
---

# ripgrep（rg）：把「快」做成默认项的 Rust 搜索工具

ripgrep 是一款 Rust 写的行导向递归搜索工具，二进制名是 `rg`。README 开头就点明定位：递归搜索当前目录里的正则，默认尊重 `.gitignore`/`.ignore`/`.rgignore`，自动跳过隐藏文件与二进制文件；`rg -uuu` 可一次性关掉这些自动过滤。我读了根 `Cargo.toml`、`crates/core/main.rs` 与 `README.md` 三个文件。

## 版本与规模

`Cargo.toml` 里写着 `version = "15.2.0"`、`rust-version = "1.96"`、`edition = "2024"`。GitHub 元数据显示约 6.9 万 star（69,338），最近一次推送在 2026-08-04，仍在活跃维护，默认分支 `master`，双重许可（MIT 或 UNLICENSE）。包被拆成 workspace（`crates/globset`、`grep`、`cli`、`matcher`、`pcre2`、`printer`、`regex`、`searcher`、`ignore`），但主包的 `[[bin]]` 直接指向 `crates/core/main.rs`，所以根 `Cargo.toml` 就是 ripgrep 自己的清单——这也是 `crates/core/Cargo.toml` 会 404 的原因。

## 它为什么快

README 用 Linux 内核源码树做基准（作者机器 i9-12900K）：`rg -n -w '[A-Z]+_SUSPEND'` 出 536 行、耗时 0.082s（基线 1.00x），对比 `git grep -P` 的 0.273s、`ag` 的 0.443s。作者把速度归因于：底层是 Rust regex 引擎（有限自动机 + SIMD + 字面量优化）；`.gitignore` 规则用 `ignore` crate 的 `RegexSet` 一次匹配多条 glob；目录遍历靠 crossbeam 的无锁并行迭代器；搜索时在「内存映射」与「增量缓冲」之间自动选策略。

## 入口代码的细节

`crates/core/main.rs` 的 `main() -> ExitCode` 里有个很贴 Unix 习惯的处理：捕获到 `BrokenPipe` 时直接返回退出码 0，而不是报错——管道下游（比如 `rg foo | head`）提前关闭不算失败。`run()` 再按模式分发：单线程 `search`、并行 `search_parallel`（走 `walk_builder()?.build_parallel()`，用原子布尔 `matched`/`searched` 汇总）、`files`、`types`、`generate`；退出码语义是 0=有匹配、1=无匹配、2=出错。

## 上手命令（均来自 README）

Windows 上装：`winget install BurntSushi.ripgrep.MSVC`，或 `scoop install ripgrep`、`choco install ripgrep`；macOS 是 `brew install ripgrep`。按文件类型过滤用 `rg -tpy foo`（只搜 Python）/ `rg -Tjs foo`（排除 JS）；需要 look-around、反向引用时开 PCRE2：`-P/--pcre2` 或 `--auto-hybrid-regex`；搜 GBK/Shift_JIS 等非 UTF-8 用 `-E/--encoding`；搜压缩包（gzip/xz/zstd 等）用 `-z/--search-zip`。

## 我的取舍

ripgrep 的价值不在「又一个 grep」，而在同时做对三件事：默认就快、默认尊重 gitignore、Unicode 不减速。它也很诚实：不遵循 POSIX、不无所不在，README 明说「到处都要有」的场景还是该用 GNU grep。对主要在 Windows 上干活的场景，`rg` + 编辑器插件基本能覆盖日常全文检索，值得装。
