Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# CNI 网络策略设计
> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）云原生体系核心子文档，定义 Agent 间网络隔离、Cupolas 安全策略与 CNI NetworkPolicy 的联动机制\
> **文档版本**：0.1.1\
> **最后更新**：2026-07-09\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **同源映射**：Linux 6.6 容器网络（IRON-9 v2 [IND] 完全独立层，CNI 为 agentrt-linux 专属扩展）\
> **理论根基**：Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论\
> **SPDX-License-Identifier**：AGPL-3.0-or-later OR Apache-2.0\
> **IRON-9 v2 层次**：[SS] Cupolas 安全模型同源 + [IND] CNI 实现独立

---

## 1. 设计目标与网络模型

### 1.1 设计目标

agentrt-linux（AirymaxOS）在云原生场景下需要为 Agent 提供细粒度的网络隔离与安全策略。CNI 网络策略设计达成以下工程目标：

1. **Agent 间网络隔离**：默认禁止 Agent 间网络通信，仅显式声明的允许通信
2. **Cupolas 策略联动**：CNI NetworkPolicy 与 Cupolas capability 令牌联动，双重过滤
3. **AgentsIPC 优先**：Agent 间通信优先走 AgentsIPC（io_uring 零拷贝），网络作为补充
4. **零信任模型**：所有 Agent 间流量默认拒绝，按需放行
5. **可观测审计**：所有网络策略决策记录至 Cupolas 审计日志

### 1.2 网络分层模型

| 层级 | 类型 | 机制 | 范围 |
|------|------|------|------|
| L1 | 容器网络 | CNI + Calico/Cilium | Pod 间网络 |
| L2 | 网络策略 | Kubernetes NetworkPolicy | 流量过滤 |
| L3 | 微核心网络 | AgentsIPC（io_uring） | Agent 间高性能通信 |
| **L4** | **Cupolas 联动** | **capability + LSM 钩子** | **agentrt-linux 专属** |

### 1.3 通信路径决策

```
Agent A -> Agent B 通信请求
    |
    v
[Cupolas capability 检查]
    |（持有 IPC_SEND cap?）
    v
[决策：是否走 AgentsIPC?]
    |
    +-- 是 --> AgentsIPC（io_uring 零拷贝，性能最高）
    |
    +-- 否 --> CNI 网络（NetworkPolicy 过滤）
                    |
                    v
              [Calico/Cilium 策略检查]
                    |
                    v
              [NetworkPolicy 允许?] --> 放行
```

---

## 2. NetworkPolicy 设计

### 2.1 默认拒绝策略

agentrt-linux 命名空间默认部署「默认拒绝所有」策略：

```yaml
# default-deny-all.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: agentrt-production
spec:
  podSelector: {}    # 选择命名空间内所有 Pod
  policyTypes:
    - Ingress
    - Egress
  # 无 ingress/egress 规则 = 默认拒绝所有
```

### 2.2 Agent 间显式放行策略

仅显式声明的 Agent 间通信被放行：

```yaml
# allow-cognition-to-llm.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-cognition-to-llm
  namespace: agentrt-production
spec:
  podSelector:
    matchLabels:
      agent.airymaxos.dev/name: llm-agent
  policyTypes:
    - Ingress
  ingress:
    - from:
        # 仅允许 cognition-agent 访问 llm-agent
        - podSelector:
            matchLabels:
              agent.airymaxos.dev/name: cognition-agent
      ports:
        # 仅允许 AgentsIPC 端口
        - protocol: TCP
          port: 7766    # AgentsIPC 标准端口
        - protocol: TCP
          port: 7767    # AgentsIPC 流式端口
```

### 2.3 出站白名单策略

Agent 出站流量需显式声明，仅允许访问必要的 LLM Provider 与外部服务：

```yaml
# allow-egress-llm-provider.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-llm-provider
  namespace: agentrt-production
spec:
  podSelector:
    matchLabels:
      agent.airymaxos.dev/name: llm-agent
  policyTypes:
    - Egress
  egress:
    # 允许 DNS
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
    # 允许访问 LLM Provider API
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            exceptions:
              - 10.0.0.0/8       # 禁止访问内网
              - 169.254.0.0/16   # 禁止访问元数据服务
      ports:
        - protocol: TCP
          port: 443    # 仅 HTTPS
    # 允许访问同命名空间 AgentsIPC
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: agentrt-production
      ports:
        - protocol: TCP
          port: 7766
```

### 2.4 Cupolas 联动策略

agentrt-linux 专属 CRD，将 NetworkPolicy 与 Cupolas capability 联动：

```yaml
# cloudnative/manifests/cupolas-network-policy.yaml
apiVersion: agent.airymaxos.dev/v1
kind: AgentNetworkPolicy
metadata:
  name: cupolas-cognition-policy
  namespace: agentrt-production
spec:
  agentSelector:
    matchLabels:
      agent.airymaxos.dev/name: cognition-agent
  # Cupolas capability 要求
  requiredCapabilities:
    - AIRY_CAP_NET_ACCESS
    - AIRY_CAP_IPC_SEND
  # 联动 NetworkPolicy 名称
  networkPolicyRef: allow-cognition-to-llm
  # 流量审计
  audit:
    enabled: true
    logLevel: INFO
    # 审计哈希链写入 MemoryRovol L1
    sink: memoryrovol
  # 异常流量自动封禁
  enforcement:
    mode: enforcing   # enforcing | permissive
    actionOnViolation: block   # block | throttle | alert
    violationThreshold: 3      # 3 次违规后封禁
```

---

## 3. CNI 插件设计

### 3.1 airymaxos-cni 插件架构

agentrt-linux 提供专属 CNI 插件，与 Cupolas 联动：

```
┌─────────────────────────────────────────────────────┐
│ kubelet CRI 调用 CNI                                │
│         |                                            │
│         v                                            │
│ ┌─────────────────────────────────────────────────┐ │
│ │ airymaxos-cni（CNI 插件）                       │ │
│ │  - 创建 veth pair                              │ │
│ │  - 配置 IP 地址                                │ │
│ │  - 注册至 Cupolas 网络过滤表                    │ │
│ │  - 应用 Landlock 网络规则                       │ │
│ └─────────────────────────────────────────────────┘ │
│         |                                            │
│         v                                            │
│ ┌─────────────────────────────────────────────────┐ │
│ │ Cupolas 网络穹顶（内核态 eBPF 程序）            │ │
│ │  - TC ingress/egress 钩子                       │ │
│ │  - XDP 程序（早期丢弃）                         │ │
│ │  - capability 令牌校验                         │ │
│ │  - 审计日志（哈希链）                           │ │
│ └─────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

### 3.2 CNI 插件接口实现

```go
// cloudnative/cni/airymaxos-cni.go
package main

import (
	"encoding/json"
	"fmt"
	"os"

	"github.com/containernetworking/cni/pkg/skel"
	"github.com/containernetworking/cni/pkg/types"
	"github.com/containernetworking/cni/pkg/version"
)

/* NetConfig airymaxos-cni 网络配置 */
type NetConfig struct {
	types.NetConf
	CupolasEnabled  bool     `json:"cupolasEnabled"`
	DefaultDeny    bool     `json:"defaultDeny"`
	AllowedPeers   []string `json:"allowedPeers"`
	AuditEnabled   bool     `json:"auditEnabled"`
	MasterPlugin   string   `json:"masterPlugin"`  /* 委托给 Calico/Cilium */
}

func main() {
	skel.PluginMainFuncs(skel.CNIFuncs{
		Add:   cmdAdd,
		Check: cmdCheck,
		Del:   cmdDel,
	}, version.All, "airymaxos-cni v0.1.1")
}

/* cmdAdd CNI ADD 命令：创建容器网络接口 */
func cmdAdd(args *skel.CmdArgs) error {
	cfg, err := parseNetConfig(args.StdinData)
	if err != nil {
		return err
	}

	/* 1. 委托给 master plugin（Calico/Cilium）完成标准 CNI */
	masterResult, err := delegateToMaster(cfg.MasterPlugin, args)
	if err != nil {
		return fmt.Errorf("master plugin failed: %w", err)
	}

	/* 2. 解析容器 Agent 元数据 */
	agentMeta, err := parseAgentMeta(args.ContainerID, args.IfName)
	if err != nil {
		return fmt.Errorf("parse agent meta failed: %w", err)
	}

	/* 3. 注册至 Cupolas 网络过滤表 */
	if cfg.CupolasEnabled {
		if err := registerToCupolas(agentMeta, cfg); err != nil {
			return fmt.Errorf("cupolas register failed: %w", err)
		}
	}

	/* 4. 应用 Landlock 网络规则 */
	if err := applyNetworkLandlock(agentMeta, cfg); err != nil {
		return fmt.Errorf("landlock apply failed: %w", err)
	}

	/* 5. 加载 eBPF TC 程序（ingress/egress 钩子） */
	if err := loadEBPFProgram(agentMeta, cfg); err != nil {
		return fmt.Errorf("ebpf load failed: %w", err)
	}

	return masterResult.Print()
}

/* cmdDel CNI DEL 命令：清理容器网络接口 */
func cmdDel(args *skel.CmdArgs) error {
	cfg, err := parseNetConfig(args.StdinData)
	if err != nil {
		return err
	}

	agentMeta, err := parseAgentMeta(args.ContainerID, args.IfName)
	if err == nil {
		/* 从 Cupolas 注销 */
		unregisterFromCupolas(agentMeta)
		/* 卸载 eBPF 程序 */
		unloadEBPFProgram(agentMeta)
	}

	/* 委托给 master plugin 清理 */
	return delegateDelToMaster(cfg.MasterPlugin, args)
}
```

### 3.3 Cupolas 网络过滤表

```go
/* registerToCupolas 向 Cupolas 注册 Agent 网络过滤规则 */
func registerToCupolas(meta *AgentMeta, cfg *NetConfig) error {
	/* 通过 /proc/agentrt/cupolas/net/register 写入注册信息 */
	regInfo := CupolasNetReg{
		AgentID:       meta.AgentID,
		ContainerID:   meta.ContainerID,
		NetNS:         meta.NetNS,
		VethHost:      meta.VethHost,
		VethContainer: meta.VethContainer,
		DefaultDeny:   cfg.DefaultDeny,
		AllowedPeers:  cfg.AllowedPeers,
		AuditEnabled:  cfg.AuditEnabled,
	}

	data, _ := json.Marshal(regInfo)
	return writeProcFile("/proc/agentrt/cupolas/net/register", data)
}
```

---

## 4. Cupolas 网络穹顶（内核 eBPF）

### 4.1 eBPF TC 程序

Cupolas 通过 eBPF TC 程序在内核网络栈进行流量过滤：

```c
/* security/ebpf/cupolas_net_tc.c */
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <linux/tcp.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>

/* Cupolas 网络策略映射表（内核态） */
struct cupolas_policy_key {
	__u32 src_agent_id;
	__u32 dst_agent_id;
	__u16 dst_port;
};

struct cupolas_policy_value {
	__u8 action;     /* 0=deny 1=allow 2=audit */
	__u8 audit_level;
};

struct {
	__uint(type, BPF_MAP_TYPE_LPM_TRIE);
	__uint(max_entries, 65536);
	__type(key, struct cupolas_policy_key);
	__type(value, struct cupolas_policy_value);
} cupolas_net_policy SEC(".maps");

/* TC ingress 钩子：进入容器的流量 */
SEC("tc_ingress")
int cupolas_tc_ingress(struct __sk_buff *skb)
{
	struct ethhdr *eth = bpf_hdr_pointer(skb, 0, sizeof(*eth));
	struct iphdr *ip = bpf_hdr_pointer(skb, sizeof(*eth), sizeof(*ip));
	struct tcphdr *tcp = bpf_hdr_pointer(skb, sizeof(*eth) + sizeof(*ip),
					      sizeof(*tcp));
	struct cupolas_policy_key key = {};
	struct cupolas_policy_value *policy;

	if (!eth || !ip || !tcp)
		return TC_ACT_SHOT;

	/* 查找策略 */
	/* 通过 BPF map（agent_map）维护 skb->mark → agent ID 映射，
	 * 替代 Linux 6.6 中不存在的 bpf_skb_get_agent_id() helper */
	key.src_agent_id = bpf_map_lookup_elem(&agent_map, &skb->mark);  /* 从 skb->mark 查 agent ID */
	key.dst_agent_id = bpf_map_lookup_elem(&agent_map, &skb->mark);   /* 同上，接收方通过 socket cookie */
	key.dst_port = bpf_ntohs(tcp->dest);

	policy = bpf_map_lookup_elem(&cupolas_net_policy, &key);
	if (!policy) {
		/* 默认拒绝：无策略匹配 */
		cupolas_audit_log(skb, "DENY_NO_POLICY");
		return TC_ACT_SHOT;
	}

	switch (policy->action) {
	case 0:  /* deny */
		cupolas_audit_log(skb, "DENY");
		return TC_ACT_SHOT;
	case 1:  /* allow */
		if (policy->audit_level > 0)
			cupolas_audit_log(skb, "ALLOW");
		return TC_ACT_OK;
	case 2:  /* audit only */
		cupolas_audit_log(skb, "AUDIT");
		return TC_ACT_OK;
	}

	return TC_ACT_SHOT;
}

/* TC egress 钩子：离开容器的流量 */
SEC("tc_egress")
int cupolas_tc_egress(struct __sk_buff *skb)
{
	/* 类似 ingress，检查发送方是否有 NET_ACCESS capability */
	return TC_ACT_OK;
}

char _license[] SEC("license") = "GPL";
```

### 4.2 XDP 早期丢弃

对于明确的恶意流量，Cupolas 通过 XDP 程序在网卡驱动层早期丢弃：

```c
/* security/ebpf/cupolas_xdp.c */
SEC("xdp")
int cupolas_xdp_drop(struct xdp_md *ctx)
{
	/* 检查源 IP 是否在黑名单 */
	/* 若是，直接 XDP_DROP，不进入内核协议栈 */
	return XDP_PASS;
}
```

### 4.3 capability 令牌校验

每个网络包的发送/接收均校验 Agent 是否持有 `AIRY_CAP_NET_ACCESS` capability：

```c
/* cupolas_check_net_cap 校验网络 capability */
int cupolas_check_net_cap(uint32_t agent_id, bool is_egress)
{
	uint64_t cap_token = cupolas_get_agent_cap_token(agent_id);
	uint64_t required = is_egress ?
		AIRY_CAP_NET_SEND : AIRY_CAP_NET_RECV;

	if (!(cap_token & (1ULL << required))) {
		cupolas_audit_log_violation(agent_id,
			"net capability missing");
		return -AIRY_EPERM;
	}
	return 0;
}
```

---

## 5. AgentsIPC 与 CNI 协同

### 5.1 通信路径选择

| 通信场景 | 路径 | 网络策略 |
|----------|------|----------|
| 同节点 Agent 间 | AgentsIPC（io_uring） | 不经 CNI（内核直通） |
| 跨节点 Agent 间 | CNI 网络（TCP） | NetworkPolicy 过滤 |
| Agent -> 外部服务 | CNI 网络（TCP） | Egress 策略过滤 |
| Agent -> daemon | AgentsIPC | 不经 CNI |

### 5.2 同节点优先 AgentsIPC

```c
/* airy_ipc_route 选择通信路径 */
int airy_ipc_route(uint32_t src_agent, uint32_t dst_agent,
		      void *payload, size_t len)
{
	/* 检查目标 Agent 是否同节点 */
	if (airy_is_local_agent(dst_agent)) {
		/* 同节点：走 io_uring 零拷贝 */
		return airy_ipc_send_local(src_agent, dst_agent,
					      payload, len);
	}

	/* 跨节点：走 CNI 网络 */
	return airy_ipc_send_remote(src_agent, dst_agent,
					payload, len);
}
```

### 5.3 网络性能对比

| 通信方式 | 延迟 | 吞吐 | 适用场景 |
|----------|------|------|----------|
| AgentsIPC（io_uring） | 微秒级 | 最高 | 同节点 Agent |
| CNI + Calico VXLAN | 毫秒级 | 中 | 跨节点 Agent |
| CNI + Cilium eBPF | 亚毫秒 | 高 | 跨节点 Agent |
| Unix socket | 微秒级 | 中 | Agent -> daemon |

---

## 6. 审计与可观测性

### 6.1 审计日志哈希链

所有网络策略决策记录至 Cupolas 审计哈希链（写入 MemoryRovol L1）：

```c
/* cupolas_audit_log 网络审计日志 */
struct cupolas_net_audit_record {
	uint64_t timestamp;
	uint32_t src_agent_id;
	uint32_t dst_agent_id;
	uint16_t src_port;
	uint16_t dst_port;
	uint8_t  action;        /* allow/deny/audit */
	uint8_t  reason;
	uint32_t packet_len;
	uint8_t  prev_hash[32]; /* SHA-256 前一记录哈希 */
	uint8_t  record_hash[32]; /* 当前记录哈希 */
};

int cupolas_audit_log(const struct __sk_buff *skb, const char *action)
{
	struct cupolas_net_audit_record rec = {0};
	uint8_t prev[32];

	/* 读取上一条记录哈希 */
	cupolas_get_last_hash(prev);
	memcpy(rec.prev_hash, prev, 32);

	rec.timestamp = ktime_get_real_ns();
	rec.action = parse_action(action);
	/* 填充 src/dst agent_id、port 等 */

	/* 计算当前记录哈希（SHA-256） */
	cupolas_sha256(&rec, sizeof(rec) - 32, rec.record_hash);

	/* 写入 MemoryRovol L1（append-only） */
	return airy_mr_l1_append(CUPOLAS_AUDIT_AGENT_ID,
					   &rec, sizeof(rec));
}
```

### 6.2 Prometheus 指标

CNI 插件与 Cupolas 暴露 Prometheus 指标：

```yaml
# 网络策略指标示例
- airy_net_policy_allow_total{src_agent="100",dst_agent="200"} 12345
- airy_net_policy_deny_total{src_agent="100",dst_agent="300"} 67
- airy_net_audit_records_total  456789
- airy_ipc_local_messages_total 1234567
- airy_ipc_remote_messages_total 45678
```

---

## 7. 测试与验证

### 7.1 网络策略测试矩阵

| 测试用例 | 策略配置 | 预期结果 |
|----------|----------|----------|
| 默认拒绝 | default-deny-all | 所有流量被拒 |
| 显式放行 | allow-cognition-to-llm | cognition -> llm 通 |
| 无 capability | cap_token 缺失 | 流量被 Cupolas 拒绝 |
| 出站白名单 | allow-egress-llm | 仅 443 端口通 |
| 内网禁止 | exceptions 10.0.0.0/8 | 内网访问被拒 |
| 审计记录 | audit enabled | MemoryRovol L1 有记录 |

### 7.2 端到端测试

```bash
# 部署网络策略
kubectl apply -f default-deny-all.yaml
kubectl apply -f allow-cognition-to-llm.yaml

# 测试 Agent 间通信（应被拒绝）
kubectl exec cognition-agent -- curl llm-agent:7766
# 预期：超时或拒绝连接

# 应用放行策略后重试
kubectl apply -f allow-cognition-to-llm.yaml
kubectl exec cognition-agent -- curl llm-agent:7766
# 预期：连接成功

# 验证审计日志
cat /var/lib/agentrt/memoryrovol/cupolas-audit/last_record
```

---

## 8. 相关文档

- `150-cloudnative/README.md`（云原生主索引）
- `150-cloudnative/01-k8s-crd-design.md`（K8s CRD 设计）
- `150-cloudnative/02-containerd-shim.md`（containerd shim 设计）
- `30-interfaces/02-ipc-protocol.md`（AgentsIPC 协议）
- `110-security/README.md`（Cupolas 安全穹顶）

---

## 9. 参考材料

- Kubernetes NetworkPolicy 文档
- CNI Specification 1.0
- Calico 项目
- Cilium 项目（eBPF 网络方案）
- Linux 6.6 TC（Traffic Control）子系统
- Linux 6.6 XDP（eXpress Data Path）
- eBPF 程序设计
- Cupolas 安全模型（IRON-9 v2 [SS] 语义同源层）

---

> **文档结束** | agentrt-linux（AirymaxOS）CNI 网络策略设计 v0.1.1
> 遵循 IRON-9 v2 [SS] Cupolas 安全模型同源 + [IND] CNI 实现独立
