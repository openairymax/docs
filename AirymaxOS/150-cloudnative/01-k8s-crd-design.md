Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# K8s CRD 设计
> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）云原生体系核心子文档，定义 Agent 自定义资源（CRD）的声明式模型、controller 逻辑与调度到 agentrt-linux 节点的机制\
> **文档版本**：0.1.1\
> **最后更新**：2026-07-09\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **同源映射**：agentrt gateway + Linux 6.6 容器编排（IRON-9 v2 [IND] 完全独立层，云原生为 agentrt-linux 专属扩展）\
> **理论根基**：Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论\
> **SPDX-License-Identifier**：AGPL-3.0-or-later OR Apache-2.0\
> **IRON-9 v2 层次**：[IND] 完全独立层（K8s CRD 与 controller 为 agentrt-linux 云原生专属实现）

---

## 1. 设计目标与 CRD 模型

### 1.1 设计目标

agentrt-linux（AirymaxOS）将 Agent 应用建模为 Kubernetes 自定义资源（Custom Resource），实现声明式管理。CRD 设计达成以下工程目标：

1. **声明式 Agent 管理**：通过 YAML 声明 Agent 的镜像、Token 预算、记忆卷载、认知循环等属性
2. **OS 级调度联动**：CRD controller 与 agentrt-linux 节点的 AIRY_SCHED_AGENT 用户态调度器联动
3. **记忆卷载 CSI 集成**：Agent CRD 自动声明 MemoryRovol CSI 卷并挂载至容器
4. **Cupolas 安全策略联动**：CRD 安全字段映射至 Cupolas capability 令牌与 Landlock 规则
5. **水平弹性伸缩**：基于 Token 预算消耗与认知负载的 HPA 自动伸缩

### 1.2 CRD 资源模型

| 资源类型 | API 组 | 用途 |
|----------|--------|------|
| `Agent` | `agent.airymaxos.dev/v1` | Agent 应用实例 |
| `AgentWorkflow` | `agent.airymaxos.dev/v1` | 多 Agent DAG 编排 |
| `AgentBudget` | `agent.airymaxos.dev/v1` | Token 预算配额 |
| `MemoryRovolClaim` | `agent.airymaxos.dev/v1` | 记忆卷载声明 |

### 1.3 与传统 K8s 工作负载对比

| 维度 | Deployment/Pod | agentrt-linux Agent CRD |
|------|----------------|--------------------------|
| 调度目标 | 通用节点 | agentrt-linux 节点（标签选择） |
| 资源声明 | CPU/Memory | CPU/Memory + Token 预算 |
| 存储 | PVC | MemoryRovol CSI 卷 |
| 安全 | SecurityContext | Cupolas capability + Landlock |
| 调度类 | CFS（默认） | AIRY_SCHED_AGENT（用户态调度器策略） |
| 生命周期 | RestartPolicy | Agent 状态机（8 状态） |

---

## 2. Agent CRD 定义

### 2.1 CRD 完整 YAML 定义

```yaml
# cloudnative/manifests/crd-agent.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: agents.agent.airymaxos.dev
spec:
  group: agent.airymaxos.dev
  names:
    kind: Agent
    listKind: AgentList
    plural: agents
    singular: agent
    shortNames:
      - ag
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          required:
            - spec
          properties:
            spec:
              type: object
              required:
                - image
                - tokenBudget
              properties:
                image:
                  type: string
                  description: Agent 容器镜像地址
                  pattern: '^[a-zA-Z0-9./:_-]+$'
                imagePullPolicy:
                  type: string
                  enum: [Always, IfNotPresent, Never]
                  default: IfNotPresent

                tokenBudget:
                  type: object
                  required: [total]
                  properties:
                    total:
                      type: integer
                      minimum: 1
                      description: 总 Token 预算
                    perRequestLimit:
                      type: integer
                      minimum: 1
                      description: 单次请求上限
                    refillRate:
                      type: integer
                      minimum: 0
                      description: 补充速率（tokens/秒）
                    policy:
                      type: string
                      enum: [throttle, suspend, hardFail]
                      default: throttle
                      description: 超额策略

                memoryRovol:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                      default: false
                    layers:
                      type: array
                      items:
                        type: string
                        enum: [L1, L2, L3, L4]
                      minItems: 1
                      maxItems: 4
                    capacity:
                      type: string
                      default: "100Gi"
                    forgettingStrategy:
                      type: string
                      enum: [NONE, EBBINGHAUS, LINEAR, ACCESS_BASED]
                      default: EBBINGHAUS

                cognition:
                  type: object
                  properties:
                    cycle:
                      type: string
                      enum: [CoreLoopThree]
                      default: CoreLoopThree
                    scheduler:
                      type: string
                      enum: [AIRY_SCHED_AGENT, SCHED_NORMAL]
                      default: AIRY_SCHED_AGENT
                    thinkMode:
                      type: string
                      enum: [t2, t1-f, t1-p, dual]
                      default: dual

                security:
                  type: object
                  properties:
                    capabilities:
                      type: array
                      items:
                        type: string
                      description: Cupolas capability 列表
                    landlockPaths:
                      type: array
                      items:
                        type: string
                      description: Landlock 允许路径
                    seccompProfile:
                      type: string
                      default: agentrt-default

                resources:
                  type: object
                  properties:
                    cpu:
                      type: string
                      default: "2"
                    memory:
                      type: string
                      default: "4Gi"
                    pids:
                      type: integer
                      default: 100

                nodeSelector:
                  type: object
                  additionalProperties:
                    type: string
                  description: 节点选择器（默认选 agentrt-linux 节点）

            status:
              type: object
              properties:
                agentId:
                  type: integer
                  description: 内核分配的 Agent ID
                state:
                  type: string
                  enum: [Creating, Registered, Ready, Running,
                         Blocked, Suspended, Terminating, Dead]
                tokensUsed:
                  type: integer
                tokensRemaining:
                  type: integer
                conditions:
                  type: array
                  items:
                    type: object
                    properties:
                      type:
                        type: string
                      status:
                        type: string
                        enum: [True, False, Unknown]
                      lastTransitionTime:
                        type: string
                        format: date-time
```

### 2.2 Agent 实例 YAML 示例

```yaml
# cognition-agent-01.yaml
apiVersion: agent.airymaxos.dev/v1
kind: Agent
metadata:
  name: cognition-agent-01
  namespace: production
  labels:
    app: cognition
    tier: core
spec:
  image: registry.airymaxos.dev/cognition:v1.0.1
  imagePullPolicy: IfNotPresent

  tokenBudget:
    total: 1000000
    perRequestLimit: 10000
    refillRate: 100
    policy: throttle

  memoryRovol:
    enabled: true
    layers: [L1, L2, L3, L4]
    capacity: "100Gi"
    forgettingStrategy: EBBINGHAUS

  cognition:
    cycle: CoreLoopThree
    scheduler: AIRY_SCHED_AGENT
    thinkMode: dual

  security:
    capabilities:
      - AIRY_CAP_MEMORY_READ
      - AIRY_CAP_MEMORY_WRITE
      - AIRY_CAP_IPC_SEND
    landlockPaths:
      - /var/lib/agentrt/agent-01/data
      - /var/log/agentrt/agent-01
    seccompProfile: agentrt-default

  resources:
    cpu: "4"
    memory: "8Gi"
    pids: 200

  nodeSelector:
    airymaxos.dev/node-type: agentrt-linux
    airymaxos.dev/user-sched: "true"
```

---

## 3. Controller 设计

### 3.1 Controller 架构

```
┌─────────────────────────────────────────────────────┐
│ K8s API Server                                      │
│         |                                           │
│         v                                           │
│ ┌─────────────────────────────────────────────────┐ │
│ │ agentrt-controller-manager                     │ │
│ │  ┌───────────────────────────────────────────┐ │ │
│ │  │ AgentController                          │ │ │
│ │  │  - Informer (watch Agent CR)             │ │ │
│ │  │  - WorkQueue                             │ │ │
│ │  │  - Reconcile loop                       │ │ │
│ │  └───────────────────────────────────────────┘ │ │
│ │           |                                     │
│ │           v                                     │
│ │  ┌───────────────────────────────────────────┐ │
│ │  │ Node Allocator                            │ │ │
│ │  │  - 选择 agentrt-linux 节点                │ │ │
│ │  │  - 与 AIRY_SCHED_AGENT 用户态调度器联动     │ │ │
│ │  └───────────────────────────────────────────┘ │
│ │           |                                     │
│ │           v                                     │
│ │  ┌───────────────────────────────────────────┐ │
│ │  │ Pod Spec Builder                         │ │ │
│ │  │  - 生成 Agent Pod                         │ │ │
│ │  │  - 挂载 MemoryRovol CSI 卷                │ │ │
│ │  │  - 注入 Cupolas 安全策略                  │ │ │
│ │  └───────────────────────────────────────────┘ │
│ └─────────────────────────────────────────────────┘
│         |                                           │
└─────────|───────────────────────────────────────────┘
          v
   ┌────────────────────────────────────┐
   │ agentrt-linux Node                  │
   │  - kubelet + containerd             │
   │  - AIRY_SCHED_AGENT 用户态调度器    │
   │  - MemoryRovol CSI 插件             │
   │  - Cupolas 安全穹顶                 │
   └────────────────────────────────────┘
```

### 3.2 Controller 伪代码

```go
// cloudnative/controllers/agent_controller.go
package controllers

import (
	"context"
	"fmt"

	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"

	agentv1 "airymaxos.dev/cloudnative/api/v1"
)

// AgentReconciler 协调 Agent CRD
type AgentReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=agent.airymaxos.dev,resources=agents,verbs=*
// +kubebuilder:rbac:groups=agent.airymaxos.dev,resources=agents/status,verbs=*
// +kubebuilder:rbac:groups="",resources=pods,verbs=*

func (r *AgentReconciler) Reconcile(ctx context.Context,
	req ctrl.Request) (ctrl.Result, error) {
	log := ctrl.Log.WithValues("agent", req.NamespacedName)

	/* 1. 获取 Agent CR */
	var agent agentv1.Agent
	if err := r.Get(ctx, req.NamespacedName, &agent); err != nil {
		if errors.IsNotFound(err) {
			return ctrl.Result{}, nil
		}
		return ctrl.Result{}, err
	}

	/* 2. 状态机分发 */
	switch agent.Status.State {
	case "", "Creating":
		return r.handleCreate(ctx, &agent)
	case "Running":
		return r.handleRunning(ctx, &agent)
	case "Terminating":
		return r.handleTerminate(ctx, &agent)
	}

	return ctrl.Result{}, nil
}

/* handleCreate 处理创建事件 */
func (r *AgentReconciler) handleCreate(ctx context.Context,
	agent *agentv1.Agent) (ctrl.Result, error) {
	log := ctrl.Log.WithValues("agent", agent.Name)

	/* 校验镜像与节点选择器 */
	if err := validateAgentSpec(agent); err != nil {
		return ctrl.Result{}, err
	}

	/* 分配 agentrt-linux 节点 */
	nodeName, err := r.allocateNode(ctx, agent)
	if err != nil {
		log.Error(err, "failed to allocate node")
		return ctrl.Result{Requeue: true}, nil
	}

	/* 生成 Pod Spec（含 MemoryRovol 卷挂载） */
	pod := r.buildAgentPod(agent, nodeName)
	if err := r.Create(ctx, pod); err != nil {
		if errors.IsAlreadyExists(err) {
			return ctrl.Result{}, nil
		}
		return ctrl.Result{}, err
	}

	/* 更新 CR 状态为 Registered */
	agent.Status.State = "Registered"
	agent.Status.AgentId = allocateAgentID()
	if err := r.Status().Update(ctx, agent); err != nil {
		return ctrl.Result{}, err
	}

	return ctrl.Result{}, nil
}

/* buildAgentPod 构造 Agent Pod Spec */
func (r *AgentReconciler) buildAgentPod(agent *agentv1.Agent,
	nodeName string) *corev1.Pod {
	pod := &corev1.Pod{
		ObjectMeta: metav1.ObjectMeta{
			Name:      fmt.Sprintf("agent-%s", agent.Name),
			Namespace: agent.Namespace,
			Labels: map[string]string{
				"app":                      "agentrt-agent",
				"agent.airymaxos.dev/name": agent.Name,
			},
		},
		Spec: corev1.PodSpec{
			NodeName:  nodeName,
			RuntimeClassName: strPtr("airymaxos-agent"),
			Containers: []corev1.Container{{
				Name:  "agent",
				Image: agent.Spec.Image,
				Resources: corev1.ResourceRequirements{
					Limits: buildResourceList(agent.Spec.Resources),
				},
				Env: []corev1.EnvVar{
					{Name: "AIRY_AGENT_ID",
					 Value: fmt.Sprintf("%d", agent.Status.AgentId)},
					{Name: "AIRY_TOKEN_BUDGET",
					 Value: fmt.Sprintf("%d",
						agent.Spec.TokenBudget.Total)},
					{Name: "AIRY_SCHED_CLASS",
					 Value: agent.Spec.Cognition.Scheduler},
				},
				VolumeMounts: buildMemoryRovolMounts(agent),
				SecurityContext: buildSecurityContext(agent),
			}},
			Volumes:       buildMemoryRovolVolumes(agent),
			NodeSelector:  buildNodeSelector(agent),
		},
	}

	ctrl.SetControllerReference(agent, pod, r.Scheme)
	return pod
}

/* allocateNode 选择 agentrt-linux 节点 */
func (r *AgentReconciler) allocateNode(ctx context.Context,
	agent *agentv1.Agent) (string, error) {
	var nodes corev1.NodeList
	labelSelector := client.MatchingLabels{
		"airymaxos.dev/node-type": "agentrt-linux",
		"airymaxos.dev/user-sched": "true",
	}
	if err := r.List(ctx, &nodes, labelSelector); err != nil {
		return "", err
	}

	/* 选择 Token 预算余量充足的节点 */
	for _, node := range nodes.Items {
		if hasBudgetCapacity(&node, agent.Spec.TokenBudget.Total) {
			return node.Name, nil
		}
	}
	return "", fmt.Errorf("no eligible agentrt-linux node with budget")
}
```

---

## 4. 调度到 agentrt-linux 节点机制

### 4.1 节点标签体系

agentrt-linux 节点通过标签声明能力：

```yaml
# agentrt-linux 节点标签
airymaxos.dev/node-type: agentrt-linux       # 节点类型
airymaxos.dev/kernel-version: "6.6-airymax"   # 内核版本
airymaxos.dev/user-sched: "true"               # 支持方案 C-Prime 用户态调度器
airymaxos.dev/sched-agent: "true"             # 支持 AIRY_SCHED_AGENT 策略
airymaxos.dev/memoryrovol: "true"             # 支持 MemoryRovol
airymaxos.dev/cupolas: "true"                 # 部署 Cupolas
airymaxos.dev/cxl-pool: "true"                # CXL 内存池化
airymaxos.dev/token-budget-available: "5000000"
```

### 4.2 RuntimeClass 定义

Agent Pod 通过 RuntimeClass 指定使用 agentrt-linux 专属 OCI runtime：

```yaml
# runtimeclass-agent.yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: airymaxos-agent
handler: airymaxos-shim   # 指向 containerd shim v2
scheduling:
  nodeSelector:
    airymaxos.dev/node-type: agentrt-linux
    airymaxos.dev/sched-agent: "true"
  tolerations:
    - key: airymaxos.dev/agent-only
      operator: Equal
      value: "true"
      effect: NoSchedule
overhead:
  podFixed:
    cpu: "100m"      # AIRY_SCHED_AGENT 调度开销
    memory: "128Mi"  # MicroCoreRT 额外内存
```

### 4.3 AIRY_SCHED_AGENT 联动

Controller 通过节点的 daemonset 与 AIRY_SCHED_AGENT 用户态调度器联动：

```go
/* agentrt-sched-agent-agent: daemonset 运行于 agentrt-linux 节点
 * 监听 Pod 创建事件，向内核 AIRY_SCHED_AGENT 用户态调度器提交调度参数 */
func handlePodCreate(pod *corev1.Pod) error {
	if pod.Labels["app"] != "agentrt-agent" {
		return nil
	}

	agentID := pod.Annotations["agentrt.agent/id"]
	budget := pod.Annotations["agentrt.token/budget"]
	schedClass := pod.Annotations["agentrt.cognition/scheduler"]

	/* 通过 /proc/agentrt/user_sched 注册调度参数 */
	return writeProcFile("/proc/agentrt/user_sched/register",
		fmt.Sprintf("%s %s %s", agentID, budget, schedClass))
}
```

---

## 5. Token 预算配额管理

### 5.1 AgentBudget CRD

Token 预算通过 `AgentBudget` CRD 进行命名空间级配额管理：

```yaml
apiVersion: agent.airymaxos.dev/v1
kind: AgentBudget
metadata:
  name: production-budget
  namespace: production
spec:
  totalBudget: 100000000        # 命名空间总预算
  perAgentLimit: 10000000        # 单 Agent 上限
  refillInterval: 3600          # 补充间隔（秒）
  hardCap: true                  # 硬上限（超额拒绝创建）
```

### 5.2 预算校验 webhook

ValidatingAdmissionWebhook 在 Agent 创建时校验预算：

```go
/* validateBudget 校验命名空间 Token 预算 */
func (v *AgentValidator) validateBudget(ctx context.Context,
	agent *agentv1.Agent) error {
	var budget agentv1.AgentBudget
	err := v.Client.Get(ctx, types.NamespacedName{
		Name:      "production-budget",
		Namespace: agent.Namespace,
	}, &budget)
	if err != nil {
		return err
	}

	if agent.Spec.TokenBudget.Total > budget.Spec.PerAgentLimit {
		return fmt.Errorf("token budget %d exceeds per-agent limit %d",
			agent.Spec.TokenBudget.Total,
			budget.Spec.PerAgentLimit)
	}

	/* 校验命名空间总额度 */
	used, err := v.getNamespaceBudgetUsed(ctx, agent.Namespace)
	if err != nil {
		return err
	}

	if used+agent.Spec.TokenBudget.Total > budget.Spec.TotalBudget {
		return fmt.Errorf("namespace budget exhausted")
	}

	return nil
}
```

---

## 6. 状态同步与可观测性

### 6.1 状态同步

Controller 周期性从 agentrt-linux 节点同步 Agent 状态：

```go
/* syncAgentStatus 周期性同步状态 */
func (r *AgentReconciler) syncAgentStatus(ctx context.Context,
	agent *agentv1.Agent) error {
	/* 通过 /proc/agentrt/agent/<id>/status 读取内核状态 */
	status, err := readAgentStatusFromNode(agent.Status.AgentID,
		agent.Spec.NodeName)
	if err != nil {
		return err
	}

	if status.State != agent.Status.State {
		agent.Status.State = status.State
		agent.Status.TokensUsed = status.TokensUsed
		agent.Status.TokensRemaining = status.TokensRemaining
		return r.Status().Update(ctx, agent)
	}
	return nil
}
```

### 6.2 Prometheus 指标

Controller 暴露 Prometheus 指标供监控：

```go
var (
	agentCreated = promauto.NewCounterVec(prometheus.CounterOpts{
		Name: "airy_agent_created_total",
		Help: "Total agents created",
	}, []string{"namespace"})

	agentTokenUsed = promauto.NewGaugeVec(prometheus.GaugeOpts{
		Name: "airy_agent_token_used",
		Help: "Tokens used by agent",
	}, []string{"namespace", "agent"})
)
```

---

## 7. 弹性伸缩

### 7.1 HPA 配置

基于 Token 预算消耗的 HPA 自动伸缩：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cognition-agent-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: agent.airymaxos.dev/v1
    kind: Agent
    name: cognition-agent
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: External
      external:
        metric:
          name: airy_agent_token_utilization
          selector:
            matchLabels:
              agent: cognition-agent
        target:
          type: AverageValue
          averageValue: "80"   # 平均利用率 80% 触发扩容
```

---

## 8. 测试与验证

### 8.1 Controller 测试矩阵

| 测试用例 | 验证目标 | 预期结果 |
|----------|----------|----------|
| 创建 Agent CR | Pod 创建 + 状态 Registered | CR 与 Pod 均创建 |
| 节点无预算 | allocateNode 失败 | Requeue 等待 |
| 销毁 Agent CR | Pod 销毁 + 状态 Dead | 资源全部回收 |
| 状态同步 | 内核状态同步至 CR | status 字段更新 |
| 预算超额 | webhook 拒绝 | 创建被拒 |

---

## 9. 相关文档

- `150-cloudnative/README.md`（云原生主索引）
- `150-cloudnative/02-containerd-shim.md`（containerd shim 设计）
- `150-cloudnative/03-cni-network-policy.md`（CNI 网络策略）
- `140-application-development/01-agent-lifecycle.md`（Agent 生命周期）
- `90-observability/README.md`（可观测性）

---

## 10. 参考材料

- Kubernetes CustomResourceDefinition 文档
- controller-runtime 项目
- Kubebuilder 框架
- Kubernetes RuntimeClass 文档
- Prometheus Operator
- agentrt gateway（IRON-9 v2 [IND] 完全独立层）

---

> **文档结束** | agentrt-linux（AirymaxOS）K8s CRD 设计 v0.1.1
> 遵循 IRON-9 v2 [IND] 完全独立层（agentrt-linux 云原生专属）
