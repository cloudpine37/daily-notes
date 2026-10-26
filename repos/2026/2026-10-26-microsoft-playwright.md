---
date: 2026-10-26
type: repo
subject: microsoft/playwright
tags: [browser-automation, testing, typescript, e2e]
summary: 一套 API 驱动 Chromium/Firefox/WebKit 的浏览器自动化与测试框架，正从测试工具扩展为 AI Agent 操作网页的底座。
---

# Playwright：一套 API 驱动三大浏览器的自动化框架

微软开源的 Playwright（Apache-2.0，抓取时 star 98,555）今天读。它的核心卖点一句话：用**同一套 API** 驱动 Chromium、Firefox、WebKit 三个浏览器，既做端到端测试，也做脚本自动化，这两年又成了 AI Agent 操作网页的底座。

仓库是用 npm workspaces（`"workspaces": ["packages/*"]`）组织的 monorepo。`main` 分支根 `package.json` 的 `version` 是 **`1.65.0-next`**，`engines.node` 要求 **`>=20`**。README 标注的内置浏览器版本：Chromium **157.0.8103.15**、Firefox **158.0**、WebKit **27.2**，三者在 Linux/macOS/Windows 全部支持。

安装分五条路，这是它现在最明显的演进方向：跑测试用 `npm init playwright@latest`（或 `npm i -D @playwright/test` 再 `npx playwright install`）；当库用 `npm i playwright`；给编码 Agent 用 `@playwright/cli`；给 LLM 用 `npx @playwright/mcp@latest`；另有 VS Code 扩展。

我印象最深的是「自动等待 + web-first 断言」：不用手写 sleep，`await expect(page).toHaveTitle(/Playwright/)` 这类断言会自动重试到条件成立；定位元素用贴近用户视角的 locator：

```js
page.getByRole('button', { name: 'Submit' })
page.getByLabel('Email')
page.getByTestId('login-form')
```

源码层面，`packages/playwright/src/index.ts` 是测试运行器核心，用 fixture 注入 `browser`/`context`/`page`，`defaultBrowserType` 默认 `chromium`、`headless` 默认 true；`ArtifactsRecorder` 在失败时自动收集截图、trace 与视频。`packages/playwright-core/src/server/index.ts` 导出服务端类型（`Browser`、`BrowserContext`、`Page`）与 dispatcher 层。

一个可复现的库模式截图：

```js
import { chromium } from 'playwright';
const browser = await chromium.launch();
const page = await browser.newPage();
await page.goto('https://playwright.dev/');
await page.screenshot({ path: 'screenshot.png' });
await browser.close();
```

trace 查看器（`npx playwright show-trace trace.zip`）能回放每一步 DOM、网络请求与控制台，排障很顺手。总体看，Playwright 已经从「测试框架」长成面向人和 AI 双重用户的浏览器自动化平台。
