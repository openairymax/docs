# Airymax 统一错误码规范

> **版本**: v0.1.0-draft | **状态**: 草案 | **日期**: 2026-07-04
> **版权归属人**: SPHARX Ltd. | **许可证**: AGPL-3.0-or-later OR Apache-2.0（双许可证，SPHARX Ltd.）

---

## 1. 现状问题（已验证）

### 1.1 Cupolas 三套并行错误码系统（CONFIRMED）

| 错误码系统 | 位置 | 问题 |
|-----------|------|------|
| commons 权威码 | `commons/utils/error/include/error.h:111-120` | `AGENTOS_ERR_INVALID_PARAM=-2` 等，权威源 |
| cupolas enum | `cupolas/src/security/cupolas_error.h:39-60` | `cupolas_ERR_INVALID_PARAM=-2` 等，重复定义（独立前缀） |
| cupolas 本地 #define | `cupolas/src/platform/platform.h:846-879` | `cupolas_ERROR_NO_MEMORY=-3` 等，**与 enum 数值冲突** |

### 1.2 数值冲突详情

| 语义 | commons 权威值 | cupolas enum 值 | cupolas 本地 #define 值 |
|------|:---:|:---:|:---:|
| NO_MEMORY | -4 | -4 | **-3** ⚠️ |
| NOT_FOUND | -6 | -6 | **-4** ⚠️ |
| PERMISSION | -10 | -10 | **-5** ⚠️ |
| TIMEOUT | -8 | -8 | **-7** ⚠️ |
| OVERFLOW | -14 | -14 | **-9** ⚠️ |
| NOT_SUPPORTED | -9 | -9 | **-10** ⚠️ |

### 1.3 模块专属错误码区间重叠
- `cupolas_sig_error_t`、`cupolas_ent_error_t`、`cupolas_vault_error_t`、`cupolas_net_error_t`、`cupolas_runtime_error_t` 全部从 0/-1 起编号
- 靠 `cupolas_error_from_*` 转换函数桥接（脆弱）

## 2. 标准规范

### 2.1 权威源唯一性（ARE-ERR-01）

`commons/utils/error/include/error.h` 是全项目唯一错误码权威源。禁止任何模块（含 cupolas）重复定义通用错误码。

### 2.2 Cupolas 错误码统一（ARE-ERR-02）

- 删除 `cupolas/src/platform/platform.h:846-879` 的本地 `#define` 错误码
- 删除 `cupolas/src/security/cupolas_error.h:39-60` 的 `cupolas_ERR_*` enum（重复定义）
- cupolas 直接复用 `AGENTOS_ERR_*` 宏

### 2.3 模块专属错误码分段（ARE-ERR-03）

cupolas 模块专属错误码（非通用）必须使用独立分段：

| 段 | 范围 | 用途 |
|----|------|------|
| 通用 | -1 ~ -99 | commons 权威源 |
| 系统 | -100 ~ -199 | 系统与平台 |
| 内核 | -200 ~ -299 | 内核层 |
| 服务 | -300 ~ -399 | 服务层 |
| LLM | -400 ~ -499 | LLM/AI 服务 |
| 执行 | -500 ~ -599 | 执行/工具 |
| 记忆 | -600 ~ -699 | 记忆/存储 |
| 安全 | -700 ~ -799 | cupolas 专属 |
| 协议 | -800 ~ -899 | 协议层 |

cupolas 模块专属错误码必须在 -700 ~ -799 段内定义，避免与通用码重叠。

## 3. 修复任务

详见 [0.1.1详细任务清单.md](../../../DocsClosed/0.1.1详细任务清单.md) P0.25（Cupolas 错误码统一，12h）

---

<div align="center">

**© 2026 SPHARX Ltd.**

</div>
