# ARE Standards — Agent Runtime Environment Standards

> **版本**: v0.1.0-draft | **状态**: 草案（v0.1.1 发行前冻结）| **日期**: 2026-07-04
> **版权归属人**: SPHARX Ltd. | **许可证**: AGPL-3.0-or-later OR Apache-2.0（双许可证，SPHARX Ltd.）
>
> **战略定位**: ARE Standards 是 Airymax 推动的"行业底座"开放标准体系，目标是定义 Agent Runtime Environment 的三层标准接口，使第三方可独立实现兼容的 Agent 运行时。

---

## 1. 标准分层

| 层级 | 标准名称 | 范围 | 目标 |
|------|---------|------|------|
| **L1** | 核心运行时接口 | atoms 层抽象接口、生命周期、内存模型 | 任何实现者可提供兼容的微核心 |
| **L2** | 服务通信协议 | IPC Bus 消息头、JSON-RPC 命名空间、服务发现 | 第三方可独立实现兼容 daemon |
| **L3** | 安全与治理 | Cupolas 权限引擎、错误码体系、审计日志 | 跨实现的安全行为可互验 |

## 2. 标准文档清单

| 文档 | 描述 | 状态 |
|------|------|------|
| [L1_runtime_interface.md](L1_runtime_interface.md) | L1 核心运行时接口规范 | 草案 |
| [L2_service_protocol.md](L2_service_protocol.md) | L2 服务通信协议规范 | 草案 |
| [L3_security_governance.md](L3_security_governance.md) | L3 安全与治理规范 | 草案 |

## 3. 独立仓库可行性

**结论**: 高度可行。
- `Docs/` 本身已是独立 git submodule（`git@atomgit.com:openairymax/docs.git`）
- `Capital_Specifications/are_standards/` 作为标准容器，可独立发布
- 未来可拆分为独立 `are-standards` 仓库（`git@atomgit.com:openairymax/are-standards.git`）
- 参考实现保留在 Airymax 主仓库，标准文档独立发布

## 4. 标准化路线图

| 阶段 | 时间 | 目标 |
|------|------|------|
| 草案 | v0.1.1 发行前 | 三层标准文档完成，标注 draft |
| 试用 | v0.2.0 | 第三方实现反馈，迭代修订 |
| 候选 | v0.3.0 | 标准冻结，提供一致性测试套件 |
| 正式 | v1.0.0 | 标准正式发布，版本化治理 |

---

<div align="center">

**© 2025-2026 SPHARX Ltd.** | ARE Standards 是 SPHARX Ltd. 推动的开放标准

</div>
