# Airymax 快速入门指南

**最新**: 2026-06-09
**状态**: 维护中
**路径**: OpenAirymax/Docs/30-api/50-examples/01-quickstart.md
---

## 📋 前置要求

### 系统要求
- **操作系统**: Linux (Ubuntu 22.04+), macOS 12+, Windows 10/11 (WSL2)
- **编译器**: GCC 9+ / Clang 10+
- **CMake**: 3.16+
- **依赖库**:
  - OpenSSL (1.1+)
  - SQLite3 (3.30+)
  - cJSON (1.7+)
  - libcurl (7.68+)

### 快速安装依赖

#### Ubuntu/Debian
```bash
sudo apt-get update
sudo apt-get install -y build-essential cmake ninja-build \
    pkg-config git libssl-dev libcjson-dev libsqlite3-dev \
    libcurl4-openssl-dev
```

#### macOS (Homebrew)
```bash
brew install cmake openssl cJSON sqlite curl pkg-config
```

#### Docker (推荐)
```bash
# 使用预构建镜像，无需本地编译
docker pull spharx/agentos:v0.0.4
```

---

## 🚀 方式一: 从源码构建 (5分钟)

### Step 1: 克隆代码

```bash
git clone https://github.com/spharx/Airymax.git
cd Airymax
```

### Step 2: 配置构建

```bash
# 创建构建目录
mkdir -p build && cd build

# 配置 (Release模式)
cmake .. -DCMAKE_BUILD_TYPE=Release
```

**可选配置项**:
```bash
cmake .. \
    -DCMAKE_BUILD_TYPE=Release \        # Release/Debug/RelWithDebInfo
    -DBUILD_TESTING=ON \                # 构建测试
    -DAGENTOS_ENABLE_EXAMPLES=ON \       # 构建示例程序
    -DCMAKE_INSTALL_PREFIX=/usr/local \   # 安装路径
    -DAGENTOS_SANITIZE=ON                # 启用地址消毒器(调试)
```

### Step 3: 编译

```bash
# 并行编译 (使用所有CPU核心)
make -j$(nproc)
```

**预期输出**:
```
[100%] Built target gateway_d
[100%] Built target agentos-sched-d
...
✅ Build completed successfully!
```

### Step 4: 运行测试 (可选)

```bash
# 运行所有测试
ctest --output-on-failure -j$(nproc)

# 运行特定测试
./bin/test_cl3_loop          # 核心循环测试
./bin/test_gateway_d_test_service  # 网关服务测试
```

### Step 5: 安装

```bash
# 安装到系统目录
sudo make install

# 验证安装
gateway_d --version
agentos-llm-d --help
```

---

## 🐳 方式二: Docker部署 (2分钟)

### 使用docker-compose (推荐)

```bash
# 进入项目目录
cd AgentRT/Docker/30-api/docker

# 启动所有服务
docker-compose up -d

# 查看状态
docker-compose ps

# 查看日志
docker-compose logs -f gateway
```

### 单独运行网关服务

```bash
# 构建镜像
docker build -t agentos:v0.0.4 .

# 运行
docker run -d \
    --name agentos-gateway \
    -p 8080:8080 \
    -p 8081:8081 \
    -e OPENAI_API_KEY=your-api-key-here \
    agentos:v0.0.4
```

### 验证Docker部署

```bash
# 健康检查
curl http://localhost:8080/health

# 预期响应:
# {"status":"ok","service":"gateway","version":"0.0.4"}
```

---

## 💻 第一个Agent程序

### 示例1: 简单问答Agent

创建文件 `hello_agent.c`:

```c
/**
 * @file hello_agent.c
 * @brief 第一个Airymax程序 - 简单问答
 */

#include "agentos.h"
#include "loop.h"
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char* argv[]) {
    // 解析命令行参数
    const char* question = "你好，Airymax！";
    if (argc > 1) {
        question = argv[1];
    }

    printf("🤖 Airymax v0.0.4 - Hello Agent\n");
    printf("━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n");
    printf("问题: %s\n", question);
    printf("━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n\n");

    // 1️⃣ 初始化核心系统
    printf("[1/6] 初始化核心系统...");
    agentos_error_t err = agentos_core_init();
    if (err != AGENTOS_OK) {
        fprintf(stderr, "❌ 初始化失败: %d\n", err);
        return 1;
    }
    printf(" ✅ 完成\n");

    // 2️⃣ 创建三层循环
    printf("[2/6] 创建三层循环...");
    agentos_core_loop_t* loop = NULL;
    err = agentos_loop_create(NULL, &loop);
    if (err != AGENTOS_OK) {
        fprintf(stderr, "❌ 创建循环失败: %d\n", err);
        agentos_core_shutdown();
        return 1;
    }
    printf(" ✅ 完成\n");

    // 3️⃣ 启动后台处理线程
    printf("[3/6] 启动循环线程...");
    // 注意：实际应用中应该在独立线程中调用run
    // 这里为了简化，我们直接使用submit+wait同步模式
    printf(" ✅ 就绪\n");

    // 4️⃣ 提交任务
    printf("[4/6] 提交问题给认知层...");
    char* task_id = NULL;
    err = agentos_loop_submit(loop, question, strlen(question), &task_id);
    if (err != AGENTOS_OK) {
        fprintf(stderr, "❌ 提交失败: %d\n", err);
        agentos_loop_destroy(loop);
        agentos_core_shutdown();
        return 1;
    }
    printf(" ✅ 任务ID: %s\n", task_id);

    // 5️⃣ 等待结果
    printf("[5/6] 等待Agent思考...");
    char* answer = NULL;
    size_t answer_len = 0;
    
    // 设置超时为30秒
    err = agentos_loop_wait(loop, task_id, 30000, &answer, &answer_len);
    
    if (err == AGENTOS_OK && answer) {
        printf("\n🎯 Agent回答:\n");
        printf("━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n");
        printf("%.*s\n", (int)answer_len, answer);
        printf("━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n");
        
        // 释放结果内存
        AGENTOS_FREE(answer);
    } else if (err == AGENTOS_ETIMEDOUT) {
        printf("\n⏰ 超时：Agent在30秒内未完成思考\n");
    } else {
        printf("\n❌ 错误: %d\n", err);
    }

    // 6️⃣ 清理资源
    printf("\n[6/6] 清理资源...\n");
    AGENTOS_FREE(task_id);
    agentos_loop_destroy(loop);
    agentos_core_shutdown();
    printf("✅ 所有资源已释放\n");

    printf("\n感谢使用 Airymax! 👋\n");
    return 0;
}
```

### 编译与运行

```bash
# 编译
gcc -o hello_agent hello_agent.c \
    -I/usr/local/include/agentos \
    -L/usr/local/lib \
    -lagentos_coreloopthree -lagentos_cognition -lagentos_execution \
    -lagentos_memoryrovol -lagentos_common -lpthread

# 运行
export LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH
./hello_agent "什么是人工智能？"

# 或使用默认问题
./hello_agent
```

**预期输出**:
```
🤖 Airymax v0.0.4 - Hello Agent
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
问题: 什么是人工智能？
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[1/6] 初始化核心系统... ✅ 完成
[2/6] 创建三层循环... ✅ 完成
[3/6] 启动循环线程... ✅ 就绪
[4/6] 提交问题给认知层... ✅ 任务ID: task_abc123
[5/6] 等待Agent思考...

🎯 Agent回答:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
人工智能(AI)是计算机科学的一个分支，致力于创建...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[6/6] 清理资源...
✅ 所有资源已释放

感谢使用 Airymax! 👋
```

---

## 📚 示例2: 记忆系统演示

创建文件 `memory_demo.c`:

```c
#include "memoryrovol.h"
#include <stdio.h>
#include <stdlib.h>

int main() {
    printf("📚 MemoryRovol 四层记忆系统演示\n");
    printf("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n\n");

    // 1. 创建记忆系统
    printf("[1] 创建四层记忆系统...\n");
    agentos_memoryrov_handle_t* memory = agentos_memoryrov_create();
    if (!memory) {
        fprintf(stderr, "❌ 创建记忆系统失败\n");
        return 1;
    }
    printf("   ✅ L1原始卷 + L2特征层 + L3结构层 + L4模式层\n\n");

    // 2. 写入多条记忆
    const char* memories[] = {
        "用户偏好：喜欢简洁的回答，不要过于冗长",
        "用户职业：软件工程师，熟悉Python和Go",
        "上次对话：讨论了微服务架构的设计",
        "用户兴趣：对量子计算和AI安全感兴趣",
        "项目背景：正在开发一个智能客服系统"
    };

    printf("[2] 写入 %zu 条记忆...\n", sizeof(memories)/sizeof(memories[0]));
    for (size_t i = 0; i < sizeof(memories)/sizeof(memories[0]); i++) {
        agentos_error_t err = agentos_memoryrov_add_memory(
            memory, 
            memories[i], 
            strlen(memories[i])
        );
        
        if (err == AGENTOS_OK) {
            printf("   ✅ [%zu] %s\n", i+1, memories[i]);
        } else {
            printf("   ❌ [%zu] 写入失败: %d\n", i+1, err);
        }
    }
    printf("\n");

    // 3. 相似性检索
    const char* queries[] = {
        "用户的技术背景是什么？",
        "用户喜欢什么样的沟通风格？",
        "当前项目的目标是什么？"
    };

    printf("[3] 语义相似性检索...\n");
    for (size_t q = 0; q < sizeof(queries)/sizeof(queries[0]); q++) {
        printf("\n   🔍 查询: \"%s\"\n", queries[q]);
        
        agentos_memory_query_t query = {
            .memory_query_text = (char*)queries[q],
            .memory_query_text_len = strlen(queries[q]),
            .memory_query_limit = 3,
            .memory_query_include_raw = 1
        };
        
        agentos_memory_result_ext_t* result = NULL;
        agentos_error_t err = agentos_memoryrov_query(memory, &query, &result);
        
        if (err == AGENTOS_OK && result && result->memory_result_count > 0) {
            printf("   📝 找到 %zu 条相关记忆:\n", result->memory_result_count);
            
            for (size_t i = 0; i < result->memory_result_count && i < 3; i++) {
                agentos_memory_record_t* record = 
                    result->memory_result_items[i]->memory_result_item_record;
                
                printf("      [%zu] %.80s%s\n", 
                       i+1, 
                       (const char*)record->memory_record_data,
                       strlen((const char*)record->memory_record_data) > 80 ? "..." : "");
            }
            
            agentos_memory_result_free(result);
        } else {
            printf("   ℹ️  未找到相关记忆\n");
        }
    }

    // 4. 统计信息
    printf("\n[4] 记忆系统统计:\n");
    // 可以添加统计API调用
    
    // 5. 销毁
    printf("\n[5] 销毁记忆系统...\n");
    agentos_memoryrov_destroy(memory);
    printf("   ✅ 完成\n");

    printf("\n━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n");
    printf("🎉 演示完成！\n");
    
    return 0;
}
```

---

## 🌐 示例3: 协议栈集成

创建文件 `protocol_demo.c`:

```c
#include "unified_protocol.h"
#include "mcp_v1_adapter.h"
#include "a2a_v03_adapter.h"
#include <stdio.h>
#include <stdlib.h>

void demo_mcp_protocol() {
    printf("\n=== MCP v1.0 Protocol Demo ===\n");
    
    // 获取MCP适配器
    const protocol_adapter_t* mcp = mcp_v1_get_adapter();
    if (!mcp) {
        printf("❌ MCP adapter not available\n");
        return;
    }
    
    printf("Adapter: %s (v%s)\n", mcp->name, mcp->version ? mcp->version : "unknown");
    
    // 初始化
    if (mcp->init) {
        int rc = mcp->init(mcp->context);
        printf("Init: %s\n", rc == 0 ? "✅ Success" : "❌ Failed");
    }
    
    // 查询能力
    if (mcp->capabilities) {
        uint32_t caps = mcp->capabilities(mcp->context);
        printf("Capabilities: 0x%08X\n", caps);
        
        if (caps & PROTO_CAP_REQUEST_RESPONSE) printf("  ✓ Request-Response\n");
        if (caps & PROTO_CAP_STREAMING) printf("  ✓ Streaming\n");
        if (caps & PROTO_CAP_TOOL_CALLING) printf("  ✓ Tool Use\n");
        if (caps & PROTO_CAP_AUTHENTICATION) printf("  ✓ Authentication\n");
    }
    
    // 发送消息
    unified_message_t msg = {
        .protocol = PROTOCOL_MCP_V1,
        .message_type = MSG_TYPE_REQUEST,
        .payload = "{\"method\":\"tools/list\",\"id\":1}",
        .payload_size = strlen("{\"method\":\"tools/list\",\"id\":1}")
    };
    
    void* encoded = NULL;
    size_t encoded_size = 0;
    
    if (mcp->encode) {
        int rc = mcp->encode(mcp->context, &msg, &encoded, &encoded_size);
        printf("Encode: %s (%zu bytes)\n", rc == 0 ? "✅" : "❌", encoded_size);
        
        if (encoded) {
            printf("Encoded data: %.100s...\n", (char*)encoded);
            free(encoded);
        }
    }
    
    // 清理
    if (mcp->destroy) {
        mcp->destroy(mcp->context);
        printf("Destroy: ✅\n");
    }
}

void demo_a2a_protocol() {
    printf("\n=== A2A v0.3 Protocol Demo ===\n");
    
    const protocol_adapter_t* a2a = a2a_v03_get_adapter();
    if (!a2a) {
        printf("❌ A2A adapter not available\n");
        return;
    }
    
    printf("Adapter: %s\n", a2a->name ? a2a->name : "A2A v0.3");
    
    // 类似MCP的初始化和使用流程...
    printf("(A2A protocol follows similar pattern)\n");
}

int main() {
    printf("╔══════════════════════════════════╗\n");
    printf("║  Airymax Protocol Stack Demo       ║\n");
    printf("╚══════════════════════════════════╝\n\n");
    
    demo_mcp_protocol();
    demo_a2a_protocol();
    
    printf("\n✨ All demos completed!\n");
    return 0;
}
```

---

## 🔧 故障排查

### 编译错误

**问题**: `fatal error: 'agentos.h' file not found`
```bash
# 解决：指定正确的include路径
-I/path/to/AgentRT/agentos/include
```

**问题**: `undefined reference to 'agentos_core_init'`
```bash
# 解决：链接必要的库
-lagentos_core -lagentos_common -lpthread
```

### 运行时错误

**问题**: `libagentos_common.so: cannot open shared object file`
```bash
# 解决：设置库路径
export LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH

# 或添加到/etc/ld.so.conf.d/
echo '/usr/local/lib' | sudo tee /etc/ld.so.conf.d/agentos.conf
sudo ldconfig
```

**问题**: Segmentation fault
```bash
# 解决：使用AddressSanitizer重新编译
cmake .. -DCMAKE_BUILD_TYPE=Debug -DAGENTOS_SANITIZE=ON
make clean && make
# 再次运行，会显示详细的错误位置
```

### 性能问题

**问题**: 响应缓慢
```bash
# 检查日志级别（debug模式会很慢）
export AGENTOS_LOG_LEVEL=warn

# 增加执行线程数
# 在创建loop时配置：
agentos_loop_config_t config = { .loop_config_execution_threads = 16 };
agentos_loop_create(&config, &loop);
```

---

## 📖 下一步学习路径

### 初学者 (刚接触Airymax)
1. ✅ 完成本快速入门
2. 📖 阅读 [CoreLoopThree API](../core/coreloop_api.md)
3. 💡 尝试修改 `hello_agent.c` 的参数
4. 🧪 运行更多 [examples](../examples/)

### 中级开发者
1. 📖 学习 [用户态服务 API](../daemon/gateway_api.md)
2. 🔧 实现 [自定义工具](../examples/custom_tool_example.c)
3. 🌐 集成外部LLM API
4. 🧪 编写单元测试

### 高级开发者
1. 📖 研究 [协议栈扩展（待编写）](../protocols/protocol_extension_api.md)
2. 🏗️ 开发自定义协议适配器
3. 🔬 性能优化与调优
4. 🤝 贡献代码到主仓库

---

## 🆘 获取帮助

- **文档**: https://docs.agentos.dev
- **GitHub Issues**: https://github.com/spharx/AgentRT/issues
- **Discussions**: https://github.com/spharx/AgentRT/discussions
- **邮件列表**: dev@agentos.dev (即将开放)

---

**最后更新**: 2026-04-28  
**适用版本**: Airymax v0.0.4+
