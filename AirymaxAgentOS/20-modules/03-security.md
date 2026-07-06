# AirymaxOS 安全设计文档（airymaxos-security，极境安全）

> 子仓编号：03
> 子仓代号：极境安全（Airymax Security）
> 文档版本：v1.0（2026-07-06）
> 设计基准：capability 安全 + LSM + 机密计算
> 同源 agentrt：cupolas（Cupolas）

---

## 1. 子仓职责

`airymaxos-security` 是 AirymaxOS 的安全子仓，承担以下核心职责：

1. **capability 系统**：基于 seL4 风格的不可伪造令牌实现最小权限访问控制。
2. **LSM Hook**：agent_lsm 提供 Linux Security Module 钩子，与 Cupolas 同源。
3. **沙箱隔离**：Landlock + seccomp 构建用户态沙箱。
4. **机密计算**：支持 TEE/SGX/SEV-SNP/TDX 等可信执行环境。
5. **国密算法**：遵循 AirymaxOS 标准集成 SM2/SM3/SM4 国密算法。
6. **零信任网络**：基于身份的零信任网络架构。
7. **eBPF kfunc + dynamic pointer**：利用 Linux 6.6 原生特性对 eBPF 程序进行签名验证。

---

## 2. 同源关系

| 维度 | agentrt（cupolas） | AirymaxOS（airymaxos-security） |
|------|-------------------|--------------------------------|
| LSM | Cupolas | agent_lsm（Cupolas 同源） |
| 沙箱 | 进程沙箱 | Landlock + seccomp + capability |
| 加密 | 应用层加密 | 国密 + TEE 机密计算 |
| 权限 | 应用权限模型 | capability 令牌系统 |

**同源传承要点**：
- 保留 agentrt Cupolas 的"LSM hook 风格"安全策略注入。
- 保留 Cupolas 的"声明式安全策略"配置范式。
- 升级为 OS 级 capability 系统（seL4 风格）。

---

## 3. 目录结构

```
airymaxos-security/
├── capability/             # capability 系统（seL4 风格）
├── lsm/                    # agent_lsm（LSM hook）
├── sandbox/               # Landlock + seccomp
├── confidential-compute/   # 机密计算（TEE/SGX/SEV-SNP/TDX）
├── crypto/                 # 国密算法（AirymaxOS 标准）
├── zero-trust/             # 零信任网络
├── ebpf-verify/            # eBPF kfunc + dynamic pointer（6.6 原生）
└── docs/
```

### 3.1 capability/（capability 系统）

参考 **seL4 capability** 设计：
- `cap-types`：capability 类型定义（CNode、Endpoint、Thread、Frame、IO 等）。
- `cap-table`：capability 表（per-process 树状结构）。
- `cap-transfer`：跨进程 capability 传递（通过消息传递）。
- `cap-derive`：capability 派生（mint、mintcopy、derive）。
- `cap-revoke`：capability 撤销（递归撤销派生 capability）。

### 3.2 lsm/（agent_lsm）

与 Cupolas 同源的 LSM 实现：
- `agent_lsm.c`：LSM hook 注册（file_ops、net_ops、task_ops、ipc_ops）。
- `policy-engine`：声明式策略引擎（YAML/JSON 策略）。
- `policy-compiler`：策略编译器（编译为 BPF 程序）。
- `audit`：安全审计日志。

### 3.3 sandbox/（Landlock + seccomp）

- `landlock-rules/`：Landlock 规则集（文件访问控制）。
- `seccomp-filters/`：seccomp BPF 过滤器（系统调用白名单）。
- `sandbox-runner`：沙箱启动器（创建 namespace + 应用规则）。
- `bubblewrap`：bubblewrap 集成（容器化沙箱）。

### 3.4 confidential-compute/（机密计算）

支持多种 TEE 技术：
- `sgx/`：Intel SGX enclave 支持。
- `sev-snp/`：AMD SEV-SNP 虚拟机加密。
- `tdx/`：Intel TDX 信任域。
- `arm-cca/`：ARM CCA 机密计算架构。
- `attestation`：远程证明框架。
- `key-broker`：密钥代理服务（与 KBS 协作）。

### 3.5 crypto/（国密算法）

遵循 **AirymaxOS 安全治理组** 标准：
- `sm2/`：SM2 椭圆曲线公钥密码算法。
- `sm3/`：SM3 密码杂凑算法。
- `sm4/`：SM4 分组密码算法。
- `sm9/`：SM9 标识密码算法。
- `tls-gm`：国密 TLS 协议支持。
- `openssl-provider`：OpenSSL provider 集成。

### 3.6 zero-trust/（零信任网络）

- `identity`：身份认证服务（基于 capability）。
- `policy-engine`：零信任策略引擎（持续验证）。
- `micro-segmentation`：微分段网络隔离。
- `mtls`：双向 TLS 通信。
- `beyondcorp`：BeyondCorp 模型参考。

### 3.7 ebpf-verify/（eBPF kfunc + dynamic pointer）

利用 **Linux 6.6** 原生特性：
- `signing`：eBPF 程序签名工具。
- `verification`：内核签名验证集成。
- `keyring`：签名密钥环管理。
- `policy`：仅允许已签名 eBPF 程序加载的策略。

---

## 4. 核心特性

### 4.1 capability 系统（seL4 风格）

**设计原则**：
- 不可伪造：capability 由内核生成，用户态无法伪造。
- 可传递：capability 可通过消息传递给其他进程。
- 可派生：capability 可派生子 capability（受限权限）。
- 可撤销：capability 可被递归撤销。

**capability 类型**：
| 类型 | 权限 |
|------|------|
| CNode | capability 表节点 |
| Endpoint | 消息端点（IPC） |
| Thread | 线程控制 |
| Frame | 物理页映射 |
| IO | I/O 端口访问 |
| IRQ | 中断控制 |
| ASID | 地址空间标识 |

### 4.2 agent_lsm（LSM hook，Cupolas 同源）

**LSM Hook 点**：
- `file_open`、`file_permission`、`file_ioctl`
- `socket_bind`、`socket_connect`、`socket_accept`
- `task_create`、`task_kill`、`task_setpgid`
- `ipc_permission`、`msg_queue_msgctl`

**策略示例**（YAML）：
```yaml
policy: agent-default
version: v1
rules:
  - name: restrict-network
    selector:
      domain: untrusted.slice
    effect: deny
    operations:
      - socket_connect
      - socket_bind
  - name: allow-vfs-read
    selector:
      domain: agent.slice
    effect: allow
    operations:
      - file_open
    paths:
      - /var/lib/airymaxos/**
```

### 4.3 Landlock + seccomp（用户态沙箱）

**Landlock**：
- 文件路径访问控制。
- 网络绑定/连接控制。
- 进程间通信控制。

**seccomp**：
- 系统调用白名单/黑名单。
- 参数级过滤（BPF 过滤器）。

**组合使用**：
```c
// 创建沙箱
sandbox_create()
    -> landlock_restrict_paths(rules)
    -> seccomp_load(filter)
    -> execve(target);
```

### 4.4 机密计算（TEE/SGX/SEV-SNP/TDX）

**支持的 TEE**：
| 技术 | 厂商 | 粒度 |
|------|------|------|
| Intel SGX | Intel | Enclave |
| Intel TDX | Intel | VM |
| AMD SEV-SNP | AMD | VM |
| ARM CCA | ARM | Realm |

**应用场景**：
- LLM 推理保护（模型权重、推理数据）。
- 密钥管理（HSM 替代）。
- 隐私计算（联邦学习）。
- Agent 记忆加密（与 `airymaxos-memory` 协作）。

### 4.5 eBPF kfunc + dynamic pointer（6.6 原生特性）

**特性**：
- Linux 6.6 引入 eBPF 程序签名验证机制。
- 仅允许已签名 eBPF 程序加载至内核。
- 防止恶意 eBPF 程序破坏内核安全。

**策略**：
- `enforce`：仅允许已签名程序加载。
- `log`：记录未签名程序加载尝试。
- `disable`：禁用签名验证（仅开发环境）。

### 4.6 国密算法支持（AirymaxOS 标准）

遵循 **GB/T 32918、GB/T 32905、GB/T 32907** 等国密标准：
- SM2：公钥密码（替代 RSA/ECDSA）。
- SM3：杂凑算法（替代 SHA-256）。
- SM4：分组密码（替代 AES）。
- SM9：标识密码（基于身份的加密）。
- 国密 TLS：GMT 0024 标准的 TLS 协议。

### 4.7 零信任网络架构

**核心原则**：
- 从不信任，始终验证（Never trust, always verify）。
- 最小权限访问。
- 持续验证身份与设备状态。
- 微分段隔离。

**架构**：
- 身份认证（基于 capability）。
- 设备健康度评估。
- 持续授权（动态策略）。
- 微分段（网络隔离）。

---

## 5. 微内核思想体现

### 5.1 capability-based security

遵循 **seL4 capability** 模型：
- 所有权限通过 capability 令牌表示。
- capability 不可伪造、可传递、可派生、可撤销。
- 每个进程仅持有完成其功能所需的最小 capability 集。

### 5.2 最小权限

- 进程仅持有必要的 capability。
- 沙箱限制系统调用与文件访问。
- LSM hook 在每个安全敏感操作点验证权限。

### 5.3 安全即机制

- capability 系统：安全机制（在内核）。
- 策略引擎：安全策略（在用户态，可热替换）。
- 符合 Liedtke 的"机制与策略分离"原则。

---

## 6. AirymaxOS 工程基线

- **AirymaxOS 安全治理组**：安全子系统最佳实践。
- **AirymaxOS 国密**：SM2/SM3/SM4 国密算法实现。
- **AirymaxOS LSM**：LSM hook 集成经验。
- **AirymaxOS 机密计算**：TEE 集成基线。

---

## 7. 前沿理论参考

| 理论 | 来源 | 应用 |
|------|------|------|
| seL4 capability | seL4 项目 | capability 系统设计 |
| Zircon handle | Google Zircon | capability 句柄传递 |
| eBPF kfunc + dynamic pointer | Linux 6.6 | eBPF 程序签名验证 |
| 机密计算 | CCC | TEE 集成 |
| Landlock | Linux 5.x+ | 用户态沙箱 |
| seccomp BPF | Linux | 系统调用过滤 |
| LSM | Linux | 安全模块 hook |
| BeyondCorp | Google | 零信任网络 |
| 国密标准 | GB/T | 国密算法集成 |

---

## 8. 与其他子仓的协作

| 协作子仓 | 协作内容 |
|---------|---------|
| `airymaxos-kernel` | 提供 capability 内核接口、LSM hook 注册点 |
| `airymaxos-services` | 为每个服务颁发 capability、应用 LSM 策略 |
| `airymaxos-memory` | 提供 MemoryRovol 加密、TEE 保护 |
| `airymaxos-cognition` | 提供 Wasm 沙箱、LLM 推理 TEE 保护 |
| `airymaxos-cloudnative` | 提供容器沙箱、零信任网络 |
| `airymaxos-system` | 提供安全配置工具 |
| `airymaxos-tests` | 安全测试、形式化验证 |

---

## 9. 里程碑

| 阶段 | 目标 | 时间 |
|------|------|------|
| M1 | capability 系统内核接口 | 2026 Q3 |
| M2 | agent_lsm 集成 + 策略引擎 | 2026 Q4 |
| M3 | Landlock + seccomp 沙箱 | 2027 Q1 |
| M4 | 国密算法集成 | 2027 Q2 |
| M5 | 机密计算（SGX/SEV-SNP） | 2027 Q3 |
| M6 | 零信任网络 + eBPF kfunc + dynamic pointer | 2027 Q4 |

---

## 10. 参考

- seL4 项目文档（capability 系统）
- Zircon 内核设计文档（handle 传递）
- Linux 6.6 eBPF kfunc + dynamic pointer 文档
- CCC（Confidential Computing Consortium）白皮书
- Landlock 官方文档
- seccomp BPF 教程
- AirymaxOS 安全治理组文档
- 国密标准文档（GB/T 32918 等）
- agentrt cupolas 设计文档
