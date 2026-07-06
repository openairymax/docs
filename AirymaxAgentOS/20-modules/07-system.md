# AirymaxOS 系统设计文档（airymaxos-system，极境系统）

> 子仓编号：07
> 子仓代号：极境系统（Airymax System）
> 文档版本：v1.0（2026-07-06）
> 设计基准：包管理 + 配置 + shell + 基础库
> 同源 agentrt：commons

---

## 1. 子仓职责

`airymaxos-system` 是 AirymaxOS 的系统管理工具子仓，承担以下核心职责：

1. **包管理**：基于 RPM + dnf 的包管理系统（AirymaxOS 标准）。
2. **配置工具**：系统配置工具（sysctl、systemd-config 等）。
3. **shell**：提供 bash + fish 等 shell 环境。
4. **基础库**：提供 glibc + musl 基础 C 库。
5. **系统监控**：提供 top、htop、perf 等监控工具。
6. **DevStation**：基于 AirymaxOS DevStation，提供 AI 智能助手辅助开发运维。

作为发行版必需的工具集合，本子仓为其他子仓提供基础系统工具支持。

---

## 2. 同源关系

| 维度 | agentrt（commons） | AirymaxOS（airymaxos-system） |
|------|--------------------|-------------------------------|
| 公共工具 | commons（应用层） | 系统管理工具（OS 级） |
| 配置 | 应用配置 | 系统配置（sysctl、systemd） |
| 监控 | 应用监控 | 系统监控（top、htop、perf） |
| 助手 | 自研助手 | DevStation（AirymaxOS 自研） |

**同源传承要点**：
- 保留 agentrt commons 的"公共工具"语义。
- 升级为 OS 级系统管理工具。
- 基于 AirymaxOS DevStation 提供 AI 智能助手。

---

## 3. 目录结构

```
airymaxos-system/
├── package-manager/        # 包管理（RPM + dnf）
├── config/                 # 配置工具
├── shell/                  # shell（bash + fish）
├── base-libs/              # 基础库（glibc + musl）
├── monitoring/             # 系统监控
├── devstation/             # DevStation（AI 智能助手）
└── docs/
```

### 3.1 package-manager/（包管理）

遵循 **AirymaxOS 标准**：
- `rpm/`：RPM 包构建工具（rpmbuild）。
- `dnf/`：dnf 包管理器配置。
- `repos/`：仓库配置（airymaxos.repo）。
- `build/`：构建脚本（mock、koji 集成）。
- `sign/`：包签名（GPG）。

### 3.2 config/（配置工具）

- `sysctl/`：sysctl 配置（/etc/sysctl.d/）。
- `systemd-config/`：systemd 配置（/etc/systemd/）。
- `network-config/`：网络配置（NetworkManager）。
- `kernel-config/`：内核配置工具（kernel-config）。
- `airymaxos-config/`：AirymaxOS 专属配置。

### 3.3 shell/（shell）

- `bash/`：Bash shell（默认）。
- `fish/`：Fish shell（友好交互）。
- `zsh/`：Zsh shell（可选）。
- `oh-my-zsh/`：Oh My Zsh 配置（可选）。

### 3.4 base-libs/（基础库）

- `glibc/`：GNU C Library（默认）。
- `musl/`：musl libc（精简、嵌入式场景）。
- `compat-libs/`：兼容库（32 位、旧版本兼容）。
- `static-libs/`：静态库。

### 3.5 monitoring/（系统监控）

- `procps/`：top、ps、free 等基础工具。
- `htop/`：htop 交互式监控。
- `perf/`：perf 性能分析工具。
- `bpftrace/`：bpftrace 动态追踪。
- `sysstat/`：sar、iostat 等系统统计。
- `airymaxmon/`：AirymaxOS 专属监控工具。

### 3.6 devstation/（DevStation，AI 智能助手）

基于 **AirymaxOS DevStation**：
- `ai-assistant`：AI 智能助手（自然语言交互）。
- `dev-tools`：开发工具集成。
- `ops-tools`：运维工具集成。
- `knowledge-base`：知识库（AirymaxOS 文档）。
- `auto-fix`：自动修复（常见问题自动诊断与修复）。
- `code-gen`：代码生成（基于 LLM）。

---

## 4. 核心特性

### 4.1 包管理（RPM + dnf，AirymaxOS 标准）

遵循 **AirymaxOS 包管理标准**：
- RPM 包格式：与 AirymaxOS、Fedora、CentOS 兼容。
- dnf 包管理器：依赖解析、仓库管理、事务处理。
- 包签名：GPG 签名验证，防止篡改。
- 模块化：支持模块（module）与流（stream）。

**仓库配置**：
```ini
# /etc/yum.repos.d/airymaxos.repo
[airymaxos-base]
name=AirymaxOS Base Repository
baseurl=https://repo.airymaxos.io/$releasever/base/$basearch/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-airymaxos

[airymaxos-updates]
name=AirymaxOS Updates Repository
baseurl=https://repo.airymaxos.io/$releasever/updates/$basearch/
enabled=1
gpgcheck=1
```

### 4.2 配置工具（sysctl、systemd-config）

**sysctl 配置**：
```ini
# /etc/sysctl.d/99-airymaxos.conf
# 调度优化
kernel.sched_agent_enabled = 1

# 内存优化
vm.swappiness = 10
vm.watermark_scale_factor = 125
vm.mglru_enabled = 1

# io_uring 优化
fs.io_uring_max_entries = 32768

# 网络优化
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
```

**systemd 配置**：
- 默认 target：`airymaxos.target`。
- 资源限制：cgroup v2 配置。
- 日志：journald 配置。
- 网络：systemd-networkd 配置。

### 4.3 shell（bash + fish）

**默认 shell：bash**
- 兼容 POSIX。
- 丰富的脚本生态。

**可选 shell：fish**
- 友好的交互式体验。
- 智能补全。
- 语法高亮。

### 4.4 基础库（glibc、musl）

**默认：glibc**
- 完整 POSIX 兼容。
- 广泛的软件生态。
- 性能优化（glibc 2.40+）。

**可选：musl**
- 精简、轻量。
- 静态链接友好。
- 嵌入式场景适用。

### 4.5 系统监控（top、htop、perf）

**top/htop**：进程级监控。
**perf**：性能分析（基于 perf_events）。
**bpftrace**：动态追踪（基于 eBPF）。
**sysstat**：系统统计（sar、iostat）。

**AirymaxOS 专属监控**：
- `airymaxmon`：综合监控 Agent 认知循环、LLM 推理、GPU/NPU 利用率等。
- 与 `airymaxos-cloudnative/observability` 集成。

### 4.6 DevStation（基于 AirymaxOS AI 智能助手）

基于 **AirymaxOS DevStation**：
- AI 智能助手：自然语言交互，辅助开发运维。
- 知识库：集成 AirymaxOS 文档。
- 自动修复：常见问题自动诊断与修复。
- 代码生成：基于 LLM 生成配置脚本、systemd unit 等。
- 与 `airymaxos-cognition` 协作，调用 LLM 推理。

**示例交互**：
```bash
$ devstation
> 我的 Agent 进程 OOM 了，怎么排查？
DevStation: 检测到 agent.slice 内存使用达到上限。
建议：
1. 查看 dmesg 是否有 OOM killer 记录：dmesg | grep -i oom
2. 查看 cgroup 内存限制：systemctl status agent.slice
3. 提升 agent.slice 内存限制：systemctl set-property agent.slice MemoryMax=8G
是否需要我执行上述命令？
> 是
```

---

## 5. 微内核思想体现

### 5.1 发行版必需工具

本子仓提供发行版必需的工具：
- 包管理：RPM + dnf。
- 配置：sysctl、systemd。
- shell：bash、fish。
- 基础库：glibc、musl。

这些工具不属于微内核本身，而是发行版的"用户态工具集"。

### 5.2 工具与内核解耦

- 工具通过标准接口（系统调用、sysctl、procfs）与内核交互。
- 工具可独立升级，不影响内核稳定性。
- 符合微内核"机制在内核，工具在用户态"原则。

### 5.3 AI 原生工具

- DevStation 引入 AI 智能助手，体现 AI 原生 OS 设计。
- 与 `airymaxos-cognition` 协作，调用 LLM 能力。
- 工具智能化，降低运维门槛。

---

## 6. AirymaxOS 工程基线

- **AirymaxOS 基础系统治理组**：基础系统工具最佳实践。
- **AirymaxOS 包管理**：RPM + dnf 集成经验。
- **AirymaxOS DevStation**：AI 智能助手基线。
- **AirymaxOS 监控工具**：系统监控工具基线。
- **AirymaxOS shell**：shell 配置基线。

---

## 7. 前沿理论参考

| 理论 | 来源 | 应用 |
|------|------|------|
| AirymaxOS DevStation | AirymaxOS | AI 智能助手 |
| 标准 Linux 工具链 | Linux 生态 | 系统工具 |
| RPM + dnf | Fedora/AirymaxOS | 包管理 |
| systemd | systemd 项目 | 系统管理 |
| bpftrace | eBPF | 动态追踪 |
| perf | Linux | 性能分析 |

---

## 8. 与其他子仓的协作

| 协作子仓 | 协作内容 |
|---------|---------|
| `airymaxos-kernel` | 提供内核配置工具所需接口 |
| `airymaxos-services` | 提供 systemd unit 管理 |
| `airymaxos-security` | 提供安全配置工具 |
| `airymaxos-memory` | 提供内存监控工具 |
| `airymaxos-cognition` | DevStation 调用 LLM 能力 |
| `airymaxos-cloudnative` | 提供云原生监控集成 |
| `airymaxos-tests` | 提供测试工具 |

---

## 9. 里程碑

| 阶段 | 目标 | 时间 |
|------|------|------|
| M1 | RPM + dnf 仓库搭建 | 2026 Q3 |
| M2 | 基础库（glibc）+ shell（bash） | 2026 Q4 |
| M3 | 配置工具（sysctl、systemd） | 2027 Q1 |
| M4 | 系统监控工具（top、htop、perf） | 2027 Q2 |
| M5 | DevStation Phase 1（AI 助手） | 2027 Q3 |
| M6 | DevStation Phase 2（自动修复） | 2027 Q4 |

---

## 10. 参考

- AirymaxOS 基础系统治理组文档
- AirymaxOS DevStation 文档
- RPM 项目文档
- dnf 项目文档
- systemd 官方文档
- glibc 文档
- musl libc 文档
- bpftrace 文档
- perf 文档
- agentrt commons 设计文档
