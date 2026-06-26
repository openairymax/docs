# SUBSYSTEM_openlab — 开放实验室（生态贡献平台）

**版本**: v0.1.0  
**状态**: 演进中  
**路径**: `ecosystem/openlab/`  
**负责人**: Team B (生态 + SDK)  
**最后更新**: 2026-06-17

---

## 1. 职责边界

openlab 是 Airymax 的开放实验室，即生态贡献平台。它为社区开发者提供标准化的 Agent、Skill、Strategy 贡献框架，使第三方开发者能够以统一规范贡献可复用的智能体组件。

### 1.1 核心职责

| 职责 | 说明 |
|------|------|
| **Agent 贡献框架** | 7 种专业角色 Agent 的骨架实现（Architect / Backend / Frontend / DevOps / Security / Tester / Product Manager） |
| **Skill 贡献框架** | 社区 Skill 开发规范与骨架（database_skill / browser_skill / github_skill 等） |
| **Strategy 贡献框架** | 规划策略 (planning) 和调度策略 (dispatching) 的社区贡献 |
| **契约规范** | 统一的 Agent 契约 (`contract.json`) 和 Skill 契约格式 |
| **提示词模板** | 标准化的双提示词模板（`system1.md` + `system2.md`） |
| **核心抽象** | `openlab.core.Agent` 基类，定义统一生命周期接口 |

### 1.2 社区 Agent 角色

| Agent | Agent ID | 核心能力 | 默认任务类型 |
|-------|----------|----------|-------------|
| **Architect** | `contrib-architect` | `ARCHITECTURE_DESIGN` | `design` |
| **Backend** | `contrib-backend` | `CODE_GENERATION` | `implement` |
| **Frontend** | `contrib-frontend` | `CODE_GENERATION` | `generate` |
| **DevOps** | `contrib-devops` | `OPTIMIZATION` | `deploy` |
| **Security** | `contrib-security` | `DEBUGGING` | `audit` |
| **Tester** | `contrib-tester` | `TEST_GENERATION` | `test` |
| **Product Manager** | `contrib-product-manager` | `DOCUMENTATION` | `plan` |

### 1.3 不在范围内的职责

- 不负责 Agent 运行时执行（由 coreloopthree 负责）
- 不负责 Agent 市场分发（由 market_d 负责）
- 不负责 LLM 推理（由 llm_d 负责）
- 不负责工具实现（由 tool_d 负责）

---

## 2. 输入/输出接口

### 2.1 Agent 基类 (`openlab.core.Agent`)

```python
from openlab.core import Agent, AgentCapability, AgentContext, AgentStatus, TaskResult

class CustomAgent(Agent):
    def __init__(self, config: Optional[Dict[str, Any]] = None):
        super().__init__(
            agent_id="contrib-custom",
            capabilities={AgentCapability.CUSTOM},
        )
        self.config = config or {}

    async def initialize(self) -> None:
        """初始化 Agent 资源"""
        pass

    async def execute(self, input_data: Dict, context: AgentContext) -> TaskResult:
        """执行 Agent 任务"""
        pass

    async def shutdown(self) -> None:
        """清理 Agent 资源"""
        pass
```

### 2.2 契约规范 (`contract.json`)

每个 Agent 配备标准化的契约文件，核心字段：

| 字段 | 说明 |
|------|------|
| `agent_id` | Agent 唯一标识符 |
| `agent_name` | Agent 显示名称 |
| `description` | 功能描述 |
| `version` | 语义化版本号 |
| `capabilities` | 能力声明列表（含输入/输出 Schema） |
| `performance` | 性能指标 |
| `resources` | 资源需求 |
| `dependencies` | 依赖声明 |
| `security` | 安全配置 |
| `lifecycle` | 生命周期参数 |

### 2.3 提示词模板 (`prompts/`)

| 文件 | 说明 |
|------|------|
| `system1.md` | 主系统提示词，定义 Agent 的核心行为和角色定位 |
| `system2.md` | 辅助系统提示词，提供补充约束和交互规范 |

### 2.4 目录结构

```
ecosystem/openlab/
├── core/                    # 核心抽象（Agent 基类等）
├── contrib/
│   ├── agents/              # 社区 Agent 贡献
│   │   ├── architect/       # 架构师 Agent
│   │   ├── backend/         # 后端开发 Agent
│   │   ├── frontend/        # 前端开发 Agent
│   │   ├── devops/          # DevOps Agent
│   │   ├── security/        # 安全审计 Agent
│   │   ├── tester/          # 测试生成 Agent
│   │   └── product_manager/ # 产品经理 Agent
│   ├── skills/              # 社区 Skill 贡献
│   │   ├── database_skill/
│   │   ├── browser_skill/
│   │   └── github_skill/
│   └── strategies/          # 社区 Strategy 贡献
│       ├── planning/        # 规划策略
│       └── dispatching/     # 调度策略
```

---

## 3. 依赖关系

### 3.1 依赖项

| 依赖 | 类型 | 说明 |
|------|------|------|
| **Python >= 3.10** | 运行时 | Python 运行环境 |
| **openlab.core** | 强依赖 | Agent 基类、核心抽象 |
| **Airymax protocols** | 协议依赖 | JSON-RPC 2.0 协议 |
| **market_d** | 可选依赖 | 通过市场服务分发 Agent |

### 3.2 被依赖方

- **market_d**: 从 openlab 加载社区 Agent 注册信息
- **coreloopthree**: 实例化 openlab Agent 进行任务执行
- **社区开发者**: 基于 openlab 框架开发自定义 Agent

### 3.3 集成架构

```
┌──────────────────────────────────────────┐
│              openlab                      │
│  ┌────────────────────────────────────┐  │
│  │  openlab.core.Agent (基类)          │  │
│  └────────────────────────────────────┘  │
│  ┌──────────┐ ┌──────────┐ ┌─────────┐  │
│  │  Agents  │ │  Skills  │ │Strategies│  │
│  └──────────┘ └──────────┘ └─────────┘  │
└──────────────────┬───────────────────────┘
                   │ JSON-RPC 2.0
                   ▼
┌──────────────────────────────────────────┐
│           Airymax Core Runtime            │
│   (coreloopthree + gateway_d + llm_d)    │
└──────────────────────────────────────────┘
```

---

## 4. 配置参数

### 4.1 Agent 配置

```python
config = {
    "llm_provider": "openai",     # LLM 提供商
    "model": "gpt-4",             # 模型名称
    "temperature": 0.7,           # 温度参数
    "max_tokens": 4096,           # 最大 Token 数
    "timeout": 300,               # 超时时间（秒）
}
```

### 4.2 契约配置

| 参数 | 说明 |
|------|------|
| `version` | 语义化版本号 (MAJOR.MINOR.PATCH) |
| `capabilities[].input_schema` | 输入 JSON Schema |
| `capabilities[].output_schema` | 输出 JSON Schema |
| `performance.avg_latency_ms` | 平均延迟 |
| `performance.max_tokens_per_call` | 每次调用最大 Token |
| `resources.memory_mb` | 内存需求 |
| `resources.cpu_cores` | CPU 需求 |
| `dependencies` | 依赖声明列表 |

---

## 5. 开发指南

### 5.1 创建社区 Agent

```python
import asyncio
from contrib.agents.architect import ArchitectAgent
from openlab.core import AgentContext

async def main():
    architect = ArchitectAgent(config={"llm_provider": "openai"})
    await architect.initialize()

    result = await architect.execute(
        input_data={"task_type": "design", "requirements": {...}},
        context=AgentContext(agent_id="test")
    )
    print(f"Success: {result.success}, Output: {result.output}")

    await architect.shutdown()

asyncio.run(main())
```

### 5.2 多 Agent 协作

```python
from contrib.agents.architect import ArchitectAgent
from contrib.agents.backend import BackendAgent
from contrib.agents.tester import TesterAgent

# 设计 → 实现 → 测试 流水线
design = await architect.execute({"task_type": "design", ...}, context)
impl = await backend.execute({"task_type": "implement", ...}, context)
tests = await tester.execute({"task_type": "test", ...}, context)
```

---

## 6. 相关文档

- [Agent 契约规范](../../Capital_Specifications/agentos_contract/agent_contract.md)
- [Skill 契约规范](../../Capital_Specifications/agentos_contract/skill_contract.md)
- [创建 Agent 指南](../../Capital_Guides/create_agent.md)
- [创建 Skill 指南](../../Capital_Guides/create_skill.md)
- [市场服务子系统](SUBSYSTEM_market_d.md)