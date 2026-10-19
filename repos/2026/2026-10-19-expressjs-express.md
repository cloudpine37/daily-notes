---
date: 2026-10-19
type: repo
subject: expressjs/express
tags: [node, backend, web-framework, express]
summary: 读 Express 5.3.0 的 package.json、lib/express.js 与 History.md，梳理 v5 的模块化重构和近期安全修复
---

# Express 5.3.0：老牌 Node 后端框架的模块化演进

Express 是 Node.js 生态里资历最老的 Web 框架之一，GitHub **69,576 stars**、MIT 协议，默认分支仍是 `master`，最近一次推送在 2026-10-17，维护相当活跃。`package.json` 当前版本为 **5.3.0**，`engines` 要求 **Node >= 18**。

## 模块化：router 与 body-parser 被拆成独立依赖

读 `lib/express.js` 能看到 v5 的核心动作：把路由器与请求体解析从框架本体剥离。`var Router = require('router')`、`exports.json = bodyParser.json` 等行把 `body-parser` 的 `json / raw / text / urlencoded` 重新导出，`exports.Router = Router` 也一并透出。`package.json` 的 `dependencies` 印证了这一点：`"router": "^2.2.0"`、`"body-parser": "^2.3.0"`、`"send": "^1.1.0"`、`"serve-static": "^2.2.0"`。`createApplication()` 仍是入口：`index.js` 只做 `module.exports = require('./lib/express')`，把 `app` 做成一个函数并混入 `EventEmitter` 与 application 原型，再挂上 `app.request` / `app.response`。

## 上手与测试

安装只需 `npm install express`。`Readme.md` 的 Quick Start 已改用 ESM：

```js
import express from 'express'
const app = express()
app.get('/', (req, res) => res.send('Hello World'))
app.listen(3000)
```

跑测试的命令来自 `package.json` 的 `scripts.test`：`mocha --require test/support/env --reporter spec --check-leaks test/ test/acceptance/`。

## 5.3.0 值得留意的变更

`History.md` 顶部即 5.3.0。Bug 修复里，`res.send()` 不再同时输出 `Content-Length` 与 `Transfer-Encoding` 两个互斥头；依赖升级带出多处安全修复——`qs` 升到 `^6.16.0` 修掉 CVE-2026-2391、CVE-2026-82417、CVE-2026-82562，`proxy-addr` 升到 `^2.0.8` 修掉 CVE-2026-90711。功能侧新增对 **QUERY 方法**的条件重验证：`req.fresh` 过去只认 GET/HEAD，现在 QUERY 也能命中 304；`res.redirect()` 的 HTML 响应补上了 `<!DOCTYPE html>`、`<title>`、`<body>` 结构。

（待验证：`app.query()` 路由是否已收录进稳定文档，此处仅据变更日志示例。）
