Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# Airymax Integration Standards / Airymax 集成标准集

**最新**: 2026-06-09
**状态**: 维护中
**路径**: OpenAirymax/Docs/50-specifications/50-integration/README.md
---

## 📋 目录 / Table of Contents

本目录包含 Airymax 各模块间的集成标准和最佳实践。

This directory contains integration standards and best practices between Airymax modules.

### 已发布的集成标准 / Published Integration Standards

| 标准名称 | 版本 | 适用模块 | 最后更新 | 状态 |
|---------|------|---------|---------|------|
| [Manager Config Integration Standard](./INTEGRATION_STANDARD.md) | v1.0.0 | manager ↔ agentos/commons/utils/config_unified | 2026-04-01 | ✅ Production Ready |

### 计划中的标准 / Planned Standards

| 标准名称 | 目标模块 | 计划发布日期 | 状态 |
|---------|---------|-------------|------|
| Gateway Integration Standard | gateway ↔ agentos/daemon/* | TBD | 🔨 In Progress |
| MemoryRovol Integration Standard | memoryrovol ↔ coreloopthree | TBD | 📋 Planning |
| Security Dome Integration Standard | cupolas ↔ all modules | TBD | 📋 Planning |

> **📝 状态说明**  
> - **🔨 In Progress**: Gateway 集成标准正在制定中，基于 `agentos/gateway/` 模块的 HTTP 网关和 MCP/A2A 协议适配器实现
> - **📋 Planning**: MemoryRovol 和 Security Dome 集成标准尚在规划阶段，待对应模块 API 稳定后启动制定

### 新标准模板 / Standard Template

创建新的集成标准时，应遵循以下模板结构：

```markdown
# {Module} 模块与 {Target} 集成标准

**版本**: v1.0.0  
**最后更新**: {日期}  
**适用范围**: {模块} 与 {目标模块} 的集成  
**理论基础**: 工程两论（接口契约化）+ 五维正交设计 + Thinkdual 双思考系统  

## 一、集成概述
### 1.1 集成目标
### 1.2 架构关系

## 二、接口定义
### 2.1 数据结构
### 2.2 API 规范
### 2.3 错误码映射

## 三、配置与初始化
### 3.1 环境变量
### 3.2 配置文件

## 四、测试覆盖要求
### 4.1 单元测试
### 4.2 集成测试

## 五、最佳实践清单

## 六、故障排除

## 七、版本历史

## 八、参考文档
```

详细贡献指南参见 [standards_contribution.md](./standards_contribution.md)。

---

## 🎯 目标 / Objectives

建立 Airymax 模块间集成的标准化机制，确保：

1. **统一接口契约**: 所有模块间调用遵循一致的接口定义
2. **配置管理标准化**: 统一的配置加载、验证和热更新机制
3. **错误处理一致性**: 跨模块的错误传播和处理规范
4. **测试覆盖要求**: 集成点的测试策略和覆盖率标准
5. **向后兼容性**: 保证模块升级时的兼容性

---

## 📐 架构原则对齐 / Architecture Principles Alignment

基于 **00-architectural-principles.md V2.0** 五维正交系统。

### 应用的核心原则 / Key Principles Applied

| 维度 | 原则 | 应用场景 |
|------|------|---------|
| **内核观 K-2** | 接口契约化 | 所有集成点必须明确定义接口契约 |
| **工程观 E-4** | 跨平台一致性 | 集成标准在 Linux/macOS/Windows 一致 |
| **工程观 E-8** | 可测试性 | 每个集成点必须有对应的测试策略 |

---

## 🔧 使用指南 / Usage Guide

### 开发者 / For Developers

1. 在开发新模块集成前，查阅对应的集成标准
2. 遵循标准中定义的 API 使用规范
3. 实施标准要求的测试覆盖率
4. 更新相关文档以反映集成变更

### 审查者 / For Reviewers

1. 检查代码是否符合集成标准的所有条款
2. 验证错误处理是否遵循标准规范
3. 确认测试覆盖率达到标准要求
4. 评估向后兼容性影响

---

## 📝 贡献指南 / Contributing

### 提交新的集成标准

1. 在本目录创建新的 Markdown 文件
2. 遵循现有的文档格式和结构
3. 包含完整的示例代码和测试用例
4. 更新本文档的目录表

### 格式要求

- 文件命名：`{Module}_INTEGRATION_STANDARD.md`
- 编码：UTF-8，无 BOM
- 语言：中英双文（优先中文，关键术语附英文）
- 版本格式：v{Major}.{Minor}.{Patch}

---

## 📚 相关文档 / Related Documents

- [00-architectural-principles.md](../00-architectural-principles.md) - 架构设计原则
- [10-contracts/](../10-contracts/) - 接口合同规范
- [30-coding-standard/](../30-coding-standard/) - 编码标准
- [70-project-erp/](../70-project-erp/) - 项目企业资源规划

---

## 📞 联系方式 / Contact

- **问题反馈**: [Gitee Issues](https://gitee.com/spharx/agentos/issues)
- **安全 issues**: security@spharx.com
- **技术讨论**: 查看 CONTRIBUTING.md

---

> *"From data intelligence emerges."*  
> *始于数据，终于智能。*

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
