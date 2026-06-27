Copyright (c) 2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# Airymax C 语言编码规范

**最新**: 2026-06-09
**状态**: 维护中
**路径**: OpenAirymax/Docs/Capital_Specifications/coding_standard/C_coding_style_standard.md
---

## 一、核心理念

本规范基于 Airymax 架构设计原则的五维正交设计体系：

- **《工程控制论》**（原则 S-1, E-2）：通过错误码、日志、健康检查和指标构建反馈闭环，使系统能自我观测并对异常自动响应
- **《论系统工程》**（原则 S-2, K-2）：模块化、接口驱动，边界清晰、实现可替换
- **Thinkdual 双思考系统**（原则 C-1）：提供 t1 快思考（快速、低延迟）与 t2 慢思考（安全、全面）两条路径，并允许运行时策略切换
- **微核心哲学**（原则 K-1, K-4）：接口精炼、命名优雅、注释说明"为什么"，而非"做什么"
- **设计美学**（原则 A-1, A-2, A-4）：代码结构简约对称，命名自解释，注释清晰优雅，追求完美主义

**关联原则**:
- E-1 安全内生原则：代码中内置安全检查，防止安全漏洞
- E-3 资源确定性原则：明确资源所有权和生命周期管理
- E-4 跨平台一致性原则：确保代码在不同平台上的行为一致性
- E-5 命名语义化原则：命名清晰表达意图，避免歧义
- E-6 错误可追溯原则：错误信息包含完整上下文，支持根源分析
- E-8 可测试性原则：代码设计支持全面的单元测试、集成测试和性能测试

---

## 二、范围与术语

- **agentos_error_t：** 统一错误码类型（int32_t）；提供 `agentos_strerror()` 输出可读字符串。
- **公共头（public headers）：** 位于 `module/include/`，仅包含稳定对外 API。
- **私有实现：** 位于 `module/src/` 或 `module/src/internal/`。
- **Trace/Span：** 遵循 OpenTelemetry 与 W3C Trace Context（`traceparent`）规范。
- **SBOM：** 由 CI 生成的软件物料清单（SPDX 或 CycloneDX），随发布附件提供。

---

## 三、目录模板（强制）

每个模块必须且仅包含以下五项顶级内容（保证模块一致性与可打包性）：
- include/           —— 稳定公共头（导出 API）
- src/               —— 实现（可含 src/internal/）
- tests/             —— 单元与集成测试
- CMakeLists.txt     —— 构建、导出与安装规则
- README.md          —— 模块概述、API 兼容级别、配置与示例

**说明：** include/ 仅放对外接口；实现细节保留在 src/。README.md 须声明 API 版本（MAJOR.MINOR）、支持平台与快速上手示例。tests/ 至少包含一个单元测试与一个可在 CI 中重复运行的集成测试用例。

---

## 四、命名与可见性

- **命名风格：** 函数/变量使用 snake_case；宏/常量使用 UPPER_CASE；类型以 `_t` 结尾。
- **公共符号：** 需带模块前缀（如 `atoms_`、`cupolas_`、`cognition_`、`llm_`）。
- **私有符号：** 跨文件符号使用 `_module_` 前缀或声明为 `static` 并置于 internal header。
- **可见性控制：** 必须在release构建中启用 `-fvisibility=hidden` 并显式导出公共符号。

---

## 五、API 与 ABI 管理

- **版本声明：** 公共头文件顶部必须声明 API 版本宏，例如 `#define MODULE_API_VERSION 1`。
- **ABI 兼容：** 在相同 MAJOR 版本内保证 ABI 兼容；破坏性更改需递增 MAJOR 并发布迁移说明。
- **不透明指针：** 优先使用不透明指针（opaque types）作为导出类型，避免直接暴露结构字段。
- **符号管理：** 建议采用符号版本管理（ELF symbol versioning）或显式导出列表来管理符号演进（推荐但可选）。

---

## 六、函数设计与文档

- **单一职责且短小：** 必须 ≤80 行；推荐 ≤50 行。若 >80 行必须重构并添加单元测试。
- **参数数量：** 不超过 5 个，若超过请封装成结构体。
- **文档要求：** 公共 API 必须使用 Doxygen 风格注释，且明确 ownership（谁释放）、线程语义与错误语义。
- **返回约定：** 优先使用 `agentos_error_t`（int32_t），或返回指针（失败返回 NULL 并记录日志）。提供常用错误定义与辅助宏。

**示例：**

```c
/**
 * @brief 提交计划并返回任务 ID。
 * @param engine [in] 引擎句柄（非 NULL）。
 * @param plan   [in] 待调度计划（只读）。
 * @param out_id [out] 输出任务 ID（调用者负责 free）。
 * @return AGENTOS_OK (0) 成功；其他值为错误码。
 */
int cognition_schedule(cognition_engine_t* engine,
                       const task_plan_t* plan,
                       char** out_id);
```

---

## 七、错误处理与诊断

- 使用统一错误类型 `agentos_error_t`，并提供 `agentos_strerror()`。
- 错误码命名带模块前缀（例如 `COG_ERR_OUT_OF_MEMORY`）。
- 每条错误路径至少记录 WARN/ERROR 级别日志，日志中须包含 `trace_id` 及关键信息（不可包含敏感数据）。
- 在测试构建中保留 fault-injection 钩子，用以验证错误路径与降级策略。

---

## 八、资源与生命周期管理

- **成对管理：** 所有资源按 create/destroy 成对管理，并在 API 文档中明确释放责任。
- **推荐使用 scope-guard：** 如 cleanup attribute / 宏，以减少手动释放错误。
- **realloc 安全用法：** 必使用临时指针以避免原指针丢失：

```c
void* tmp = realloc(ptr, new_size);
if (!tmp) {
    // 处理失败，ptr 仍然有效
}
ptr = tmp;
```

- **内存配额：** 长期运行服务应有内存配额与回收策略，并在健康检查中暴露相关指标。

---

## 九、并发与线程安全

- **优先使用 C11 atomics：** 使用 `stdatomic.h`，避免使用遗留的 `_sync*`。
- **热路径优化：** 优先无锁/原子实现，并提供锁保护的安全回退（System2）。
- **并发合约：** 为每个 API 明确并发合约：线程安全/不可重入/需外部同步等。
- **锁顺序：** 避免锁嵌套；若必须嵌套，定义并文档化全局锁顺序。
- **跨平台线程抽象：** 禁止直接使用 `pthread.h` API（违反 CROSS-01）。必须通过 `platform.h` 抽象层使用：
  - `agentos_thread_create()` 替代 `pthread_create()`
  - `agentos_mutex_lock()` 替代 `pthread_mutex_lock()`
  - `agentos_cond_wait()` 替代 `pthread_cond_wait()`
  - `agentos_tls_get/set()` 替代 `pthread_key_t` 保存 per-request 上下文（如 `trace_id`/`request_id`）
- **线程局部存储：** 使用 `AGENTOS_THREAD_LOCAL` 宏（`platform.h` 提供跨平台实现），禁止直接使用 `__thread` 或 `pthread_key_t`。

---

## 十、日志、追踪与指标（可观测性）

- **日志格式：** 使用结构化 JSON（兼容 OpenTelemetry logs），最小字段：`timestamp`、`level`、`module`、`function`、`trace_id`、`request_id`、`message`、`err_code`。
- **追踪规范：** 采用 W3C Trace Context（`traceparent`），并在请求入口生成 `trace_id`。
- **指标格式：** 采用 Prometheus exposition 格式；模块应提供 `/metrics`、`/healthz`（liveness）与 `/readyz`（readiness）端点。
- **日志采样：** 高吞吐场景需实现日志采样或汇总策略以防日志风暴，但错误日志应保持完整。

---

## 十一、配置管理与热重载

- **配置优先级：** `defaults ← manager file ← environment variables ← runtime overrides`。
- **配置验证：** 使用 YAML/JSON Schema 验证配置，启动时拒绝非法配置。
- **热重载：** 应以原子方式替换配置句柄（或使用读写锁），确保短期向后兼容。敏感配置应通过 secret manager 注入，不写入普通文件。

---

## 十二、安全实践与运行时加固

### 12.1 编译/链接硬化（release 构建）

必须在release构建中启用以下编译/链接硬化选项：

- `-fstack-protector-strong`
- `-D_FORTIFY_SOURCE=2`
- `-fPIE -pie`
- `-Wl,-z,relro -Wl,-z,now`
- `-fvisibility=hidden`
- 在适当时启用 LTO

### 12.2 Sanitizers

在 CI 矩阵中运行 Sanitizers：ASAN、UBSAN、TSAN、MSAN（按目标配置）。

### 12.3 模糊测试

对解析器与外部输入持续做模糊测试（libFuzzer/AFL++），并保留回归 corpus。

### 12.4 供应链安全

- CI 生成 SBOM（SPDX/CycloneDX）
- 定期依赖漏洞扫描
- 发布时用 Sigstore/Cosign 签名制品

### 12.5 运行时安全

- **最小权限原则：** 降权运行、限制文件/网络权限、并在支持的平台上使用 seccomp/AppArmor。
- **敏感数据保护：** 严禁在日志中输出密钥或敏感数据，使用脱敏/掩码工具。

---

## 十三、构建、静态/动态分析与 CI

### 13.1 最低 CI 流程

1. 格式检查（clang-format）
2. 构建（CMake）
3. 静态分析（clang-tidy / cppcheck）
4. 单元测试（coverage）
5. Sanitizer jobs（ASAN/UBSAN/TSAN/MSAN）
6. Fuzz job（持续/定期运行）
7. 依赖与许可检查、SBOM 生成、制品签名与发布

### 13.2 pre-commit 钩子

必须配置以下 pre-commit 钩子：

- clang-format
- license header 检查
- 脚本 shellcheck
- commit-msg（Conventional Commits）

### 13.3 CI 阻断策略

CI 在发现高危安全问题或格式/静态分析回归时应阻止合并。

---

## 十四、测试策略（原则 E-8 可测试性原则）

- **单元测试：** 使用 CMocka、Criterion 或等效框架，覆盖成功、失败与边界条件。
- **集成测试：** 容器化或隔离的测试环境，模拟外部依赖（如 LLM mock、数据库 mock 等）。
- **模糊测试：** 对解析器、反序列化器和网络协议持续 fuzz，并保存回归样本。
- **性能基准：** 对关键路径提供 microbenchmarks，并对比回归。
- **覆盖率目标：** 关键模块 ≥ 85%（可根据模块重要性分级）；CI 可根据策略阻断或发出警告。

---

## 十五、发布与供应链合规

- **语义化版本：** 使用 SemVer。每次发布附带 changelog、SBOM 与签名文件。
- **漏洞阈值：** 若依赖漏洞超过阈值（高危），应阻断发布。
- **可重现构建：** 每次发布应保留可重现构建元数据（构建日志、编译标志、SBOM、签名）。

---

## 十六、开发流程与代码审查

- **提交规范：** 提交与 PR 遵循 Conventional Commits。PR 必包含相应测试与文档更新。
- **CODEOWNERS：** 在仓库中设置 CODEOWNERS，关键模块由指定人员审查；关键模块 PR 至少需要两位 reviewer。
- **Review 清单（必检项）：** API/ABI 影响、测试覆盖、格式/静态分析/sanitizer 结果、SBOM/第三方许可影响与安全考虑。
- **BAN 检查：** PR 合并前必须通过 BAN-01~BAN-13 自动扫描（零违规）。
- **CROSS 检查：** PR 合并前必须通过 CROSS-01~CROSS-06 跨平台规范检查。

---

## 十七、跨平台编译规范（CROSS-01~06）

所有 C 代码必须遵循以下跨平台规范，确保在 MSVC（Windows）和 GCC/Clang（Linux/macOS）上均可编译通过：

### CROSS-01：禁止直接使用 POSIX 线程 API

**违规示例**: `pthread_create()`, `pthread_mutex_lock()`, `pthread_cond_wait()`
**正确做法**: 全部替换为 `platform.h` 提供的 `agentos_thread_create()`, `agentos_mutex_lock()`, `agentos_cond_wait()`

### CROSS-02：禁止使用 C99 VLA 变长数组

**违规示例**: `int arr[n]`
**正确做法**: `int* arr = AGENTOS_MALLOC(n * sizeof(int))` + 对应 `AGENTOS_FREE`

> **Airymax 内存宏完整列表**（定义在 `memory_compat.h`）：
> - `AGENTOS_MALLOC(size)` —— 带返回值检查的 malloc
> - `AGENTOS_CALLOC(nmemb, size)` —— 带返回值检查的 calloc（零初始化）
> - `AGENTOS_REALLOC(ptr, size)` —— 安全 realloc（使用临时指针，避免原指针丢失）
> - `AGENTOS_STRDUP(s)` —— 带返回值检查的 strdup
> - `AGENTOS_STRNDUP(s, n)` —— 带返回值检查的 strndup
> - `AGENTOS_FREE(ptr)` —— 安全释放（释放后置 NULL）
> - `AGENTOS_STRNCPY_TERM(dst, src, dst_size)` —— 安全字符串拷贝（保证 null 终止）
> - `AGENTOS_MEMCPY_SAFE(dst, dst_size, src, src_size)` —— 安全内存拷贝（带边界检查）
> - `SAFE_MALLOC_ARRAY(type, count)` —— 安全数组分配（带整数溢出检查）
> - `AGENTOS_SECURE_ZERO(ptr, size)` —— 安全内存清零（防止编译器优化，跨平台替代 `explicit_bzero`）

> **安全释放宏**（定义在 `agentos_memory.h`）：
> - `MEMORY_FREE_SAFE(pptr)` —— 安全释放宏，接受**指针的指针**，释放后自动将原指针置 NULL。相比 `AGENTOS_FREE`（接受指针本身），`MEMORY_FREE_SAFE` 通过二级指针彻底消除悬垂指针风险：
>   ```c
>   char* buf = AGENTOS_MALLOC(1024);
>   MEMORY_FREE_SAFE(&buf);  // 释放并置 buf = NULL
>   // 此时 buf 一定为 NULL，不会误用
>   ```

> **禁止函数清单**（定义在 `banned_functions.h`）：
> Airymax 在严格模式下禁止使用以下 30 个不安全函数，编译时由 `banned_functions.h` 触发编译错误：
> - 字符串操作：`strcpy`, `strcat`, `sprintf`, `vsprintf`, `gets`, `scanf`, `sscanf` 等
> - 内存操作：直接使用 `malloc`/`free`/`realloc`/`calloc`/`strdup`（必须使用 `AGENTOS_*` 宏替代）
> - 格式化输出：`printf`（生产代码应使用日志宏 `AGENTOS_LOG_*` / `SVC_LOG_*`）
> - 时间函数：`localtime`, `ctime`, `asctime`（非线程安全）
> - 其他：`rand`, `strtok`, `tmpnam` 等
>
> **BAN 规则交叉引用**：
> | BAN 编号 | 禁止模式 | 对应宏/替代方案 |
> |----------|---------|---------------|
> | BAN-70 | 禁止使用 `printf` 系列直接输出 | 使用 `AGENTOS_LOG_*` 或 `SVC_LOG_*` 日志宏 |
> | BAN-71 | 禁止直接使用 `malloc`/`free` | 使用 `AGENTOS_MALLOC`/`AGENTOS_FREE`（memory_compat.h） |
> | BAN-72 | 禁止直接使用 `realloc` | 使用 `AGENTOS_REALLOC`（使用临时指针，避免原指针丢失） |
> | BAN-73 | 禁止使用 `strcpy`/`strcat`/`sprintf` | 使用 `AGENTOS_STRNCPY_TERM`/`AGENTOS_MEMCPY_SAFE`（memory_compat.h） |
> | BAN-74 | 禁止使用 `memset` 清零敏感数据 | 使用 `AGENTOS_SECURE_ZERO`（memory_compat.h） |

### CROSS-03：禁止直接使用 POSIX 时间函数

**违规示例**: `clock_gettime(CLOCK_MONOTONIC, &ts)`
**正确做法**: 使用 `agentos_time_ns()`, `agentos_time_ms()`

### CROSS-04：禁止使用 GCC 扩展语法

**违规示例**: 嵌套函数指针、语句表达式
**正确做法**: 使用 ANSI C 标准语法

### CROSS-05：Windows 头文件保护

**违规示例**: `#define NOMINMAX` 缺少 `#ifndef`
**正确做法**: 使用 `#ifndef NOMINMAX` / `#define NOMINMAX` / `#endif` 三行保护

### CROSS-06：MSVC /WX 级别零警告

**要求**: MSVC 编译必须 `/WX` 标志零警告通过，应移除所有未使用变量和隐式类型转换

---

## 十八、禁止模式清单（BAN-01~74）

所有 PR 必须通过以下禁止模式的自动扫描：

| 编号 | 禁止模式 | 检测命令 |
|------|---------|----------|
| BAN-01 | `return AGENTOS_SUCCESS` 占位返回 | `grep -rn "return AGENTOS_SUCCESS" --include="*.c"` |
| BAN-02 | `(void)param` 忽略参数 | `grep -rn "(void)" --include="*.c"` |
| BAN-03 | 空函数体 `{}` | 代码审查 |
| BAN-04 | TODO/FIXME 无关联 Issue | `grep -rn "TODO\|FIXME" --include="*.c"` |
| BAN-05 | simplified/placeholder/stub 注释 | `grep -rn "simplified\|placeholder\|stub" --include="*.c"` |
| BAN-06 | `return STRDUP(input)` 回显 | 代码审查 |
| BAN-07 | `(void*)1` 假指针 | 代码审查 |
| BAN-08 | `while(cond){break;}` 假循环 | 代码审查 |
| BAN-09 | `free("literal")` 释放字符串字面量 | `grep -rn 'free("' --include="*.c"` |
| BAN-10 | `*(ptr)=val` 假原子操作 | 代码审查 + TSan |
| BAN-11 | `*_stub.h` 桩头文件 | `find . -name "*_stub.h"` |
| BAN-12 | 降级模式返回假成功 | `grep -rn "degraded\|fallback.*SUCCESS" --include="*.c"` |
| BAN-13 | DUMMY 假数据 | `grep -rn "DUMMY\|dummy" --include="*.c" --include="*.h"` |
| BAN-70 | 直接使用 `printf` 系列输出 | `grep -rn "printf(" --include="*.c" --include="*.h"` |
| BAN-71 | 直接使用 `malloc`/`free` | `grep -rn "\\bmalloc\\b\|\\bfree\\b" --include="*.c" --include="*.h"` |
| BAN-72 | 直接使用 `realloc` | `grep -rn "\\brealloc\\b" --include="*.c" --include="*.h"` |
| BAN-73 | 使用 `strcpy`/`strcat`/`sprintf` | `grep -rn "\\bstrcpy\\b\|\\bstrcat\\b\|\\bsprintf\\b" --include="*.c" --include="*.h"` |
| BAN-74 | 使用 `memset` 清零敏感数据 | `grep -rn "memset.*key\|memset.*password\|memset.*secret\|memset.*token" --include="*.c"` |

> **替代方案**：BAN-70~74 的替代方案见 §17 CROSS-02 中的宏列表和 BAN 规则交叉引用表。

---

## 附录 A — 推荐编译选项（CMake 示例）

```cmake
# Release 构建建议 flags
target_compile_options(${target} PRIVATE
  -Wall -Wextra -Werror
  -Wformat -Wformat-security
  -fstack-protector-strong
  -D_FORTIFY_SOURCE=2
  -fPIE
  -fno-omit-frame-pointer
)

target_link_options(${target} PRIVATE
  -Wl,-z,relro -Wl,-z,now -pie
)

set(CMAKE_INTERPROCEDURAL_OPTIMIZATION_RELEASE ON) # 视情况启用 LTO
```

---

## 附录 B — 示例代码片段（精简且惯用）

### B.1 资源分配与清理

```c
service_t* service_create(const char* path) {
    if (!path) return NULL;
    
    service_t* svc = calloc(1, sizeof(*svc));
    if (!svc) return NULL;
    
    svc->file = fopen(path, "rb");
    if (!svc->file) goto fail;
    
    // ... 初始化 ...
    
    return svc;
    
fail:
    service_destroy(svc);
    return NULL;
}
```

### B.2 无锁环形缓冲骨架（简化）

```c
typedef struct ring_buffer {
    void** buffer;
    size_t mask; // capacity = mask + 1，且为 2 的幂
    atomic_size_t head;
    atomic_size_t tail;
} ring_buffer_t;
```

### B.3 健康检查示例

```c
char* module_health_check(void) {
    cJSON* root = cJSON_CreateObject();
    cJSON_AddStringToObject(root, "status", "healthy");
    cJSON_AddNumberToObject(root, "requests_total",
                           (double)atomic_load(&requests_total));
    
    char* json = cJSON_PrintUnformatted(root);
    cJSON_Delete(root);
    
    return json; // 调用者需 free
}
```

---

## 附录 C — 常用错误码（示例）

> 错误码权威定义见 agentos/commons/utils/error/include/error.h，以下仅为示例。

```c
#define AGENTOS_OK                       0
#define AGENTOS_ERR_INVALID_PARAM       -2
#define AGENTOS_ERR_OUT_OF_MEMORY       -4
#define AGENTOS_ERR_BUSY                -17
#define AGENTOS_ERR_TIMEOUT             -8
```

---

## 版权

© 2026 SPHARX Ltd. All Rights Reserved.
