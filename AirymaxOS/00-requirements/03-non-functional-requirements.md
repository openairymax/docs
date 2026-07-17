Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 非功能需求分析
> **文档定位**：agentrt-linux（AirymaxOS） 非功能需求（Non-Functional Requirements）的详细分析，回答"agentrt-linux 的能力要做到多好（质量属性约束）"。\
> **文档版本**：0.1.1\
> **最后更新**：2026-07-06\
> **上级文档**：[agentrt-linux 设计文档](README.md)

---

## 1. 概述

本文档定义 agentrt-linux 的非功能需求（NFR，Non-Functional Requirements），覆盖五大质量属性维度：

1. **性能需求（NFR-P）**：延迟、吞吐、Token 能效、启动时间、内存占用
2. **安全需求（NFR-S）**：capability、国密、机密计算、LSM、审计
3. **可靠性需求（NFR-R）**：Soak Test、混沌工程、形式化验证、故障恢复、灾备
4. **兼容性需求（NFR-C）**：RPM、dnf、systemd、SELinux、架构支持
5. **可观测性需求（NFR-O）**：Metrics、Tracing、Logging、Health Check

每条 NFR 约束至少一条功能需求（FR），并定义明确的量化指标与验证方法。

---

## 2. 性能需求（NFR-P）

性能需求关注 agentrt-linux 在 Agent 工作负载下的延迟、吞吐与资源效率。

### 2.1 NFR-P-001：调度延迟

| 项 | 内容 |
|---|---|
| 指标 | Agent 任务调度延迟 |
| 目标 | < 100ms（实时反馈场景 < 10ms） |
| 验证方法 | perf + ftrace 测量调度延迟 |
| 约束功能 | FR-001 内核调度、FR-042 认知层 |
| 业务来源 | BR-001 科研 Agent、BR-003 工业控制、BR-004 具身智能 |

**详细说明**：

- Agent 任务从提交到开始执行的调度延迟必须 < 100ms
- 实时反馈场景（如工业控制、具身智能）必须 < 10ms
- 调度延迟通过 EEVDF + 方案 C-Prime 用户态调度器保证
- 延迟分布 P99 < 100ms，P99.9 < 200ms

**验证方法**：

```bash
# 使用 perf 测量调度延迟
perf sched record -a -- sleep 60
perf sched latency

# 使用 ftrace 测量调度延迟
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_wakeup/enable
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable
```

### 2.2 NFR-P-002：IPC 消息吞吐

| 项 | 内容 |
|---|---|
| 指标 | IPC 消息吞吐 |
| 目标 | > 100K msg/s（io_uring 零拷贝） |
| 验证方法 | io_uring 基准测试 |
| 约束功能 | FR-003 IPC 子系统 |
| 业务来源 | BR-005 超节点协同 |

**详细说明**：

- IPC 消息吞吐必须 > 100K msg/s
- 基于 io_uring 零拷贝实现，避免 syscall 开销
- 消息延迟 < 10μs（单节点内）
- 消息头同源 agentrt AgentsIPC（128B 头）

### 2.3 NFR-P-003：节点间通信延迟

| 项 | 内容 |
|---|---|
| 指标 | 超节点间通信延迟 |
| 目标 | < 10μs（RDMA / NVLink / CXL） |
| 验证方法 | 节点间 ping-pong 测试 |
| 约束功能 | FR-048 超节点 OS |
| 业务来源 | BR-005 超节点 OS |

**详细说明**：

- 超节点间通信延迟必须 < 10μs
- 支持 RDMA、NVLink、CXL 高速互联
- 任务迁移延迟 < 100ms
- 通信吞吐 > 100GB/s

### 2.4 NFR-P-004：Token 能效

| 项 | 内容 |
|---|---|
| 指标 | Token 生成吞吐与能效 |
| 目标 | 吞吐 +30%，能效 +25%（参考 agentrt-linux Token 能效框架） |
| 验证方法 | Token 基准测试 + 能耗测量 |
| 约束功能 | FR-047 LLM 调度 |
| 业务来源 | BR-005 AI 原生 |

**详细说明**：

- Token 生成吞吐相比基线提升 > 30%
- 单位能耗的 Token 产出提升 > 25%
- 通过内核感知的推理调度（Token 能效框架）实现
- KV-cache 命中率 > 90%，批处理效率 > 85%

### 2.5 NFR-P-005：系统冷启动时间

| 项 | 内容 |
|---|---|
| 指标 | 系统冷启动时间 |
| 目标 | < 30s（systemd 优化，边缘部署 < 10s） |
| 验证方法 | systemd-analyze 测量 |
| 约束功能 | FR-014 systemd 服务管理 |
| 业务来源 | BR-003 工业边缘部署 |

**详细说明**：

- 系统冷启动时间必须 < 30s
- 边缘部署场景 < 10s
- 通过 systemd 并行启动与依赖优化实现
- 关键服务启动 < 5s

**验证方法**：

```bash
# 测量启动时间
systemd-analyze

# 分析启动瓶颈
systemd-analyze blame
systemd-analyze critical-chain
```

### 2.6 NFR-P-006：内存占用

| 项 | 内容 |
|---|---|
| 指标 | 基础系统内存占用 |
| 目标 | < 512MB（边缘部署 < 256MB） |
| 验证方法 | 系统空闲时内存测量 |
| 约束功能 | FR-004 内存管理、FR-068 系统裁剪 |
| 业务来源 | BR-003 工业边缘部署 |

**详细说明**：

- 基础系统内存占用 < 512MB
- 边缘部署场景 < 256MB
- 通过 MGLRU 多代 LRU（Linux 6.6）优化内存回收
- 系统裁剪可进一步降低内存占用

---

## 3. 安全需求（NFR-S）

安全需求对应设计原则「E-1 安全内生」，安全是系统的内生属性而非附加层。

### 3.1 NFR-S-001：capability 安全模型

| 项 | 内容 |
|---|---|
| 指标 | capability 安全模型完整性 |
| 目标 | seL4 风格 capability，未授权访问 100% 拒绝 |
| 验证方法 | 渗透测试 + 形式化验证 |
| 约束功能 | FR-021 capability 安全模型 |
| 业务来源 | BR-007 SELinux 对齐 |

**详细说明**：

- 采用 seL4 风格的 capability 安全模型
- 所有守护进程默认无权限，必须通过策略文件显式授予
- 权限策略支持运行时动态更新，更新时重新评估所有活动会话
- capability 的传递、委托、撤销有完整的审计追踪
- 形式化验证 capability 模型的安全性

**验证方法**：

- 未授权访问测试：100% 拒绝
- 权限提升攻击测试：100% 防御
- capability 传递链追踪：100% 可追溯

### 3.2 NFR-S-002：国密算法支持

| 项 | 内容 |
|---|---|
| 指标 | 国密算法支持完整性 |
| 目标 | SM2/SM3/SM4/SM9 全支持，内核态加速 |
| 验证方法 | 国密合规性测试 |
| 约束功能 | FR-023 国密算法 |
| 业务来源 | BR-007 国密支持 |

**详细说明**：

- 完整支持 SM2（非对称加密）、SM3（哈希）、SM4（对称加密）、SM9（标识加密）
- 国密算法内核态加速，性能 > 10Gbps
- TLS 1.3 国密套件支持
- 国密签名的 RPM 包验证
- 国密的证书体系

**验证方法**：

- 国密算法正确性测试（与标准测试向量对比）
- 国密算法性能测试（吞吐与延迟）
- 国密 TLS 套件兼容性测试

### 3.3 NFR-S-003：机密计算

| 项 | 内容 |
|---|---|
| 指标 | 机密计算支持完整性 |
| 目标 | Intel SGX / AMD SEV / ARM TrustZone 全支持 |
| 验证方法 | 飞地隔离测试 |
| 约束功能 | FR-024 机密计算 |
| 业务来源 | BR-005 AI 原生 |

**详细说明**：

- 支持 Intel SGX 飞地隔离
- 支持 AMD SEV 虚拟机加密
- 支持 ARM TrustZone 安全世界
- 飞地内的数据加密保护
- 飞地间的安全通信

### 3.4 NFR-S-004：LSM hook

| 项 | 内容 |
|---|---|
| 指标 | LSM hook 兼容性 |
| 目标 | SELinux + Landlock 双 LSM 支持 |
| 验证方法 | LSM 策略测试 |
| 约束功能 | FR-022 LSM hook |
| 业务来源 | BR-007 SELinux 对齐 |

**详细说明**：

- 支持 SELinux 策略完整兼容
- 支持 Landlock 用户态沙箱
- 双 LSM 可同时启用（Linux 5.14+）
- LSM hook 的审计日志

### 3.5 NFR-S-005：审计日志不可篡改

| 项 | 内容 |
|---|---|
| 指标 | 审计日志完整性 |
| 目标 | SHA-256 哈希链，不可篡改 |
| 验证方法 | 哈希链验证测试 |
| 约束功能 | FR-026 审计追踪 |
| 业务来源 | BR-007 安全审计 |

**详细说明**：

- 审计日志采用 SHA-256 哈希链，每条日志包含前一条的哈希
- 日志一旦写入不可修改（仅追加）
- 任何篡改都能通过哈希链验证检测
- 日志的长期存储与归档

**验证方法**：

```bash
# 验证哈希链完整性
airymaxos-audit-verify /var/log/audit/audit.log

# 检测篡改
airymaxos-audit-check --tamper-detect
```

### 3.6 NFR-S-006：输入净化

| 项 | 内容 |
|---|---|
| 指标 | 注入攻击防御率 |
| 目标 | > 99.9%（SQL/命令/权限提升注入） |
| 验证方法 | 渗透测试 + 模糊测试 |
| 约束功能 | FR-025 输入净化管道 |
| 业务来源 | BR-002 客服 Agent |

**详细说明**：

- 四阶段输入净化管道：正则过滤 → 类型检查 → 长度限制 → 语义验证
- SQL 注入防御率 > 99.9%
- 命令注入防御率 > 99.9%
- 权限提升防御率 100%

### 3.7 NFR-S-007：沙箱隔离

| 项 | 内容 |
|---|---|
| 指标 | 沙箱逃逸防御率 |
| 目标 | 100%（进程/容器/WASM 沙箱） |
| 验证方法 | 沙箱逃逸测试 |
| 约束功能 | FR-027 沙箱隔离 |
| 业务来源 | BR-004 具身智能 |

**详细说明**：

- 进程沙箱：seccomp + namespaces
- 容器沙箱：OCI runtime + namespaces + cgroups
- WASM 沙箱：Wasm 3.0 安全运行时
- 沙箱逃逸防御率 100%

### 3.8 NFR-S-008：镜像签名验证

| 项 | 内容 |
|---|---|
| 指标 | 镜像签名验证 |
| 目标 | 未签名镜像 100% 拒绝 |
| 验证方法 | 镜像签名测试 |
| 约束功能 | FR-055 镜像签名验证 |
| 业务来源 | BR-006 云原生 |

**详细说明**：

- 镜像的 cosign 数字签名
- 未签名镜像 100% 拒绝运行
- 签名验证 < 1s
- 签名密钥的轮换与撤销

---

## 4. 可靠性需求（NFR-R）

可靠性需求关注 agentrt-linux 在长时间运行下的稳定性与故障恢复能力。

### 4.1 NFR-R-001：Soak Test

| 项 | 内容 |
|---|---|
| 指标 | 7×24 持续运行稳定性 |
| 目标 | 7 天无故障，无内存泄漏 |
| 验证方法 | Soak Test 自动化 |
| 约束功能 | FR-075 Soak Test |
| 业务来源 | BR-003 工业控制 |

**详细说明**：

- 系统必须支持 7×24 小时持续运行
- 7 天内无故障停机
- 无内存泄漏（通过 ASan 检测）
- 无资源泄漏（文件句柄、线程等）
- 性能不退化（吞吐与延迟保持稳定）

**验证方法**：

```bash
# 启动 Soak Test
airymaxos-soak-test --duration 7d --workload agent-mixed

# 内存泄漏检测
valgrind --leak-check=full --show-leak-kinds=all ./agent-workload

# 资源监控
airymaxos-monitor --resources --interval 60s --duration 7d
```

### 4.2 NFR-R-002：混沌工程

| 项 | 内容 |
|---|---|
| 指标 | 故障注入下的韧性 |
| 目标 | 故障恢复 < 5min，数据不丢失 |
| 验证方法 | Chaos Mesh 类混沌测试 |
| 约束功能 | FR-076 混沌工程 |
| 业务来源 | BR-006 云原生 |

**详细说明**：

- 支持网络故障注入（延迟、丢包、分区）
- 支持进程故障注入（kill、stop、hang）
- 支持资源故障注入（CPU、内存、磁盘）
- 故障恢复时间 < 5min
- 故障期间数据不丢失

### 4.3 NFR-R-003：形式化验证

| 项 | 内容 |
|---|---|
| 指标 | 关键路径形式化验证 |
| 目标 | seL4 风格验证，关键路径 100% 通过 |
| 验证方法 | 形式化验证工具 |
| 约束功能 | FR-077 形式化验证 |
| 业务来源 | BR-003 工业控制 |

**详细说明**：

- 关键路径（调度、IPC、capability）采用 seL4 风格形式化验证
- 验证内存安全（无缓冲区溢出、无野指针）
- 验证类型安全（无类型混淆）
- 验证并发安全（无死锁、无竞态）
- 验证功能正确性（行为符合规约）

### 4.4 NFR-R-004：故障恢复

| 项 | 内容 |
|---|---|
| 指标 | MTTR（平均恢复时间） |
| 目标 | MTTR < 5min |
| 验证方法 | 故障注入 + 恢复测试 |
| 约束功能 | FR-043 执行层补偿事务 |
| 业务来源 | BR-003 工业控制、BR-006 云原生 |

**详细说明**：

- 故障恢复时间（MTTR）< 5min
- 自动故障检测（健康检查）
- 自动故障转移（主备切换）
- 自动故障恢复（重启、重试、回滚）
- 补偿事务保证数据一致性

### 4.5 NFR-R-005：灾备

| 项 | 内容 |
|---|---|
| 指标 | RPO / RTO |
| 目标 | RPO < 1min，RTO < 30min |
| 验证方法 | 灾备演练 |
| 约束功能 | FR-035 CXL 内存池化 |
| 业务来源 | BR-006 云原生 |

**详细说明**：

- RPO（恢复点目标）< 1min，数据丢失 < 1min
- RTO（恢复时间目标）< 30min，恢复时间 < 30min
- 数据的异地备份与恢复
- 系统的异地容灾

### 4.6 NFR-R-006：资源确定性

| 项 | 内容 |
|---|---|
| 指标 | 资源生命周期确定性 |
| 目标 | 无资源泄漏，所有权明确 |
| 验证方法 | ASan + TSan + 泄漏检测 |
| 约束功能 | FR-004 内存管理 |
| 业务来源 | 全部业务需求 |

**详细说明**：

- 每个资源有明确的所有者
- 资源释放通过 RAII 或 cleanup attribute 自动化
- 禁止隐式资源转移
- 异步资源显式关联所有者
- 调试模式下记录资源分配栈

---

## 5. 兼容性需求（NFR-C）

兼容性需求确保 agentrt-linux 与企业级 Linux 生态的兼容性。

### 5.1 NFR-C-001：Linux 企业级生态 RPM 包格式兼容

| 项 | 内容 |
|---|---|
| 指标 | RPM 包格式兼容性 |
| 目标 | RPM 4.x 100% 兼容 |
| 验证方法 | agentrt-linux 集成测试框架测试 |
| 约束功能 | FR-061 RPM 包格式 |
| 业务来源 | BR-007 Linux 企业级生态对齐 |

**详细说明**：

- RPM 4.x 包格式 100% 兼容
- 企业级 Linux 生态 SPEC 文件可直接构建
- 软件源格式兼容
- GPG 签名验证兼容

### 5.2 NFR-C-002：dnf 包管理器兼容

| 项 | 内容 |
|---|---|
| 指标 | dnf 兼容性 |
| 目标 | dnf 4.x 100% 兼容 |
| 验证方法 | dnf 操作测试 |
| 约束功能 | FR-062 dnf 包管理器 |
| 业务来源 | BR-007 Linux 企业级生态对齐 |

**详细说明**：

- dnf 4.x 100% 兼容
- 仓库管理兼容
- 依赖解析兼容
- 事务管理兼容

### 5.3 NFR-C-003：systemd 服务管理兼容

| 项 | 内容 |
|---|---|
| 指标 | systemd 兼容性 |
| 目标 | systemd 255+ 100% 兼容 |
| 验证方法 | systemd 操作测试 |
| 约束功能 | FR-014 systemd 服务管理 |
| 业务来源 | BR-007 Linux 企业级生态对齐 |

**详细说明**：

- systemd 255+ 100% 兼容
- unit 文件格式兼容
- 服务编排兼容
- journald 日志兼容

### 5.4 NFR-C-004：SELinux 安全模块兼容

| 项 | 内容 |
|---|---|
| 指标 | SELinux 兼容性 |
| 目标 | SELinux 策略 100% 兼容 |
| 验证方法 | SELinux 策略测试 |
| 约束功能 | FR-022 LSM hook |
| 业务来源 | BR-007 Linux 企业级生态对齐 |

**详细说明**：

- SELinux 策略 100% 兼容
- enforcing / permissive / disabled 三种模式
- 安全上下文管理兼容
- AVC 审计日志兼容

### 5.5 NFR-C-005：架构支持

| 项 | 内容 |
|---|---|
| 指标 | 多架构支持 |
| 目标 | x86_64 / ARM64 双主力，RISC-V 实验性 |
| 验证方法 | 多架构构建与测试 |
| 约束功能 | FR-069 多架构构建 |
| 业务来源 | BR-007 架构支持 |

**详细说明**：

- x86_64 主力支持（Intel / AMD）
- ARM64 主力支持（鲲鹏 920 / 飞腾 2000+）
- RISC-V 实验性支持（SiFive U740）
- LoongArch 规划支持（龙芯 3A6000）
- 跨架构的统一构建（CMake）

### 5.6 NFR-C-006：K8s 兼容性

| 项 | 内容 |
|---|---|
| 指标 | K8s 兼容性 |
| 目标 | K8s v1.30+ 100% 兼容 |
| 验证方法 | K8s conformance 测试 |
| 约束功能 | FR-051 K8s CRD |
| 业务来源 | BR-006 云原生 |

**详细说明**：

- K8s v1.30+ API 100% 兼容
- 通过 K8s conformance 认证
- CRD 扩展兼容
- 自定义调度器兼容

---

## 6. 可观测性需求（NFR-O）

可观测性需求对应设计原则「E-2 可观测性原则」，确保系统可被理解、调试与优化。

### 6.1 NFR-O-001：Metrics 指标

| 项 | 内容 |
|---|---|
| 指标 | Metrics 完整性 |
| 目标 | Prometheus 格式，关键路径 100% 覆盖 |
| 验证方法 | Metrics 覆盖检查 |
| 约束功能 | 全部功能需求 |
| 业务来源 | 全部业务需求 |

**详细说明**：

- 所有关键路径必须有 Metrics
- Metrics 采用 Prometheus 格式
- 包含性能指标（延迟、吞吐、错误率）
- 包含资源指标（CPU、内存、磁盘、网络）
- 包含业务指标（任务数、Token 数、会话数）

**Metrics 格式示例**：

```prometheus
# 性能指标
airy_task_latency_seconds{quantile="0.99"} 0.095
airy_ipc_throughput_msgs_per_second 125000
airy_error_rate 0.001

# 资源指标
airy_cpu_usage_percent 45.2
airy_memory_usage_bytes 536870912
airy_disk_io_bytes_per_second 1048576

# 业务指标
airy_agent_tasks_total 12345
airy_tokens_generated_total 9876543
airy_active_sessions 1500
```

### 6.2 NFR-O-002：Tracing 链路追踪

| 项 | 内容 |
|---|---|
| 指标 | Tracing 覆盖率 |
| 目标 | OpenTelemetry 格式，关键路径 100% 覆盖 |
| 验证方法 | Tracing 覆盖检查 |
| 约束功能 | 全部功能需求 |
| 业务来源 | 全部业务需求 |

**详细说明**：

- 所有公共 API 的入口和出口必须记录追踪 span
- Tracing 采用 OpenTelemetry 格式
- 分布式追踪支持跨节点调用链
- 关键路径的延迟通过直方图监控
- 追踪开销 < 5%

**Tracing 格式示例**：

```json
{
  "trace_id": "abc123def456",
  "span_id": "span001",
  "parent_span_id": "span000",
  "operation": "airy_cognition_process",
  "start_time": "2026-07-06T10:30:45.123Z",
  "duration_ms": 45.6,
  "attributes": {
    "task_id": "task_xyz789",
    "agent_role": "researcher",
    "model": "gpt-4"
  }
}
```

### 6.3 NFR-O-003：Logging 结构化日志

| 项 | 内容 |
|---|---|
| 指标 | Logging 结构化 |
| 目标 | 结构化 JSON 日志 + ANSI 颜色 |
| 验证方法 | 日志格式检查 |
| 约束功能 | 全部功能需求 |
| 业务来源 | 全部业务需求 |

**详细说明**：

- 所有日志必须结构化（JSON 格式）
- 日志包含 trace_id 和完整上下文
- 日志包含模块名、函数名、行号
- 控制台输出支持 ANSI 颜色（开发友好）
- 文件输出为纯 JSON（机器可解析）
- 日志级别：DEBUG / INFO / WARN / ERROR / FATAL

**日志格式示例**：

```json
{
  "timestamp": "2026-07-06T10:30:45.123Z",
  "level": "INFO",
  "trace_id": "req_abc123",
  "module": "coreloopthree.cognition",
  "function": "airy_cognition_process",
  "line": 142,
  "message": "任务规划完成",
  "context": {
    "task_id": "task_xyz789",
    "plan_nodes": 15,
    "estimated_time_ms": 4500
  }
}
```

### 6.4 NFR-O-004：Health Check 健康检查

| 项 | 内容 |
|---|---|
| 指标 | Health Check 完整性 |
| 目标 | 依赖项状态 100% 覆盖 |
| 验证方法 | 健康检查测试 |
| 约束功能 | 全部功能需求 |
| 业务来源 | 全部业务需求 |

**详细说明**：

- 健康检查必须包含依赖项状态（不能只报告自己健康）
- 健康检查包含：自身状态 + 依赖服务状态 + 资源状态
- 健康检查的 API 标准化（/health、/ready、/live）
- 健康检查支持深度检查（deep check）与浅度检查（shallow check）

**Health Check 响应示例**：

```json
{
  "status": "healthy",
  "timestamp": "2026-07-06T10:30:45.123Z",
  "checks": {
    "self": {
      "status": "healthy",
      "uptime_seconds": 86400
    },
    "dependencies": {
      "database": {
        "status": "healthy",
        "latency_ms": 2.3
      },
      "redis": {
        "status": "healthy",
        "latency_ms": 0.8
      },
      "llm_service": {
        "status": "degraded",
        "latency_ms": 1500,
        "message": "response time above threshold"
      }
    },
    "resources": {
      "cpu_usage_percent": 45.2,
      "memory_usage_percent": 62.5,
      "disk_usage_percent": 38.1
    }
  }
}
```

### 6.5 NFR-O-005：错误可追溯

| 项 | 内容 |
|---|---|
| 指标 | 错误可追溯性 |
| 目标 | 错误链完整，调用栈可追溯 |
| 验证方法 | 错误链测试 |
| 约束功能 | 全部功能需求 |
| 业务来源 | 全部业务需求 |

**详细说明**：

- 每个错误有唯一的错误码
- 错误消息包含上下文信息（任务 ID、文件路径等）
- 错误通过结构化日志记录
- 关键错误触发告警
- 错误链（wrapping）传递上下文，格式化显示完整调用栈

**错误链示例**：

```
[ERROR] AIRY_ENOMEM: 内存分配失败
  at coreloopthree/cognition/planner.c:142 (airy_planner_expand)
  at coreloopthree/cognition/cognition.c:87 (airy_cognition_process)
  at syscalls/syscall.c:234 (airy_sys_task_submit)
  context: requested=256MB, available=128MB, process=airy_cogn_d, pid=1234
  suggestion: 1) 减少批量大小 2) 启用内存压缩 3) 增加系统内存
```

---

## 7. 非功能需求优先级

| 优先级 | NFR 系列 | 说明 |
|---|---|---|
| P0 | NFR-S 安全 | 安全是内生需求，必须优先满足 |
| P0 | NFR-C 兼容性 | 生态兼容是可用性基础 |
| P0 | NFR-P 性能 | 性能是 Agent 工作负载的核心 |
| P1 | NFR-R 可靠性 | 可靠性是生产部署的保障 |
| P1 | NFR-O 可观测性 | 可观测性是运维的基础 |

---

## 8. 非功能需求验收方法

每条 NFR 必须有明确的验收方法与量化指标，对应设计原则「E-8 可测试性原则」：

| NFR 系列 | 验收方法 | 验收工具 | 责任子仓 |
|---|---|---|---|
| NFR-P 性能 | 性能基准测试 | Locust + k6 + perf + ftrace | tests-linux |
| NFR-S 安全 | 渗透测试 + 形式化验证 | seL4 风格验证 + 静态分析 | tests-linux + security |
| NFR-R 可靠性 | Soak Test + 混沌工程 | Chaos Mesh + agentrt-linux 系统级测试套件 + ASan/TSan | tests-linux |
| NFR-C 兼容性 | 兼容性测试矩阵 | agentrt-linux 集成测试框架 + K8s conformance | tests-linux + system |
| NFR-O 可观测性 | 可观测性覆盖检查 | Prometheus + OpenTelemetry + 日志检查 | tests-linux |

---

## 9. 非功能需求约束关系

非功能需求对功能需求的约束关系如下表所示：

| NFR | 约束的 FR | 约束内容 |
|---|---|---|
| NFR-P-001 调度延迟 | FR-001 内核调度 | 延迟 < 100ms |
| NFR-P-002 IPC 吞吐 | FR-003 IPC 子系统 | 吞吐 > 100K msg/s |
| NFR-P-004 Token 能效 | FR-047 LLM 调度 | 能效 +25% |
| NFR-S-001 capability | FR-021 capability 安全 | 未授权 100% 拒绝 |
| NFR-S-005 审计不可篡改 | FR-026 审计追踪 | 哈希链验证 |
| NFR-R-001 Soak Test | FR-075 Soak Test | 7 天无故障 |
| NFR-R-003 形式化验证 | FR-077 形式化验证 | 关键路径通过 |
| NFR-C-001 RPM 兼容 | FR-061 RPM 包格式 | RPM 4.x 兼容 |
| NFR-O-001 Metrics | 全部 FR | Prometheus 格式 |
| NFR-O-003 Logging | 全部 FR | 结构化 JSON |

---

## 10. 非功能需求与设计原则映射

非功能需求与「五维正交 24 原则」的映射关系：

| NFR 系列 | 对应设计原则 | 映射说明 |
|---|---|---|
| NFR-P 性能 | S-1 反馈闭环、E-3 资源确定性 | 性能通过反馈闭环持续优化 |
| NFR-S 安全 | E-1 安全内生、K-1 内核极简 | 安全是内生属性 |
| NFR-R 可靠性 | S-4 涌现性管理、E-3 资源确定性 | 抑制负面涌现（级联故障） |
| NFR-C 兼容性 | E-4 跨平台一致性 | 跨平台、跨架构一致性 |
| NFR-O 可观测性 | E-2 可观测性、E-6 错误可追溯 | 系统可被理解与调试 |

---

## 11. 相关文档

- [需求分析概览](README.md)：需求分层模型与追溯框架
- [业务需求分析](01-business-requirements.md)：Agent 工作负载与生态对齐
- [功能需求分析](02-functional-requirements.md)：8 子仓功能矩阵与能力清单
- [agentrt-linux 总览](../README.md)：agentrt-linux 整体设计
- [Airymax 架构设计原则](../../AirymaxRT/00-architectural-principles.md)：五维正交 24 原则

---

## 12. 文档变更记录

| 版本 | 日期 | 变更内容 | 变更人 |
|---|---|---|---|
| 0.1.1 | 2026-07-06 | 初始版本，定义 5 大系列 30+ 条非功能需求 | 工程规范委员会 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."
