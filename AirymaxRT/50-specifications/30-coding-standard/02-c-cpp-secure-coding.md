Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# Airymax C&C++安全编程指南

**最新**: 2026-06-09
**状态**: 维护中
**路径**: OpenAirymax/Docs/50-specifications/30-coding-standard/02-c-cpp-secure-coding.md
---

## 1. 内存安全

### 1.1 缓冲区溢出防护

所有涉及固定大小缓冲区的操作必须使用边界检查函数：

| 不安全函数 | 安全替代 |
|-----------|---------|
| `strcpy(dst, src)` | `strncpy(dst, src, dst_size)` 或 `snprintf(dst, dst_size, "%s", src)` |
| `strcat(dst, src)` | `strncat(dst, src, remaining)` |
| `sprintf(buf, fmt, ...)` | `snprintf(buf, buf_size, fmt, ...)` |
| `gets(buf)` | 禁止使用 `fgets(buf, buf_size, stdin)` |
| `scanf("%s", buf)` | `fgets(buf, buf_size, stdin)` 或限制宽度 `scanf("%Ns", buf)` |

> **⚠️ strncpy 已知问题**：`strncpy` 本身存在以下缺陷：
> 1. **不保证 null 终止**：当源字符串长度 ≥ 目标缓冲区大小时，目标缓冲区不会以 `'\0'` 结尾
> 2. **性能问题**：当源字符串远短于目标缓冲区时，`strncpy` 会用 `'\0'` 填充剩余空间，造成不必要的开销
>
> **推荐替代方案**（优先级从高到低）：
> - `AGENTOS_STRNCPY_TERM(dst, src, dst_size)` —— Airymax 安全拷贝宏，自动保证 null 终止（定义在 `memory_compat.h`）
> - `snprintf(dst, dst_size, "%s", src)` —— 标准库方案，始终保证 null 终止
> - `strncpy(dst, src, dst_size - 1); dst[dst_size - 1] = '\0';` —— 手动补零，仅在前两者不可用时使用

**违规示例**：
```c
char buf[64];
strcpy(buf, user_input);  // ❌ buffer overflow
```

**正确做法**：
```c
char buf[64];
strncpy(buf, user_input, sizeof(buf) - 1);
buf[sizeof(buf) - 1] = '\0';
```

### 1.2 内存分配检查

所有内存分配操作必须检查返回值：

```c
// ✅ 正确
void* ptr = malloc(size);
if (!ptr) {
    log_error("Out of memory");
    return AGENTOS_ENOMEM;
}

// ✅ 使用 AGENTOS_MALLOC 宏（自带检查）
void* ptr = AGENTOS_MALLOC(size);
```

**禁止**：直接使用 `malloc` 而不检查返回值；使用 `calloc` 而不初始化。

### 1.2.1 Airymax 安全宏（推荐）

Airymax 在 `memory_compat.h` 中提供以下安全宏，所有 C 代码应优先使用：

| 宏名称 | 功能 | 说明 |
|--------|------|------|
| `AGENTOS_MALLOC(size)` | 带返回值检查的 malloc | 分配失败时自动记录日志并返回 NULL |
| `AGENTOS_CALLOC(nmemb, size)` | 带返回值检查的 calloc | 零初始化内存，分配失败时自动记录日志 |
| `AGENTOS_REALLOC(ptr, size)` | 安全 realloc（使用临时指针） | 避免原指针丢失 |
| `AGENTOS_STRDUP(s)` | 带返回值检查的 strdup | 分配失败时自动记录日志 |
| `AGENTOS_STRNDUP(s, n)` | 带返回值检查的 strndup | 分配失败时自动记录日志 |
| `AGENTOS_STRNCPY_TERM(dst, src, dst_size)` | 安全字符串拷贝 | 自动保证 null 终止 |
| `AGENTOS_MEMCPY_SAFE(dst, dst_size, src, src_size)` | 安全内存拷贝 | 带边界检查，防止缓冲区溢出 |
| `SAFE_MALLOC_ARRAY(type, count)` | 安全数组分配 | 带整数溢出检查的数组分配 |
| `AGENTOS_SECURE_ZERO(ptr, size)` | 安全内存清零 | 防止编译器优化，跨平台替代 `explicit_bzero` |

### 1.3 释放后使用（Use-After-Free）防护

```c
// ✅ 正确：释放后立即置空
free(ptr);
ptr = NULL;

// ❌ 错误：释放后继续使用
free(ptr);
ptr->field = value;  // use-after-free
```

### 1.4 双重释放防护

```c
// ✅ 正确：释放前检查
if (ptr) {
    free(ptr);
    ptr = NULL;
}
```

### 1.5 资源所有权明确

所有资源遵循 create/destroy 成对管理原则，在 API 文档中明确释放责任：

```c
/**
 * @brief 创建资源
 * @return 资源句柄（调用者负责通过 resource_destroy 释放）
 */
resource_t* resource_create(void);

/**
 * @brief 销毁资源
 * @param res 资源句柄（NULL 安全）
 */
void resource_destroy(resource_t* res);
```

---

## 2. 指针安全

### 2.1 空指针检查

所有函数入口必须对指针参数进行有效性检查：

```c
int process_data(data_t* data) {
    if (!data) return AGENTOS_EINVAL;     // ✅ 空指针检查
    if (!data->buffer) return AGENTOS_EINVAL;  // ✅ 成员空指针检查
    // ... 处理逻辑
}
```

### 2.2 悬挂指针防护

避免返回局部变量地址：

```c
// ❌ 错误：返回局部变量地址
int* get_value(void) {
    int val = 42;
    return &val;  // 函数返回后 val 被销毁
}

// ✅ 正确：使用堆分配或 static
int* get_value(void) {
    int* val = malloc(sizeof(int));
    if (val) *val = 42;
    return val;
}
```

### 2.3 类型转换安全

避免不必要的类型转换，必须转换时使用显式转换：

```c
// ✅ 正确：显式转换
uintptr_t addr = (uintptr_t)ptr;

// ❌ 错误：通过 int 截断指针
int addr = (int)ptr;  // 64位系统上指针被截断
```

---

## 3. 并发安全

### 3.1 跨平台线程抽象

禁止直接使用 `pthread.h` API（违反 CROSS-01），必须使用 `platform.h` 抽象层：

```c
#include "platform.h"

// ✅ 正确
agentos_thread_t thread;
agentos_mutex_t mutex;
agentos_cond_t cond;

agentos_thread_create(&thread, func, arg);
agentos_mutex_lock(&mutex);
agentos_cond_wait(&cond, &mutex);
agentos_mutex_unlock(&mutex);
```

### 3.2 原子操作

优先使用 C11 `stdatomic.h` 原子操作：

```c
#include <stdatomic.h>

atomic_int counter = 0;
atomic_fetch_add(&counter, 1);
```

当 `AGENTOS_USE_STDATOMIC` 未定义时，使用 `platform.h` 提供的跨平台原子宏。

### 3.3 锁顺序

避免锁嵌套；若必须嵌套，定义并文档化全局锁顺序以防止死锁：

```c
// 全局锁顺序声明
// Level 1: cache_lock
// Level 2: registry_lock
// Level 3: service_lock

agentos_mutex_lock(&cache_lock);
agentos_mutex_lock(&registry_lock);
// ... 操作 ...
agentos_mutex_unlock(&registry_lock);
agentos_mutex_unlock(&cache_lock);
```

### 3.4 TOCTOU 竞态条件防护

检查与使用之间的时间差（Time-of-Check to Time-of-Use）可能导致竞态条件：

```c
// ❌ 错误：检查与使用分离，存在 TOCTOU 竞态
if (access(filepath, R_OK) == 0) {
    // 攻击者可能在此窗口替换 filepath 为符号链接
    FILE* f = fopen(filepath, "r");  // 不安全
}

// ✅ 正确：直接尝试操作，依赖内核原子性
int fd = open(filepath, O_RDONLY | O_NOFOLLOW);
if (fd < 0) {
    return AGENTOS_EINVAL;
}
// 使用 fd 而非 filepath 进行后续操作
```

**规则**：
- 禁止先检查后使用的模式（`access()` + `open()`、`stat()` + `open()`）
- 使用 `O_NOFOLLOW` 防止符号链接攻击
- 使用文件描述符（fd）而非文件路径进行后续操作
- 对共享资源使用 `agentos_mutex_t` 或原子操作保护

### 3.5 信号处理器安全

信号处理器中只能调用异步信号安全（async-signal-safe）函数：

```c
// ❌ 错误：信号处理器中调用非 async-signal-safe 函数
void signal_handler(int sig) {
    printf("Received signal %d\n", sig);  // ❌ printf 非异步信号安全
    syslog(LOG_ERR, "Signal %d", sig);    // ❌ syslog 非异步信号安全
    free(ptr);                            // ❌ free 非异步信号安全
}

// ✅ 正确：仅使用 async-signal-safe 函数
volatile sig_atomic_t g_signal_received = 0;

void signal_handler(int sig) {
    // 仅设置 volatile sig_atomic_t 标志
    g_signal_received = sig;
    // 写入 pipe 通知主循环（write 是 async-signal-safe）
    char c = (char)sig;
    write(g_signal_pipe[1], &c, 1);  // ✅ write 是 async-signal-safe
}
```

**异步信号安全函数白名单**（POSIX 定义）：`write`、`read`、`_exit`、`signal`、`sigprocmask`、`kill`、`getpid`、`alarm` 等。完整列表参见 POSIX.1-2008 §2.4.3。

**禁止在信号处理器中调用**：`printf`、`malloc`、`free`、`syslog`、`pthread_mutex_lock`、任何非可重入函数。

---

## 4. 输入验证

### 4.1 边界值检查

所有外部输入必须在处理前进行边界检查：

```c
int process_input(const char* input, size_t len) {
    if (!input) return AGENTOS_EINVAL;
    if (len > MAX_INPUT_SIZE) return AGENTOS_EINVAL;  // ✅ 长度检查
    if (len == 0) return AGENTOS_EINVAL;              // ✅ 空输入检查
    
    // 内容过滤：禁止控制字符
    for (size_t i = 0; i < len; i++) {
        if (input[i] < 0x20 && input[i] != '\n' && input[i] != '\t') {
            return AGENTOS_EINVAL;
        }
    }
    // ... 处理逻辑
}
```

### 4.2 整数溢出防护

```c
// ❌ 错误：可能整数溢出
size_t total = count * size;

// ✅ 正确：溢出检查
if (count > 0 && size > SIZE_MAX / count) {
    return AGENTOS_ENOMEM;  // overflow
}
size_t total = count * size;
```

### 4.3 路径遍历防护

```c
// ❌ 错误：直接使用用户输入的路径
FILE* f = fopen(user_path, "r");  // 可能访问 ../ 上级目录

// ✅ 正确：路径规范化检查
if (strstr(user_path, "..") != NULL) {
    return AGENTOS_EINVAL;  // 路径遍历攻击
}
if (user_path[0] == '/') {
    return AGENTOS_EINVAL;  // 绝对路径
}
char safe_path[256];
snprintf(safe_path, sizeof(safe_path), "./data/%s", user_path);
```

### 4.4 有符号/无符号整数混用防护

有符号与无符号整数之间的隐式转换可能导致安全漏洞：

```c
// ❌ 错误：有符号/无符号比较，条件永远为真
int length = get_length();         // 可能返回负值
size_t buf_size = 1024;
if (length > (int)buf_size) { ... }  // 当 length < 0 时，隐式转换为极大无符号值

// ❌ 错误：无符号回绕
unsigned int remaining = total - consumed;  // 当 consumed > total 时回绕

// ✅ 正确：统一使用有符号类型进行范围检查
int length = get_length();
if (length < 0 || length > MAX_LENGTH) {
    return AGENTOS_EINVAL;
}

// ✅ 正确：使用显式范围检查防止回绕
if (consumed > total) {
    return AGENTOS_EINVAL;  // 防止无符号回绕
}
size_t remaining = total - consumed;
```

**规则**：
- 禁止有符号/无符号整数之间的隐式转换，必须显式转换并添加范围检查
- 比较操作中两操作数类型必须一致（同为有符号或同为无符号）
- 使用 `-Wsign-compare -Wsign-conversion` 编译选项检测隐式转换
- 循环变量与数组索引统一使用 `size_t`

---

## 5. 字符串安全

### 5.1 字符串截断安全

```c
// ✅ 正确：确保 null-terminated
char buf[32];
snprintf(buf, sizeof(buf), "%s", long_string);
// buf[sizeof(buf)-1] 必定为 '\0'

// ❌ 错误：可能缺少 null-terminator
strncpy(buf, long_string, sizeof(buf));
// 当 long_string 长度 >= sizeof(buf) 时，buf 不以 '\0' 结尾
```

### 5.2 格式化字符串安全

```c
// ❌ 错误：格式化字符串注入
printf(user_input);           // 用户可注入 %s/%n

// ✅ 正确：固定格式化字符串
printf("%s", user_input);     // 安全

// ❌ 错误
syslog(LOG_ERR, user_input);  // 格式化字符串漏洞

// ✅ 正确
syslog(LOG_ERR, "%s", user_input);
```

**格式化字符串完整防护规则**：

| 场景 | 禁止 | 正确做法 |
|------|------|----------|
| `printf` 族 | `printf(user_input)` | `printf("%s", user_input)` |
| `syslog` | `syslog(pri, user_msg)` | `syslog(pri, "%s", user_msg)` |
| `fprintf` | `fprintf(fp, user_fmt)` | `fprintf(fp, "%s", user_msg)` |
| `snprintf` | `snprintf(buf, n, user_fmt)` | `snprintf(buf, n, "%s", user_msg)` |
| `errx/err` | `errx(1, user_msg)` | `errx(1, "%s", user_msg)` |
| 自定义日志 | `log_fn(user_msg)` | `log_fn("%s", user_msg)` |

**禁止使用 `%n`**：`%n` 可写入任意内存地址，所有格式化字符串中禁止使用 `%n`。编译时应添加 `-Wformat=2 -Wno-format-extra-args` 开启格式化字符串检查。

---

## 6. 安全配置

### 6.1 最小权限原则

```c
// ✅ 正确：降权运行
if (getuid() == 0) {
    // 绑定特权端口后立即降权
    bind_privileged_port();
    setuid(non_privileged_user);
}

// 文件权限最小化
open(path, O_RDONLY, 0);           // 只读
open(path, O_RDWR, S_IRUSR);       // 读写但仅所有者可读
```

### 6.2 敏感数据处理

```c
// ✅ 正确：使用后立即清除敏感数据
char password[64];
get_password(password, sizeof(password));
// ... 使用密码 ...
AGENTOS_SECURE_ZERO(password, sizeof(password));  // 推荐：跨平台安全清零宏
// 或使用 explicit_bzero（注意：Windows/MSVC 不可用）：
// explicit_bzero(password, sizeof(password));
// 或使用 volatile 指针（兼容性回退）：
// volatile char* p = password;
// memset((void*)p, 0, sizeof(password));
```

> **⚠️ explicit_bzero 可移植性说明**：`explicit_bzero` 在 Linux/glibc 上可用，但在 Windows/MSVC 上不可用。Airymax 提供 `AGENTOS_SECURE_ZERO` 宏（定义在 `memory_compat.h`）作为跨平台替代方案，在 Linux 上使用 `explicit_bzero`，在 Windows 上使用 `SecureZeroMemory` 或 volatile 指针方案。所有需要安全清零的代码必须使用 `AGENTOS_SECURE_ZERO` 而非直接调用 `explicit_bzero`。

**禁止**：
- 在日志中输出密钥或密码
- 将敏感数据存储在全局变量中
- 敏感数据释放前不清零

### 6.3 动态库加载安全

动态库搜索路径可能被攻击者利用，实现代码注入：

```c
// ❌ 错误：未清理环境变量，攻击者可通过 LD_LIBRARY_PATH 注入恶意库
void* handle = dlopen("libplugin.so", RTLD_NOW);  // 受 LD_LIBRARY_PATH 影响

// ✅ 正确：使用绝对路径加载，并清理环境变量
// 1. 启动时清理危险环境变量
unsetenv("LD_LIBRARY_PATH");
unsetenv("LD_PRELOAD");
unsetenv("DYLD_INSERT_LIBRARIES");

// 2. 使用绝对路径加载动态库
void* handle = dlopen("/usr/lib/agentos/libplugin.so", RTLD_NOW);
if (!handle) {
    log_error("dlopen failed: %s", dlerror());
    return AGENTOS_ELOAD;
}
```

**规则**：
- 禁止依赖 `LD_LIBRARY_PATH`、`LD_PRELOAD`、`DYLD_INSERT_LIBRARIES` 等环境变量定位库
- `dlopen()` 必须使用绝对路径
- 程序启动时必须清理上述危险环境变量
- 使用 `RPATH`/`RUNPATH`（`-Wl,-rpath,/usr/lib/agentos`）替代 `LD_LIBRARY_PATH`
- 禁止加载来自可写目录（如 `/tmp`、用户 home 目录）的动态库

---

## 7. 日志安全

```c
// ✅ 正确：日志脱敏
log_info("User %s login from %s", sanitize_username(user), sanitize_ip(ip));
// log_error 应包含错误码，方便追溯

// ❌ 错误：日志泄露敏感信息
log_info("Password: %s", password);
log_info("Session token: %s", token);
```

---

## 8. 编译安全选项

Release 构建必须启用以下安全选项：

| 选项 | 作用 | 平台 |
|------|------|------|
| `-fstack-protector-strong` | 栈溢出检测 | GCC/Clang |
| `-D_FORTIFY_SOURCE=2` | 运行时缓冲区检查 | GCC/Clang |
| `-fPIE -pie` | 位置无关可执行 | GCC/Clang |
| `-Wl,-z,relro -Wl,-z,now` | 只读重定位 | GCC/Clang |
| `/GS-` 或 `/guard:cf` | 控制流防护 | MSVC |

---

## 附录：跨文档规范引用

本规范与以下 Airymax 工程规范一致，所有 C/C++ 安全代码须同时遵循：

| 规范集 | 说明 | 来源文档 |
|--------|------|---------|
| **BAN-01~13** | 13 项禁止模式（桩函数/假数据/空返回等） | [C编码规范 §18](./C_coding_style_standard.md) |
| **CROSS-01~06** | 6 项跨平台编译规则 | [C编码规范 §17](./C_coding_style_standard.md) |
| **REQ-01~08** | 8 项强制规范 | [C编码规范 §1.2](./C_coding_style_standard.md) |
| **标准术语** | 8 个架构组件标准名称 | [10-terminology.md](../../50-specifications/10-terminology.md) |

### BAN 规则安全交叉引用

以下 BAN 规则与本安全编程指南直接相关，违反即构成安全漏洞：

| BAN 规则 | 安全影响 | 本文档对应章节 |
|----------|----------|---------------|
| **BAN-09** 禁止 `free("literal")` | 释放字符串字面量导致未定义行为，可被利用执行任意代码 | §1.3 释放后使用防护 |
| **BAN-10** 禁止假原子操作 | 自定义原子操作缺乏内存屏障，导致竞态条件和数据损坏 | §3.2 原子操作 |
| **BAN-04** 禁止无 Issue 引用的 TODO/FIXME | 安全修复缺乏跟踪，漏洞可能被遗忘 | [注释规范 §7.1-7.2](./Code_comment_template.md) |
| **BAN-01** 禁止桩函数 | 安全检查桩函数返回"成功"使防护形同虚设 | §4.1 边界值检查 |

### CI 安全规则执行

所有安全规则必须在 CI 流水线中强制执行，禁止绕过：

| 检查类型 | 工具 | 触发时机 | 阻断规则 |
|----------|------|----------|----------|
| 静态安全分析 | CodeQL（必选） | 每次 PR 提交 | Critical/High 阻断合并 |
| 静态安全分析 | Coverity（推荐） | 每日全量扫描 | Critical 阻断发布 |
| 内存安全 | AddressSanitizer (ASan) | CI 测试阶段 | 任何发现阻断合并 |
| 未定义行为 | UndefinedBehaviorSanitizer (UBSan) | CI 测试阶段 | 任何发现阻断合并 |
| 编译警告 | `-Wall -Wextra -Werror` | 每次构建 | 警告即错误 |
| 格式化字符串 | `-Wformat=2` | 每次构建 | 任何发现阻断合并 |
| 整数溢出 | `-ftrapv`（调试构建） | CI 测试阶段 | 运行时陷阱 |
| 栈保护 | `-fstack-protector-strong` | Release 构建 | 必须启用 |

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.  
"From data intelligence emerges."
