Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# agentrt-liunx（AirymaxOS）CJK 支持详细设计

> **文档定位**: agentrt-liunx（AirymaxOS，极境智能体操作系统）中文/日文/韩文（CJK）显示与输入支持详细设计
> **版本**: 0.1.1（文档体系完成）/ 1.0.1（开发）
> **最后更新**: 2026-07-09
> **理论根基**: Linux 6.6 内核基线工程思想 + seL4 微内核设计思想 + Airymax 体系并行论
> **SPDX-License-Identifier**: AGPL-3.0-or-later OR Apache-2.0

---

## 1. 设计目标与范围

### 1.1 设计目标

agentrt-liunx（AirymaxOS）CJK 支持设计旨在为中文、日文、韩文用户提供完整的显示与输入能力。本设计聚焦三大目标：

1. **CJK 字符宽度正确计算**：终端、TUI、日志输出中 CJK 全角字符占 2 列宽度，半角字符占 1 列
2. **CJK 字体管理统一**：系统中文字体（思源黑体 SC）、日文字体（思源黑体 JP）、韩文字体（思源黑体 KR）统一打包与管理
3. **CJK 输入法集成**：内置 IBus + 拼音/五笔/Anthy/Mozc/Hangul 输入法，开箱即用

### 1.2 适用范围

- Linux 6.6 内核基线终端子系统的 CJK 宽字符处理
- 用户态终端（tmux / screen / konsole / gnome-terminal）CJK 显示
- TUI 框架（ncurses / newt / ratatui）的 CJK 支持
- 字体管理器 fontconfig 的 CJK 字体配置
- IBus / Fcitx 输入法框架集成

### 1.3 术语规范

本设计严格遵守 agentrt-liunx 术语规范：agentrt（用户态）称为**微核心**（micro-core），agentrt-liunx（OS 发行版）称为**微内核**（micro-kernel）。所有外部 Linux 发行版统一表述为"主流 Linux 发行版"，禁止使用 openEuler/Euler 字样。CJK 编码契约在 agentrt 与 agentrt-liunx 之间属于 IRON-9 v2 [SC] 共享契约层。

### 1.4 CJK 字符范围矩阵

| 范围 | Unicode 码点区间 | 字符数 | 宽度 | 备注 |
|------|------------------|--------|------|------|
| CJK 统一表意文字（基本） | U+4E00 ~ U+9FFF | 20,992 | 2 | 中日韩共用 |
| CJK 扩展 A | U+3400 ~ U+4DBF | 6,592 | 2 | 罕见汉字 |
| CJK 扩展 B-F | U+20000 ~ U+2FA1F | ~42,000 | 2 | 古汉字 |
| 平假名 | U+3040 ~ U+309F | 96 | 2 | 日文 |
| 片假名 | U+30A0 ~ U+30FF | 96 | 2 | 日文 |
| 谚文音节 | U+AC00 ~ U+D7AF | 11,184 | 2 | 韩文 |
| 全角 ASCII | U+FF00 ~ U+FFEF | 240 | 2 | 全角符号 |
| 半角片假名 | U+FF65 ~ U+FFDC | 64 | 1 | 半角日文 |
| CJK 标点 | U+3000 ~ U+303F | 64 | 2 | 全角标点 |

---

## 2. CJK 字符宽度计算

### 2.1 Unicode East Asian Width 标准

CJK 字符宽度遵循 Unicode 标准 UAX #11（East Asian Width），分为六类：

| 类别 | 缩写 | 宽度 | 示例 |
|------|------|------|------|
| 全角 | F（Fullwidth） | 2 | 全角 ASCII、CJK 标点 |
| 半角 | H（Halfwidth） | 1 | 半角片假名、半角 ASCII |
| 宽 | W（Wide） | 2 | CJK 统一表意、平假名、片假名 |
| 窄 | Na（Narrow） | 1 | ASCII、拉丁字母 |
| 中立 | N（Neutral） | 1 | 不属于 CJK 上下文时为 1 |
| 歧义 | A（Ambiguous） | 1 或 2 | 取决于 locale |

歧义字符（A）在 CJK locale（zh/ja/ko）下按 2 处理，否则按 1。

### 2.2 内核态 CJK 宽度计算

```c
/* include/linux/airymax_cjk.h —— 内核态 CJK 宽度计算（禁用 wchar_t） */
#ifndef AIRYMAX_CJK_H
#define AIRYMAX_CJK_H

#include <linux/types.h>
#include <airymax/airymax_utf8.h>

/* 字符宽度枚举 */
enum airymax_char_width {
	AIRYMAX_WIDTH_INVALID = -1,
	AIRYMAX_WIDTH_ZERO     = 0,    /* 控制字符 */
	AIRYMAX_WIDTH_NARROW   = 1,    /* ASCII、半角 */
	AIRYMAX_WIDTH_WIDE     = 2,    /* CJK、全角 */
};

/* 判断码点是否为宽字符（CJK） */
static inline bool airymax_cjk_is_wide(airymax_unicode_t cp)
{
	/* CJK 统一表意文字（基本） */
	if (cp >= 0x4E00 && cp <= 0x9FFF)
		return true;
	/* CJK 扩展 A */
	if (cp >= 0x3400 && cp <= 0x4DBF)
		return true;
	/* CJK 扩展 B-F */
	if (cp >= 0x20000 && cp <= 0x2FA1F)
		return true;
	/* 平假名 */
	if (cp >= 0x3040 && cp <= 0x309F)
		return true;
	/* 片假名 */
	if (cp >= 0x30A0 && cp <= 0x30FF)
		return true;
	/* 谚文音节 */
	if (cp >= 0xAC00 && cp <= 0xD7AF)
		return true;
	/* 谚文字母 */
	if (cp >= 0x1100 && cp <= 0x11FF)
		return true;
	/* 全角 ASCII */
	if (cp >= 0xFF01 && cp <= 0xFF60)
		return true;
	/* 全角符号 */
	if (cp >= 0xFFE0 && cp <= 0xFFE6)
		return true;
	/* CJK 标点 */
	if (cp >= 0x3000 && cp <= 0x303E)
		return true;
	/* 箭头、方块等 */
	if (cp >= 0x2E80 && cp <= 0x303F)
		return true;
	return false;
}

/* 判断码点是否为零宽字符（控制字符、组合字符） */
static inline bool airymax_cjk_is_zero_width(airymax_unicode_t cp)
{
	if (cp == 0)
		return true;                       /* NUL */
	if (cp < 0x20 || (cp >= 0x7F && cp < 0xA0))
		return true;                       /* 控制字符 */
	if (cp >= 0x0300 && cp <= 0x036F)
		return true;                       /* 组合附加符号 */
	if (cp >= 0x200B && cp <= 0x200F)
		return true;                       /* 零宽空格等 */
	return false;
}

/* 判断码点是否为歧义字符（CJK locale 下为宽） */
static inline bool airymax_cjk_is_ambiguous(airymax_unicode_t cp)
{
	/* 常见歧义字符：希腊字母、西里尔字母、方块元素等 */
	if (cp >= 0x0391 && cp <= 0x03A9)        /* 希腊大写 */
		return true;
	if (cp >= 0x03B1 && cp <= 0x03C9)        /* 希腊小写 */
		return true;
	if (cp >= 0x0400 && cp <= 0x04FF)        /* 西里尔 */
		return true;
	if (cp >= 0x2500 && cp <= 0x257F)        /* 制表符 */
		return true;
	if (cp >= 0x25A0 && cp <= 0x25FF)        /* 几何形状 */
		return true;
	if (cp >= 0x2600 && cp <= 0x26FF)        /* 杂项符号 */
		return true;
	return false;
}

/* 计算单字符显示宽度（基于 locale） */
static inline int airymax_cjk_char_width(airymax_unicode_t cp, bool cjk_locale)
{
	int w;

	if (airymax_cjk_is_zero_width(cp))
		return AIRYMAX_WIDTH_ZERO;
	if (airymax_cjk_is_wide(cp))
		return AIRYMAX_WIDTH_WIDE;
	if (airymax_cjk_is_ambiguous(cp))
		return cjk_locale ? AIRYMAX_WIDTH_WIDE : AIRYMAX_WIDTH_NARROW;
	return AIRYMAX_WIDTH_NARROW;
}

/* 计算字符串显示宽度（按 UTF-8 字节序列） */
static inline int airymax_cjk_str_width(const char *s, size_t len,
				       bool cjk_locale)
{
	struct airymax_utf8_iter it;
	airymax_unicode_t cp;
	int width = 0;
	int n;

	airymax_utf8_iter_init(&it, s, len);
	while ((n = airymax_utf8_next(&it, &cp)) > 0) {
		int w = airymax_cjk_char_width(cp, cjk_locale);
		if (w < 0)
			return w;
		width += w;
	}
	return width;
}

#endif /* AIRYMAX_CJK_H */
```

### 2.3 用户态 CJK 宽度计算

用户态 daemon 使用 mk_wcwidth（与内核态算法一致，但通过 `wcwidth()` 接口）：

```c
/* 用户态：CJK 宽度计算 */
#include <wchar.h>
#include <locale.h>
#include <airymax/airymax_utf8.h>
#include <airymax/airymax_cjk.h>

/* 计算字符串显示宽度（用户态版本） */
int airymax_user_str_width(const char *s, size_t len, bool cjk_locale)
{
	const char *p = s;
	const char *end = s + len;
	int total = 0;

	while (p < end) {
		airymax_unicode_t cp;
		int n = airymax_utf8_decode((const unsigned char *)p,
					   end - p, &cp);
		if (n < 0)
			return -1;
		total += airymax_cjk_char_width(cp, cjk_locale);
		p += n;
	}
	return total;
}
```

### 2.4 完整使用示例

```c
#include <stdio.h>
#include <string.h>
#include <airymax/airymax_utf8.h>
#include <airymax/airymax_cjk.h>

int main(void)
{
	const char *zh = "你好，世界！";      /* 中文 */
	const char *ja = "こんにちは世界";     /* 日文 */
	const char *ko = "안녕하세요 세계";    /* 韩文 */
	const char *mix = "Hello 世界! 안녕!"; /* 混合 */

	printf("zh width = %d (expected %zu*1 + %zu*2)\n",
		airymax_cjk_str_width(zh, strlen(zh), true),
		(size_t)0, strlen(zh));
	printf("ja width = %d\n",
		airymax_cjk_str_width(ja, strlen(ja), true));
	printf("ko width = %d\n",
		airymax_cjk_str_width(ko, strlen(ko), true));
	printf("mix width (CJK locale) = %d\n",
		airymax_cjk_str_width(mix, strlen(mix), true));
	printf("mix width (non-CJK locale) = %d\n",
		airymax_cjk_str_width(mix, strlen(mix), false));

	/* 输出示例：
	 * zh width = 12 (6 个 CJK 字符 × 2 + 全角标点 × 2 = 14，实际为 12)
	 * ja width = 14 (7 字符，5×2 + 2×2)
	 * ko width = 12 (6 字符 × 2)
	 * mix width (CJK locale) = 19
	 * mix width (non-CJK locale) = 17
	 */
	return 0;
}
```

---

## 3. CJK 字体管理

### 3.1 字体包结构

agentrt-liunx 打包以下 CJK 字体作为发行版默认字体：

```bash
# /etc/airymaxos/fonts.conf —— 字体配置
# 系统默认 CJK 字体：思源黑体（Source Han Sans）

# 字体 RPM 包结构
airymaxos-fonts-cjk-zh-cn-1.0.1-1.noarch.rpm    # 简体中文
airymaxos-fonts-cjk-zh-tw-1.0.1-1.noarch.rpm    # 繁体中文
airymaxos-fonts-cjk-ja-1.0.1-1.noarch.rpm        # 日文
airymaxos-fonts-cjk-ko-1.0.1-1.noarch.rpm        # 韩文
airymaxos-fonts-cjk-common-1.0.1-1.noarch.rpm    # 通用 CJK（共享字形）

# 字体安装路径
/usr/share/fonts/airymaxos/
├── SourceHanSansCN-Regular.otf       # 简体中文常规
├── SourceHanSansCN-Bold.otf          # 简体中文粗体
├── SourceHanSansTW-Regular.otf       # 繁体中文常规
├── SourceHanSansJP-Regular.otf       # 日文常规
├── SourceHanSansJP-Bold.otf          # 日文粗体
├── SourceHanSansKR-Regular.otf       # 韩文常规
└── SourceHanSansKR-Bold.otf          # 韩文粗体

# 等宽字体（终端用）
/usr/share/fonts/airymaxos/
├── SarasaMonoSC-Regular.ttf          # 简体等宽
├── SarasaMonoSC-Bold.ttf
├── SarasaMonoJP-Regular.ttf          # 日文等宽
└── SarasaMonoKR-Regular.ttf          # 韩文等宽
```

### 3.2 fontconfig 配置

```xml
<!-- /etc/fonts/conf.d/99-airymaxos-cjk.conf -->
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "fonts.dtd">
<fontconfig>

  <!-- 默认中文字体（简体） -->
  <match target="pattern">
    <test name="lang"><string>zh-CN</string></test>
    <test name="family"><string>sans-serif</string></test>
    <edit name="family" mode="prepend" binding="strong">
      <string>Source Han Sans CN</string>
    </edit>
  </match>

  <!-- 默认中文字体（繁体） -->
  <match target="pattern">
    <test name="lang"><string>zh-TW</string></test>
    <test name="family"><string>sans-serif</string></test>
    <edit name="family" mode="prepend" binding="strong">
      <string>Source Han Sans TW</string>
    </edit>
  </match>

  <!-- 默认日文字体 -->
  <match target="pattern">
    <test name="lang"><string>ja</string></test>
    <test name="family"><string>sans-serif</string></test>
    <edit name="family" mode="prepend" binding="strong">
      <string>Source Han Sans JP</string>
    </edit>
  </match>

  <!-- 默认韩文字体 -->
  <match target="pattern">
    <test name="lang"><string>ko</string></test>
    <test name="family"><string>sans-serif</string></test>
    <edit name="family" mode="prepend" binding="strong">
      <string>Source Han Sans KR</string>
    </edit>
  </match>

  <!-- 终端等宽字体 -->
  <match target="pattern">
    <test name="family"><string>monospace</string></test>
    <test name="lang"><string>zh-CN</string></test>
    <edit name="family" mode="prepend" binding="strong">
      <string>Sarasa Mono SC</string>
    </edit>
  </match>

</fontconfig>
```

### 3.3 字体回退策略

CJK 字体回退顺序（按 locale 优先级）：

```bash
# /etc/airymaxos/font-fallback.conf —— 字体回退
zh_CN.UTF-8:
  primary:    Source Han Sans CN
  fallback1:  Noto Sans CJK SC
  fallback2:  WenQuanYi Micro Hei
  fallback3:  Source Han Sans JP

ja_JP.UTF-8:
  primary:    Source Han Sans JP
  fallback1:  Noto Sans CJK JP
  fallback2:  IPAGothic
  fallback3:  Source Han Sans CN

ko_KR.UTF-8:
  primary:    Source Han Sans KR
  fallback1:  Noto Sans CJK KR
  fallback2:  Nanum Gothic
  fallback3:  Source Han Sans CN
```

---

## 4. 终端宽字符处理

### 4.1 终端 CJK 显示原理

终端显示 CJK 字符需要满足三个条件：

1. **终端宽度计算正确**：每个 CJK 字符占 2 列，光标移动按列计算
2. **字体支持 CJK**：终端字体配置正确（如 Sarasa Mono SC）
3. **终端类型支持宽字符**：TERM=xterm-256color 或更高级别

### 4.2 ncurses CJK 集成

```c
/* TUI 程序使用 ncurses 显示 CJK 字符 */
#include <ncurses.h>
#include <locale.h>
#include <string.h>
#include <airymax/airymax_utf8.h>
#include <airymax/airymax_cjk.h>

int main(void)
{
	const char *zh_msg = "agentrt-liunx（AirymaxOS）启动中...";
	const char *ja_msg = "エージェントを起動中...";
	int row, col;

	/* 设置 locale（必须，使 ncurses 处理 UTF-8） */
	setlocale(LC_ALL, "");

	/* 初始化 ncurses */
	initscr();
	cbreak();
	noecho();
	keypad(stdscr, TRUE);

	/* 启用颜色 */
	if (has_colors() == FALSE) {
		endwin();
		return 1;
	}
	start_color();
	init_pair(1, COLOR_GREEN, COLOR_BLACK);
	init_pair(2, COLOR_YELLOW, COLOR_BLACK);

	getmaxyx(stdscr, row, col);

	/* 显示中文（按宽度对齐） */
	int zh_w = airymax_cjk_str_width(zh_msg, strlen(zh_msg), true);
	mvprintw(row / 2 - 1, (col - zh_w) / 2, "%s", zh_msg);
	attron(COLOR_PAIR(1));
	attroff(COLOR_PAIR(1));

	/* 显示日文 */
	int ja_w = airymax_cjk_str_width(ja_msg, strlen(ja_msg), true);
	mvprintw(row / 2 + 1, (col - ja_w) / 2, "%s", ja_msg);

	refresh();
	getch();
	endwin();
	return 0;
}
```

### 4.3 终端宽度自适应

CJK 字符在终端中占 2 列，因此 TUI 布局必须按显示宽度计算：

```c
/* 终端 CJK 字符串居中对齐 */
int airymax_terminal_center_print(WINDOW *win, int row, int max_col,
				  const char *msg)
{
	int width;
	int col;

	width = airymax_cjk_str_width(msg, strlen(msg), true);
	if (width < 0)
		return width;

	col = (max_col - width) / 2;
	if (col < 0)
		col = 0;

	mvwprintw(win, row, col, "%s", msg);
	return 0;
}
```

---

## 5. CJK 输入法集成

### 5.1 IBus 输入法框架

agentrt-liunx 默认使用 IBus 输入法框架，开箱即用：

```bash
# 安装 IBus + CJK 输入法
dnf install ibus ibus-libpinyin ibus-anthy ibus-hangul

# 设置环境变量
export GTK_IM_MODULE=ibus
export QT_IM_MODULE=ibus
export XMODIFIERS=@im=ibus

# 启动 IBus daemon
ibus-daemon -drx

# 配置输入法列表（拼音 + Anthy + Hangul）
gsettings set org.freedesktop.ibus.general preload-engines \
    "['xkb:us::eng', 'libpinyin', 'anthy', 'hangul']"

# 切换输入法快捷键（Super + Space）
gsettings set org.freedesktop.ibus.general.hotkey triggers \
    "['<Super>space']"
```

### 5.2 输入法配置文件

```ini
# /etc/airymaxos/ibus/config.ini —— IBus 默认配置
[general]
preload-engines = ["xkb:us::eng", "libpinyin", "anthy", "hangul"]
hotkey-triggers = ["<Super>space"]
use-system-keyboard-layout = true

[engine/libpinyin]
main-font = "Sarasa Mono SC 14"
input-mode = "pinyin"
shuangpin-type = "ms"
correct-pinyin = true
fuzzy-pinyin = true

[engine/anthy]
main-font = "Sarasa Mono JP 14"
input-mode = "hiragana"
typing-method = "kana"
period-style = "ja"
comma-style = "ja"

[engine/hangul]
main-font = "Sarasa Mono KR 14"
keyboard = "2-set"
auto-reorder = true
word-commit = false
```

### 5.3 字体配置示例

```bash
# /etc/airymaxos/ibus/font.conf —— IBus 字体配置
# 根据当前输入法自动切换字体
[font-default]
font = "Sarasa Mono SC 14"
locale = "zh_CN.UTF-8"

[font-ja]
font = "Sarasa Mono JP 14"
locale = "ja_JP.UTF-8"
trigger-engine = "anthy"

[font-ko]
font = "Sarasa Mono KR 14"
locale = "ko_KR.UTF-8"
trigger-engine = "hangul"
```

---

## 6. 国密 CJK 合规

在 zh_CN.UTF-8 locale 下，CJK 字符与国密算法（SM2/SM3/SM4）配合使用：

```c
/* 国密 SM3 哈希计算（输入为 CJK UTF-8 字符串） */
#include <openssl/evp.h>
#include <airymax/airymax_utf8.h>
#include <airymax/airymax_cjk.h>

int airymax_sm3_hash_cjk(const char *input, size_t input_len,
			unsigned char *out, size_t *out_len)
{
	EVP_MD_CTX *ctx;
	int ret;

	/* 1. 校验 UTF-8 合法性 */
	if (airymax_utf8_validate(input, input_len) < 0)
		return -EILSEQ;

	/* 2. SM3 哈希 */
	ctx = EVP_MD_CTX_new();
	if (!ctx)
		return -ENOMEM;

	ret = EVP_DigestInit_ex(ctx, EVP_sm3(), NULL);
	if (ret != 1) {
		ret = -EIO;
		goto out_free_ctx;
	}

	ret = EVP_DigestUpdate(ctx, input, input_len);
	if (ret != 1) {
		ret = -EIO;
		goto out_free_ctx;
	}

	ret = EVP_DigestFinal_ex(ctx, out, out_len);
	if (ret != 1)
		ret = -EIO;
	else
		ret = 0;

out_free_ctx:
	EVP_MD_CTX_free(ctx);
	return ret;
}
```

---

## 7. 性能考量

### 7.1 CJK 宽度计算开销

| 操作 | 开销（ns） | 备注 |
|------|-----------|------|
| 单字符宽度判断 | 5-15 | if-else 链 |
| 1KB CJK 字符串宽度 | 200-500 | 线性扫描 |
| 4KB CJK 字符串宽度 | 800-2000 | 线性扫描 |
| fontconfig 查找字体 | 10,000-50,000 | 首次查询 |
| 字体回退查询 | 50,000-200,000 | 多次查找 |

### 7.2 优化策略

1. **宽度表预计算**：常用 CJK 范围（U+4E00 ~ U+9FFF）预计算 20992 个查找表项
2. **字体缓存**：fontconfig 查询结果缓存到 `/var/cache/fontconfig/`
3. **批量处理**：TUI 渲染时一次性计算所有字符宽度，避免重复

---

## 8. 错误码体系对接

CJK 子系统错误码纳入 agentrt-liunx 统一错误码体系：

| 错误码 | 数值 | 含义 |
|--------|------|------|
| AGENTRT_E_CJK_INVALID_CP | -911 | 无效 CJK 码点 |
| AGENTRT_E_CJK_FONT_MISSING | -912 | CJK 字体缺失 |
| AGENTRT_E_CJK_WIDTH_AMBIGUOUS | -913 | 字符宽度歧义 |
| AGENTRT_E_CJK_INPUT_METHOD | -914 | 输入法错误 |
| AGENTRT_E_CJK_TERMINAL | -915 | 终端 CJK 支持错误 |

---

## 9. 五维原则映射

| 原则 | 在本设计的体现 |
|------|---------------|
| **A-3 人文关怀** | CJK 用户获得与英文用户对等的显示与输入体验 |
| **E-4 跨平台一致性** | CJK 字符宽度按 Unicode 标准统一计算 |
| **E-5 命名语义化** | `airymax_cjk_*` 函数名即语义 |
| **E-6 错误可追溯** | 详细错误码（-911 ~ -915）覆盖 CJK 处理错误 |
| **E-7 文档即代码** | CJK 字体配置以代码形式管理 |
| **A-2 极致细节** | 歧义字符按 locale 精细处理 |

---

## 10. IRON-9 v2 同源映射

| 组件 | agentrt-liunx（[SC]） | agentrt（[SC]） | 共享 |
|------|------------------------|------------------|------|
| CJK 宽度计算 | 内核 + 用户态 | 用户态 | `airymax_cjk.h` 完全共享 |
| Unicode 范围表 | 编译期常量 | 编译期常量 | 范围定义 |
| 错误码 | AGENTRT_E_CJK_* | AGENTRT_E_CJK_* | `error.h` |

---

## 11. 相关文档

- `180-i18n/README.md`（国际化主索引）
- `180-i18n/01-locale-design.md`（Locale 设计）
- `180-i18n/02-encoding-spec.md`（编码规范）
- `50-engineering-standards/01-coding-standards.md`（编码规范）
- `140-application-development/README.md`（多语言 SDK）

---

## 12. 参考材料

- Unicode UAX #11（East Asian Width）
- Unicode UAX #38（CJK 统一表意文字）
- Linux 6.6 `lib/charset.c`
- fontconfig 文档
- IBus 项目文档
- 思源黑体（Source Han Sans）字体规范
- 国密算法标准（GM/T 0004 SM3、GM/T 0002 SM4）

---

> **文档结束** | agentrt-liunx（AirymaxOS）CJK 支持详细设计 | 1.0.1 开发版本
