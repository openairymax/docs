# AirymaxOS 认知设计文档（airymaxos-cognition，极境认知）

> 子仓编号：05
> 子仓代号：极境认知（Airymax Cognition）
> 文档版本：v1.0（2026-07-06）
> 设计基准：CoreLoopThree kthread + Wasm + LLM 调度
> 同源 agentrt：coreloopthree + frameworks（CoreLoopThree + Thinkdual）

---

## 1. 子仓职责

`airymaxos-cognition` 是 AirymaxOS 的认知与 AI 推理子仓，承担以下核心职责：

1. **CoreLoopThree kthread 实现**：将 agentrt 的 CoreLoopThree（三层认知循环）升级为 OS 级 kthread 实现，提供 Agent 认知循环的内核态加速。
2. **Thinkdual 双思考系统内核态加速**：将 agentrt 的 Thinkdual（双思考系统）通过内核态加速提升响应速度。
3. **Wasm runtime 3.0**：集成 Wasm 3.0 runtime，提供安全沙箱执行环境。
4. **LLM 推理感知调度**：基于 AirymaxOS 认知中枢，实现 LLM 推理任务的感知调度。
5. **GPU/NPU 调度与池化**：统一调度 GPU/NPU 异构算力。
6. **Token 能效优化**：参考 KVC-Gateway + LMCache + Bifrost 优化 Token 能效。
7. **超节点沙箱**：基于 AirymaxOS 超节点 OS，实现软硬协同优化镜像快照。
8. **具身智能支持**：基于 AirymaxOS Claw 提供具身智能运行时支持。

---

## 2. 同源关系

| 维度 | agentrt（coreloopthree + frameworks） | AirymaxOS（airymaxos-cognition） |
|------|--------------------------------------|----------------------------------|
| 认知循环 | CoreLoopThree（用户态） | CoreLoopThree kthread（内核态） |
| 双思考 | Thinkdual（用户态） | Thinkdual 内核态加速 |
| 推理调度 | 用户态调度器 | LLM 推理感知调度（基于 AirymaxOS 认知中枢） |
| 算力调度 | 应用层调度 | GPU/NPU 调度与池化（OS 级） |
| 沙箱 | 进程沙箱 | Wasm 3.0 + 超节点沙箱 |

**同源传承要点**：
- 保留 agentrt CoreLoopThree 的"三层认知循环"语义（感知-思考-行动）。
- 保留 Thinkdual 的"双思考系统"架构（快思考 + 慢思考）。
- 升级为 OS 级实现，获得内核态加速与硬件感知。

---

## 3. 目录结构

```
airymaxos-cognition/
├── coreloopthree/          # CoreLoopThree kthread 实现
├── thinkdual/             # Thinkdual 双思考系统内核态加速
├── wasm-runtime/           # Wasm 3.0 runtime（安全沙箱）
├── llm-scheduler/          # LLM 推理感知调度
├── gpu-npu/                # GPU/NPU 调度与池化
├── token-efficiency/       # Token 能效优化（AirymaxOS 自研）
├── super-node-sandbox/     # 超节点沙箱（AirymaxOS 自研）
├── embodied-ai/            # 具身智能支持
└── docs/
```

### 3.1 coreloopthree/（CoreLoopThree kthread 实现）

参考 agentrt CoreLoopThree 设计：
- `clt-kmod`：内核模块，注册 kthread 执行 CoreLoopThree。
- `perception-loop`：感知循环（输入采集）。
- `thinking-loop`：思考循环（LLM 推理）。
- `action-loop`：行动循环（输出执行）。
- `phase-notify`：阶段通知（与 sched_ext sub-scheduler 协作）。

### 3.2 thinkdual/（Thinkdual 双思考系统内核态加速）

参考 agentrt Thinkdual 设计：
- `system1`：快思考（直觉式，低延迟路径）。
- `system2`：慢思考（推理式，高准确度路径）。
- `switcher`：快慢思考切换器（基于任务复杂度）。
- `kernel-accel`：内核态加速（共享内存、零拷贝数据传递）。

### 3.3 wasm-runtime/（Wasm 3.0 runtime）

集成 **Wasm 3.0** runtime（2026 成熟）：
- `wasmtime`：Wasmtime runtime 集成。
- `wamr`：WAMR（WebAssembly Micro Runtime）集成。
- `component-model`：Wasm Component Model 支持。
- `wasi`：WASI（WebAssembly System Interface）支持。
- `simd`：Wasm SIMD 指令支持。
- `gc`：Wasm GC（垃圾回收）支持。
- `threads`：Wasm 线程支持。

### 3.4 llm-scheduler/（LLM 推理感知调度）

基于 **AirymaxOS 认知中枢**：
- `inference-aware`：推理感知调度器（识别 LLM 推理阶段）。
- `kv-cache-aware`：KV Cache 感知调度。
- `batch-scheduler`：动态 batching 调度。
- `prefill-decode`：prefill 与 decode 阶段分离调度。
- `speculative-decoding`：投机解码调度支持。

### 3.5 gpu-npu/（GPU/NPU 调度与池化）

- `gpu-scheduler`：GPU 调度器（与 DRM/KMS 协作）。
- `npu-scheduler`：NPU 调度器（与厂商驱动协作）。
- `mps`：MPS（Multi-Process Service）支持。
- `mig`：MIG（Multi-Instance GPU）支持。
- `pooling`：算力池化（跨设备调度）。
- `vfio-mdev`：VFIO-mdev 虚拟化支持。

### 3.6 token-efficiency/（Token 能效优化）

参考 **KVC-Gateway + LMCache + Bifrost**：
- `kvc-gateway`：KV Cache 网关（跨请求复用）。
- `lmcache`：LMCache 集成（KV Cache 跨节点缓存）。
- `bifrost`：Bifrost 集成（推测解码加速）。
- `prefix-cache`：前缀缓存（共享 prompt 复用）。
- `quantization`：量化支持（INT8/INT4）。
- `speculative`：投机解码优化。

### 3.7 super-node-sandbox/（超节点沙箱）

基于 **AirymaxOS 超节点 OS**：
- `snapshot`：镜像快照（软硬协同优化）。
- `restore`：快照恢复（基于 userfaultfd）。
- `clone`：快速克隆（COW 共享）。
- `migrate`：跨节点迁移（与 MemoryRovol 协作）。
- `scheduling`：超节点调度（NUMA 感知）。

### 3.8 embodied-ai/（具身智能支持）

基于 **AirymaxOS Claw**：
- `sensor-hub`：传感器数据汇聚。
- `motor-control`：运动控制接口。
- `realtime-loop`：实时控制循环。
- `perception-fusion`：多模态感知融合。
- `safety-monitor`：安全监控（紧急停止）。

---

## 4. 核心特性

### 4.1 CoreLoopThree kthread 实现（三层认知循环 OS 化）

**三层循环**：
1. **Perception Loop（感知循环）**：采集多模态输入（文本、图像、音频、传感器）。
2. **Thinking Loop（思考循环）**：LLM 推理，决策制定。
3. **Action Loop（行动循环）**：执行决策，输出结果。

**kthread 实现**：
- CoreLoopThree 作为内核 kthread 运行，减少用户态/内核态切换开销。
- 阶段通知通过 sched_ext 接口传递给 sub-scheduler。
- sub-scheduler 根据阶段动态调整调度策略（思考阶段优先级高）。

**与 agentrt 同源**：保留 CoreLoopThree 的 API 兼容性，但底层实现升级为 kthread。

### 4.2 Thinkdual 双思考系统内核态加速

**双思考架构**（参考 Daniel Kahneman "Thinking, Fast and Slow"）：
- **System 1（快思考）**：直觉式、低延迟、低能耗。适用于简单决策。
- **System 2（慢思考）**：推理式、高延迟、高能耗。适用于复杂推理。

**内核态加速**：
- System 1 与 System 2 共享内核态内存（零拷贝数据传递）。
- 切换器在内核态运行，减少切换延迟。
- LLM 推理任务通过 io_uring 提交至 System 2。

### 4.3 Wasm runtime 3.0（安全沙箱，2026 成熟）

**Wasm 3.0 特性**：
- Component Model：跨语言组件互操作。
- WASI Preview 3：完整系统接口。
- GC：垃圾回收支持。
- Threads：多线程支持。
- SIMD：向量指令支持。
- Exception Handling：异常处理。

**安全沙箱**：
- 内存隔离（线性内存模型）。
- capability-based security（WASI capability）。
- 资源限制（fuel metering）。
- 与 `airymaxos-security/sandbox` 协作。

### 4.4 LLM 推理感知调度（基于 AirymaxOS 认知中枢）

**调度策略**：
- 推理阶段感知：识别 prefill 与 decode 阶段，分别调度。
- KV Cache 感知：调度时考虑 KV Cache 局部性。
- 动态 batching：根据负载动态调整 batch size。
- 投机解码：支持 speculative decoding 调度。

**与 sched_ext 协作**：
- LLM 推理任务标记为 `agent-inference` cgroup。
- sub-scheduler `scx_agent` 识别推理阶段动态调度。

### 4.5 GPU/NPU 调度与池化

**GPU 调度**：
- 时间片调度：多任务共享 GPU。
- 空间分区：MIG/MPS 支持多实例。
- 上下文切换：快速上下文切换。

**NPU 调度**：
- 厂商驱动集成（华为昇腾、英伟达、AMD）。
- 算力池化：跨设备调度。

**算力池化**：
- 跨节点 GPU/NPU 调度。
- 故障切换。
- 弹性扩缩容。

### 4.6 Thinkdual 双思考系统内核态加速

参见 4.2。

### 4.7 Token 能效优化（KVC-Gateway + LMCache + Bifrost）

**KVC-Gateway**：KV Cache 网关，跨请求复用 KV Cache。
**LMCache**：KV Cache 跨节点缓存，减少重复计算。
**Bifrost**：推测解码加速，减少 decode 阶段延迟。

**优化效果**：
- 共享 prompt 复用：减少 50%+ KV Cache 计算。
- 跨节点缓存：减少 30%+ 重复推理。
- 推测解码：减少 2-3x decode 延迟。

### 4.8 超节点沙箱（软硬协同优化镜像快照）

基于 **AirymaxOS 超节点 OS**：
- 镜像快照：基于 userfaultfd 与 CXL 实现快速快照。
- 快速克隆：COW 共享内存，秒级克隆。
- 跨节点迁移：基于 MemoryRovol 迁移。
- NUMA 感知调度：优先本地节点调度。

### 4.9 具身智能支持（基于 AirymaxOS Claw）

基于 **AirymaxOS Claw** 具身智能框架：
- 传感器数据汇聚：多模态传感器接入。
- 运动控制接口：标准化的运动控制 API。
- 实时控制循环：硬实时保证（与 sched_ext sub-scheduler 协作）。
- 多模态感知融合：视觉、听觉、触觉融合。
- 安全监控：紧急停止机制。

---

## 5. 微内核思想体现

### 5.1 Agent 认知作为独立服务

遵循微内核"机制在内核，策略在用户态"原则：
- 内核提供 CoreLoopThree kthread 机制（调度、加速）。
- 认知策略（思考模型、决策逻辑）在用户态（与 `airymaxos-services/daemons/llm_d` 协作）。

### 5.2 算力调度解耦

- 算力调度机制在内核（GPU/NPU 调度器）。
- 调度策略在用户态（与 `airymaxos-services/daemons/sched_d` 协作）。

### 5.3 安全沙箱隔离

- Wasm runtime 提供安全沙箱隔离。
- 与 `airymaxos-security/sandbox` 协作提供多层防护。

---

## 6. AirymaxOS 工程基线

- **AirymaxOS 认知中枢**：AI 原生调度框架基线。
- **AirymaxOS 超节点 OS**：超节点沙箱基线。
- **AirymaxOS 具身智能（Claw）**：具身智能运行时基线。
- **AirymaxOS AI 原生**：AI 原生 OS 设计哲学基线。

---

## 7. 前沿理论参考

| 理论 | 来源 | 应用 |
|------|------|------|
| AirymaxOS AI 原生 | AirymaxOS | AI 原生 OS 设计 |
| Wasm 3.0 | Bytecode Alliance | 安全沙箱 runtime |
| LLM 调度 | 学术研究 | LLM 推理感知调度 |
| Thinking, Fast and Slow | Daniel Kahneman | Thinkdual 双思考 |
| KVC-Gateway | 业界实践 | KV Cache 网关 |
| LMCache | 学术研究 | KV Cache 跨节点缓存 |
| Bifrost | 学术研究 | 推测解码加速 |
| Wasm Component Model | Bytecode Alliance | 组件互操作 |
| WASI Preview 3 | Bytecode Alliance | 系统接口 |

---

## 8. 与其他子仓的协作

| 协作子仓 | 协作内容 |
|---------|---------|
| `airymaxos-kernel` | 提供 CoreLoopThree kthread、Wasm runtime 内核支持 |
| `airymaxos-services` | 与 llm_d/sched_d 协作，调度策略在用户态 |
| `airymaxos-security` | 提供 Wasm 沙箱、LLM 推理 TEE 保护 |
| `airymaxos-memory` | 提供 MemoryRovol 快照、超节点沙箱迁移 |
| `airymaxos-cloudnative` | 提供 Agent 容器化、超节点 OS 集成 |
| `airymaxos-system` | 提供认知监控工具 |
| `airymaxos-tests` | 认知测试、性能基准 |

---

## 9. 里程碑

| 阶段 | 目标 | 时间 |
|------|------|------|
| M1 | CoreLoopThree kthread 实现 | 2026 Q3 |
| M2 | Wasm 3.0 runtime 集成 | 2026 Q4 |
| M3 | LLM 推理感知调度 | 2027 Q1 |
| M4 | Thinkdual 内核态加速 | 2027 Q2 |
| M5 | GPU/NPU 调度与池化 | 2027 Q3 |
| M6 | 超节点沙箱 + 具身智能 | 2027 Q4 |

---

## 10. 参考

- AirymaxOS 认知中枢文档
- AirymaxOS 超节点 OS 文档
- AirymaxOS Claw 具身智能文档
- Wasm 3.0 规范（Bytecode Alliance）
- WASI Preview 3 规范
- Wasm Component Model 规范
- Daniel Kahneman "Thinking, Fast and Slow"
- KVC-Gateway、LMCache、Bifrost 论文
- agentrt coreloopthree + frameworks 设计文档
