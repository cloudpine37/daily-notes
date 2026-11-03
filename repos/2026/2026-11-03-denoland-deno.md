---
date: 2026-11-03
type: repo
subject: denoland/deno
tags: [runtime, typescript, rust, cli]
summary: Deno 是用 Rust + V8 打造的现代 JS/TS 运行时，2.9.8 以单二进制交付并默认安全沙箱。
---

# Deno：默认安全、单二进制的 JS/TS 运行时

denoland/deno（10.8 万+ star，最近推送 2026-10-29）在 README 里自称「A modern runtime for JavaScript and TypeScript」——一个基于 V8、Rust 与 Tokio 的现代运行时，主打「secure defaults」（安全默认值）。把它的 README 和 Cargo.toml 对照着读，能很清楚看到它和 Node 的路线差异。

## 安装与第一段代码

README 给 Windows 用户的命令很直接：

```powershell
irm https://deno.land/install.ps1 | iex
# 或
winget install --id=DenoLand.Deno
```

跑一个 HTTP 服务只需要内置的 `Deno.serve`：

```ts
Deno.serve((_req: Request) => new Response("Hello, world!"));
```

```sh
deno run --allow-net server.ts   # 监听 localhost:8000
```

注意 `--allow-net`：Deno 默认不给网络、文件等权限，必须显式授权——这是它相对 Node 最本质的差异。

## 从 Cargo.toml 看架构

根 Cargo.toml 是一个庞大的 workspace（Rust edition 2024，MIT 许可）。`cli/Cargo.toml` 里 `version = "2.9.8"`、`default-run = "deno"`，说明整个运行时最终收敛成一个 `deno` 二进制。feature 里默认 `v8`，另留 `quickjs` 作为备选引擎，还有 `dhat-heap`（堆分析）、`hmr`（开发期热重载）等开关。

workspace 成员里 `ext/*` 是亮点：`ext/http`、`ext/fetch`、`ext/crypto`、`ext/webgpu`、`ext/websocket`、`ext/ffi` 等实现 Web 平台能力；`ext/node`、`libs/npm`、`libs/node_resolver`、`libs/node_shim` 则专门做 Node/npm 生态兼容。核心层是 `libs/core`、`libs/deno_v8` 与 `runtime/`。

## 入口与 CLI 分层

`cli/main.rs` 只有三行，调用 `deno::main()`——注释说明拆成 lib.rs/main.rs 是为了 CI 上不构建二进制也能跑测试。真正的逻辑在 `cli/lib.rs`（约 52 KB），模块列表直接暴露了它「不止是运行时」的野心：`lsp`、`npm`、`node`、`task_runner`、`tsc`、`standalone`、`worker`、`tools`……也就是内置语言服务、npm 集成、任务运行、独立二进制打包等全都做进了同一个程序里。

## 我的判断

Deno 把「运行时 + 工具链」装进一个二进制，再用权限模型解决 Node 时代默认全开的隐患；2.x 之后又靠 `ext/node` + `libs/npm` 补上生态兼容。对想直接用 TypeScript 写 CLI 或脚本、又不想搭一堆构建工具的人来说，它比 Node + ts-node 更省心（性能与兼容边角我未实测，标注「待验证」）。
