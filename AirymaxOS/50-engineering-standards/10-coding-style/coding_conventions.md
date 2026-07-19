Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 编码约定规范
> **文档定位**：代码注释、配置审计、日志、命名约定与安全设计规范合集\
> **文档版本**：0.1.1\
> **最后更新**：2026-07-12\
> **上级文档**：[agentrt-linux（AirymaxOS）工程标准规范](README.md)

---

## Part I: Airymax 代码注释与合规规范

### 一、概述

#### 1.1 编制目的

代码注释是软件工程中不可或缺的组成部分，是代码可读性、可维护性、合规性和协作效率的关键保障。本文档为 Airymax 项目提供统一的代码注释与合规规范，旨在：

1. **统一注释风格**：确保整个项目注释格式一致，降低认知负担
2. **保障代码合规**：规范版权声明、许可证标识、第三方代码引用，确保法律合规性
3. **提升代码质量**：通过规范化的注释促进开发者深入思考代码设计
4. **加速团队协作**：清晰的注释使代码意图一目了然，减少沟通成本
5. **支持自动化工具**：规范的注释可被 Doxygen、Sphinx、TypeDoc 等工具解析生成文档
6. **确保开源合规**：遵循 SPDX 标准，保障项目开源过程的合法性与透明性

#### 1.2 核心理念与架构映射

Airymax 的代码注释规范建立在"工程两论"、五维正交系统和现代合规框架的基础之上，体现了以下核心设计理念：

##### 1.2.1 基于工程控制论的注释理念
- **反馈闭环设计**：注释构成代码的"自解释系统"，为开发者提供实时的设计意图反馈，形成"编写-注释-理解"的认知闭环
- **稳定性保障**：公共 API 注释作为系统接口的"控制契约"，确保模块间交互的稳定性和可预测性（映射原则：S-3接口稳定性）
- **前馈预测性设计**：基于历史注释模式的趋势预测，提前调整注释策略和内容结构

##### 1.2.2 基于系统工程的注释理念  
- **层次分解方法**：注释体系遵循五维正交系统，从系统观（架构意图）到工程观（实现细节）提供不同层次的解释
- **模块化文档**：注释与代码结构严格对应，支持"按需加载"的文档认知模式（映射原则：S-2模块化设计）
- **总体设计部**：注释作为代码的总体设计部，只做解释说明、不做具体实现

##### 1.2.3 基于 Thinkdual 双思考系统的注释理念
- **t1 快思考 注释**：简单接口的注释应直观易懂，支持快速认知（快速路径），如简单的 getter/setter
- **t2 慢思考 注释**：复杂算法的注释需提供深度原理说明，支持深度思考（慢速路径），如核心算法实现

##### 1.2.4 基于设计美学的注释理念
- **极简主义**（A-1原则）：注释应简洁精炼，避免冗余信息污染代码空间
- **细节关注**（A-2原则）：关键设计决策必须在注释中明确记录，传承工程智慧
- **人文关怀**（A-3原则）：注释应设身处地为读者着想，提供必要的上下文和背景
- **完美主义**（A-4原则）：注释质量应与代码质量同等要求，追求文档的完美无瑕

##### 1.2.5 基于合规框架的注释理念
- **版权清晰**：每个源文件必须明确版权归属，遵循 SPDX 标准
- **许可证透明**：许可证标识必须准确、完整，避免兼容性问题
- **第三方合规**：引入第三方代码必须保留原始声明，明确修改记录
- **风险控制**：安全敏感代码必须标注安全等级和风险提示

##### 1.2.6 注释的多重角色
- **注释即认知桥梁**：连接代码实现与设计意图，降低认知负荷
- **注释即知识传承**：记录技术决策背后的思考过程，避免知识丢失
- **注释即质量指标**：注释完整度直接反映代码的工程成熟度
- **注释即协作协议**：为团队协作提供统一的沟通语言和标准
- **注释即合规凭证**：证明代码的合法来源和授权状态

#### 1.3 适用范围

| 语言 | 注释风格 | 文档生成工具 | 合规检查工具 |
|------|----------|--------------|--------------|
| C/C++ | Doxygen + SPDX | Doxygen | clang-tidy, REUSE |
| Python | Google/NumPy Docstring + SPDX | Sphinx | pydocstyle, licensecheck |
| JavaScript/TypeScript | JSDoc/TSDoc + SPDX | TypeDoc | ESLint, license-checker |
| Go | GoDoc + SPDX | go doc | go-licenses, govulncheck |
| Rust | Rustdoc + SPDX | cargo doc | cargo-audit, cargo-deny |
| 配置/数据文件 | 语言特定格式 | - | 相应格式检查工具 |

---

### 二、代码合规注释规范

基于《代码合规建议.md》（总线审查版）的核心要求，本章节系统规定代码注释中必须包含的合规信息，确保项目符合开源合规、安全合规等法律法规要求。

#### 2.1 文件头部合规注释

所有自研源代码文件必须在文件头部添加版权和许可证声明，遵循 SPDX（Software Package Data Exchange）标准，这是开源项目合规性的基础要求。

##### 2.1.1 自研代码文件头部声明格式

**C/C++ 文件示例**：
```c
/*
 * Copyright (C) 2025-2026 SPHARX Ltd. All Rights Reserved.
 * SPDX-FileCopyrightText: 2025-2026 SPHARX Ltd.
 * SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
 * 
 * @file memory_manager.h
 * @brief Airymax 内存管理器接口
 * 
 * 提供内核级内存分配、释放、监控功能。采用 slab 分配器
 * 优化小对象分配性能，支持 NUMA 感知内存布局。
 * 
 * @author SPHARX Ltd. - Airymax Team
 * @date 2026-03-30
 * @version 2.0
 * 
 * @note 线程安全：所有公共接口均为线程安全
 * @see agentrt/atoms/corekern/memory/
 */

#ifndef AIRY_MEMORY_MANAGER_H
#define AIRY_MEMORY_MANAGER_H

#include "airy_types.h"

#ifdef __cplusplus
extern "C" {
#endif
```

**Python 文件示例**：
```python
"""
Copyright (C) 2025-2026 SPHARX Ltd. All Rights Reserved.
SPDX-FileCopyrightText: 2025-2026 SPHARX Ltd.
SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0

Airymax 任务调度模块

本模块提供任务调度核心功能，包括任务提交、状态管理、
优先级队列和依赖解析。调度器采用双思考系统架构：
- t1 快思考：快速路径，处理简单任务
- t2 慢思考：深度路径，处理复杂任务

典型用法:
    from agentrt.scheduler import TaskScheduler
    
    scheduler = TaskScheduler()
    task_id = scheduler.submit(task_plan)
    result = scheduler.wait(task_id)

版本: 2.0.0
作者: SPHARX Ltd. - Airymax Team
日期: 2026-03-30
"""

from typing import Optional, List, Dict, Any
```

**TypeScript/JavaScript 文件示例**：
```typescript
/**
 * Copyright (C) 2025-2026 SPHARX Ltd. All Rights Reserved.
 * SPDX-FileCopyrightText: 2025-2026 SPHARX Ltd.
 * SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
 * 
 * @fileoverview Airymax 任务调度模块
 * 
 * 提供任务调度核心功能，包括任务提交、状态管理、
 * 优先级队列和依赖解析。
 * 
 * @module agentrt/scheduler
 * @author SPHARX Ltd. - Airymax Team
 * @version 2.0.0
 * @date 2026-03-30
 */

import { TaskPlan, TaskResult, TaskStatus } from './types';
```

##### 2.1.2 声明格式规范

1. **版权声明格式**：
   - **标准格式**：`Copyright (C) [第一次发布年份]-[末次修改年份] [版权所有者名称]`
   - **SPDX格式**：`SPDX-FileCopyrightText: [第一次发布年份]-[末次修改年份] [版权所有者名称]`
   - **示例**：`SPDX-FileCopyrightText: 2025-2026 SPHARX Ltd.`

2. **许可证标识格式**：
   - **标准格式**：`SPDX-License-Identifier: [许可证标识符]`
   - **常见标识符**：`Apache-2.0`、`MIT`、`GPL-3.0-or-later`、`BSD-3-Clause`
   - **多许可证**：`SPDX-License-Identifier: (Apache-2.0 OR MIT)`

3. **声明位置要求**：
   - 必须位于文件头部，在任何其他内容之前
   - 对于支持多行注释的语言，使用多行注释格式
   - 对于不支持多行注释的文件格式（如JSON），在单行注释中添加

##### 2.1.3 特殊文件处理

1. **配置文件（JSON/YAML/XML）**：
   ```json
   {
     "_comment": "SPDX-FileCopyrightText: 2025-2026 SPHARX Ltd.",
     "_license": "SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0",
     "name": "agentrt-config",
     "version": "2.0.0"
   }
   ```

2. **脚本文件（Shell/Batch）**：
   ```bash
   #!/bin/bash
   # SPDX-FileCopyrightText: 2025-2026 SPHARX Ltd.
   # SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
   #
   # Airymax 系统初始化脚本
   # 版本: 2.0.0
   # 日期: 2026-03-30
   ```

#### 2.2 第三方代码合规注释

引入第三方开源代码时，必须严格遵守合规要求，保留原始版权和许可证声明。

##### 2.2.1 原样引入第三方代码

```c
/*
 * 以下代码来自第三方开源项目 [项目名称]，版本 [版本号]
 * 原版权声明保留如下：
 * 
 * Copyright (C) 2010-2024 [原作者/组织]
 * SPDX-FileCopyrightText: 2010-2024 [原作者/组织]
 * SPDX-License-Identifier: [原许可证标识]
 * 
 * 此部分代码按原样引入，未作任何修改。
 * 原代码仓库: [仓库URL]
 * 引入日期: 2026-03-30
 * 引入原因: [简要说明原因]
 */
```

##### 2.2.2 修改后引入第三方代码

```c
/*
 * 以下代码基于第三方开源项目 [项目名称] 修改而来
 * 原版权声明保留如下：
 * 
 * Copyright (C) 2010-2024 [原作者/组织]
 * SPDX-FileCopyrightText: 2010-2024 [原作者/组织]
 * SPDX-License-Identifier: [原许可证标识]
 * 
 * 修改声明:
 * This file was modified by SPHARX Ltd. on 2026-03-30.
 * 主要修改内容:
 * 1. 添加了 [功能A] 支持
 * 2. 优化了 [性能B]
 * 3. 修复了 [问题C]
 * 
 * 原代码仓库: [仓库URL]
 * 修改依据: [需求文档/Issue链接]
 */
```

##### 2.2.3 第三方代码合规检查清单

| 检查项 | 要求 | 验证方法 |
|--------|------|----------|
| 版权声明 | 必须原样保留 | 比对原始文件 |
| 许可证标识 | 必须原样保留 | 检查SPDX标识 |
| 修改声明 | 如有修改必须添加 | 检查修改记录 |
| 许可证兼容性 | 与项目许可证兼容 | 使用许可证兼容性矩阵 |
| 引入记录 | 记录引入原因和日期 | 检查版本控制日志 |

#### 2.3 许可证兼容性注释

对于涉及多个许可证的代码，必须在注释中明确说明兼容性，确保法律合规。

```c
/*
 * 许可证兼容性说明：
 * 
 * 本文件包含以下许可证代码：
 * 1. 主要部分: AGPL-3.0-or-later OR Apache-2.0 (Airymax 自研代码，双许可证)
 * 2. 算法库部分: BSD-3-Clause (第三方库 [库名])
 * 3. 工具函数: MIT (第三方库 [库名])
 * 
 * 兼容性分析：
 * - Apache-2.0 与 BSD-3-Clause 兼容 
 * - Apache-2.0 与 MIT 兼容 
 * - 组合许可证: Apache-2.0 AND BSD-3-Clause AND MIT
 * 
 * 使用限制：
 * - 必须保留所有版权声明
 * - 必须提供原始许可证文本副本
 * - 修改必须标注修改记录
 */
```

#### 2.4 安全合规注释

安全敏感代码必须添加安全合规注释，明确安全等级、威胁模型和防护措施。

##### 2.4.1 密码学相关代码

```c
/**
 * @brief AES-256-GCM 加密函数
 * 
 * 使用硬件加速的 AES-GCM 模式进行数据加密。
 * 符合 NIST SP 800-38D 标准，提供认证加密。
 * 
 * @security 安全等级: CRITICAL
 * @security 密钥管理: 使用内核安全密钥环存储
 * @security 随机数生成: 使用硬件 RDRAND 指令
 * @security 侧信道防护: 恒定时间实现，抗时序攻击
 * 
 * @compliance 符合标准:
 *   - FIPS 140-3 Level 2
 *   - PCI DSS v4.0
 *   - GDPR 第32条（安全处理）
 * 
 * @audit 审计记录:
 *   - 2026-03-15: 第三方安全审计通过
 *   - 2026-03-20: 模糊测试覆盖率达到100%
 * 
 * @see 安全设计指南: Security_design_standard.md
 */
```

##### 2.4.2 数据隐私相关代码

```python
def process_user_data(user_data: Dict[str, Any]) -> Dict[str, Any]:
    """处理用户数据，确保符合隐私法规。
    
    对用户数据进行匿名化处理，移除直接标识符，
    应用差分隐私保护敏感信息。
    
    Args:
        user_data: 原始用户数据字典
        
    Returns:
        处理后的匿名化数据
        
    Privacy:
        - 数据分类: PII（个人身份信息）
        - 处理依据: 用户明示同意
        - 保留期限: 30天（符合数据最小化原则）
        - 跨境传输: 使用标准合同条款（SCCs）
        
    Compliance:
        - GDPR: 第5条（合法性、公平性、透明性）
        - CCPA: 第1798.100条（消费者隐私权）
        - PIPL: 第22条（个人信息处理规则）
        
    Security:
        - 加密存储: AES-256 加密
        - 访问控制: RBAC 基于角色的访问控制
        - 审计日志: 完整的数据访问记录
        
    Example:
        >>> processed = process_user_data({"name": "张三", "email": "zhangsan@example.com"})
        >>> print(processed["anonymous_id"])
        "user_9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08"
    """
```

#### 2.5 专利合规注释

涉及第三方专利技术的代码必须添加专利合规注释，明确专利使用状态和合规措施。

##### 2.5.1 专利技术使用声明

```c
/**
 * @brief 基于 LZ4 压缩算法优化的数据压缩函数
 * 
 * 使用改进的 LZ4 压缩算法，针对特定数据类型优化哈希表配置和匹配策略。
 * 相比标准 LZ4，压缩率提升 15-20%，解压速度保持相近水平。
 * 
 * @patent 专利声明:
 *   - 基础算法: LZ4 压缩算法（专利已过期）
 *   - 优化技术: 自研优化，不涉及第三方专利
 *   - 专利检查: 已进行专利前置审查，未发现侵权风险
 *   
 * @patent 第三方专利排查:
 *   1. US Patent 5,838,919 (数据压缩方法) - 已过期 
 *   2. US Patent 6,009,177 (LZ派生算法) - 已过期  
 *   3. CN Patent ZL201110123456.7 (中文优化) - 不相关 ❌
 *   4. EP Patent 1234567 (哈希优化) - 已规避 
 *   
 * @patent 合规措施:
 *   1. 使用已进入公共领域的算法基础
 *   2. 对核心优化技术进行专利检索
 *   3. 建立专利监控机制，定期更新专利状态
 *   4. 提供专利替代方案（如 GZIP 回退）
 *   
 * @warning 专利风险提示:
 *   - 禁止在专利保护期内国家/地区商业使用（如有相关专利）
 *   - 定期检查专利状态变化
 *   - 发现潜在专利冲突时立即通知法务部门
 *   
 * @see 专利合规指南: patent_compliance.md
 * @see LZ4 开源项目: https://github.com/lz4/lz4
 */
int lz4_optimized_compress(const uint8_t* src, uint8_t* dst, int src_size);
```

#### 2.6 商业秘密合规注释

涉及商业秘密或保密技术的代码必须添加商业秘密合规注释，明确保密要求和披露限制。

##### 2.6.1 商业秘密保护声明

```c
/**
 * @brief 自研高性能数据压缩算法
 * 
 * 实现基于改进LZ77算法的高性能数据压缩，采用专有的哈希匹配策略
 * 和滑动窗口优化技术。相比标准DEFLATE算法，压缩率提升15-25%，
 * 解压速度保持相同水平。
 * 
 * @trade_secret 商业秘密声明:
 *   
 *   **核心商业秘密**:
 *   1. 哈希匹配策略: 专利待申请，目前作为商业秘密保护
 *   2. 滑动窗口优化: 专有的内存访问模式和预取算法
 *   3. 编码优化技巧: 熵编码参数动态调整技术
 *   4. 硬件适配方案: 针对不同CPU架构的SIMD指令优化
 *   
 *   **保密等级**: CONFIDENTIAL
 *   **保密期限**: 永久（除非通过专利公开）
 *   **知悉范围**: 核心算法团队、授权合作伙伴
 *   
 * @trade_secret 保护措施:
 *   1. **访问控制**: 仅授权人员可访问源代码
 *   2. **代码混淆**: 关键算法部分使用代码混淆
 *   3. **水印技术**: 在输出中添加数字水印追踪泄露
 *   4. **审计日志**: 完整的代码访问和修改记录
 *   5. **员工协议**: 所有接触人员签署保密协议
 *   
 * @trade_secret 披露限制:
 *   1. 禁止开源分发商业秘密代码
 *   2. 禁止向未授权第三方透露算法细节
 *   3. 技术交流需脱敏处理（移除关键参数）
 *   4. 学术发表需内部技术评审委员会批准
 *   
 * @warning 法律责任:
 *   - 违反保密协议可能面临刑事指控
 *   - 商业秘密泄露可能导致重大经济损失
 *   - 侵权方可能被要求支付惩罚性赔偿
 *   
 * @see 商业秘密保护政策: trade_secret_policy.md
 * @see 员工保密协议范本: templates/nda.md
 */
void high_performance_compress(const uint8_t* input, uint8_t* output, int input_size, int* output_size);
```

---

### 三、通用注释原则

#### 3.1 注释的必要性判断

基于工程控制论的反馈闭环原理，注释应与代码复杂度成正比，形成自适应的注释策略。

**必须注释的场景**：
- **公共 API 接口**：函数、类、模块的对外接口（映射原则：S-3接口稳定性）
- **复杂算法**：非直观逻辑、数学推导、优化技巧（映射原则：C-4认知辅助）
- **性能关键代码**：性能优化、内存管理、并发控制（映射原则：E-6性能工程）
- **安全敏感代码**：身份验证、数据加密、权限检查（映射原则：D-2安全隔离）
- **平台特定代码**：操作系统、硬件架构、编译器相关（映射原则：E-7跨平台）
- **临时解决方案**：TODO、FIXME、HACK等临时代码（映射原则：E-8可测试性）
- **已废弃接口**：DEPRECATED标记的旧接口（映射原则：S-4演化适应）
- **合规相关代码**：版权、许可证等（合规要求）

**无需注释的场景**：
- **简单的 getter/setter**：属性访问方法，代码意图明显
- **标准库调用包装**：对标准库函数的简单包装
- **代码本身已足够清晰**：命名规范、结构清晰的代码
- **自动生成的代码**：由工具生成的代码（需标注生成来源）
- **重复模式代码**：项目中已建立模式的重复代码

#### 3.2 注释内容要素

完整的函数/方法注释应包含以下要素，形成系统化的文档结构：

| 要素 | 必要性 | 说明 | 示例 |
|------|--------|------|------|
| 简要描述 | 必须 | 一句话说明功能，t1 快思考认知 | `分配指定大小的内存块` |
| 详细描述 | 推荐 | 解释设计意图、算法原理，t2 慢思考认知 | `从内核内存池中分配连续内存...` |
| 参数说明 | 必须 | 每个参数的含义、约束、单位 | `@param size 字节数，必须大于0` |
| 返回值说明 | 必须 | 返回值含义、成功/失败条件 | `@return 成功返回指针，失败返回NULL` |
| 异常说明 | 必须 | 可能抛出的异常及触发条件 | `@throws ValueError 当参数无效时` |
| 线程安全 | 推荐 | 是否线程安全、并发使用注意事项 | `@note 线程安全：可多线程调用` |
| 使用示例 | 推荐 | 典型用法代码示例，降低使用门槛 | `@example void* ptr = alloc(1024);` |
| 合规信息 | 条件必须 | 版权、许可证、安全等级等。仅安全敏感函数必须标注（见下方说明） | `@compliance 安全等级: HIGH` |
| 注意事项 | 可选 | 边界条件、性能考量、资源管理 | `@note 调用者负责释放内存` |
| 相关接口 | 可选 | 相关函数、参考文档链接 | `@see airy_mem_free()` |
| 版本历史 | 可选 | 重大修改记录、版本变更 | `@since 2.0.0` |
| 作者信息 | 可选 | 创建者、维护者、联系方式 | `@author SPHARX Ltd.` |

**`@compliance` 适用范围说明**：

`@compliance` 标注仅对以下安全敏感函数**必须**添加，其他函数为可选：

| 必须标注 @compliance 的函数类型 | 示例 |
|--------------------------------|------|
| 输入净化/验证函数 | `sanitize_input()`、`validate_path()` |
| 加密/解密函数 | `crypto_encrypt_symmetric()`、`crypto_decrypt_symmetric()` |
| 身份认证函数 | `session_create()`、`session_validate()` |
| 权限检查函数 | `check_permission()`、`verify_access()` |
| 审计日志函数 | `audit_log()`、`audit_query()` |
| 密钥管理函数 | `key_generate()`、`key_rotate()` |

普通业务逻辑函数（如数据格式化、配置读取、日志输出）无需添加 `@compliance` 标注。

#### 3.3 注释语言规范

- **中文为主**：注释使用规范中文，技术术语保留英文原词
- **Doxygen 标签英文**：Doxygen 标签（`@param`、`@return`、`@brief`、`@note`、`@see`、`@throws`、`@deprecated` 等）使用英文；标签后的描述性文字使用中文
- **术语一致**：使用项目统一术语表中的术语，避免歧义
- **语法规范**：符合现代汉语语法，避免口语化、网络用语
- **避免冗余**：不重复代码已表达的信息
- **合规准确**：版权年份、许可证标识必须准确无误

---

### 四、C/C++ 注释规范（Doxygen + SPDX 风格）

#### 4.1 文件头注释模板

```c
/*
 * Copyright (C) 2025-2026 SPHARX Ltd. All Rights Reserved.
 * SPDX-FileCopyrightText: 2025-2026 SPHARX Ltd.
 * SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
 * 
 * @file memory_manager.h
 * @brief Airymax 内存管理器接口
 * 
 * 提供内核级内存分配、释放、监控功能。采用 slab 分配器
 * 优化小对象分配性能，支持 NUMA 感知内存布局。
 * 
 * @author SPHARX Ltd. - Airymax Team
 * @date 2026-03-30
 * @version 2.0
 * 
 * @note 线程安全：所有公共接口均为线程安全
 * @see agentrt/atoms/corekern/memory/
 */

#ifndef AIRY_MEMORY_MANAGER_H
#define AIRY_MEMORY_MANAGER_H

#include "airy_types.h"

#ifdef __cplusplus
extern "C" {
#endif
```

#### 4.2 函数注释模板

```c
/**
 * @brief 分配指定大小的内存块
 * 
 * 从内核内存池中分配一块连续内存。分配器会根据请求大小
 * 自动选择最优的分配策略：
 * - 小于 256 字节：使用 slab 分配器
 * - 256 字节 ~ 4KB：使用 buddy 分配器
 * - 大于 4KB：直接向操作系统申请
 * 
 * @param size 请求分配的字节数，必须大于 0
 * @param flags 分配标志位，见 airy_mem_flags_t
 * @return 成功返回内存块指针，失败返回 NULL
 * 
 * @compliance 安全等级: HIGH
 * @compliance 许可证: AGPL-3.0-or-later OR Apache-2.0
 * 
 * @note 调用者负责释放内存（调用 airy_mem_free）
 * @note 返回的内存块已按 16 字节对齐
 * 
 * @warning 不要在信号处理函数中调用此函数
 * 
 * @see airy_mem_free()
 * @see airy_mem_realloc()
 * 
 * @example
 * // 基本用法
 * void* buffer = airy_mem_alloc(1024, AIRY_MEM_FLAG_ZERO);
 * if (buffer == NULL) {
 *     log_error("Memory allocation failed");
 *     return AIRY_ENOMEM;
 * }
 * // 使用 buffer...
 * airy_mem_free(buffer);
 */
void* airy_mem_alloc(size_t size, uint32_t flags);
```

#### 4.3 结构体注释模板

```c
/**
 * @brief 任务执行上下文结构体
 * 
 * 封装任务执行所需的全部上下文信息，包括输入数据、
 * 执行配置和输出缓冲区。此结构体为不透明类型，
 * 外部代码应通过访问函数操作。
 * 
 * @compliance 数据分类: 内部数据结构，不包含用户敏感信息
 */
typedef struct airy_task_context {
    uint64_t task_id;           /**< 任务唯一标识符 */
    char* input_data;           /**< 输入数据指针，内部管理 */
    size_t input_size;          /**< 输入数据大小（字节） */
    char* output_buffer;        /**< 输出缓冲区指针 */
    size_t output_capacity;     /**< 输出缓冲区容量 */
    size_t output_size;         /**< 实际输出大小 */
    uint32_t flags;             /**< 执行标志位 */
    int32_t error_code;         /**< 错误码，0 表示成功 */
    struct timespec start_time; /**< 任务开始时间 */
    struct timespec end_time;   /**< 任务结束时间 */
} airy_task_context_t;
```

---

### 五、Python 注释规范（Google Docstring + SPDX 风格）

#### 5.1 模块注释模板

```python
"""
Copyright (C) 2025-2026 SPHARX Ltd. All Rights Reserved.
SPDX-FileCopyrightText: 2025-2026 SPHARX Ltd.
SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0

Airymax 任务调度模块

本模块提供任务调度核心功能，包括任务提交、状态管理、
优先级队列和依赖解析。调度器采用双思考系统架构：
- t1 快思考：快速路径，处理简单任务
- t2 慢思考：深度路径，处理复杂任务

典型用法:
    from agentrt.scheduler import TaskScheduler
    
    scheduler = TaskScheduler()
    task_id = scheduler.submit(task_plan)
    result = scheduler.wait(task_id)

属性:
    MAX_PRIORITY (int): 最大优先级值，默认为 10
    DEFAULT_TIMEOUT (float): 默认超时时间（秒），默认为 30.0

作者:
    SPHARX Ltd. - Airymax Team

版本:
    2.0.0
日期:
    2026-03-30
"""

from typing import Optional, List, Dict, Any
from dataclasses import dataclass
import logging

MAX_PRIORITY: int = 10
DEFAULT_TIMEOUT: float = 30.0
```

#### 5.2 类注释模板

```python
class TaskScheduler:
    """任务调度器，管理任务的生命周期和执行。
    
    调度器负责接收任务请求、解析依赖关系、分配执行资源，
    并跟踪任务状态。支持优先级队列、超时控制和错误重试。
    
    调度器是线程安全的，可在多线程环境中使用。
    
    合规信息:
        版权: SPHARX Ltd. (2025-2026)
        许可证: AGPL-3.0-or-later OR Apache-2.0
        安全等级: MEDIUM
    
    属性:
        name (str): 调度器名称，用于日志标识
        max_workers (int): 最大工作线程数
        queue_size (int): 当前队列长度
        
    示例:
        >>> scheduler = TaskScheduler(name="main", max_workers=4)
        >>> task_id = scheduler.submit(plan)
        >>> result = scheduler.wait(task_id, timeout=60)
        >>> print(result.status)
        'completed'
    
    注意:
        使用完毕后应调用 shutdown() 方法释放资源，
        或使用上下文管理器自动管理生命周期。
    """
    
    def __init__(self, name: str = "default", max_workers: int = 4) -> None:
        """初始化任务调度器。
        
        Args:
            name: 调度器名称，用于日志和监控标识
            max_workers: 最大工作线程数，必须为正整数
            
        Raises:
            ValueError: 当 max_workers <= 0 时抛出
            
        合规:
            数据保护: 不收集用户个人信息
        """
        pass
```

---

### 六、JavaScript/TypeScript 注释规范（JSDoc/TSDoc + SPDX 风格）

#### 6.1 文件头注释模板

```typescript
/**
 * Copyright (C) 2025-2026 SPHARX Ltd. All Rights Reserved.
 * SPDX-FileCopyrightText: 2025-2026 SPHARX Ltd.
 * SPDX-License-Identifier: AGPL-3.0-or-later OR Apache-2.0
 * 
 * @fileoverview Airymax 任务调度模块
 * 
 * 提供任务调度核心功能，包括任务提交、状态管理、
 * 优先级队列和依赖解析。
 * 
 * @module agentrt/scheduler
 * @author SPHARX Ltd. - Airymax Team
 * @version 2.0.0
 * @date 2026-03-30
 * 
 * @example
 * import { TaskScheduler } from 'agentrt/scheduler';
 * 
 * const scheduler = new TaskScheduler({ maxWorkers: 4 });
 * const taskId = await scheduler.submit(plan);
 * const result = await scheduler.wait(taskId);
 */

import { TaskPlan, TaskResult, TaskStatus } from './types';
```

#### 6.2 类注释模板

```typescript
/**
 * 任务调度器，管理任务的生命周期和执行。
 * 
 * 调度器负责接收任务请求、解析依赖关系、分配执行资源，
 * 并跟踪任务状态。支持优先级队列、超时控制和错误重试。
 * 
 * 合规信息:
 *   - 版权: SPHARX Ltd. (2025-2026)
 *   - 许可证: AGPL-3.0-or-later OR Apache-2.0
 *   - 安全等级: MEDIUM
 *   - 数据保护: GDPR 合规
 * 
 * @template T - 任务输入数据类型
 * @template R - 任务输出数据类型
 * 
 * @example
 * const scheduler = new TaskScheduler<TaskInput, TaskOutput>({
 *   name: 'main',
 *   maxWorkers: 4
 * });
 * 
 * const taskId = await scheduler.submit(plan);
 * const result = await scheduler.wait(taskId);
 * console.log(result.status); // 'completed'
 */
export class TaskScheduler<T = unknown, R = unknown> {
    /**
     * 创建任务调度器实例。
     * 
     * @param manager - 调度器配置选项
     * @param manager.name - 调度器名称，用于日志标识
     * @param manager.maxWorkers - 最大工作线程数
     * @param manager.defaultTimeout - 默认超时时间（毫秒）
     * 
     * @throws {Error} 当 maxWorkers <= 0 时抛出
     * 
     * @compliance 数据最小化: 仅收集必要配置信息
     */
    constructor(manager: SchedulerConfig) {
        // 实现...
    }
}
```

---

### 七、特殊场景注释

#### 7.1 TODO 注释

> **⚠️ BAN-04 合规要求**：所有 TODO/FIXME 必须引用 Issue 编号，无 Issue 引用的 TODO/FIXME 违反 [C编码规范 BAN-04](./C_Cpp_coding_style.md)。

```c
// TODO(zhangsan): 添加任务取消功能 #1234 [优先级: P1] [预计完成: 2026-04-15]
// 技术方案: 使用原子操作实现无锁取消
// 测试计划: 添加并发取消测试用例
```

```python
# TODO(lisi): 实现任务重试逻辑 #567 [优先级: P2]
# 实现思路: 指数退避 + 抖动算法
# 依赖模块: retry.py (需要先重构)
```

#### 7.2 FIXME 注释

> **⚠️ BAN-04 合规要求**：所有 FIXME 必须引用 Issue 编号，无 Issue 引用的 FIXME 违反 [C编码规范 BAN-04](./C_Cpp_coding_style.md)。

```c
// FIXME(wangwu): 并发访问时存在竞态条件 #890 [严重程度: Critical]
// 问题描述: task_id 生成器在多线程环境下可能产生重复ID
// 临时解决方案: 使用线程本地存储
// 永久解决方案: 实现分布式ID生成器
// 影响范围: 所有任务调度相关功能
```

#### 7.3 DEPRECATED 注释

```typescript
/**
 * @deprecated 自版本 2.0.0 起废弃
 * 
 * 请使用 `TaskScheduler.submit()` 代替。
 * 此方法将在 3.0.0 版本中移除。
 * 
 * @deprecation_reason 新的提交接口支持更丰富的选项配置
 * @replacement TaskScheduler.submit()
 * @removal_version 3.0.0
 * 
 * 迁移指南:
 * 旧代码: submitTask(plan)
 * 新代码: scheduler.submit(plan, options)
 * 
 * @see TaskScheduler.submit
 */
submitTask(plan: TaskPlan): string {
    console.warn('submitTask is deprecated, use scheduler.submit()');
    return this.submit(plan);
}
```

#### 7.4 HACK 注释

```c
// HACK(zhaoliu): 临时绕过第三方库的内存泄漏问题
// 第三方库: libevent 2.1.12 (已知bug: #789)
// 问题链接: https://github.com/libevent/libevent/issues/789
// 临时方案: 每1000次调用后重启连接
// 永久方案: 升级到 libevent 2.1.13+ (修复版本)
// 监控指标: 内存使用量、连接重启次数
```

#### 7.5 SECURITY 注释

```python
# SECURITY: 此函数处理用户输入，必须进行严格的参数验证
# 威胁模型: 攻击者可能注入恶意输入
# 防护措施:
#   1. 输入验证: 白名单过滤
#   2. 输出编码: HTML/URL 编码
#   3. 权限检查: 最小权限原则
# 参考文档: OWASP Input Validation Cheat Sheet
# 测试用例: 包含 SQL 注入、XSS 攻击测试
```

---

### 八、注释质量检查与合规验证

#### 8.1 自动化检查工具

| 语言 | 文档生成工具 | 合规检查工具 | 代码质量工具 |
|------|--------------|--------------|--------------|
| C/C++ | Doxygen | REUSE, FOSSA | clang-tidy, cppcheck |
| Python | Sphinx | licensecheck, pip-licenses | pylint, mypy, bandit |
| JavaScript/TypeScript | TypeDoc | license-checker, npm-license | ESLint, TSC, SonarJS |
| Go | go doc | go-licenses, govulncheck | golangci-lint, staticcheck |
| Rust | cargo doc | cargo-deny, cargo-audit | clippy, rustfmt |
| 通用 | - | ScanCode, ORT | - |

#### 8.1.1 SPDX 头部 CI 强制执行（REUSE 工具）

所有源代码文件的 SPDX 版权和许可证声明必须在 CI 中强制检查，使用 [REUSE 工具](https://reuse.software/) 实现：

| 检查项 | CI 配置 | 阻断规则 |
|--------|---------|----------|
| SPDX 声明完整性 | `reuse lint` | 缺少 SPDX 声明阻断合并 |
| 许可证标识有效性 | `reuse lint` | 无效许可证标识阻断合并 |
| 版权年份准确性 | `reuse lint` | 版权年份异常发出警告 |
| 第三方代码声明 | `reuse lint` + 自定义脚本 | 缺少第三方声明阻断合并 |

**CI 集成示例**（GitHub Actions）：
```yaml
# .github/workflows/reuse-check.yml
name: REUSE Compliance
on: [pull_request]
jobs:
  reuse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: REUSE Compliance Check
        uses: fsfe/reuse-action@v3
```

**规则**：`reuse lint` 必须在 CI 中以零错误通过，否则阻断 PR 合并。

#### 8.2 检查规则

**必须检查项**：
- 文件头部是否包含SPDX版权和许可证声明
- 公共 API 是否有完整注释
- 参数说明是否完整
- 返回值说明是否准确
- 异常说明是否完整
- 第三方代码是否保留原始声明
- 许可证兼容性是否符合要求

**建议检查项**：
- 注释是否解释"为什么"（设计意图）
- 注释是否与代码同步
- 注释语言是否一致
- 注释格式是否规范
- 安全注释是否完整
- 合规注释是否准确

#### 8.3 人工审查清单

| 审查维度 | 审查要点 | 通过标准 |
|----------|----------|----------|
| 合规性 | SPDX声明完整性 | 所有文件包含正确的版权和许可证标识 |
| 准确性 | 注释与代码一致性 | 注释准确描述代码功能和行为 |
| 完整性 | 必需注释要素 | 公共API包含所有必需注释要素 |
| 安全性 | 安全注释完整性 | 安全敏感代码包含完整的安全注释 |
| 可读性 | 注释语言质量 | 中文语法正确，表达清晰 |
| 维护性 | 注释时效性 | 注释与代码同步更新，无过时信息 |

---

### 九、最佳实践

#### 9.1 注释编写原则

1. **及时更新**：代码修改时同步更新注释
2. **避免冗余**：不重复代码已表达的信息
3. **保持简洁**：注释应精炼，避免过度描述
4. **使用示例**：复杂接口必须提供使用示例
5. **说明边界**：明确说明参数边界、返回值边界
6. **记录决策**：关键设计决策必须在注释中记录
7. **合规优先**：始终优先考虑合规要求

> **⚠️ 注释与代码同步要求**：当 PR 涉及公共 API 变更（新增、修改、删除函数/方法/类）时，PR 必须同步更新对应的注释。代码审查者应将注释同步性作为 PR 审查的必检项。未更新注释的 PR 应被要求修改后再合并。

#### 9.2 合规最佳实践

1. **SPDX标准化**：始终使用SPDX标准格式
2. **版权年份更新**：每年更新版权年份范围
3. **许可证兼容性检查**：引入新代码前检查许可证兼容性
4. **第三方代码记录**：详细记录第三方代码来源和修改
5. **安全注释完整**：安全敏感代码必须有完整的安全注释
6. **定期合规审计**：定期进行代码合规审计

#### 9.3 常见错误与修正

| 错误类型 | 错误示例 | 正确做法 |
|----------|----------|----------|
| 缺少合规声明 | 无SPDX声明 | 添加完整的SPDX版权和许可证声明 |
| 过时版权年份 | Copyright 2020-2022 | 更新为 Copyright 2025-2026 |
| 错误许可证标识 | SPDX-License-Identifier: MIT | 使用正确的许可证标识 |
| 第三方代码声明缺失 | 直接使用第三方代码 | 添加第三方代码引入声明 |
| 安全注释不完整 | 仅简单标注安全 | 提供完整的安全等级、威胁模型、防护措施 |
| 合规信息不准确 | 错误的专利分类 | 咨询法务部门确定正确的分类 |
| 注释与代码不同步 | 注释描述旧实现 | 更新注释以匹配当前代码 |
| 冗余注释 | `i++; // i 加 1` | 删除无用注释 |
| 误导性注释 | 注释与实际行为矛盾 | 修正注释或代码 |
| 过长注释 | 注释比代码还长 | 重构代码或提取为独立文档 |

---

### 十、参考资料

1. **SPDX 规范**: https://spdx.dev/specifications/
2. **REUSE 合规指南**: https://reuse.software/
3. **Doxygen 官方文档**: https://www.doxygen.nl/manual/
4. **Google Python Style Guide**: https://google.github.io/styleguide/pyguide.html
5. **TypeScript Documentation**: https://www.typescriptlang.org/docs/
6. **Effective Go**: https://golang.org/doc/effective_go
7. **Rust API Guidelines**: https://rust-lang.github.io/api-guidelines/
8. **Airymax 架构设计原则**: [00-architectural-principles.md](../../00-architectural-principles.md)
9. **Airymax 统一术语表**: [90-terminology.md](../90-terminology.md)
10. **Airymax 安全设计指南**: [Security_design_standard.md](../Security_design_standard.md)

---

### 更新记录

| 版本 | 日期 | 修改内容 | 修改人 |
|------|------|----------|--------|
| 1.7 | 2026-03-31 | 重新整理，删除AI和出口管制相关内容，精简文档结构 | Airymax Team |
| 1.6 | 2026-03-25 | 初始版本，基于工程两论和五维正交系统设计 | Airymax文档组 |

---

### 附录：跨文档规范引用

本规范与以下 Airymax 工程规范一致，所有代码注释须同时遵循：

| 规范集 | 说明 | 来源文档 |
|--------|------|---------|
| **BAN-01~13** | 13 项禁止模式（桩函数/假数据/空返回等） | [C编码规范 §18](./C_Cpp_coding_style.md) |
| **REQ-01~08** | 8 项强制规范 | [C编码规范 §1.2](./C_Cpp_coding_style.md) |
| **标准术语** | 8 个架构组件标准名称 | [90-terminology.md](../90-terminology.md) |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.  
SPDX-FileCopyrightText: 2025-2026 SPHARX Ltd.  

---

## Part II: Airymax 配置变更审计日志格式规范

### 目录

1. [概述](#1-概述)
2. [日志格式定义](#2-日志格式定义)
3. [字段详细说明](#3-字段详细说明)
4. [动作类型定义](#4-动作类型定义)
5. [使用示例](#5-使用示例)
6. [最佳实践](#6-最佳实践)
7. [工具与集成](#7-工具与集成)
8. [常见问题](#8-常见问题)

---

### 1. 概述

#### 1.1 目的

配置变更审计日志用于记录Airymax Manager模块中所有配置文件的变更操作，提供：

- **可追溯性**: 完整记录谁在什么时间做了什么变更
- **合规性**: 满足安全审计和合规要求
- **故障排查**: 快速定位配置问题根源
- **变更回滚**: 支持基于审计日志的配置回滚

#### 1.2 设计原则

遵循 **E-2 可观测性原则** 和 **E-1 安全内生原则**：

- **完整性**: 记录所有关键信息，无遗漏
- **不可篡改**: 日志采用加密存储（AES-256-GCM）
- **结构化**: JSON格式，便于解析和查询
- **标准化**: 遵循JSON Schema Draft-07规范

---

### 2. 日志格式定义

#### 2.1 整体结构

审计日志采用 **JSON数组** 格式，每个数组元素为一条独立的审计日志条目：

```json
[
  {
    "timestamp": "2026-04-05T10:30:00.123456Z",
    "action": "CHANGE",
    "config_file": "kernel/settings.yaml",
    "operator": { ... },
    "changes": [ ... ],
    "checksum": { ... },
    "metadata": { ... },
    "result": { ... }
  }
]
```

#### 2.2 必需字段

每条审计日志 **必须** 包含以下字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `timestamp` | string | ISO 8601格式的时间戳 |
| `action` | string | 审计动作类型 |
| `config_file` | string | 配置文件相对路径 |
| `operator` | object | 操作者信息 |
| `checksum` | object | 文件校验和 |

---

### 3. 字段详细说明

#### 3.1 timestamp（时间戳）

**格式**: ISO 8601 (RFC 3339)  
**示例**: `"2026-04-05T10:30:00.123456Z"`

- 必须使用UTC时间（以`Z`结尾）
- 精确到微秒（6位小数）
- 格式: `YYYY-MM-DDTHH:MM:SS.ffffffZ`

> **📝 与运行时日志的时间戳格式差异说明**  
> 本规范（审计日志）使用 ISO 8601 字符串格式（如 `"2026-04-05T10:30:00.123456Z"`），而 [logging_format.md](../10-contracts/05-logging-format.md) 中的运行时日志使用 Unix 时间戳数值格式（如 `1701234567.890`）。两者格式不同的原因：
> - **审计日志**面向合规审查，ISO 8601 格式人类可读，便于安全审计人员直接阅读和比对时间线
> - **运行时日志**面向机器解析和性能监控，Unix 时间戳数值格式解析效率高、存储紧凑、便于计算时间差
> - 两种格式可通过标准库函数互相转换，不存在信息丢失

#### 3.2 action（动作类型）

| 值 | 说明 | 触发场景 |
|---|------|---------|
| `LOAD` | 配置加载 | 系统启动、手动加载 |
| `RELOAD` | 热更新重载 | 文件监听器检测到变更 |
| `CHANGE` | 配置变更 | 用户/API修改配置 |
| `ROLLBACK` | 回滚 | 验证失败、手动回滚 |
| `VALIDATE` | Schema验证 | CI/CD、定期检查 |
| `EXPORT` | 导出 | 备份、迁移 |
| `IMPORT` | 导入 | 部署、恢复 |
| `AUDIT` | 审计检查 | 安全审计、合规检查 |

> **📝 与 logging_format.md 的 AUDIT 级别对齐**  
> 本规范新增 `AUDIT` 动作类型，与 [logging_format.md](../10-contracts/05-logging-format.md) 中定义的 `AUDIT` 日志级别（级别 5）对齐。`AUDIT` 动作用于记录安全审计和合规检查操作，其对应的运行时日志应使用 `audit` 级别输出。

#### 3.3 config_file（配置文件路径）

**格式**: 相对路径（相对于`manager/`目录）

常见配置文件：
- `kernel/settings.yaml` - 内核配置
- `model/model.yaml` - 模型配置
- `security/policy.yaml` - 安全策略
- `agent/registry.yaml` - Agent注册表

#### 3.4 operator（操作者信息）

```json
{
  "type": "user | system | ci_cd",
  "identity": "操作者标识",
  "ip_address": "IP地址（可选）",
  "session_id": "会话ID（可选）"
}
```

| 类型 | 说明 | identity示例 |
|------|------|-------------|
| `user` | 真实用户 | admin, devops_engineer |
| `system` | 系统组件 | config-manager, file-watcher |
| `ci_cd` | CI/CD流水线 | github-actions, jenkins |

#### 3.5 changes（变更详情）

```json
{
  "changes": [
    {
      "path": "YAML路径表达式",
      "old_value": "旧值",
      "new_value": "新值",
      "field_type": "字段类型（可选）"
    }
  ]
}
```

#### 3.6 checksum（校验和）

```json
{
  "checksum": {
    "algorithm": "sha256",
    "before": "变更前的SHA-256哈希",
    "after": "变更后的SHA-256哈希"
  }
}
```

> **📝 checksum.before 的 null 处理**  
> 对于 `LOAD` 动作（首次加载配置文件），由于不存在变更前的文件状态，`checksum.before` 字段应为 `null`：
> ```json
> {
>   "checksum": {
>     "algorithm": "sha256",
>     "before": null,
>     "after": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
>   }
> }
> ```
> `checksum.before` 为 `null` 仅在 `LOAD` 动作时合法，其他动作类型（`CHANGE`、`RELOAD`、`ROLLBACK` 等）的 `checksum.before` 必须为有效的 SHA-256 哈希值。

#### 3.7 metadata（扩展元数据）

| 字段 | 类型 | 说明 |
|------|------|------|
| `environment` | string | 运行环境 |
| `version` | string | Manager模块版本号 |
| `correlation_id` | string | 关联ID |
| `source` | string | 变更来源 |
| `reason` | string | 变更原因说明 |
| `approved_by` | string | 审批人 |
| `ticket_id` | string | 关联工单ID |

#### 3.8 result（操作结果）

| 字段 | 类型 | 说明 |
|------|------|------|
| `success` | boolean | 操作是否成功 |
| `error_code` | integer \| string | 错误码（C 内核为负整数如 `-2`，SDK 为十六进制字符串如 `"0x0003"`） |
| `error_message` | string | 错误消息 |
| `duration_ms` | integer | 操作耗时（毫秒） |

---

### 4. 最佳实践

#### 4.1 日志记录原则

1. **及时记录**: 变更发生后立即记录，不要延迟
2. **完整记录**: 包含所有必需字段
3. **准确记录**: 确保时间戳、哈希值信息准确
4. **安全存储**: 使用AES-256-GCM加密存储

#### 4.2 日志保留策略

```yaml
audit:
  enabled: true
  log_path: "audit/config_audit.log"
  events:
    - config.load
    - config.reload
    - config.change
    - config.rollback
    - config.validate
    - config.export
    - config.import
  retention_days: 90
```

---

### 5. 工具与集成

| 工具 | 路径 | 用途 |
|------|------|------|
| 审计日志生成器 | `tools/audit_log_generator.py` | 生成测试日志 |
| Schema验证器 | `tests/test_schema_validation.py` | 验证配置文件 |

---

### 附录

#### A. 版本历史

| 版本 | 日期 | 变更说明 |
|------|------|---------|
| V1.0 | 2026-04-05 | 初始版本 |

---

**文档维护**: SPHARX Ltd.
**最后更新**: 2026-04-05  

---

## Part III: Airymax 日志打印规范

> **SSoT 声明**： 本文档的日志格式与传输管道契约以 AirymaxOS [`50-engineering-standards/20-contracts/contracts.md`](../../../AirymaxOS/50-engineering-standards/20-contracts/contracts.md) 为单一数据源（SSoT）。本文档仅补充 agentrt 用户态运行时特有的日志打印差异（如 `AIRY_LOG_*` / `SVC_LOG_*` 宏的跨平台用法、HiLog 语法参考映射）。当两者冲突时，以 AirymaxOS SSoT 为准。

### 第 0 章 Airymax 日志体系总览

#### 0.1 日志层级与对应宏

Airymax 采用分层日志架构，各层使用不同的日志宏和头文件：

| 层级 | 日志宏 | 头文件 | 说明 |
|------|--------|--------|------|
| daemon 服务层 | `SVC_LOG_*` | `svc_log.h` | 系统用户态服务日志，如 IPC 服务、资源管理 |
| atoms/gateway/protocols | `AIRY_LOG_*` | `airy_log.h` | 微核心核心、网关、协议层日志 |
| cupolas 安全穹顶 | `AIRY_LOG_AUDIT` / `SVC_LOG_AUDIT` | `airy_log.h` / `svc_log.h` | 安全审计日志，用于合规审查和取证分析 |
| Desktop | `logger.*()` | `logger.ts` | 桌面端 TypeScript 日志 |
| Python SDK | `logging.*` | Python `logging` 模块 | Python 绑定层日志 |
| Go SDK | `zap.*` | `zap` logger | Go 绑定层日志 |
| Rust SDK | `tracing::*` | `tracing` crate | Rust 绑定层日志 |

#### 0.2 HiLog 与 Airymax 日志系统的关系

> **⚠️ 重要说明**：HiLog 是 OpenHarmony 的日志系统，**不是** Airymax 的日志系统。本文档中出现的 HiLog 示例仅作为日志用法参考，Airymax 实际代码中应使用对应的 `AIRY_LOG_*`（atoms/gateway/protocols 层）或 `SVC_LOG_*`（daemon 服务层）宏。
>
> 对应关系：
> - `HiLog::FATAL(LABEL, ...)` → `AIRY_LOG_FATAL(MODULE, ...)` 或 `SVC_LOG_FATAL(TAG, ...)`
> - `HiLog::ERROR(LABEL, ...)` → `AIRY_LOG_ERROR(MODULE, ...)` 或 `SVC_LOG_ERROR(TAG, ...)`
> - `HiLog::WARN(LABEL, ...)` → `AIRY_LOG_WARN(MODULE, ...)` 或 `SVC_LOG_WARN(TAG, ...)`
> - `HiLog::INFO(LABEL, ...)` → `AIRY_LOG_INFO(MODULE, ...)` 或 `SVC_LOG_INFO(TAG, ...)`
> - `HiLog::DEBUG(LABEL, ...)` → `AIRY_LOG_DEBUG(MODULE, ...)` 或 `SVC_LOG_DEBUG(TAG, ...)`
> - `HiLog::AUDIT(LABEL, ...)` → `AIRY_LOG_AUDIT(MODULE, ...)` 或 `SVC_LOG_AUDIT(TAG, ...)`

#### 0.3 结构化 JSON 日志格式（v0.1.0+ 标准）

自 v0.1.0 起，Airymax 所有模块必须采用结构化 JSON 日志格式，兼容 OpenTelemetry Logs 规范。最小必填字段：

```json
{
  "timestamp": "2026-04-27T12:34:56.789Z",
  "level": "ERROR",
  "module": "atoms",
  "function": "atoms_secure_alloc",
  "trace_id": "0af7651916cd43dd8448eb211c80319c",
  "request_id": "req-abc123",
  "message": "NUMA allocation failed",
  "err_code": -12
}
```

**要求**：
- 所有日志输出必须为合法 JSON（每行一条，NDJSON 格式）
- `trace_id` 和 `request_id` 为必填字段，若无上下文则填 `"0"` 占位
- 兼容 OpenTelemetry Logs 数据模型，支持直接导入 OTel Collector

#### 0.4 trace_id / request_id 传播规范

所有跨模块、跨进程调用必须传播 `trace_id` 和 `request_id`，以实现全链路追踪：

**传播规则**：
1. **请求入口**：在请求入口（如 IPC 网关、HTTP handler）生成 `trace_id`（W3C Trace Context 格式，32 位 hex）和 `request_id`（UUID v4）
2. **进程内传播**：通过 `airy_tls_get/set()`（参见 `platform.h`）在线程局部存储中保存当前 `trace_id`/`request_id`
3. **跨进程传播**：通过 IPC 消息头或 HTTP header（`traceparent` / `x-request-id`）传递
4. **日志输出**：每条日志必须包含当前 `trace_id` 和 `request_id`

**示例**：
```c
// 请求入口生成
airy_tls_set(AIRY_TLS_KEY_TRACE_ID, trace_id);
airy_tls_set(AIRY_TLS_KEY_REQUEST_ID, request_id);

// 日志中自动携带
AIRY_LOG_INFO(MODULE, "Processing request: op=%s", operation);
// 输出: {"trace_id":"0af765...","request_id":"req-abc123","message":"Processing request: op=submit",...}
```

---

### 第 1 章 日志系统架构

#### 1.1 日志的控制论意义

##### 1.1.1 日志作为反馈机制

从控制论角度看，日志系统是 Airymax 的**感觉神经系统**，承担以下关键功能：

1. **状态观测**：采集系统运行状态信息
2. **偏差检测**：识别实际行为与预期目标的偏离
3. **反馈信号**：为控制系统提供调节依据
4. **历史追溯**：记录系统演化轨迹用于分析优化

```
传感器（代码埋点） → 信号传输（日志输出） → 控制器（运维人员/自动化系统）
                                                         ↓
执行器（系统调节） ← 决策制定（问题分析） ← 信息处理（日志分析）
```

##### 1.1.2 日志的可观测性理论

**可观测性三要素**：

1. **Logs（日志）**：离散事件记录，回答"发生了什么"
2. **Metrics（指标）**：聚合统计数据，回答"系统状态如何"
3. **Traces（追踪）**：请求链路记录，回答"请求经过哪些组件"

本规范主要针对 Logs，但需与 Metrics 和 Traces 协调设计。

#### 1.2 日志系统设计原则

##### 原则 1-1【信噪比最大化】：在提供足够信息和减少噪音间找到平衡

**解释**：日志的价值 = 有用信息量 / 日志总量。过多的日志会淹没关键信息，过少的日志无法有效观测。

**实施指南**：
- 关键流程节点必须记录（高价值信息）
- 高频正常操作减少记录（降低噪音）
- 异常情况详细记录（故障诊断需要）

##### 原则 1-2【分层观测】：按控制层次组织日志级别

```
战略层（FATAL/ERROR）   → 系统级异常，需要立即干预
审计层（AUDIT）         → 安全审计事件，合规审查和取证分析
战术层（WARN）          → 功能级异常，需要关注和处理
操作层（INFO）          → 业务流程记录，用于审计和追溯
调试层（DEBUG/TRACE）   → 详细技术信息，用于问题定位
```

##### 原则 1-3【自适应调节】：日志系统应支持动态级别调整和流量控制

**解释**：固定不变的日志策略无法适应变化的环境，需要具备负反馈调节能力。

**实施指南**：
- 支持运行时调整日志级别
- 实现日志流量限制防止资源耗尽
- 异常情况自动提升相关模块日志级别

---

### 第 2 章 日志分级与分类

> **⚠️ 提示**：以下示例使用 HiLog 语法仅作参考。Airymax 实际代码必须使用 `SVC_LOG_*`（daemon 层）或 `AIRY_LOG_*`（atoms 层）宏，定义分别位于 `svc_log.h` 和 `airy_log.h`。

#### 2.1 日志级别定义

基于控制论的"偏差程度"概念，定义以下日志级别：

##### FATAL（致命级）

**定义**：系统发生灾难性故障，即将或已经崩溃，需要立即干预。

**特征**：
- 系统核心功能不可用
- 数据丢失或损坏风险
- 安全防线被突破
- 无法自动恢复

**示例**：
```cpp
HiLog::FATAL(LABEL, "Database connection pool exhausted, all retries failed");
HiLog::FATAL(LABEL, "Kernel panic: Unable to mount root filesystem");
```

**响应要求**：7×24 小时立即响应，自动触发告警

##### ERROR（错误级）

**定义**：功能发生错误，影响正常使用，可以恢复但代价较高。

**特征**：
- 单个功能模块失效
- 用户体验受损
- 需要人工介入或系统重置
- 可能扩散为 FATAL

**示例**：
```cpp
HiLog::ERROR(LABEL, "Failed to load user profile after 3 retries");
HiLog::ERROR(LABEL, "Network timeout: API request failed with status 500");
```

**响应要求**：工作日 4 小时内响应，纳入问题跟踪系统

##### WARN（警告级）

**定义**：发生非预期情况，对用户影响不大，可以自动恢复或简单处理。

**特征**：
- 系统降级运行但未失效
- 性能下降但功能可用
- 可能发展为 ERROR
- 通常可自动恢复

**示例**：
```cpp
HiLog::WARN(LABEL, "High memory usage detected: 85% of total capacity");
HiLog::WARN(LABEL, "Slow query detected: Execution time exceeded threshold (5s)");
```

**响应要求**：定期审查，趋势分析，预防性优化

##### INFO（信息级）

**定义**：记录业务关键流程节点，用于还原主要运行过程。

**特征**：
- 正常业务流程的关键里程碑
- 重要的状态变更
- 用户操作的审计记录
- 默认版本中开启

**示例**：
```cpp
HiLog::INFO(LABEL, "User login successful: userId=12345, ip=192.168.1.100");
HiLog::INFO(LABEL, "Order created: orderId=ORD20260321001, amount=299.00");
```

**响应要求**：用于审计、统计、问题排查参考

##### DEBUG（调试级）

**定义**：比 INFO 更详细的流程记录，用于分析问题。

**特征**：
- 函数入口出口参数
- 中间计算结果
- 条件分支选择
- 仅在调试版本或调试开关打开时输出

**示例**：
```cpp
HiLog::DEBUG(LABEL, "Entering CalculatePrice: basePrice=199, discount=0.8, taxRate=0.13");
HiLog::DEBUG(LABEL, "Cache hit for key=user_profile_12345");
```

**使用要求**：正式发布版本默认关闭，按需开启

##### AUDIT（审计级）

**定义**：记录安全审计事件，用于合规审查和取证分析。cupolas 安全穹顶模块的审计日志必须使用此级别。

**特征**：
- 安全相关操作（认证、授权、密钥操作）
- 不可篡改，必须持久化存储
- 包含完整的操作上下文（主体、客体、操作、结果）
- 受合规法规约束（如 PCI-DSS、GDPR、HIPAA）

**示例**：
```cpp
HiLog::AUDIT(LABEL, "Access granted: subject=%s, resource=%s, action=%s, risk_score=%.2f",
             subject_id, resource_id, action, risk_score);
HiLog::AUDIT(LABEL, "Capability created: subject=%s, resource=%s, permissions=0x%x",
             subject_id, resource_id, permissions);
```

**响应要求**：审计日志必须持久化，保留期限符合合规要求，不可被应用层删除

> **Airymax 映射**：`HiLog::AUDIT(LABEL, ...)` → `AIRY_LOG_AUDIT(MODULE, ...)`（atoms 层）或 `SVC_LOG_AUDIT(TAG, ...)`（daemon 层）

#### 2.2 日志级别选择矩阵

| 场景类型 | 频率 | 影响范围 | 推荐级别 | 示例 |
|---------|------|---------|---------|------|
| 系统崩溃 | 极低 | 全局 | FATAL | 内核 panic |
| 功能失效 | 低 | 局部 | ERROR | 数据库连接失败 |
| 性能下降 | 中 | 局部 | WARN | 内存使用率高 |
| 业务流程 | 高 | 单请求 | INFO | 订单创建成功 |
| 安全审计 | 低 | 单操作 | AUDIT | 访问授权/拒绝 |
| 详细跟踪 | 很高 | 单操作 | DEBUG | 函数参数值 |

---

### 第 3 章 日志内容规范

> **⚠️ 提示**：以下示例使用 HiLog 语法仅作参考。Airymax 实际代码必须使用 `SVC_LOG_*`（daemon 层）或 `AIRY_LOG_*`（atoms 层）宏，定义分别位于 `svc_log.h` 和 `airy_log.h`。

#### 3.1 信息论视角下的日志内容

从信息论角度看，日志是**消除不确定性**的信息载体。好的日志应该：

1. **高信息量**：能够显著减少系统状态的不确定性
2. **低冗余度**：避免重复和无意义的信息
3. **可解码性**：接收方能够准确理解含义

#### 3.2 日志内容基本要求

##### 规则 3-1【语言规范】：日志使用英文描述，拼写无误，符合语法规范

**【反例】**
```cpp
HiLog::ERROR(LABEL, "Error happened");  // 哪里出错？什么原因？
HiLog::ERROR(LABEL, "1234");            // 完全不知道什么含义
```

**【正例】**
```cpp
HiLog::ERROR(LABEL, "Failed to connect to Redis server: host=redis-master, port=6379, error=Connection refused");
```

##### 规则 3-2【禁止隐私信息】：日志中禁止打印隐私敏感信息

**隐私信息范围**：
- 硬件序列号
- 个人账号和密码
- 生物识别信息
- 身份证号等身份证明
- 医疗记录等敏感数据

**【反例】**
```cpp
HiLog::INFO(LABEL, "User password verification: userId=admin, password=Secret123");
```

**【正例】**
```cpp
HiLog::INFO(LABEL, "User authentication result: userId=admin, success=false, reason=invalid_password");
```

##### 规则 3-3【禁止无关信息】：日志中禁止打印与业务无关的信息

**禁止内容**：
- issue 单号、需求单号
- 公司部门名称、开发人员姓名
- 工号、名字缩写
- 当天的心情、天气等

##### 规则 3-4【避免重复】：日志中禁止打印重复信息

**【反例】**
```cpp
// 文件 A.cpp
HiLog::ERROR(LABEL, "Database connection failed");

// 文件 B.cpp（不同位置，相同内容）
HiLog::ERROR(LABEL, "Database connection failed");
```

**【正例】**
```cpp
// 添加上下文区分
HiLog::ERROR(LABEL, "Primary database connection failed: host=db-primary");
HiLog::ERROR(LABEL, "Secondary database connection failed: host=db-backup");
```

##### 规则 3-5【禁止业务调用】：禁止在日志打印语句中调用业务接口函数

**解释**：日志不应影响业务逻辑，避免产生副作用。

**【反例】**
```cpp
HiLog::INFO(LABEL, "Processing order: %s", order.ToString().c_str());  // ToString() 可能有副作用
```

**【正例】**
```cpp
std::string orderStr = order.ToString();
HiLog::INFO(LABEL, "Processing order: %s", orderStr.c_str());
```

#### 3.3 日志格式模式

##### 模式 3-1【事件记录】who do what

**格式**：`Actor Action Object [Result]`

**示例**：
```cpp
HiLog::INFO(LABEL, "UserService created new user: userId=12345, email=user@example.com");
HiLog::INFO(LABEL, "PaymentGateway processed payment: orderId=ORD001, amount=299.00, status=success");
```

##### 模式 3-2【状态变化】state_name:s1→s2, reason:msg

**格式**：`StateName: OldState -> NewState, Reason: Message`

**示例**：
```cpp
HiLog::INFO(LABEL, "Connection state: CONNECTED -> DISCONNECTED, reason: Server timeout after 30s");
HiLog::WARN(LABEL, "Order status: PENDING -> CANCELLED, reason: Payment timeout exceeded 15 minutes");
```

##### 模式 3-3【参数值】name1=value1, name2=value2

**格式**：键值对列表，逗号分隔

**示例**：
```cpp
HiLog::DEBUG(LABEL, "Function parameters: inputSize=1024, compressionRatio=0.7, outputSize=716");
```

##### 模式 3-4【成功结果】xxx successful

**示例**：
```cpp
HiLog::INFO(LABEL, "Database backup completed successfully: duration=300s, size=1.2GB");
```

##### 模式 3-5【失败结果】xxx failed, please xxx

**示例**：
```cpp
HiLog::ERROR(LABEL, "Connect to server failed, please check network configuration: host=api.example.com, port=443");
```

---

### 第 4 章 日志打印策略

> **⚠️ 提示**：以下示例使用 HiLog 语法仅作参考。Airymax 实际代码必须使用 `SVC_LOG_*`（daemon 层）或 `AIRY_LOG_*`（atoms 层）宏，定义分别位于 `svc_log.h` 和 `airy_log.h`。

#### 4.1 打印时机控制

##### 规则 4-1【高频代码禁打日志】：高频代码的正常流程中禁止打印日志

**高频场景**：
- 被高频调用的接口函数（>1000 次/秒）
- 大数据量处理的循环中
- 高频的软硬件中断处理中
- 协议数据流处理中
- 多媒体音视频流处理中
- 显示屏幕刷新处理中

**例外**：错误分支可以打印 ERROR 级别日志

##### 规则 4-2【频率限制】：可能重复发生的日志需要进行频率限制

**实施方法**：

**方法 1：时间间隔限制**
```cpp
// 每秒最多打印一次
static std::chrono::steady_clock::time_point last_log_time;
auto now = std::chrono::steady_clock::now();
if (now - last_log_time > std::chrono::seconds(1)) {
    HiLog::WARN(LABEL, "High memory usage detected: 85%");
    last_log_time = now;
}
```

**方法 2：计数限制**
```cpp
// 每 100 次只打印一次
static std::atomic<int> counter{0};
if (++counter % 100 == 0) {
    HiLog::DEBUG(LABEL, "Processing packet #%d", counter.load());
}
```

**方法 3：采样打印**
```cpp
// 随机采样 1%
if (rand() % 100 == 0) {
    HiLog::DEBUG(LABEL, "Request processing details: ...");
}
```

##### 规则 4-3【关键点必打】：在基本不可能发生的点必须要打印日志

**解释**：根据墨菲定律，只要有可能发生就一定会发生。这些点一旦发生问题就是疑难杂症。

**典型场景**：
- switch 语句的 default 分支（理论上不会走到）
- 断言检查失败（认为永远不成立的条件）
- 防御性编程检查（认为不会出现的异常情况）

**示例**：
```cpp
switch (status) {
    case STATUS_A: HandleA(); break;
    case STATUS_B: HandleB(); break;
    case STATUS_C: HandleC(); break;
    default:
        // 理论上不会到这里，但必须记录
        HiLog::FATAL(LABEL, "Unexpected status value: %d in state machine", static_cast<int>(status));
        abort();
}
```

#### 4.2 性能影响最小化

##### 规则 4-4【延迟生成】：日志字符串应在日志打印时再生成

**解释**：推迟字符串构造可以减少不必要的开销，当日志级别关闭时避免生成。

**【反例】**
```cpp
std::string detailed_info = GenerateDetailedInfo();  // 总是生成
HiLog::DEBUG(LABEL, "%s", detailed_info.c_str());     // 但 DEBUG 可能关闭
```

**【正例】**
```cpp
// 使用条件判断延迟生成：仅当日志级别启用时才生成字符串
if (AIRY_LOG_DEBUG_ENABLED()) {
    std::string detailed_info = GenerateDetailedInfo();
    AIRY_LOG_DEBUG(MODULE, "%s", detailed_info.c_str());
}
```

> **注意**：原示例 `HiLog::DEBUG(LABEL, "%s", [](){ return GenerateDetailedInfo(); }())` 中的 lambda 被 `()` 立即调用，实际上并未实现延迟求值。正确的延迟求值方式是先检查日志级别是否启用，再生成字符串。

##### 规则 4-5【避免长日志】：日志打印长度不要过长，尽可能使日志记录显示在一行以内

**建议**：
- 单行长度控制在 100 字符左右
- 尽量不要超过 160 字符
- 超长内容考虑分条或截断

---

### 第 5 章 HiLog 接口使用规范

> **⚠️ 提示**：以下示例使用 HiLog 语法仅作参考。Airymax 实际代码必须使用 `SVC_LOG_*`（daemon 层）或 `AIRY_LOG_*`（atoms 层）宏，定义分别位于 `svc_log.h` 和 `airy_log.h`。

#### 5.1 Domain ID 分配的系统工程方法

##### 规则 5-1【独立 Domain】：每个业务须有独立的 Domain ID

**解释**：Domain ID 是日志系统的"命名空间"，用于隔离不同业务的日志流，支持独立的度量和管理。

**Domain ID 结构**：
```
32 位整数，16 进制表示：0xD0xxxyy
├─ D0: domain 域标识（固定）
├─ xxx: 高 12 位，DFX 分配值（业务领域）
└─ yy: 低 8 位，业务内部分配（模块/层次）
```

**申请流程**：
1. 向 DFX 领域提交申请
2. 说明业务范围和组织归属
3. DFX 分配高 12 位标识
4. 业务内部自行分配低 8 位

##### 规则 5-2【层次化分配】：每个业务内部基于 Domain ID 分配机制按层次、模块粒度划分

**示例：蓝牙（BT）业务 Domain ID 分配**

| Domain 名称 | Domain ID | 说明 |
|-----------|---------|------|
| BT | 0xD000100 | 蓝牙总领域 |
| BT-App1 | 0xD000101 | 应用层 1 |
| BT-App2 | 0xD000102 | 应用层 2 |
| BT-Service1 | 0xD000103 | 服务层 1 |
| BT-Service2 | 0xD000104 | 服务层 2 |
| BT-HAL | 0xD000105 | 硬件抽象层 |
| BT-Driver1 | 0xD000106 | 驱动层 1 |
| BT-Driver2 | 0xD000107 | 驱动层 2 |

**层次映射**：
```
0xD0001yy
       └─ yy: 00=总领域，01-7F=层次/模块，80-FF=保留
```

#### 5.2 流量管控

##### 规则 5-3【阈值管理】：日志服务会对每个业务的日志量进行流量管控，修改默认阈值需要经过评审

**默认阈值**：10240 字节/秒/Domain ID

**超阈值处理**：
1. 系统自动限流，丢弃超出部分的日志
2. 记录限流事件（FATAL 级别）
3. 通知运维人员

**阈值调整流程**：
1. 提交书面申请，说明调整理由和预期值
2. DFX 领域组织评审
3. 评估对其他业务的影响
4. 批准后实施调整

#### 5.3 隐私参数标识

##### 规则 5-4【隐私标识】：正确填写日志格式化隐私参数标识{public},{private}

**解释**：HiLog API 会自动过滤标记为{private}的参数内容，保护敏感信息。

**【示例】**
```cpp
// 源码
HiLog::INFO(LABEL, "Device Name:%{public}s, IP:%{private}s, Token:%{private}s.", 
            deviceName, ipAddress, authToken);

// 日志输出
Device Name:MyPad001, IP:<private>, Token:<private>.
```

**使用要求**：
- 默认所有参数标记为{private}
- 明确确认不含隐私的参数标记为{public}
- 定期审查{public}参数的安全性

---

### 第 6 章 日志质量管理

#### 6.1 日志质量评估指标

##### 指标 1：日志覆盖率

**定义**：关键流程节点的日志记录比例

**计算公式**：
```
日志覆盖率 = (有关键日志的流程节点数 / 总流程节点数) × 100%
```

**目标值**：≥95%

##### 指标 2：日志准确率

**定义**：日志内容与实际情况的一致性

**评估方法**：抽样审查，对比日志与实际行为

**目标值**：≥98%

##### 指标 3：日志信噪比

**定义**：有价值日志与总日志量的比例

**计算公式**：
```
信噪比 = (ERROR + WARN + 关键 INFO)数量 / 总日志数量 × 100%
```

**目标值**：INFO 级别以上 ≥30%，DEBUG 级别 ≥5%

#### 6.2 日志质量改进 PDCA 循环

##### Plan（计划）

- 分析当前日志质量问题（如噪音过多、关键信息缺失）
- 设定改进目标和优先级
- 制定改进行动计划

##### Do（执行）

- 实施日志优化（删除无用日志、补充关键日志）
- 调整日志级别和频率
- 更新日志规范文档

##### Check（检查）

- 收集日志使用反馈
- 分析问题定位效率是否提升
- 测量日志质量指标

##### Act（处理）

- 成功经验标准化（纳入规范）
- 失败教训案例化（加入培训）
- 遗留问题进入下一轮 PDCA

#### 6.3 日志自适应调节

##### 机制 1：动态级别调整

**场景**：系统检测到异常时自动提升相关模块日志级别

**实现**：
```cpp
// 伪代码示例
if (system.AnomalyDetected()) {
    LogLevelManager::SetLevel("PaymentModule", LogLevel::DEBUG);
    // 自动提升支付模块日志级别以便详细跟踪
}
```

##### 机制 2：智能采样

**场景**：正常情况下采样打印，异常时全量打印

**实现**：
```cpp
// 伪代码示例
if (request.IsSuccess()) {
    // 成功请求 1% 采样
    if (rand() % 100 == 0) {
        HiLog::INFO(LABEL, "Request succeeded: %s", request.GetId().c_str());
    }
} else {
    // 失败请求全量记录
    HiLog::ERROR(LABEL, "Request failed: %s, error=%s", 
                 request.GetId().c_str(), request.GetError().c_str());
}
```

---

### 附录 A：常见场景日志打印要求

#### A.1 数据库操作

**增删改查操作**：
- 记录操作类型、操作对象、操作结果
- 不记录具体内容（防止泄露隐私）
- 性能敏感操作记录耗时

**示例**：
```cpp
HiLog::INFO(LABEL, "Database INSERT: table=users, rows=1, duration=5ms, result=success");
HiLog::WARN(LABEL, "Database DELETE: table=sessions, rows=0, warning=no matching records");
```

#### A.2 文件操作

**常规操作**：
- 记录操作类型、文件名（系统文件）、操作结果
- 不记录文件内容
- 批量操作汇总记录

**示例**：
```cpp
HiLog::INFO(LABEL, "File created: path=/var/log/app.log, size=1024KB");
HiLog::INFO(LABEL, "Batch delete completed: directory=/tmp, deletedCount=150, totalSize=50MB");
```

#### A.3 线程操作

**线程生命周期**：
- 记录创建、启动、暂停、终止
- 记录线程号、线程名

**示例**：
```cpp
HiLog::INFO(LABEL, "Thread started: name=Worker-1, id=0x12345678");
HiLog::ERROR(LABEL, "Thread deadlock detected: name=DataProcessor, blockedFor=30s");
```

#### A.4 网络通信

**连接管理**：
- 记录连接建立、维持、断开及原因

**示例**：
```cpp
HiLog::INFO(LABEL, "TCP connection established: local=192.168.1.100:5000, remote=10.0.0.1:8080");
HiLog::WARN(LABEL, "TCP connection closed: reason=peer_timeout, duration=300s");
```

---
### 第 7 章 Airymax 模块日志示例

> **⚠️ 提示**：以下示例使用 HiLog 语法仅作参考。Airymax 实际代码必须使用 `SVC_LOG_*`（daemon 层）或 `AIRY_LOG_*`（atoms 层）宏，定义分别位于 `svc_log.h` 和 `airy_log.h`。

#### 7.1 Atoms（原子层）日志规范
Atoms模块实现微核心核心功能，日志要求最高级别的性能和精度：

##### 7.1.1 内存管理日志（映射原则：M-3 拓扑优化）
```cpp
/**
 * @brief NUMA感知内存分配日志 - 体现工程观（E-2）和系统观（S-1）原则
 * 
 * 记录内存分配拓扑信息，支持性能分析和故障诊断。
 * 日志级别根据分配频率动态调节，避免高频操作产生过多日志。
 * 
 * @see memoryrovol.md 中的记忆进化算法
 */
void atoms_mem_alloc_numa(size_t size, int numa_node, uint32_t flags) {
    // 低频分配：详细记录
    if (size > LARGE_ALLOC_THRESHOLD) {
        HiLog::INFO(LABEL, "NUMA large allocation: size=%zu, node=%d, flags=0x%x, caller=%s",
                   size, numa_node, flags, __FUNCTION__);
    }
    // 高频分配：简化记录（避免日志洪水）
    else if (should_log_memory_allocation()) {
        HiLog::DEBUG(LABEL, "NUMA alloc: size=%zu, node=%d", size, numa_node);
    }
    
    // 分配失败：错误日志（必打）
    void* ptr = internal_alloc_numa(size, numa_node, flags);
    if (ptr == nullptr) {
        HiLog::ERROR(LABEL, "NUMA allocation failed: size=%zu, node=%d, errno=%d, free_memory=%zu",
                    size, numa_node, errno, get_free_memory());
        return nullptr;
    }
    
    return ptr;
}
```

##### 7.1.2 任务调度日志（映射原则：C-2 认知优化）
```cpp
/**
 * @brief 双思考系统任务调度日志 - 体现认知观（C-1, C-2）原则
 * 
 * 记录t1 快思考/t2 慢思考路径选择，支持认知模式分析。
 * 结构化日志支持机器学习驱动的调度优化。
 */
void schedule_task(Task* task) {
    // 记录调度决策
    SystemSelection selection = select_system_based_on_complexity(task);
    HiLog::INFO(LABEL, "Task scheduled: id=%s, system=%s, complexity=%.2f, priority=%d",
               task->id, 
               selection == SYSTEM_1 ? "System1" : "System2",
               task->complexity_score,
               task->priority);
    
    // 记录执行时间（性能关键）
    auto start_time = std::chrono::high_resolution_clock::now();
    execute_task(task);
    auto end_time = std::chrono::high_resolution_clock::now();
    
    auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end_time - start_time);
    if (duration > SLOW_TASK_THRESHOLD) {
        HiLog::WARN(LABEL, "Slow task execution: id=%s, duration=%lldμs, system=%s",
                   task->id, duration.count(), 
                   selection == SYSTEM_1 ? "System1" : "System2");
    }
}
```

#### 7.2 daemon（用户态服务层）日志规范
daemon 用户态服务层作为系统服务，强调可靠性和可观测性：

##### 7.2.1 IPC通信服务日志（映射原则：E-3 通信基础设施）
```python
class IpcDaemon:
    """IPC用户态服务层日志 - 体现系统观（S-3）和工程观（E-4）原则
    
    结构化日志支持OpenTelemetry集成和分布式追踪。
    安全相关日志必须包含完整上下文用于审计。
    """
    
    async def process_message(self, message: IpcMessage) -> ProcessResult:
        # 请求日志（包含消息摘要）
        logger.info(
            "IPC message received",
            extra={
                "message_id": message.id,
                "sender": message.sender,
                "operation": message.operation,
                "size": len(message.payload),
                "timestamp": message.timestamp,
                "log_type": "ipc_request"
            }
        )
        
        try:
            # 处理逻辑
            result = await self._do_process(message)
            
            # 成功日志（简化）
            logger.debug(
                "IPC message processed",
                extra={
                    "message_id": message.id,
                    "duration_ms": result.duration_ms,
                    "success": True,
                    "log_type": "ipc_response"
                }
            )
            return result
            
        except SecurityException as e:
            # 安全异常日志（详细，用于审计）
            logger.warning(
                "IPC security violation",
                extra={
                    "message_id": message.id,
                    "sender": message.sender,
                    "operation": message.operation,
                    "error": str(e),
                    "stack_trace": traceback.format_exc(),
                    "log_type": "security_audit"
                }
            )
            raise
            
        except Exception as e:
            # 一般异常日志
            logger.error(
                "IPC processing failed",
                extra={
                    "message_id": message.id,
                    "error": str(e),
                    "log_type": "error"
                },
                exc_info=True
            )
            raise
```

#### 7.3 cupolas（安全穹顶）日志规范
cupolas模块实现零信任安全模型，日志必须支持安全审计和合规性：

##### 7.3.1 安全策略审计日志（映射原则：D-4 安全审计）
```typescript
/**
 * 安全策略审计日志 - 体现安全工程（D-4）和设计美学（A-1）原则
 * 
 * 结构化审计日志支持监管合规和取证分析。
 * 敏感信息脱敏处理，防止日志泄露隐私。
 */
class SecurityAuditLogger {
  private readonly logger: Logger;
  
  logAccessDecision(request: AccessRequest, decision: AccessDecision): void {
    // 结构化审计日志（支持SIEM系统集成）
    this.logger.info('Access decision recorded', {
      timestamp: new Date().toISOString(),
      audit_id: generateAuditId(),
      subject: this.maskSensitiveData(request.subject),
      resource: request.resource,
      action: request.action,
      decision: decision.result,
      reason: decision.reason,
      risk_score: decision.riskScore,
      context_hash: hashContext(request.context),
      log_category: 'security_audit',
      compliance_tags: ['pci_dss', 'gdpr', 'hipaa']
    });
    
    // 高风险访问：额外记录
    if (decision.riskScore > RISK_THRESHOLD_HIGH) {
      this.logger.warning('High risk access detected', {
        audit_id: generateAuditId(),
        risk_score: decision.riskScore,
        justification: request.justification,
        reviewer: request.reviewer,
        log_category: 'risk_alert'
      });
    }
  }
  
  private maskSensitiveData(data: string): string {
    // 脱敏处理，防止日志泄露敏感信息
    return data.replace(/\\d{4}-\\d{4}-\\d{4}-\\d{4}/g, '****-****-****-****');
  }
}
```

#### 7.4 commons（公共库层）日志规范
Common模块提供跨层基础设施，日志需强调通用性和一致性：

##### 7.4.1 向量数据库客户端日志（映射原则：E-1 基础设施）
```go
// VectorDB客户端日志 - 体现工程观（E-1）和认知观（C-3）原则
//
// 性能关键操作使用分级日志，避免影响查询延迟。
// 结构化日志支持容量规划和性能分析。
type VectorDBClient struct {
	logger *zap.Logger
	metrics *MetricsCollector
}

func (c *VectorDBClient) Search(query Vector, k int) ([]SearchResult, error) {
	startTime := time.Now()
	
	// 查询开始日志（调试级别）
	c.logger.Debug("Vector search started",
		zap.Int("k", k),
		zap.Int("dimensions", len(query)),
		zap.String("query_hash", hashVector(query)),
	)
	
	results, err := c.index.Search(query, k)
	duration := time.Since(startTime)
	
	// 性能日志（信息级别）
	c.logger.Info("Vector search completed",
		zap.Int("results_count", len(results)),
		zap.Duration("duration", duration),
		zap.Float64("qps", 1.0/duration.Seconds()),
		zap.Error(err),
	)
	
	// 慢查询警告
	if duration > SlowQueryThreshold {
		c.logger.Warn("Slow vector search",
			zap.Duration("duration", duration),
			zap.Int("k", k),
			zap.String("index_type", c.index.Type()),
		)
	}
	
	// 指标收集
	c.metrics.RecordSearch(duration, len(results), err == nil)
	
	return results, err
}
```

---
### 附录 B：与其他规范的引用关系

| 引用文档 | 关系说明 |
|---------|---------|
| [架构设计原则](../../00-architectural-principles.md) | 本规范是原则 E-2（可观测性原则）和 E-4（运维友好原则）在日志打印方面的具体实施 |
| [C&C++安全编程指南](./02-c-cpp-secure-coding.md) | 日志中的错误处理应遵循安全编程指南的异常处理规范 |
| [统一术语表](../90-terminology.md) | 本规范使用的术语定义和解释 |
| [Airymax 核心架构文档](../../10-architecture/) | 与本规范密切相关的架构文档：<br>- logging_system.md（可观测性核心架构）<br>- coreloopthree.md（运行时日志）<br>- memoryrovol.md（内存相关日志）<br>- microcorert.md（内核日志） |

---

### 附录 C：快速索引

#### 按规则编号

- **0-1** 日志层级与对应宏
- **0-2** HiLog 与 Airymax 日志系统的关系
- **0-3** 结构化 JSON 日志格式
- **0-4** trace_id / request_id 传播规范
- **1-1** 信噪比最大化
- **1-2** 分层观测
- **1-3** 自适应调节
- **3-1** 语言规范
- **3-2** 禁止隐私信息
- **3-3** 禁止无关信息
- **3-4** 避免重复
- **3-5** 禁止业务调用
- **4-1** 高频代码禁打日志
- **4-2** 频率限制
- **4-3** 关键点必打
- **4-4** 延迟生成
- **4-5** 避免长日志
- **5-1** 独立 Domain
- **5-2** 层次化分配
- **5-3** 阈值管理
- **5-4** 隐私标识

#### 按主题分类

**日志体系总览**：0.1-0.4
**日志级别**：2.1 节（含 AUDIT 审计级）
**内容规范**：3.1-3.5  
**打印策略**：4.1-4.5  
**HiLog 接口**：5.1-5.4  

---

### 附录：跨文档规范引用

本规范与以下 Airymax 工程规范一致，所有日志打印须同时遵循：

| 规范集 | 说明 | 来源文档 |
|--------|------|---------|
| **BAN-01~13** | 13 项禁止模式（桩函数/假数据/空返回等） | [C编码规范 §18](./C_Cpp_coding_style.md) |
| **REQ-01~08** | 8 项强制规范 | [C编码规范 §1.2](./C_Cpp_coding_style.md) |
| **标准术语** | 8 个架构组件标准名称 | [90-terminology.md](../90-terminology.md) |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.  
"From data intelligence emerges."
**质量管理**：6.1-6.3

---

## Part IV: Airymax 命名规范 (P4.4.2)

本文档定义了 Airymax 项目中所有关键组件、文件、函数、类型、常量、错误码及 API 的命名约定，确保整个代码库的一致性与可读性。

---

### 1. 核心组件命名

#### 1.1 Airymax

**Airymax**（极境）是产品名。**agentrt/** 是代码仓库目录路径，为 **Agent** **R**un**t**ime（即"Agent 运行时"）的缩写。Airymax 是一个 OS 级的 AI Agent 运行时平台，为 AI Agent 提供操作系统级别的调度、资源管理和生命周期管理能力。

"RT" 的命名参照了传统操作系统中的"Runtime"概念（如 Windows Runtime / WinRT），强调 Airymax 不是上层框架，而是位于操作系统层面的基础运行时。

#### 1.2 agentrt

**agentrt** = **agent** + **rt**（Runtime，全小写，无空格），即 Airymax 内部的"Agent 运行时层"。它是整个 Airymax 平台的核心基础设施层，提供进程管理、内存管理、IPC 通信、文件系统等底层能力。

- 所有 C 源代码文件位于 `agentrt/` 目录下
- 代码中的命名空间前缀统一使用 `airy_`（如 `airy_err_t`、`airy_log_write`）
- **注意**：代码仓库目录、子目录名与代码前缀统一使用小写（`agentrt/`、`agentrt/atoms/`、`airy_`），与产品名 Airymax（PascalCase）形成层次区分

#### 1.3 Atoms

**Atoms** = 原子（复数），即"原子构建块"。Atoms 是 Airymax 中最小的计算单元，代表不可再分的原子操作。

- 每个 Atom 是一个独立的、无副作用的计算单元
- 多个 Atoms 可以组合成更复杂的 TaskFlow
- 命名灵感来源于函数式编程中的原子操作概念

#### 1.4 CoreKern

**CoreKern** = **Core** + **Kern**el，即"原子核心"。它是 Airymax 的中央调度器和分发器（Central Scheduler and Dispatcher）。

- 采用微核心（MicroCoreRT）架构设计
- 负责所有 Cupolas 服务的生命周期管理
- 实现 CoreLoopThree 认知循环的调度
- 目录名为 `corekern/`（小写），代码前缀为 `airy_core_*`
- **注**: MicroCoreRT（微核心）是 CoreKern 的架构概念层名称，两者共享同一目录和代码前缀，非独立模块

#### 1.5 CoreLoopThree

**CoreLoopThree** = **Core** + **Loop** + **Three**，即"三层认知循环"。它描述了 Airymax 的认知处理流程：

```
Cognition（认知）→ Planning（规划）→ Action（执行）
```

- 第一阶段 **Cognition**：感知环境、理解上下文、提取意图
- 第二阶段 **Planning**：制定策略、分解任务、资源预估
- 第三阶段 **Action**：执行操作、调用工具、产生输出
- 循环往复，形成持续运行的认知回路

#### 1.6 Cupolas

**Cupolas** = 穹顶（复数），即"安全穹顶"。命名灵感来源于大教堂建筑顶部的穹顶结构（Cupola），寓意每个 Cupolas 是一个独立、自包含、向上拱起的安全服务单元。

- 每个 Cupolas 是一个独立的服务进程，通过 IPC Bus 与 CoreKern 通信
- Cupolas 之间相互隔离，通过消息传递进行协作
- 典型的 Cupolas 包括：cogn_d、dev_d、mem_d、logger_d 等
- 目录名为 `cupolas/`（小写），代码前缀为 `cupolas_`

#### 1.7 TaskFlow

**TaskFlow** = **Task** + **Flow**，即"任务编排流水线"。它表示任务在系统中的流动和处理过程。

- 任务从入口经过 CoreKern 调度，流经各个 Cupolas 服务
- 每个阶段对任务进行特定的加工和处理
- 支持串行、并行、条件分支等编排模式
- 目录名为 `taskflow/`（小写），代码前缀为 `taskflow_`

#### 1.8 HeapStore

**HeapStore** = **Heap** + **Store**，即"堆式持久化存储"。它结合了"堆"（Heap，动态内存）和"存储"（Store，持久化）两个概念。

- 提供动态分配的持久化存储能力
- 支持内存堆和磁盘存储的透明切换
- 自动管理数据的序列化和反序列化
- 目录名为 `heapstore/`（小写），代码前缀为 `heapstore_`

#### 1.9 MemoryRovol

**MemoryRovol** = **Memory** + **Rovol**（**Roll** + **Volume** 的融合造词），即"记忆卷载"。它实现了 Airymax 的 4 层记忆系统：

| 层级 | 名称 | 说明 |
|------|------|------|
| L1 | Raw（原始记忆） | 未经处理的原始输入数据 |
| L2 | Feature（特征记忆） | 从原始数据中提取的关键特征 |
| L3 | Structure（结构记忆） | 特征之间的结构化关系 |
| L4 | Pattern（模式记忆） | 高层次的行为模式和知识 |

- 目录名为 `memoryrovol/`（小写），代码前缀为 `memoryrovol_`

#### 1.10 ARE 标准接口前缀（`are_*`）

> **SSoT 声明**：本节补充 ARE（Airymax Runtime Environment）标准接口的命名契约，消除 `are_*` 前缀在命名规范中的缺失。

**ARE** = **A**irymax **R**untime **E**nvironment，是 Airymax 对外公开的**标准化运行时接口层**。`are_*` 前缀用于 L1/L2/L3 标准接口（见 `50-engineering-standards/30-runtime-interfaces/`），确保第三方可在任意 POSIX/Windows 平台上实现本接口集，无源码改动接入 ARE 生态。

| 前缀 | 层次 | 用途 | ABI 稳定性 | 桥接关系 |
|------|------|------|-----------|---------|
| `are_*` | L1/L2/L3 标准接口 | 对外公开的可移植 API（如 `are_ipc_send()`、`are_mem_alloc()`） | 同一 MAJOR 版本内 ABI 兼容 | 标准层——由适配层桥接至原生实现 |
| `airy_*` | 原生实现（通用） | Airymax atoms 层原生符号（如 `airy_log_write`、`airy_err_t`） | 内部 API，不保证稳定 | 实现层——`are_*` 的参考实现 |
| `airy_core_*` | 原生实现（CoreKern 专属） | CoreKern 模块专属函数（如 `airy_core_start()`、`airy_core_stop()`） | 内部 API，不保证稳定 | 实现层——CoreKern 专属原生符号 |

**桥接契约**：`are_*` 标准接口与 `airy_*`/`airy_core_*` 原生实现之间通过**适配层（adaptation layer）**桥接。适配层负责：

1. **符号映射**：`are_ipc_send()` → `airy_ipc_send()`（或 `airy_core_ipc_send()`）
2. **类型转换**：`are_error_t` ↔ `airy_err_t`（均为 `int32_t`，值空间一致）
3. **ABI 隔离**：`are_*` 接口的 ABI 稳定性由标准层保证；`airy_*` 内部 API 可自由变更，不影响 `are_*` 调用方

> **注意**：`are_*` 不是 `airy_*` 的别名——两者是不同的 API 层次。`are_*` 是面向第三方的标准接口，`airy_*`/`airy_core_*` 是 Airymax 内部的参考实现。第三方实现者可直接实现 `are_*` 接口，无需依赖 `airy_*` 实现。

---

### 2. 用户态服务（Daemon）命名

所有 Airymax 的后台用户态服务使用 `*_d` 后缀命名，遵循 Unix 用户态服务命名惯例（`d` = daemon）。

#### 2.1 完整 Daemon 列表

| Daemon 名 | 完整名称 | 职责描述 |
|-----------|---------|---------|
| `gateway_d` | Gateway Daemon | 外部请求入口，协议适配，认证鉴权，速率限制 |
| `cogn_d` | Cognition Daemon | LLM 推理调度，模型路由，Token 管理，流式输出 |
| `dev_d` | Device Daemon | 工具调用管理，工具注册表，执行沙箱，结果校验 |
| `sched_d` | Scheduler Daemon | 任务调度引擎，优先级队列，资源分配，负载均衡 |
| `mem_d` | Memory Daemon | 信息聚合服务，知识检索，上下文注入，数据转换 |
| `logger_d` | Logger Daemon | 可观测性服务，Metrics 采集，Tracing，健康检查 |
| `audit_d` | Audit Daemon | 监控告警服务，阈值检测，告警路由，自愈策略 |
| `gateway_d` | Gateway Daemon | 能力市场，计费计量，配额管理 |
| `net_d` | Network Daemon | 通道管理服务，长连接管理，流式通道，会话保持 |
| `sec_d` | Security Daemon | 通知推送服务，消息推送，多渠道分发，投递保证 |

#### 2.2 Daemon 命名规则

```
<service_name>_d
```

- 服务名使用小写英文单词，多个单词间用下划线分隔
- 统一以 `_d` 后缀结尾
- 对应的 C 源码文件使用相同的命名（如 `gateway_d.c`）

---

### 3. 文件命名

#### 3.1 C 源文件：snake_case

所有 C 源文件（`.c`）和头文件（`.h`）使用 **snake_case** 命名。

```
agentrt/commons/utils/observability/src/logger.c
agentrt/commons/utils/observability/include/logger.h
agentrt/commons/platform/include/platform.h
agentrt/commons/utils/quality/airy_quality.h
agentrt/commons/utils/config_unified/include/core_config.h
```

#### 3.2 目录名：PascalCase（顶层）/ snake_case（代码层）

- **顶层概念目录** 使用 PascalCase：`Docs/`、`Tests/`、`Scripts/`
- **代码层目录** 使用 snake_case：`agentrt/`、`commons/`、`protocols/`、`cupolas/`、`corekern/`

#### 3.3 配置文件：kebab-case

所有配置文件使用 **kebab-case**（小写字母 + 连字符）命名。

```
.env.example
.editorconfig
.gitattributes
```

#### 3.4 脚本文件：snake_case / kebab-case

Shell 脚本和 Python 脚本使用 snake_case 或 kebab-case：

```
scripts/ci/pipeline/run-tests.sh
scripts/ci/pipeline/security-scan.sh
scripts/ci/quality/check-quality.sh
scripts/dev/utils/run_all_fixes.sh
```

---

### 4. 函数命名

#### 4.1 基本规则：`module_action_object()`

所有函数遵循 `module_action_object()` 的命名规范，使用 snake_case。

```
<模块前缀>_<动作>_<对象>()
```

#### 4.2 代码示例

```c
// agentrt 模块 - 日志操作
const char *airy_log_set_trace_id(const char *trace_id);
const char *airy_log_get_trace_id(void);
void        airy_log_write(int level, const char *file, int line, const char *fmt, ...);

// config 模块 - 配置值操作
config_value_t *config_value_create_null(void);
config_value_t *config_value_create_bool(bool value);
config_value_t *config_value_create_int(int32_t value);
config_value_t *config_value_get_type(const config_value_t *value);
void            config_value_destroy(config_value_t *value);

// config 模块 - 配置上下文操作
config_context_t *config_context_create(const char *name);
void              config_context_destroy(config_context_t *ctx);
config_error_t    config_context_set(config_context_t *ctx, const char *key, config_value_t *value);
const config_value_t *config_context_get(const config_context_t *ctx, const char *key);

// heapstore 模块
int heapstore_init(const heapstore_config_t *config);
int heapstore_put(const char *key, const void *data, size_t size);
int heapstore_get(const char *key, void **data, size_t *size);

// CoreKern 模块（代码前缀 airy_core_）
int airy_core_start(void);
int airy_core_stop(void);
int airy_core_register_cupola(const char *name, cupola_entry_t entry);
```

#### 4.3 常见动作动词

| 动词 | 含义 | 示例 |
|------|------|------|
| `create` | 创建/分配资源 | `config_value_create_null` |
| `destroy` | 销毁/释放资源 | `config_value_destroy` |
| `get` | 获取值（不修改状态） | `config_value_get_type` |
| `set` | 设置值 | `config_context_set` |
| `init` | 初始化 | `heapstore_init` |
| `start` | 启动 | `airy_core_start` |
| `stop` | 停止 | `airy_core_stop` |
| `write` | 写入 | `airy_log_write` |
| `clone` | 克隆/深拷贝 | `config_value_clone` |
| `has` | 检查是否存在 | `config_context_has` |
| `delete` | 删除 | `config_context_delete` |
| `clear` | 清空 | `config_context_clear` |
| `count` | 计数 | `config_context_count` |
| `lock` | 加锁 | `config_context_lock` |
| `unlock` | 解锁 | `config_context_unlock` |

#### 4.4 函数前缀体系

> **SSoT 声明**：本节是 Airymax 函数前缀体系的**语义层权威来源**。三层 API 前缀架构（`are_*`/`airy_*`/`airy_core_*`）的契约定义见 §1.10；本节聚焦于模块前缀清单、项目隔离规则与选择决策。

##### 4.4.1 三层 API 前缀架构（回顾 §1.10）

Airymax 函数 API 分为三个层次，层次关系与 ABI 稳定性约定如下（完整定义见 §1.10）：

| 层次 | 前缀 | 用途 | ABI 稳定性 |
|------|------|------|-----------|
| 标准接口层 | `are_*` | L1/L2/L3 对外公开的可移植 API | 同一 MAJOR 版本内 ABI 兼容 |
| 原生实现层（通用） | `airy_*` | Airymax atoms 层原生符号 | 内部 API，不保证稳定 |
| 原生实现层（CoreKern） | `airy_core_*` | CoreKern 模块专属函数 | 内部 API，不保证稳定 |

**桥接契约**：`are_*` 标准接口与 `airy_*`/`airy_core_*` 原生实现之间通过**适配层**桥接。适配层负责符号映射（`are_ipc_send()` → `airy_ipc_send()`）、类型转换（`are_error_t` ↔ `airy_err_t`）与 ABI 隔离。

##### 4.4.2 项目隔离前缀规则

agentrt（用户态运行底座）与 agentrt-linux（智能体操作系统）共享 IRON-9 v3 四层模型，前缀按 IRON-9 v3 层次隔离：

| 前缀 | IRON-9 v3 层次 | 适用项目 | 用途 | 示例 |
|------|---------------|---------|------|------|
| `airy_*` | [SC] / [SS] | agentrt + agentrt-linux 共享 | 同源 API 与共享契约层符号 | `airy_ipc_send()`、`airy_log_write()` |
| `AIRY_*` | [SC] | agentrt + agentrt-linux 共享 | 共享宏与常量 | `AIRY_API`、`AIRY_SYS_CALL` |
| `AIRYMAXOS_*` | [IND] | agentrt-linux 专属 | 内核态/发行版专属宏与常量 | `AIRYMAXOS_CONFIG_*` |

**选择规则**：

1. **同源符号**（[SC]/[SS] 层）使用 `airy_*` 前缀——两端代码 byte-identical 或语义同源
2. **agentrt-linux 专属符号**（[IND] 层）使用 `airy_*` 前缀（应用层函数）或 `AIRYMAXOS_*`（内核态/发行版专属宏与常量）——对齐 06-iron9 §3.12：[SC] 共享契约层与应用层共用 `airy_*` 前缀，体现"共享契约即应用基础"的设计理念
3. **标准 ABI 层使用 `are_*` 前缀**——对外发布的稳定 ABI 接口（SDK 公开符号）使用 `are_` 前缀（**ARE** = AiRtymax Engine/ABI）；`are_*` 不得用于 [SC]/[SS] 共享层或应用层内部实现
4. **禁止使用 `agentrt_*` 前缀**——仓库名 `agentrt` 是工程组织名称，不应作为函数前缀（06-iron9 §3.12）
5. **`agentos_*` / `airymaxos_*` 已废弃**——0.1.1 完成改名后已统一改用 `airy_*`，禁止使用（06-iron9 §3.12）

##### 4.4.3 模块前缀清单总表

> **SSoT 声明**：本节为 Airymax 项目**模块前缀注册表**的唯一权威来源（SSoT）。所有模块前缀必须在本节注册后方可使用（注册流程见 §4.4.6）。其他文档（包括 `01-coding-standards.md`、`07-maintainers-and-governance.md`）仅可引用本节编号，**禁止重新定义**模块前缀。

`airy_*` 通用前缀通过 **2-4 字符模块子前缀**区分子系统。模块子前缀遵循 `airy_<module>_` 格式，其中 `<module>` 标识所属子系统。下表按功能域分组列出全部已注册模块前缀：

**A. 核心运行时层（Core Runtime）**

| 模块前缀 | 所属模块 | IRON-9 v3 | 示例 |
|---------|---------|-----------|------|
| `airy_core_*` | CoreKern / MicroCoreRT | [SS] | `airy_core_start()`、`airy_core_register_cupola()` |
| `airy_sys_*` | 系统调用路由层 | [SS] | `airy_sys_task_submit()`、`airy_sys_memory_write()` |
| `airy_ipc_*` | AgentsIPC 进程间通信 | [SS] | `airy_ipc_send()`、`airy_ipc_recv()` |
| `airy_sched_*` | 调度器通用接口 | [SS] | `airy_sched_enqueue()`、`airy_sched_yield()` |
| `airy_sched_agent_*` | Agent 调度策略（sched_tac） | [SS] | `airy_sched_agent_compute_weight()` |
| `airy_task_*` | 任务管理 | [SS] | `airy_task_create()`、`airy_task_destroy()` |
| `airy_taskflow_*` | TaskFlow 任务编排 | [SS] | `taskflow_graph_build()`、`taskflow_execute()` |

**B. 认知与智能层（Cognition & Intelligence）**

| 模块前缀 | 所属模块 | IRON-9 v3 | 示例 |
|---------|---------|-----------|------|
| `airy_cognition_*` | 认知引擎 | [SS] | `airy_cognition_perceive()`、`airy_cognition_act()` |
| `airy_loop_*` | CoreLoopThree 三层认知循环 | [SS] | `airy_loop_run()`、`airy_loop_next_phase()` |
| `airy_thinking_*` | 思考引擎 | [SS] | `airy_thinking_fast()`、`airy_thinking_slow()` |
| `airy_intent_*` | 意图识别 | [SS] | `airy_intent_extract()`、`airy_intent_classify()` |
| `airy_plan_*` | 规划引擎 | [SS] | `airy_plan_create()`、`airy_plan_execute()` |
| `airy_llm_*` | LLM 推理 | [SS] | `airy_llm_infer()`、`airy_llm_route()` |
| `airy_prompt_*` | Prompt 模板加载 | [SS] | `airy_prompt_loader_init()`、`airy_prompt_render()` |
| `airy_token_*` | Token 管理 | [SS] | `airy_token_count()`、`airy_token_budget()` |

**C. 双思考系统（Thinkdual）**

| 模块前缀 | 所属模块 | IRON-9 v3 | 示例 |
|---------|---------|-----------|------|
| `tc_*` | Thinkdual Core | [SS] | `tc_select_mode()`、`tc_switch()` |
| `mc_*` | Multi-agent Cognition | [SS] | `mc_coordinate()`、`mc_dispatch()` |
| `sc_*` | Slow Cognition（System 2） | [SS] | `sc_deliberate()`、`sc_analyze()` |

**D. 安全与隔离层（Security & Isolation）**

| 模块前缀 | 所属模块 | IRON-9 v3 | 示例 |
|---------|---------|-----------|------|
| `airy_cap_*` | Capability 令牌系统 | [SS] | `airy_cap_mint()`、`airy_cap_revoke()` |
| `airy_sandbox_*` | 沙箱隔离 | [SS] | `airy_sandbox_create()`、`airy_sandbox_enter()` |
| `airy_security_*` | 安全策略 | [SS] | `airy_security_check()`、`airy_security_enforce()` |
| `cupolas_*` | Cupolas 安全穹顶服务 | [SS] | `cupolas_init()`、`cupolas_config_load()` |
| `cupolas_sig_*` | Cupolas 签名验证 | [SS] | `cupolas_sig_verify()`、`cupolas_sig_generate()` |
| `cupolas_net_*` | Cupolas 网络安全 | [SS] | `cupolas_net_check()`、`cupolas_net_block()` |
| `cupolas_ent_*` | Cupolas 权限授权 | [SS] | `cupolas_ent_check()`、`cupolas_ent_grant()` |

**E. 记忆与存储层（Memory & Storage）**

| 模块前缀 | 所属模块 | IRON-9 v3 | 示例 |
|---------|---------|-----------|------|
| `airy_mem_*` | 内存管理通用 | [SS] | `airy_mem_alloc()`、`airy_mem_free()` |
| `airy_mempool_*` | 内存池 | [SS] | `airy_mempool_create()`、`airy_mempool_alloc()` |
| `airy_slab_*` | Slab 分配器 | [SS] | `airy_slab_create()`、`airy_slab_alloc()` |
| `airy_arena_*` | Arena 分配器 | [SS] | `airy_arena_create()`、`airy_arena_reset()` |
| `airy_tcache_*` | 线程本地缓存 | [SS] | `airy_tcache_get()`、`airy_tcache_put()` |
| `airy_mr_*` | MemoryRovol 记忆卷载 | [SS] | `airy_mr_init()`、`airy_mr_query()` |
| `airy_checkpoint_*` | 检查点与快照 | [SS] | `airy_checkpoint_save()`、`airy_checkpoint_restore()` |
| `airy_oom_*` | OOM 处理 | [SS] | `airy_oom_handle()`、`airy_oom_register_cb()` |
| `heapstore_*` | HeapStore 堆式持久化存储 | [SS] | `heapstore_init()`、`heapstore_put()` |
| `memoryrovol_*` | MemoryRovol（模块直称） | [SS] | `memoryrovol_init()`、`memoryrovol_query()` |

**F. 平台与运行时基础设施（Platform & Runtime Infrastructure）**

| 模块前缀 | 所属模块 | IRON-9 v3 | 示例 |
|---------|---------|-----------|------|
| `airy_platform_*` | 平台抽象层 | [SS] | `airy_platform_thread_create()`、`airy_platform_thread_join()` |
| `airy_thread_*` | 线程管理 | [SS] | `airy_thread_id()`、`airy_thread_sleep()` |
| `airy_mtx_*` | 互斥锁 | [SS] | `airy_mtx_init()`、`airy_mtx_lock()` |
| `airy_cond_*` | 条件变量 | [SS] | `airy_cond_init()`、`airy_cond_wait()` |
| `airy_rwlock_*` | 读写锁 | [SS] | `airy_rwlock_rdlock()`、`airy_rwlock_wrlock()` |
| `airy_sem_*` | 信号量 | [SS] | `airy_sem_init()`、`airy_sem_post()` |
| `airy_atomic_*` | 原子操作 | [SS] | `airy_atomic_load()`、`airy_atomic_store()` |
| `airy_time_*` | 时间工具 | [SS] | `airy_time_now()`、`airy_time_diff()` |
| `airy_timer_*` | 定时器 | [SS] | `airy_timer_create()`、`airy_timer_start()` |
| `airy_uuid_*` | UUID 生成 | [SS] | `airy_uuid_generate()`、`airy_uuid_parse()` |

**G. 网络与通信层（Network & Communication）**

| 模块前缀 | 所属模块 | IRON-9 v3 | 示例 |
|---------|---------|-----------|------|
| `airy_sock_*` | Socket 抽象 | [SS] | `airy_sock_create()`、`airy_sock_connect()` |
| `airy_net_*` | 网络管理 | [SS] | `airy_net_init()`、`airy_net_cleanup()` |
| `airy_http_*` | HTTP 客户端/服务端 | [SS] | `airy_http_get()`、`airy_http_post()` |
| `airy_conn_*` | 连接管理 | [SS] | `airy_conn_create()`、`airy_conn_close()` |

**H. 可观测性与运维（Observability & Operations）**

| 模块前缀 | 所属模块 | IRON-9 v3 | 示例 |
|---------|---------|-----------|------|
| `airy_log_*` | 日志系统 | [SS] | `airy_log_write()`、`airy_log_set_trace_id()` |
| `airy_trace_*` | 链路追踪 | [SS] | `airy_trace_start()`、`airy_trace_span()` |
| `airy_metrics_*` | 指标采集 | [SS] | `airy_metrics_inc()`、`airy_metrics_set()` |
| `airy_observability_*` | 可观测性聚合 | [SS] | `airy_observability_init()`、`airy_observability_export()` |
| `airy_telemetry_*` | 遥测数据 | [SS] | `airy_telemetry_metrics()`、`airy_telemetry_traces()` |
| `airy_event_*` | 事件系统 | [SS] | `airy_event_publish()`、`airy_event_subscribe()` |
| `airy_hook_*` | Hook 系统 | [SS] | `airy_hook_register()`、`airy_hook_unregister()` |

**I. 配置与工具（Configuration & Tools）**

| 模块前缀 | 所属模块 | IRON-9 v3 | 示例 |
|---------|---------|-----------|------|
| `airy_config_*` | Airymax 配置框架 | [SS] | `airy_config_load()`、`airy_config_get()` |
| `config_*` | Config 模块（历史保留） | [SS] | `config_value_create()`、`config_context_get()` |
| `airy_yaml_*` | YAML 解析 | [SS] | `airy_yaml_parse()`、`airy_yaml_dump()` |
| `airy_cjson_*` | cJSON 封装 | [SS] | `airy_cjson_parse()`、`airy_cjson_print()` |
| `airy_validate_*` | 验证工具 | [SS] | `airy_validate_string()`、`validate_schema()` |
| `airy_version_*` | 版本管理 | [SS] | `airy_version_get()`、`airy_version_check()` |

**J. 网关与协议（Gateway & Protocol）**

| 模块前缀 | 所属模块 | IRON-9 v3 | 示例 |
|---------|---------|-----------|------|
| `airy_gw_*` | Gateway 网关（短前缀） | [SS] | `airy_gw_route()`、`airy_gw_authenticate()` |
| `airy_gateway_*` | Gateway 网关（全称） | [SS] | `airy_gateway_init()`、`airy_gateway_start()` |
| `airy_proto_*` | 协议适配 | [SS] | `airy_proto_init()`、`airy_proto_handle_request()` |
| `airy_protocol_*` | 协议扩展框架 | [SS] | `airy_protocol_register()`、`airy_protocol_invoke()` |
| `airy_router_*` | 路由引擎 | [SS] | `airy_router_select_provider()` |
| `airy_provider_*` | Provider 管理 | [SS] | `airy_provider_register()`、`airy_provider_list()` |

**K. Agent 管理与执行（Agent Management & Execution）**

| 模块前缀 | 所属模块 | IRON-9 v3 | 示例 |
|---------|---------|-----------|------|
| `airy_agent_*` | Agent 生命周期 | [SS] | `airy_agent_spawn()`、`airy_agent_terminate()` |
| `airy_session_*` | 会话管理 | [SS] | `airy_session_create()`、`airy_session_close()` |
| `airy_daemon_*` | Daemon 管理 | [SS] | `airy_daemon_start()`、`airy_daemon_stop()` |
| `airy_browser_*` | 浏览器自动化 | [SS] | `airy_browser_launch()`、`airy_browser_close()` |
| `airy_execution_*` | 执行单元 | [SS] | `airy_execution_unit_create()` |
| `airy_dispatcher_*` | 任务分发 | [SS] | `airy_dispatcher_select()`、`airy_dispatcher_dispatch()` |
| `airy_coordinator_*` | 多智能体协调 | [SS] | `airy_coordinator_register()`、`airy_coordinator_dispatch()` |
| `airy_delegate_*` | 委托执行 | [SS] | `airy_delegate_create()`、`airy_delegate_invoke()` |
| `airy_compensation_*` | 补偿事务 | [SS] | `airy_compensation_register()`、`airy_compensation_rollback()` |
| `airy_resource_*` | 资源管理 | [SS] | `airy_resource_alloc()`、`airy_resource_free()` |
| `airy_context_*` | 上下文管理 | [SS] | `airy_context_create()`、`airy_context_switch()` |
| `airy_error_*` / `airy_err_*` | 错误处理 | [SS] | `airy_error_to_string()`、`airy_err_t` |

##### 4.4.4 函数名长度约束

| 约束项 | 上限 | 说明 |
|--------|------|------|
| 模块子前缀 | ≤ 12 字符 | `airy_` (5) + 模块名 (≤ 7) = ≤ 12；如 `airy_sched_agent_` 属例外（长前缀仅用于sched_tac 用户态调度器策略） |
| 函数动作部分 | ≤ 15 字符 | `<action>_<object>` 组合长度 ≤ 15 字符 |
| 完整函数名总长 | ≤ 30 字符 | 模块前缀 + 动作部分 ≤ 30 字符 |
| 超长处理 | 缩写或分层 | 超过 30 字符时拆分为辅助函数或使用约定缩写 |

**长度计算示例**：

```
airy_ipc_send                  → 5 + 3 + 4  = 12 字符 ✓
airy_core_register_cupola      → 5 + 4 + 14 = 23 字符 ✓
airy_sched_agent_compute_weight → 5 + 11 + 14 = 30 字符 ✓（边界）
airy_mr_checkpoint_restore    → 5 + 3 + 18 = 26 字符 ✓
```

**超长函数名修正策略**：

1. **拆分**：将复合动作拆分为两个函数（如 `airy_mr_restore_checkpoint()` → `airy_mr_restore()`）
2. **缩写**：使用约定缩写（如 `init` 代替 `initialize`、`cfg` 代替 `config`、`ctx` 代替 `context`）
3. **分层**：引入中间层函数（如 `airy_mem_ckpt_save()` 代替 `airy_memory_checkpoint_save()`）

##### 4.4.5 前缀选择决策规则

新增函数时，按以下顺序确定前缀：

1. **是否为对外公开标准接口？**
   - 是 → 使用 `are_*` 前缀（§1.10），并通过适配层桥接至 `airy_*` 实现
   - 否 → 进入下一步

2. **是否为 agentrt-linux 内核态/发行版专属功能？**
   - 是 → 使用 `airy_*` 前缀（[IND] 层）
   - 否 → 进入下一步

3. **是否为 CoreKern 模块专属函数？**
   - 是 → 使用 `airy_core_*` 前缀
   - 否 → 进入下一步

4. **选择模块子前缀**
   - 从 §4.4.3 模块前缀清单总表中选择对应功能域的前缀
   - 若功能域跨多个模块，选择最贴近的主模块前缀
   - 若无对应前缀，向工程规范委员会申请注册新前缀（流程见 §4.4.6）

5. **验证长度约束**
   - 按 §4.4.4 规则验证完整函数名 ≤ 30 字符
   - 超长时按修正策略调整

##### 4.4.6 新前缀注册流程

新增模块前缀须经过以下流程注册：

1. **RFC**：在 GitHub Discussions 发起 RFC，说明前缀字符串、所属模块、IRON-9 v3 层次、适用范围
2. **评审**：经至少一名顶级子系统维护者审查，公示 14 天接受异议
3. **注册**：通过评审后，由工程规范委员会将前缀写入本节 §4.4.3 模块前缀清单总表，并同步更新 [90-terminology.md 标准化名称映射总表](../90-terminology.md#四标准化名称映射总表)

##### 4.4.7 前缀使用禁忌

| 禁忌 | 说明 | 正确做法 |
|------|------|---------|
| 混用 `airy_` 与 `airy_` | 同一功能在两端使用不同前缀导致符号冲突 | [SC]/[SS] 层用 `airy_`，[IND] 层用 `airy_` |
| 省略模块子前缀 | `airy_send()` 无法识别所属模块 | 必须带模块子前缀：`airy_ipc_send()` |
| 使用未注册前缀 | `airy_foo_*` 未在清单中注册 | 先按 §4.4.6 注册，再使用 |
| 模块前缀过长 | `airy_internationalization_*` 超长 | 缩写为 `airy_i18n_*` 并注册 |
| `are_*` 与 `airy_*` 混淆 | 将内部实现函数标记为 `are_*` | `are_*` 仅用于对外公开标准接口 |
| Thinkdual 前缀误用 | `tc_`/`mc_`/`sc_` 用于非 Thinkdual 模块 | Thinkdual 前缀仅限双思考系统使用 |

---

### 5. 类型命名

#### 5.1 基本规则：`module_type_t`

所有类型定义遵循 `module_type_t` 的命名规范，使用 snake_case，以 `_t` 后缀结尾。

```c
<模块前缀>_<类型名>_t
```

#### 5.2 代码示例

```c
// 基础类型
typedef int32_t airy_err_t;       // agentrt 错误码类型

// 结构体类型
typedef struct config_value    config_value_t;     // 配置值
typedef struct config_context  config_context_t;   // 配置上下文
typedef struct config_schema   config_schema_t;    // 配置模式

// 枚举类型
typedef enum {
    CONFIG_TYPE_NULL = 0,
    CONFIG_TYPE_BOOL = 1,
    CONFIG_TYPE_INT  = 2,
    // ...
} config_value_type_t;

typedef enum {
    CONFIG_SUCCESS = 0,
    CONFIG_ERROR_INVALID_ARG = 1,
    // ...
} config_error_t;

// heapstore 模块类型
typedef struct heapstore_config heapstore_config_t;
typedef struct heapstore_entry  heapstore_entry_t;

// memoryrovol 模块类型
typedef struct memoryrovol_layer memoryrovol_layer_t;
typedef enum   memoryrovol_level memoryrovol_level_t;
```

#### 5.3 枚举值命名

枚举值使用 `MODULE_UPPER_SNAKE_CASE` 格式：

```c
typedef enum {
    AIRY_MEM_LAYER1_RAW = 0,       // 注意：不是全大写，而是模块前缀 + 含义
    AIRY_MEM_LAYER2_FEATURE = 1,
    AIRY_MEM_LAYER3_EPISODIC = 2,
    AIRY_MEM_LAYER4_PATTERN = 3,
} airy_memory_layer_t;

typedef enum {
    CONFIG_TYPE_NULL = 0,
    CONFIG_TYPE_BOOL = 1,
    CONFIG_TYPE_INT = 2,
    CONFIG_TYPE_INT64 = 3,
    CONFIG_TYPE_DOUBLE = 4,
    CONFIG_TYPE_STRING = 5,
    CONFIG_TYPE_ARRAY = 6,
    CONFIG_TYPE_OBJECT = 7,
    CONFIG_TYPE_BINARY = 8,
} config_value_type_t;
```

---

### 6. 常量与宏命名

#### 6.1 宏定义：UPPER_SNAKE_CASE

所有预处理器宏使用 **UPPER_SNAKE_CASE**（全大写 + 下划线分隔）。

```c
// 日志级别宏
#define AIRY_LOG_LEVEL_DEBUG 0
#define AIRY_LOG_LEVEL_INFO  1
#define AIRY_LOG_LEVEL_WARN  2
#define AIRY_LOG_LEVEL_ERROR 3
#define AIRY_LOG_LEVEL_FATAL 4

// 平台检测宏
#define AIRY_PLATFORM_LINUX   1
#define AIRY_PLATFORM_MACOS   1
#define AIRY_PLATFORM_WINDOWS 1
#define AIRY_PLATFORM_NAME    "Linux"
#define AIRY_PLATFORM_BITS    64
#define AIRY_PLATFORM_POSIX   1

// 质量保证宏
#define AIRY_CHECK_NULL(ptr, error_code)       ...
#define AIRY_CHECK_NULL_GOTO(ptr, label, code) ...
#define AIRY_CHECK_CONDITION(cond, error_code) ...
#define AIRY_CHECK_RANGE(value, min, max, err) ...

// 头文件保护宏
#define AIRY_TYPES_H
#define AIRY_UTILS_LOGGER_H
#define AIRY_CORE_CONFIG_H
#define AIRY_PLATFORM_H
#define AIRY_QUALITY_H
```

#### 6.2 模块前缀规则

所有宏必须以模块前缀开头，避免全局命名冲突：

```
AIRY_<类别>_<名称>
```

---

### 7. 错误码命名

#### 7.1 基本规则：`MODULE_ERR_CATEGORY`

错误码的命名遵循 `MODULE_ERR_CATEGORY` 的格式：

| 模块 | 格式 | 示例 |
|------|------|------|
| agentrt（通用） | `AIRY_E<CODE>` | `AIRY_EINVAL`、`AIRY_ENOMEM` |
| config | `CONFIG_ERROR_<NAME>` | `CONFIG_ERROR_INVALID_ARG`、`CONFIG_ERROR_NOT_FOUND` |

#### 7.2 通用错误码（agentrt 层）— 已废弃

> **⚠️ 本节已废弃（方案 C，v3.0 SSoT 统一收敛）**
>
> 本节原包含方案 C 错误码数值（-2/-4/-6 等自定义序列），与 AirymaxOS 错误码体系存在数值冲突与不一致。
> **v3.0 SSoT 统一收敛后，唯一权威方案为方案 A（POSIX errno 负值）**，权威定义见：
> - [跨项目代码共享 §2.5](../120-cross-project-code-sharing.md)（SSoT 声明）
> - `agentrt/commons/include/airy_types.h`（`airy_err_t` 类型 + `AIRY_E*` POSIX 码）
> - `agentrt/commons/utils/error/include/error.h`（`AIRY_ERR_*` 扩展码）
>
> 以下为方案 A 权威值（POSIX errno 负值），**禁止在新代码中使用方案 C 的旧数值**：

```c
// 方案 A 权威值（POSIX errno 负值），定义于 agentrt/commons/include/airy_types.h
#define AIRY_EOK         (0)     // 成功
#define AIRY_EPERM       (-1)    // 权限不足（POSIX EPERM=1）
#define AIRY_ENOENT      (-2)    // 资源不存在（POSIX ENOENT=2）
#define AIRY_EIO         (-5)    // I/O 错误（POSIX EIO=5）
#define AIRY_EAGAIN      (-11)   // 资源暂时不可用（POSIX EAGAIN=11）
#define AIRY_ENOMEM      (-12)   // 内存不足（POSIX ENOMEM=12）
#define AIRY_EACCES      (-13)   // 权限拒绝（POSIX EACCES=13）
#define AIRY_EBUSY       (-16)   // 资源忙碌（POSIX EBUSY=16）
#define AIRY_EEXIST      (-17)   // 资源已存在（POSIX EEXIST=17）
#define AIRY_EINVAL      (-22)   // 参数无效（POSIX EINVAL=22）
#define AIRY_ENOSPC      (-28)   // 空间不足（POSIX ENOSPC=28）
#define AIRY_ERANGE      (-34)   // 数值范围错误（POSIX ERANGE=34）
#define AIRY_EDEADLK     (-35)   // 死锁（POSIX EDEADLK=35）
#define AIRY_EPROTO      (-71)   // 协议错误（POSIX EPROTO=71）
#define AIRY_EOVERFLOW   (-75)   // 溢出错误（POSIX EOVERFLOW=75）
#define AIRY_EMSGSIZE    (-90)   // 消息过长（POSIX EMSGSIZE=90）
#define AIRY_ENOTSUP     (-95)   // 操作不支持（POSIX ENOTSUP=95）
#define AIRY_ECONNRESET  (-104)  // 连接重置（POSIX ECONNRESET=104）
#define AIRY_ENOTCONN    (-107)  // 未连接（POSIX ENOTCONN=107）
#define AIRY_ETIMEDOUT   (-110)  // 操作超时（POSIX ETIMEDOUT=110）
#define AIRY_ECONNREFUSED (-111) // 连接被拒绝（POSIX ECONNREFUSED=111）
// 以下为自定义错误码（无 POSIX 对应，保留自定义负值）
#define AIRY_ENOTINIT     (-9)   // 引擎未初始化（自定义）
#define AIRY_ECANCELLED   (-10)  // 操作已取消（自定义）
#define AIRY_EPLATFORM    (-27)  // 平台未初始化（自定义）
#define AIRY_EPROTONOSUPPORT (-93) // 协议/命令不支持（POSIX EPROTONOSUPPORT=93）
#define AIRY_ESERVICE     (-29)  // 服务不可用（自定义）
#define AIRY_EUNKNOWN     (-99)  // 未知错误（自定义）
```

#### 7.3 模块级错误码（config 模块示例）

```c
typedef enum {
    CONFIG_SUCCESS = 0,              // 成功
    CONFIG_ERROR_INVALID_ARG = 1,    // 参数无效
    CONFIG_ERROR_NOT_FOUND = 2,      // 配置项不存在
    CONFIG_ERROR_TYPE_MISMATCH = 3,  // 类型不匹配
    CONFIG_ERROR_OUT_OF_MEMORY = 4,  // 内存不足
    CONFIG_ERROR_IO = 5,             // I/O 错误
    CONFIG_ERROR_PARSE = 6,          // 解析错误
    CONFIG_ERROR_VALIDATION = 7,     // 验证失败
    CONFIG_ERROR_LOCKED = 8,         // 配置被锁定
    CONFIG_ERROR_UNSUPPORTED = 9,    // 不支持的操作
    CONFIG_ERROR_THREAD = 10,        // 线程操作失败
} config_error_t;
```

#### 7.4 错误码设计原则

1. **通用层**（agentrt）使用类 POSIX 风格的宏定义（`AIRY_E*`），所有错误码为负值，`0` 表示成功
2. **模块层**（如 config）使用枚举类型，正值表示错误类别，`0` 表示成功
3. 错误码语义应与 POSIX errno 保持一致的直觉（如 `EINVAL` = 无效参数）
4. 每个模块的错误码转换为字符串的函数命名为 `module_error_to_string()`

---

### 8. API 版本化

#### 8.1 `@since` 标签

所有公开 API 在 Doxygen 注释中使用 `@since vX.Y.Z` 标签标注引入版本，遵循语义化版本（Semantic Versioning）。

#### 8.2 代码示例

```c
/**
 * @brief 创建空配置值
 * @return 配置值对象，失败返回 NULL
 * @since v0.1.1
 */
config_value_t *config_value_create_null(void);

/**
 * @brief 创建布尔配置值
 * @param value 布尔值
 * @return 配置值对象，失败返回 NULL
 * @since v0.1.1
 */
config_value_t *config_value_create_bool(bool value);

/**
 * @brief 获取配置上下文中的值
 * @param ctx 配置上下文
 * @param key 键名
 * @return 配置值对象，不存在时返回 NULL
 * @since v0.1.1
 */
const config_value_t *config_context_get(const config_context_t *ctx, const char *key);

/**
 * @brief 设置跨平台线程局部存储变量
 * @param key 变量键名
 * @param value 变量值
 * @return AIRY_EOK 成功，AIRY_ENOMEM 内存不足
 * @since v0.1.1
 */
int airy_tls_set(const char *key, void *value);
```

#### 8.3 版本号规则

- 主版本号（MAJOR）：不兼容的 API 修改
- 次版本号（MINOR）：向下兼容的功能新增
- 修订号（PATCH）：向下兼容的问题修正

---

### 9. 命名速查表

| 类别 | 规则 | 示例 |
|------|------|------|
| 平台名 | PascalCase | `Airymax` |
| 操作系统层 | 全小写 | `agentrt` |
| 核心组件 | PascalCase | `CoreKern`、`CoreLoopThree`、`TaskFlow`、`HeapStore`、`MemoryRovol` |
| 复数组件 | PascalCase（复数） | `Atoms`、`Cupolas` |
| 用户态服务 | snake_case + `_d` | `gateway_d`、`cogn_d`、`dev_d` |
| C 源文件 | snake_case | `logger.c`、`core_config.c` |
| 头文件 | snake_case | `logger.h`、`platform.h` |
| 代码目录 | snake_case | `agentrt/`、`commons/`、`protocols/` |
| 顶层目录 | PascalCase | `Docs/`、`Tests/`、`Scripts/` |
| 配置文件 | kebab-case | `.editorconfig`、`.env.example` |
| 函数 | `module_action_object()` | `heapstore_init`、`config_value_create` |
| 三层 API 前缀 | `are_*` / `airy_*` / `airy_core_*` | `are_ipc_send` / `airy_ipc_send` / `airy_core_start`（§1.10、§4.4） |
| 项目隔离前缀 | `airy_*`（[SC]/[SS]） / `airy_*`（[IND]） | `airy_ipc_send` / `airy_kthread_create`（§4.4.2） |
| 模块子前缀 | `airy_<module>_*` | `airy_ipc_*`、`airy_log_*`、`airy_mem_*`（§4.4.3） |
| 函数名长度 | ≤ 30 字符 | `airy_ipc_send`（12 字符）、`airy_sched_agent_compute_weight`（30 字符，边界）（§4.4.4） |
| 类型 | `module_type_t` | `airy_err_t`、`config_value_t` |
| 枚举 | `MODULE_UPPER` | `CONFIG_TYPE_INT`、`AIRY_MEM_LAYER1_RAW` |
| 宏 | `UPPER_SNAKE_CASE` | `AIRY_LOG_LEVEL_DEBUG`、`AIRY_CHECK_NULL` |
| 通用错误码 | `AIRY_E*` | `AIRY_EINVAL`、`AIRY_ENOMEM` |
| 模块错误码 | `MODULE_ERROR_*` | `CONFIG_ERROR_NOT_FOUND` |
| 头文件保护 | `UPPER_SNAKE_CASE` | `AIRY_TYPES_H`、`AIRY_PLATFORM_H` |
| API 版本 | `@since vX.Y.Z` | `@since v0.1.1` |

---

### 10. 常见错误与反模式

#### 10.1 应避免的命名

| 反模式 | 问题 | 正确写法 |
|--------|------|---------|
| `agentrtLogWrite` | 混用 camelCase，C 代码应使用 snake_case | `airy_log_write` |
| `ConfigValue` | C 类型不应使用 PascalCase | `config_value_t` |
| `AIRY_log_level` | 宏不应混用大小写 | `AIRY_LOG_LEVEL` |
| `gatewayDaemon` | Daemon 名应使用 `_d` 后缀 | `gateway_d` |
| `core_kern` | 组件名应使用 PascalCase | `CoreKern` |
| `heap-store.h` | C 头文件应使用 snake_case | `heapstore.h` |

#### 10.2 关键原则

1. **一致性优先**：当不确定时，参考已有代码中的命名模式
2. **模块前缀必须**：所有公开符号必须带模块前缀，避免链接冲突
3. **语义明确**：命名应准确表达其用途，避免缩写歧义（除非是广泛接受的缩写如 `init`、`cfg`）
4. **C 惯例为准**：由于 Airymax 核心以 C 语言实现，命名以 C 社区惯例为基准

---

## Part V: Airymax 安全设计指南

### 一、概述

#### 1.1 编制目的

本指南为 Airymax 项目提供全面的安全设计标准。基于项目架构设计原则的五维正交系统，本指南聚焦于安全观维度（D-1至D-4安全工程原则），阐述如何通过安全穹顶 (cupolas) 实现纵深防御。

#### 1.2 与 Airymax 架构的关系

基于架构设计原则**E-1（安全内生原则）**，Airymax 的安全穹顶（cupolas）采用四层防护体系：

| 层次 | 名称 | 组件路径 | 功能 | 实现机制 |
|------|------|---------|------|---------|
| D1 | **虚拟工位** | `agentrt/cupolas/workbench/` | 进程/容器级隔离 | 容器命名空间、WASM 沙箱、资源限额 |
| D2 | **权限裁决** | `agentrt/cupolas/permission/` | 动态规则引擎 | YAML 规则、RBAC、ABAC |
| D3 | **输入净化** | `agentrt/cupolas/sanitizer/` | 输入验证过滤 | 正则表达式、白名单、类型检查 |
| D4 | **审计追踪** | `agentrt/cupolas/audit/` | 全链路追踪 | 异步日志、不可篡改、轮转归档 |

**安全原则**（关联原则 E-1）:

| 原则 | 说明 | 在安全穹顶 (cupolas) 中的体现 | 关联原则 |
|------|------|---------------------|---------|
| 最小权限 | 只授予完成任务所需的最小权限 | D2 权限裁决 | K-1 |
| 纵深防御 | 多层安全检查 | D1-D4 四层防护 | S-2 |
| 默认安全 | 默认配置应是安全的 | 默认拒绝策略 | E-1 |
| 故障安全 | 发生错误时默认拒绝 | 安全机制故障时拒绝访问 | E-6 |
| 开放设计 | 安全性不依赖算法保密 | 公开算法和协议 | K-2 |
| 可观测性 | 所有安全事件必须可追踪 | D4 审计追踪 | E-2 |

本指南详细阐述每一层的设计原则和实现要求。

---

### 二、安全穹顶（cupolas）设计

#### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                     Airymax 系统边界                         │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              D1: 虚拟工位 (Virtual Workspace)           │  │
│  │  ┌───────────────────────────────────────────────────┐  │  │
│  │  │        D2: 权限裁决 (Permission Arbitration)      │  │  │
│  │  │  ┌─────────────────────────────────────────────┐  │  │  │
│  │  │  │      D3: 输入净化 (Input Sanitization)       │  │  │  │
│  │  │  │  ┌───────────────────────────────────────┐  │  │  │  │
│  │  │  │  │       D4: 审计日志 (Audit Logging)      │  │  │  │  │
│  │  │  │  │                                       │  │  │  │  │
│  │  │  │  └───────────────────────────────────────┘  │  │  │  │
│  │  │  └─────────────────────────────────────────────┘  │  │  │
│  │  └───────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

#### 2.2 D1: 虚拟工位

```yaml
# agentrt/cupolas/workspace.yaml
workspace:
  # 进程级隔离配置
  isolation:
    type: "container"  # process | container | vm
    resource_limits:
      memory_mb: 512
      cpu_shares: 256
      oom_score_adj: 500
    
  # 网络隔离
  network:
    mode: "none"  # none | bridge | host
    allowed_outbound:
      - "api.agentrt.internal"
      - "metrics.internal"
      
  # 文件系统隔离
  filesystem:
    root: "/var/agentrt/workspace/{workspace_id}"
    readonly_paths:
      - "/etc"
      - "/usr/bin"
    writable_paths:
      - "/tmp/agentrt"
```

```c
// agentrt/atoms/agentrt/cupolas/workspace.h
/**
 * @file workspace.h
 * @brief 虚拟工位接口
 *
 * 提供进程/容器级隔离，确保任务在受限环境中执行。
 */

typedef struct airy_workspace {
    uint64_t workspace_id;
    WorkspaceConfig manager;
    WorkspaceState state;
    
    // 资源限制
    struct {
        uint64_t memory_limit_bytes;
        uint32_t cpu_shares;
        int oom_score_adj;
    } limits;
    
    // 隔离状态
    struct {
        bool initialized;
        bool is_isolated;
        airy_pid_t pid;  // 使用平台抽象层，禁止直接使用 pid_t（违反 CROSS-01）
    } isolation;
} airy_workspace_t;

/**
 * @brief 创建虚拟工位
 */
int workspace_create(
    const WorkspaceConfig* manager,
    airy_workspace_t** out_workspace
);

/**
 * @brief 销毁虚拟工位
 */
int workspace_destroy(airy_workspace_t* workspace);

/**
 * @brief 在工位中执行任务
 */
int workspace_execute(
    airy_workspace_t* workspace,
    const TaskConfig* task_config,
    TaskResult* out_result
);
```

#### 2.3 D2: 权限裁决

```yaml
# agentrt/cupolas/permission_rules.yaml
permission_rules:
  # 默认策略：拒绝
  default_policy: "deny"
  
  # 角色定义
  roles:
    admin:
      permissions:
        - "task:*"
        - "memory:*"
        - "manager:*"
        
    operator:
      permissions:
        - "task:submit"
        - "task:query"
        - "memory:read"
        
    developer:
      permissions:
        - "task:submit"
        - "task:query"
        
    viewer:
      permissions:
        - "task:query"
        
  # 用户绑定
  bindings:
    - user: "admin@agentrt.internal"
      roles: ["admin"]
    - user: "operator01@agentrt.internal"
      roles: ["operator"]
```

```c
// agentrt/atoms/agentrt/cupolas/permission.h
/**
 * @brief 权限裁决结果
 */
typedef enum {
    PERMISSION_DENIED = 0,
    PERMISSION_GRANTED = 1,
} PermissionResult;

/**
 * @brief 检查权限
 *
 * @param user_id 用户标识
 * @param permission 权限字符串，格式：resource:action
 * @param context 权限上下文（资源所有者等）
 * @return 权限裁决结果
 *
 * @example
 * PermissionResult result = check_permission(
 *     "user123",
 *     "task:submit",
 *     &(PermissionContext){ .owner = "org456" }
 * );
 */
PermissionResult check_permission(
    const char* user_id,
    const char* permission,
    const PermissionContext* context
);

/**
 * @brief 权限裁决日志
 */
void log_permission_check(
    const char* user_id,
    const char* permission,
    PermissionResult result,
    const char* reason
);
```

#### 2.4 D3: 输入净化

```yaml
# agentrt/cupolas/input_sanitization_rules.yaml
sanitization_rules:
  # SQL 注入防护
  sql_injection:
    enabled: true
    # ⚠️ 重要：关键字黑名单 NOT sufficient，仅作为辅助防御层。
    # SQL 注入防护必须使用参数化查询（parameterized queries / prepared statements），
    # 黑名单可被编码绕过（如 URL 编码、Unicode、注释注入等）。
    # 参见第八章漏洞防护中的 SQL 注入条目。
    patterns:
      - "'"
      - "\""
      - ";"
      - "--"
      - "/*"
      - "*/"
      - "UNION"
      - "SELECT"
      - "DROP"
      
  # 命令注入防护
  command_injection:
    enabled: true
    patterns:
      - ";"
      - "|"
      - "&"
      - "$"
      - "`"
      - "\n"
      - "\r"
      
  # 路径遍历防护
  path_traversal:
    enabled: true
    blocked_patterns:
      - "../"
      - "..\\"
      - "%2e%2e"
      
  # XSS 防护
  xss:
    enabled: true
    patterns:
      - "<script"
      - "javascript:"
      - "onerror="
```

```c
// agentrt/atoms/agentrt/cupolas/sanitizer.h
/**
 * @brief 输入净化上下文
 */
typedef struct sanitizer_context {
    const char* input;
    size_t input_length;
    SanitizationRuleType rule_type;
    bool is_sanitized;
} sanitizer_context_t;

/**
 * @brief 净化输入
 *
 * @param context 净化上下文
 * @param output 输出缓冲区
 * @param output_size 输出缓冲区大小
 * @return 净化后的字符串长度，负值表示错误
 */
int sanitize_input(
    const sanitizer_context_t* context,
    char* output,
    size_t output_size
);

/**
 * @brief 预定义的净化规则
 */
typedef enum {
    SANITIZE_SQL_INJECTION = 0x01,
    SANITIZE_COMMAND_INJECTION = 0x02,
    SANITIZE_PATH_TRAVERSAL = 0x04,
    SANITIZE_XSS = 0x08,
    SANITIZE_ALL = 0xFF
} SanitizationRuleType;
```

#### 2.5 D4: 审计日志

```yaml
# agentrt/cupolas/audit_config.yaml
audit:
  # 审计日志配置
  log:
    path: "/var/agentrt/logs/audit"
    rotation:
      max_size_mb: 100
      max_files: 30
      compress: true
      
  # 异步写入
  async:
    enabled: true
    queue_size: 10000
    flush_interval_ms: 1000
    
  # 审计事件类型
  event_types:
    authentication:
      - AUTH_SUCCESS
      - AUTH_FAILURE
      - AUTH_EXPIRED
      
    authorization:
      - PERMISSION_GRANTED
      - PERMISSION_DENIED
      
    data_access:
      - DATA_READ
      - DATA_WRITE
      - DATA_DELETE
      
    config_change:
      - CONFIG_MODIFIED
      - RULE_UPDATED
```

```c
// agentrt/atoms/agentrt/cupolas/audit.h
/**
 * @brief 审计事件类型
 */
typedef enum {
    // 认证事件 (1xxx)
    AUDIT_AUTH_SUCCESS = 1001,
    AUDIT_AUTH_FAILURE = 1002,
    AUDIT_AUTH_EXPIRED = 1003,
    
    // 授权事件 (2xxx)
    AUDIT_PERMISSION_GRANTED = 2001,
    AUDIT_PERMISSION_DENIED = 2002,
    
    // 数据访问事件 (3xxx)
    AUDIT_DATA_READ = 3001,
    AUDIT_DATA_WRITE = 3002,
    AUDIT_DATA_DELETE = 3003,
    
    // 配置变更事件 (4xxx)
    AUDIT_CONFIG_MODIFIED = 4001,
    AUDIT_RULE_UPDATED = 4002
} AuditEventType;

/**
 * @brief 审计记录
 */
typedef struct audit_record {
    uint64_t record_id;
    AuditEventType event_type;
    uint64_t timestamp_us;
    const char* user_id;
    const char* resource;
    const char* result;
    const char* ip_address;
    const char* session_id;
    KeyValuePair* metadata;
} audit_record_t;

/**
 * @brief 记录审计事件
 */
int audit_log(const audit_record_t* record);

/**
 * @brief 查询审计日志
 */
int audit_query(
    const AuditQuery* query,
    AuditRecordList* out_records
);
```

---

### 三、加密设计

#### 3.1 密钥管理架构

```
┌─────────────────────────────────────────────────────────────┐
│                    密钥管理架构                              │
│  ┌─────────────────┐    ┌─────────────────┐                │
│  │   KEK (密钥加密密钥)  │ ←→  │   密钥存储服务   │                │
│  └────────┬──────────┘    └─────────────────┘                │
│           │                                                  │
│           ▼                                                  │
│  ┌─────────────────┐    ┌─────────────────┐                │
│  │   DEK (数据加密密钥)  │ ←→  │   envelope encryption │                │
│  └────────┬──────────┘    └─────────────────┘                │
│           │                                                  │
│           ▼                                                  │
│  ┌─────────────────┐                                        │
│  │      加密数据     │                                        │
│  └─────────────────┘                                        │
└─────────────────────────────────────────────────────────────┘
```

#### 3.2 加密算法标准

| 用途 | 算法 | 密钥长度 | 模式 |
|------|------|----------|------|
| 对称加密 | AES | 256 位 | GCM |
| 非对称加密 | RSA | 3072 位 | OAEP |
| 哈希 | SHA-256/SHA-384 | - | - |
| HMAC | SHA-256 | - | - |
| 密钥交换 | ECDH | 256 位 | - |
| 签名 | ECDSA | 256 位 | - |

#### 3.3 加密接口错误返回值语义

所有加密相关函数遵循统一的错误返回值语义：

| 返回值 | 含义 | 示例错误码 |
|--------|------|-----------|
| `0` | 成功 | - |
| 负值 | 错误码（定义于 `error.h`） | `AIRY_EINVAL`（参数无效）、`AIRY_ENOMEM`（内存不足）、`AIRY_EDECRYPT`（解密失败）、`AIRY_EKEYREVOKED`（密钥已吊销） |

**规则**：
- 调用方必须检查返回值，不得忽略错误
- 负值错误码与 `error.h` 中定义的 `AIRY_E*` 常量一致
- 禁止使用正数表示部分成功（违反一致性原则）

#### 3.4 密钥轮换策略

| 策略项 | 要求 |
|--------|------|
| **常规轮换周期** | DEK 每 90 天轮换一次；KEK 每 365 天轮换一次 |
| **轮换触发条件** | 密钥使用次数达到阈值、密钥疑似泄露、算法安全等级下降 |
| **轮换过程** | 1. 生成新密钥 → 2. 使用新密钥加密新数据 → 3. 旧数据按需重新加密 → 4. 旧密钥标记为"过渡期"（保留 30 天用于解密） → 5. 过渡期结束后安全擦除旧密钥 |
| **紧急轮换** | 密钥泄露确认后 4 小时内完成轮换；紧急轮换跳过过渡期，立即吊销旧密钥并强制重新加密所有受影响数据 |
| **密钥吊销** | 吊销的密钥立即加入 CRL（证书吊销列表），所有使用该密钥的会话和令牌强制失效 |
| **密钥归档** | 已轮换密钥的安全擦除必须使用 `AIRY_SECURE_ZERO`，禁止依赖 `memset`（编译器可能优化掉） |

```c
// agentrt/atoms/security/crypto.h
/**
 * @brief 加密配置
 */
typedef struct crypto_config {
    /** 对称加密算法 */
    SymmetricAlgorithm symmetric_algo;
    
    /** 非对称加密算法 */
    AsymmetricAlgorithm asymmetric_algo;
    
    /** 哈希算法 */
    HashAlgorithm hash_algo;
    
    /** 密钥长度（位） */
    uint32_t key_length;
    
    /** 是否启用硬件加速 */
    bool hw_acceleration;
} crypto_config_t;

/**
 * @brief 对称加密
 *
 * @return 0 表示成功；负值表示错误码（定义于 error.h，如 AIRY_EINVAL、AIRY_ENOMEM）
 */
int crypto_encrypt_symmetric(
    const uint8_t* plaintext,
    size_t plaintext_len,
    const uint8_t* key,
    size_t key_len,
    uint8_t** out_ciphertext,
    size_t* out_ciphertext_len
);

/**
 * @brief 对称解密
 *
 * @return 0 表示成功；负值表示错误码（定义于 error.h，如 AIRY_EINVAL、AIRY_EDECRYPT）
 */
int crypto_decrypt_symmetric(
    const uint8_t* ciphertext,
    size_t ciphertext_len,
    const uint8_t* key,
    size_t key_len,
    uint8_t** out_plaintext,
    size_t* out_plaintext_len
);
```

---

### 四、安全通信

#### 4.1 传输层安全

```yaml
# security/tls_config.yaml
tls:
  # 最低 TLS 版本
  min_version: "1.3"
  
  # 密码套件
  cipher_suites:
    - "TLS_AES_256_GCM_SHA384"
    - "TLS_CHACHA20_POLY1305_SHA256"
    - "TLS_AES_128_GCM_SHA256"
    
  # 证书配置
  certificate:
    path: "/etc/agentrt/tls/server.crt"
    private_key_path: "/etc/agentrt/tls/server.key"
    ca_path: "/etc/agentrt/tls/ca.crt"
    
  # 客户端认证
  client_auth:
    enabled: true
    cert_path: "/etc/agentrt/tls/client.crt"
```

#### 4.2 mTLS 配置

```c
// agentrt/atoms/security/tls.h
/**
 * @brief mTLS 连接配置
 */
typedef struct mtls_config {
    /** 服务器证书 */
    const char* server_cert_path;
    
    /** 服务器私钥 */
    const char* server_key_path;
    
    /** CA 证书（用于验证客户端） */
    const char* ca_cert_path;
    
    /** 是否验证客户端证书 */
    bool verify_client;
    
    /** CRL 列表路径 */
    const char* crl_paths;
} mtls_config_t;

/**
 * @brief 创建 mTLS 连接
 */
TLSSession* mtls_connect(const mtls_config_t* manager);
```

---

### 五、身份认证

#### 5.1 认证架构

```yaml
# security/auth_config.yaml
authentication:
  # 默认认证方式
  default_method: "jwt"
  
  # JWT 配置
  jwt:
    issuer: "agentrt.auth"
    audience: "agentrt.api"
    access_token_ttl_seconds: 3600
    refresh_token_ttl_days: 30
    algorithm: "ES256"
    
  # 外部认证集成
  external_providers:
    - name: "oauth2"
      enabled: true
      providers:
        - "google"
        - "github"
        
    - name: "saml"
      enabled: false
```

#### 5.2 会话管理

```c
// agentrt/atoms/security/session.h
/**
 * @brief 会话状态
 */
typedef enum {
    SESSION_ACTIVE = 0,
    SESSION_EXPIRED = 1,
    SESSION_REVOKED = 2,
    SESSION_INVALID = 3
} SessionState;

/**
 * @brief 会话信息
 */
typedef struct session {
    uint64_t session_id;
    const char* user_id;
    SessionState state;
    uint64_t created_at_us;
    uint64_t expires_at_us;
    const char* ip_address;
    const char* user_agent;
} session_t;

/**
 * @brief 创建会话
 */
int session_create(
    const char* user_id,
    const SessionConfig* manager,
    session_t** out_session
);

/**
 * @brief 验证会话
 */
int session_validate(
    const char* session_token,
    session_t** out_session
);

/**
 * @brief 撤销会话
 */
int session_revoke(uint64_t session_id);
```

---

### 六、隐私保护

#### 6.1 数据脱敏

```yaml
# security/data_masking.yaml
data_masking:
  # 敏感字段配置
  sensitive_fields:
    - name: "password"
      mask_type: "always"  # always | partial | none
      mask_char: "*"
      visible_chars: 0
      
    - name: "email"
      mask_type: "partial"
      mask_char: "*"
      visible_start: 2
      visible_end: 2
      
    - name: "phone"
      mask_type: "partial"
      mask_char: "*"
      visible_start: 3
      visible_end: 4
      
    - name: "credit_card"
      mask_type: "partial"
      mask_char: "*"
      visible_start: 0
      visible_end: 4
```

#### 6.2 数据最小化

```c
// agentrt/atoms/security/privacy.h
/**
 * @brief 数据最小化配置
 */
typedef struct privacy_config {
    /** 是否记录个人数据 */
    bool log_personal_data;
    
    /** 是否记录查询参数 */
    bool log_query_params;
    
    /** 敏感字段列表 */
    const char** sensitive_fields;
    size_t num_sensitive_fields;
    
    /** 数据保留期限（天） */
    uint32_t retention_days;
} privacy_config_t;

/**
 * @brief 脱敏数据
 */
int privacy_mask(
    const char* field_name,
    const uint8_t* data,
    size_t data_len,
    uint8_t* out_masked,
    size_t* out_masked_len
);

/**
 * @brief 检查是否为敏感数据
 */
bool privacy_is_sensitive(const char* field_name);
```

---

### 七、安全监控

#### 7.1 健康检查

```yaml
# security/health_check.yaml
health_check:
  # 安全组件健康检查
  security:
    check_interval_seconds: 60
    
    # 检查项
    checks:
      - name: "encryption_module"
        enabled: true
        critical: true
        
      - name: "permission_cache"
        enabled: true
        critical: false
        
      - name: "audit_queue"
        enabled: true
        critical: true
        
      - name: "tls_cert_expiry"
        enabled: true
        critical: true
        warning_days: 30
```

```c
// agentrt/atoms/security/health.h
/**
 * @brief 健康检查结果
 */
typedef struct health_check_result {
    const char* check_name;
    bool healthy;
    bool critical;
    const char* message;
    uint64_t last_check_us;
} health_check_result_t;

/**
 * @brief 执行安全健康检查
 */
int security_health_check(
    health_check_result_t** out_results,
    size_t* out_num_results
);

/**
 * @brief 获取安全指标
 */
int security_get_metrics(SecurityMetrics* out_metrics);
```

#### 7.2 威胁检测

```yaml
# security/threat_detection.yaml
threat_detection:
  # 启用威胁检测
  enabled: true
  
  # 速率限制
  rate_limiting:
    - name: "auth_attempts"
      window_seconds: 300
      max_attempts: 5
      action: "block"
      
    - name: "api_requests"
      window_seconds: 60
      max_attempts: 1000
      action: "throttle"
      
  # 异常检测
  anomaly_detection:
    # 失败登录检测
    failed_login:
      threshold: 5
      window_minutes: 15
      action: "alert"
      
    # 异常访问模式
    access_pattern:
      enabled: true
      baseline_hours: 72
      deviation_threshold: 2.5
```

---

### 八、漏洞防护

#### 8.1 常见漏洞防护

| 漏洞类型 | 防护措施 | 实现层 |
|----------|----------|--------|
| SQL 注入 | 参数化查询（必须）；关键字黑名单仅辅助 | D3 输入净化 |
| XSS | 输出编码 | D3 输入净化 |
| CSRF | Token 验证 | D2 权限裁决 |
| 命令注入 | 白名单验证 | D3 输入净化 |
| 路径遍历 | 路径规范化 | D3 输入净化 |
| 敏感信息泄露 | 加密存储 | 加密模块 |

#### 8.2 内存安全

```c
// agentrt/atoms/security/memory.h
/**
 * @brief 安全内存操作
 */

/**
 * @brief 安全内存复制
 */
void* secure_memcpy(void* dest, const void* src, size_t n);

/**
 * @brief 安全内存设置（防编译器优化）
 */
void secure_memset(void* ptr, uint8_t value, size_t n);

/**
 * @brief 安全内存释放
 */
void secure_free(void* ptr, size_t size);

/**
 * @brief 分配安全内存
 */
void* secure_malloc(size_t size);

/**
 * @brief 分配并初始化安全内存
 */
void* secure_calloc(size_t nmemb, size_t size);
```

---

### 九、安全威胁建模（STRIDE）

#### 9.1 建模方法论

所有 Airymax 组件在安全设计阶段**必须**进行 STRIDE 威胁建模，识别并记录潜在威胁后再制定防护措施。

| 威胁类型 | 英文 | 安全属性 | Airymax 典型场景 |
|----------|------|----------|-----------------|
| **欺骗** | Spoofing | 身份认证 | 伪造 JWT 令牌、冒充合法进程 |
| **篡改** | Tampering | 完整性 | 修改审计日志、注入恶意配置 |
| **否认** | Repudiation | 不可抵赖 | 否认执行过危险操作、删除操作记录 |
| **信息泄露** | Information Disclosure | 机密性 | 密钥明文写入日志、内存未清零导致泄露 |
| **拒绝服务** | Denial of Service | 可用性 | 资源耗尽攻击、死锁导致服务不可用 |
| **权限提升** | Elevation of Privilege | 授权 | 容器逃逸、利用漏洞获取 root 权限 |

#### 9.2 建模流程（强制）

1. **绘制数据流图（DFD）**：标识信任边界、数据流、数据存储和外部实体
2. **逐元素应用 STRIDE**：对 DFD 中每个元素逐一分析六类威胁
3. **威胁评级**：使用风险矩阵（影响 × 可能性）对威胁排序
4. **制定防护措施**：高/中风险威胁必须有对应防护措施并映射到安全穹顶（D1-D4）层
5. **文档化**：威胁建模结果记录在 `docs/security/threat_model/` 目录，随架构变更更新

---

### 十、安全测试要求

#### 10.1 静态应用安全测试（SAST）

| 要求 | 说明 |
|------|------|
| **工具** | CodeQL（必选）、Coverity（推荐）、cppcheck（C/C++ 基础检查） |
| **触发时机** | 每次 PR/MR 提交自动运行；主分支每日全量扫描 |
| **阻断规则** | 高严重度（Critical/High）发现必须阻断合并；中严重度需在 7 天内修复 |
| **自定义规则** | 针对 Airymax BAN-01~13 禁止模式编写 CodeQL 自定义查询 |
| **误报处理** | 标记为误报的发现需经安全团队审核确认，不得由开发者单方面关闭 |

#### 10.2 动态应用安全测试（DAST）

| 要求 | 说明 |
|------|------|
| **工具** | OWASP ZAP（API 扫描）、Burp Suite（深度渗透测试） |
| **触发时机** | 每次发布前必须完成 DAST 扫描；staging 环境每周自动扫描 |
| **覆盖范围** | 所有对外暴露的 HTTP/gRPC 接口 |

#### 10.3 渗透测试

| 要求 | 说明 |
|------|------|
| **频率** | 每季度至少一次外部渗透测试；重大版本发布前必须进行 |
| **范围** | 覆盖安全穹顶四层（D1-D4）、加密模块、认证模块 |
| **报告** | 渗透测试报告归档于 `docs/security/pentest/`，发现的问题录入 Issue 跟踪 |
| **修复时限** | Critical 级别 24 小时内修复；High 级别 7 天内修复 |

---

### 十一、合规性

#### 11.1 数据保护合规

| 法规 | 要求 | Airymax 实现 |
|------|------|--------------|
| GDPR | 数据主体权利 | D4 审计、D3 输入控制 |
| 最小化收集 | 只收集必要数据 | 数据最小化配置 |
| 数据保留 | 定期删除过期数据 | 保留策略配置 |
| 数据可携 | 支持数据导出 | 数据导出 API |

#### 11.2 审计合规

```yaml
# compliance/audit_requirements.yaml
compliance:
  # 审计日志保留期
  retention:
    standard: 365  # 天
    financial: 2555  # 天 (7年)
    healthcare: 1825  # 天 (5年)
    
  # 审计日志格式
  format:
    type: "json"
    encoding: "utf-8"
    include_fields:
      - timestamp
      - user_id
      - action
      - resource
      - result
      - ip_address
      
  # 审计日志签名
  integrity:
    enabled: true
    algorithm: "HMAC-SHA256"
    key_rotation_days: 90
```

---

### 十二、Airymax 模块安全设计示例

#### 12.1 安全穹顶 (cupolas) 安全设计
安全穹顶 (cupolas) 模块实现Airymax的安全穹顶，是安全设计的核心：

##### 12.1.1 虚拟工位（Virtual Workbench）安全设计（映射原则：D-2 安全隔离）
```python
"""
虚拟工位安全设计 - 体现系统观（S-1）和工程观（E-2）原则

基于容器命名空间和WASM沙箱实现进程级隔离。
集成资源限额和权限控制，防止任务间相互影响。
"""
from typing import Dict, Any
from dataclasses import dataclass
from contextlib import contextmanager

@dataclass
class SecurityPolicy:
    """安全策略定义 - 体现防御深度（D-3）原则"""
    isolation_level: str  # container, wasm, process
    resource_limits: Dict[str, Any]  # CPU, memory, disk quotas
    network_policy: Dict[str, Any]  # 网络访问控制
    capabilities: List[str]  # Linux capabilities
    audit_config: Dict[str, Any]  # 审计配置
    
    def validate(self) -> bool:
        """策略验证 - 多层安全检查"""
        # 层次1：资源限制验证
        if not self.validate_resource_limits():
            return False
        
        # 层次2：能力最小化验证
        if not self.validate_capabilities():
            return False
        
        # 层次3：网络策略验证
        if not self.validate_network_policy():
            return False
        
        return True

class VirtualWorkbench:
    """虚拟工位实现 - 体现安全工程（D-1至D-4）原则"""
    
    def __init__(self, policy: SecurityPolicy):
        self.policy = policy
        self.audit_logger = AuditLogger()
        self.resource_monitor = ResourceMonitor()
        
    @contextmanager
    def create_sandbox(self, task_config: Dict[str, Any]):
        """创建安全沙箱 - 体现资源确定性原则"""
        sandbox_id = self.generate_sandbox_id()
        
        # 记录审计日志
        self.audit_logger.log_sandbox_creation(sandbox_id, task_config)
        
        try:
            # 应用资源限制
            self.resource_monitor.apply_limits(self.policy.resource_limits)
            
            # 创建隔离环境
            sandbox = self.create_isolation_environment(
                sandbox_id, 
                self.policy.isolation_level
            )
            
            yield sandbox
            
        except SecurityViolation as e:
            # 安全违规处理
            self.audit_logger.log_security_violation(sandbox_id, e)
            raise
            
        finally:
            # 清理资源
            self.cleanup_sandbox(sandbox_id)
            self.audit_logger.log_sandbox_cleanup(sandbox_id)
```

##### 12.1.2 权限裁决引擎安全设计（映射原则：D-1 最小权限）
```typescript
/**
 * 权限裁决引擎安全设计 - 体现安全工程（D-1, D-4）原则
 * 
 * 基于YAML规则引擎实现细粒度访问控制。
 * 支持动态策略更新和形式化验证。
 */
interface AccessPolicy {
  subject: string;
  resource: string;
  action: string;
  conditions: Condition[];
  effect: 'allow' | 'deny';
}

interface SecurityContext {
  user: User;
  environment: Environment;
  riskAssessment: RiskAssessment;
}

class PermissionArbiter {
  private readonly policyStore: PolicyStore;
  private readonly auditLogger: AuditLogger;
  private readonly riskEngine: RiskEngine;
  
  /**
   * 安全访问决策 - 体现防御深度（D-3）原则
   */
  async decideAccess(request: AccessRequest): Promise<AccessDecision> {
    const startTime = Date.now();
    
    try {
      // 层次1：身份验证
      const subject = await this.authenticate(request.subject);
      if (!subject) {
        await this.auditLogger.log('authentication_failed', request);
        return AccessDecision.deny('Authentication failed');
      }
      
      // 层次2：策略评估
      const policies = await this.policyStore.findApplicablePolicies(
        subject, request.resource, request.action
      );
      
      // 层次3：风险评估
      const context: SecurityContext = {
        user: subject,
        environment: await this.getEnvironment(),
        riskAssessment: await this.assessRisk(request)
      };
      
      // 层次4：策略决策
      const decision = await this.evaluatePolicies(policies, context);
      
      // 审计日志
      await this.auditLogger.log('access_decision', {
        request,
        decision,
        processingTime: Date.now() - startTime,
        riskScore: context.riskAssessment.score
      });
      
      return decision;
      
    } catch (error) {
      await this.auditLogger.log('decision_error', {
        request,
        error: error.message,
        stack: error.stack
      });
      
      // 失效安全：默认拒绝
      return AccessDecision.deny('Security decision error');
    }
  }
  
  /**
   * 策略形式化验证 - 体现工程观（E-1）原则
   */
  async verifyPolicy(policy: AccessPolicy): Promise<VerificationResult> {
    // 使用形式化方法验证策略正确性
    const verifier = new PolicyVerifier();
    
    // 检查策略冲突
    const conflicts = await verifier.checkConflicts(policy);
    if (conflicts.length > 0) {
      return {
        valid: false,
        errors: conflicts.map(c => `Policy conflict: ${c.description}`)
      };
    }
    
    // 检查安全属性
    const safetyProperties = await verifier.verifySafetyProperties(policy);
    if (!safetyProperties.allSatisfied) {
      return {
        valid: false,
        errors: safetyProperties.violations
      };
    }
    
    return { valid: true, errors: [] };
  }
}
```

#### 12.2 Atoms（原子层）安全设计
Atoms模块作为微核心核心，需要实现基础安全原语：

##### 12.2.1 安全内存管理（映射原则：M-3 拓扑优化）

> **跨平台合规说明**：本节代码中 `secure_memory_pool_t` 使用 `airy_mtx_t` 而非 `pthread_mutex_t`，遵循 [C编码规范 CROSS-01](./C_Cpp_coding_style.md) 的跨平台规则。直接使用 POSIX 线程 API（如 `pthread_mutex_t`）违反 CROSS-01，必须使用 `platform.h` 提供的 `airy_mtx_lock()`/`airy_mtx_unlock()` 抽象。

```c
/**
 * @brief 安全内存管理器 - 体现内核观（K-1）和工程观（E-3）原则
 * 
 * 实现NUMA感知的安全内存分配，集成内存保护和安全擦除。
 * 防御use-after-free和缓冲区溢出攻击。
 * 
 * @see memoryrovol.md 中的记忆进化算法
 */
typedef struct secure_memory_pool {
    size_t total_size;
    size_t allocated;
    size_t watermark;
    uint8_t canary[SECURITY_CANARY_SIZE];
    airy_mtx_t lock;  // 使用平台抽象层，禁止直接使用 pthread_mutex_t（违反 CROSS-01）
    numa_node_t numa_node;
} secure_memory_pool_t;

/**
 * @brief 安全内存分配 - 体现防御深度（D-3）原则
 * 
 * 多层安全保护：
 * 1. 边界检查
 * 2. 内存初始化
 * 3. 使用前验证
 * 4. 释放后擦除
 */
void* atoms_secure_alloc(size_t size, numa_node_t node, uint32_t flags) {
    // 安全检查1：大小验证
    if (size == 0 || size > MAX_SECURE_ALLOC_SIZE) {
        log_security("Invalid secure allocation size: %zu", size);
        return NULL;
    }
    
    // 安全检查2：NUMA节点验证
    if (!validate_numa_node(node)) {
        log_security("Invalid NUMA node: %d", node);
        return NULL;
    }
    
    // 分配内存（包含保护区域）
    size_t total_size = size + SECURITY_PADDING * 2;
    void* ptr = numa_alloc_onnode(total_size, node);
    if (!ptr) {
        log_security("NUMA allocation failed: size=%zu, node=%d", total_size, node);
        return NULL;
    }
    
    // 内存初始化（防止信息泄漏）
    secure_memset(ptr, SECURITY_PATTERN, total_size);
    
    // 设置边界保护
    setup_boundary_guards(ptr, total_size);
    
    // 返回可用内存区域（跳过保护区域）
    void* user_ptr = (uint8_t*)ptr + SECURITY_PADDING;
    
    // 记录分配（审计跟踪）
    log_audit("Secure memory allocated: ptr=%p, size=%zu, node=%d, caller=%s",
              user_ptr, size, node, get_caller_info());
    
    return user_ptr;
}

/**
 * @brief 安全内存释放 - 体现失效安全原则
 */
void atoms_secure_free(void* ptr, size_t size) {
    if (!ptr) return;
    
    // 验证指针有效性
    if (!validate_secure_pointer(ptr, size)) {
        log_security("Invalid secure free attempt: ptr=%p, size=%zu", ptr, size);
        return;
    }
    
    // 获取完整分配区域
    void* base_ptr = (uint8_t*)ptr - SECURITY_PADDING;
    size_t total_size = size + SECURITY_PADDING * 2;
    
    // 检查边界保护（检测缓冲区溢出）
    if (!check_boundary_guards(base_ptr, total_size)) {
        log_security("Boundary guard violation detected: ptr=%p", ptr);
        // 记录安全事件但不崩溃（fail-secure）
    }
    
    // 安全擦除（防止信息泄漏）
    secure_memset(base_ptr, SECURITY_ERASE_PATTERN, total_size);
    
    // 实际释放
    numa_free(base_ptr, total_size);
    
    // 审计日志
    log_audit("Secure memory freed: ptr=%p, size=%zu", ptr, size);
}
```

#### 12.3 跨模块安全集成
重要跨模块接口必须实现额外的安全防护：

##### 12.3.1 核心三循环安全集成（映射原则：S-1 垂直分层）
```go
// 核心三循环安全集成 - 体现系统观（S-1）和工程观（E-2）原则
//
// 连接coreloopthree调度器与MicroCoreRT任务管理。
// 实现双向身份验证和完整性保护。
type SecureSchedulerBridge struct {
	crypto      CryptoService
	auth        AuthService
	auditLogger AuditLogger
	rateLimiter RateLimiter
}

// ScheduleSecureTask 安全任务调度 - 体现防御深度原则
func (b *SecureSchedulerBridge) ScheduleSecureTask(task SecureTask) error {
	// 层次1：请求签名验证
	if !b.verifyTaskSignature(task) {
		b.auditLogger.LogSecurityEvent("invalid_task_signature", task.ID)
		return errors.New("invalid task signature")
	}
	
	// 层次2：抗重放检查
	if b.isReplayAttack(task.Nonce) {
		b.auditLogger.LogSecurityEvent("replay_attack_detected", task.ID)
		return errors.New("replay attack detected")
	}
	
	// 层次3：权限检查
	if !b.checkSchedulePermission(task) {
		b.auditLogger.LogAuditEvent("unauthorized_schedule_attempt", task)
		return errors.New("insufficient permission")
	}
	
	// 层次4：速率限制
	if !b.rateLimiter.Allow(task.Subject) {
		return errors.New("rate limit exceeded")
	}
	
	// 执行安全调度
	result, err := b.performSecureSchedule(task)
	if err != nil {
		b.auditLogger.LogError("schedule_failed", task.ID, err)
		return err
	}
	
	// 审计日志
	b.auditLogger.LogAuditEvent("task_scheduled", map[string]interface{}{
		"task_id":    task.ID,
		"scheduler":  task.Scheduler,
		"priority":   task.Priority,
		"timestamp":  time.Now(),
		"result":     result,
	})
	
	return nil
}
```

---

### 十三、参考文献

1. **Airymax 架构设计原则**: [00-architectural-principles.md](../../00-architectural-principles.md)
2. **Airymax 微核心设计**: [microcorert.md](../../10-architecture/04-microcorert.md)
3. **Airymax 统一术语表**: [90-terminology.md](../90-terminology.md)
4. **OWASP Top 10**: https://owasp.org/www-project-top-ten/
5. **NIST Cybersecurity Framework**: https://www.nist.gov/cyberframework
6. **ISO 27001**: https://www.iso.org/isoiec-27001-information-security.html
7. **Airymax 核心架构文档**:
   - [coreloopthree.md](../../10-architecture/02-coreloopthree.md)
   - [memoryrovol.md](../../10-architecture/03-memoryrovol.md)
   - [ipc.md](../../10-architecture/30-kernel/01-ipc.md)
   - [syscall.md](../../10-architecture/05-syscall.md)
   - [logging_system.md](../../10-architecture/50-services/02-logging.md)

---

### 附录：跨文档规范引用

本规范与以下 Airymax 工程规范一致，所有安全设计须同时遵循：

| 规范集 | 说明 | 来源文档 |
|--------|------|---------|
| **BAN-01~13** | 13 项禁止模式（桩函数/假数据/空返回等） | [C编码规范 §18](./C_Cpp_coding_style.md) |
| **CROSS-01~06** | 6 项跨平台编译规则 | [C编码规范 §17](./C_Cpp_coding_style.md) |
| **REQ-01~08** | 8 项强制规范 | [C编码规范 §1.2](./C_Cpp_coding_style.md) |
| **标准术语** | 8 个架构组件标准名称 | [90-terminology.md](../90-terminology.md) |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
