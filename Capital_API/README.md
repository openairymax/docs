# AgentOS Capital API - 完整API参考文档

**最新**: 2026-06-09
**状态**: 维护中
**路径**: OpenAirymax/Docs/Capital_API/README.md
**作者**:
    - Liren Wang
---

## 📚 文档导航

### 核心模块 API
- [三层认知循环 (CoreLoopThree)](core/coreloop_api.md)
  - 认知引擎 API
  - 执行引擎 API
  - 记忆引擎 API
  - 主循环 API
  
- [内核系统 (CoreKern)](core/corekern_api.md)
  - 初始化与关闭
  - 任务管理
  - IPC通信
  - 时间事件系统

- [记忆系统 (MemoryRovol)](core/memoryrovol_api.md)
  - 四层记忆架构
  - L1原始卷 API
  - L2特征层 API
  - L3结构层 API
  - L4模式层 API
  - 检索与遗忘 API

- [任务流 (TaskFlow)](core/taskflow_api.md)
  - 任务定义
  - 流程编排
  - 依赖管理

- [系统调用 (SysCall)](core/syscall_api.md)
  - 路由器
  - 调用入口
  - 权限控制

### 守护进程 API
- [网关守护进程 (gateway_d)](daemon/gateway_api.md)
  - HTTP/WebSocket/Stdio网关
  - 服务管理
  - 配置接口

- [LLM服务 (llm_d)](daemon/llm_api.md)
  - LLM调用接口
  - Token计数
  - 缓存管理
  - 定价规则

- [通道服务 (channel_d)](daemon/channel_api.md)
  - 通道管理
  - 消息传递
  - 线程安全保证

- [调度器 (sched_d)](daemon/scheduler_api.md)
  - 调度策略
  - 任务队列
  - 优先级管理

- [监控服务 (monit_d)](daemon/monitoring_api.md)
  - 指标收集
  - 告警系统
  - 追踪功能

- [市场服务 (market_d)](daemon/market_api.md)
  - 代理注册
  - 技能注册
  - 安装器接口

- [工具执行器 (tool_d)](daemon/tool_api.md)
  - 工具注册
  - 执行引擎
  - 验证器
  - 缓存机制

### 协议栈 API
- [统一协议框架](protocols/unified_protocol_api.md)
  - 协议适配器接口
  - 消息编解码
  - 协议路由

- [协议扩展框架](protocols/protocol_extension_api.md)
  - 适配器注册
  - 中间件链
  - 能力协商

- [标准协议适配器](protocols/adapters_api.md)
  - MCP v1.0
  - A2A v0.3
  - OpenAI Enterprise

- [第三方集成](protocols/integrations_api.md)
  - OpenClaw
  - Claude
  - LangChain
  - AutoGen

### 公共工具库 API
- [字符串处理](commons/string_api.md)
- [内存管理](commons/memory_api.md)
- [日志系统](commons/logging_api.md)
- [错误处理](commons/error_api.md)
- [同步原语](commons/sync_api.md)
- [配置管理](commons/config_api.md)
- [UUID生成](commons/uuid_api.md)

### Docker部署
- [Docker快速启动](docker/README.md)
- [Dockerfile参考](docker/Dockerfile)
- [docker-compose.yml](docker/docker-compose.yml)
- [环境变量配置](docker/env_config.md)

### 示例代码
- [快速入门示例](examples/quickstart.md)
- [完整Agent实现](examples/full_agent_example.c)
- [自定义协议适配器](examples/custom_adapter_example.c)
- [多Agent协作](examples/multi_agent_example.c)

---

## 🔑 快速查找常用API

### 创建并运行一个完整的Agent循环

```c
#include "agentos.h"
#include "loop.h"

int main() {
    // 1. 初始化核心系统
    agentos_error_t err = agentos_core_init();
    if (err != AGENTOS_SUCCESS) {
        return -1;
    }

    // 2. 创建三层循环
    agentos_core_loop_t* loop = NULL;
    err = agentos_loop_create(NULL, &loop);
    if (err != AGENTOS_SUCCESS) {
        agentos_core_shutdown();
        return -1;
    }

    // 3. 启动循环（后台线程）
    agentos_loop_run(loop);

    // 4. 提交任务
    char* task_id = NULL;
    const char* input = "分析这段文本的情感倾向";
    err = agentos_loop_submit(loop, input, strlen(input), &task_id);
    
    // 5. 等待结果
    char* result = NULL;
    size_t result_len = 0;
    err = agentos_loop_wait(loop, task_id, 30000, &result, &result_len);
    
    if (err == AGENTOS_SUCCESS && result) {
        printf("Agent响应: %.*s\n", (int)result_len, result);
        AGENTOS_FREE(result);
    }
    
    AGENTOS_FREE(task_id);

    // 6. 清理资源
    agentos_loop_destroy(loop);
    agentos_core_shutdown();
    
    return 0;
}
```

### 使用记忆系统

```c
#include "memoryrovol.h"

// 创建四层记忆系统
agentos_memoryrov_handle_t* memory = agentos_memoryrov_create();

// 写入记忆
const char* content = "用户偏好：喜欢简洁的回复风格";
agentos_error_t err = agentos_memoryrov_add_memory(memory, content, strlen(content));

// 相似性检索
agentos_memory_query_t query = {
    .memory_query_text = "用户喜欢什么样的回复？",
    .memory_query_limit = 5,
    .memory_query_include_raw = 1
};

agentos_memory_result_ext_t* result = NULL;
err = agentos_memoryrov_query(memory, &query, &result);

if (result && result->memory_result_count > 0) {
    for (size_t i = 0; i < result->memory_result_count; i++) {
        printf("记忆 %zu: %s\n", i, 
               result->memory_result_items[i]->memory_result_item_record->memory_record_data);
    }
}

// 清理
agentos_memory_result_free(result);
agentos_memoryrov_destroy(memory);
```

### 使用协议栈发送消息

```c
#include "unified_protocol.h"
#include "mcp_v1_adapter.h"

// 初始化MCP适配器
const protocol_adapter_t* mcp = mcp_v1_get_adapter();
if (mcp && mcp->init) {
    mcp->init(mcp->context);
}

// 构建统一消息
unified_message_t msg = {
    .protocol = PROTOCOL_MCP_V1,
    .message_type = MSG_TYPE_REQUEST,
    .payload = "{\"method\":\"tools/list\"}",
    .payload_size = strlen("{\"method\":\"tools/list\"}")
};

// 编码并发送
void* encoded = NULL;
size_t encoded_size = 0;
if (mcp->encode) {
    mcp->encode(mcp->context, &msg, &encoded, &encoded_size);
    
    // 通过handle_request发送
    void* response = NULL;
    mcp->handle_request(mcp->context, "send", encoded, &response);
    
    free(encoded);
    free(response);
}
```

---

## 📖 API设计原则

### 1. 错误处理约定
所有API函数返回 `agentos_error_t` 类型：
- `AGENTOS_SUCCESS` (0): 成功
- `AGENTOS_EINVAL` (-1): 无效参数
- `AGENTOS_ENOMEM` (-2): 内存不足
- `AGENTOS_ENOTINIT` (-3): 未初始化
- `AGENTOS_ETIMEDOUT` (-4): 操作超时
- `AGENTOS_EBUSY` (-5): 资源忙
- `AGENTOS_ENOENT` (-6): 不存在

### 2. 内存管理约定
- **分配**: 使用 `AGENTOS_MALLOC`, `AGENTOS_CALLOC`, `AGENTOS_STRDUP`
- **释放**: 使用 `AGENTOS_FREE`
- **规则**: 谁分配谁释放（除非文档明确说明转移所有权）

### 3. 线程安全约定
- 所有标记为 `_t` 的句柄类型都是线程安全的
- 内部使用互斥锁保护共享状态
- 回调函数会在独立的线程上下文中调用

### 4. 命名规范
- 函数: `module_action_object` (如 `agentos_loop_create`)
- 类型: `module_object_t` (如 `agentos_core_loop_t`)
- 常量: `MODULE_CONSTANT_NAME` (如 `AGENTOS_SUCCESS`)
- 枚举: `MODULE_TYPE_VALUE` (如 `TASK_STATUS_PENDING`)

---

## 🔄 版本历史

| 版本 | 日期 | 主要变更 |
|------|------|----------|
| v0.0.4 | 2026-04-28 | Foundation Release - 完整API文档 |
| v0.0.3 | 2026-04-25 | 协议栈v2.1.0集成 |
| v0.0.2 | 2026-04-20 | 三层认知循环架构实现 |
| v0.0.1 | 2026-04-15 | 初始版本 |

---

## 📞 支持与反馈

- **问题报告**: [GitHub Issues](https://github.com/spharx/AgentOS/issues)
- **讨论社区**: [Discussions](https://github.com/spharx/AgentOS/discussions)
- **API变更日志**: 见各模块文档末尾的"版本历史"章节

---

## 📜 许可证

```
SPHARX AgentOS v0.0.4
Copyright (c) 2026 SPHARX Ltd.
SPDX-License-Identifier: GPL-3.0-or-later OR Apache-2.0
```

**注意**: 本文档描述的是Developer Preview版本，API可能在后续版本中调整。生产环境使用请等待正式Release版本。
