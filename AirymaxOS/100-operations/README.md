Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）运维体系设计

> **文档定位**： agentrt-linux（AirymaxOS）运维工程体系主索引
> **版本**： 0.1.1
> **最后更新**： 2026-07-13
> **同源映射**： agentrt daemons（12 个用户态服务）+ Linux 6.6 systemd 集成
> **理论根基**： Linux 发行版运维哲学 + Airymax S-1 反馈闭环 + E-2 可观测性

---

## 1. 模块定位

agentrt-linux 运维体系是系统稳定运行的核心保障。它继承 Linux 发行版 30+ 年沉淀的运维哲学（systemd 集成 + 包管理 + 配置管理 + 监控告警 + 故障恢复），并在其上扩展智能体操作系统专属的 Agent 运维（Agent 健康检查、Token 预算监控、记忆卷载运维等）。

### 1.1 运维体系分层

| 层级 | 类型 | 工具/机制 | 范围 |
|------|------|---------|------|
| L1 | 部署 | systemd + 包管理（RPM/dnf） | 系统安装与升级 |
| L2 | 配置 | sysctl + 配置文件 + 配置管理工具 | 内核与用户态配置 |
| L3 | 监控 | ftrace + eBPF + perf + 监控代理 | 运行时状态 |
| L4 | 告警 | Alertmanager + 自定义规则 | 异常检测与通知 |
| L5 | 故障恢复 | systemd restart + 故障转移 | 自动恢复 |
| **L6** | **Agent 运维** | **agentrt-linux 专属** | **Agent 健康与 Token** |
| **L7** | **记忆运维** | **agentrt-linux 专属** | **记忆卷载维护** |

### 1.2 agentrt-linux 扩展

- **Agent 健康检查**：通过 systemd watchdog 监控 Agent 进程健康
- **Token 预算监控**：实时告警 Token 消耗超预算
- **记忆卷载运维**：L1→L2→L3→L4 记忆演化运维与故障恢复

---

## 2. 核心运维机制

### 2.1 systemd 集成

agentrt-linux 12 个 daemons 与 systemd 集成：
```ini
# /etc/systemd/system/agentrt-gateway.service
[Unit]
Description=agentrt-linux Gateway Daemon
After=network.target

[Service]
Type=simple
ExecStart=/usr/lib/agentrt/gateway_d
Restart=on-failure
WatchdogSec=30

[Install]
WantedBy=multi-user.target
```

### 2.2 包管理（RPM + dnf）

agentrt-linux 采用 RPM 包格式 + dnf 包管理器：
- 内核包：`airymaxos-kernel`
- 服务包：`airymaxos-services-*`
- SDK 包：`airymaxos-sdk-*`

### 2.3 配置管理

- **sysctl**：内核运行时参数（`/etc/sysctl.d/`）
- **配置文件**：`/etc/agentrt/`（与 agentrt 同源）
- **配置管理工具**：Ansible / Puppet（可选）

### 2.4 监控告警

- **Prometheus + Grafana**：指标采集与可视化
- **Alertmanager**：告警路由与通知
- **Loki**：日志聚合

### 2.5 故障恢复

- **systemd restart**：进程崩溃自动重启
- **Watchdog**：硬件看门狗 + 软件看门狗
- **故障转移**：高可用场景下的主备切换

---

## 3. 文档索引

```
100-operations/
├── README.md                       # 本文件
├── 01-deployment.md                # 部署体系（RPM + dnf + ISO）
├── 02-configuration.md             # 配置管理（sysctl + 配置文件）
├── 03-monitoring.md                # 监控体系（Prometheus + Grafana）
├── 04-alerting.md                  # 告警体系（Alertmanager + 规则）
├── 05-incident-response.md         # 故障响应（runbook + 应急预案）
├── 06-backup-recovery.md           # 备份恢复
├── 07-systemd-integration.md       # systemd 集成
├── 08-agent-operations.md          # agentrt-linux 专属：Agent 运维
├── 09-memory-operations.md         # agentrt-linux 专属：记忆卷载运维
└── 10-devstation.md                # DevStation 开发运维环境
```

### 3.1 0.1.1 版本范围

README + 01 + 02 文档（3 文档奠基），确立运维体系设计框架与部署/配置核心机制。其余 8 文档（03-monitoring 至 10-devstation）在 1.0.1 版本完成。

### 3.2 1.0.1 版本范围

完成剩余 8 文档（03-monitoring 至 10-devstation），实施运维工程标准，进行代码级验证。

---

## 4. agentrt-linux 专属扩展

### 4.1 Agent 运维

- **Agent 健康检查**：通过 `airy_health_check()` API + systemd watchdog
- **Token 预算监控**：实时告警 Token 消耗超预算
- **Agent 故障恢复**：Agent 崩溃后自动重启 + 状态恢复

### 4.2 记忆卷载运维

- **L1 原始卷**：定期清理过期数据
- **L2 特征层**：特征更新与版本管理
- **L3 结构层**：结构一致性检查
- **L4 模式层**：模式挖掘任务监控

### 4.3 同源 agentrt 运维

agentrt 的 12 个 daemons（gateway_d/llm_d/tool_d/sched_d/market_d/monit_d/channel_d/info_d/notify_d/observe_d/hook_d/plugin_d）在 agentrt-linux 中以 systemd 服务运行：
- 进程二进制名保持 `*_d` 后缀不变
- systemd 服务名采用 `agentrt-*.service` 格式
- 配置文件统一在 `/etc/agentrt/`（与 agentrt 同源）

---

## 5. 五维原则映射

| 原则 | 在本模块的体现 |
|------|---------------|
| **S-1 反馈闭环** | 监控告警 → 故障恢复 → 状态反馈 |
| **E-2 可观测性** | 全栈监控 |
| **E-6 错误可追溯** | 故障 runbook + 事后分析 |
| **IRON-9 v2 同源且部分代码共享** | 与 agentrt daemons 同源 |
| **A-3 人文关怀** | DevStation 提升开发者体验 |

---

## 6. 相关文档

- `50-engineering-standards/06-toolchain-and-automation.md`（运维自动化）
- `90-observability/README.md`（可观测性）
- `110-security/README.md`（安全运维）
- `20-modules/02-services.md`（services 子仓设计）
- `20-modules/07-system.md`（system 子仓设计）

---

## 7. 参考材料

- Linux 6.6 systemd 集成
- Linux 发行版包管理（RPM + dnf）
- Linux 6.6 sysctl 与配置管理
- Prometheus + Grafana 监控栈

---

> **文档结束** | README + 01 + 02
