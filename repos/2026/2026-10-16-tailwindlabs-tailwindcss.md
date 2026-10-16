---
date: 2026-10-16
type: repo
subject: tailwindlabs/tailwindcss
tags: [css, frontend, rust, monorepo]
summary: Tailwind CSS v4.3.3 把候选扫描引擎下沉到 Rust（oxide），98k star 的 pnpm+turbo 单仓库，主题配置转向 CSS 文件。
---

# Tailwind CSS v4.3.3：把扫描引擎写进 Rust 的 utility-first 框架

Tailwind CSS 是目前最流行的 utility-first CSS 框架：不写语义类，而是组合大量原子类（如 `flex`、`px-4`）来拼界面，最终只输出「你真正用到的」那些样式。仓库 MIT 协议，star 约 **97,886**（`GET /repos` 实测），最新 release 是 **v4.3.3**（2026-07-16 发布），主分支 `main` 当天仍有提交（pushed_at 2026-10-16）。v4 相较 v3 最大的变化不在「写什么类」，而在「怎么编译类」——核心扫描/抽取逻辑从 JS 迁到了 Rust。

## 单仓库与构建链

仓库根 `package.json`（`@tailwindcss/root`）声明 `packageManager: pnpm@11.9.0`，是典型的 pnpm + turbo monorepo：

```json
"build": "turbo build --filter=!./playgrounds/*",
"test": "cargo test && vitest run --hideSkippedTests"
```

`test` 里同时跑 `cargo test` 与 `vitest`，直接暴露了「Rust + TS 混编」结构；devDependencies 里是 typescript 5.9.3、vitest 4.1.10、turbo 2.10.8。

## Rust「oxide」引擎

Rust 代码集中在 `crates/`，核心 crate `crates/oxide/Cargo.toml` 名为 **tailwindcss-oxide v0.1.0**，依赖 rayon（并行）、globwalk、walkdir、rustc-hash、regex。其 `src/lib.rs` 导出 `Scanner`、`ChangedContent`，模块含 `scanner`、`extractor`、`glob`、`fast_skip`、`throughput`——职责是在源码里高速扫描并抽取用到的 class 候选；`fast_skip` 与 `throughput` 这些名字也暗示团队在为监听模式与基准吞吐做专门优化。JS 侧 `packages/tailwindcss/package.json` 通过 `@tailwindcss/oxide: workspace:^` 挂到这个原生绑定。

## 包结构与「CSS 优先」入口

`packages/` 下是一套集成矩阵：`tailwindcss`（核心）、`@tailwindcss/cli`、`-postcss`、`-vite`、`-node`、`-browser`、`-turbopack`、`-webpack`、`-upgrade`。主包 `packages/tailwindcss/package.json` 版本 4.3.3，exports 里 `"."` 的 import 直接指向 **`./src/index.ts`**（源码级 TS 导入），发布时 `files` 只含 `dist` 与 `index.css / preflight.css / theme.css / utilities.css`。这四个 CSS 入口是 v4「CSS-first 主题配置」的信号；`@theme` 等具体语法建议以官网为准（本摘要未逐条核对其语法，标「待验证」）。

## CLI 与可运行命令

`packages/@tailwindcss-cli` 提供 `tailwindcss` 二进制（`bin: {"tailwindcss": "./dist/index.mjs"}`），`src/index.ts` 里只有 `build` 与 `canonicalize` 两个子命令，用法串为：

```
tailwindcss [--input input.css] [--output output.css] [--watch] [--poll=ms] [options…]
```

对应可运行：`npx @tailwindcss/cli --input input.css --output output.css --watch`；仓库侧构建/测试是 `pnpm build`、`pnpm test`。

**我的判断**：v4 把「扫描-抽取-生成」热路径 Native 化，换取大项目冷启动与监听性能；同时把主题配置从 JS 挪进 CSS 文件。代价是 monorepo 复杂度上升（turbo + cargo + vitest 三套工具链），对只想改一处样式的使用者门槛变高。
