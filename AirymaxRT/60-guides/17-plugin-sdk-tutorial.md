# Airymax Plugin SDK 开发教程

> **P4.3.3** — Plugin SDK 完整开发指南
>
> 预计阅读时间：**30 分钟** | 难度：初级 → 中级

---

## 目录

1. [Plugin SDK 概述](#1-plugin-sdk-概述)
2. [插件生命周期](#2-插件生命周期)
3. [创建第一个插件](#3-创建第一个插件)
4. [Plugin Manifest 格式](#4-plugin-manifest-格式)
5. [可用 Hooks 参考](#5-可用-hooks-参考)
6. [创建 Tool Plugin（Web 搜索工具）](#6-创建-tool-pluginweb-搜索工具)
7. [创建 Hook Plugin（Token 用量追踪器）](#7-创建-hook-plugintoken-用量追踪器)
8. [本地测试插件](#8-本地测试插件)
9. [发布插件到 Marketplace](#9-发布插件到-marketplace)

---

## 1. Plugin SDK 概述

Airymax 的 Plugin SDK 是一套插件化扩展框架，允许开发者通过标准化的接口扩展 Airymax 的核心能力。插件运行在沙箱环境中，支持热加载、依赖管理和资源隔离。

### 1.1 插件能做什么

| 能力 | 说明 |
|:-----|:-----|
| **扩展 Agent 行为** | 自定义 Agent 的推理策略、认知循环和执行逻辑 |
| **添加 Tool 工具** | 为 Agent 提供可调用的外部工具（搜索、计算、API 调用等） |
| **注入 Hook 钩子** | 在 Agent 生命周期中插入自定义逻辑（日志、审计、权限控制） |
| **封装 Skill 技能** | 将多步骤、多工具协作的复合任务封装为可复用的技能 |

### 1.2 四种插件类型

Airymax 定义了四种插件基类，分别对应不同的扩展场景：

| 插件类型 | 基类 | 用途 | 典型场景 |
|:---------|:-----|:-----|:---------|
| **AgentPlugin** | `agentos.plugin_types.AgentPlugin` | 自定义 Agent 推理策略 | RAG Agent、反思 Agent |
| **ToolPlugin** | `agentos.plugin_types.ToolPlugin` | 注册可调用工具 | 搜索引擎、计算器、天气查询 |
| **HookPlugin** | `agentos.plugin_types.HookPlugin` | 生命周期事件拦截 | 日志审计、成本追踪、权限检查 |
| **SkillPlugin** | `agentos.plugin_types.SkillPlugin` | 封装可复用技能 | 代码审查、文档生成、安全扫描 |

### 1.3 安装 SDK

```bash
pip install agentos
```

---

## 2. 插件生命周期

每个插件在 Airymax 运行时中经历以下状态转换：

```
DISCOVERED  →  LOADED  →  ACTIVATING  →  ACTIVE  →  DEACTIVATING  →  INACTIVE
                                                           ↓
                                                       UNLOADED
```

### 2.1 状态说明

| 状态 | 枚举值 | 说明 |
|:-----|:-------|:-----|
| **DISCOVERED** | `PluginState.DISCOVERED` | 插件已被发现（注册），尚未加载模块 |
| **LOADED** | `PluginState.LOADED` | 插件模块已加载，`on_load()` 回调已执行 |
| **ACTIVATING** | `PluginState.ACTIVATING` | 正在激活中，`on_activate()` 正在执行 |
| **ACTIVE** | `PluginState.ACTIVE` | 插件已激活，正在运行中 |
| **DEACTIVATING** | `PluginState.DEACTIVATING` | 正在停用中，`on_deactivate()` 正在执行 |
| **INACTIVE** | `PluginState.INACTIVE` | 已停用，可重新激活 |
| **ERROR** | `PluginState.ERROR` | 错误状态，需人工介入 |
| **UNLOADED** | `PluginState.UNLOADED` | 已卸载，`on_unload()` 回调已执行 |

### 2.2 生命周期回调

插件基类 `BasePlugin` 提供以下生命周期回调方法，子类可按需覆写：

```python
from agentos.framework.plugin import BasePlugin

class MyPlugin(BasePlugin):

    async def on_load(self, context: dict) -> None:
        """插件加载时调用。适合初始化资源、建立连接。"""
        pass

    async def on_activate(self, context: dict) -> None:
        """插件激活时调用。适合启动后台任务、注册事件监听。"""
        pass

    async def on_deactivate(self) -> None:
        """插件停用时调用。适合暂停后台任务、释放临时资源。"""
        pass

    async def on_unload(self) -> None:
        """插件卸载时调用。适合关闭连接、清理持久化状态。"""
        pass

    async def on_error(self, error: Exception) -> None:
        """插件出错时调用。适合错误上报、降级处理。"""
        pass

    def get_capabilities(self) -> list[str]:
        """声明插件提供的能力标签，用于服务发现。"""
        return []
```

---

## 3. 创建第一个插件

### 3.1 项目结构

一个典型的 Airymax 插件项目目录结构如下：

```
my-first-plugin/
├── plugin.yaml          # 插件清单（Manifest）
├── __init__.py          # 插件入口模块
├── main.py              # 插件主逻辑
└── requirements.txt     # Python 依赖
```

### 3.2 编写插件清单 (plugin.yaml)

```yaml
# plugin.yaml — Airymax Plugin Manifest
plugin_id: my_first_plugin
name: My First Plugin
version: 0.1.0
description: 一个简单的入门插件示例
author: Your Name
plugin_type: tool

# 入口配置
entry_point: main.py
entry_class: HelloWorldTool

# 能力声明
capabilities:
  - hello_world
  - greeting

# 权限声明
permissions:
  - network.outbound
  - file_system.read

# 依赖声明
dependencies:
  - plugin_id: agentos.core
    version_range: ">=0.1.0"
    optional: false

# 资源限制
max_memory_mb: 128
max_cpu_percent: 50.0
max_threads: 4
timeout_seconds: 30.0

# 元数据
tags:
  - demo
  - beginner
homepage: https://github.com/yourname/my-first-plugin
license: MIT
```

### 3.3 编写插件代码 (main.py)

```python
# main.py — Hello World Tool Plugin
from agentos.plugin_types import ToolPlugin, ToolMetadata, ToolParameter


class HelloWorldTool(ToolPlugin):
    """一个简单的 Hello World 工具插件。"""

    def get_metadata(self) -> ToolMetadata:
        return ToolMetadata(
            name="hello_world",
            description="返回一条问候消息",
            version="0.1.0",
            parameters=[
                ToolParameter(
                    name="name",
                    type="string",
                    description="要问候的名字",
                    required=True,
                ),
                ToolParameter(
                    name="language",
                    type="string",
                    description="问候语言",
                    required=False,
                    default="zh",
                    enum=["zh", "en", "ja"],
                ),
            ],
            returns="string",
            category="demo",
            tags=["hello", "greeting"],
        )

    async def execute(self, params: dict) -> dict:
        name = params.get("name", "World")
        language = params.get("language", "zh")

        greetings = {
            "zh": f"你好，{name}！欢迎使用 Airymax Plugin SDK。",
            "en": f"Hello, {name}! Welcome to Airymax Plugin SDK.",
            "ja": f"こんにちは、{name}！Airymax Plugin SDK へようこそ。",
        }

        return {"greeting": greetings.get(language, greetings["zh"])}
```

### 3.4 在 config.yaml 中注册插件

```yaml
# config.yaml
agent:
  name: demo-agent
  model: gpt-4o

plugins:
  tools:
    - module: my-first-plugin.main
      class: HelloWorldTool
```

### 3.5 运行插件

```bash
# 通过 Airymax CLI 运行
agentrt run --config config.yaml "向张三问好"

# 或直接使用 Python 测试
python -c "
import asyncio
from my_first_plugin.main import HelloWorldTool

tool = HelloWorldTool()
result = asyncio.run(tool.execute({'name': '张三'}))
print(result['greeting'])
"
```

---

## 4. Plugin Manifest 格式

Plugin Manifest（插件清单）是插件的身份标识和元数据描述文件，支持 YAML 和 JSON 两种格式。Airymax 运行时通过 Manifest 发现、加载和配置插件。

### 4.1 Manifest 字段参考

| 字段 | 类型 | 必填 | 说明 |
|:-----|:-----|:----:|:-----|
| `plugin_id` | string | ✅ | 插件唯一标识符，使用 snake_case 命名 |
| `name` | string | ✅ | 插件显示名称 |
| `version` | string | ✅ | 语义化版本号，如 `1.0.0` |
| `description` | string | | 插件功能描述 |
| `author` | string | | 作者名称 |
| `plugin_type` | string | ✅ | 插件类型：`agent` / `tool` / `hook` / `skill` |
| `entry_point` | string | ✅ | 插件入口文件路径（Python 模块路径或文件路径） |
| `entry_class` | string | ✅ | 插件主类名 |
| `dependencies` | array | | 依赖的其他插件列表 |
| `capabilities` | array | | 插件提供的能力标签列表 |
| `permissions` | array | | 插件所需的权限列表 |
| `max_memory_mb` | int | | 最大内存限制（MB），默认 128 |
| `max_cpu_percent` | float | | 最大 CPU 使用率（%），默认 50.0 |
| `max_threads` | int | | 最大线程数，默认 4 |
| `timeout_seconds` | float | | 执行超时时间（秒），默认 30.0 |
| `tags` | array | | 分类标签 |
| `homepage` | string | | 项目主页 URL |
| `license` | string | | 许可证类型 |

### 4.2 依赖声明格式

每个依赖项为一个字典，包含以下字段：

```yaml
dependencies:
  - plugin_id: other_plugin          # 依赖的插件 ID
    version_range: ">=1.0.0,<2.0.0"  # 版本范围（支持语义化版本约束）
    optional: false                   # 是否为可选依赖
```

### 4.3 权限声明

Airymax 在沙箱中运行插件，需要显式声明所需权限：

| 权限标识 | 说明 |
|:---------|:-----|
| `network.outbound` | 出站网络访问 |
| `network.inbound` | 入站网络监听 |
| `file_system.read` | 文件系统读取 |
| `file_system.write` | 文件系统写入 |
| `process.spawn` | 创建子进程 |
| `memory.heapstore` | 访问 HeapStore 持久化存储 |
| `llm.call` | 调用 LLM 接口 |
| `tool.invoke` | 调用其他工具 |

---

## 5. 可用 Hooks 参考

HookPlugin 是 Airymax 插件系统中最灵活的类型，允许开发者在 Agent 生命周期的关键节点注入自定义逻辑。以下列出所有可用的 Hook 事件。

### 5.1 Agent 生命周期 Hooks

```python
class MyHook(HookPlugin):

    def on_agent_start(self, ctx, data=None):
        """
        当 Agent 启动执行时触发。
        
        Args:
            ctx: 运行时上下文对象
            data: 可选的附加数据
        
        Returns:
            None 或修改后的数据
        """
        pass

    def on_agent_end(self, ctx, data=None):
        """
        当 Agent 完成执行时触发。
        
        Args:
            ctx: 运行时上下文对象
            data: 可选的附加数据（包含执行结果）
        """
        pass
```

### 5.2 Tool 事件 Hooks

```python
    def on_tool_call(self, ctx, tool_name="", tool_input=None):
        """
        在工具被调用之前触发。
        可用于：权限校验、参数改写、调用审计。
        
        Args:
            ctx: 运行时上下文对象
            tool_name: 被调用的工具名称
            tool_input: 工具输入参数
        """
        pass

    def on_tool_result(self, ctx, tool_name="", result=None):
        """
        在工具返回结果之后触发。
        可用于：结果过滤、日志记录、性能统计。
        
        Args:
            ctx: 运行时上下文对象
            tool_name: 工具名称
            result: 工具返回结果
        """
        pass
```

### 5.3 LLM 事件 Hooks

```python
    def on_llm_request(self, ctx, messages=None, model=""):
        """
        在向 LLM 发送请求之前触发。
        可用于：Prompt 改写、内容审查、请求日志。
        
        Args:
            ctx: 运行时上下文对象
            messages: 发送给 LLM 的消息列表
            model: 使用的模型名称
        """
        pass

    def on_llm_response(self, ctx, response=None, usage=None):
        """
        在收到 LLM 响应之后触发。
        可用于：Token 用量统计、成本追踪、响应缓存。
        
        Args:
            ctx: 运行时上下文对象
            response: LLM 的响应内容
            usage: Token 用量信息 dict，包含 prompt_tokens、
                   completion_tokens、total_tokens
        """
        pass
```

### 5.4 Memory 事件 Hooks

```python
    def on_memory_read(self, ctx, key="", layer=""):
        """
        当从 Memory 中读取数据时触发。
        
        Args:
            ctx: 运行时上下文对象
            key: 读取的键
            layer: Memory 层级（L1/L2/L3/L4）
        """
        pass

    def on_memory_write(self, ctx, key="", data=None):
        """
        当向 Memory 中写入数据时触发。
        
        Args:
            ctx: 运行时上下文对象
            key: 写入的键
            data: 写入的数据
        """
        pass
```

### 5.5 获取监听事件列表

HookPlugin 基类提供了 `get_events()` 方法，自动检测子类中覆写了哪些 `on_*` 方法：

```python
class MyHook(HookPlugin):
    def on_llm_request(self, ctx, messages=None, model=""):
        print(f"LLM Request: {model}")
    
    def on_llm_response(self, ctx, response=None, usage=None):
        print(f"LLM Response: {usage}")

# 自动检测到的事件：
# -> ['on_llm_request', 'on_llm_response']
print(MyHook().get_events())
```

---

## 6. 创建 Tool Plugin（Web 搜索工具）

下面创建一个完整的 Web 搜索工具插件，演示 ToolPlugin 的实际开发流程。

### 6.1 项目结构

```
web-search-plugin/
├── plugin.yaml
├── requirements.txt
└── web_search.py
```

### 6.2 plugin.yaml

```yaml
plugin_id: web_search
name: Web Search Tool
version: 0.1.0
description: 为 Agent 提供网络搜索能力的工具插件
author: Your Name
plugin_type: tool

entry_point: web_search.py
entry_class: WebSearchTool

capabilities:
  - web_search
  - information_retrieval

permissions:
  - network.outbound

dependencies:
  - plugin_id: agentos.core
    version_range: ">=0.1.0"
    optional: false

max_memory_mb: 256
timeout_seconds: 60.0

tags:
  - search
  - web
  - information
```

### 6.3 requirements.txt

```
requests>=2.28.0
beautifulsoup4>=4.12.0
```

### 6.4 web_search.py

```python
# web_search.py — Web Search Tool Plugin
import hashlib
import time
from typing import Any

import requests
from bs4 import BeautifulSoup

from agentos.plugin_types import ToolPlugin, ToolMetadata, ToolParameter


class WebSearchTool(ToolPlugin):
    """网络搜索工具，Agent 可调用此工具在互联网上搜索信息。"""

    def __init__(self):
        super().__init__()
        self._cache: dict[str, dict[str, Any]] = {}
        self._cache_ttl: float = 300.0  # 缓存 5 分钟

    def get_metadata(self) -> ToolMetadata:
        return ToolMetadata(
            name="web_search",
            description="在互联网上搜索指定关键词，返回搜索结果摘要",
            version="0.1.0",
            parameters=[
                ToolParameter(
                    name="query",
                    type="string",
                    description="搜索关键词",
                    required=True,
                ),
                ToolParameter(
                    name="max_results",
                    type="integer",
                    description="最大返回结果数，默认 5",
                    required=False,
                    default=5,
                ),
                ToolParameter(
                    name="language",
                    type="string",
                    description="搜索结果语言，默认 zh",
                    required=False,
                    default="zh",
                    enum=["zh", "en"],
                ),
            ],
            returns="array",
            category="search",
            tags=["web", "search", "information"],
            is_async=True,
            timeout_seconds=30.0,
        )

    async def execute(self, params: dict) -> dict:
        """执行 Web 搜索。"""
        query = params.get("query", "")
        max_results = int(params.get("max_results", 5))
        language = params.get("language", "zh")

        # 检查缓存
        cache_key = self._make_cache_key(query, language)
        if cache_key in self._cache:
            cached = self._cache[cache_key]
            if time.time() - cached["timestamp"] < self._cache_ttl:
                return cached["data"]

        # 执行搜索
        results = await self._search(query, max_results, language)

        # 写入缓存
        response_data = {
            "query": query,
            "total": len(results),
            "results": results,
        }
        self._cache[cache_key] = {
            "timestamp": time.time(),
            "data": response_data,
        }

        return response_data

    async def _search(
        self, query: str, max_results: int, language: str
    ) -> list[dict]:
        """
        执行实际搜索逻辑。
        此处以 DuckDuckGo 为例，实际项目中可替换为其他搜索引擎 API。
        """
        try:
            url = "https://html.duckduckgo.com/html/"
            headers = {
                "User-Agent": (
                    "Mozilla/5.0 (X11; Linux x86_64) "
                    "AppleWebKit/537.36 (KHTML, like Gecko) "
                    "Chrome/120.0.0.0 Safari/537.36"
                ),
            }
            params = {"q": query}
            if language == "zh":
                params["kl"] = "cn-zh"

            response = requests.get(
                url, headers=headers, params=params, timeout=10
            )
            response.raise_for_status()

            soup = BeautifulSoup(response.text, "html.parser")
            results = []

            for item in soup.select(".result")[:max_results]:
                title_el = item.select_one(".result__title")
                snippet_el = item.select_one(".result__snippet")
                link_el = item.select_one(".result__url")

                if title_el:
                    results.append({
                        "title": title_el.get_text(strip=True),
                        "snippet": (
                            snippet_el.get_text(strip=True)
                            if snippet_el
                            else ""
                        ),
                        "url": (
                            link_el.get_text(strip=True)
                            if link_el
                            else ""
                        ),
                    })

            return results

        except requests.RequestException as e:
            return [{"error": f"搜索请求失败: {str(e)}"}]

    async def on_error(self, error: Exception, params: dict) -> dict:
        """优雅的错误处理。"""
        return {
            "error": str(error),
            "error_type": type(error).__name__,
            "query": params.get("query", ""),
            "results": [],
        }

    @staticmethod
    def _make_cache_key(query: str, language: str) -> str:
        raw = f"{query}:{language}"
        return hashlib.md5(raw.encode()).hexdigest()
```

### 6.5 注册和使用

```yaml
# config.yaml
plugins:
  tools:
    - module: web-search-plugin.web_search
      class: WebSearchTool
```

---

## 7. 创建 Hook Plugin（Token 用量追踪器）

下面创建一个 Token 用量追踪 Hook 插件，演示如何拦截 LLM 请求/响应来计算和记录 Token 消耗。

### 7.1 项目结构

```
cost-tracker-plugin/
├── plugin.yaml
└── cost_tracker.py
```

### 7.2 plugin.yaml

```yaml
plugin_id: cost_tracker
name: LLM Cost Tracker
version: 0.1.0
description: 追踪并记录每次 LLM 调用的 Token 用量和预估成本
author: Your Name
plugin_type: hook

entry_point: cost_tracker.py
entry_class: CostTrackerHook

capabilities:
  - cost_tracking
  - token_usage
  - observability

dependencies:
  - plugin_id: agentos.core
    version_range: ">=0.1.0"
    optional: false

max_memory_mb: 64
timeout_seconds: 5.0

tags:
  - cost
  - token
  - monitoring
  - observability
```

### 7.3 cost_tracker.py

```python
# cost_tracker.py — LLM Cost Tracker Hook Plugin
import json
import logging
import time
from collections import defaultdict
from datetime import datetime
from pathlib import Path
from typing import Any

from agentos.plugin_types import HookPlugin

logger = logging.getLogger(__name__)


# 常见模型的 Token 定价（每 1K tokens，美元）
# 价格仅供参考，以各厂商官网为准
MODEL_PRICING = {
    "gpt-4o": {"input": 0.005, "output": 0.015},
    "gpt-4o-mini": {"input": 0.00015, "output": 0.0006},
    "gpt-4-turbo": {"input": 0.01, "output": 0.03},
    "gpt-3.5-turbo": {"input": 0.0005, "output": 0.0015},
    "claude-3-opus": {"input": 0.015, "output": 0.075},
    "claude-3-sonnet": {"input": 0.003, "output": 0.015},
    "claude-3-haiku": {"input": 0.00025, "output": 0.00125},
    "default": {"input": 0.001, "output": 0.002},
}


class CostTrackerHook(HookPlugin):
    """
    LLM Token 用量与成本追踪 Hook。

    功能：
    - 统计每次 LLM 调用的 prompt / completion token 数量
    - 根据模型定价估算成本
    - 按会话和模型汇总统计
    - 将统计结果持久化到 JSON 文件
    """

    def __init__(self):
        super().__init__()
        # 按模型汇总的统计
        self._model_stats: dict[str, dict[str, Any]] = defaultdict(
            lambda: {
                "total_requests": 0,
                "total_prompt_tokens": 0,
                "total_completion_tokens": 0,
                "total_tokens": 0,
                "total_cost_usd": 0.0,
            }
        )
        # 按会话汇总的统计
        self._session_stats: dict[str, dict[str, Any]] = defaultdict(
            lambda: {
                "total_requests": 0,
                "total_tokens": 0,
                "total_cost_usd": 0.0,
            }
        )
        self._start_time: float = 0.0
        self._output_dir: Path = Path("./cost_tracker_output")

    # ── Agent 生命周期 ──────────────────────────────────

    def on_agent_start(self, ctx, data=None):
        """Agent 启动时初始化统计。"""
        self._start_time = time.time()
        session_id = getattr(ctx, "session_id", "unknown")
        logger.info(
            f"[CostTracker] Agent 启动 — Session: {session_id}"
        )
        return None

    def on_agent_end(self, ctx, data=None):
        """Agent 结束时输出汇总报告并持久化。"""
        elapsed = time.time() - self._start_time
        session_id = getattr(ctx, "session_id", "unknown")

        # 计算全局汇总
        total_requests = sum(
            s["total_requests"] for s in self._model_stats.values()
        )
        total_tokens = sum(
            s["total_tokens"] for s in self._model_stats.values()
        )
        total_cost = sum(
            s["total_cost_usd"] for s in self._model_stats.values()
        )

        # 打印汇总报告
        report_lines = [
            "",
            "=" * 60,
            "  📊 LLM Token 用量与成本报告",
            "=" * 60,
            f"  Session ID : {session_id}",
            f"  运行时长    : {elapsed:.1f} 秒",
            f"  总请求数    : {total_requests}",
            f"  总 Token 数 : {total_tokens:,}",
            f"  预估总成本  : ${total_cost:.4f} USD",
            "",
            "  ── 按模型汇总 ──",
        ]

        for model, stats in sorted(self._model_stats.items()):
            report_lines.extend([
                f"  [{model}]",
                f"    请求数      : {stats['total_requests']}",
                f"    Prompt      : {stats['total_prompt_tokens']:,} tokens",
                f"    Completion  : {stats['total_completion_tokens']:,} tokens",
                f"    总计         : {stats['total_tokens']:,} tokens",
                f"    成本         : ${stats['total_cost_usd']:.4f} USD",
            ])

        report_lines.append("=" * 60)
        report = "\n".join(report_lines)
        logger.info(report)

        # 持久化到文件
        self._save_report(session_id, elapsed, total_cost)

        return None

    # ── LLM 事件 ────────────────────────────────────────

    def on_llm_request(self, ctx, messages=None, model=""):
        """LLM 请求前：记录请求信息。"""
        session_id = getattr(ctx, "session_id", "unknown")
        msg_count = len(messages) if messages else 0
        logger.debug(
            f"[CostTracker] LLM Request — "
            f"Model: {model}, Messages: {msg_count}"
        )
        return None

    def on_llm_response(self, ctx, response=None, usage=None):
        """LLM 响应后：统计 Token 用量和成本。"""
        if usage is None:
            return None

        model = getattr(ctx, "last_model", "unknown")
        session_id = getattr(ctx, "session_id", "unknown")

        prompt_tokens = usage.get("prompt_tokens", 0)
        completion_tokens = usage.get("completion_tokens", 0)
        total_tokens = usage.get("total_tokens", 0)

        # 计算成本
        pricing = MODEL_PRICING.get(model, MODEL_PRICING["default"])
        cost = (
            (prompt_tokens / 1000) * pricing["input"]
            + (completion_tokens / 1000) * pricing["output"]
        )

        # 更新模型统计
        ms = self._model_stats[model]
        ms["total_requests"] += 1
        ms["total_prompt_tokens"] += prompt_tokens
        ms["total_completion_tokens"] += completion_tokens
        ms["total_tokens"] += total_tokens
        ms["total_cost_usd"] += cost

        # 更新会话统计
        ss = self._session_stats[session_id]
        ss["total_requests"] += 1
        ss["total_tokens"] += total_tokens
        ss["total_cost_usd"] += cost

        logger.info(
            f"[CostTracker] Model: {model} | "
            f"Prompt: {prompt_tokens} | "
            f"Completion: {completion_tokens} | "
            f"Cost: ${cost:.6f} USD"
        )

        return None

    # ── 辅助方法 ────────────────────────────────────────

    def _save_report(
        self, session_id: str, elapsed: float, total_cost: float
    ):
        """将统计报告持久化为 JSON 文件。"""
        self._output_dir.mkdir(parents=True, exist_ok=True)

        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        filename = self._output_dir / f"cost_report_{session_id}_{timestamp}.json"

        report = {
            "session_id": session_id,
            "timestamp": datetime.now().isoformat(),
            "elapsed_seconds": elapsed,
            "total_cost_usd": total_cost,
            "model_stats": {
                model: dict(stats)
                for model, stats in self._model_stats.items()
            },
            "session_stats": {
                sid: dict(stats)
                for sid, stats in self._session_stats.items()
            },
        }

        with open(filename, "w", encoding="utf-8") as f:
            json.dump(report, f, indent=2, ensure_ascii=False)

        logger.info(f"[CostTracker] 报告已保存至: {filename}")

    def get_events(self) -> list[str]:
        """声明此 Hook 监听的事件。"""
        return [
            "on_agent_start",
            "on_agent_end",
            "on_llm_request",
            "on_llm_response",
        ]
```

### 7.4 注册和使用

```yaml
# config.yaml
plugins:
  hooks:
    - module: cost-tracker-plugin.cost_tracker
      class: CostTrackerHook
```

运行后，每次 LLM 调用都会在日志中输出 Token 用量，Agent 结束时输出汇总报告并保存为 JSON 文件：

```
==============================================================
  📊 LLM Token 用量与成本报告
==============================================================
  Session ID : abc123
  运行时长    : 45.2 秒
  总请求数    : 12
  总 Token 数 : 8,450
  预估总成本  : $0.0234 USD

  ── 按模型汇总 ──
  [gpt-4o]
    请求数      : 8
    Prompt      : 5,200 tokens
    Completion  : 2,100 tokens
    总计         : 7,300 tokens
    成本         : $0.0205 USD
  [gpt-4o-mini]
    请求数      : 4
    Prompt      : 800 tokens
    Completion  : 350 tokens
    总计         : 1,150 tokens
    成本         : $0.0029 USD
==============================================================
```

---

## 8. 本地测试插件

### 8.1 单元测试

Airymax 插件为普通 Python 类，可直接使用 `pytest` 编写单元测试：

```python
# tests/test_cost_tracker.py
import pytest
from unittest.mock import Mock

from cost_tracker_plugin.cost_tracker import CostTrackerHook


class TestCostTrackerHook:

    @pytest.fixture
    def hook(self):
        return CostTrackerHook()

    def test_model_pricing_has_default(self):
        """验证默认定价存在。"""
        from cost_tracker_plugin.cost_tracker import MODEL_PRICING
        assert "default" in MODEL_PRICING

    def test_on_llm_response_updates_stats(self, hook):
        """验证 LLM 响应后统计数据正确更新。"""
        ctx = Mock()
        ctx.last_model = "gpt-4o-mini"
        ctx.session_id = "test-session"

        usage = {
            "prompt_tokens": 500,
            "completion_tokens": 200,
            "total_tokens": 700,
        }

        hook.on_llm_response(ctx, response="test", usage=usage)

        stats = hook._model_stats["gpt-4o-mini"]
        assert stats["total_requests"] == 1
        assert stats["total_prompt_tokens"] == 500
        assert stats["total_completion_tokens"] == 200
        assert stats["total_tokens"] == 700
        # 成本: 500/1000 * 0.00015 + 200/1000 * 0.0006
        expected_cost = (500 / 1000) * 0.00015 + (200 / 1000) * 0.0006
        assert stats["total_cost_usd"] == pytest.approx(expected_cost)

    def test_get_events(self, hook):
        """验证返回正确的事件列表。"""
        events = hook.get_events()
        assert "on_agent_start" in events
        assert "on_agent_end" in events
        assert "on_llm_request" in events
        assert "on_llm_response" in events
```

运行测试：

```bash
pytest tests/ -v
```

### 8.2 集成测试（使用 PluginRegistry）

```python
# tests/test_integration.py
import asyncio
import pytest

from agentos.framework.plugin import PluginRegistry, PluginState
from web_search_plugin.web_search import WebSearchTool


class TestWebSearchIntegration:

    @pytest.fixture
    def registry(self):
        return PluginRegistry()

    def test_register_and_load(self, registry):
        """测试插件注册和加载。"""
        # 注册
        plugin_id = registry.register(WebSearchTool)
        assert plugin_id == "web_search"

        # 加载
        instance = registry.load(plugin_id)
        assert instance is not None
        assert registry.get_state(plugin_id) == PluginState.LOADED

    def test_execute_tool(self, registry):
        """测试工具执行。"""
        plugin_id = registry.register(WebSearchTool)
        instance = registry.load(plugin_id)

        result = asyncio.run(
            instance.execute({"query": "Airymax", "max_results": 3})
        )
        assert "query" in result
        assert "results" in result
        assert result["query"] == "Airymax"

    def test_cache_works(self, registry):
        """测试缓存功能。"""
        plugin_id = registry.register(WebSearchTool)
        instance = registry.load(plugin_id)

        # 第一次查询
        result1 = asyncio.run(
            instance.execute({"query": "test", "max_results": 1})
        )

        # 第二次查询相同关键词，应命中缓存
        result2 = asyncio.run(
            instance.execute({"query": "test", "max_results": 1})
        )

        assert result1 == result2
```

### 8.3 使用 Airymax CLI 进行端到端测试

```bash
# 1. 创建测试配置文件
cat > test_config.yaml << 'EOF'
agent:
  name: test-agent
  model: gpt-4o-mini
  system_prompt: "你是一个测试助手。"

plugins:
  tools:
    - module: web-search-plugin.web_search
      class: WebSearchTool
  hooks:
    - module: cost-tracker-plugin.cost_tracker
      class: CostTrackerHook

runtime:
  max_retries: 1
  timeout: 30
  log_level: debug
EOF

# 2. 运行 Airymax 并测试插件
agentrt run --config test_config.yaml "搜索 Airymax 的最新信息"
```

---

## 9. 发布插件到 Marketplace

### 9.1 准备发布

在发布之前，确保你的插件满足以下条件：

- [ ] `plugin.yaml` 中所有必填字段完整
- [ ] 所有代码通过单元测试
- [ ] `requirements.txt` 列出所有依赖
- [ ] 编写了 README.md 说明插件的用途和使用方法
- [ ] 版本号遵循语义化版本规范
- [ ] 插件在本地测试通过

### 9.2 标准目录结构

```
my-awesome-plugin/
├── plugin.yaml          # 插件清单（必需）
├── README.md            # 使用说明（必需）
├── LICENSE              # 许可证（推荐）
├── requirements.txt     # Python 依赖
├── main.py              # 插件入口
├── tests/               # 测试目录
│   ├── __init__.py
│   └── test_main.py
└── examples/            # 使用示例（可选）
    └── example_config.yaml
```

### 9.3 发布到 OpenLab Marketplace

```bash
# 1. 登录 Airymax CLI
agentrt login

# 2. 验证插件完整性
agentrt plugin validate ./my-awesome-plugin

# 输出示例：
# ✓ plugin.yaml 格式正确
# ✓ 入口模块可导入
# ✓ 入口类继承正确
# ✓ 依赖声明完整
# ✓ 权限声明合法

# 3. 发布插件
agentrt market publish ./my-awesome-plugin

# 输出示例：
# 📦 打包插件: my-awesome-plugin v0.1.0
# 📤 上传到 OpenLab Marketplace...
# ✅ 发布成功！
# 🔗 查看: https://openlab.agentrt.dev/plugins/my-awesome-plugin
```

### 9.4 版本管理

发布新版本时更新 `plugin.yaml` 中的版本号：

```yaml
# v0.1.0 → v0.2.0
version: 0.2.0
```

建议遵循语义化版本规范：

| 版本变更 | 说明 | 示例 |
|:---------|:-----|:-----|
| **Major** | 不兼容的 API 变更 | `1.0.0` → `2.0.0` |
| **Minor** | 向后兼容的新功能 | `1.0.0` → `1.1.0` |
| **Patch** | 向后兼容的 Bug 修复 | `1.0.0` → `1.0.1` |

### 9.5 从 Marketplace 安装插件

```bash
# 搜索插件
agentrt market search "web search"

# 安装插件
agentrt market install community/web-search-plugin

# 安装后即可在 config.yaml 中引用
```

---

## 附录

### A. 常见问题

**Q: 插件可以调用其他插件吗？**

A: 可以。在 `plugin.yaml` 中声明依赖关系，Airymax 运行时会自动注入依赖插件的实例。

**Q: 插件必须异步吗？**

A: ToolPlugin 和 SkillPlugin 的 `execute` 方法要求异步（`async def`）。AgentPlugin 和 HookPlugin 的某些方法支持同步。

**Q: 如何调试插件？**

A: 设置 `runtime.log_level: debug` 可以获取详细的插件生命周期日志。也可以直接在 Python 中实例化插件进行调试。

**Q: 插件执行有超时限制吗？**

A: 是的。默认超时时间为 30 秒（可在 `plugin.yaml` 中通过 `timeout_seconds` 配置）。超时后沙箱会终止插件执行。

### B. 相关资源

- [Airymax 快速上手指南](./quickstart.md)
- [Airymax 架构文档](../10-architecture/01-system-architecture.md)
- [Plugin Demo 示例项目](../ecosystem/examples/plugin-demo/)
- [OpenLab Marketplace](https://openlab.agentrt.dev)

---

> © 2025-2026 SPHARX Ltd. All Rights Reserved.