---

## 项目定位

Hermes Agent 是一个**可自我进化的 AI Agent 框架**，核心卖点：

| 能力 | 说明 |
|------|------|
| 自学习闭环 | 从经验创建 Skill、使用中改进、持久化记忆、跨会话搜索 |
| 多入口 | CLI TUI、Telegram/Discord/Slack/飞书/微信等 20+ 消息平台、ACP（IDE 集成） |
| 多模型 | OpenAI、Anthropic、OpenRouter、Nous Portal 等 18+ Provider，无厂商锁定 |
| 多运行环境 | 本地、Docker、SSH、Modal、Daytona、Singularity 六种终端后端 |
| 定时任务 | 内置 Cron，自然语言定义定时自动化 |

---

## 技术栈

- **语言**：Python 3.11–3.13（`pyproject.toml` 明确限制 `<3.14`）
- **包管理**：`uv` + `setuptools`，依赖**精确版本锁定**（防供应链攻击）
- **存储**：SQLite + FTS5 全文搜索（会话、记忆）
- **前端**：Docusaurus 文档站（`website/`）
- **规模**：约 **2225 个 Python 文件**，**~89 万行代码**（含测试）

---

## 核心架构

```mermaid
flowchart TD
    subgraph Entry["入口层"]
        CLI["cli.py / hermes_cli"]
        GW["gateway/run.py"]
        ACP["acp_adapter/"]
        Batch["batch_runner.py"]
    end

    subgraph Core["核心引擎"]
        Agent["AIAgent (run_agent.py)"]
        PB["prompt_builder.py"]
        RP["runtime_provider.py"]
        MT["model_tools.py"]
    end

    subgraph Tools["工具层"]
        Reg["tools/registry.py"]
        Term["terminal_tool (6 backends)"]
        Browser["browser_tool"]
        MCP["mcp_tool"]
        Delegate["delegate_tool (子 Agent)"]
    end

    subgraph Persist["持久化"]
        State["hermes_state.py (SQLite+FTS5)"]
        Memory["plugins/memory/"]
        Skills["skills/ + optional-skills/"]
    end

    CLI --> Agent
    GW --> Agent
    ACP --> Agent
    Batch --> Agent
    Agent --> PB
    Agent --> RP
    Agent --> MT
    MT --> Reg
    Reg --> Term
    Reg --> Browser
    Reg --> MCP
    Reg --> Delegate
    Agent --> State
    Agent --> Memory
    Agent --> Skills
```

**设计原则**：一个 `AIAgent` 类服务所有入口（CLI、Gateway、ACP、Batch），平台差异只在入口层，核心逻辑完全复用。

---

## 关键模块拆解

### 1. 核心 Agent 循环 — `run_agent.py`

```320:360:C:\Users\Time\Projects\hermes-agent\run_agent.py
class AIAgent:
    """
    AI Agent with tool calling capabilities.

    This class manages the conversation flow, tool execution, and response handling
    for AI models that support function calling.
    """
    // ...
    def __init__(
        self,
        base_url: str = None,
        api_key: str = None,
        provider: str = None,
        api_mode: str = None,
        // ...
        model: str = "",
        max_iterations: int = 90,  # Default tool-calling iterations
        enabled_toolsets: List[str] = None,
        disabled_toolsets: List[str] = None,
```

`run_conversation()` 已抽到 `agent/conversation_loop.py`，`AIAgent` 作为薄封装。每轮循环：

1. 组装 system prompt（`prompt_builder.py`）
2. 解析 Provider → API 模式（`chat_completions` / `codex_responses` / `anthropic_messages`）
3. 调用 LLM（可中断）
4. 若有 `tool_calls` → 执行工具 → 继续循环
5. 若文本回复 → 持久化会话 → 返回

### 2. 工具注册表 — `tools/registry.py`

```1:15:C:\Users\Time\Projects\hermes-agent\tools\registry.py
"""Central registry for all hermes-agent tools.

Each tool file calls ``registry.register()`` at module level to declare its
schema, handler, toolset membership, and availability check.  ``model_tools.py``
queries the registry instead of maintaining its own parallel data structures.

Import chain (circular-import safe):
    tools/registry.py  (no imports from model_tools or tool files)
           ^
    tools/*.py  (import from tools.registry at module level)
           ^
    model_tools.py  (imports tools.registry + all tool modules)
```

- **70+ 工具**，分布在 **28 个 toolset**
- 每个 `tools/*.py` 在模块顶层调用 `registry.register()`，**导入时自动发现**，无需手动维护列表
- 支持 `check_fn` 运行时可用性检测（如 Docker 是否启动）

### 3. CLI 入口 — `hermes_cli/main.py`

```1:14:C:\Users\Time\Projects\hermes-agent\hermes_cli\main.py
#!/usr/bin/env python3
"""
Hermes CLI - Main entry point.

Usage:
    hermes                     # Interactive chat (default)
    hermes chat                # Interactive chat
    hermes gateway             # Run gateway in foreground
    hermes gateway start       # Start gateway as service
    hermes setup               # Interactive setup wizard
    hermes cron                # Manage cron jobs
    hermes doctor              # Check configuration and dependencies
```

`hermes` 命令映射到 `hermes_cli.main:main`，子命令涵盖 setup、gateway、cron、doctor、skills、tools 等。

### 4. 消息网关 — `gateway/`

支持 **20 个平台适配器**，包括：

- 国际：Telegram、Discord、Slack、WhatsApp、Signal、Matrix、Email
- 国内：**飞书（feishu）**、钉钉、企业微信、QQ 机器人、微信（weixin）、元宝（yuanbao）

流程：`平台事件 → Adapter.on_message() → GatewayRunner._handle_message() → AIAgent.run_conversation() → 回传响应`

### 5. Skills 系统 — `skills/` + `optional-skills/`

- 内置 **73 个 Skill**（`SKILL.md` 格式，兼容 [agentskills.io](https://agentskills.io) 标准）
- 分类：软件开发、研究、创意、生产力、MLOps、GitHub 等
- Agent 可在复杂任务后**自动创建 Skill**，并在使用中自我改进

### 6. 记忆系统 — `plugins/memory/`

- 插件化 Memory Provider（单选激活）
- 支持 Honcho 辩证用户建模
- FTS5 跨会话搜索 + LLM 摘要
- 周期性 nudge 提醒 Agent 持久化知识

---

## 数据流（三条主路径）

**CLI 会话**
```
用户输入 → HermesCLI.process_input()
  → AIAgent.run_conversation()
    → prompt_builder → API 调用 → tool_calls 循环
    → 最终回复 → 显示 → 写入 SessionDB
```

**Gateway 消息**
```
平台事件 → Adapter → GatewayRunner
  → 鉴权 → 解析 session → AIAgent → 回传平台
```

**Cron 定时任务**
```
Scheduler tick → 加载 jobs.json
  → 新建 AIAgent（无历史）→ 注入 Skill → 执行 → 投递到目标平台
```

---

## 值得关注的工程细节

1. **依赖安全**：所有核心依赖用 `==X.Y.Z` 精确锁定，注释里明确写了 Mini Shai-Hulud 供应链攻击的应对
2. **Windows 原生支持**：`hermes_bootstrap.py` 处理 UTF-8 stdio，安装器捆绑 MinGit
3. **Profile 隔离**：`hermes -p <name>` 每个 profile 独立 HERMES_HOME、配置、记忆、会话
4. **子 Agent 委托**：`delegate_tool.py` 支持并行子任务，有迭代预算控制
5. **测试规模**：约 25,000 个测试，分布在 ~1,250 个文件

---

## 推荐阅读顺序（深入源码）

若你想继续深挖，建议按这个顺序读：

1. `website/docs/developer-guide/architecture.md` — 架构总览
2. `run_agent.py` + `agent/conversation_loop.py` — Agent 主循环
3. `agent/prompt_builder.py` — System prompt 组装
4. `tools/registry.py` + 任意 `tools/*.py` — 工具注册与实现
5. `gateway/run.py` — 消息网关调度
6. `hermes_state.py` — SQLite 会话存储
7. `cron/` — 定时任务

---
