# 普通用户操作手册：安全配置 secrets.env 并验证生效

> **文档定位**：面向非开发者的操作手册\
> **适用对象**：使用 AgentRT 但不想接触命令行/代码的普通用户\
> **最后更新**：2026-08-07\
> **上级文档**：[AirymaxAgentRT 文档中心](../README.md)

---

## 1. 这个文件是干什么的？

`secrets.env` 是 AgentRT 存放 **LLM 服务商 API Key**（密钥）的唯一位置。

你可以把它理解为"智能体的钥匙串"——你填了哪家服务商的钥匙，AgentRT 就能用哪家来思考、回答、执行任务。不填钥匙，AgentRT 的服务（聊天、任务编排等）就无法真正工作。

**核心结论（记住这一条就够了）**：

> 打开 `secrets.env`，把钥匙粘进去，保存。**不需要重启、不需要敲任何命令**，下一次使用时自动生效。

---

## 2. 这个文件在哪里？

`secrets.env` 位于 AgentRT 的安装目录下：

| 概念 | 位置 |
|------|------|
| 安装目录（AIRY_HOME） | `~/.airymaxrt` |
| 配置文件 | `~/.airymaxrt/config/secrets.env` |

> 提示：`~` 表示你的个人主目录。不同操作系统的实际路径：
> - Linux/macOS：`/home/你的用户名/.airymaxrt/config/secrets.env` 或 `/Users/你的用户名/.airymaxrt/config/secrets.env`
> - Windows：`C:\Users\你的用户名\.airymaxrt\config\secrets.env`

如果你是通过安装向导（`get-agentrt.sh`）安装的，安装完成后这个文件会自动生成好，里面是空模板，等你填。

---

## 3. 如何安全地填写

### 3.1 打开文件

用任意文本编辑器（记事本、VS Code、文本编辑等）打开：

```bash
# Linux/macOS
nano ~/.airymaxrt/config/secrets.env

# 或用图形界面
xdg-open ~/.airymaxrt/config/secrets.env
```

### 3.2 文件长什么样

文件里每一行是一个"钥匙"，格式是 `名字=钥匙内容`：

```text
# ── DeepSeek（默认提供商）──────────────────────────────────────────────
DEEPSEEK_API_KEY=
```

`=` 后面是空的，等你把从服务商官网申请到的 API Key 粘进去。

### 3.3 填写步骤（以 DeepSeek 为例）

1. 打开 DeepSeek 开放平台（platform.deepseek.com），注册并创建 API Key，复制它以 `sk-` 开头的字符串
2. 回到 `secrets.env`，找到这一行：

   ```text
   DEEPSEEK_API_KEY=
   ```

3. 把 Key 粘到 `=` 后面，**不要留空格**，保存：

   ```text
   DEEPSEEK_API_KEY=sk-1234567890abcdef
   ```

4. 完成。**保存即生效，无需重启。**

### 3.4 支持哪些服务商

| 服务商 | 变量名 | 申请地址 |
|--------|--------|----------|
| DeepSeek（默认） | `DEEPSEEK_API_KEY` | platform.deepseek.com |
| OpenAI | `OPENAI_API_KEY` | platform.openai.com |
| Anthropic Claude | `ANTHROPIC_API_KEY` | console.anthropic.com |
| Google Gemini | `GOOGLE_AI_API_KEY` | aistudio.google.com |
| 智谱 GLM | `GLM_API_KEY` | open.bigmodel.cn |
| 通义千问 Qwen | `DASHSCOPE_API_KEY` | dashscope.aliyuncs.com |
| Moonshot Kimi | `MOONSHOT_API_KEY` | platform.moonshot.cn |
| 硅基流动 | `SILICONFLOW_API_KEY` | cloud.siliconflow.cn |
| 讯飞星火 | `SPARK_API_KEY` | xinghuo.xfyun.cn |
| 自定义 OpenAI 兼容 | `CUSTOM_LLM_API_KEY` | 你的服务商 |

> 只需填写你实际使用的服务商，其余留空即可，不影响使用。

---

## 4. 安全须知（必读）

### ✅ 应该这样做

- **文件权限设为仅自己可读写**（安装时已自动设置，可再次确认）：

  ```bash
  chmod 600 ~/.airymaxrt/config/secrets.env
  ```

- **只把 Key 填进这个文件**，不要填到别的地方（聊天窗口、代码、网页表单等）
- **换新 Key 时**：直接覆盖保存即可，下一次请求自动生效
- **废弃的 Key**：从文件里清空该行保存

### ❌ 绝对不要这样做

- ❌ 不要把你的 Key 发给任何人、任何群、任何"帮你验证"的网站
- ❌ 不要把 `secrets.env` 文件内容截图发到网上
- ❌ 不要把 Key 写进聊天对话中让 AgentRT 帮你"测试"（它不会在日志中显示 Key，但避免不必要的传播）
- ❌ 不要把 `secrets.env` 放进任何代码仓库（它已被自动忽略，不会上传）
- ❌ 不要在公共电脑上保存 Key 后不清理

### 🔒 AgentRT 是如何保护你的 Key 的

- Key 只保存在 `secrets.env` 中，安装目录权限为 600（仅本人可读写）
- `secrets.env` 不进入版本控制（git 自动忽略），不会随代码上传
- AgentRT 日志**不会记录 Key 内容**，只记录"Key 来自哪个环境变量"
- 每次请求使用的 Key 仅在进程内存中短暂存在，用后即清除

---

## 5. 如何验证是否生效

### 方法一：直接和 AgentRT 对话（最直观）

1. 启动 AgentRT（安装时已配置好的启动方式）
2. 打开交互界面，输入一句简单的话，例如"你好，请介绍一下你自己"
3. 如果 AgentRT 正常回答，说明 Key 配置成功

### 方法二：查看服务状态

在浏览器或命令行中访问本地健康检查接口：

```bash
curl http://127.0.0.1:8080/health
```

返回 `{"status":"healthy"}` 即服务正常。

### 方法三：查看模型列表

```bash
curl -X POST http://127.0.0.1:8080/jsonrpc \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"llm.list_models","params":{}}'
```

返回的 `models` 列表中包含你配置的模型，说明已加载。

### 方法四：查看日志确认热加载（进阶）

如果希望确认"刚才保存的 Key 已被读取"，可以查看运行日志：

```bash
# Linux/macOS
grep "KEY-RELOAD" ~/.airymaxrt/logs/llm_d.log
```

出现类似 `KEY-RELOAD api_key_env=DEEPSEEK_API_KEY source=secrets.env` 的记录，表示热加载成功。

> 注意：这条日志是诊断级别的，正常运行时可能不显示。**验证 Key 是否有效最可靠的方式是方法一（直接对话）。**

---

## 6. 常见问题（FAQ）

### Q1：我填了 Key，保存了，但 AgentRT 还是不回答？

- 确认 Key 是否完整复制（以 `sk-` 开头，没有前后空格）
- 确认填的是**当前有效**的 Key（服务商后台可能已过期/被删除）
- 确认填写的位置正确（`=` 后、无空格）
- 若仍不行，尝试重启一次 AgentRT（热加载绝大多数情况生效，个别边缘场景需重启）

### Q2：怎么知道我的 Key 是不是失效了？

调用对话接口，若返回错误中包含 `401` 或 `invalid API key`，说明 Key 已失效，需要去服务商后台重新生成。

### Q3：填了多个服务商的 Key，用哪个？

AgentRT 有默认优先级（通常 DeepSeek 优先），也可在配置中指定。未指定时自动使用已配置的服务商。

### Q4：`secrets.env` 会被别人看到吗？

默认权限 600（仅自己可读写），且不在版本控制中。只要不主动传播，别人无法看到。

### Q5：我想换一家服务商，怎么操作？

在 `secrets.env` 中填写新服务商的 Key 并保存即可，无需卸载重装。

### Q6：安装目录可以换位置吗？

可以。安装时指定安装目录（如 `--prefix` 参数）。无论安装在哪，`secrets.env` 始终在 `安装目录/config/secrets.env`。

---

## 7. 一句话总结

| 操作 | 方法 |
|------|------|
| 配置 Key | 编辑 `~/.airymaxrt/config/secrets.env`，保存即生效 |
| 验证 | 打开 AgentRT 对话问一句话 |
| 换 Key | 直接覆盖保存 |
| 安全 | 权限 600，不分享、不上传、不进日志 |

---

*如果你仍然遇到问题，请将错误提示信息（不含 Key 内容）提供给技术支持。*
