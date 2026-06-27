# Airymax 5 分钟快速上手指南

> **P4.3.1** — 5 分钟从零到 Hello World
>
> 预计阅读 / 操作时间：**5 分钟** | 难度：入门级

---

## 1. 环境要求

在开始之前，请确保你的系统满足以下最低要求：

| 依赖项 | 版本要求 | 说明 |
|:-------|:---------|:-----|
| **操作系统** | Ubuntu 22.04+ / macOS 13+ / Windows 11 (WSL2) | 推荐 Linux 环境 |
| **CMake** | ≥ 3.20 | 构建系统 |
| **编译器** | GCC 11+ / Clang 14+ | 支持 C11 和 C++17 |
| **Python** | ≥ 3.10 | SDK 和 OpenLab 需要 |
| **Docker** | 可选 | 容器化部署（推荐） |

### 安装系统依赖（Ubuntu）

```bash
sudo apt update && sudo apt install -y \
    build-essential cmake gcc g++ \
    libssl-dev libsqlite3-dev ninja-build \
    python3 python3-pip
```

---

## 2. 快速安装

### 2.1 克隆仓库并构建

```bash
# 克隆仓库
git clone https://atomgit.com/openairymax/agentos.git
cd agentos

# 源外构建（BAN-33 强制要求）
cmake -B ../AgentRT-build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build ../AgentRT-build --parallel $(nproc)
```

### 2.2 验证安装

```bash
# 运行测试套件
cd ../AgentRT-build && ctest --output-on-failure
```

### 2.3 Docker 快速启动（可选）

```bash
# 构建镜像
docker build -f deploy/docker/Dockerfile -t airymax/agentos:latest .

# 启动容器
docker run -d --name agentrt \
    -p 8080:8080 \
    -v ./config:/app/config \
    airymax/agentos:latest
```

---

## 3. Hello World：启动网关并提交第一个任务

Airymax 通过 **Gateway（网关）** 对外暴露 HTTP 和 WebSocket 接口。以下示例演示如何启动网关、注册一个 Agent 并提交任务。

### 3.1 启动网关

```bash
# 启动 Airymax 网关（默认监听 0.0.0.0:8080）
./agentrt-gateway --host 0.0.0.0 --port 8080
```

启动成功后你会看到类似输出：

```
[INFO] Gateway started on http://0.0.0.0:8080
[INFO] WebSocket endpoint: ws://0.0.0.0:8081
```

### 3.2 注册一个 Hello Agent

在另一个终端中，使用 curl 向网关注册一个简单的 Agent：

```bash
# 注册 Agent
curl -X POST http://localhost:8080/api/v1/agents/register \
  -H "Content-Type: application/json" \
  -d '{
    "name": "hello-agent",
    "version": "1.0.0",
    "model": "gpt-4o-mini",
    "system_prompt": "你是一个友好的助手，请用简洁的中文回答用户的问题。"
  }'
```

### 3.3 提交第一个任务

```bash
# 提交对话任务
curl -X POST http://localhost:8080/api/v1/chat \
  -H "Content-Type: application/json" \
  -d '{
    "agent": "hello-agent",
    "message": "你好，请介绍一下 Airymax"
  }'
```

预期响应：

```json
{
  "jsonrpc": "2.0",
  "result": {
    "agent": "hello-agent",
    "reply": "Airymax 是一个智能体运行时平台，它为驱动 AI Agent 团队提供操作系统级的支持..."
  },
  "id": 1
}
```

---

## 4. SDK 快速上手：用 Python 创建你的第一个 Agent

Airymax 提供多语言 SDK，支持 Python、Go、Rust、TypeScript。以下以 Python SDK 为例展示如何创建一个简单的 Agent。

### 4.1 安装 Python SDK

```bash
pip install agentos
```

### 4.2 编写第一个 Agent

创建一个文件 `hello_agent.py`：

```python
"""hello_agent.py — 你的第一个 Airymax Agent"""

from agentos import Airymax

# 连接 Airymax 运行时
client = Airymax(endpoint="http://localhost:18789", timeout=30)

# 提交一个任务
task = client.submit_task(
    description="请用一句话介绍 Airymax 的核心优势",
    agent_config={
        "name": "hello-agent",
        "model": "gpt-4o-mini",
        "system_prompt": "你是一个专业的 Airymax 技术顾问。"
    }
)

# 等待任务完成并获取结果
result = task.wait(timeout=60)
print(f"Agent 回复: {result.output}")

# 写入记忆
memory_id = client.write_memory(
    content="Airymax 的核心优势：操作系统级架构、安全内生设计、Token 节省约 500%。",
    metadata={"source": "hello_agent", "tag": "intro"}
)
print(f"记忆已写入: {memory_id}")

# 搜索记忆
memories = client.search_memory("Airymax 优势", top_k=3)
for m in memories:
    print(f"  [{m.score:.2f}] {m.memory.content}")

# 关闭连接
client.close()
```

### 4.3 运行

```bash
python hello_agent.py
```

### 4.4 使用异步客户端

对于高并发场景，推荐使用 `AsyncAgentOS`：

```python
"""async_agent.py — 异步 Agent 示例"""

import asyncio
from agentos import AsyncAgentOS

async def main():
    async with AsyncAgentOS(endpoint="http://localhost:18789") as client:
        # 同时提交多个任务
        tasks = [
            client.submit_task("分析市场趋势"),
            client.submit_task("生成周报摘要"),
            client.submit_task("审查代码安全漏洞"),
        ]
        results = await asyncio.gather(*tasks)

        for i, result in enumerate(results):
            print(f"任务 {i + 1}: {result.output[:80]}...")

asyncio.run(main())
```

### 4.5 使用 Framework 层构建完整应用

```python
"""framework_agent.py — 使用 Framework 层构建 Agent 应用"""

from agentos.framework import Application, Plugin, Lifecycle

class MyAgent(Application):
    """自定义 Agent 应用"""

    def on_start(self):
        print(f"[{self.name}] Agent 已启动")

    def on_message(self, message: str) -> str:
        # 处理用户消息，调用 CoreLoopThree 认知循环
        response = self.process(message)
        return response

    def on_stop(self):
        print(f"[{self.name}] Agent 已停止")

# 启动应用
app = MyAgent(config={
    "name": "my-custom-agent",
    "model": "gpt-4o-mini",
    "system_prompt": "你是一个专业的 AI 助手。"
})
app.run()
```

---

## 5. 下一步

恭喜！你已经完成了 Airymax 的快速上手。接下来可以深入了解：

| 文档 | 说明 |
|:-----|:-----|
| [📘 架构原则](https://atomgit.com/openairymax/docs/blob/main/ARCHITECTURAL_PRINCIPLES.md) | 理解 Airymax 的五维正交设计体系 |
| [⚙️ 编译指南](https://atomgit.com/openairymax/docs/blob/main/BUILD_GUIDE.md) | 详细的构建配置与选项说明 |
| [🧪 测试指南](https://atomgit.com/openairymax/docs/blob/main/TESTING_GUIDE.md) | 单元测试、集成测试与契约测试 |
| [🐳 部署指南](https://atomgit.com/openairymax/docs/blob/main/DEPLOYMENT_GUIDE.md) | Docker / Kubernetes 生产部署 |
| [🔒 安全穹顶 (Cupolas)](../AgentRT/agentos/cupolas/README.md) | 四重内生安全体系详解 |
| [🧠 CoreLoopThree](../AgentRT/agentos/atoms/coreloopthree/README.md) | Agent 认知循环核心机制 |
| [📦 SDK 工具包](../../AgentRT/sdk/README.md) | Python / Go / Rust / TypeScript SDK 完整文档 |
| [🎯 示例项目](../../AgentRT/ecosystem/examples/hello-agent/README.md) | Hello Agent 示例项目详解 |

### 推荐学习路径

```
快速上手指南（本文）→ 架构原则 → CoreLoopThree → SDK 文档 → 示例项目
```

---

> "From data intelligence emerges." — 始于数据，终于智能。
>
> © 2026 SPHARX Ltd. All Rights Reserved.