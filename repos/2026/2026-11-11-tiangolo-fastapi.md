---
date: 2026-11-11
type: repo
subject: tiangolo/fastapi
tags: [python, backend, web-framework, api]
summary: FastAPI 是用 Python 类型注解构建 HTTP API 的高性能框架，当前版本 0.162.4，底层由 Starlette 与 Pydantic 驱动。
---

# FastAPI：用标准类型注解声明整条 API

## 是什么

FastAPI 是一个用 Python 写 HTTP API 的 Web 框架，卖点是「只用标准 Python 类型提示」，不用学新的 DSL。它把 Web 层交给 [Starlette](https://starlette.dev/)、数据校验交给 [Pydantic](https://pydantic.dev/docs/)，自己负责把二者粘合成一条声明式的路径。

## 真实版本与依赖

`fastapi/__init__.py` 第 3 行写死了 `__version__ = "0.162.4"`，这是仓库当前的发布版本。`pyproject.toml` 声明 `requires-python = ">=3.10"`，核心依赖有 `starlette>=0.46.0`、`pydantic>=2.9.0`、`anyio>=4.1.0`、`typing-extensions>=4.8.0`，以及较新的 `annotated-doc>=0.0.2` 和 `opentelemetry-api>=1.44.0`（观测性）。构建后端是 `pdm-backend`，许可证 MIT，classifiers 覆盖 Python 3.10～3.15。

## 核心设计

`fastapi/applications.py` 里 `class FastAPI(Starlette)`——FastAPI 直接继承 Starlette 的应用类，路由、中间件、异常处理都复用了 Starlette 的 ASGI 实现，自己专注在类型驱动与文档生成上。

## 上手命令

README 的最小示例用 `@app.get("/")` 和 `@app.get("/items/{item_id}")` 声明两个端点。安装和启动：

```console
uv add "fastapi[standard]"
uv run fastapi dev
```

`fastapi dev` 会自动读取 `main.py`、探测 app 对象，借 Uvicorn 在 `http://127.0.0.1:8000` 启动并带自动重载。访问 `/items/5?q=somequery` 会原样返回 `{"item_id": 5, "q": "somequery"}`（README 里的真实输出片段）。

## 值得注意

- 函数参数的类型注解一次性声明路径/查询/请求体参数，框架据此自动校验、序列化，并生成 OpenAPI 文档：`/docs` 是 Swagger UI，`/redoc` 是 ReDoc。
- 依赖注入（`Depends`）、OAuth2/JWT、WebSocket、CORS 都由框架或 Starlette 提供。
- README 引用 TechEmpower 基准，称其在 Uvicorn 下是「最快的 Python 框架之一」、性能与 NodeJS/Go 相当——这是官方说法，具体数字我未复测，**待验证**。
- 还有一条 `uv run fastapi deploy` 可以一键部署到 FastAPI Cloud（同作者团队维护的托管服务）。

## 我的判断

FastAPI 真正的竞争力，是把「类型」当成了唯一的 API 描述语言：一次注解同时换来校验、序列化、编辑器补全和在线文档。代价是它绑定在 Pydantic 生态上，想换校验层会比较别扭；但对急着把 Python 服务上生产的小团队来说，这个取舍很划算。
