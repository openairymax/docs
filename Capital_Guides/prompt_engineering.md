# P4.3.4: Airymax Prompt 工程指南

> **版本**: v1.0.0 | **最后更新**: 2026-06-18 | **目标读者**: Agent 开发者、Prompt 工程团队

---

## 1. Airymax Prompt 系统概述

Airymax 的 Prompt 系统是一个多层架构，覆盖从模板定义、变量渲染、上下文注入到版本管理、A/B 测试和自动调优的完整生命周期。系统由以下核心组件构成：

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Airymax Prompt System Architecture                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐ │
│  │ Prompt Templates │    │ Prompt Injector │    │  Prompt Tuner   │ │
│  │ (YAML 模板)       │───▶│ (Hook 动态注入)  │───▶│ (评估 & 调优)    │ │
│  │ ecosystem/prompts│    │ sdk-python/hooks │    │ prompts/tuner   │ │
│  └────────┬────────┘    └────────┬────────┘    └────────┬────────┘ │
│           │                      │                      │          │
│  ┌────────▼──────────────────────▼──────────────────────▼────────┐ │
│  │                    Prompt Loader (C Core)                      │ │
│  │  agentos/atoms/coreloopthree/src/prompt_loader.c              │ │
│  │  • YAML 解析  • {variable} 替换  • JSON Schema 校验  • 缓存    │ │
│  └───────────────────────────┬───────────────────────────────────┘ │
│                              │                                      │
│  ┌───────────────────────────▼───────────────────────────────────┐ │
│  │                    Context Injection Layer                     │ │
│  │  • MemoryRovol L1-L4 记忆上下文  • 工具描述  • 会话状态         │ │
│  │  • Agent 元数据  • 多 Agent 协调信息  • 安全策略                │ │
│  └───────────────────────────┬───────────────────────────────────┘ │
│                              │                                      │
│  ┌───────────────────────────▼───────────────────────────────────┐ │
│  │                      LLM Gateway (gateway_d)                  │ │
│  │  Final prompt assembly → Model routing → Response parsing     │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.1 核心组件说明

| 组件 | 路径 | 职责 |
|:------|:-----|:-----|
| **Prompt Loader** | `agentos/atoms/coreloopthree/src/prompt_loader.c` | C 语言核心，从 YAML 模板加载并渲染 Prompt，支持变量替换和 JSON Schema 校验 |
| **Prompt Templates** | `ecosystem/prompts/templates/` | YAML 格式的模板库，按 Cognition / Memory / Security / System 四大类别组织 |
| **Prompt Injector** | `ecosystem/hooks/prompt_injector.py` | Hook 插件，在 `on_llm_request` 事件中动态注入 Prompt 片段 |
| **Prompt Tuner** | `ecosystem/prompts/tuner/` | Python 评估框架，包含 `PromptEvaluator`、`AutoScorer` 和 `ABTestRunner` |
| **Registry** | `ecosystem/prompts/registry.yaml` | 集中管理所有 Prompt 模板的元信息（版本、分类、状态） |

### 1.2 数据流

```
YAML 模板文件
    │
    ▼
Prompt Loader 加载 (agentos_prompt_template_load)
    │
    ▼
渲染变量替换 (agentos_prompt_render)
    │
    ▼
Prompt Injector Hook 注入动态片段 (on_llm_request)
    │
    ▼
MemoryRovol 记忆上下文注入
    │
    ▼
Tool 描述格式化
    │
    ▼
LLM Gateway 发送最终 Prompt
```

---

## 2. Prompt 模板语法

Airymax 使用类 Jinja2 风格的模板语法，支持 `{variable_name}` 占位符变量替换。模板以 YAML 格式定义。

### 2.1 模板文件结构

```yaml
# Airymax Prompt Template: <template_name>
# Category: <cognition|memory|security|system>
# Version: <semver>

name: "template_name"         # 模板唯一标识
version: "1.0.0"              # 语义化版本
description: "模板描述"        # 用途说明
model_family: "any"           # 目标模型族: any | gpt | claude | gemini
temperature: 0.7              # 推荐温度参数
max_tokens: 4096              # 最大输出 token 数

system: |                     # System Prompt（支持多行 | 符号）
  You are {agent_name}, an intelligent assistant.

  ## Available Tools
  {tools}

  ## Memory Context
  {memory_context}

  {additional_instructions}

user_template: |              # User Message 模板
  {user_message}

output_schema:                # 可选：JSON Schema 输出格式约束
  type: object
  properties:
    answer:
      type: string
    confidence:
      type: number
      enum: [0, 0.25, 0.5, 0.75, 1.0]
  required: [answer]

metrics:                      # 可选：评估指标目标值
  target_precision: 0.85
  target_recall: 0.80
  max_hallucination_rate: 0.05
```

### 2.2 变量替换语法

模板中的 `{variable_name}` 会在渲染阶段被替换为实际值。支持的变量类型：

| 变量类别 | 示例变量 | 说明 |
|:---------|:---------|:-----|
| **Agent 元数据** | `{agent_name}`, `{agent_id}`, `{agent_type}` | Agent 标识信息 |
| **会话信息** | `{session_id}`, `{timestamp}`, `{date}`, `{time}` | 当前会话上下文 |
| **模型信息** | `{model}`, `{model_family}` | 目标 LLM 信息 |
| **工具列表** | `{tools}`, `{capabilities}` | 可用工具和能力描述 |
| **记忆上下文** | `{memory_context}`, `{l1_memory}`, `{l2_memory}` | MemoryRovol 各层记忆 |
| **自定义变量** | `{primary_language}`, `{framework}`, `{project_context}` | 模板自定义变量 |
| **用户输入** | `{user_message}`, `{input}` | 用户消息内容 |

**C API 渲染示例**：

```c
#include "prompt_loader.h"

// 加载模板
agentos_prompt_template_t *tmpl = NULL;
agentos_prompt_template_load("ecosystem/prompts/templates/system/coding_agent.yaml", &tmpl);

// 准备变量
agentos_prompt_variable_t vars[] = {
    {"agent_name", "CodeReviewBot"},
    {"primary_language", "Python"},
    {"framework", "FastAPI"},
    {"project_context", "E-commerce backend service"},
    {"tools", "search_codebase, run_tests, deploy_preview"},
    {"user_message", "Review the authentication module for security issues"},
};

// 渲染
agentos_prompt_rendered_t *rendered = NULL;
agentos_prompt_render(tmpl, vars, 6, &rendered);

// 使用渲染后的 Prompt
printf("System: %s\n", rendered->system_prompt);
printf("User: %s\n", rendered->user_message);

// 清理
agentos_prompt_rendered_free(rendered);
agentos_prompt_template_free(tmpl);
```

**Python 渲染示例**：

```python
from ecosystem.prompts import load_template, render_template

# 加载模板
template = load_template("coding_agent", category="system")

# 渲染
rendered = render_template(template, variables={
    "agent_name": "CodeReviewBot",
    "primary_language": "Python",
    "framework": "FastAPI",
    "project_context": "E-commerce backend service",
    "tools": "- search_codebase: Semantic code search\n- run_tests: Execute test suite",
    "user_message": "Review the authentication module for security issues",
})

print(rendered.system_prompt)
print(rendered.user_message)
```

### 2.3 模板分类体系

Airymax 将 Prompt 模板按四大类别组织，对应 `ecosystem/prompts/templates/` 下的四个子目录：

| 分类 | 目录 | 模板示例 | 说明 |
|:-----|:-----|:---------|:-----|
| **Cognition** | `templates/cognition/` | `intent_classify`, `entity_extract`, `plan_generate`, `reflection` | 认知推理类 Prompt |
| **Memory** | `templates/memory/` | `extract_facts`, `dedup_decision`, `summarize`, `rule_generate` | 记忆管理类 Prompt |
| **Security** | `templates/security/` | `code_review`, `security_scan`, `input_validate` | 安全审查类 Prompt |
| **System** | `templates/system/` | `default_agent`, `coding_agent`, `research_agent` | 系统级 Agent 定义 |

---

## 3. System Prompt 设计原则

### 3.1 角色定义 (Role Definition)

有效的 System Prompt 应清晰定义 Agent 的角色身份、核心原则和能力边界：

```yaml
system: |
  You are {agent_name}, a software engineering assistant powered by Airymax.

  ## Core Principles
  1. **Correctness first**: Code must be correct before it is elegant.
  2. **Readability**: Write code that is easy to understand and maintain.
  3. **Best practices**: Follow language-specific best practices and idioms.
  4. **Security**: Never generate code with known security vulnerabilities.
  5. **Testing**: Include tests when generating new functionality.

  ## Technical Stack
  Primary language: {primary_language}
  Framework: {framework}
  Project context: {project_context}
```

**设计要点**：

- **身份声明**：`You are {agent_name}, a ... powered by Airymax` 作为首句，建立角色认知
- **核心原则**：用编号列表明确行为准则，每条原则简洁有力
- **能力边界**：通过 `{tools}` 和 `{capabilities}` 变量声明可用能力
- **技术上下文**：通过 `{primary_language}`, `{framework}` 等变量注入项目环境

### 3.2 约束定义 (Constraints)

约束确保 Agent 行为在预期范围内：

```yaml
system: |
  ## Constraints
  - NEVER generate code with known security vulnerabilities (SQL injection, XSS, etc.)
  - NEVER expose sensitive information (API keys, passwords, tokens)
  - ALWAYS validate user inputs before processing
  - ALWAYS cite sources when making factual claims
  - If you don't know something, say so clearly — do not fabricate information
  - Use tools when they can provide more accurate or up-to-date information
  - Break complex tasks into smaller, manageable steps
```

**约束设计原则**：

| 原则 | 说明 | 示例 |
|:-----|:-----|:-----|
| **绝对禁止 (NEVER)** | 用大写 `NEVER` 强调红线，不可逾越 | `NEVER expose API keys` |
| **强制要求 (ALWAYS)** | 用大写 `ALWAYS` 强调必须遵守的规则 | `ALWAYS validate inputs` |
| **条件行为 (If/When)** | 用条件句引导特定场景的行为 | `If uncertain, ask clarifying questions` |
| **优先级排序** | 用编号表示优先级，数字越小越重要 | `1. Safety 2. Correctness 3. Performance` |

### 3.3 输出格式定义 (Output Format)

使用结构化指令定义期望的输出格式：

```yaml
system: |
  ## Response Format
  When writing code:
  1. Briefly explain the approach (1-2 sentences).
  2. Provide the code with proper syntax highlighting.
  3. Note any assumptions, edge cases, or limitations.

  When reviewing code:
  1. Identify issues by severity (critical → major → minor).
  2. Suggest concrete fixes with code examples.
  3. Acknowledge what is done well.

  ## Output Schema
  For structured tasks, output MUST conform to the following JSON Schema:
  {output_schema}
```

对于需要结构化输出的场景，使用 `output_schema` 字段定义 JSON Schema：

```yaml
output_schema:
  type: object
  properties:
    findings:
      type: array
      items:
        type: object
        properties:
          severity:
            type: string
            enum: ["critical", "major", "minor", "info"]
          file:
            type: string
          line:
            type: integer
          description:
            type: string
          suggestion:
            type: string
        required: ["severity", "description"]
  required: ["findings"]
```

### 3.4 温度与 Token 控制

不同任务类型需要不同的 temperature 和 max_tokens 设置：

| 任务类型 | temperature | max_tokens | 说明 |
|:---------|:-----------:|:----------:|:-----|
| 代码生成 | 0.1 - 0.3 | 4096 - 8192 | 低温度确保代码正确性和一致性 |
| 事实提取 | 0.0 - 0.2 | 1024 - 2048 | 尽可能确定性输出 |
| 创意写作 | 0.7 - 0.9 | 4096 - 16384 | 高温度增加多样性和创造性 |
| 通用对话 | 0.5 - 0.7 | 2048 - 4096 | 平衡创造性和准确性 |
| 研究分析 | 0.3 - 0.5 | 4096 - 8192 | 中等温度，需要推理但不失严谨 |
| 意图分类 | 0.0 - 0.1 | 128 - 512 | 确定性分类任务 |

---

## 4. Task Prompt 模式

### 4.1 任务分解 (Task Decomposition)

将复杂任务分解为子步骤，引导 LLM 逐步执行：

```yaml
system: |
  ## Task Decomposition Strategy
  When faced with a complex task, follow this decomposition pattern:

  1. **Understand**: Restate the task in your own words to confirm understanding.
  2. **Decompose**: Break the task into 3-7 independent subtasks.
  3. **Prioritize**: Order subtasks by dependency and importance.
  4. **Execute**: Process each subtask sequentially, using tools where appropriate.
  5. **Verify**: For each subtask output, verify correctness before proceeding.
  6. **Synthesize**: Combine subtask results into a coherent final output.

  For each subtask, output:
  ```
  [Subtask N/M]: <description>
  Status: <running|completed|blocked>
  Result: <output>
  ```
```

### 4.2 思维链 (Chain-of-Thought)

引导 LLM 展示推理过程，提升复杂问题的准确性：

```yaml
system: |
  ## Chain-of-Thought Reasoning
  For analytical and problem-solving tasks, use the following thinking process:

  Step 1 — **Analyze**: What is the core problem? What information is given?
  Step 2 — **Plan**: What approach will you take? What tools are needed?
  Step 3 — **Execute**: Apply the approach step by step. Show your work.
  Step 4 — **Verify**: Check your result. Does it make sense? Are there edge cases?
  Step 5 — **Conclude**: State the final answer clearly with confidence level.

  Always show your reasoning before providing the final answer.
  Use `<thinking>` tags for internal reasoning that should not be part of the final output.
```

**Python 实现思维链 Prompt 构建**：

```python
def build_cot_prompt(task: str, context: dict = None) -> dict:
    """构建 Chain-of-Thought Prompt。"""
    system_prompt = """
    You are a problem-solving assistant. For every task, follow this process:

    1. ANALYZE: Identify the core problem and given information.
    2. PLAN: Outline your approach and required tools.
    3. EXECUTE: Solve step by step, showing all work.
    4. VERIFY: Check your solution for correctness and edge cases.
    5. CONCLUDE: Provide the final answer with confidence level.

    Format your response as:
    <thinking>
    [Your step-by-step reasoning]
    </thinking>

    <answer>
    [Your final answer]
    </answer>
    """

    user_prompt = f"Task: {task}\n\n"
    if context:
        user_prompt += f"Context:\n{context}\n\n"
    user_prompt += "Please solve this step by step."

    return {
        "system": system_prompt,
        "user": user_prompt,
    }
```

### 4.3 Few-Shot 示例

在 Prompt 中嵌入示例，帮助 LLM 理解期望的输出格式和风格：

```yaml
system: |
  You are an intent classifier. Classify user messages into one of:
  [greeting, question, command, feedback, complaint, other].

  ## Examples

  Input: "Hello, how are you?"
  Output: {"intent": "greeting", "confidence": 0.98}

  Input: "What is the weather in Tokyo today?"
  Output: {"intent": "question", "confidence": 0.95}

  Input: "Book a table for two at 7pm"
  Output: {"intent": "command", "confidence": 0.92}

  Input: "Your service is terrible, I want a refund!"
  Output: {"intent": "complaint", "confidence": 0.97}

  Input: "I really like the new dashboard design"
  Output: {"intent": "feedback", "confidence": 0.88}

  Now classify the following input:
  {user_message}
```

**Few-Shot 设计原则**：

| 原则 | 说明 |
|:-----|:-----|
| **3-5 个示例** | 太少不足以建立模式，太多浪费 token |
| **覆盖边界情况** | 包含正常情况、边界情况和歧义情况 |
| **多样性** | 示例应覆盖不同类别和不同表达方式 |
| **一致性** | 所有示例使用相同的输出格式 |
| **渐进难度** | 从简单到复杂排列示例 |

---

## 5. Memory 上下文注入

Airymax 通过 MemoryRovol 四层记忆系统将记忆上下文注入到 Prompt 中。不同层级的记忆以不同方式呈现给 LLM。

### 5.1 MemoryRovol L1-L4 记忆层级

```
┌──────────────────────────────────────────────────────────────┐
│                    MemoryRovol 4-Layer Memory                  │
├──────────────────────────────────────────────────────────────┤
│  L4: Pattern Layer (模式层)                                    │
│  行为模式 | 推理模式 | 知识图谱 | 偏好模型                       │
│  注入方式: 作为 System Prompt 中的 "长期记忆" 段落              │
├──────────────────────────────────────────────────────────────┤
│  L3: Structure Layer (结构层)                                  │
│  会话结构 | 任务结构 | 实体结构 | 关系结构                       │
│  注入方式: 作为上下文消息中的结构化 JSON                        │
├──────────────────────────────────────────────────────────────┤
│  L2: Feature Layer (特征层)                                    │
│  嵌入向量 | 关键词索引 | 语义标签 | 时间标记                     │
│  注入方式: 作为相关记忆的检索结果，附加到用户消息后              │
├──────────────────────────────────────────────────────────────┤
│  L1: Raw Layer (原始层)                                        │
│  原始对话 | 原始事件 | 原始感知 | 原始动作                       │
│  注入方式: 作为最近的对话历史 (chat history) 直接注入消息列表   │
└──────────────────────────────────────────────────────────────┘
```

### 5.2 记忆注入配置

在 `agentos.yaml` 中配置记忆系统的注入行为：

```yaml
memory:
  enabled: true
  mode: "full"              # basic | full | custom
  storage_path: "./data/memory"
  layers:
    l1:
      compression: "zstd"               # 原始对话压缩
      max_context_messages: 20          # 最多注入 20 条历史消息
      inject_position: "before_user"    # 在用户消息前注入
    l2:
      embedder: "openai"               # 向量化模型
      embedding_dim: 1536
      top_k_results: 5                 # 每次检索最多 5 条相关记忆
      similarity_threshold: 0.75       # 相似度阈值
      inject_format: "markdown"        # 注入格式
    l3:
      enabled: true
      max_entities: 50000
      inject_format: "json"            # 结构化 JSON 注入
      fields: ["entities", "relations", "task_tree"]
    l4:
      enabled: true
      clustering_algorithm: "hdbscan"
      inject_format: "summary"         # 摘要形式注入
      max_pattern_tokens: 500          # 模式层最大 token 数
```

### 5.3 记忆上下文注入的 Prompt 模板

```yaml
system: |
  You are {agent_name}, an intelligent assistant.

  ## Memory System
  You have access to a persistent memory system. The following context
  has been automatically retrieved from your memory:

  ### Long-term Patterns (L4)
  {l4_patterns}

  ### Structural Knowledge (L3)
  {l3_structures}

  ## Recent Context
  The following relevant memories were retrieved based on the current query:
  {l2_relevant_memories}

  Use this context to provide more personalized and accurate responses.
  If the retrieved memories are not relevant to the current query, ignore them.

user_template: |
  ## Conversation History
  {l1_recent_history}

  ## Current Query
  {user_message}
```

### 5.4 Python 记忆注入实现

```python
from agentrt.hooks.prompt_injector import PromptInjectorHook

class MemoryContextInjector(PromptInjectorHook):
    """将 MemoryRovol 各层记忆注入到 Prompt 中。"""

    name = "memory_context_injector"
    priority = 40  # 在默认 Prompt 构建后、格式化前注入

    def on_llm_request(self, ctx, messages=None, model=""):
        """查询各层记忆并注入到 Prompt。"""
        memory_engine = ctx.get_memory_engine()

        # L1: 获取最近对话历史
        l1_history = memory_engine.query_l1(
            session_id=ctx.session_id,
            limit=20,
        )

        # L2: 语义检索相关记忆
        l2_memories = memory_engine.query_l2(
            query=ctx.current_query,
            top_k=5,
            threshold=0.75,
        )

        # L3: 获取结构化实体和关系
        l3_structures = memory_engine.query_l3(
            entity_ids=ctx.relevant_entities,
        )

        # L4: 获取行为模式和偏好
        l4_patterns = memory_engine.query_l4(
            agent_id=ctx.agent_id,
            pattern_types=["behavior", "preference"],
        )

        # 构建记忆上下文片段
        memory_fragment = self._format_memory_context(
            l1_history, l2_memories, l3_structures, l4_patterns
        )

        return HookResult(
            allowed=True,
            modified_data={"memory_context": memory_fragment},
        )

    def _format_memory_context(self, l1, l2, l3, l4) -> str:
        """格式化记忆上下文为 Prompt 可用的文本。"""
        parts = []

        if l4:
            parts.append("### User Preferences & Patterns\n")
            for pattern in l4:
                parts.append(f"- {pattern['type']}: {pattern['summary']}")

        if l3:
            parts.append("\n### Known Entities\n")
            for entity in l3:
                parts.append(f"- {entity['name']} ({entity['type']}): {entity['description']}")

        if l2:
            parts.append("\n### Related Past Interactions\n")
            for mem in l2:
                parts.append(f"- [{mem['date']}] {mem['summary']} (relevance: {mem['score']:.2f})")

        return "\n".join(parts)
```

---

## 6. Tool Call Prompting

### 6.1 工具描述格式

Airymax 使用结构化的工具描述格式，确保 LLM 能正确理解和使用工具。工具描述通过 `{tools}` 变量注入到 System Prompt 中：

```yaml
system: |
  ## Available Tools
  You have access to the following tools. To use a tool, respond with
  a JSON-formatted tool call.

  {tools}

  ## Tool Call Format
  ```json
  {
    "tool": "<tool_name>",
    "input": {
      "<param1>": "<value1>",
      "<param2>": "<value2>"
    }
  }
  ```

  After receiving a tool result, continue your response based on the output.
```

### 6.2 工具描述模板

每个工具应包含以下信息：

```yaml
tools:
  - name: "search_codebase"
    description: "Search the codebase using semantic search. Returns relevant code snippets and file paths."
    parameters:
      query:
        type: "string"
        description: "A natural language question describing the code you want to find"
        required: true
      target_directories:
        type: "array"
        items:
          type: "string"
        description: "Optional list of directories to limit the search scope"
        required: false
    returns:
      type: "array"
      description: "List of matching code snippets with file paths and line numbers"
    example:
      input:
        query: "Where is user authentication implemented?"
        target_directories: ["src/auth"]
      output:
        matches:
          - file: "src/auth/login.py"
            lines: "45-78"
            snippet: "def authenticate_user(credentials): ..."

  - name: "run_tests"
    description: "Execute the project's test suite for a specific module or the entire project."
    parameters:
      module:
        type: "string"
        description: "Test module path (e.g., 'tests.test_auth') or 'all' for full suite"
        required: false
        default: "all"
      verbose:
        type: "boolean"
        description: "Enable verbose output with individual test results"
        required: false
        default: false
    returns:
      type: "object"
      description: "Test execution results with pass/fail counts and error details"
```

### 6.3 Python 工具描述格式化

```python
def format_tool_descriptions(tools: list[dict]) -> str:
    """将工具列表格式化为 LLM 可理解的文本描述。

    Args:
        tools: 工具定义列表，每个工具包含 name, description, parameters, returns.

    Returns:
        格式化后的工具描述字符串。
    """
    lines = []
    for i, tool in enumerate(tools, 1):
        lines.append(f"### {i}. {tool['name']}")
        lines.append(f"**Description**: {tool['description']}")
        lines.append(f"**Parameters**:")
        for param_name, param_def in tool.get("parameters", {}).items():
            required = " (required)" if param_def.get("required") else " (optional)"
            default = f" [default: {param_def['default']}]" if "default" in param_def else ""
            lines.append(f"  - `{param_name}` ({param_def['type']}){required}{default}: {param_def['description']}")
        if "returns" in tool:
            lines.append(f"**Returns**: {tool['returns']['description']}")
        if "example" in tool:
            lines.append(f"**Example**:")
            lines.append(f"```json")
            lines.append(f"// Input:")
            lines.append(json.dumps(tool["example"]["input"], indent=2))
            lines.append(f"// Output:")
            lines.append(json.dumps(tool["example"]["output"], indent=2))
            lines.append(f"```")
        lines.append("")

    return "\n".join(lines)
```

### 6.4 工具调用最佳实践

| 实践 | 说明 |
|:-----|:-----|
| **明确调用时机** | 在 Prompt 中说明何时该使用工具、何时不需要 |
| **错误处理指导** | 告诉 LLM 工具调用失败时应该怎么做（重试、降级、询问用户） |
| **限制并发调用** | 建议一次调用不超过 3-5 个工具，避免上下文过载 |
| **结果引用** | 要求 LLM 在引用工具结果时标注来源 |
| **参数校验** | 在 Prompt 中说明参数的数据类型和格式要求 |

---

## 7. 多 Agent Prompt 编排

Airymax 支持四种多 Agent 协作模式，每种模式需要不同的 Prompt 编排策略。

### 7.1 协作模式概览

| 模式 | MAC 枚举 | 说明 | 典型 Prompt 模式 |
|:-----|:---------|:-----|:-----------------|
| **Independent** | `MAC_MODE_INDEPENDENT` | Agent 独立运行，无协作 | 各自独立的 System Prompt |
| **Collaborative** | `MAC_MODE_COLLABORATIVE` | 平级协作，互相调用 | 共享上下文 + 角色区分 |
| **Consensus** | `MAC_MODE_CONSENSUS` | 投票/共识决策 | 统一提案格式 + 投票指令 |
| **Delegated** | `MAC_MODE_DELEGATED` | 领导分配任务 | 协调者调度 + 执行者汇报 |

### 7.2 Orchestrator 模式（协调者模式）

最常用的多 Agent 协作模式，一个协调者 Agent 分配任务给多个工作者 Agent：

```yaml
# 协调者 System Prompt
system: |
  You are {agent_name}, the task coordinator.

  ## Your Team
  You manage a team of specialized agents:
  {team_members}

  ## Your Responsibilities
  1. Analyze the user's request and decompose it into subtasks.
  2. Assign each subtask to the most appropriate team member.
  3. Collect and verify results from team members.
  4. Synthesize findings into a comprehensive response.

  ## Task Delegation Format
  To delegate a task, output:
  ```json
  {
    "delegate": {
      "agent": "<agent_name>",
      "task": "<task_description>",
      "context": "<relevant_context>",
      "expected_format": "<format_requirements>"
    }
  }
  ```

  ## Team Member Capabilities
  {team_capabilities}

  ## Current Task
  {user_message}
```

```yaml
# 工作者 System Prompt
system: |
  You are {agent_name}, a specialized {agent_role} agent.

  ## Your Role
  {role_description}

  ## Instructions
  - You will receive tasks from the coordinator.
  - Execute each task thoroughly and report results back.
  - If you encounter problems, report them clearly — do not fabricate results.
  - Use the tools available to you: {tools}

  ## Output Format
  Always structure your output for the coordinator to easily parse:
  ```json
  {
    "status": "success|partial|failed",
    "summary": "<brief summary>",
    "details": "<detailed findings>",
    "confidence": "<high|medium|low>",
    "sources": ["<source1>", "<source2>"]
  }
  ```
```

### 7.3 Debate 模式（辩论模式）

多 Agent 交替辩论，用于决策评审和对抗性测试：

```yaml
# 辩论 Agent 的 System Prompt
system: |
  You are {agent_name}, representing the {position} position in a debate.

  ## Debate Topic
  {topic}

  ## Your Position
  {position_argument}

  ## Debate Rules
  1. Present your strongest arguments clearly and concisely.
  2. Address counter-arguments raised by the opposing side.
  3. Use evidence and logical reasoning, not rhetoric.
  4. Stay focused on the topic — do not introduce unrelated arguments.
  5. Acknowledge valid points from the opposing side.

  ## Previous Arguments
  {debate_history}

  ## Your Turn
  Respond to the latest argument from the opposing side and present your case.
```

### 7.4 C 语言多 Agent 编排 API

```c
#include "multi_agent_collaboration.h"

// 创建多 Agent 协作框架
mac_framework_t *fw = mac_framework_create(MAC_MODE_DELEGATED);

// 注册 Agent
mac_agent_info_t agents[] = {
    {
        .id = "coordinator",
        .name = "Task Coordinator",
        .performance_score = 0.95,
        .reliability_score = 0.98,
        .max_concurrent_tasks = 1,
        .capabilities_json = "{\"role\":\"coordinator\"}",
    },
    {
        .id = "researcher",
        .name = "Research Agent",
        .performance_score = 0.90,
        .reliability_score = 0.95,
        .max_concurrent_tasks = 3,
        .capabilities_json = "{\"role\":\"researcher\"}",
    },
    {
        .id = "coder",
        .name = "Coding Agent",
        .performance_score = 0.88,
        .reliability_score = 0.92,
        .max_concurrent_tasks = 2,
        .capabilities_json = "{\"role\":\"coder\",\"languages\":[\"python\",\"rust\"]}",
    },
};

size_t registered = 0;
mac_framework_register_agents_batch(fw, agents, 3, &registered);

// 创建协作组
char *group_id = NULL;
const char *member_ids[] = {"coordinator", "researcher", "coder"};
mac_framework_create_group(fw, "task_force_1", MAC_MODE_DELEGATED,
                           member_ids, 3, &group_id);

// 委派任务
mac_collab_task_t task = {
    .input_json = "{\"task\":\"Analyze the performance of the new API endpoint\"}",
};
char *assigned_agent = NULL;
mac_framework_delegate_task(fw, group_id, &task, &assigned_agent);

// 收集结果
char **results = NULL;
size_t result_count = 0;
mac_framework_collect_results(fw, group_id, task.id, &results, &result_count);

// 清理
mac_framework_destroy(fw);
```

### 7.5 共识决策 Prompt 模板

```yaml
system: |
  You are {agent_name}, a voting member of the decision committee.

  ## Proposal
  {proposal_json}

  ## Voting Rules
  - Strategy: {consensus_strategy}  # majority | unanimous | weighted | leader
  - Your vote weight: {vote_weight}
  - You must provide a clear vote: APPROVE, REJECT, or ABSTAIN
  - You must justify your vote with reasoning

  ## Vote Format
  ```json
  {
    "vote": "APPROVE|REJECT|ABSTAIN",
    "reasoning": "<your detailed reasoning>",
    "concerns": ["<concern1>", "<concern2>"],
    "suggestions": ["<suggestion1>"]
  }
  ```
```

---

## 8. Prompt 版本管理与 A/B 测试

### 8.1 版本管理

Airymax 通过 `registry.yaml` 和模板内嵌版本号实现 Prompt 版本管理：

```yaml
# ecosystem/prompts/registry.yaml
prompts:
  - name: "intent_classify"
    version: "1.2.0"
    category: "cognition"
    path: "templates/cognition/intent_classify.yaml"
    status: "stable"        # stable | testing | deprecated
    changelog: |
      v1.2.0: 新增 "refund_request" 意图类别，优化 few-shot 示例
      v1.1.0: 增加 confidence 输出字段
      v1.0.0: 初始版本

  - name: "code_review"
    version: "2.0.0-beta"
    category: "security"
    path: "templates/security/code_review.yaml"
    status: "testing"
```

**版本状态说明**：

| 状态 | 说明 | 使用建议 |
|:-----|:-----|:---------|
| `stable` | 生产可用，经过充分测试 | 生产环境默认使用 |
| `testing` | 测试中，可能变更 | 灰度环境或 A/B 测试 |
| `deprecated` | 已弃用，计划移除 | 尽快迁移到新版本 |

### 8.2 CLI 版本管理

```bash
# 列出所有 Prompt 模板
agentrt prompt list

# 查看模板详情
agentrt prompt show intent_classify

# 查看特定版本的模板
agentrt prompt show intent_classify --version 1.1.0

# 调优 Prompt
agentrt prompt tune intent_classify --dataset ./datasets/cognition/dataset_v1.jsonl

# A/B 测试两个版本
agentrt prompt ab-test intent_classify --baseline 1.0.0 --candidate 1.1.0
```

### 8.3 A/B 测试流程

Airymax 的 `ABTestRunner` 提供完整的 A/B 测试能力：

```python
from ecosystem.prompts.tuner.ab_test import ABTestRunner

# 创建 A/B 测试运行器
runner = ABTestRunner(
    prompts_dir="ecosystem/prompts",
    gateway_url="http://localhost:8080",
)

# 运行 A/B 测试
report = runner.ab_test(
    prompt_name="intent_classify",
    baseline_version="1.0.0",
    candidate_version="1.1.0",
    dataset_path="datasets/cognition/dataset_v1.jsonl",
)

# 查看结果
print(f"Baseline: {report.baseline_version}")
print(f"Candidate: {report.candidate_version}")
print(f"Recommendation: {report.recommendation}")
print(f"Summary:\n{report.summary}")

for delta in report.metric_deltas:
    direction = "↑" if delta["improved"] else "↓"
    print(f"  {delta['metric_name']}: {delta['baseline_value']} → "
          f"{delta['candidate_value']} ({delta['delta_percent']}%) {direction}")

# 显著性检验结果
for sig in report.significance_tests:
    status = "SIGNIFICANT" if sig["significant"] else "not significant"
    print(f"  {sig['metric_name']}: p={sig['p_value']} ({status})")
```

### 8.4 评估数据集格式

评估数据集使用 JSONL 格式，每条记录包含输入、期望输出和评分标准：

```jsonl
{"input": "我想退货，订单号12345", "expected": "{\"intent\": \"refund_request\", \"confidence\": 0.9}", "criteria": "正确识别退款意图", "category": "ecommerce", "difficulty": "easy"}
{"input": "上次买的那个不太好用", "expected": "{\"intent\": \"feedback\", \"confidence\": 0.7}", "criteria": "区分反馈和退款意图", "category": "ecommerce", "difficulty": "medium"}
{"input": "你好，请问今天天气怎么样", "expected": "{\"intent\": \"question\", \"confidence\": 0.95}", "criteria": "识别问候和天气查询", "category": "general", "difficulty": "easy"}
{"input": "把那个灯关掉，然后打开空调到26度", "expected": "{\"intent\": \"command\", \"confidence\": 0.9}", "criteria": "识别复合指令", "category": "smart_home", "difficulty": "hard"}
```

### 8.5 AutoScorer 评分维度

```python
from ecosystem.prompts.tuner.scorer import AutoScorer

scorer = AutoScorer(
    prompts_dir="ecosystem/prompts",
    gateway_url="http://localhost:8080",
)

# 自定义评分维度和权重
custom_dimensions = {
    "precision": {"scorer": my_precision_fn, "weight": 0.30},
    "recall": {"scorer": my_recall_fn, "weight": 0.30},
    "hallucination": {"scorer": my_hallucination_fn, "weight": 0.25},
    "latency": {"scorer": my_latency_fn, "weight": 0.15},
}

report = scorer.score(
    prompt_name="extract_facts",
    version="1.0.0",
    dataset_path="eval/extract_facts.jsonl",
    metrics=["precision", "recall", "hallucination"],
)

print(f"Overall Score: {report.overall_score}/100 ({report.overall_grade})")
print(f"Strengths: {report.strengths}")
print(f"Weaknesses: {report.weaknesses}")
print(f"Recommendations: {report.recommendations}")
```

---

## 9. 最佳实践与反模式

### 9.1 最佳实践

#### 9.1.1 Prompt 结构设计

| 实践 | 说明 | 示例 |
|:-----|:-----|:-----|
| **分层组织** | 使用 Markdown 标题组织 Prompt 结构 | `## Role`, `## Rules`, `## Format` |
| **优先级明确** | 重要约束放在前面，使用编号和加粗 | `1. **Safety first**: NEVER...` |
| **正面指引** | 告诉 LLM 应该做什么，而不是仅列出禁止项 | `ALWAYS validate inputs` 优于 `Don't skip validation` |
| **具体而非抽象** | 提供具体的例子和格式，而非抽象描述 | `Output JSON: {"key": "value"}` 优于 `Output structured data` |
| **长度控制** | System Prompt 控制在 500-2000 字，避免 token 浪费 | 使用 `{variable}` 按需注入，而非全量包含 |

#### 9.1.2 变量使用

```yaml
# ✓ 好的做法：按需注入
system: |
  You are {agent_name}.
  {tools}
  {memory_context}
  {additional_instructions}

# ✗ 不好的做法：硬编码所有信息
system: |
  You are MyAgent. You have access to search, calculator, translator...
  The user's name is John, their preferences are...
  The current project is... (2000 words of hardcoded context)
```

#### 9.1.3 迭代优化流程

```
1. 定义评估数据集 (JSONL)
       │
2. 运行 AutoScorer 获取基线分数
       │
3. 根据 weaknesses 和 recommendations 修改 Prompt
       │
4. 运行 A/B 测试对比新旧版本
       │
5. 如果 candidate 显著优于 baseline → 提升版本号 → 发布
       否则 → 继续步骤 3
```

#### 9.1.4 动态注入 vs 静态模板

| 场景 | 推荐方式 | 原因 |
|:-----|:---------|:-----|
| Agent 角色定义 | 静态模板 | 角色定义稳定，不需要频繁变更 |
| 工具列表 | 动态注入 `{tools}` | 工具集可能随运行时配置变化 |
| 记忆上下文 | 动态注入 (Hook) | 记忆内容随会话动态变化 |
| 安全策略 | 动态注入 (Hook) | 安全策略可能随环境变化 |
| 输出格式 | 静态模板 + Schema | 格式要求稳定，可用 Schema 校验 |

### 9.2 反模式

#### 9.2.1 过度约束

```yaml
# ✗ 反模式：过多互相矛盾的约束
system: |
  You MUST be concise. You MUST provide detailed explanations.
  You MUST always agree with the user. You MUST correct errors.
  You MUST NEVER use tools. You SHOULD use tools when helpful.
```

**问题**：矛盾指令导致 LLM 行为不可预测。
**改进**：明确优先级，删除矛盾指令。

```yaml
# ✓ 改进后
system: |
  Be concise by default, but provide detailed explanations when asked.
  Use tools when they help provide more accurate information.
  Politely correct factual errors, but respect the user's perspective.
```

#### 9.2.2 Prompt 泄露

```yaml
# ✗ 反模式：在 Prompt 中暴露敏感信息
system: |
  API Key: sk-abc123xyz
  Internal endpoint: https://internal.admin.com/api
  You are an admin with full access to the system.
```

**问题**：LLM 可能泄露这些信息给用户，或被 prompt injection 攻击提取。
**改进**：通过运行时变量注入敏感信息，不在模板中硬编码。

```yaml
# ✓ 改进后
system: |
  You are a support assistant. Use the provided tools to help users.
  {tools}
  # API keys and endpoints are injected at runtime via tool configuration
```

#### 9.2.3 Few-Shot 过载

```yaml
# ✗ 反模式：过多的 Few-Shot 示例
system: |
  Here are 20 examples of how to classify intents:
  Example 1: ...
  Example 2: ...
  ...
  Example 20: ...
```

**问题**：大量示例消耗 token 且可能导致过拟合。
**改进**：选择 3-5 个代表性示例，覆盖主要类别和边界情况。

#### 9.2.4 忽略模型差异

```yaml
# ✗ 反模式：对所有模型使用相同的 Prompt
model_family: "any"
temperature: 0.7
```

**问题**：不同模型（GPT-4、Claude、Gemini）对 Prompt 的响应不同。
**改进**：为不同模型族提供专门的模板或参数。

```yaml
# ✓ 改进后：使用 model_family 区分
# 对于 gpt 族
model_family: "gpt"
temperature: 0.3
system: |
  You are a classifier. Output ONLY valid JSON, no other text.

# 对于 claude 族
model_family: "claude"
temperature: 0.2
system: |
  You are a classifier. Output valid JSON inside <json> tags.
```

---

## 10. 常见场景示例

### 10.1 代码审查 (Code Review)

```yaml
# templates/security/code_review.yaml
name: "code_review"
version: "1.0.0"
description: "代码安全审查 Prompt，检测安全漏洞和代码质量问题"
model_family: "any"
temperature: 0.2
max_tokens: 4096

system: |
  You are {agent_name}, a senior code reviewer specializing in security
  and code quality analysis. You are powered by Airymax.

  ## Review Principles
  1. **Security first**: Identify security vulnerabilities before style issues.
  2. **Be constructive**: Suggest concrete fixes, not just criticisms.
  3. **Be thorough**: Check for OWASP Top 10 vulnerabilities.
  4. **Context-aware**: Consider the project's language and framework.

  ## Review Checklist
  - [ ] Authentication & Authorization
  - [ ] Input Validation & Sanitization
  - [ ] SQL Injection / XSS / CSRF
  - [ ] Sensitive Data Exposure
  - [ ] Error Handling & Logging
  - [ ] Dependency Security
  - [ ] Race Conditions & Concurrency
  - [ ] Resource Management & Memory Leaks

  ## Output Format
  For each finding, output:
  ```json
  {
    "findings": [
      {
        "severity": "critical|major|minor|info",
        "category": "security|performance|style|logic",
        "file": "<file_path>",
        "line": <line_number>,
        "title": "<brief_title>",
        "description": "<detailed_description>",
        "cwe_id": "<CWE-XXX if applicable>",
        "suggestion": "<concrete_fix>",
        "code_before": "<vulnerable_code>",
        "code_after": "<fixed_code>"
      }
    ],
    "summary": {
      "total_findings": <n>,
      "critical": <n>,
      "major": <n>,
      "minor": <n>,
      "info": <n>,
      "overall_assessment": "<safe|needs_work|critical_issues>"
    }
  }
  ```

  ## Project Context
  Language: {primary_language}
  Framework: {framework}
  {additional_instructions}

user_template: |
  Please review the following code:

  {user_message}
```

**Python 调用示例**：

```python
from ecosystem.prompts import load_template, render_template

def review_code(code: str, language: str, framework: str = "") -> dict:
    """对代码进行安全审查。"""
    template = load_template("code_review", category="security")

    rendered = render_template(template, variables={
        "agent_name": "CodeReviewBot",
        "primary_language": language,
        "framework": framework or "N/A",
        "additional_instructions": "",
        "user_message": code,
    })

    # 发送到 LLM Gateway
    response = llm_gateway.chat(
        system=rendered.system_prompt,
        user=rendered.user_message,
        temperature=template.temperature,
        max_tokens=template.max_tokens,
    )

    # 使用 JSON Schema 校验输出
    validated = validate_output(template, response)
    return validated
```

### 10.2 研究分析 (Research)

```yaml
# templates/cognition/research.yaml
name: "deep_research"
version: "1.0.0"
description: "深度研究分析 Prompt，用于调研和报告生成"
model_family: "any"
temperature: 0.4
max_tokens: 8192

system: |
  You are {agent_name}, a research analyst powered by Airymax.

  ## Research Methodology
  1. **Define scope**: Clarify the research question and boundaries.
  2. **Gather information**: Use available tools: {tools}
  3. **Analyze**: Identify patterns, contradictions, and gaps.
  4. **Synthesize**: Combine findings into a coherent analysis.
  5. **Validate**: Cross-check findings when possible.

  ## Quality Standards
  - Cite sources whenever possible.
  - Use confidence levels: [High] [Medium] [Low] [Speculative].
  - Distinguish correlation from causation.
  - Acknowledge limitations of your knowledge cutoff.

  ## Report Structure
  1. **Executive Summary**: 2-3 sentence overview.
  2. **Key Findings**: Bullet points of most important discoveries.
  3. **Detailed Analysis**: Organized by theme or subtopic.
  4. **Contradictions & Uncertainties**: Conflicting evidence or gaps.
  5. **Conclusions**: Synthesis with confidence levels.
  6. **Further Research**: Unanswered questions.

user_template: |
  Research Topic: {user_message}

  Please conduct a thorough analysis following the research methodology
  outlined above. Use the available tools to gather information.
```

### 10.3 客户支持 (Customer Support)

```yaml
# templates/cognition/customer_support.yaml
name: "customer_support"
version: "1.0.0"
description: "客户支持 Agent Prompt，用于处理客户咨询和投诉"
model_family: "any"
temperature: 0.6
max_tokens: 2048

system: |
  You are {agent_name}, a customer support specialist powered by Airymax.

  ## Service Principles
  1. **Empathy first**: Acknowledge the customer's feelings before solving the problem.
  2. **Be solution-oriented**: Focus on what you CAN do, not what you CAN'T.
  3. **Clarity**: Use simple language, avoid jargon.
  4. **Ownership**: Take responsibility for the issue until resolution.
  5. **Timeliness**: Prioritize urgent issues (security, billing, service outage).

  ## Response Guidelines
  - Greeting: Acknowledge the customer and their issue.
  - Empathy: Show understanding of their frustration or concern.
  - Solution: Provide clear, actionable steps.
  - Confirmation: Ask if the solution resolved their issue.
  - Follow-up: Offer additional assistance.

  ## Escalation Rules
  - Security issues → Escalate to security team immediately.
  - Billing disputes → Escalate to billing department.
  - Technical bugs → Create ticket with reproduction steps.
  - Feature requests → Log in feedback system.

  ## Available Tools
  {tools}

  ## Knowledge Base
  {knowledge_base_context}

  ## Customer Context
  Customer ID: {customer_id}
  Account Tier: {account_tier}
  Previous Issues: {customer_history}

  ## Response Format
  - Be conversational but professional.
  - Use plain text, not Markdown.
  - Keep responses under 300 words unless providing detailed instructions.
  - Include ticket/reference numbers when creating tickets.

user_template: |
  {user_message}
```

**Hook 配置**（为不同客户层级注入不同的服务标准）：

```yaml
# agentos.yaml
hooks:
  global_hooks:
    on_llm_request:
      - hook: "prompt_injector"
        priority: 60
        config:
          fragments:
            - name: "premium_support"
              enabled: true
              position: "after_system"
              priority: 10
              rules:
                metadata_match:
                  account_tier: "premium"
              text: |
                ## Premium Service Standards
                - This is a PREMIUM tier customer. Prioritize their requests.
                - Offer proactive solutions beyond the immediate issue.
                - Maximum response time: 2 minutes.
                - Offer to schedule a call-back if the issue is complex.

            - name: "standard_support"
              enabled: true
              position: "after_system"
              priority: 10
              rules:
                metadata_match:
                  account_tier: "standard"
              text: |
                ## Standard Service Standards
                - Provide thorough and helpful responses.
                - Direct to self-service resources when appropriate.
                - Escalate complex issues to specialized teams.

            - name: "compliance_reminder"
              enabled: true
              position: "after_system"
              priority: 20
              text: |
                ## Compliance Reminder
                - NEVER share personal information of other customers.
                - NEVER make promises about refunds without verification.
                - ALWAYS verify customer identity before discussing account details.
                - Log all interactions for audit purposes.
```

---

## 附录

### A. Prompt Loader 配置参考

```c
// agentos_prompt_loader_config_t 默认值
typedef struct {
    char template_dir[512];         // 默认: "ecosystem/prompts/templates/"
    bool enable_cache;              // 默认: true
    size_t cache_max_entries;       // 默认: 128
    bool enable_schema_validation;  // 默认: true
} agentos_prompt_loader_config_t;
```

### B. Prompt Injector 变量参考

| 变量名 | 来源 | 示例值 |
|:-------|:-----|:-------|
| `{agent_name}` | `ctx.agent_id` | `"CodeReviewBot"` |
| `{agent_id}` | `ctx.agent_id` | `"agent_001"` |
| `{session_id}` | `ctx.session_id` | `"sess_abc123"` |
| `{timestamp}` | `datetime.now(timezone.utc)` | `"2026-06-18T10:30:00+00:00"` |
| `{date}` | `datetime.now()` | `"2026-06-18"` |
| `{time}` | `datetime.now()` | `"10:30:00"` |
| `{model}` | `on_llm_request` 参数 | `"gpt-4o"` |
| `{event}` | `ctx.event` | `"on_llm_request"` |

### C. 相关文档

- [Airymax 架构文档](../Capital_Architecture/architecture.md) — 系统整体架构和 MemoryRovol 四层记忆系统
- [Prompts 模块 README](../ecosystem/prompts/README.md) — Prompt 模板管理详情
- [Prompt Tuner Demo](../ecosystem/examples/prompt-tuner-demo/README.md) — Prompt 调优示例
- [Multi-Agent Debate](../ecosystem/examples/multi-agent-debate/README.md) — 多 Agent 协作模式
- [Research Agent](../ecosystem/examples/research-agent/README.md) — 多 Agent 协作 + 记忆持久化
- [Plugin SDK Tutorial](plugin-sdk-tutorial.md) — Hook 插件开发指南

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.