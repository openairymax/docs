Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# AirymaxOS 安全加固设计

> **文档定位**: AirymaxOS（agentrt-linux）安全工程体系主索引
> **版本**: 0.1.1（占位）/ 1.0.1（开发）
> **最后更新**: 2026-07-06
> **同源映射**: agentrt Cupolas（安全穹顶）+ Linux 6.6 LSM/Landlock/capability
> **理论根基**: Linux 内核安全机制 + Airymax E-1 安全内生 + K-3 服务隔离

---

## 1. 模块定位

AirymaxOS 安全加固体系是系统可信运行的核心保障。它继承 Linux 内核 30+ 年沉淀的多层安全哲学（LSM 框架 + Landlock 用户态沙箱 + capability + 模块签名 + Lockdown + 4 层密钥环），并在其上扩展智能体操作系统专属的 Cupolas 安全穹顶、机密计算、国密算法等。

### 1.1 安全体系分层

| 层级 | 类型 | 机制 | 范围 |
|------|------|------|------|
| L1 | LSM 框架 | `security_hook_heads` + `lsm_blob_sizes` + 排序 | 内核安全钩子 |
| L2 | 用户态沙箱 | Landlock（非特权进程自限制） | 进程级隔离 |
| L3 | capability | seL4 风格 capability + POSIX caps | 权限细粒度 |
| L4 | 模块签名 | eBPF 签名验证 + 模块签名 | 代码完整性 |
| L5 | Lockdown | 内核 Lockdown 模式 | 限制 root |
| L6 | 密钥环 | builtin/secondary/machine/platform 4 层 | 密钥管理 |
| **L7** | **Cupolas 安全穹顶** | **AirymaxOS 专属** | **Agent 行为约束** |
| **L8** | **机密计算** | **VirtCCA CVM** | **可信执行环境** |
| **L9** | **国密算法** | **SM2/SM3/SM4** | **合规要求** |

### 1.2 AirymaxOS 扩展

- **Cupolas 安全穹顶**：从 agentrt 同源的 7 大子系统（Guards/Permission/Sanitizer/Audit/Workbench/Vault/Network）
- **机密计算**：基于 VirtCCA 的 CVM（Confidential Virtual Machine）
- **国密算法**：SM2（签名）/SM3（哈希）/SM4（对称加密）合规支持

---

## 2. 核心安全机制

### 2.1 LSM 框架

LSM（Linux Security Module）是内核安全钩子框架：
```c
// security/security.c
struct security_hook_heads security_hook_heads __ro_after_init;
struct lsm_blob_sizes __ro_after_init;

// 安全钩子声明
struct security_hook_heads {
    struct hlist_heads inode_alloc_security;
    struct hlist_heads file_open;
    struct hlist_heads task_create;
    // ...
};
```

LSM 排序机制：`chosen_lsm_order` / `chosen_major_lsm` / `builtin_lsm_order` 控制多个 LSM 共存。

### 2.2 Landlock（用户态沙箱）

Landlock 允许非特权进程定义自己的安全策略：
```c
// security/landlock/setup.c
struct lsm_blob_sizes landlock_blob_sizes __ro_after_init = {
    .lbs_cred = sizeof(struct landlock_cred_security),
    .lbs_file = sizeof(struct landlock_file_security),
    .lbs_inode = sizeof(struct landlock_inode_security),
    .lbs_superblock = sizeof(struct landlock_superblock_security),
};
```

Landlock 提供：
- 文件系统细粒度访问控制
- 网络细粒度访问控制
- 声明式安全策略（进程通过声明规则限制自身能力）

### 2.3 capability（seL4 风格）

AirymaxOS 采用 seL4 风格的 capability 安全模型：
- **能力传递**：通过 capability 传递权限，而非 ACL
- **能力撤销**：支持运行时撤销已授予的 capability
- **能力审计**：所有 capability 操作可审计

### 2.4 模块签名

内核模块与 eBPF 程序必须签名验证：
- `CONFIG_MODULE_SIG=y`：内核模块签名
- `CONFIG_BPF_JIT_ALWAYS_ON=y`：eBPF JIT 始终启用
- Linux 6.15+：eBPF 程序签名验证

### 2.5 Lockdown

内核 Lockdown 模式限制 root 权限：
- `integrity`：阻止修改运行中内核
- `confidentiality`：阻止内核状态泄露

### 2.6 4 层密钥环

| 密钥环 | 用途 | 来源 |
|--------|------|------|
| `.builtin_trusted_keys` | 编译时内嵌可信密钥 | 内核镜像 |
| `.secondary_trusted_keys` | 运行时添加可信密钥 | 管理员 |
| `.machine` | 硬件可信密钥 | TPM |
| `.platform` | 平台可信密钥 | 固件 |

---

## 3. 文档索引

```
110-security/
├── README.md                       # 本文件
├── 01-lsm-framework.md             # LSM 框架详解
├── 02-landlock-sandbox.md          # Landlock 用户态沙箱
├── 03-capability-model.md          # seL4 风格 capability
├── 04-module-signing.md            # 模块签名验证
├── 05-lockdown.md                  # 内核 Lockdown 模式
├── 06-keyrings.md                  # 4 层密钥环
├── 07-cupolas-dome.md              # AirymaxOS 专属：Cupolas 安全穹顶
├── 08-confidential-computing.md    # AirymaxOS 专属：机密计算（VirtCCA）
└── 09-cryptography-compliance.md   # AirymaxOS 专属：国密算法合规
```

### 3.1 0.1.1 版本范围

仅完成 README + 01 + 02（3 文档）。

### 3.2 1.0.1 版本范围

完成全部 9 文档，并实施安全工程标准。

---

## 4. AirymaxOS 专属扩展

### 4.1 Cupolas 安全穹顶

从 agentrt 同源的 7 大子系统：
| 子系统 | 职责 | AirymaxOS 实现 |
|--------|------|----------------|
| Guards 守卫 | 入口防护 | 内核态 + 用户态双层守卫 |
| Permission 权限裁决 | 策略裁决 | capability + LSM 钩子 |
| Sanitizer 输入净化 | 输入验证 | 内核态输入过滤 |
| Audit 审计追踪 | 行为审计 | ftrace + eBPF 审计 |
| Workbench 虚拟工作台 | 隔离环境 | Landlock 沙箱 |
| Security Vault 安全金库 | 敏感数据保护 | TPM + 机密计算 |
| Network Security 网络安全 | 网络防护 | LSM + 防火墙 |

### 4.2 机密计算（VirtCCA）

基于 VirtCCA 的 CVM（Confidential Virtual Machine）：
- **内存加密**：虚拟机内存对宿主机不可见
- **CPU 隔离**：虚拟机 CPU 状态对宿主机不可见
- **中断隔离**：虚拟机中断对宿主机不可见

### 4.3 国密算法合规

- **SM2**：椭圆曲线数字签名算法（替代 ECDSA）
- **SM3**：密码哈希算法（替代 SHA-256）
- **SM4**：分组密码算法（替代 AES）

### 4.4 同源 agentrt 安全

agentrt 的 `cupolas/` 模块与 AirymaxOS 安全体系同源：
- agentrt 用户态：`agentrt_cupolas_*` API
- AirymaxOS 内核态：LSM 钩子 + capability
- 两端通过 AgentsIPC 协议传递安全策略

---

## 5. 五维原则映射

| 原则 | 在本模块的体现 |
|------|---------------|
| **E-1 安全内生** | 安全机制内置于系统每一层 |
| **K-3 服务隔离** | Landlock 沙箱 + 进程隔离 |
| **K-4 可插拔策略** | LSM 钩子可插拔 |
| **IRON-9 同源但独立** | Cupolas 与 agentrt 同源 |
| **A-4 完美主义** | 形式化验证 + 机密计算 |

---

## 6. 相关文档

- `50-engineering-standards/01-coding-standards.md`（安全编码规范）
- `80-testing/README.md`（安全测试）
- `90-observability/README.md`（安全审计）
- `20-modules/03-security.md`（security 子仓设计）

---

## 7. 参考材料

- Linux 6.6 `security/security.c`（LSM 框架核心）
- Linux 6.6 `security/landlock/`（Landlock 实现）
- Linux 6.6 `security/keys/`（密钥环实现）
- Linux 6.6 `Documentation/admin-guide/lockdown.rst`（Lockdown）
- seL4 项目（capability 安全模型参考）

---

> **文档结束** | 0.1.1 P0 优先完成 README + 01 + 02
