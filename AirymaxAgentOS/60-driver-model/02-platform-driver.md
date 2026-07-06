Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# AirymaxOS 驱动模型 — platform 总线与 SoC 设备

> **文档定位**: AirymaxOS（agentrt-linux）驱动子系统 60 模块第二篇——platform 总线实例与 SoC 设备资源管理
> **版本**: 0.1.1（占位）/ 1.0.1（开发）
> **最后更新**: 2026-07-06
> **同源映射**: agentrt `daemons`（用户态 SoC 适配器）+ Linux 6.6 `drivers/base/platform.c`（platform_bus_type 实现）
> **理论根基**: Linux 6.6 内核基线 + Airymax 五维正交 24 原则
> **核心约束**: IRON-9 同源但独立——platform 总线语义与上游一致，SoC 设备适配扩展在用户态

---

## 1. 概述

Linux 设备模型中 platform 总线是"伪总线"——它不是物理总线（如 PCI、USB），而是为 SoC 上无法被总线枚举的设备（内存映射寄存器、固定中断线）提供的"接入点"。AirymaxOS 选择 Linux 6.6 内核基线作为同源起点，platform 总线是 SoC 设备驱动的核心载体。

MicroCoreRT 不在内核态处理 SoC 设备语义——这是 K-1 内核极简原则的体现。SoC 设备的"业务语义"（如哪个寄存器对应哪个 Agent 的 Token 预算计数器）由用户态 agentrt daemon 解释；内核态只提供 platform_bus 的标准匹配与资源管理。AgentsIPC 在用户态承载进程间通信，daemon 通过 sysfs 与内核 platform driver 交互。

| 总线 | 设备发现方式 | 典型设备 | AirymaxOS 角色 |
|------|-------------|---------|----------------|
| **platform** | DT/ACPI 静态声明 | SoC 上的 UART、I2C、GPIO | AirymaxOS SoC 适配 driver 主要载体 |
| PCI/USB | 总线枚举 | 网卡、显卡、NVMe | 留给通用 Linux 生态 |
| I2C/SPI | platform driver 注册 adapter | I2C/SPI 子设备 | 传感器 driver 接入点 |
| agent_bus | daemon 注册 | Agent 虚拟设备 | AirymaxOS 自研（用户态） |

> **OS-DRV-020**: SoC 上 AirymaxOS 专属硬件必须通过 platform 总线接入内核，禁止新增独立总线类型。这是 K-1 内核极简原则的硬性约束——平台总线已足够承载。

> **OS-DRV-021**: PCI/USB 总线上发现的设备若需被 agentrt daemon 使用，必须通过 platform driver 提供"适配层"——daemon 通过 sysfs 访问，不直接调用 PCI/USB driver 内部 API。

---

## 2. platform_bus_type 详解

### 2.1 总线定义

Linux 6.6 `drivers/base/platform.c` 第 1454 行定义了 platform 总线实例：

```c
struct bus_type platform_bus_type = {
    .name           = "platform",
    .dev_groups     = platform_dev_groups,
    .driver_override = true,
    .match          = platform_match,
    .uevent         = platform_uevent,
    .probe          = platform_probe,
    .remove         = platform_remove,
    .shutdown       = platform_shutdown,
    .dma_configure  = platform_dma_configure,
    .dma_cleanup     = platform_dma_cleanup,
    .pm             = &platform_dev_pm_ops,
};
EXPORT_SYMBOL_GPL(platform_bus_type);
```

同时定义"总线父设备" `struct device platform_bus = { .init_name = "platform" };`，所有 platform_device 注册时默认 parent 为 `platform_bus`，构成 `/sys/devices/platform/` 拓扑根。

### 2.2 platform_match 五级回退

```mermaid
graph TD
    START["platform_match(dev, drv)"] --> L1{"driver_override<br/>设置?"}
    L1 -- "是" --> R1["返回 override 结果"]
    L1 -- "否" --> L2{"of_driver_match_device<br/>OF 匹配?"}
    L2 -- "是" --> R2["返回 1（匹配）"]
    L2 -- "否" --> L3{"acpi_driver_match_device<br/>ACPI 匹配?"}
    L3 -- "是" --> R3["返回 1（匹配）"]
    L3 -- "否" --> L4{"pdrv->id_table<br/>非空?"}
    L4 -- "是" --> R4["返回 platform_match_id 结果"]
    L4 -- "否" --> L5["strcmp(pdev->name, drv->name)"]
    L5 --> R5["返回比较结果"]
    style R2 fill:#e8f5e9
    style R3 fill:#e8f5e9
```

| 级别 | 匹配源 | 适用场景 | AirymaxOS 推荐度 |
|------|--------|---------|------------------|
| 1 | driver_override | 调试时强制绑定 | 仅调试 |
| 2 | OF（Device Tree） | ARM/ARM64/RISC-V SoC | **首选** |
| 3 | ACPI | x86 SoC | **首选**（x86） |
| 4 | id_table | 旧式平台设备 | 仅遗留代码 |
| 5 | 名称匹配 | 最简回退 | 不推荐 |

### 2.3 platform_probe 预处理

platform 总线的 `probe` 回调不是直接调用 driver 的 `probe`，而是先做预处理（`drivers/base/platform.c` 第 1351 行）：防御 `platform_driver_probe` 注册的不可再次绑定；应用 OF 默认时钟配置 `of_clk_set_defaults`；附加 PM 域 `dev_pm_domain_attach`；调用 driver 自身 probe；失败时回滚 PM 域；`prevent_deferred_probe` 标志的 driver 不允许延迟探测。

> **OS-DRV-022**: platform driver 的 `probe` 被调用时，`of_clk_set_defaults` 与 `dev_pm_domain_attach` 已完成——driver 无需重复处理时钟与 PM 域初始化。

> **OS-DRV-023**: 设置 `prevent_deferred_probe = true` 的 driver 必须在文档中显式声明原因。此标志禁用 `-EPROBE_DEFER` 重试机制，仅用于"必须立即可用"的关键设备（如调试串口）。

### 2.4 platform_remove 的 remove_new 优先

Linux 6.6 的 `platform_remove` 优先调用 `remove_new`（返回 `void`），回退到 `remove`（返回 `int` 但被忽略）：`if (drv->remove_new) drv->remove_new(dev); else if (drv->remove) { ... }`。最后调用 `dev_pm_domain_detach` 解绑 PM 域。

---

## 3. platform_device / platform_driver 结构

### 3.1 platform_device 结构

`struct platform_device`（`include/linux/platform_device.h` 第 23 行）核心字段：`name`（设备名称，用于名称匹配）、`id`（实例 ID，`PLATFORM_DEVID_NONE`=-1 表单实例，`PLATFORM_DEVID_AUTO`=-2 表自动分配）、`dev`（嵌入的通用 device）、`num_resources`/`resource`（设备资源数组）、`id_entry`（匹配到的 id_table 项）、`driver_override`（用户态强制绑定的 driver 名）、`mfd_cell`（MFD 子设备指针）、`archdata`（架构特定数据）。

### 3.2 platform_driver 结构

`struct platform_driver`（`include/linux/platform_device.h` 第 233 行）核心字段：

```c
struct platform_driver {
    int  (*probe)(struct platform_device *);
    int  (*remove)(struct platform_device *);       /* 旧式，返回 int（被忽略） */
    void (*remove_new)(struct platform_device *);    /* 新式，返回 void */
    void (*shutdown)(struct platform_device *);
    int  (*suspend)(struct platform_device *, pm_message_t state);
    int  (*resume)(struct platform_device *);
    struct device_driver driver;                     /* 嵌入的通用 driver */
    const struct platform_device_id *id_table;
    bool prevent_deferred_probe;
    bool driver_managed_dma;
};
#define to_platform_driver(drv) container_of((drv), struct platform_driver, driver)
```

`platform_driver_register` 内部调用 `driver_register`，并设置 `driver.bus = &platform_bus_type`，处理 `device*` 到 `platform_device*` 的适配。

> **OS-DRV-024**: AirymaxOS 内核态 driver 必须使用 `platform_driver_register`（或 `module_platform_driver` 宏）注册，禁止直接调用 `driver_register` 并手动设置 `bus` 字段。

---

## 4. of_device_id 匹配

### 4.1 完整 OF 匹配示例

`struct of_device_id` 核心字段：`name`（节点名匹配，可选）、`type`（设备类型匹配，可选）、`compatible`（compatible 字符串匹配，最常用，最长 128 字节）、`data`（匹配后传给 driver 的私有数据指针）。

```c
#include <linux/module.h>
#include <linux/platform_device.h>
#include <linux/of.h>

struct my_drvdata_v1 { u32 reg_offset; bool has_dma; };
struct my_drvdata_v2 { u32 reg_offset; bool has_dma; u32 fifo_size; };
static const struct my_drvdata_v1 v1_data = { .reg_offset = 0x100, .has_dma = false };
static const struct my_drvdata_v2 v2_data = { .reg_offset = 0x100, .has_dma = true, .fifo_size = 64 };

static const struct of_device_id my_match[] = {
    { .compatible = "spharx,my-soc-v1", .data = &v1_data },
    { .compatible = "spharx,my-soc-v2", .data = &v2_data },
    { /* sentinel — 必填 */ }
};
MODULE_DEVICE_TABLE(of, my_match);

static int my_probe(struct platform_device *pdev)
{
    const struct of_device_id *match = of_match_device(my_match, &pdev->dev);
    const void *data;
    void __iomem *base;
    struct resource *res;
    if (!match) return -ENODEV;
    data = match->data;  /* v1_data 或 v2_data */
    base = devm_platform_get_and_ioremap_resource(pdev, 0, &res);
    if (IS_ERR(base)) return PTR_ERR(base);
    dev_info(&pdev->dev, "compatible=%s, res=%pr\n", match->compatible, res);
    return 0;
}

static struct platform_driver my_driver = {
    .probe = my_probe,
    .driver = { .name = "my-soc-driver", .of_match_table = my_match, .pm = &my_pm_ops },
};
module_platform_driver(my_driver);
MODULE_AUTHOR("AirymaxOS Driver Team");
MODULE_DESCRIPTION("SoC device driver for my-soc");
MODULE_LICENSE("GPL v2");
```

> **OS-DRV-025**: OF 匹配表的 `.data` 字段必须指向静态 const 数据（`static const`），禁止指向栈变量或动态分配内存——`of_device_id` 表在模块加载后长期存在。

> **OS-STD-020**: `compatible` 字符串的 vendor 前缀必须与设备树中一致。AirymaxOS 自研设备使用 `spharx,` 前缀；通用设备沿用上游 vendor 前缀（如 `ti,`、`nxp,`）。

### 4.2 ACPI 匹配

```c
static const struct acpi_device_id my_acpi_match[] = {
    { "SPMX0001", (kernel_ulong_t)&v1_data },
    { "SPMX0002", (kernel_ulong_t)&v2_data },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(acpi, my_acpi_match);

static struct platform_driver my_driver = {
    .driver = {
        .name = "my-soc-driver",
        .of_match_table = my_match,         /* ARM/ARM64 路径 */
        .acpi_match_table = my_acpi_match,  /* x86 路径 */
    },
};
```

---

## 5. module_platform_driver 宏

### 5.1 宏展开

`module_platform_driver` 是消除 init/exit 样板的核心宏（`include/linux/platform_device.h` 第 299 行）：

```c
#define module_platform_driver(__platform_driver) \
    module_driver(__platform_driver, platform_driver_register, \
                  platform_driver_unregister)

/* module_driver 进一步展开（include/linux/device/driver.h 第 262 行） */
#define module_driver(__driver, __register, __unregister, ...) \
static int __init __driver##_init(void) \
{ return __register(&(__driver) , ##__VA_ARGS__); } \
module_init(__driver##_init); \
static void __exit __driver##_exit(void) \
{ __unregister(&(__driver) , ##__VA_ARGS__); } \
module_exit(__driver##_exit);
```

### 5.2 三种变体对比

| 宏 | 用途 | 限制 | AirymaxOS 推荐 |
|----|------|------|----------------|
| `module_platform_driver` | 标准模块驱动，可加载/卸载 | 模块可加载时使用 | **首选** |
| `builtin_platform_driver` | 内置驱动，不可卸载 | 仅 `device_initcall`，无 exit | 内核内置时使用 |
| `module_platform_driver_probe` | probe 函数在 `__init` 段，节省运行时内存 | 注册后不可解绑 | 仅遗留驱动 |

> **OS-DRV-026**: 新增 driver 必须使用 `module_platform_driver`（可加载）或 `builtin_platform_driver`（内置）。`module_platform_driver_probe` 因"注册后不可解绑"限制，仅用于维护遗留代码，新增代码禁用。

> **OS-STD-021**: 每个驱动源文件中 `module_platform_driver` 等注册宏只能出现一次。重复注册同一 driver 会触发 `driver_register` 的 `BUG_ON`。

### 5.3 多 driver 批量注册

当模块需注册多个 driver 时，使用 `platform_register_drivers`：传入 driver 指针数组与计数，`__init` 调用 `platform_register_drivers(drivers, count)`，`__exit` 调用 `platform_unregister_drivers(drivers, count)`。

---

## 6. DT/ACPI 双源回退

### 6.1 双源场景

AirymaxOS 的 SoC 设备可能部署在两类硬件上：ARM/ARM64/RISC-V SoC 使用 Device Tree 描述设备拓扑；x86 SoC（如 Intel/AMD 嵌入式）使用 ACPI 描述设备。同一 driver 若声明了 OF 与 ACPI 两个匹配表，`platform_match` 会先尝试 OF，再尝试 ACPI：

```mermaid
graph LR
    DEV["platform_device<br/>注册"] --> M1{"dev->of_node<br/>非空?"}
    M1 -- "是" --> OF["of_driver_match_device<br/>遍历 of_match_table"]
    OF --> R1{"匹配?"}
    R1 -- "是" --> BIND["绑定 driver"]
    R1 -- "否" --> M2
    M1 -- "否" --> M2{"has_acpi_companion<br/>非空?"}
    M2 -- "是" --> ACPI["acpi_driver_match_device"]
    ACPI --> R2{"匹配?"}
    R2 -- "是" --> BIND
    R2 -- "否" --> FALLBACK["id_table / 名称匹配"]
    M2 -- "否" --> FALLBACK
    style BIND fill:#e8f5e9
```

### 6.2 资源获取的统一接口

Linux 6.6 提供统一资源获取 API，屏蔽 DT/ACPI 差异：

| API | 功能 | DT/ACPI 透明 |
|-----|------|--------------|
| `platform_get_resource(pdev, IORESOURCE_MEM, idx)` | 获取内存资源 | 是 |
| `platform_get_irq(pdev, idx)` | 获取中断号 | 是（内部按 of_node/acpi 回退） |
| `devm_platform_ioremap_resource(pdev, idx)` | 获取并映射内存 | 是 |
| `device_property_read_u32(dev, "field", &val)` | 读设备属性 | 是 |
| `fwnode_property_read_string(dev_fwnode(dev), "name", &str)` | 读字符串属性 | 是 |

### 6.3 IRQ 获取的 DT/ACPI 回退

`platform_get_irq` 内部实现展示了 DT/ACPI 回退逻辑（`drivers/base/platform.c` 第 171 行）：优先从 `dev->dev.of_node` 通过 `of_irq_get` 获取；次选从 `platform_device` 的 `resource` 数组获取；回退到 ACPI 路径，若 `has_acpi_companion` 且资源 `IORESOURCE_DISABLED`，调用 `acpi_irq_get` 填充。

> **OS-DRV-027**: driver 必须使用 `platform_get_irq` 而非直接读 DT/ACPI 中断字段。前者处理了 DT/ACPI 双源回退与 `EPROBE_DEFER` 重试。

> **OS-DRV-028**: 设备属性（如时钟频率、FIFO 大小）必须通过 `device_property_read_*` 而非 `of_property_read_*` 读取。前者在 DT/ACPI 上都能工作，后者仅在 DT 设备上有效。

---

## 7. SoC 设备资源管理

### 7.1 resource 类型与获取

`struct resource`（`include/linux/ioport.h`）描述 SoC 设备硬件资源，核心字段 `start`/`end`/`name`/`flags`。`flags` 标识类型：`IORESOURCE_MEM`（内存映射寄存器）、`IORESOURCE_IO`（I/O 端口）、`IORESOURCE_IRQ`（中断号）、`IORESOURCE_DMA`（DMA 通道）。

```c
static int my_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct resource *mem_res;
    void __iomem *base;
    int irq;

    /* 1. 同时获取 resource 与映射（推荐） */
    base = devm_platform_get_and_ioremap_resource(pdev, 0, &mem_res);
    if (IS_ERR(base)) return PTR_ERR(base);

    /* 2. 获取中断号 */
    irq = platform_get_irq(pdev, 0);
    if (irq < 0) return irq;

    /* 3. 按名称获取（DT 中可命名） */
    irq = platform_get_irq_byname(pdev, "rx");
    if (irq < 0) return irq;

    /* 4. 多区域设备：遍历所有 mem 资源 */
    int i = 0;
    struct resource *r;
    while ((r = platform_get_resource(pdev, IORESOURCE_MEM, i++)))
        dev_info(dev, "region %d: %pr\n", i - 1, r);
    return 0;
}
```

### 7.2 devm_ 资源托管

SoC 设备的 probe 中所有 `devm_` 资源在 detach 时自动释放——这是 E-3 资源确定性的核心实践：

| 资源类型 | 非托管 API（需手动释放） | 托管 API（自动释放） |
|---------|------------------------|---------------------|
| 内存分配 | `kzalloc` / `kfree` | `devm_kzalloc` |
| 寄存器映射 | `ioremap` / `iounmap` | `devm_ioremap_resource` |
| 中断注册 | `request_irq` / `free_irq` | `devm_request_irq` |
| 时钟使能 | `clk_prepare_enable` / `clk_disable_unprepare` | `devm_clk_get` + `devm_add_action` |
| GPIO | `gpio_request` / `gpio_free` | `devm_gpio_request` |

> **OS-DRV-029**: SoC driver 的 `probe` 中所有可托管资源必须使用 `devm_` 系列。`remove_new` 中不应出现 `kfree`/`iounmap`/`free_irq` 调用——若出现，说明该资源未托管。

> **OS-DRV-030**: 自定义资源（如 Agent Token 预算）必须通过 `devm_add_action_or_reset` 注册释放回调：

```c
static void my_token_budget_release(void *data)
{
    struct my_token_budget *budget = data;
    agentrt_token_budget_destroy(budget);
}

static int my_probe(struct platform_device *pdev)
{
    struct my_token_budget *budget;
    budget = agentrt_token_budget_create(/* config */);
    if (IS_ERR(budget)) return PTR_ERR(budget);
    /* 注册托管释放 — probe 失败时也会调用（_or_reset 后缀） */
    return devm_add_action_or_reset(&pdev->dev, my_token_budget_release, budget);
}
```

### 7.3 SoC 设备的电源管理

platform 总线的 `pm` 回调由 `platform_dev_pm_ops` 提供，最终委托给 driver 的 `dev_pm_ops`：

```c
static const struct dev_pm_ops my_pm_ops = {
    SET_RUNTIME_PM_OPS(my_runtime_suspend, my_runtime_resume, NULL)
    SET_SYSTEM_SLEEP_PM_OPS(my_system_suspend, my_system_resume)
};

static struct platform_driver my_driver = {
    .driver = { .name = "my-soc-driver", .pm = &my_pm_ops },
};
```

> **OS-DRV-031**: SoC driver 必须实现 `runtime_suspend`/`runtime_resume`，至少禁用/启用时钟。即使设备不支持深度睡眠，空回调也需返回 0，避免 PM 子系统警告。

> **OS-DRV-032**: `runtime_resume` 必须在 `probe` 末尾通过 `pm_runtime_enable` 启用运行时 PM；`remove_new` 开头通过 `pm_runtime_disable` 禁用。顺序错误会导致 PM 子系统在设备已释放时调用回调。

---

## 8. 五维原则映射

platform 总线在 AirymaxOS 五维正交 24 原则上的映射：

| 原则 | 在本模块的体现 |
|------|---------------|
| **S-1 反馈闭环** | deferred probe 机制：依赖未就绪时返回 `-EPROBE_DEFER`，PM 域/时钟就绪后自动重试 |
| **S-2 层次分解** | platform_driver 包装 device_driver，platform_device 包装 device——每层只增加 SoC 特定语义 |
| **S-3 总体设计部** | platform_bus_type 是协调者——它不执行 probe，只决定"哪个 device 该绑哪个 driver" |
| **S-4 涌现性管理** | DT/ACPI 双源自动回退是 platform 总线的涌现行为——driver 只需声明两个匹配表 |
| **K-1 内核极简** | platform 总线不解释设备语义（如 Token 预算含义）——只提供匹配与资源管理 |
| **K-2 接口契约化** | `platform_get_resource`、`platform_get_irq`、`devm_platform_ioremap_resource` 契约完整声明 |
| **K-3 服务隔离** | platform_driver 失败不波及其他 driver；deferred probe 队列独立于注册顺序 |
| **K-4 可插拔策略** | 同一 platform_device 可在运行时通过 sysfs bind/unbind 切换 driver；`pm` 策略可替换 |
| **C-1 双系统协同** | sysfs 同步访问是慢路径，硬件中断是快路径，mutex 协同 |
| **C-2 增量演化** | deferred probe 允许设备在依赖未就绪时延后绑定，无需重启系统 |
| **C-3 记忆卷载** | `platform_set_drvdata` 持有 driver 私有数据；DT/ACPI 属性是"持久记忆" |
| **C-4 遗忘机制** | `device_unregister` 后 sysfs 节点立即移除，但 `kobject` 引用计数未归零前结构保留 |
| **E-1 安全内生** | `driver_override` 经 sysfs 写入需 CAP_SYS_ADMIN；`devm_` 资源在 detach 时强制释放 |
| **E-2 可观测性** | platform_bus 暴露 `/sys/bus/platform/`；`dev_info` 自动附带 device 名；modalias 支持 `modprobe` |
| **E-3 资源确定性** | `devm_` 系列强制资源生命周期与 device 绑定；`devm_add_action_or_reset` 支持自定义托管 |
| **E-4 跨平台一致性** | DT/ACPI 双源匹配让同一 driver 服务异构平台；`device_property_read_*` 屏蔽差异 |
| **E-5 命名语义化** | `module_platform_driver`、`platform_get_irq`、`devm_platform_ioremap_resource` 名称精确 |
| **E-6 错误可追溯** | probe 失败通过 `dev_err` 记录；deferred probe 在 dmesg 留下重试轨迹 |
| **E-7 文档即代码** | OF 匹配表与 DT binding 文档（`Documentation/devicetree/bindings/`）同步更新 |
| **E-8 可测试性** | driver 可作为模块加载/卸载；sysfs bind/unbind 支持运行时测试；KUnit 可测试 |
| **A-1 极简主义** | `module_platform_driver` 宏消除 init/exit 样板；`devm_` 系列消除手动释放 |
| **A-2 极致细节** | sentinel 必填、`remove_new` 优先 `void`、`sysfs_emit` 加固——每个细节都为安全 |
| **A-3 人文关怀** | `dev_info` 自动带设备名前缀；OF `compatible` 字符串人类可读 |
| **A-4 完美主义** | `-Wall -Wextra -Werror` 编译；OF 表无 sentinel 触发静态分析告警 |

> **OS-STD-022**: 任何新增 platform driver 必须在 `MODULE_DEVICE_TABLE(of, ...)` 或 `MODULE_DEVICE_TABLE(acpi, ...)` 中导出匹配表——否则 `modprobe` 无法根据硬件自动加载模块。

> **OS-STD-023**: 平台 driver 的 `.name` 字段必须与 `compatible` 字符串中的 device 部分一致（如 `compatible="spharx,my-soc-v2"` 对应 `name="my-soc-driver"`）。这是人类阅读 sysfs 时的认知锚点。

---

## 9. 同源 agentrt 映射

platform 总线在 AirymaxOS 用户态（agentrt）中的同源映射：

| 内核态 platform 抽象 | 用户态 agentrt 同源 | 映射说明 |
|---------------------|---------------------|---------|
| `platform_bus_type` | `agentrt_bus_type` | daemon 注册总线，匹配 Agent SDK 接口与运行时实现 |
| `platform_device` | `agentrt_agent_instance` | Agent 实例对应 platform_device |
| `platform_driver` | `agentrt_agent_module` | Agent 实现模块，含 probe/remove |
| `of_device_id` | `agentrt_sdk_interface_id` | Agent SDK 接口版本匹配 |
| `MODULE_DEVICE_TABLE(of, ...)` | `agentrt_module_export_interfaces()` | daemon 导出可被发现的接口 |
| `platform_get_resource` | `agentrt_module_get_resource()` | 获取 Token 预算、记忆卷等资源 |
| `devm_platform_ioremap_resource` | `devm_agentrt_token_budget_alloc` | 托管资源映射 |
| `module_platform_driver` | `agentrt_module_agent_driver` | 消除 daemon 注册样板 |
| sysfs `/sys/bus/platform/` | `agentrt-fs` `/agents/` | 用户态枚举 Agent 拓扑 |

### 9.1 IRON-9 同源但独立的实践

- **同源**：用户态 `agentrt_bus_type` 的匹配逻辑与内核 `platform_match` 在语义上一致——都先尝试精确匹配（OF/SDK 接口），再回退到名称匹配
- **独立**：AgentsIPC 是用户态 IPC，不依赖内核 uevent；内核态 platform driver 不依赖 daemon 存在
- **解耦验证**：删除 `agentos/daemon/` 后内核 platform driver 仍可正常 probe 硬件；删除内核 platform driver 后 daemon 仍可运行（但失去硬件加速）——这是 IRON-9 同源但独立的可验证标准

### 9.2 跨边界资源托管链

SoC 设备的完整资源托管链跨越内核/用户态：内核 `devm_` 资源在 device detach 时释放（最先）；用户态 `devm_agentrt_*` 资源在 daemon 退出时释放（次之）；Agent SDK 句柄在 Agent 卸载时清理（最后）。MicroCoreRT 与 AgentsIPC 分别承载内核态与用户态 IPC，两者不直接耦合。

> **OS-DRV-033**: 跨内核/用户态的资源托管链必须保证一致性——内核 `devm_` 资源先释放（device detach 触发），用户态 `devm_agentrt_*` 后释放（daemon 退出触发）。daemon 退出前必须等待内核 device detach 完成，否则会访问已释放的寄存器。

> **OS-KER-040**: 内核态 platform driver 的 `remove` 回调必须通知用户态 daemon（通过 uevent 或 AgentsIPC 消息），让 daemon 先释放对硬件资源的引用，再让内核释放硬件本身。这是跨边界资源托管链的关键时序保证。

---

## 10. 最佳实践与反模式

最佳实践的核心：用 `module_platform_driver` 宏消除样板；新代码用 `.remove_new` 返回 `void`；OF 表必填 sentinel；用 `devm_platform_get_and_ioremap_resource` 一步到位；用 `platform_get_irq` 而非 `platform_get_resource(IORESOURCE_IRQ)`；用 `device_property_read_*` 而非 `of_property_read_*`；`devm_` 托管所有可托管资源使 `remove_new` 几乎为空；`runtime_resume`/`runtime_suspend` 至少禁用/启用时钟；`MODULE_DEVICE_TABLE` 导出匹配表。

| 反模式 | 后果 | 正确做法 |
|--------|------|---------|
| `probe` 中 `ioremap` 后 `remove` 忘 `iounmap` | 寄存器空间泄漏 | 用 `devm_platform_ioremap_resource` |
| `probe` 中 `request_irq` 后 `remove` 忘 `free_irq` | 中断泄漏 + 崩溃 | 用 `devm_request_irq` |
| OF 表无 sentinel | 越界读取，未定义行为 | 必填 `{ /* sentinel */ }` |
| 用 `of_property_read_*` 读属性 | x86 平台失效 | 用 `device_property_read_*` |
| 用 `platform_get_resource(IORESOURCE_IRQ)` 取中断 | DT/ACPI 不一致 | 用 `platform_get_irq` |
| `probe` 中启用中断后才注册 handler | 中断在 handler 未就绪时触发 | 先 `devm_request_irq` 再启用 |
| `remove` 返回非 0 期望回滚 | 返回值被忽略 | `remove` 必须成功 |
| `runtime_resume` 不恢复时钟 | 硬件访问失效 | 至少 `clk_prepare_enable` |
| `MODULE_DEVICE_TABLE` 缺失 | `modprobe` 无法自动加载 | 必填导出表 |

> **OS-DRV-034**: `probe` 中 `devm_request_irq` 必须在硬件中断使能之前调用。若顺序颠倒，硬件可能在 handler 未注册时触发中断，导致空指针解引用。

> **OS-DRV-035**: `probe` 末尾必须调用 `pm_runtime_enable`，`remove_new` 开头必须调用 `pm_runtime_disable`。两者必须配对，缺失任一会触发 PM 子系统警告或资源泄漏。

> **OS-KER-041**: 内核态 platform driver 不应在 `probe` 中调用任何 `printk` 级别高于 `KERN_INFO` 的日志来报告"非致命"情况——这会污染 dmesg。`dev_dbg` 用于调试，`dev_info` 用于关键状态变更，`dev_err` 仅用于失败路径。

---

## 11. 相关文档

- `60-driver-model/README.md`（模块主索引）
- `60-driver-model/01-device-model.md`（device/driver/bus 三元组详解——本文档前置）
- `60-driver-model/03-devm-resource.md`（devm_ 资源管理与生命周期）
- `60-driver-model/04-misc-framework.md`（misc 框架与轻量级字符设备）
- `60-driver-model/05-agent-driver.md`（Agent 虚拟设备驱动扩展）
- `60-driver-model/06-driver-samples.md`（驱动示例集）
- `50-engineering-standards/01-coding-standards.md`（驱动代码规范）
- `20-modules/01-kernel.md`（kernel 子仓设计——MicroCoreRT 与内核态边界）
- `20-modules/02-services.md`（services 子仓设计——agentrt daemon 与 AgentsIPC）
- Linux 6.6 `drivers/base/platform.c`、`include/linux/platform_device.h`、`Documentation/driver-api/driver-model/platform.rst`

---

## 12. 文档版本与维护

| 版本 | 日期 | 维护者 | 变更摘要 |
|------|------|--------|---------|
| 0.1.1 | 2026-07-06 | AirymaxOS 驱动子系统组 | 占位版本，建立文档骨架与规则编号体系 |
| 1.0.1 | 2026-07-06 | AirymaxOS 驱动子系统组 | 开发版本，完成 platform 总线、device/driver 结构、OF/ACPI 匹配、宏、双源回退、资源管理六章正文 |

### 12.1 维护规则

- **同步性**：与 Linux 6.6 内核基线 `drivers/base/platform.c` 同步——上游 platform_bus_type 字段或 `platform_match` 优先级变更时本文档需在下一个版本周期内更新
- **规则编号稳定性**：`OS-DRV-020`~`OS-DRV-035`、`OS-STD-020`~`OS-STD-023`、`OS-KER-040`~`OS-KER-041` 编号一旦分配不再变更
- **同源验证**：每次发布前通过 `scripts/check-platform-driver-sync.sh` 验证文档中 API 签名与 Linux 6.6 头文件一致
- **原则映射完整性**：五维原则映射章节覆盖全部 24 条原则

### 12.2 一致性检查清单

发布前验证：`platform_bus_type` 字段与 Linux 6.6 第 1454 行一致；`struct platform_device`/`platform_driver` 字段与头文件一致；`platform_match` 五级回退与源码一致；`module_platform_driver` 宏展开与第 299 行一致；OF/ACPI 示例可编译；五维映射覆盖全部 24 条原则；同源映射与 `20-modules/02-services.md` 一致。

> **文档结束** | AirymaxOS 驱动模型 60 模块第二篇 | IRON-9 同源但独立 | Linux 6.6 内核基线
