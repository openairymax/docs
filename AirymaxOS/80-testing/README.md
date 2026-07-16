Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）测试体系设计
> **文档定位**：agentrt-linux（AirymaxOS）测试工程体系主索引\
> **文档版本**：0.1.1\
> **最后更新**：2026-07-13\
> **同源映射**：agentrt 7 层自动化验证 + Linux 6.6 测试框架（KUnit/kselftest/动态分析）\
> **理论根基**：Linux 内核测试体系 + Airymax E-8 可测试性 + A-4 完美主义

---

## 1. 模块定位

agentrt-linux 测试体系是工程标准可执行性的核心保障。它继承 Linux 内核 30+ 年沉淀的多层测试哲学（KUnit 白盒 + kselftest 系统级 + 动态分析 + 静态分析 + 覆盖度量），并在其上扩展智能体操作系统专属的 Agent 行为契约测试、模糊测试、形式化验证等。

### 1.1 测试体系分层

| 层级 | 类型 | 工具 | 时间尺度 | 范围 |
|------|------|------|---------|------|
| L1 | 白盒单元测试 | KUnit | 毫秒级 | 单函数 |
| L2 | 系统级测试 | kselftest | 秒级 | 子系统 |
| L3 | 内核自检 | `lib/test_*` | 秒级 | 内核特性 |
| L4 | 动态分析 | KASAN/KFENCE/UBSAN/KCSAN/lockdep/kmemleak | 运行时 | 内存/并发 |
| L5 | 静态分析 | Sparse/Smatch/Coccinelle/clang-analyzer | 编译时 | 代码模式 |
| L6 | 覆盖度量 | KCOV/gcov | 运行时 | 代码覆盖 |
| L7 | ftrace 启动自检 | ftrace + ring buffer | 启动时 | 跟踪 |
| **L8** | **Agent 契约测试** | **agentrt-linux 专属** | **秒级** | **Agent 行为** |
| **L9** | **模糊测试** | **syzkaller + agentrt-linux 扩展** | **小时级** | **输入边界** |
| **L10** | **形式化验证** | **seL4 风格 + TLA+** | **天级** | **关键路径** |

### 1.2 agentrt-linux 扩展

- **Agent 行为契约测试**：验证 Agent 通过 SDK 调用系统能力的契约（输入/输出/异常）
- **Token 能效测试**：测量 Agent 工作负载的 Token 消耗与能效
- **记忆卷载测试**：验证 L1→L2→L3→L4 记忆演化的正确性
- **IPC 协议测试**：AgentsIPC 128B 消息头 + 5 种 payload 的协议校验

---

## 2. 核心测试框架

### 2.1 KUnit（白盒单元测试）

KUnit 是 Linux 内核官方单元测试框架，无需真实硬件，毫秒级执行：
```c
#include <kunit/test.h>

static void test_my_function(struct kunit *test) {
    KUNIT_EXPECT_EQ(test, 1, my_function(0));
    KUNIT_ASSERT_NOT_ERR_OR_NULL(test, my_function(1));
}

static struct kunit_case my_test_cases[] = {
    KUNIT_CASE(test_my_function),
    {}
};

static struct kunit_suite my_test_suite = {
    .name = "my_module_tests",
    .test_cases = my_test_cases,
};

kunit_test_suite(my_test_suite);
```

### 2.2 kselftest（用户态系统级测试）

kselftest 从用户态测试内核特性：
```bash
make -C tools/testing/selftests TARGETS=sched
./tools/testing/selftests/sched/sched_ext
```

### 2.3 lib/test_*（内核自检）

`lib/test_*.c` 文件通过 `CONFIG_TEST_*` 启用，启动时执行：
```bash
make CONFIG_TEST_BITMAP=m modules
```

### 2.4 动态分析工具

| 工具 | 用途 | 启用方式 |
|------|------|---------|
| KASAN | 地址消毒器（越界/use-after-free） | `CONFIG_KASAN=y` |
| KFENCE | 轻量级内存错误检测 | `CONFIG_KFENCE=y` |
| UBSAN | 未定义行为消毒器 | `CONFIG_UBSAN=y` |
| KCSAN | 并发消毒器（数据竞争） | `CONFIG_KCSAN=y` |
| lockdep | 锁依赖检查 | `CONFIG_PROVE_LOCKING=y` |
| kmemleak | 内存泄漏检测 | `CONFIG_DEBUG_KMEMLEAK=y` |

### 2.5 静态分析工具

| 工具 | 用途 |
|------|------|
| Sparse | 类型检查（`__user`、`__rcu` 等） |
| Smatch | 模式匹配（未初始化变量、空指针解引用） |
| Coccinelle | 语义补丁（API 迁移、模式重构） |
| clang-analyzer | Clang 静态分析器 |
| rust-clippy | Rust 代码改进建议 |

### 2.6 覆盖度量

- **KCOV**：代码覆盖率（`CONFIG_KCOV=y`）
- **gcov**：GCC 覆盖率（`CONFIG_GCOV_KERNEL=y`）
- **最低覆盖率门槛**：≥80%（关键路径必须 100%）

### 2.7 ftrace 启动自检

`ftrace` 在启动时进行自检：
```c
// kernel/trace/trace_selftest.c
int __init ftrace_startup(struct tracer *tracer, int command) {
    // 自检逻辑
}
```

---

## 3. 文档索引

```
80-testing/
├── README.md                       # 本文件
├── 01-kunit-framework.md           # KUnit 单元测试框架
├── 02-kselftest.md                 # kselftest 系统级测试
├── 03-kernel-selftests.md          # lib/test_* 内核自检
├── 04-dynamic-analysis.md          # 动态分析（KASAN/KFENCE/UBSAN/KCSAN/lockdep）
├── 05-static-analysis.md           # 静态分析（Sparse/Smatch/Coccinelle）
├── 06-coverage-metrics.md          # 覆盖度量（KCOV/gcov）
├── 07-ftrace-selftest.md           # ftrace 启动自检
├── 08-agent-contract-testing.md    # agentrt-linux 专属：Agent 行为契约测试
├── 09-fuzz-testing.md              # 模糊测试（syzkaller + 扩展）
└── 10-formal-verification.md       # 形式化验证（seL4 风格 + TLA+）
```

### 3.1 0.1.1 版本范围

README + 01 + 02 文档（3 文档奠基），确立测试体系设计框架与 KUnit/kselftest 核心机制。其余 8 文档（03-kernel-selftests 至 10-formal-verification）在 1.0.1 版本完成。

### 3.2 1.0.1 版本范围

完成剩余 8 文档（03-kernel-selftests 至 10-formal-verification），实施测试工程标准，进行代码级验证。

---

## 4. agentrt-linux 专属扩展

### 4.1 Agent 行为契约测试

```c
// 测试 Agent 通过 SDK 调用系统能力的契约
static void test_agent_cognition_contract(struct kunit *test) {
    struct agent_handle *agent = airy_cognition_client_create();
    KUNIT_ASSERT_NOT_ERR_OR_NULL(test, agent);
    
    struct cognition_response *resp = airy_cognition_process(agent, "hello");
    KUNIT_EXPECT_EQ(test, AIRY_EOK, resp->status);
    KUNIT_EXPECT_NOT_NULL(test, resp->output);
    
    airy_cognition_client_destroy(agent);
}
```

### 4.2 模糊测试扩展

基于 syzkaller 扩展 AgentsIPC 协议模糊测试：
- 128B 消息头的字段边界模糊
- 5 种 payload 类型的语义模糊
- Agent SDK 接口的输入边界模糊

### 4.3 形式化验证（seL4 风格）

关键路径采用形式化验证：
- MicroCoreRT 调度算法：TLA+ 时序逻辑验证
- AgentsIPC 协议：模型检查（SPIN/nuXmv）
- MemoryRovol 记忆演化：定理证明（Coq/Isabelle）

### 4.4 测试覆盖率门槛

| 模块 | 最低覆盖率 | 关键路径 |
|------|-----------|---------|
| kernel | 90% | 100%（调度/内存/IPC） |
| security | 95% | 100%（capability/LSM） |
| memory | 90% | 100%（CXL/PMEM） |
| cognition | 85% | 100%（CoreLoopThree） |
| 其他 | 80% | 90% |

---

## 5. 五维原则映射

| 原则 | 在本模块的体现 |
|------|---------------|
| **E-8 可测试性** | 多层测试体系确保可测试 |
| **A-4 完美主义** | 覆盖率门槛 + 形式化验证 |
| **E-1 安全内生** | 动态分析强制 + 安全测试 |
| **S-1 反馈闭环** | 测试失败即 CI 反馈 |
| **IRON-9 v2 同源且部分代码共享** | Agent 契约测试与 agentrt 互操作 |

---

## 6. 相关文档

- `50-engineering-standards/06-toolchain-and-automation.md`（7 层验证）
- `50-engineering-standards/01-coding-standards.md`（错误处理强制）
- `110-security/README.md`（安全测试）
- `20-modules/08-tests-linux.md`（tests-linux 子仓设计）

---

## 7. 参考材料

- Linux 6.6 `lib/kunit/`（KUnit 框架）
- Linux 6.6 `tools/testing/selftests/`（kselftest）
- Linux 6.6 `Documentation/dev-tools/testing-overview.rst`（测试工具全景）
- Linux 6.6 `lib/test_*.c`（内核自检样本）
- seL4 项目（形式化验证参考）

---

> **文档结束** | README + 01 + 02
