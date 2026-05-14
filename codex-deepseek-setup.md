---
name: codex-model-integration
description: 通用 Codex 第三方大模型接入 Skill。支持将任意兼容 OpenAI Chat Completions 的模型（DeepSeek/通义千问/Groq/Mistral/智谱/Kimi/百川/SiliconFlow/OpenRouter 等）接入 Codex 桌面版。自动完成代理搭建、协议翻译、配置写入、路由修复、模型目录生成、开机自启、端到端验证。用户只需告知模型名 + Base URL + API Key，其余全自动。
metadata:
  type: skill
  triggers:
    - 接入
    - 模型
    - 换模型
    - 配置模型
    - 连不上
    - 模型连不上
    - deepseek
    - qwen
    - 千问
    - groq
    - mistral
    - 智谱
    - kimi
    - moonshot
    - 百川
    - siliconflow
    - openrouter
    - 第三方模型
    - api_key
    - base_url
---

# Codex 第三方大模型通用接入 Skill

> **你的角色：** 用户的 Codex 助手。用户想把 Codex 桌面版接入某个第三方大模型。所有兼容 OpenAI Chat Completions 格式的模型都能接。本 Skill 教你自动完成全套配置。用户提供模型信息 + API Key，剩下你全干。

## 原理

Codex 默认说 **Responses API**（OpenAI 私有协议），第三方模型说 **Chat Completions API**（行业标准）。在本地运行 `codex-bridge` 做实时协议翻译：

```
Codex ──Responses──▶ codex-bridge (:4000) ──Chat Completions──▶ 第三方模型 API
```

不管目标是什么模型，只要它提供 OpenAI 兼容的 Chat Completions 端点，这套方案都适用。

---

## ⚠️ 关键设计原则

1. **你干所有技术活**：执行命令、写文件、改代码。用户只提供信息。
2. **先识别模型再干活**：不同模型 = 不同 Base URL + 不同 Model Slug。不要瞎猜。
3. **所有配置路径里 `<用户名>` 要用实际值替换。** 执行 `whoami` 获取。

---

## 🤖 自动配置流程

### 步骤 0：识别目标模型

执行 `whoami` 获取 macOS 用户名。

问用户三个问题：

> 1. 你要接入哪个模型 / 供应商？
> 2. 它的 Chat Completions Base URL 是什么？
> 3. API Key 是什么？

如果用户只知道「模型名」不知道 Base URL，按下表推断。表里没有的，告诉用户去该模型官方文档找 Chat Completions endpoint。

### 已知供应商速查表

| 供应商 / 模型 | Base URL |
|---|---|
| DeepSeek | `https://api.deepseek.com/v1` |
| 通义千问 / Qwen / DashScope | `https://dashscope.aliyuncs.com/compatible-mode/v1` |
| Groq | `https://api.groq.com/openai/v1` |
| Mistral | `https://api.mistral.ai/v1` |
| 智谱 / GLM / Zhipu | `https://open.bigmodel.cn/api/paas/v4` |
| Moonshot / Kimi | `https://api.moonshot.cn/v1` |
| 百川 / Baichuan | `https://api.baichuan-ai.com/v1` |
| SiliconFlow / 硅基流动 | `https://api.siliconflow.cn/v1` |
| 零一万物 / Yi | `https://api.lingyiwanwu.com/v1` |
| Together AI | `https://api.together.xyz/v1` |
| OpenRouter | `https://openrouter.ai/api/v1` |
| 火山引擎 / 豆包 | `https://ark.cn-beijing.volces.com/api/v3` |

### 已知供应商可用模型速查

| 供应商 | 推荐填的模型 slug |
|---|---|
| DeepSeek | `deepseek-chat`, `deepseek-reasoner` |
| 通义千问 | `qwen-plus`, `qwen-max`, `qwen-turbo-latest` |
| Groq | `llama-3.3-70b-versatile`, `mixtral-8x7b-32768` |
| Mistral | `mistral-large-latest`, `mistral-small-latest` |
| 智谱 | `glm-4-plus`, `glm-4-flash` |
| Moonshot | `moonshot-v1-8k`, `moonshot-v1-32k` |
| 百川 | `Baichuan4` |
| SiliconFlow | `Qwen/Qwen2.5-72B-Instruct`, `deepseek-ai/DeepSeek-V3` |
| 零一万物 | `yi-large` |
| OpenRouter | 任意模型的完整路径如 `openai/gpt-4o` |

如果用户要接的模型不在上表，让用户自己提供模型 slug 列表。

从用户给的信息中提取三个关键变量，后续步骤全部引用它们：

- **`<BASE_URL>`**：Chat Completions 端点
- **`<API_KEY>`**：用户提供的 API Key
- **`<PROVIDER>`**：供应商简称（用于 .env 和显示，英文小写无空格）
- **`<MODELS>`**：逗号分隔的模型 slug 列表
- **`<USER>`**：`whoami` 结果

---

### 步骤 1：环境检查

```bash
node --version    # ≥ 18
whoami            # 确认用户名 = <USER>
```

---

### 步骤 2：克隆 codex-bridge

```bash
git clone https://github.com/wujfeng712-ui/codex-bridge.git ~/.codex/codex-bridge
```

目录已存在则跳过克隆。如果已存在，先读 `~/.codex/codex-bridge/proxy.mjs` 检查是否需要步骤 7 的修复。

---

### 步骤 3：生成 .env

先生成本地代理密钥：
```bash
openssl rand -hex 24
```

路径：`~/.codex/codex-bridge/.env`

用文件工具写入（替换 `<...>` 占位符为实际值）：

```
PROXY_AUTH_KEY=sk-proxy-local-<RANDOM_HEX>
PROVIDER_API_KEY=<API_KEY>
PROVIDER_BASE_URL=<BASE_URL>
PROVIDER_MODELS=<MODELS>
DEFAULT_PROVIDER=<PROVIDER>
LOG_LEVEL=info
MODEL_CATALOG_PATH=/Users/<USER>/.codex/proxy-models.json
```

⚠️ 如果 codex-bridge 版本较老只用 `DEEPSEEK_API_KEY` 而不用 `PROVIDER_API_KEY`，需要额外注意——这种情况下要读 proxy.mjs 确认环境变量名。如果只有 DeepSeek 专属变量，那就用 `DEEPSEEK_API_KEY` / `DEEPSEEK_BASE_URL` / `DEEPSEEK_MODELS` 作为变量名。

**记下 `PROXY_AUTH_KEY` 值，步骤 5 用。**

---

### 步骤 4：配置 config.toml

读取 `~/.codex/config.toml` 现有内容，合并以下配置（不覆盖无关字段）：

```toml
model = "<MODELS列表中第一个>"
model_provider = "local_proxy"
model_catalog_json = "/Users/<USER>/.codex/proxy-models.json"

[model_providers.local_proxy]
name = "local_proxy"
base_url = "http://127.0.0.1:4000/v1"
wire_api = "responses"
requires_openai_auth = true

[projects."/Users/<USER>"]
trust_level = "trusted"
```

---

### 步骤 5：配置 auth.json

路径：`~/.codex/auth.json`

```json
{
  "auth_mode": "apikey",
  "OPENAI_API_KEY": "<步骤3的PROXY_AUTH_KEY>"
}
```

---

### 步骤 6：生成模型目录

路径：`~/.codex/proxy-models.json`

根据步骤 0 确定的 `<MODELS>` 列表生成。每条模型格式：

```json
{
  "slug": "<模型slug>",
  "display_name": "<显示名，首字母大写，人类可读>",
  "default_reasoning_level": "medium",
  "supported_reasoning_levels": [
    { "effort": "none",    "description": "Thinking disabled" },
    { "effort": "minimal", "description": "Minimal reasoning" },
    { "effort": "low",     "description": "Low reasoning" },
    { "effort": "medium",  "description": "Medium reasoning" },
    { "effort": "high",    "description": "High reasoning" },
    { "effort": "xhigh",   "description": "Extra-high reasoning" }
  ],
  "shell_type": "default",
  "visibility": "list",
  "supported_in_api": true,
  "priority": <从0递增的整数>,
  "truncation_policy": { "mode": "tokens", "limit": 204800 },
  "supports_parallel_tool_calls": true
}
```

每个模型一条，`priority` 从 0 递增。整 JSON 包在 `{ "models": [...] }` 里。

---

### 步骤 7：修复 proxy.mjs（关键！）

打开 `~/.codex/codex-bridge/proxy.mjs`，做三处修改：

#### 7a：提取 URL path（防止 query string 导致路由匹配失败）

在 `http.createServer` 回调开头，`const authHeader =` 之前插入：

```js
const urlPath = req.url.split("?")[0];
```

然后全局搜索 `req.url ===` / `req.url.startsWith(`，全部替换为对应的 `urlPath ===` / `urlPath.startsWith(`。

#### 7b：/v1/models 免认证

找到 `isHealth` 定义行，确保 `/v1/models` 和 `/models` 在免认证列表：

```js
const isHealth = req.method === "GET" && (
  urlPath === "/health" || urlPath === "/" ||
  urlPath === "/v1/models" || urlPath === "/models"
);
```

#### 7c：模型列表响应加 models 字段

找到 `/v1/models` 路由的 `sendJson`，确保响应格式：

```js
sendJson(res, 200, {
  object: "list",
  data: modelCatalog,
  models: modelCatalog,
  default_provider: getFallbackProvider(),
});
```

---

### 步骤 8：开机自启（macOS）

路径：`~/Library/LaunchAgents/com.codex.bridge.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.codex.bridge</string>
    <key>ProgramArguments</key>
    <array>
        <string>/opt/homebrew/bin/node</string>
        <string>--env-file=/Users/<USER>/.codex/codex-bridge/.env</string>
        <string>/Users/<USER>/.codex/codex-bridge/proxy.mjs</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/Users/<USER>/.codex/codex-bridge</string>
    <key>RunAtLoad</key><true/>
    <key>KeepAlive</key><true/>
    <key>StandardOutPath</key><string>/tmp/codex-bridge.log</string>
    <key>StandardErrorPath</key><string>/tmp/codex-bridge.err</string>
</dict>
</plist>
```

加载：

```bash
launchctl load ~/Library/LaunchAgents/com.codex.bridge.plist
```

---

### 步骤 9：Shell 环境变量

追加到 `~/.zshrc`：

```bash
# codex-bridge — 旁路系统代理访问 localhost
export NO_PROXY="127.0.0.1,localhost${NO_PROXY:+,$NO_PROXY}"
export no_proxy="127.0.0.1,localhost${no_proxy:+,$no_proxy}"

alias proxy-start='launchctl load ~/Library/LaunchAgents/com.codex.bridge.plist'
alias proxy-stop='launchctl unload ~/Library/LaunchAgents/com.codex.bridge.plist'
alias proxy-status='curl -s http://127.0.0.1:4000/health'
```

---

### 步骤 10：验证

```bash
# 验证 1：代理健康检查
curl -s http://127.0.0.1:4000/health
# 期望: {"status":"ok","proxy":"codex-bridge","providers":["<PROVIDER>"]}

# 验证 2：端到端测试
codex -c model_provider=local_proxy exec "回复一个字：好"
# 期望输出包含 "好"
```

两步通过 → **告诉用户重启 Codex 桌面版**。

---

## 🔧 故障排查

用户反馈"连不上"时，按优先级排查。

### 🔴 1. 代理没跑？

```bash
curl -s http://127.0.0.1:4000/health
```

- **Connection refused** → 加载 launchd 或手动 `cd ~/.codex/codex-bridge && node proxy.mjs` 看报错
- **返回非 JSON / HTML** → 系统代理拦截 localhost。`env | grep -i proxy` 确认 NO_PROXY 含 127.0.0.1

### 🔴 2. Codex 绕过代理直连 OpenAI？

报错含 `api.openai.com` → model_provider 未生效。

检查：
- `config.toml` 的 `model_provider = "local_proxy"`
- CLI 加 `-c model_provider=local_proxy`
- 桌面版 CC Switch 中 local_proxy 为 Enabled

### 🟡 3. 502 Bad Gateway？

→ 检查 proxy.mjs 步骤 7 三处修复。

### 🟡 4. 模型列表无目标模型？

→ 检查 `model_catalog_json` 路径。检查 `proxy-models.json` 中 slug 是否与 .env 的 PROVIDER_MODELS 一致。

### 🟡 5. config.toml 被反复覆盖？

→ 用 CC Switch UI 添加 local_proxy 并启用。

### 🟢 6. 401 Unauthorized？

→ .env 的 PROXY_AUTH_KEY ≠ auth.json 的 OPENAI_API_KEY。对齐两者。

### 🟢 7. 代理报错「环境变量未设置」？

→ 检查 .env 文件变量名是否与 proxy.mjs 读取的变量名一致。不同版本 codex-bridge 可能用 DEEPSEEK_API_KEY / PROVIDER_API_KEY / OPENAI_API_KEY 等不同名称。以 proxy.mjs 源码为准。

---

## 模型切换

接入成功后用户可在对话中切换：

```
/model <模型slug>
```

---

## 参考

- codex-bridge: https://github.com/wujfeng712-ui/codex-bridge
- CC Switch: https://github.com/farion1231/cc-switch
