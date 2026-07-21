Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-linux（AirymaxOS）Locale 设计详细规范
> **文档定位**：agentrt-linux（AirymaxOS，极境智能体操作系统）多语言支持架构与 locale 机制详细设计\
> **文档版本**：0.1.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[agentrt-linux 设计文档](README.md)\
> **理论根基**：Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论\
> **SPDX-License-Identifier**：AGPL-3.0-or-later OR Apache-2.0

---

## 1. 设计目标与范围

### 1.1 设计目标

agentrt-linux（AirymaxOS）Locale 设计旨在为全球开发者与用户提供一致的多语言支持架构。本设计聚焦三大目标：

1. **用户态 locale 全链路覆盖**：12 个 daemon 与 9 种协议适配层在 zh-CN / en-US / ja-JP 三种核心 locale 下行为一致
2. **内核日志 UTF-8 原生支持**：printk 与 trace_printk 输出完整 UTF-8 多语言消息，无需依赖用户态翻译表
3. **多区域记忆卷载 locale 隔离**：MemoryRovol L1-L4 按区域（zh-CN / en-US / ja-JP）独立组织，互不污染

### 1.2 适用范围

- Linux 6.6 内核基线 printk 子系统的 UTF-8 多语言消息表
- 用户态 glibc / musl 的 locale 与 LC_* 机制
- Agent 系统提示词的多语言切换
- 终端、TUI、Web UI 三类前端的 locale 一致性
- 多语言文档同步（README.md + README_zh.md + README_ja.md）

### 1.3 术语规范

本设计严格遵守 agentrt-linux 术语规范：agentrt（用户态）称为**微核心**（micro-core），agentrt-linux（OS 发行版）称为**微内核**（micro-kernel）。所有外部 Linux 发行版统一表述为"主流 Linux 发行版"，禁止使用 主流 Linux 发行版/主流 Linux 发行版 字样。Agent 提示词与 locale 在 agentrt 与 agentrt-linux 之间属于 IRON-9 v3 [SS] 语义同源层。

### 1.4 locale 支持矩阵

| locale | 语言 | 区域 | 字符集 | 优先级 | 备注 |
|--------|------|------|--------|--------|------|
| en_US.UTF-8 | English | United States | UTF-8 | P0 | 默认 locale |
| zh_CN.UTF-8 | 简体中文 | 中国大陆 | UTF-8 | P0 | 国密合规 |
| ja_JP.UTF-8 | 日本語 | 日本 | UTF-8 | P1 | CJK 支持 |
| ko_KR.UTF-8 | 한국어 | 한국 | UTF-8 | P1 | CJK 支持 |
| zh_TW.UTF-8 | 繁体中文 | 台湾 | UTF-8 | P2 | — |
| en_GB.UTF-8 | English | United Kingdom | UTF-8 | P2 | — |
| de_DE.UTF-8 | Deutsch | Deutschland | UTF-8 | P2 | — |
| fr_FR.UTF-8 | Français | France | UTF-8 | P2 | — |

---

## 2. 多语言支持架构

### 2.1 四层 locale 架构

```
┌─────────────────────────────────────────────────────────┐
│ L4 应用层（Agent 提示词 / Web UI / TUI）               │
│   - Agent system_prompt 按 locale 切换                 │
│   - Web UI i18n（vue-i18n / react-intl）                │
├─────────────────────────────────────────────────────────┤
│ L3 服务层（12 daemon + 9 协议适配）                     │
│   - daemon 错误消息按 locale 本地化                     │
│   - JSON-RPC 错误消息字段按 locale 翻译                 │
├─────────────────────────────────────────────────────────┤
│ L2 用户态库层（glibc locale + gettext + .mo）           │
│   - locale 命名 / LC_* 分类 / collate                  │
│   - .po / .mo 翻译文件                                  │
├─────────────────────────────────────────────────────────┤
│ L1 内核层（printk + 内核日志 UTF-8 消息表）             │
│   - KERN_INFO / KERN_WARN / KERN_ERR 多语言             │
│   - trace_printk 多语言                                 │
└─────────────────────────────────────────────────────────┘
```

### 2.2 locale 标识符规范

agentrt-linux 严格遵循 POSIX locale 命名规范 `language[_TERRITORY][.codeset][@modifier]`：

```c
/* include/uapi/linux/airymax/locale.h —— agentrt-linux locale 标识符 */
#ifndef AIRY_LOCALE_H
#define AIRY_LOCALE_H

#include <linux/types.h>

/* agentrt-linux 支持的 locale 枚举 */
enum airy_locale_id {
	AIRY_LOCALE_EN_US = 0,    /* 默认 locale */
	AIRY_LOCALE_ZH_CN,        /* 简体中文 + 国密 */
	AIRY_LOCALE_JA_JP,        /* 日本語 */
	AIRY_LOCALE_KO_KR,        /* 한국어 */
	AIRY_LOCALE_ZH_TW,        /* 繁体中文 */
	AIRY_LOCALE_EN_GB,        /* 英国英语 */
	AIRY_LOCALE_DE_DE,        /* 德语 */
	AIRY_LOCALE_FR_FR,        /* 法语 */
	AIRY_LOCALE_MAX,
};

/* locale 标识符（POSIX 格式，最长 32 字节） */
struct airy_locale_desc {
	enum airy_locale_id id;
	char posix_name[32];          /* 如 "zh_CN.UTF-8" */
	char language[8];             /* 如 "zh" */
	char territory[8];            /* 如 "CN" */
	char codeset[16];             /* 如 "UTF-8" */
};

/* locale 全局表（编译期生成，只读） */
static const struct airy_locale_desc airy_locales[] = {
	[AIRY_LOCALE_EN_US] = { AIRY_LOCALE_EN_US, "en_US.UTF-8", "en", "US", "UTF-8" },
	[AIRY_LOCALE_ZH_CN] = { AIRY_LOCALE_ZH_CN, "zh_CN.UTF-8", "zh", "CN", "UTF-8" },
	[AIRY_LOCALE_JA_JP] = { AIRY_LOCALE_JA_JP, "ja_JP.UTF-8", "ja", "JP", "UTF-8" },
	[AIRY_LOCALE_KO_KR] = { AIRY_LOCALE_KO_KR, "ko_KR.UTF-8", "ko", "KR", "UTF-8" },
	[AIRY_LOCALE_ZH_TW] = { AIRY_LOCALE_ZH_TW, "zh_TW.UTF-8", "zh", "TW", "UTF-8" },
	[AIRY_LOCALE_EN_GB] = { AIRY_LOCALE_EN_GB, "en_GB.UTF-8", "en", "GB", "UTF-8" },
	[AIRY_LOCALE_DE_DE] = { AIRY_LOCALE_DE_DE, "de_DE.UTF-8", "de", "DE", "UTF-8" },
	[AIRY_LOCALE_FR_FR] = { AIRY_LOCALE_FR_FR, "fr_FR.UTF-8", "fr", "FR", "UTF-8" },
};

#endif /* AIRY_LOCALE_H */
```

### 2.3 LC_* 分类与应用

agentrt-linux 完整支持 glibc 的 LC_* 分类，每个 daemon 可独立配置：

```bash
# /etc/airymaxos/locale.conf —— 系统级 locale 配置
LANG=zh_CN.UTF-8
LC_CTYPE=zh_CN.UTF-8
LC_NUMERIC=en_US.UTF-8
LC_TIME=en_US.UTF-8
LC_COLLATE=zh_CN.UTF-8
LC_MONETARY=zh_CN.UTF-8
LC_MESSAGES=zh_CN.UTF-8
LC_PAPER=en_US.UTF-8
LC_NAME=zh_CN.UTF-8
LC_ADDRESS=zh_CN.UTF-8
LC_TELEPHONE=zh_CN.UTF-8
LC_MEASUREMENT=en_US.UTF-8
LC_IDENTIFICATION=zh_CN.UTF-8

# 生成 locale
locale-gen zh_CN.UTF-8 en_US.UTF-8 ja_JP.UTF-8 ko_KR.UTF-8

# 验证 locale
locale -a | grep -E "zh_CN|en_US|ja_JP|ko_KR"
```

---

## 3. 内核日志 UTF-8 支持

### 3.1 printk 多语言消息表

agentrt-linux 在 Linux 6.6 内核基线上扩展 printk，支持运行时 locale 切换的多语言消息。所有内核消息以 UTF-8 编码存储，禁止使用 `wchar_t`（内核态不支持）。

```c
/* kernel/kernel/airy_i18n.c */
#include <linux/spinlock.h>
#include <linux/string.h>
#include <linux/types.h>
#include <uapi/airymax/locale.h>

/* 内核消息 ID 枚举（与 agentrt 共享，[SC] 共享契约层） */
enum airy_kmsg_id {
	KMSG_BOOT_START = 1,
	KMSG_SCHED_LOADED,
	KMSG_MEMORY_INIT,
	KMSG_IPC_READY,
	KMSG_CUPOLAS_DENY,
	KMSG_MEMORYROV_LOADED,
	KMSG_SCHED_TIMEOUT,
};

/* 多语言消息条目 */
struct airy_kmsg_entry {
	enum airy_kmsg_id id;
	const char *en_us;     /* 英文（默认） */
	const char *zh_cn;     /* 简体中文 */
	const char *ja_jp;     /* 日本語 */
};

/* 编译期消息表（rodata 段，无运行时分配） */
static const struct airy_kmsg_entry kmsg_table[] = {
	{KMSG_BOOT_START,
	 "agentrt-linux (AirymaxOS) starting up...",
	 "agentrt-linux（AirymaxOS，极境智能体操作系统）启动中...",
	 "agentrt-linux（AirymaxOS）を起動しています..."},
	{KMSG_SCHED_LOADED,
	 "User-space scheduler (sched_tac) loaded successfully",
	 "用户态调度器（sched_tac）加载成功",
	 "ユーザースペーススケジューラ（スキームsched_tac）を正常にロードしました"},
	{KMSG_MEMORY_INIT,
	 "MemoryRovol L1-L4 initialized, capacity=%lu MB",
	 "MemoryRovol L1-L4 已初始化，容量=%lu MB",
	 "MemoryRovol L1-L4 を初期化しました、容量=%lu MB"},
	{KMSG_IPC_READY,
	 "AgentsIPC io_uring ring ready, depth=%u",
	 "AgentsIPC io_uring 环就绪，深度=%u",
	 "AgentsIPC io_uring リングの準備ができました、深さ=%u"},
	{KMSG_CUPOLAS_DENY,
	 "Cupolas security dome denied op=%u pid=%d",
	 "Cupolas 安全穹顶拒绝操作 op=%u pid=%d",
	 "Cupolas セキュリティドームが拒否しました op=%u pid=%d"},
	{KMSG_MEMORYROV_LOADED,
	 "MemoryRovol layers L1=%u L2=%u L3=%u L4=%u pages loaded",
	 "MemoryRovol 层 L1=%u L2=%u L3=%u L4=%u 页已加载",
	 "MemoryRovol レイヤ L1=%u L2=%u L3=%u L4=%u ページをロードしました"},
	{KMSG_SCHED_TIMEOUT,
	 "stc_agent watchdog timeout, fallback to default scheduler",
	 "stc_agent 看门狗超时，降级到默认调度器",
	 "stc_agent ウォッチドッグがタイムアウト、デフォルトスケジューラに切り替えます"},
};

static DEFINE_SPINLOCK(kmsg_locale_lock);
static enum airy_locale_id kmsg_locale = AIRY_LOCALE_EN_US;

/* 设置内核日志 locale */
int airy_kmsg_set_locale(enum airy_locale_id id)
{
	unsigned long flags;

	if (id >= AIRY_LOCALE_MAX)
		return -EINVAL;

	spin_lock_irqsave(&kmsg_locale_lock, flags);
	kmsg_locale = id;
	spin_unlock_irqrestore(&kmsg_locale_lock, flags);
	return 0;
}

/* 查找消息条目（按 ID） */
static const struct airy_kmsg_entry *airy_kmsg_lookup(enum airy_kmsg_id id)
{
	size_t i;
	for (i = 0; i < ARRAY_SIZE(kmsg_table); i++)
		if (kmsg_table[i].id == id)
			return &kmsg_table[i];
	return NULL;
}

/* 根据 locale 选择消息文本 */
static const char *airy_kmsg_text(const struct airy_kmsg_entry *e,
				     enum airy_locale_id loc)
{
	switch (loc) {
	case AIRY_LOCALE_ZH_CN:
		return e->zh_cn ? e->zh_cn : e->en_us;
	case AIRY_LOCALE_JA_JP:
		return e->ja_jp ? e->ja_jp : e->en_us;
	default:
		return e->en_us;
	}
}

/* 多语言 printk */
#define airy_printk(level, msg_id, fmt_args...) \
	do { \
		const struct airy_kmsg_entry *__e; \
		unsigned long __flags; \
		enum airy_locale_id __loc; \
		__e = airy_kmsg_lookup(msg_id); \
		if (__e) { \
			spin_lock_irqsave(&kmsg_locale_lock, __flags); \
			__loc = kmsg_locale; \
			spin_unlock_irqrestore(&kmsg_locale_lock, __flags); \
			printk(level airy_kmsg_text(__e, __loc) "\n", ##fmt_args); \
		} \
	} while (0)

/* 使用示例 */
void airy_kernel_init_example(void)
{
	airy_printk(KERN_INFO, KMSG_BOOT_START);
	airy_printk(KERN_INFO, KMSG_SCHED_LOADED);
	airy_printk(KERN_INFO, KMSG_MEMORY_INIT, 16384UL);
	airy_printk(KERN_WARNING, KMSG_CUPOLAS_DENY, 42U, current->pid);
}
```

### 3.2 用户态 locale 切换 API

```c
/* 用户态：切换内核日志 locale */
#include <sys/ioctl.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <airymax/locale.h>

int airy_set_kernel_locale(enum airy_locale_id id)
{
	int fd, ret;

	fd = open("/dev/airy_i18n", O_RDWR);
	if (fd < 0)
		return -errno;

	ret = ioctl(fd, AIRY_IOC_SET_KMSG_LOCALE, &id);
	close(fd);
	return ret;
}

/* 使用示例 */
int main(void)
{
	/* 切换内核日志为简体中文 */
	airy_set_kernel_locale(AIRY_LOCALE_ZH_CN);

	/* dmesg 将输出中文 */
	/* 输出示例：
	 * [    0.123456] agentrt-linux（AirymaxOS，极境智能体操作系统）启动中...
	 * [    0.234567] 用户态调度器（sched_tac）加载成功
	 * [    0.345678] MemoryRovol L1-L4 已初始化，容量=16384 MB
	 */
	return 0;
}
```

---

## 4. 用户态 locale 机制

### 4.1 daemon 级 locale 配置

每个 daemon 通过 systemd unit 配置独立的 locale 环境：

```ini
# /etc/systemd/system/cogn_d.service —— cogn_d daemon 的 locale 配置
[Unit]
Description=agentrt-linux LLM Inference Daemon (cogn_d)
After=network.target

[Service]
Type=simple
ExecStart=/usr/lib/airymaxos/services/cogn_d
Environment=LANG=zh_CN.UTF-8
Environment=LC_MESSAGES=zh_CN.UTF-8
Environment=LC_CTYPE=zh_CN.UTF-8
Environment=AIRY_LOCALE_ID=2
Environment=AIRY_LOG_LOCALE=zh_CN.UTF-8
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 4.2 locale 切换伪代码（运行时动态切换）

```c
/* services/common/airy_locale.c */
#include <locale.h>
#include <libintl.h>
#include <stdlib.h>
#include <string.h>
#include <airymax/locale.h>

#define AIRY_TEXT_DOMAIN "airymaxos"

struct airy_locale_state {
	enum airy_locale_id id;
	char posix_name[32];
	char *mo_path;
};

/* 初始化 locale 系统 */
int airy_locale_init(struct airy_locale_state *st,
			enum airy_locale_id id)
{
	const char *posix;
	int ret;

	if (id >= AIRY_LOCALE_MAX)
		return -EINVAL;

	posix = airy_locales[id].posix_name;
	if (!setlocale(LC_ALL, posix))
		return -ENOENT;

	bindtextdomain(AIRY_TEXT_DOMAIN, "/usr/share/locale");
	bind_textdomain_codeset(AIRY_TEXT_DOMAIN, "UTF-8");
	textdomain(AIRY_TEXT_DOMAIN);

	st->id = id;
	ret = strscpy(st->posix_name, posix, sizeof(st->posix_name));
	if (ret < 0)
		return -E2BIG;
	return 0;
}

/* 运行时切换 locale（用于响应客户端请求语言偏好） */
int airy_locale_switch(struct airy_locale_state *st,
			  enum airy_locale_id new_id)
{
	const char *posix;
	int ret;

	if (new_id >= AIRY_LOCALE_MAX)
		return -EINVAL;
	if (new_id == st->id)
		return 0;

	posix = airy_locales[new_id].posix_name;
	if (!setlocale(LC_ALL, posix)) {
		ret = -ENOENT;
		goto out_err;
	}

	st->id = new_id;
	ret = strscpy(st->posix_name, posix, sizeof(st->posix_name));
	if (ret < 0) {
		ret = -E2BIG;
		goto out_err;
	}
	return 0;

out_err:
	return ret;
}

/* 根据客户端 Accept-Language 选择最佳 locale */
enum airy_locale_id airy_locale_from_accept_language(const char *accept)
{
	/* 简化版：按优先级匹配前缀 */
	if (!accept)
		return AIRY_LOCALE_EN_US;

	if (strncasecmp(accept, "zh-CN", 5) == 0 || strncasecmp(accept, "zh-Hans", 7) == 0)
		return AIRY_LOCALE_ZH_CN;
	if (strncasecmp(accept, "zh-TW", 5) == 0 || strncasecmp(accept, "zh-Hant", 7) == 0)
		return AIRY_LOCALE_ZH_TW;
	if (strncasecmp(accept, "ja", 2) == 0)
		return AIRY_LOCALE_JA_JP;
	if (strncasecmp(accept, "ko", 2) == 0)
		return AIRY_LOCALE_KO_KR;
	if (strncasecmp(accept, "de", 2) == 0)
		return AIRY_LOCALE_DE_DE;
	if (strncasecmp(accept, "fr", 2) == 0)
		return AIRY_LOCALE_FR_FR;
	if (strncasecmp(accept, "en-GB", 5) == 0)
		return AIRY_LOCALE_EN_GB;
	return AIRY_LOCALE_EN_US;
}
```

### 4.3 .po / .mo 翻译文件组织

```
/usr/share/locale/
├── zh_CN/LC_MESSAGES/
│   ├── airymaxos.mo              # 主消息目录
│   ├── airymaxos-cogn_d.mo        # cogn_d daemon
│   ├── airymaxos-sched_d.mo      # sched_d daemon
│   └── airymaxos-dev_d.mo       # dev_d daemon
├── en_US/LC_MESSAGES/
│   ├── airymaxos.mo
│   └── ...
├── ja_JP/LC_MESSAGES/
│   ├── airymaxos.mo
│   └── ...
└── ko_KR/LC_MESSAGES/
    └── ...
```

.po 文件示例：

```po
# /usr/share/locale/zh_CN/LC_MESSAGES/airymaxos.po
msgid ""
msgstr ""
"Project-Id-Version: agentrt-linux 1.0.1\n"
"Content-Type: text/plain; charset=UTF-8\n"
"Language: zh_CN\n"

msgid "agentrt-linux started"
msgstr "agentrt-linux（AirymaxOS）已启动"

msgid "Agent task submitted: %s"
msgstr "Agent 任务已提交：%s"

msgid "MemoryRovol query timeout"
msgstr "MemoryRovol 记忆查询超时"

msgid "Permission denied by Cupolas"
msgstr "Cupolas 安全穹顶拒绝访问"
```

---

## 5. 多区域记忆卷载 locale 隔离

### 5.1 MemoryRovol 区域配置

MemoryRovol L1-L4 按 locale 独立组织，避免跨语言知识污染：

```yaml
# /etc/airymaxos/memoryrovol/regions.yaml —— MemoryRovol 区域配置
memoryRovol:
  default_region: zh-CN
  regions:
    - name: zh-CN
      locale: zh_CN.UTF-8
      locale_id: 1
      knowledge_base: /var/lib/airymaxos/memoryrovol/zh-CN
      capacity_gb: 64
      forgetting_strategy: EBBINGHAUS
      tau_days: 7
    - name: en-US
      locale: en_US.UTF-8
      locale_id: 0
      knowledge_base: /var/lib/airymaxos/memoryrovol/en-US
      capacity_gb: 64
      forgetting_strategy: EBBINGHAUS
      tau_days: 7
    - name: ja-JP
      locale: ja_JP.UTF-8
      locale_id: 2
      knowledge_base: /var/lib/airymaxos/memoryrovol/ja-JP
      capacity_gb: 32
      forgetting_strategy: EBBINGHAUS
      tau_days: 7
```

### 5.2 跨区域查询与回退策略

当目标 locale 区域无结果时，按"语言家族回退"策略查询：

```c
/* 跨区域回退查询策略 */
static const enum airy_locale_id fallback_chain_zh_cn[] = {
	AIRY_LOCALE_ZH_CN,    /* 首选：简体中文 */
	AIRY_LOCALE_ZH_TW,    /* 回退 1：繁体中文 */
	AIRY_LOCALE_EN_US,   /* 回退 2：英文 */
};

static const enum airy_locale_id fallback_chain_ja_jp[] = {
	AIRY_LOCALE_JA_JP,    /* 首选：日本語 */
	AIRY_LOCALE_EN_US,   /* 回退 1：英文 */
};

static const enum airy_locale_id fallback_chain_en_us[] = {
	AIRY_LOCALE_EN_US,    /* 首选：英文，无回退 */
};
```

---

## 6. 国密算法 locale 合规

在 zh_CN.UTF-8 locale 下，自动启用国密算法（SM2/SM3/SM4）以满足合规要求：

```c
/* locale 感知的加密算法选择 */
enum airy_crypto_algo {
	AIRY_CRYPTO_RSA_SHA256,   /* 国际默认 */
	AIRY_CRYPTO_SM2_SM3,      /* 国密（zh_CN locale 强制） */
	AIRY_CRYPTO_AES_GCM,     /* 国际默认对称 */
	AIRY_CRYPTO_SM4,         /* 国密对称（zh_CN locale 强制） */
};

enum airy_crypto_algo airy_crypto_select_algo(enum airy_locale_id loc)
{
	switch (loc) {
	case AIRY_LOCALE_ZH_CN:
		return AIRY_CRYPTO_SM2_SM3;   /* 国密合规 */
	case AIRY_LOCALE_ZH_TW:
		return AIRY_CRYPTO_RSA_SHA256;
	default:
		return AIRY_CRYPTO_RSA_SHA256;
	}
}
```

---

## 7. 性能考量

### 7.1 locale 切换开销

| 操作 | 开销（ns） | 缓存策略 |
|------|-----------|----------|
| setlocale（首次） | 50,000-200,000 | 进程级缓存 |
| setlocale（已缓存） | 200-500 | glibc 内部缓存 |
| gettext 查询 | 100-300 | glibc mo 文件 mmap |
| 内核 kmsg_lookup | 50-150 | 线性表，编译期固化 |
| locale 切换 ioctl | 1,000-2,000 | 内核全局缓存 |

### 7.2 大规模并发场景下的优化

12 个 daemon 共享同一份 mo 文件（mmap 只读），避免重复加载：

```c
/* 共享 mo 文件 mmap */
void *airy_mo_mmap(const char *domain, const char *locale)
{
	char path[256];
	int fd;
	void *addr;

	snprintf(path, sizeof(path),
		"/usr/share/locale/%s/LC_MESSAGES/%s.mo", locale, domain);
	fd = open(path, O_RDONLY | O_CLOEXEC);
	if (fd < 0)
		return MAP_FAILED;

	addr = mmap(NULL, 0, PROT_READ, MAP_SHARED, fd, 0);
	close(fd);
	return addr;
}
```

---

## 8. 错误码体系对接

国际化子系统错误码纳入 agentrt-linux 统一错误码体系（[SC] 共享契约层，扩展段）：

| 错误码 | 数值 | 含义 |
|--------|------|------|
| AIRY_E_I18N_LOCALE_NOT_FOUND | -901 | locale 未安装 |
| AIRY_E_I18N_MO_LOAD | -902 | mo 文件加载失败 |
| AIRY_E_I18N_ENCODING | -903 | 编码错误 |
| AIRY_E_I18N_INVALID_UTF8 | -904 | 无效 UTF-8 序列 |
| AIRY_E_I18N_REGION | -905 | 区域配置错误 |

---

## 9. 五维原则映射

| 原则 | 在本设计的体现 |
|------|---------------|
| **A-3 人文关怀** | 多语言支持全球开发者与用户 |
| **E-7 文档即代码** | README.md + README_zh.md + README_ja.md 三语言同步 |
| **K-2 接口契约化** | locale 接口契约（POSIX + 自定义枚举） |
| **E-4 跨平台一致性** | 同一 locale 在不同架构行为一致 |
| **E-8 可测试性** | locale 切换与多语言消息覆盖测试 |

---

## 10. IRON-9 v3 同源映射

| 组件 | agentrt-linux（[SS]） | agentrt（[SS]） | 共享（[SC]） |
|------|------------------------|------------------|--------------|
| locale 枚举 | 内核 + 用户态 | 用户态 | locale_id 枚举 |
| 多语言错误码 | AIRY_E_I18N_* | AIRY_E_I18N_* | `error.h` 错误码段 |
| 内核消息表 | printk KMSG_* | 用户态日志 | KMSG_ID 枚举 |
| 国密算法 | LSM + capability | 用户态安全模块 | SM2/SM3/SM4 算法 ID |

---

## 11. 相关文档

- `180-i18n/README.md`（国际化主索引）
- `180-i18n/02-encoding-spec.md`（编码规范）
- `180-i18n/04-cjk-support.md`（CJK 支持设计）
- `20-modules/04-memory.md`（MemoryRovol 多区域）
- `110-security/README.md`（国密算法合规）

---

## 12. 参考材料

- Linux 6.6 `kernel/printk/`（printk 子系统）
- POSIX locale 规范（IEEE Std 1003.1）
- GNU gettext 文档
- glibc locale 机制源码
- agentrt 多语言错误码实现规范

---

> **文档结束** | agentrt-linux（AirymaxOS）Locale 设计详细规范 
