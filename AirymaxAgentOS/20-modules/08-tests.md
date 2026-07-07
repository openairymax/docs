# agentrt-liunx（AirymaxOS）测试设计文档（airymaxos-tests，极境测试）

> 子仓编号：08
> 子仓代号：极境测试（Airymax Tests）
> 文档版本：v1.0（2026-07-06）
> 设计基准：单元测试 + 集成测试 + 形式化验证 + Soak + 混沌
> 同源 agentrt：全模块测试

---

## 1. 子仓职责

`airymaxos-tests` 是 agentrt-liunx（AirymaxOS）的测试与验证子仓，承担以下核心职责：

1. **单元测试框架**：为各子仓提供单元测试框架。
2. **集成测试框架**：基于 agentrt-liunx 系统级测试套件，提供集成测试框架。
3. **形式化验证**：参考 seL4 风格，对微内核关键部分进行形式化验证。
4. **Soak Test**：72 小时持续运行的稳定性测试。
5. **混沌工程**：参考 Chaos Mesh，提供故障注入测试。
6. **性能基准测试**：性能基准与回归测试。
7. **eBPF 可观测性验证**：验证 eBPF 可观测性正确性。

测试覆盖全部 8 个子仓，确保 agentrt-liunx 的可靠性、稳定性与安全性。

---

## 2. 同源关系

| 维度 | agentrt（全模块测试） | agentrt-liunx（airymaxos-tests） |
|------|---------------------|------------------------------|
| 单元测试 | 全模块单元测试 | 全子仓单元测试 |
| 集成测试 | 模块间集成测试 | 子仓间集成测试（agentrt-liunx 自研） |
| 形式化验证 | 无 | seL4 风格形式化验证 |
| Soak Test | 长时间运行测试 | 72h Soak Test |
| 混沌工程 | 无 | Chaos Mesh 类似混沌测试 |
| 性能基准 | 性能测试 | 性能基准与回归 |

**同源传承要点**：
- 保留 agentrt 的"全模块测试"覆盖度。
- 升级为 OS 级测试，引入形式化验证与混沌工程。
- 基于 agentrt-liunx 系统级测试套件与 QA SIG 标准。

---

## 3. 目录结构

```
airymaxos-tests/
├── unit/                   # 单元测试框架
├── integration/            # 集成测试框架（agentrt-liunx 自研）
├── formal-verification/    # 形式化验证（seL4 风格）
├── soak/                   # Soak Test（72h 持续运行）
├── chaos/                  # 混沌工程（Chaos Mesh 类似）
├── benchmark/              # 性能基准测试
├── observability-verify/   # eBPF 可观测性验证
└── docs/
```

### 3.1 unit/（单元测试框架）

- `framework/`：单元测试框架（基于 Rust cargo test、Go testing、C googletest）。
- `cases/`：测试用例。
  - `kernel/`：内核单元测试。
  - `services/`：服务单元测试。
  - `security/`：安全单元测试。
  - `memory/`：记忆单元测试。
  - `cognition/`：认知单元测试。
  - `cloudnative/`：云原生单元测试。
  - `system/`：系统单元测试。
- `coverage/`：代码覆盖率工具（llvm-cov、tarpaulin）。

### 3.2 integration/（集成测试框架，agentrt-liunx 自研）

基于 **agentrt-liunx 集成测试框架**：
- `airymaxos-itf/`：agentrt-liunx 集成测试框架。
- `testcases/`：测试用例。
  - `cross-subrepo/`：跨子仓集成测试。
  - `end-to-end/`：端到端测试。
  - `compatibility/`：兼容性测试。
- `runner/`：测试运行器。
- `report/`：测试报告生成。

### 3.3 formal-verification/（形式化验证）

参考 **seL4 形式化验证**：
- `isabelle/`：Isabelle/HOL 证明。
- `coq/`：Coq 证明。
- `spec/`：形式化规约。
  - `kernel-spec/`：内核规约。
  - `capability-spec/`：capability 系统规约。
  - `ipc-spec/`：IPC 规约。
- `proof/`：证明脚本。
- `automation/`：自动化证明工具。

### 3.4 soak/（Soak Test）

- `72h-runner/`：72 小时持续运行框架。
- `workloads/`：工作负载。
  - `agent-workload/`：Agent 工作负载。
  - `llm-inference/`：LLM 推理负载。
  - `mixed/`：混合负载。
- `monitoring/`：监控（内存泄漏、性能衰减）。
- `analysis/`：结果分析。

### 3.5 chaos/（混沌工程）

参考 **Chaos Mesh**：
- `chaos-framework/`：混沌测试框架。
- `experiments/`：实验。
  - `pod-kill/`：进程杀死。
  - `network-delay/`：网络延迟。
  - `network-loss/`：网络丢包。
  - `disk-fill/`：磁盘填充。
  - `memory-stress/`：内存压力。
  - `cpu-burn/`：CPU 燃烧。
  - `clock-skew/`：时钟偏移。
- `steady-state/`：稳态假设验证。

### 3.6 benchmark/（性能基准测试）

- `micro-bench/`：微基准测试。
  - `ipc-latency/`：IPC 延迟。
  - `sched-latency/`：调度延迟。
  - `memory-throughput/`：内存吞吐。
  - `io-throughput/`：I/O 吞吐。
- `macro-bench/`：宏基准测试。
  - `agent-throughput/`：Agent 吞吐。
  - `llm-tokens-per-sec/`：LLM Token/s。
  - `end-to-end-latency/`：端到端延迟。
- `regression/`：回归测试（性能不退化）。

### 3.7 observability-verify/（eBPF 可观测性验证）

- `ebpf-correctness/`：eBPF 程序正确性验证。
- `trace-verify/`：追踪结果验证。
- `metric-verify/`：指标正确性验证。
- `log-verify/`：日志完整性验证。
- `compare/`：与传统工具对比（perf、strace）。

---

## 4. 核心特性

### 4.1 单元测试框架

**多语言支持**：
- Rust：`cargo test` + `tarpaulin` 覆盖率。
- Go：`go test` + `go test -cover`。
- C/C++：`googletest` + `llvm-cov`。

**覆盖率目标**：
- 内核核心代码：≥ 90%
- 服务代码：≥ 80%
- 工具代码：≥ 70%

### 4.2 集成测试框架（agentrt-liunx 集成测试标准）

基于 **agentrt-liunx 集成测试框架**：
- 测试用例以 shell 脚本 + 配置文件描述。
- 支持测试套件组织。
- 支持依赖管理。
- 支持并行执行。
- 支持 x86_64、aarch64 多架构。

**测试用例示例**：
```bash
#!/bin/bash
# testcases/cross-subrepo/agent-ipc.sh
source $OET_PATH/libs/locallibs/common_lib.sh

@test "Agent IPC zero-copy"
{
    result=$(agentctl ipc-test --zerocopy)
    check_result $result 0
}
```

### 4.3 形式化验证（seL4 风格）

参考 **seL4 形式化验证**：
- 使用 Isabelle/HOL 与 Coq 证明微内核关键部分。
- 验证对象：capability 系统、IPC 机制、调度器框架。
- 验证性质：安全性（safety）、活性（liveness）、终止性（termination）。

**验证范围**：
| 验证对象 | 验证性质 | 工具 |
|---------|---------|------|
| capability 系统 | 不可伪造、可撤销 | Isabelle/HOL |
| IPC 机制 | 无死锁、无消息丢失 | Coq |
| 调度器框架 | 公平性、有界延迟 | Isabelle/HOL |
| MemoryRovol | 快照一致性 | Coq |

### 4.4 Soak Test（72 小时持续运行）

**目标**：验证系统长时间运行的稳定性。

**工作负载**：
- Agent 工作负载：持续运行 CoreLoopThree。
- LLM 推理负载：持续 LLM 推理。
- 混合负载：Agent + LLM + I/O。

**监控指标**：
- 内存泄漏：RSS 是否持续增长。
- 性能衰减：吞吐/延迟是否退化。
- 句柄泄漏：fd 数量是否增长。
- 日志错误：错误日志数量。

**结果判定**：
- 72 小时无崩溃。
- 内存增长 < 5%。
- 性能衰减 < 10%。

### 4.5 混沌工程（Chaos Mesh 类似）

参考 **Chaos Mesh**：
- 故障注入：进程杀死、网络分区、磁盘故障等。
- 稳态假设：注入故障后系统应保持稳态。
- 自动恢复：验证系统自动恢复能力。

**实验类型**：
| 实验 | 描述 |
|------|------|
| Pod Kill | 随机杀死 Agent 进程 |
| Network Delay | 注入网络延迟 |
| Network Loss | 注入网络丢包 |
| Disk Fill | 填充磁盘 |
| Memory Stress | 内存压力 |
| CPU Burn | CPU 满载 |
| Clock Skew | 时钟偏移 |

### 4.6 性能基准测试

**微基准**：
- IPC 延迟：测量 io_uring IPC 延迟（μs 级）。
- 调度延迟：测量 sched_ext 调度延迟。
- 内存吞吐：测量 CXL/PMEM 内存带宽。
- I/O 吞吐：测量 io_uring I/O 吞吐。

**宏基准**：
- Agent 吞吐：Agent 每秒处理任务数。
- LLM Token/s：LLM 每秒生成 Token 数。
- 端到端延迟：Agent 端到端响应延迟。

**回归测试**：
- 每次提交运行性能基准。
- 与基线对比，性能不退化。

### 4.7 eBPF 可观测性验证

**验证内容**：
- eBPF 程序正确性：验证 eBPF 程序输出正确。
- 追踪完整性：验证追踪覆盖所有事件。
- 指标准确性：验证指标与实际一致。
- 日志完整性：验证日志无丢失。

**对比方法**：
- 与 perf 对比：性能事件计数对比。
- 与 strace 对比：系统调用追踪对比。
- 与传统监控对比：指标数值对比。

---

## 5. 微内核思想体现

### 5.1 形式化验证（seL4 风格）

遵循 **seL4 形式化验证**传统：
- 微内核关键部分经过形式化证明。
- 验证安全性（safety）：不会发生不好的事情。
- 验证活性（liveness）：好的事情最终会发生。

**验证范围**：
- capability 系统（最小权限保证）。
- IPC 机制（无死锁、无消息丢失）。
- 调度器框架（公平性、有界延迟）。

### 5.2 全面测试覆盖

覆盖全部 8 个子仓：
- 内核、服务、安全、记忆。
- 认知、云原生、系统、测试自身。

### 5.3 自动化测试

- CI/CD 集成：每次提交自动运行测试。
- 性能回归：自动检测性能退化。
- 形式化验证：自动运行证明检查。

---

## 6. agentrt-liunx 工程基线

- **agentrt-liunx 集成测试框架**：集成测试框架基线。
- **agentrt-liunx QA SIG**：质量保证最佳实践。
- **agentrt-liunx 性能测试**：性能基准基线。
- **agentrt-liunx 兼容性测试**：兼容性测试基线。

---

## 7. 前沿理论参考

| 理论 | 来源 | 应用 |
|------|------|------|
| seL4 形式化验证 | seL4 项目 | 微内核形式化验证 |
| 混沌工程 | Netflix | Chaos Mesh 类似混沌测试 |
| eBPF 可观测性 | Linux eBPF | 可观测性验证 |
| agentrt-liunx 集成测试框架 | agentrt-liunx | 集成测试框架 |
| Property-based testing | 学术研究 | 属性测试 |
| Fuzzing | 学术研究 | 模糊测试 |

---

## 8. 与其他子仓的协作

| 协作子仓 | 协作内容 |
|---------|---------|
| `airymaxos-kernel` | 内核单元测试、形式化验证、Soak Test |
| `airymaxos-services` | 服务集成测试、混沌工程（进程杀死） |
| `airymaxos-security` | 安全测试、capability 形式化验证 |
| `airymaxos-memory` | 内存测试、MemoryRovol 验证 |
| `airymaxos-cognition` | 认知测试、LLM 性能基准 |
| `airymaxos-cloudnative` | 云原生测试、K8s 集成测试 |
| `airymaxos-system` | 系统工具测试、DevStation 验证 |

---

## 9. 里程碑

| 阶段 | 目标 | 时间 |
|------|------|------|
| M1 | 单元测试框架 + 各子仓单元测试 | 2026 Q3 |
| M2 | 集成测试框架（agentrt-liunx 自研） | 2026 Q4 |
| M3 | Soak Test 框架 + 首次 72h 测试 | 2027 Q1 |
| M4 | 混沌工程框架 | 2027 Q2 |
| M5 | 性能基准测试 + 回归 | 2027 Q3 |
| M6 | 形式化验证（capability + IPC） | 2027 Q4 |

---

## 10. 参考

- seL4 形式化验证文档
- Isabelle/HOL 教程
- Coq 教程
- Chaos Mesh 项目文档
- agentrt-liunx 集成测试框架文档
- agentrt-liunx QA SIG 文档
- Linux 性能测试工具文档
- eBPF 可观测性文档
- agentrt 全模块测试设计文档
