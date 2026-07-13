Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）编码规范详细设计

> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）字符编码规范与 UTF-8 处理详细设计\
> **版本**：0.1.1\
> **最后更新**：2026-07-09\
> **理论根基**：Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论\
> **SPDX-License-Identifier**：AGPL-3.0-or-later OR Apache-2.0

---

## 1. 设计目标与范围

### 1.1 设计目标

agentrt-linux（AirymaxOS）编码规范旨在为内核态与用户态代码统一字符编码处理标准。本设计聚焦三大目标：

1. **内核态 UTF-8 一等公民支持**：内核态禁用 `wchar_t`，所有字符串以 UTF-8 字节序列表达，提供 UTF-8 校验、长度、子串、迭代原语
2. **用户态 Unicode 规范一致性**：12 daemon 与 9 协议适配层统一使用 UTF-8 编码，禁止 GB2312/GBK/Shift-JIS 等非 Unicode 编码流入系统
3. **Agent 多语言 prompt 编码契约**：Agent 提示词与 LLM 推理输入/输出严格 UTF-8，Token 计数按 Unicode 码点（而非字节）计算

### 1.2 适用范围

- Linux 6.6 内核基线内核态 UTF-8 字符串处理（`lib/charset.c`、`lib/string.c` 扩展）
- 用户态 glibc / musl 的 UTF-8 处理（iconv / mbsrtowcs 等）
- AgentsIPC 消息头的字符串字段编码（128B 消息头内 TraceID 字符串）
- Agent 系统提示词、用户输入、LLM 输出的统一 UTF-8 处理
- 文档源码注释编码规范（所有 .md / .c / .h / .py 文件 UTF-8）

### 1.3 术语规范

本设计严格遵守 agentrt-linux 术语规范：agentrt（用户态）称为**微核心**（micro-core），agentrt-linux（OS 发行版）称为**微内核**（micro-kernel）。编码契约在 agentrt 与 agentrt-linux 之间属于 IRON-9 v2 [SC] 共享契约层。

### 1.4 编码规范矩阵

| 范围 | 编码 | 字节序 | BOM | 备注 |
|------|------|--------|-----|------|
| 内核源码 | UTF-8 | N/A | 禁止 | .c/.h 文件 |
| 用户态源码 | UTF-8 | N/A | 禁止 | .py/.rs/.go 文件 |
| 文档 | UTF-8 | N/A | 禁止 | .md 文件 |
| IPC 消息 | UTF-8 | LE/BE 无关 | 禁止 | 字节流 |
| JSON-RPC | UTF-8 | N/A | 禁止 | RFC 8259 |
| 日志 | UTF-8 | N/A | 禁止 | syslog / journald |
| 配置文件 | UTF-8 | N/A | 禁止 | .yaml / .conf / .ini |
| 内核日志 | UTF-8 | N/A | 禁止 | printk 输出 |

---

## 2. 内核态 UTF-8 处理规范

### 2.1 禁止 wchar_t 规则

agentrt-linux 内核态严格禁止使用 `wchar_t` 类型，原因有三：

1. **Linux 6.6 内核不提供 wchar_t 运行时支持**：内核态没有 `wprintf`、`wcslen`、`wcscpy` 等 C 标准库宽字符函数
2. **wchar_t 大小不可移植**：Linux 上 4 字节，Windows 上 2 字节，会导致跨平台数据不兼容
3. **UTF-8 字节序列更紧凑**：CJK 字符在 UTF-8 中占 3 字节，比 wchar_t 的 4 字节省 25% 内存

替代方案：所有内核字符串以 `char *` 表达 UTF-8 字节序列，长度以字节数（`size_t`）记录。

### 2.2 UTF-8 编码规则速查

UTF-8 编码规则（RFC 3629）：

| Unicode 码点范围 | UTF-8 字节序列 | 字节数 |
|------------------|---------------|--------|
| U+0000 ~ U+007F | 0xxxxxxx | 1 |
| U+0080 ~ U+07FF | 110xxxxx 10xxxxxx | 2 |
| U+0800 ~ U+FFFF | 1110xxxx 10xxxxxx 10xxxxxx | 3 |
| U+10000 ~ U+10FFFF | 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx | 4 |

特殊范围禁止：
- U+D800 ~ U+DFFF：UTF-16 代理对，UTF-8 中非法
- U+FFFE / U+FFFF：非字符，禁止
- U+FDD0 ~ U+FDEF：非字符，禁止

### 2.3 内核 UTF-8 原语库

```c
/* include/linux/airy_utf8.h —— 内核态 UTF-8 原语 */
#ifndef AIRY_UTF8_H
#define AIRY_UTF8_H

#include <linux/types.h>
#include <linux/string.h>

/* UTF-8 字符最大字节数 */
#define AIRY_UTF8_MAX_BYTES  4

/* UTF-8 码点类型 */
typedef u32 airy_unicode_t;

/* UTF-8 字符迭代器 */
struct airy_utf8_iter {
	const u8 *p;        /* 当前字节指针 */
	const u8 *end;      /* 末尾指针 */
};

/* 初始化迭代器 */
static inline void airy_utf8_iter_init(struct airy_utf8_iter *it,
					 const char *s, size_t len)
{
	it->p = (const u8 *)s;
	it->end = (const u8 *)s + len;
}

/* 校验单个 UTF-8 字符的首字节，返回该字符的字节数（1-4），0 表示非法 */
static inline int airy_utf8_seq_len(u8 c)
{
	if (c < 0x80)
		return 1;                          /* 0xxxxxxx */
	if ((c & 0xE0) == 0xC0)
		return 2;                          /* 110xxxxx */
	if ((c & 0xF0) == 0xE0)
		return 3;                          /* 1110xxxx */
	if ((c & 0xF8) == 0xF0)
		return 4;                          /* 11110xxx */
	return 0;                              /* 非法首字节 */
}

/* 解码一个 UTF-8 字符，返回字节数（>0）或错误码（<0） */
static inline int airy_utf8_decode(const u8 *s, size_t len,
				     airy_unicode_t *cp)
{
	int n;
	airy_unicode_t u;

	if (!s || len == 0)
		return -EINVAL;

	n = airy_utf8_seq_len(s[0]);
	if (n == 0 || (size_t)n > len)
		return -EILSEQ;

	switch (n) {
	case 1:
		u = s[0];
		break;
	case 2:
		if ((s[1] & 0xC0) != 0x80)
			return -EILSEQ;
		u = ((airy_unicode_t)(s[0] & 0x1F) << 6) |
		    (s[1] & 0x3F);
		if (u < 0x80)
			return -EILSEQ;        /* 过长编码 */
		break;
	case 3:
		if ((s[1] & 0xC0) != 0x80 || (s[2] & 0xC0) != 0x80)
			return -EILSEQ;
		u = ((airy_unicode_t)(s[0] & 0x0F) << 12) |
		    ((airy_unicode_t)(s[1] & 0x3F) << 6) |
		    (s[2] & 0x3F);
		if (u < 0x800)
			return -EILSEQ;
		if (u >= 0xD800 && u <= 0xDFFF)
			return -EILSEQ;        /* 代理对非法 */
		break;
	case 4:
		if ((s[1] & 0xC0) != 0x80 || (s[2] & 0xC0) != 0x80 ||
		    (s[3] & 0xC0) != 0x80)
			return -EILSEQ;
		u = ((airy_unicode_t)(s[0] & 0x07) << 18) |
		    ((airy_unicode_t)(s[1] & 0x3F) << 12) |
		    ((airy_unicode_t)(s[2] & 0x3F) << 6) |
		    (s[3] & 0x3F);
		if (u < 0x10000 || u > 0x10FFFF)
			return -EILSEQ;
		break;
	default:
		return -EILSEQ;
	}

	*cp = u;
	return n;
}

/* 编码一个 Unicode 码点到 UTF-8，返回字节数 */
static inline int airy_utf8_encode(u8 *out, size_t out_len,
				     airy_unicode_t cp)
{
	if (!out || out_len == 0)
		return -EINVAL;

	if (cp <= 0x7F) {
		if (out_len < 1)
			return -E2BIG;
		out[0] = (u8)cp;
		return 1;
	} else if (cp <= 0x7FF) {
		if (out_len < 2)
			return -E2BIG;
		out[0] = 0xC0 | (u8)(cp >> 6);
		out[1] = 0x80 | (u8)(cp & 0x3F);
		return 2;
	} else if (cp <= 0xFFFF) {
		if (cp >= 0xD800 && cp <= 0xDFFF)
			return -EILSEQ;
		if (out_len < 3)
			return -E2BIG;
		out[0] = 0xE0 | (u8)(cp >> 12);
		out[1] = 0x80 | (u8)((cp >> 6) & 0x3F);
		out[2] = 0x80 | (u8)(cp & 0x3F);
		return 3;
	} else if (cp <= 0x10FFFF) {
		if (out_len < 4)
			return -E2BIG;
		out[0] = 0xF0 | (u8)(cp >> 18);
		out[1] = 0x80 | (u8)((cp >> 12) & 0x3F);
		out[2] = 0x80 | (u8)((cp >> 6) & 0x3F);
		out[3] = 0x80 | (u8)(cp & 0x3F);
		return 4;
	}
	return -EILSEQ;
}

/* 计算 UTF-8 字符串的码点数（非字节数） */
static inline size_t airy_utf8_strlen(const char *s, size_t byte_len)
{
	size_t count = 0;
	size_t i = 0;

	while (i < byte_len) {
		int n = airy_utf8_seq_len((u8)s[i]);
		if (n == 0 || i + (size_t)n > byte_len)
			return (size_t)-1;
		i += n;
		count++;
	}
	return count;
}

/* 校验 UTF-8 字符串合法性，返回 0 合法或 -EILSEQ 非法 */
static inline int airy_utf8_validate(const char *s, size_t len)
{
	size_t i = 0;
	airy_unicode_t cp;

	while (i < len) {
		int n = airy_utf8_decode((const u8 *)s + i, len - i, &cp);
		if (n < 0)
			return n;
		i += n;
	}
	return 0;
}

/* 取下一个 UTF-8 字符，更新迭代器，返回字节数 */
static inline int airy_utf8_next(struct airy_utf8_iter *it,
				   airy_unicode_t *cp)
{
	int n;

	if (it->p >= it->end)
		return 0;     /* EOF */

	n = airy_utf8_decode(it->p, it->end - it->p, cp);
	if (n < 0)
		return n;
	it->p += n;
	return n;
}

#endif /* AIRY_UTF8_H */
```

### 2.4 安全字符串操作规范

agentrt-linux 内核态严格禁止 `strcpy`、`strcat`、`sprintf`、`gets` 等不安全函数，统一使用 `strscpy`、`strscpy_pad`、`scnprintf`：

```c
/* 安全字符串操作示例（K&R 风格，Tab=8） */
#include <linux/string.h>
#include <linux/slab.h>
#include <airymax/airy_utf8.h>

/* 错误示例：禁止使用 strcpy（缓冲区溢出风险） */
/* int bad_example(char *dst) { strcpy(dst, "agentrt-linux"); } */

/* 正确示例：使用 strscpy（自动截断并保证 NUL 终止） */
int good_example(char *dst, size_t dst_size)
{
	int ret;

	ret = strscpy(dst, "agentrt-linux", dst_size);
	if (ret < 0)
		return -E2BIG;
	return 0;
}

/* UTF-8 安全的子串截取（按字节边界但保证 UTF-8 完整性） */
int airy_utf8_substring(const char *src, size_t src_len,
			   size_t start_byte, size_t end_byte,
			   char *out, size_t out_size)
{
	size_t i, valid_end;
	int ret;

	if (start_byte >= src_len || end_byte > src_len ||
	    start_byte >= end_byte) {
		ret = -EINVAL;
		goto out_err;
	}

	/* 调整 end_byte 向前回退到 UTF-8 字符边界 */
	valid_end = end_byte;
	for (i = end_byte; i > start_byte; i--) {
		int n = airy_utf8_seq_len((u8)src[i - 1]);
		if (n == 1 || n == 0) {
			valid_end = i;
			break;
		}
	}
	if (i == start_byte)
		valid_end = end_byte;

	if (out_size < (valid_end - start_byte) + 1) {
		ret = -E2BIG;
		goto out_err;
	}

	memcpy(out, src + start_byte, valid_end - start_byte);
	out[valid_end - start_byte] = '\0';
	return 0;

out_err:
	return ret;
}
```

---

## 3. 用户态 Unicode 规范

### 3.1 用户态字符串处理

用户态 daemon 使用 glibc 的 mbsrtowcs / iconv 进行编码转换，所有外源输入必须经过编码校验：

```c
/* services/common/airy_unicode.c */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <wchar.h>
#include <locale.h>
#include <iconv.h>
#include <airymax/airy_utf8.h>

/* 校验输入字符串为合法 UTF-8（用户态版本） */
int airy_user_utf8_validate(const char *s, size_t len)
{
	mbstate_t state;
	wchar_t wc;
	size_t consumed = 0;
	size_t n;

	memset(&state, 0, sizeof(state));
	while (consumed < len) {
		n = mbrtowc(&wc, s + consumed, len - consumed, &state);
		if (n == (size_t)-1)
			return -EILSEQ;       /* 非法字节序列 */
		if (n == (size_t)-2)
			return -EILSEQ;       /* 不完整序列（len 已耗尽） */
		if (n == 0)
			break;
		consumed += n;
	}
	return 0;
}

/* 从其他编码转换为 UTF-8（如 GB18030 → UTF-8） */
int airy_user_convert_to_utf8(const char *from_codec,
				const char *in, size_t in_len,
				char *out, size_t out_size)
{
	iconv_t cd;
	char *inbuf = (char *)in;
	char *outbuf = out;
	size_t inbytesleft = in_len;
	size_t outbytesleft = out_size;
	int ret;

	cd = iconv_open("UTF-8", from_codec);
	if (cd == (iconv_t)-1)
		return -EINVAL;

	ret = iconv(cd, &inbuf, &inbytesleft, &outbuf, &outbytesleft);
	if (ret == (size_t)-1) {
		int err = errno;
		iconv_close(cd);
		switch (err) {
		case EILSEQ:
			return -EILSEQ;
		case EINVAL:
			return -EINVAL;
		case E2BIG:
			return -E2BIG;
		default:
			return -EINVAL;
		}
	}

	/* NUL 终止 */
	if (outbytesleft == 0) {
		iconv_close(cd);
		return -E2BIG;
	}
	*outbuf = '\0';
	iconv_close(cd);
	return out_size - outbytesleft;
}
```

### 3.2 JSON-RPC 编码契约

所有 JSON-RPC 请求/响应必须使用 UTF-8 编码（RFC 8259 强制要求）：

```c
/* JSON-RPC 响应构造（确保 UTF-8） */
#include <jansson.h>
#include <airymax/airy_utf8.h>

int airy_jsonrpc_build_response(int code, const char *message,
				   const char *locale, char *out, size_t out_size)
{
	json_t *root, *error;
	char *json_str;
	int ret;

	/* 校验 message 为合法 UTF-8 */
	if (airy_user_utf8_validate(message, strlen(message)) < 0)
		return -EILSEQ;

	root = json_object();
	error = json_object();
	json_object_set_new(error, "code", json_integer(code));
	json_object_set_new(error, "message", json_string(message));
	json_object_set_new(error, "locale", json_string(locale));
	json_object_set_new(root, "error", error);

	json_str = json_dumps(root, JSON_ENSURE_ASCII | JSON_COMPACT);
	if (!json_str) {
		ret = -ENOMEM;
		goto out_free_root;
	}

	ret = strscpy(out, json_str, out_size);
	if (ret < 0) {
		ret = -E2BIG;
		goto out_free_json;
	}

out_free_json:
	free(json_str);
out_free_root:
	json_decref(root);
	return ret;
}
```

---

## 4. Agent 多语言 prompt 编码

### 4.1 prompt 编码契约

Agent 系统提示词（system prompt）与用户输入必须为 UTF-8 编码。Token 计数按 Unicode 码点（而非字节）计算，确保 CJK 用户与英文用户公平。

```python
# agentrt SDK 示例：多语言 prompt UTF-8 处理
from agentrt import CognitionClient
from agentrt.encoding import validate_utf8, count_codepoints

client = CognitionClient(language="zh-CN")

# 中文 prompt（UTF-8 编码，自动校验）
prompt = "你是一个智能助手，请用中文回答用户问题。"
if not validate_utf8(prompt.encode('utf-8')):
    raise ValueError("prompt 不是合法 UTF-8")

# Token 计数按码点（3 字节 CJK 字符算 1 码点）
token_count = count_codepoints(prompt)
print(f"prompt 码点数: {token_count}")

# 提交推理
result = client.complete(
    system_prompt=prompt,
    user_input="解释什么是微内核"
)
```

### 4.2 Token 计数规范

agentrt-linux 的 Token 计数遵循"按 Unicode 码点计数"原则：

| 字符类型 | 字节数 | 码点数 | Token 计数 |
|----------|--------|--------|------------|
| ASCII（A-Z, 0-9） | 1 | 1 | 1 |
| 拉丁扩展（é, ñ） | 2 | 1 | 1 |
| CJK（中、日、韩） | 3 | 1 | 1 |
| Emoji（😀） | 4 | 1 | 1 |

```c
/* Token 计数按码点（C SDK 示例） */
#include <airymax/airy_utf8.h>

size_t airy_token_count(const char *prompt, size_t byte_len)
{
	struct airy_utf8_iter it;
	airy_unicode_t cp;
	size_t count = 0;
	int n;

	airy_utf8_iter_init(&it, prompt, byte_len);
	while ((n = airy_utf8_next(&it, &cp)) > 0)
		count++;
	return count;
}
```

---

## 5. 内核态 UTF-8 字符串处理完整示例

### 5.1 IPC 消息头 TraceID 字符串处理

```c
/* IPC 消息头中 TraceID 字段的 UTF-8 安全处理 */
#include <linux/string.h>
#include <linux/slab.h>
#include <airymax/airy_utf8.h>
#include <airymax/ipc.h>

/* 设置 IPC 消息头的 trace_id（UTF-8 安全复制） */
int airy_ipc_set_trace_id(struct airy_ipc_msg_hdr *hdr,
			    const char *trace_id)
{
	size_t trace_len;
	int ret;

	if (!hdr || !trace_id)
		return -EINVAL;

	trace_len = strlen(trace_id);
	if (trace_len > sizeof(hdr->trace_id))
		return -E2BIG;

	/* 校验 UTF-8 合法性 */
	ret = airy_utf8_validate(trace_id, trace_len);
	if (ret < 0)
		return ret;

	memcpy(hdr->trace_id, trace_id, trace_len);
	if (trace_len < sizeof(hdr->trace_id))
		memset(hdr->trace_id + trace_len, 0,
		       sizeof(hdr->trace_id) - trace_len);
	return 0;
}

/* 从 IPC 消息头提取 trace_id 为 C 字符串 */
int airy_ipc_get_trace_id(const struct airy_ipc_msg_hdr *hdr,
			    char *out, size_t out_size)
{
	size_t i;
	int ret;

	if (!hdr || !out || out_size == 0)
		return -EINVAL;

	/* 找到第一个 NUL */
	for (i = 0; i < sizeof(hdr->trace_id); i++)
		if (hdr->trace_id[i] == 0)
			break;

	if (i == 0) {
		out[0] = '\0';
		return 0;
	}

	/* 校验 UTF-8 */
	ret = airy_utf8_validate((const char *)hdr->trace_id, i);
	if (ret < 0)
		return ret;

	if (i + 1 > out_size)
		return -E2BIG;

	memcpy(out, hdr->trace_id, i);
	out[i] = '\0';
	return 0;
}
```

### 5.2 printk 多语言消息的 UTF-8 安全长度限制

Linux 6.6 内核 printk 单行最大 1024 字节（`PRINTK_SAFE_LOG_BUF_LEN`），多语言消息需保证 UTF-8 完整性：

```c
/* 多语言 printk 长度限制（保证 UTF-8 字符完整性） */
#define AIRY_PRINTK_MAX_BYTES  1024

int airy_printk_utf8_safe(const char *level, const char *msg, size_t msg_len)
{
	size_t valid_len;
	int ret;

	if (!msg || msg_len == 0)
		return -EINVAL;

	/* 校验 UTF-8 */
	ret = airy_utf8_validate(msg, msg_len);
	if (ret < 0)
		return ret;

	/* 截断到 PRINTK_MAX_BYTES，回退到字符边界 */
	valid_len = msg_len;
	if (valid_len > AIRY_PRINTK_MAX_BYTES - 1) {
		size_t i;
		valid_len = AIRY_PRINTK_MAX_BYTES - 1;
		for (i = valid_len; i > 0; i--) {
			int n = airy_utf8_seq_len((u8)msg[i - 1]);
			if (n == 1 || n == 0) {
				valid_len = i;
				break;
			}
		}
	}

	printk("%s%.*s\n", level, (int)valid_len, msg);
	return 0;
}
```

---

## 6. 性能考量

### 6.1 UTF-8 处理开销

| 操作 | 开销（ns） | 备注 |
|------|-----------|------|
| UTF-8 校验（1KB 字符串） | 200-500 | 线性扫描 |
| UTF-8 长度计算（1KB） | 100-300 | 线性扫描 |
| UTF-8 解码（单字符） | 5-15 | 分支判断 |
| UTF-8 编码（单字符） | 5-20 | 位运算 |
| iconv 转换（GB18030→UTF-8，1KB） | 5,000-10,000 | 库调用 |

### 6.2 优化策略

1. **缓存校验结果**：daemon 启动时一次性校验配置文件，运行时不再校验
2. **SIMD 加速**：用户态可使用 SIMD 指令加速 UTF-8 校验（如 `simdutf` 库）
3. **批量处理**：IPC 批量消息的 UTF-8 校验合并为一次扫描

---

## 7. 错误码体系对接

编码错误码纳入 agentrt-linux 统一错误码体系：

| 错误码 | 数值 | 含义 |
|--------|------|------|
| AIRY_E_ENCODING_INVALID_UTF8 | -904 | 无效 UTF-8 序列 |
| AIRY_E_ENCODING_OVERLONG | -906 | 过长编码（如 ASCII 用 2 字节） |
| AIRY_E_ENCODING_SURROGATE | -907 | UTF-16 代理对（UTF-8 中非法） |
| AIRY_E_ENCODING_NONCHAR | -908 | 非字符码点（U+FFFE 等） |
| AIRY_E_ENCODING_OUT_OF_RANGE | -909 | 码点超出 U+10FFFF |
| AIRY_E_ENCODING_BOM | -910 | 出现 BOM（禁止） |

集中错误处理示例：

```c
int airy_validate_input(const char *input, size_t len)
{
	int ret;

	if (!input)
		return -EINVAL;

	/* 检查 BOM */
	if (len >= 3 && (u8)input[0] == 0xEF && (u8)input[1] == 0xBB &&
	    (u8)input[2] == 0xBF)
		return -AIRY_E_ENCODING_BOM;

	/* 校验 UTF-8 */
	ret = airy_utf8_validate(input, len);
	if (ret == -EILSEQ)
		return -AIRY_E_ENCODING_INVALID_UTF8;
	return ret;
}
```

---

## 8. 五维原则映射

| 原则 | 在本设计的体现 |
|------|---------------|
| **K-2 接口契约化** | UTF-8 编码作为 [SC] 共享契约 |
| **E-1 安全内生** | 禁止 strcpy/wchar_t，从源头消除安全风险 |
| **E-4 跨平台一致性** | UTF-8 跨架构一致，无字节序问题 |
| **E-5 命名语义化** | airy_utf8_* 函数名即语义 |
| **E-6 错误可追溯** | 详细错误码（-904 ~ -910）覆盖所有编码错误 |
| **A-3 人文关怀** | CJK 用户与英文用户公平的 Token 计数 |

---

## 9. IRON-9 v2 同源映射

| 组件 | agentrt-linux（[SC]） | agentrt（[SC]） | 共享 |
|------|------------------------|------------------|------|
| UTF-8 头文件 | `airy_utf8.h` | `airy_utf8.h` | 完全共享 |
| 错误码 | AIRY_E_ENCODING_* | AIRY_E_ENCODING_* | `error.h` |
| Token 计数规则 | 按码点 | 按码点 | 共享契约 |
| 编码规范文档 | 本文件 | 同源镜像 | [SC] |

---

## 10. 相关文档

- `180-i18n/README.md`（国际化主索引）
- `180-i18n/01-locale-design.md`（Locale 设计）
- `180-i18n/04-cjk-support.md`（CJK 支持设计）
- `50-engineering-standards/01-coding-standards.md`（编码规范）
- `170-performance/03-ipc-performance.md`（IPC 消息头）

---

## 11. 参考材料

- RFC 3629（UTF-8 编码规范）
- RFC 8259（JSON 规范，强制 UTF-8）
- Linux 6.6 `lib/charset.c`
- Linux 6.6 `include/linux/string.h`
- Unicode 标准（Unicode 15.0）
- POSIX locale 与 iconv 规范

---

> **文档结束** | agentrt-linux（AirymaxOS）编码规范详细设计 
