Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# POSIX 兼容性设计

> **文档定位**: agentrt-liunx（AirymaxOS，极境智能体操作系统）兼容性体系核心子文档，定义 agentrt-liunx 对 POSIX 标准的兼容层次、syscall 兼容性分类与语义扩展
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-09
> **理论根基**: Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0
> **同源映射**: Linux 6.6 KABI / ABI 兼容性体系（IRON-9 v2 [IND] 完全独立层）
> **IRON-9 v2 层次**: [IND] 完全独立层（POSIX 兼容性为 agentrt-liunx 作为 OS 发行版的专属设计）

---

## 1. 设计目标与 POSIX 兼容层次

### 1.1 设计目标

agentrt-liunx（AirymaxOS）作为基于 Linux 6.6 内核基线的智能体操作系统发行版，必须保持与 POSIX 标准及主流 Linux 发行版的兼容性。POSIX 兼容性设计达成以下工程目标：

1. **POSIX.1-2017 兼容**：完全兼容 POSIX.1-2017（IEEE Std 1003.1-2017）核心 syscall
2. **主流发行版二进制兼容**：在 Ubuntu/Debian/Fedora/RHEL 上编译的二进制可直接运行
3. **Agent 语义扩展**：在不破坏 POSIX 兼容的前提下，扩展 Agent 专属语义
4. **降级运行支持**：Agent 专属 syscall 在非 agentrt-liunx 环境下可降级
5. **测试套件验证**：通过 POSIX 测试套件（PCTS）+ LSB 验证

### 1.2 POSIX 兼容分层

| 层级 | 类型 | 兼容程度 | 例子 |
|------|------|----------|------|
| L1 | 完全兼容 | 语义一致 | `read()`, `write()`, `open()` |
| L2 | 语义扩展 | POSIX 兼容 + Agent 增强 | `fork()`, `clone()` |
| L3 | 行为差异 | 保留 POSIX 签名，行为不同 | `sched_yield()` |
| L4 | 不实现 | Agent 环境无意义 | 部分传统信号 |

### 1.3 兼容性承诺

| 承诺维度 | 范围 |
|----------|------|
| POSIX.1-2017 | 核心 syscall 全兼容 |
| LSB 5.0 | Linux Standard Base 兼容 |
| glibc 兼容 | glibc 2.34+ 二进制直接运行 |
| 跨发行版 | Ubuntu 22.04+ / Debian 12+ / Fedora 36+ / RHEL 9+ |

---

## 2. POSIX syscall 兼容性分类

### 2.1 L1 完全兼容的 syscall

以下 syscall 与 POSIX.1-2017 完全兼容，语义一致：

| syscall | POSIX 定义 | agentrt-liunx 实现 |
|---------|------------|---------------------|
| `read()` | 从 fd 读取字节 | 直接转发至 VFS |
| `write()` | 向 fd 写入字节 | 直接转发至 VFS |
| `open()` | 打开文件 | 直接转发至 VFS |
| `close()` | 关闭 fd | 直接转发至 VFS |
| `lseek()` | 调整文件偏移 | 直接转发至 VFS |
| `stat()` | 获取文件状态 | 直接转发至 VFS |
| `dup()` | 复制 fd | 标准实现 |
| `pipe()` | 创建管道 | 标准实现 |
| `socket()` | 创建 socket | 转发至 net_d |
| `connect()` | 连接 socket | 转发至 net_d |

### 2.2 L2 语义扩展的 syscall

以下 syscall 保留 POSIX 签名，但行为针对 Agent 工作负载扩展：

| syscall | POSIX 语义 | agentrt-liunx 扩展 |
|---------|------------|---------------------|
| `fork()` | 创建子进程 | 创建子 Agent（capability 继承） |
| `clone()` | 创建轻量进程 | 创建 Agent 线程（SCHED_AGENT） |
| `execve()` | 替换进程映像 | 替换 Agent 镜像（保留记忆卷载） |
| `mmap()` | 内存映射 | 支持映射 MemoryRovol 卷 |
| `sched_setscheduler()` | 设置调度策略 | 支持 SCHED_AGENT |
| `prctl()` | 进程控制 | 扩展 Agent 专属 PR_AGENT_* 选项 |

### 2.3 L3 行为差异的 syscall

以下 syscall 行为与 POSIX 存在差异，但保留签名：

| syscall | POSIX 行为 | agentrt-liunx 差异 | 原因 |
|---------|------------|---------------------|------|
| `sched_yield()` | 让出 CPU | 让出 CPU + Token 预算检查 | Agent 调度感知 |
| `nice()` | 调整 nice 值 | 映射至 SCHED_AGENT 优先级 | 调度类差异 |
| `getpriority()` | 获取优先级 | 返回 SCHED_AGENT 优先级 | 调度类差异 |
| `signal()` | 信号处理 | 部分信号被 Cupolas 拦截 | 安全策略 |
| `kill()` | 发送信号 | Cupolas capability 校验 | 安全策略 |

### 2.4 L4 不实现的接口

以下接口在 Agent 环境下不实现或受限：

| 接口 | POSIX 定义 | 不实现原因 |
|------|------------|------------|
| `ptrace()` | 进程追踪 | 安全风险，仅 root 可用 |
| `kexec_load()` | 内核热替换 | 仅 root 可用 |
| `uselib()` | 加载共享库 | 已废弃 |
| 部分 `ioctl()` | 设备控制 | 按 Cupolas 策略过滤 |

---

## 3. `fork()` vs `agentrt_agent_fork()` 语义差异

### 3.1 语义差异对比表

agentrt-liunx 提供两种进程创建接口：标准 POSIX `fork()` 与 Agent 专属 `agentrt_agent_fork()`。下表详细对比两者语义差异：

| 维度 | `fork()`（POSIX） | `agentrt_agent_fork()`（agentrt-liunx 扩展） |
|------|-------------------|-----------------------------------------------|
| **创建对象** | 子进程（拷贝父进程映像） | 子 Agent（独立租户） |
| **资源隔离** | COW 共享父进程地址空间 | cgroup v2 + Landlock + capability 三重隔离 |
| **capability 继承** | 完全继承父进程 capability | 显式声明继承的 capability 子集 |
| **Token 预算** | 与父进程共享 | 独立预算契约（可声明子预算） |
| **调度类** | 继承父进程调度类 | 默认 SCHED_AGENT |
| **记忆卷载** | 不继承 | 可声明挂载 MemoryRovol 子卷 |
| **IPC 通道** | 继承父进程 fd | 独立 AgentsIPC 通道 |
| **TraceID** | 不继承 | 继承父 Agent TraceID（链路追踪） |
| **生命周期** | 独立（受父进程 wait） | 独立（内核状态机管理） |
| **安全审计** | 标准 | Cupolas 全程审计 |
| **返回值** | 子进程返回 0，父进程返回 PID | 子 Agent 返回 0，父 Agent 返回 agent_id |
| **错误码** | -EAGAIN/-ENOMEM | + -AGENTRT_EBUDGET/-AGENTRT_ECAP |

### 3.2 fork() 实现

```c
/**
 * sys_fork - 标准 POSIX fork
 *
 * 行为：完全 POSIX 兼容，创建子进程
 * agentrt-liunx 扩展：子进程默认继承父进程 Agent 状态
 * 若父进程是 Agent，子进程自动注册为子 Agent
 */
SYSCALL_DEFINE0(fork)
{
#ifdef CONFIG_AIRYMAXOS_AGENT
	struct agentrt_agent *parent = current->agentrt_agent;
	struct kernel_clone_args args = {
		.exit_signal = SIGCHLD,
	};

	/* 若当前进程是 Agent，使用 agentrt_agent_fork 语义 */
	if (parent && parent->state == AGENTRT_AGENT_STATE_RUNNING) {
		/* 但保留 POSIX 行为：不强制 capability 声明 */
		args.agentrt_inherit_caps = true;
		args.agentrt_sched_class = parent->sched_class;
	}
	return kernel_clone(&args);
#else
	return -ENOSYS;
#endif
}
```

### 3.3 agentrt_agent_fork() 实现

```c
/**
 * sys_agentrt_agent_fork - Agent 专属 fork
 * @config: 子 Agent 配置（capability 子集 + Token 预算 + 记忆卷载）
 *
 * 行为：创建子 Agent，独立租户
 * - cgroup v2 + Landlock + capability 三重隔离
 * - 独立 Token 预算契约
 * - 可挂载 MemoryRovol 子卷
 * - 默认 SCHED_AGENT 调度类
 * - 继承父 Agent TraceID
 */
SYSCALL_DEFINE1(agentrt_agent_fork,
		struct agentrt_agent_fork_config __user *, uconfig)
{
	struct agentrt_agent_fork_config config;
	struct agentrt_agent *parent = current->agentrt_agent;
	int ret;

	if (!parent) {
		/* 非 Agent 进程不允许调用 */
		return -AGENTRT_EPERM;
	}

	if (copy_from_user(&config, uconfig, sizeof(config)))
		return -EFAULT;

	/* 校验 capability 子集：子 Agent 不能超过父 Agent */
	if ((config.inherit_caps & parent->isolation.permitted_caps)
	    != config.inherit_caps) {
		return -AGENTRT_ECAP;
	}

	/* 校验 Token 预算：子 Agent 预算不能超过父 Agent 剩余 */
	if (config.budget.total > parent->budget.remaining) {
		return -AGENTRT_EBUDGET;
	}

	/* 创建子 Agent（内核状态机 CREATING） */
	ret = agentrt_agent_create_internal(&config, parent->trace_id);
	if (ret < 0)
		return ret;

	/* 从父 Agent Token 预算中扣除子 Agent 预算 */
	atomic64_sub(config.budget.total, &parent->budget.remaining);

	return ret;  /* 返回子 agent_id */
}
```

### 3.4 使用场景对比

| 场景 | 推荐接口 | 原因 |
|------|----------|------|
| Agent 分解子任务 | `agentrt_agent_fork()` | 独立租户 + 预算隔离 |
| Agent 内部多线程 | `clone(CLONE_THREAD)` | 共享地址空间 |
| 传统应用迁移 | `fork()` | POSIX 兼容 |
| Agent 容器启动 | `agentrt_agent_create()` | 完整 Agent 生命周期 |

---

## 4. mmap() 与 MemoryRovol 集成

### 4.1 语义扩展

`mmap()` 在 agentrt-liunx 中扩展支持映射 MemoryRovol 卷：

```c
/* 新增 MAP_MEMORYROVOL 标志 */
#define MAP_MEMORYROVOL  0x100000   /* 映射 MemoryRovol 卷 */

/* 新增 memoryrovol fd 类型 */
int fd = open("memoryrovol://agent-100/L2", O_RDWR);
void *addr = mmap(NULL, size, PROT_READ | PROT_WRITE,
		  MAP_SHARED | MAP_MEMORYROVOL, fd, 0);
/* addr 指向 MemoryRovol L2 向量索引区域 */
```

### 4.2 兼容性保证

| 调用方式 | 行为 |
|----------|------|
| `mmap(addr, len, prot, flags, fd, offset)` 标准调用 | 完全 POSIX 兼容 |
| `mmap()` + `MAP_MEMORYROVOL` | agentrt-liunx 扩展 |
| 在非 agentrt-liunx 上用 `MAP_MEMORYROVOL` | 返回 -EINVAL（未知标志） |

---

## 5. sched_setscheduler() 与 SCHED_AGENT 集成

### 5.1 调度类扩展

```c
/*
 * 复用内核 SCHED_EXT=7 调度类编号，禁止定义 SCHED_AGENT 宏。
 * SCHED_AGENT 仅作为 sched_ext BPF 策略名称（字符串），非调度类编号。
 * 参见 include/uapi/linux/sched.h:121（OLK-6.6 已定义 SCHED_EXT=7）。
 */
#define SCHED_AGENT_NAME  "sched_agent"  /* BPF 策略名称，非调度类编号 */

/* 扩展 sched_param 结构 */
struct sched_param {
	int sched_priority;
#ifdef CONFIG_AIRYMAXOS_AGENT
	/* agentrt-liunx 扩展字段 */
	uint32_t agent_id;
	uint64_t token_budget;
	uint8_t  cognition_phase;  /* 认知阶段 */
#endif
};
```

### 5.2 兼容性保证

| 调用方式 | 行为 |
|----------|------|
| `sched_setscheduler(pid, SCHED_OTHER, &param)` | POSIX 兼容 |
| `sched_setscheduler(pid, SCHED_EXT, &param)` | agentrt-liunx 扩展（复用 SCHED_EXT=7） |
| 非 Agent 进程设置 SCHED_EXT 策略 | 返回 -EPERM |

---

## 6. 跨发行版二进制兼容

### 6.1 glibc 兼容性

agentrt-liunx 内置 glibc 2.34+，与主流发行版二进制兼容：

| 发行版 | glibc 版本 | 二进制兼容 |
|--------|------------|------------|
| Ubuntu 22.04+ | 2.35+ | 兼容 |
| Debian 12+ | 2.36+ | 兼容 |
| Fedora 36+ | 2.35+ | 兼容 |
| RHEL 9+ | 2.34+ | 兼容 |

### 6.2 兼容性测试

```bash
# 在 Ubuntu 22.04 上编译二进制
gcc -o myapp myapp.c

# 复制到 agentrt-liunx 运行
scp myapp agentrt-liunx-host:/tmp/
ssh agentrt-liunx-host /tmp/myapp
# 预期：正常运行，无需重新编译
```

### 6.3 动态链接兼容

agentrt-liunx 提供以下动态链接兼容：

| 库 | 兼容性 | 说明 |
|----|--------|------|
| glibc | 完全兼容 | 与主流发行版二进制兼容 |
| libagentrt.so | agentrt-liunx 专属 | SDK 库 |
| libcupolas.so | agentrt-liunx 专属 | 安全库 |
| libpthread | 兼容 | NPTL 实现 |

---

## 7. POSIX 测试套件验证

### 7.1 测试套件

agentrt-liunx 通过以下测试套件验证 POSIX 兼容性：

| 测试套件 | 验证范围 | 通过标准 |
|----------|----------|----------|
| PCTS（POSIX Conformance Test Suite） | POSIX.1-2017 | 95%+ 通过 |
| LSB（Linux Standard Base） | LSB 5.0 | 95%+ 通过 |
| LTP（Linux Test Project） | syscall 兼容性 | 98%+ 通过 |
| glibc test suite | glibc 兼容 | 99%+ 通过 |

### 7.2 已知不兼容项

| 项目 | 原因 | 影响 |
|------|------|------|
| 部分 `ptrace()` 功能 | Cupolas 安全策略 | 仅影响调试器 |
| `kexec_load()` | 仅 root | 不影响应用 |
| 部分 `ioctl()` | Cupolas 过滤 | 按需放行 |

---

## 8. 降级运行支持

### 8.1 非 agentrt-liunx 环境降级

Agent 应用在非 agentrt-liunx 环境下，SDK 自动降级：

```python
# SDK 检测宿主环境，自动降级
from agentrt import CognitionClient

client = CognitionClient()
result = client.process("hello")

# SDK 内部：
#   - agentrt-liunx: syscall(AGENTRT_SYS_COGNITION_PROCESS)
#   - 主流 Linux: 用户态消息队列调用 llm_d
#   - 无 daemon: 本地推理（功能受限）
```

### 8.2 降级能力对照

| 能力 | agentrt-liunx | 主流 Linux | 无 daemon |
|------|---------------|------------|-----------|
| SCHED_AGENT | 是 | 否（降级 CFS） | 否 |
| MemoryRovol | CXL + MGLRU | 文件系统 | 内存 |
| Cupolas | 内核态 | 用户态 | 无 |
| AgentsIPC | io_uring 零拷贝 | socket | 无 |

---

## 9. 测试与验证

### 9.1 兼容性测试矩阵

| 测试维度 | 验证方法 | 通过标准 |
|----------|----------|----------|
| POSIX syscall | PCTS 测试套件 | 95%+ 通过 |
| 跨发行版二进制 | Ubuntu/Debian 二进制运行 | 正常运行 |
| fork() vs agentrt_agent_fork() | 行为对比测试 | 语义符合设计 |
| mmap() MemoryRovol | 扩展功能测试 | 映射成功 |
| sched_setscheduler() | SCHED_AGENT 测试 | 调度类切换 |
| 降级运行 | 非 agentrt-liunx 测试 | 自动降级 |

### 9.2 fork() 对比测试

```bash
# 测试 fork() POSIX 兼容性
./test_posix_fork --expect=child_pid_returned

# 测试 agentrt_agent_fork() 扩展语义
./test_agent_fork --expect=agent_id_returned \
    --expect=cgroup_isolated \
    --expect=budget_independent
```

---

## 10. 相关文档

- `160-compatibility/README.md`（兼容性主索引）
- `160-compatibility/01-abi-stability.md`（ABI 稳定性设计）
- `160-compatibility/03-upstream-tracking.md`（上游追踪策略）
- `30-interfaces/01-syscalls.md`（系统调用接口）
- `140-application-development/01-agent-lifecycle.md`（Agent 生命周期）

---

## 11. 参考材料

- POSIX.1-2017（IEEE Std 1003.1-2017）
- Linux Standard Base 5.0
- Linux 6.6 `man 2 syscall`（系统调用手册）
- Linux 6.6 `Documentation/userspace-api/`
- glibc 2.34+ 兼容性文档
- PCTS（POSIX Conformance Test Suite）
- LTP（Linux Test Project）

---

> **文档结束** | agentrt-liunx（AirymaxOS）POSIX 兼容性设计 v0.1.1
> 遵循 IRON-9 v2 [IND] 完全独立层（POSIX 兼容性为 agentrt-liunx 专属设计）
