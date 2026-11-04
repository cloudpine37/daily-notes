---
date: 2026-11-04
type: repo
subject: microsoft/autogen
tags: [ai, multi-agent, python, llm]
summary: AutoGen 是微软的多智能体框架，最新 Python 版 0.7.5，现已归档并迁往 Microsoft Agent Framework。
---

# AutoGen：一个已经「归档」的多智能体框架

microsoft/autogen（6.1 万+ star）是微软做多智能体（multi-agent）编排的代表作，README 开篇仍是「A programming framework for agentic AI」。但今天再读它，最大的新闻不是功能，而是它的**状态**：README 顶部挂着 `[Archived]` 徽标和一段 CAUTION，明确写着「Project archived and no longer maintained by Microsoft」，并引导新用户改用 [Microsoft Agent Framework](https://github.com/microsoft/agent-framework)、老用户按迁移指南搬家。也就是说，这个仓库现在只作历史参考。

## 最后的版本号：0.7.5

`python/packages/autogen-agentchat/pyproject.toml` 和 `python/packages/autogen-core/pyproject.toml` 里都写着 `version = "0.7.5"`；GitHub 最新 release 也是 `python-v0.7.5`（2025-09-30）。AutoGen 早已不是早期 `pyautogen` 单包，而是拆成 uv workspace 里的一组包：`autogen-core`（运行时与基础接口）、`autogen-agentchat`（agents 与 teams）、`autogen-ext`（模型客户端/扩展）、`autogenstudio`（无代码 GUI）等。

安装命令（README 要求 Python 3.10+）：

```bash
pip install -U "autogen-agentchat" "autogen-ext[openai]"
```

## 架构：从「对话」到「事件驱动的 actor」

核心协议在 `python/packages/autogen-core/src/autogen_core/_agent.py`：`Agent` 是一个 Protocol，要求实现 `id`、`bind_id_and_runtime(id, runtime)`、`on_message(message, ctx)`、`save_state/load_state`、`close`。这暴露了 AutoGen 新一代 API 的根本设计——agent 是被绑定到 `AgentRuntime` 上的 actor，靠消息事件驱动，而不是早期 `pyautogen` 里那种围着对话历史转的多 agent 协作。

上层 `autogen_agentchat` 的 `AssistantAgent`（`agents/_assistant_agent.py`，约 79 KB）把 `model_client`、`tools`、`workbench`、`handoffs`、`memory`、`model_context` 都做成声明式配置（`AssistantAgentConfig`）。README 的 Hello World 很直白：

```python
import asyncio
from autogen_agentchat.agents import AssistantAgent
from autogen_ext.models.openai import OpenAIChatCompletionClient

async def main() -> None:
    model_client = OpenAIChatCompletionClient(model="gpt-4.1")
    agent = AssistantAgent("assistant", model_client=model_client)
    print(await agent.run(task="Say 'Hello World!'"))
    await model_client.close()

asyncio.run(main())
```

## 我的判断

AutoGen 的「actor + 事件 + 声明式配置」路线现在被 `microsoft/agent-framework`（README 称其为 enterprise-ready 的继任者，1.0 已稳定）接了过去，还加了 A2A、MCP 互操作。想研究多智能体架构，读这个已归档仓库反而更清楚——没有新提交的噪音、代码结构稳定；但若要上生产，就该按 README 的告诫改用 Agent Framework，而不是继续依赖一个明确不再维护的项目（迁移成本与兼容细节我未实测，标注「待验证」）。
