Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# AirymaxOS 驱动模型设计

> **文档定位**: AirymaxOS（agentrt-linux）驱动子系统工程设计主索引
> **版本**: 0.1.1（占位）/ 1.0.1（开发）
> **最后更新**: 2026-07-06
> **同源映射**: agentrt `daemons`（用户态服务）+ Linux 6.6 `drivers/base/`
> **理论根基**: Linux device/driver/bus 三元组解耦 + Airymax K-3 服务隔离

---

## 1. 模块定位

AirymaxOS 驱动模型是连接内核能力与硬件/虚拟设备的核心抽象层。它继承 Linux 内核 30+ 年沉淀的 device/driver/bus 三元组解耦哲学，并在其上扩展智能体工作负载所需的"软驱动"概念——Agent 行为契约作为一类虚拟设备参与驱动模型。

### 1.1 核心抽象

| 抽象 | 职责 | Linux 来源 |
|------|------|------------|
| **device** | 设备实例（硬件或虚拟） | `include/linux/device.h` |
| **driver** | 驱动逻辑（绑定到 device） | `include/linux/device/driver.h` |
| **bus** | 总线类型（匹配 device 与 driver） | `include/linux/device/bus.h` |
| **class** | 设备功能分类（电源/输入/网络等） | `include/linux/class.h` |

### 1.2 AirymaxOS 扩展

- **Agent 驱动**：将 Agent 行为契约作为虚拟设备，通过 `agent_driver_register()` 注册
- **智能体总线**：新增 `agent_bus_type`，匹配 Agent SDK 接口与运行时实现
- **资源托管扩展**：`devm_` 资源管理扩展至 Token 预算、记忆卷载等智能体资源

---

## 2. 核心设计原则

### 2.1 device/driver/bus 三元组解耦

Linux 设备模型的核心是 device、driver、bus 三者解耦：
- **device** 仅描述"有什么设备"（资源、属性、状态）
- **driver** 仅描述"如何操作设备"（probe/remove/callback）
- **bus** 负责"匹配 device 与 driver"（基于 name/OF/ACPI）

这种解耦使得同一 driver 可服务多个 device，同一 device 可被不同 driver 接管。

### 2.2 module_*_driver 宏消除样板

```c
/* 简洁形式 — 消除 module_init/module_exit 样板 */
module_platform_driver(my_driver);

/* 等价于 */
static int __init my_driver_init(void) { return platform_driver_register(&my_driver); }
module_init(my_driver_init);
static void __exit my_driver_exit(void) { platform_driver_unregister(&my_driver); }
module_exit(my_driver_exit);
```

### 2.3 devm_ 资源管理（E-3 资源确定性）

`devm_` 系列函数（`devm_kzalloc`、`devm_ioremap`、`devm_request_irq` 等）自动在 device detach 时释放资源，消除手动释放遗漏导致的内存泄漏。

### 2.4 platform 总线（SoC 设备主力）

platform 总线是 SoC 设备的主要接入点，支持 DT（Device Tree）和 ACPI 双源回退：
```c
static const struct of_device_id my_match[] = {
    { .compatible = "vendor,my-device" },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, my_match);
```

### 2.5 misc 框架（轻量级字符设备）

misc 框架为简单字符设备提供快速接入路径，无需完整 cdev 样板。

---

## 3. 文档索引

```
60-driver-model/
├── README.md                       # 本文件
├── 01-device-model.md              # device/driver/bus 三元组详解
├── 02-platform-driver.md           # platform 总线与 SoC 设备
├── 03-devm-resource.md            # devm_ 资源管理与生命周期
├── 04-misc-framework.md           # misc 框架与轻量级字符设备
├── 05-agent-driver.md             # AirymaxOS 专属：Agent 虚拟设备驱动
├── 06-driver-samples.md           # 驱动示例集（含完整代码）
└── 07-driver-testing.md           # 驱动测试方法（KUnit + kselftest）
```

### 3.1 0.1.1 版本范围

仅完成 README + 01 + 02（3 文档）。其余 5 文档在 1.0.1 版本完成。

### 3.2 1.0.1 版本范围

完成全部 7 文档，并实施 driver model 工程标准。

---

## 4. AirymaxOS 专属扩展

### 4.1 Agent 虚拟设备驱动

AirymaxOS 将 Agent 行为契约建模为虚拟设备：
- **device**：Agent 实例（包含 SDK 句柄、运行时状态）
- **driver**：Agent 实现模块（包含 probe = 加载、remove = 卸载）
- **bus**：`agent_bus_type`（匹配 Agent SDK 接口与运行时实现）

### 4.2 资源托管扩展

`devm_` 扩展至智能体资源：
- `devm_agentrt_token_budget_alloc()`：Token 预算托管
- `devm_agentrt_memory_arena_create()`：记忆卷载托管
- `devm_agentrt_ipc_channel_open()`：AgentsIPC 通道托管

### 4.3 同源 agentrt 映射

| AirymaxOS | agentrt 同源 |
|-----------|--------------|
| `agent_bus_type` | daemons 适配器 |
| `agent_driver_register()` | daemon 注册机制 |
| `devm_agentrt_*` | 用户态资源管理 |

---

## 5. 五维原则映射

| 原则 | 在本模块的体现 |
|------|---------------|
| **K-3 服务隔离** | driver 与 device 解耦，driver 失败不影响其他 device |
| **E-3 资源确定性** | devm_ 强制资源生命周期与 device 绑定 |
| **E-8 可测试性** | driver 可独立测试（KUnit + kselftest） |
| **C-1 双系统协同** | Agent driver 与 native driver 共存于同一总线 |
| **A-1 极简主义** | module_*_driver 宏消除样板 |

---

## 6. 相关文档

- `50-engineering-standards/01-coding-standards.md`（驱动代码规范）
- `70-build-system/README.md`（驱动构建系统）
- `80-testing/README.md`（驱动测试方法）
- `20-modules/01-kernel.md`（kernel 子仓设计）
- `20-modules/02-services.md`（services 子仓设计）

---

## 7. 参考材料

- Linux 6.6 `drivers/base/platform.c`（platform 总线实现）
- Linux 6.6 `include/linux/platform_device.h`（`module_platform_driver` 宏）
- Linux 6.6 `Documentation/driver-api/`（驱动 API 文档）

---

> **文档结束** | 0.1.1 P0 优先完成 README + 01 + 02
