Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# containerd shim 设计

> **文档定位**: agentrt-liunx（AirymaxOS，极境智能体操作系统）云原生体系核心子文档，定义 agentrt-liunx 专属 OCI runtime shim 的设计，将 Agent 作为容器 workload 运行
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-09
> **理论根基**: Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0
> **同源映射**: Linux 6.6 容器运行时（IRON-9 v2 [IND] 完全独立层，shim 为 agentrt-liunx 专属实现）
> **IRON-9 v2 层次**: [IND] 完全独立层（containerd shim 为 agentrt-liunx 云原生专属）

---

## 1. 设计目标与 shim 架构

### 1.1 设计目标

agentrt-liunx（AirymaxOS）提供专属 containerd shim v2 实现，作为 Agent 容器与 OS 内核之间的桥梁。shim 设计达成以下工程目标：

1. **Agent workload 感知**：shim 在容器创建时向内核注册 Agent，签订 Token 预算契约
2. **SCHED_AGENT 集成**：shim 将容器内主进程的调度类切换为 SCHED_AGENT
3. **MemoryRovol 卷挂载**：shim 在容器启动前挂载 MemoryRovol CSI 卷至指定路径
4. **Cupolas 安全注入**：shim 在容器创建时注入 capability 令牌与 Landlock 规则
5. **生命周期联动**：shim 将容器生命周期事件（start/stop/exit）上报至内核 Agent 状态机

### 1.2 shim 在容器栈中的位置

```
┌──────────────────────────────────────────────────┐
│ kubelet                                          │
│     |                                            │
│     v                                            │
│ CRI (Container Runtime Interface)               │
│     |                                            │
│     v                                            │
│ containerd                                      │
│     |                                            │
│     v (通过 RuntimeClass 选择 handler)            │
│ ┌──────────────────────────────────────────────┐ │
│ │ airymaxos-shim v2（agentrt-liunx 专属）      │ │
│ │   - 注册 Agent 至内核                        │ │
│ │   - 挂载 MemoryRovol 卷                      │ │
│ │   - 注入 Cupolas 安全策略                     │ │
│ │   - 切换 SCHED_AGENT 调度类                  │ │
│ └──────────────────────────────────────────────┘ │
│     |                                            │
│     v                                            │
│ runc (OCI runtime)                              │
│     |                                            │
│     v                                            │
│ Linux 6.6 内核（agentrt-liunx）                  │
│   - SCHED_AGENT 调度类                          │
│   - MemoryRovol CSI                             │
│   - Cupolas 安全穹顶                            │
└──────────────────────────────────────────────────┘
```

### 1.3 与标准 runc shim 对比

| 维度 | io.containerd.runc.v2 | io.containerd.airymaxos.v2 |
|------|----------------------|----------------------------|
| Agent 注册 | 无 | 容器创建时注册至内核 |
| 调度类 | CFS（默认） | SCHED_AGENT |
| 存储卷 | 标准 PVC | MemoryRovol CSI |
| 安全 | 标准 SecurityContext | Cupolas + Landlock |
| 生命周期 | 标准 | 联动 Agent 状态机 |
| Token 预算 | 无 | 自动签订契约 |

---

## 2. shim v2 接口实现

### 2.1 shim v2 接口概述

containerd shim v2 基于 ttrpc，定义以下核心接口：

```go
// containerd shim v2 接口（基于 ttrpc）
type Shim interface {
	Start(ctx, req *StartRequest) (*StartResponse, error)
	Wait(ctx, req *WaitRequest) (*WaitResponse, error)
	Create(ctx, req *CreateTaskRequest) (*CreateTaskResponse, error)
	Delete(ctx, req *DeleteRequest) (*DeleteResponse, error)
	Exec(ctx, req *ExecProcessRequest) (*ExecProcessResponse, error)
	Pause(ctx, req *PauseRequest) (*PauseResponse, error)
	Resume(ctx, req *ResumeRequest) (*ResumeResponse, error)
	Kill(ctx, req *KillRequest) (*KillResponse, error)
	Pids(ctx, req *PidsRequest) (*PidsResponse, error)
	CloseIO(ctx, req *CloseIORequest) (*Empty, error)
	Checkpoint(ctx, req *CheckpointTaskRequest) (*Empty, error)
	ShimInfo(ctx, req *ShimInfoRequest) (*ShimInfoResponse, error)
	Update(ctx, req *UpdateTaskRequest) (*Empty, error)
	Cleanup(ctx, req *CleanupRequest) (*CleanupResponse, error)
}
```

### 2.2 airymaxos-shim 完整伪代码

```go
// airymaxos-cloudnative/containerd-shim/shim.go
package airymaxos

import (
	"context"
	"fmt"
	"os"
	"os/exec"
	"syscall"

	"github.com/containerd/containerd/runtime/v2/task"
	"github.com/containerd/ttrpc"
)

type AirymaxosShim struct {
	mu          sync.Mutex
	containers  map[string]*AgentContainer
	shimAddress string
}

type AgentContainer struct {
	ID          string
	AgentID     uint32          /* 内核分配的 Agent ID */
	Bundle      string          /* OCI bundle 路径 */
	Pid         int             /* runc 启动的容器主进程 PID */
	TokenBudget uint64          /* Token 预算 */
	MemoryRovol *RovolMount     /* 记忆卷载挂载点 */
	Capability  uint64          /* Cupolas capability 令牌 */
	State       string          /* Agent 状态 */
}

/* Create 创建 Agent 容器 */
func (s *AirymaxosShim) Create(ctx context.Context,
	req *task.CreateTaskRequest) (*task.CreateTaskResponse, error) {
	s.mu.Lock()
	defer s.mu.Unlock()

	if _, exists := s.containers[req.ID]; exists {
		return nil, fmt.Errorf("container %s already exists", req.ID)
	}

	/* 1. 解析 Agent 容器镜像标签（agent.airymaxos.* 前缀） */
	agentMeta, err := parseAgentAnnotations(req.Annotations)
	if err != nil {
		return nil, err
	}

	/* 2. 向内核注册 Agent，获取 agent_id（syscall） */
	agentID, err := registerAgentToKernel(agentMeta)
	if err != nil {
		return nil, fmt.Errorf("register agent failed: %w", err)
	}

	/* 3. 签订 Token 预算契约（syscall） */
	if err := signTokenBudget(agentID, agentMeta.TokenBudget); err != nil {
		cleanupAgentRegistration(agentID)
		return nil, fmt.Errorf("sign budget failed: %w", err)
	}

	/* 4. 挂载 MemoryRovol CSI 卷（如声明） */
	var rovolMount *RovolMount
	if agentMeta.MemoryRovol.Enabled {
		rovolMount, err = mountMemoryRovol(agentID, agentMeta.MemoryRovol)
		if err != nil {
			cleanupAgentBudget(agentID)
			cleanupAgentRegistration(agentID)
			return nil, fmt.Errorf("mount memoryrovol failed: %w", err)
		}
	}

	/* 5. 注入 Cupolas 安全策略（capability + Landlock） */
	capToken, err := injectCupolasSecurity(agentID, agentMeta.Security)
	if err != nil {
		cleanupRovolMount(rovolMount)
		cleanupAgentBudget(agentID)
		cleanupAgentRegistration(agentID)
		return nil, fmt.Errorf("inject cupolas failed: %w", err)
	}

	/* 6. 调用 runc 创建容器（标准 OCI 流程） */
	runcResp, err := s.callRuncCreate(ctx, req)
	if err != nil {
		cleanupCupolas(capToken)
		cleanupRovolMount(rovolMount)
		cleanupAgentBudget(agentID)
		cleanupAgentRegistration(agentID)
		return nil, fmt.Errorf("runc create failed: %w", err)
	}

	/* 7. 记录容器信息 */
	s.containers[req.ID] = &AgentContainer{
		ID:          req.ID,
		AgentID:     agentID,
		Bundle:      req.Bundle,
		Pid:         runcResp.Pid,
		TokenBudget: agentMeta.TokenBudget.Total,
		MemoryRovol: rovolMount,
		Capability:  capToken,
		State:       "Registered",
	}

	return &task.CreateTaskResponse{
		Pid: runcResp.Pid,
	}, nil
}

/* Start 启动 Agent 容器并切换 SCHED_AGENT 调度类 */
func (s *AirymaxosShim) Start(ctx context.Context,
	req *task.StartRequest) (*task.StartResponse, error) {
	s.mu.Lock()
	container, exists := s.containers[req.ID]
	s.mu.Unlock()
	if !exists {
		return nil, fmt.Errorf("container not found")
	}

	/* 1. 调用 runc start 启动容器主进程 */
	if err := s.callRuncStart(ctx, req.ID); err != nil {
		return nil, fmt.Errorf("runc start failed: %w", err)
	}

	/* 2. 切换容器主进程调度类为 SCHED_AGENT */
	if err := setSchedAgent(container.Pid); err != nil {
		return nil, fmt.Errorf("set SCHED_AGENT failed: %w", err)
	}

	/* 3. 更新内核 Agent 状态为 Ready */
	if err := updateAgentState(container.AgentID, "Ready"); err != nil {
		return nil, fmt.Errorf("update state failed: %w", err)
	}

	container.State = "Running"
	return &task.StartResponse{
		Pid: uint32(container.Pid),
	}, nil
}

/* Delete 销毁 Agent 容器并清理内核资源 */
func (s *AirymaxosShim) Delete(ctx context.Context,
	req *task.DeleteRequest) (*task.DeleteResponse, error) {
	s.mu.Lock()
	container, exists := s.containers[req.ID]
	if !exists {
		s.mu.Unlock()
		return nil, fmt.Errorf("container not found")
	}
	delete(s.containers, req.ID)
	s.mu.Unlock()

	/* 1. 触发 Agent 优雅销毁（内核状态机 TERMINATING） */
	updateAgentState(container.AgentID, "Terminating")

	/* 2. 调用 runc delete 清理容器 */
	exitStatus, err := s.callRuncDelete(ctx, req.ID)

	/* 3. 撤销 Cupolas capability */
	cleanupCupolas(container.Capability)

	/* 4. 卸载 MemoryRovol 卷（先 flush 持久化） */
	if container.MemoryRovol != nil {
		flushAndUnmountRovol(container.MemoryRovol)
	}

	/* 5. 撤销 Token 预算契约 */
	cleanupAgentBudget(container.AgentID)

	/* 6. 从内核注销 Agent */
	cleanupAgentRegistration(container.AgentID)

	return &task.DeleteResponse{
		ExitStatus: exitStatus,
		ExitedAt:   time.Now(),
		Pid:        uint32(container.Pid),
	}, err
}

/* Kill 终止 Agent 容器 */
func (s *AirymaxosShim) Kill(ctx context.Context,
	req *task.KillRequest) (*task.KillResponse, error) {
	s.mu.Lock()
	container, exists := s.containers[req.ID]
	s.mu.Unlock()
	if !exists {
		return nil, fmt.Errorf("container not found")
	}

	/* 先通过内核通知 Agent 即将被终止（刷新记忆） */
	notifyAgentTermination(container.AgentID)

	/* 调用 runc kill */
	if err := s.callRuncKill(ctx, req.ID, req.Signal); err != nil {
		return nil, err
	}

	/* 更新 Agent 状态为 Suspended（等待销毁） */
	updateAgentState(container.AgentID, "Suspended")
	container.State = "Suspended"
	return &task.KillResponse{}, nil
}
```

---

## 3. shim 启动入口

### 3.1 shim main 函数

shim 作为独立二进制运行，由 containerd 在容器创建时 fork：

```go
// airymaxos-cloudnative/containerd-shim/main.go
package main

import (
	"context"
	"os"

	"github.com/containerd/containerd/log"
	"github.com/containerd/containerd/runtime/v2"
	"github.com/containerd/containerd/runtime/v2/task"
)

func main() {
	/* containerd 通过 ttrpc 与 shim 通信
	 * shim 监听 socket，containerd 通过该 socket 调用 shim 接口 */
	ctx := context.Background()

	if err := runShim(ctx); err != nil {
		log.G(ctx).WithError(err).Fatal("shim exited")
		os.Exit(1)
	}
}

func runShim(ctx context.Context) error {
	/* 注册 shim service */
	service := &AirymaxosShim{
		containers: make(map[string]*AgentContainer),
	}

	/* 通过 containerd shim v2 框架启动 */
	return v2.Run(ctx, "airymaxos", service, func(s *ttrpc.Server) {
		task.RegisterTaskService(s, service)
	})
}
```

### 3.2 containerd 配置

agentrt-liunx 节点的 containerd 配置注册 airymaxos runtime：

```toml
# /etc/containerd/config.toml
version = 2

[plugins."io.containerd.grpc.v1.cri"]
  [plugins."io.containerd.grpc.v1.cri.containerd"]
    default_runtime_name = "runc"
    [plugins."io.containerd.grpc.v1.cri.containerd.runtimes.runc"]
      runtime_type = "io.containerd.runc.v2"

    # agentrt-liunx 专属 Agent 运行时
    [plugins."io.containerd.grpc.v1.cri.containerd.runtimes.airymaxos"]
      runtime_type = "io.containerd.airymaxos.v2"
      runtime_path = "/usr/local/bin/containerd-shim-airymaxos-v2"
      [plugins."io.containerd.grpc.v1.cri.containerd.runtimes.airymaxos".options]
        # Token 预算默认值
        default_token_budget = "1000000"
        # MemoryRovol 默认容量
        default_memoryrovol_capacity = "10Gi"
```

---

## 4. 内核交互

### 4.1 syscall 注册 Agent

shim 通过 syscall 向内核注册 Agent：

```go
/* registerAgentToKernel 通过 syscall 注册 Agent 至内核 */
func registerAgentToKernel(meta *AgentMeta) (uint32, error) {
	/* 构造 agentrt_agent_config 结构（与 [SC] 层头文件一致） */
	config := agentrtAgentConfig{
		Budget: agentrtTokenBudgetContract{
			TotalBudget:    meta.TokenBudget.Total,
			PerRequestLimit: meta.TokenBudget.PerRequest,
			RefillRate:     meta.TokenBudget.RefillRate,
			Policy:         meta.TokenBudget.Policy,
		},
		Isolation: agentrtTenantIsolation{
			CpuMax:     meta.Resources.CPU,
			MemoryMax:  meta.Resources.Memory,
			PidsMax:    meta.Resources.PIDs,
		},
	}

	/* syscall(AGENTRT_SYS_AGENT_CREATE) */
	ret, _, errno := syscall.Syscall(
		SYS_AGENTRT_AGENT_CREATE,
		uintptr(unsafe.Pointer(&config)),
		uintptr(meta.TraceID), 0)

	if errno != 0 {
		return 0, fmt.Errorf("syscall failed: %w", errno)
	}
	return uint32(ret), nil
}
```

### 4.2 SCHED_AGENT 切换

shim 在容器启动后切换主进程调度类：

```go
/* setSchedAgent 将 PID 切换至 SCHED_AGENT 调度类 */
func setSchedAgent(pid int) error {
	/* 通过 /proc/<pid>/sched_agent 写入调度类 */
	path := fmt.Sprintf("/proc/%d/sched_agent", pid)
	f, err := os.OpenFile(path, os.O_WRONLY, 0)
	if err != nil {
		return err
	}
	defer f.Close()

	_, err = f.WriteString("SCHED_AGENT")
	return err
}
```

### 4.3 MemoryRovol 挂载

shim 在容器创建前挂载 MemoryRovol CSI 卷：

```go
/* mountMemoryRovol 挂载 MemoryRovol 卷至容器 bundle */
func mountMemoryRovol(agentID uint32,
	config *MemoryRovolConfig) (*RovolMount, error) {
	hostPath := fmt.Sprintf("/var/lib/agentrt/memoryrovol/agent-%d",
		agentID)
	if err := os.MkdirAll(hostPath, 0755); err != nil {
		return nil, err
	}

	/* 通过 syscall 挂载 MemoryRovol 卷 */
	mountSpec := agentrtMemoryRovolMount{
		AgentID:  agentID,
		Layers:   config.Layers,
		Capacity: config.Capacity,
	}
	ret, _, errno := syscall.Syscall(
		SYS_AGENTRT_MEMORYROVOL_MOUNT,
		uintptr(unsafe.Pointer(&mountSpec)),
		uintptr(unsafe.Pointer(&hostPath[0])), 0)
	if errno != 0 {
		return nil, fmt.Errorf("mount failed: %w", errno)
	}

	return &RovolMount{
		HostPath:  hostPath,
		Handle:    ret,
		Layers:    config.Layers,
	}, nil
}
```

---

## 5. Cupolas 安全注入

### 5.1 capability 令牌授予

shim 在容器创建时向 Cupolas 申请 capability 令牌：

```go
/* injectCupolasSecurity 向 Agent 注入 Cupolas 安全策略 */
func injectCupolasSecurity(agentID uint32,
	config *SecurityConfig) (uint64, error) {
	/* 构造 capability 位图 */
	capBitmap := uint64(0)
	for _, cap := range config.Capabilities {
		bit, ok := capabilityMap[cap]
		if !ok {
			return 0, fmt.Errorf("unknown capability: %s", cap)
		}
		capBitmap |= (1 << bit)
	}

	/* 通过 Cupolas syscall 申请 capability 令牌 */
	capToken, err := cupolasGrantCapability(agentID, capBitmap)
	if err != nil {
		return 0, fmt.Errorf("grant capability failed: %w", err)
	}

	/* 注入 Landlock 规则 */
	for _, path := range config.LandlockPaths {
		if err := landlockApplyRule(agentID, path); err != nil {
			cupolasRevokeCapability(capToken)
			return 0, fmt.Errorf("landlock apply failed: %w", err)
		}
	}

	return capToken, nil
}
```

### 5.2 capability 映射表

```go
/* capabilityMap: capability 名称到位的映射 */
var capabilityMap = map[string]uint8{
	"AGENTRT_CAP_MEMORY_READ":   0,
	"AGENTRT_CAP_MEMORY_WRITE":  1,
	"AGENTRT_CAP_IPC_SEND":      2,
	"AGENTRT_CAP_IPC_RECV":      3,
	"AGENTRT_CAP_SCHED_SUBMIT":  4,
	"AGENTRT_CAP_NET_ACCESS":    5,
	"AGENTRT_CAP_FILE_READ":     6,
	"AGENTRT_CAP_FILE_WRITE":    7,
	"AGENTRT_CAP_TOOL_EXEC":     8,
	"AGENTRT_CAP_ADMIN":         63,
}
```

---

## 6. 容器镜像标签规范

### 6.1 Agent 镜像标签

Agent 容器镜像通过 OCI 标签声明 agentrt-liunx 专属属性：

| 标签 | 用途 | 示例 |
|------|------|------|
| `airymaxos.agent.token-budget` | 默认 Token 预算 | `"1000000"` |
| `airymaxos.agent.memory-rovol` | 默认记忆卷载层 | `"L1,L2,L3,L4"` |
| `airymaxos.agent.scheduler` | 调度类 | `"SCHED_AGENT"` |
| `airymaxos.agent.cognition-cycle` | 认知循环类型 | `"CoreLoopThree"` |
| `airymaxos.agent.capabilities` | 默认 capability | `"MEMORY_READ,MEMORY_WRITE"` |

### 6.2 shim 解析标签

```go
/* parseAgentAnnotations 从容器 annotations 解析 Agent 元数据 */
func parseAgentAnnotations(annotations map[string]string) (*AgentMeta, error) {
	meta := &AgentMeta{
		TokenBudget: &TokenBudgetConfig{Policy: "throttle"},
		MemoryRovol: &MemoryRovolConfig{Enabled: false},
		Security:    &SecurityConfig{},
	}

	if budget, ok := annotations["airymaxos.agent.token-budget"]; ok {
		meta.TokenBudget.Total = parseUint64(budget)
	}
	if rovol, ok := annotations["airymaxos.agent.memory-rovol"]; ok {
		if rovol != "disabled" {
			meta.MemoryRovol.Enabled = true
			meta.MemoryRovol.Layers = strings.Split(rovol, ",")
		}
	}
	if sched, ok := annotations["airymaxos.agent.scheduler"]; ok {
		meta.Scheduler = sched
	}
	if caps, ok := annotations["airymaxos.agent.capabilities"]; ok {
		meta.Security.Capabilities = strings.Split(caps, ",")
	}

	/* 校验必填字段 */
	if meta.TokenBudget.Total == 0 {
		meta.TokenBudget.Total = 1000000 /* 默认值 */
	}
	return meta, nil
}
```

---

## 7. 错误处理与资源清理

### 7.1 集中错误清理

shim 创建过程任一步骤失败，按反序清理已分配资源：

```go
/* 清理函数按创建的逆序调用 */
func cleanupCupolas(token uint64) {
	cupolasRevokeCapability(token)
}

func cleanupRovolMount(m *RovolMount) {
	if m == nil {
		return
	}
	flushAndUnmountRovol(m)
}

func cleanupAgentBudget(agentID uint32) {
	syscall.Syscall(SYS_AGENTRT_BUDGET_REVOKE, uintptr(agentID), 0, 0)
}

func cleanupAgentRegistration(agentID uint32) {
	syscall.Syscall(SYS_AGENTRT_AGENT_DESTROY, uintptr(agentID), 0, 0)
}
```

### 7.2 状态一致性保证

shim 维护本地容器状态与内核 Agent 状态的一致性：

| 容器状态 | 内核 Agent 状态 | 触发动作 |
|----------|------------------|----------|
| created | Registered | 注册完成 |
| running | Running | SCHED_AGENT 切换 |
| paused | Suspended | 预算暂停 |
| stopped | Terminating | 优雅销毁 |
| deleted | Dead | 资源回收 |

---

## 8. 性能优化

### 8.1 io_uring 批量提交

shim 与内核的通信通过 io_uring 批量提交，减少 syscall 次数：

```go
/* batchSubmitOps 批量提交多个 syscall 至 io_uring */
func batchSubmitOps(ops []AgentrtOp) error {
	sqe := ioUringGetSQE(ring)
	for _, op := range ops {
		fillSQE(sqe, op)
		sqe = ioUringGetSQE(ring)
	}
	ioUringSubmit(ring)
	/* 等待所有 CQE 完成 */
	return ioUringWaitCQE(ring, len(ops))
}
```

### 8.2 共享内存减少拷贝

shim 与内核共享 MemoryRovol 页面，避免数据拷贝：

| 操作 | 标准实现 | airymaxos-shim |
|------|----------|----------------|
| 容器日志 | pipe 拷贝 | 共享内存环形缓冲 |
| 记忆卷载 | 多次 read/write | mmap 共享页面 |
| IPC 消息 | socket 拷贝 | io_uring 零拷贝 |

---

## 9. 测试与验证

### 9.1 shim 测试矩阵

| 测试用例 | 验证目标 | 预期结果 |
|----------|----------|----------|
| 创建 Agent 容器 | 注册 + 契约签订 | Agent ID 分配成功 |
| 启动 Agent 容器 | SCHED_AGENT 切换 | 调度类切换成功 |
| 销毁 Agent 容器 | 资源全部回收 | 内核 Agent 状态 Dead |
| 创建失败回滚 | 集中清理 | 无资源泄漏 |
| MemoryRovol 挂载 | 卷挂载 + 持久化 | 容器可读写记忆 |
| Cupolas 注入 | capability 授予 | 权限生效 |

### 9.2 端到端测试

```bash
# 构建并部署 airymaxos-shim
make build-shim
sudo cp bin/containerd-shim-airymaxos-v2 /usr/local/bin/

# 启动 Agent 容器
sudo crictl runp agent-pod.yaml
sudo crictl create <pod-id> agent-container.yaml agent-pod.yaml

# 验证内核 Agent 状态
cat /proc/agentrt/agent/<id>/status
```

---

## 10. 相关文档

- `150-cloudnative/README.md`（云原生主索引）
- `150-cloudnative/01-k8s-crd-design.md`（K8s CRD 设计）
- `150-cloudnative/03-cni-network-policy.md`（CNI 网络策略）
- `140-application-development/01-agent-lifecycle.md`（Agent 生命周期）
- `110-security/README.md`（Cupolas 安全穹顶）

---

## 11. 参考材料

- containerd shim v2 设计文档
- OCI Runtime Specification
- runc 项目
- Kubernetes RuntimeClass
- Linux 6.6 `Documentation/admin-guide/cgroup-v2.rst`
- io_uring 用户态接口

---

> **文档结束** | agentrt-liunx（AirymaxOS）containerd shim 设计 v0.1.1
> 遵循 IRON-9 v2 [IND] 完全独立层（agentrt-liunx 云原生专属）
