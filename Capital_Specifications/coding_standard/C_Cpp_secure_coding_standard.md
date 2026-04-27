Copyright (c) 2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# AgentOS C&C++安全编程指南

**版本**: Doc V2.0  
**最后更新**: 2026-04-27  
**作者**: LirenWang  
**适用范围**: AgentOS 所有 C/C++代码开发活动  
**理论基础**: 工程控制论（信任边界、防御深度）、系统工程（层次分解）、五维正交系统（系统观、内核观、认知观、工程观、设计美学）、双系统认知理论  
**关联规范**: [C编码规范](./C_coding_style_standard.md)的 BAN-01~13 禁止模式、CROSS-01~06 跨平台规则；[TERMINOLOGY.md](../../Capital_Specifications/TERMINOLOGY.md) 标准术语  
**原则映射**: D-1（最小权限）、D-2（安全隔离）、D-3（纵深防御）、D-4（安全审计）、C-3（认知偏差防护）

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
explicit_bzero(password, sizeof(password));  // 防止编译器优化掉清除
// 或使用 volatile 指针：
volatile char* p = password;
memset((void*)p, 0, sizeof(password));
```

**禁止**：
- 在日志中输出密钥或密码
- 将敏感数据存储在全局变量中
- 敏感数据释放前不清零

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

本规范与以下 AgentOS 工程规范一致，所有 C/C++ 安全代码须同时遵循：

| 规范集 | 说明 | 来源文档 |
|--------|------|---------|
| **BAN-01~13** | 13 项禁止模式（桩函数/假数据/空返回等） | [C编码规范 §18](./C_coding_style_standard.md) |
| **CROSS-01~06** | 6 项跨平台编译规则 | [C编码规范 §17](./C_coding_style_standard.md) |
| **REQ-01~08** | 8 项强制规范 | [C编码规范 §1.2](./C_coding_style_standard.md) |
| **标准术语** | 8 个架构组件标准名称 | [TERMINOLOGY.md](../../Capital_Specifications/TERMINOLOGY.md) |

---

© 2026 SPHARX Ltd. All Rights Reserved.  
"From data intelligence emerges."
