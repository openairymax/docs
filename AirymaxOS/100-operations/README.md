Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）运维体系设计
> **文档定位**：agentrt-linux（AirymaxOS）运维工程体系主索引（部署 + 配置 + 监控 + 告警 + 日志收集 + 12 daemon 清单）\
> **文档版本**：v1.0\
> **最后更新**：2026-07-17\
> **上级文档**：[AirymaxOS 总览](../README.md)\
> **同源映射**：agentrt daemons（12 个用户态服务）+ Linux 6.6 systemd 集成\
> **理论根基**：Linux 发行版运维哲学 + Airymax S-1 反馈闭环 + E-2 可观测性

---

## 1. 模块概述

agentrt-linux 运维体系是系统稳定运行的核心保障。它继承 Linux 发行版 30+ 年沉淀的运维哲学（systemd 集成 + 包管理 + 配置管理 + 监控告警 + 故障恢复），并在其上扩展智能体操作系统专属的 Agent 运维（Agent 健康检查、Token 预算监控、记忆卷载运维等）。本目录覆盖五方面职责：

1. **部署**：systemd 集成 + RPM/dnf 包管理 + ISO 安装镜像；12 个 daemons 以 `agentrt-*.service` 格式纳入 systemd 单元管理。
2. **配置**：sysctl 内核运行时参数 + `/etc/agentrt/` 配置文件（与 agentrt 同源）+ A-UCS（Unified Configuration Subsystem）的 JSON/TOML 双向热重载。
3. **监控**：ftrace + eBPF + perf + Prometheus + Grafana；sched_tac 调度器状态、Agent 8 态生命周期、Token 预算消耗实时监控。
4. **告警**：Alertmanager + 自定义规则；Micro-Supervisor 冷酷执法事件、Macro-Supervisor 温情裁决事件、IPC 队列冻结事件告警。
5. **日志收集**：A-ULP（Unified Logging and Printk Subsystem）的 Ring Buffer + Logger Daemon 异步消费 + Loki 日志聚合；Panic 回退到 `printk_safe` 路径的日志保活。

### 1.1 12 daemon 清单（统一归属 services/daemons/）

agentrt-linux 的 12 个用户态守护进程统一归属 `services/daemons/` 目录，与 agentrt 同源。每个 daemon 对应一个 systemd 服务单元（`agentrt-*.service` 格式）：

| # | Daemon 名称 | systemd 服务 | Unify 模块归属 | 核心职责 |
|---|------------|-------------|---------------|---------|
| 1 | `macro_d` | `agentrt-macro-superv.service` | A-ULS | Macro-Supervisor 温情裁决（接收 Micro-Supervisor 故障通知） |
| 2 | `logger_d` | `agentrt-logger-daemon.service` | A-ULP | 日志格式化、过滤、落盘（消费 Ring Buffer） |
| 3 | `config_d` | `agentrt-config-daemon.service` | A-UCS | 配置热重载分发（sysctl ↔ JSON 双向同步） |
| 4 | `gateway_d` | `agentrt-gateway.service` | A-IPC | IPC 网关守护（io_uring 命令路由） |
| 5 | `sched_d` | `agentrt-sched.service` | A-ULS | 调度策略守护（sched_tac 参数注入） |
| 6 | `vfs_d` | `agentrt-vfs.service` | A-ULS | VFS 用户态化守护（内核保留路径解析） |
| 7 | `net_d` | `agentrt-net.service` | A-IPC | 网络子系统集成守护 |
| 8 | `mem_d` | `agentrt-mem.service` | A-UEF | 记忆子系统守护（CXL/PMEM/MGLRU） |
| 9 | `cogn_d` | `agentrt-cogn.service` | A-UEF | 认知循环守护（CoreLoopThree kthread） |
| 10 | `sec_d` | `agentrt-sec.service` | A-ULS | 安全守护（纯 C LSM 策略同步） |
| 11 | `audit_d` | `agentrt-audit.service` | A-ULP | 审计追踪守护（capability 操作审计） |
| 12 | `dev_d` | `agentrt-dev.service` | A-IPC | 设备生命周期守护（VFIO 直通 + io_uring 设备 DMA） |

### 1.2 运维体系分层

| 层级 | 类型 | 工具/机制 | 范围 |
|------|------|---------|------|
| L1 | 部署 | systemd + 包管理（RPM/dnf）+ ISO | 系统安装与升级 |
| L2 | 配置 | sysctl + `/etc/agentrt/` + A-UCS JSON/TOML | 内核与用户态配置 |
| L3 | 监控 | ftrace + eBPF + perf + Prometheus + Grafana | 运行时状态 |
| L4 | 告警 | Alertmanager + 自定义规则 | 异常检测与通知 |
| L5 | 故障恢复 | systemd restart + Watchdog + Macro-Supervisor 裁决 | 自动恢复 |
| **L6** | **日志收集** | **A-ULP Ring Buffer + Logger Daemon + Loki** | **日志聚合** |
| **L7** | **Agent 运维** | **agentrt-linux 专属** | **Agent 健康与 Token** |
| **L8** | **记忆运维** | **agentrt-linux 专属** | **记忆卷载维护** |

### 1.3 agentrt-linux 扩展

- **Agent 健康检查**：通过 `airy_health_check()` API + systemd watchdog 监控 Agent 进程健康
- **Token 预算监控**：实时告警 Token 消耗超预算
- **记忆卷载运维**：L1→L2→L3→L4 记忆演化运维与故障恢复
- **12 daemon 统一管理**：所有 daemon 归属 `services/daemons/`，systemd 服务名统一 `agentrt-*.service` 格式

---

## 2. 技术选型声明

agentrt-linux v1.0 运维体系在内核调度、IPC 传输、安全钩子、内存分配与同源代码共享五个维度遵循 [AirymaxOS 总览](../README.md) §2 的不可妥协基线。运维体系通过 systemd 单元、监控规则、日志收集策略在运行期守护五大选型不被偏离。五个维度的选型在本目录的具体落地如下：

| # | 技术维度 | 选定方案 | 明确不采用的方案 | 在本目录的落地 |
|---|---------|---------|----------------|--------------|
| 1 | **内核调度** | **sched_tac**：复用 Linux 6.6 原生 `SCHED_DEADLINE` / `SCHED_FIFO` / `EEVDF` 调度类 | **不使用 sched_ext**（不引入 eBPF 调度器、不使用 `SCHED_EXT=7` 调度类） | `sched_d` daemon 通过 `nice` / `sched_setattr` / `sched_setaffinity` 注入sched_tac 策略；监控规则断言 sched_ext 未启用 |
| 2 | **IPC 零拷贝** | **IORING_OP_URING_CMD**：通过 io_uring 命令操作码实现内核↔用户态零拷贝传输 | **不使用 page flipping**（不交换物理页、不破坏内存布局稳定性） | `gateway_d` daemon 路由 io_uring 命令；监控 IPC fastpath 延迟（~160ns SLA）；告警 page flipping 路径误启用 |
| 3 | **安全钩子** | **纯 C LSM**：以纯 C 实现的 `airy_lsm` 通过 `security_hook_list` 注册 | **不使用 BPF LSM**（不依赖 BPF LSM 框架、不通过 eBPF 程序挂载安全钩子） | `sec_d` daemon 同步纯 C LSM 策略；监控 capability 校验命中率；告警 BPF LSM 误启用 |
| 4 | **内存分配** | **alloc_pages + mmap**：通过 `alloc_pages` 分配物理页后 `vm_map_pages` / `remap_pfn_range` 映射 | **不使用 DMA 一致性内存**（不调用 `dma_alloc_coherent`、不依赖硬件一致性缓存） | `logger_d` 通过 mmap 消费 Ring Buffer（`alloc_pages` 分配）；监控共享内存路径；告警 `dma_alloc_coherent` 误调用 |
| 5 | **同源代码共享** | **IRON-9 v3 四层模型**：[SC] 共享契约层 + [SS] 语义同源层 + [IND] 独立实现层 + [DSL] 降级生存层 | （v2 三层模型升级为 v3 四层模型，新增 [DSL] 降级生存层） | 12 daemon 与 agentrt 同源；运维配置 `/etc/agentrt/` 与 agentrt 同源；CI 强制校验 [SC] 头文件逐字节一致 |

### 2.1 IRON-9 v3 四层模型在运维体系的归属

| 技术点 | [SC] | [SS] | [IND] | [DSL] | 落地文档 |
|--------|:----:|:----:|:-----:|:-----:|---------|
| 12 daemon systemd 单元 | — | — | ● | — | [01-deployment.md](01-deployment.md) |
| A-UCS 配置热重载（sysctl ↔ JSON） | — | ● | — | — | [02-configuration.md](02-configuration.md) |
| `/etc/agentrt/` 同源配置 | — | ● | — | — | [02-configuration.md](02-configuration.md) |
| Panic 日志保活（[DSL]） | — | — | — | ● | [01-deployment.md](01-deployment.md) |
| [SC] 配置项固化 | ● | — | — | — | [02-configuration.md](02-configuration.md) |

---

## 3. 文档索引

本目录现有文档（v1.0 范围）：

```
100-operations/
├── README.md                       # 本文件（v1.0）
├── 01-deployment.md                # 部署体系（systemd + RPM/dnf + ISO + 12 daemon 清单）
└── 02-configuration.md             # 配置管理（sysctl + /etc/agentrt/ + A-UCS JSON/TOML 热重载）
```

| # | 文档 | 版本 | 内容概要 |
|---|------|------|---------|
| — | [README.md](README.md) | v1.0 | 运维体系主索引（本文件，含 12 daemon 清单） |
| 1 | [01-deployment.md](01-deployment.md) | v1.0 | 部署体系、systemd 集成、RPM/dnf 包管理、ISO 安装、12 daemon systemd 单元、Panic 日志保活（[DSL]） |
| 2 | [02-configuration.md](02-configuration.md) | v1.0 | 配置管理、sysctl 内核参数、`/etc/agentrt/` 同源配置、A-UCS JSON/TOML 双向热重载、[SC] 配置项固化 |

### 3.1 后续规划文档（1.0.1 版本）

以下文档在 1.0.1 版本完成，不在 v1.0 范围内：

- `03-monitoring.md`：监控体系（Prometheus + Grafana + sched_tac 调度器状态）
- `04-alerting.md`：告警体系（Alertmanager + Micro-Supervisor/Macro-Supervisor 事件规则）
- `05-incident-response.md`：故障响应（runbook + 应急预案）
- `06-backup-recovery.md`：备份恢复
- `07-systemd-integration.md`：systemd 集成详解（12 daemon 单元规范）
- `08-agent-operations.md`：agentrt-linux 专属 Agent 运维
- `09-memory-operations.md`：agentrt-linux 专属记忆卷载运维
- `10-devstation.md`：DevStation 开发运维环境

---

## 4. Airymax Unify Design 映射

本目录与 Airymax Unify Design 五模块（A-UEF/A-ULP/A-UCS/A-ULS/A-IPC）的关系详见 [AirymaxOS 总览](../README.md) §5。运维体系主要承载 **A-UCS**（配置管理）、**A-ULP**（日志收集）、**A-ULS**（监控告警），其余两模块为辅助关系。

| Unify 模块 | 关系 | 在本目录的体现 |
|-----------|------|--------------|
| **A-UEF** | 辅助 | A-UEF 的 Fault 码触发告警（`AIRY_FAULT_*`）；错误码分布监控仪表盘 |
| **A-ULP** | **核心** | 日志收集——`logger_d` 消费 Ring Buffer、Loki 日志聚合、Panic 回退到 `printk_safe` 的日志保活；128B 固定记录格式运维 |
| **A-UCS** | **核心** | 配置管理——`config_d` 同步 sysctl ↔ JSON 双向热重载、`/etc/agentrt/` 同源配置、[SC] 配置项固化、`AIRY_CONFIG_VERSION` 版本校验 |
| **A-ULS** | **核心** | 监控告警——`macro_d` / `sched_d` / `vfs_d` / `sec_d` 的 systemd 单元监控、sched_tac 调度器状态告警、Micro-Supervisor 冷酷执法事件告警、Macro-Supervisor 温情裁决事件告警、IPC 队列冻结告警 |
| **A-IPC** | 辅助 | `gateway_d` / `net_d` / `dev_d` 的 IPC 网关与设备 DMA 守护监控；IPC fastpath 延迟告警 |

### 4.1 Unify Design 权威源引用

- A-UCS 总纲：[../10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) §6
- A-ULP 总纲：[../10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) §5
- A-ULS 总纲：[../10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md) §7
- A-UCS 模块设计：[../20-modules/11-unified-config.md](../20-modules/11-unified-config.md)
- Logger Daemon 模块：[../20-modules/12-logger-daemon-module.md](../20-modules/12-logger-daemon-module.md)
- Kernel Agent Supervisor：[../20-modules/09-kernel-agent-supervisor.md](../20-modules/09-kernel-agent-supervisor.md)
- User Supervisor Daemon：[../20-modules/10-user-supervisor-daemon.md](../20-modules/10-user-supervisor-daemon.md)

---

## 5. 相关文档

- [AirymaxOS 总览](../README.md)：v1.0 技术选型声明 + 20 子目录索引
- [架构设计层](../10-architecture/README.md)：Unify Design 总纲 + IRON-9 v3 四层模型
- [10-architecture/10-unify-design.md](../10-architecture/10-unify-design.md)：Airymax Unify Design 总纲（SSoT）
- [10-architecture/06-iron9-shared-model.md](../10-architecture/06-iron9-shared-model.md)：IRON-9 v3 四层模型（SSoT）
- [10-architecture/11-degraded-survival-layer.md](../10-architecture/11-degraded-survival-layer.md)：[DSL] 降级生存层（Panic 日志保活依据）
- [20-modules/02-services.md](../20-modules/02-services.md)：services 子仓设计（12 daemons）
- [20-modules/07-system.md](../20-modules/07-system.md)：system 子仓设计（包管理 + 配置 + DevStation）
- [20-modules/09-kernel-agent-supervisor.md](../20-modules/09-kernel-agent-supervisor.md)：Micro-Supervisor（内核冷酷执法）
- [20-modules/10-user-supervisor-daemon.md](../20-modules/10-user-supervisor-daemon.md)：Macro-Supervisor（用户温情裁决）
- [20-modules/11-unified-config.md](../20-modules/11-unified-config.md)：A-UCS 统一配置管理体系
- [20-modules/12-logger-daemon-module.md](../20-modules/12-logger-daemon-module.md)：Logger Daemon 模块
- [70-build-system/README.md](../70-build-system/README.md)：构建系统（airy_defconfig + RPM spec）
- [90-observability/README.md](../90-observability/README.md)：可观测性（监控数据源）
- [110-security/README.md](../110-security/README.md)：安全运维（纯 C LSM 策略同步）
- [190-distribution/01-rpm-packaging.md](../190-distribution/01-rpm-packaging.md)：RPM 打包规范

---

## 6. 参考材料

- Linux 6.6 systemd 集成（`systemd.unit` / `systemd.service` 手册）
- Linux 发行版包管理（RPM + dnf）
- Linux 6.6 sysctl 与配置管理（`/etc/sysctl.d/`）
- Prometheus + Grafana 监控栈
- Alertmanager 告警路由
- Loki 日志聚合系统

---

## 7. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 0.1.1 | 2026-07-13 | 初始版本，README + 01 + 02 文档奠基，确立部署/配置核心机制 |
| v1.0 | 2026-07-17 | 升级为 v1.0：新增sched_tac / IORING_OP_URING_CMD / 纯 C LSM / alloc_pages + mmap / IRON-9 v3 四层模型五大技术选型声明；新增 Airymax Unify Design 映射（A-UCS 配置管理 + A-ULP 日志收集 + A-ULS 监控告警为核心）；新增 12 daemon 清单（统一归属 services/daemons/，systemd 服务名 `agentrt-*.service` 格式）；文档索引对齐实际目录文件 |

---

> **文档结束** | agentrt-linux 运维体系设计 v1.0 | 维护者：开源极境工程与规范委员会 | "From data intelligence emerges."
