---
name: codex-model-integration
description: 通用的 Codex 第三方大模型接入 Skill。支持 DeepSeek、Qwen、Claude、Gemini、Mistral、Groq 等任意 Chat Completions 兼容模型。教你如何为任何模型搭建本地代理、配置 Codex 协议翻译、写入模型目录、自动诊断常见问题。触发词包括"接入 XXX 模型"、"换 XXX 模型"、"Codex 用 XXX"、"配模型"、"模型连不上"等。
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
    - claude
    - gemini
    - mistral
    - groq
    - 通义千问
    - 千问
    - codex 模型
    - 第三方模型
---

# Codex 第三方大模型通用接入

> **你的角色：** 你是用户的 Codex 助手。用户想把 Codex 接入某个第三方大模型（DeepSeek/Qwen/Claude/Gemini/...任意 Chat Completions 兼容 API）。本 skill 教你怎么一步步自动完成——从搭代理、写配置、到端到端验证。

## 核心原理

Codex 默认只说 **Responses API**（OpenAI 专有协议），而几乎所有第三方模型只说 **Chat Completions API**（行业事实标准）。

解决方案：在本地搭一个轻量代理 `codex-bridge`，实时做 Responses ↔ Chat Completions 双向翻译。

```
Codex ──Responses API──▶ codex-bridge (:4000) ──Chat Completions──▶ 任意模型 API
```

不管目标模型是什么（DeepSeek、千问、Claude、Gemini……），只要它提供 OpenAI 兼容的 Chat Completions 端点，就能接。

---

## 🤖 自动接入流程

### 第 0 步：搞清楚用户要接什么

问用户三件事：

1. **目标模型 / 供应商**（如 deepseek / qwen / claude / gemini / groq / mistral）
2. **API 地址**（base URL，如 `https://api.deepseek.com/v1`、`https://dashscope.aliyuncs.com/compatible-mode/v1`）
3. **API Key**（去哪里申请也告诉用户）

如果用户不清楚 API 地址，根据常见供应商推断：

| 用户说 | 大概率 base URL |
|--------|----------------|
| DeepSeek | `https://api.deepseek.com/v1` |
| 千问 / Qwen / 通义千问 / DashScope | `https://dashscope.aliyuncs.com/compatible-mode/v1` |
| Claude / Anthropic | 不走通用代理（需 Anthropic Messages API） |
| Gemini / Google | `https://generativelanguage.googleapis.com/v1beta`（需额外适配） |
| Groq | `https://api.groq.com/openai/v1` |
| Mistral | `https://api.mistral.ai/v1` |
| Together AI | `https://api.together.xyz/v1` |
| SiliconFlow / 硅基流动 | `https://api.siliconflow.cn/v1` |
| 智谱 / GLM / Zhipu | `https://open.bigmodel.cn/api/paas/v4` |
| Moonshot / Kimi | `https://api.moonshot.cn/v1` |
| 百川 / Baichuan | `https://api.baichuan-ai.com/v1` |
| 零一万物 / Yi | `https://api.lingyiwanwu.com/v1` |
| DeepSeek / 火山引擎 | `https://ark.cn-beijing.volces.com/api/v3` |
| OpenRouter | `https://openrouter.ai/api/v1` |

如果不在上表，告诉用户："给模型名称 + Chat Completions base URL + API Key 即可，我来搞定剩下的。"

执行 `whoami` 取用户名，后续配置路径要用。

---

### 第 1 步：环境检查

```bash
node --version    # ≥ 18 才行
whoami            # 记下用户名
```

---

### 第 2 步：克隆 codex-bridge

```bash
git clone https://github.com/wujfeng712-ui/codex-bridge.git ~/.codex/codex-bridge
```

已存在则跳过。

---

### 第 3 步：写入 .env

路径：`~/.codex/codex-bridge/.env`

```bash
PROXY_AUTH_KEY=sk-proxy-local-$(openssl rand -hex 24  > /dev/null 2>&1 && openssl rand -hex 24 || echo "replace-with-random-hex")
# 如果 openssl 不可用，手动生成 48 位随机十六进制字符串
```

完整内容：

```
PROXY_AUTH_KEY=<生成的值>
PROVIDER_API_KEY=<用户的API_KEY>
PROVIDER_BASE_URL=<用户的BASE_URL>
PROVIDER_MODELS=<模型名列表，逗号分隔>
DEFAULT_PROVIDER=<供应商简称>
LOG_LEVEL=info
MODEL_CATALOG_PATH=/Users/<用户名>/.codex/proxy-models.json
```

**重要：记下 `PROXY_AUTH_KEY` 的值，后续配置 auth.json 要用。**

---

### 第 4 步：写入 config.toml

**合并到** `~/.codex/config.toml`（已有字段保留，只增改这几个）：

```toml
model = "<默认模型slug>"
model_provider = "local_proxy"
model_catalog_json = "/Users/<用户名>/.codex/proxy-models.json"

[model_providers.local_proxy]
name = "local_proxy"
base_url = "http://127.0.0.1:4000/v1"
wire_api = "responses"
requires_openai_auth = true

[projects."/Users/<用户名>"]
trust_level = "trusted"
```

---

### 第 5 步：写入 auth.json

路径：`~/.codex/auth.json`

```json
{
  "auth_mode": "apikey",
  "OPENAI_API_KEY": "<第3步的PROXY_AUTH_KEY>"
}
```

---

### 第 6 步：生成模型目录

路径：`~/.codex/proxy-models.json`

根据用户要接的模型，生成条目。每条格式：

```json
{
  "slug": "<模型slug，英文小写+连字符>",
  "display_name": "<显示名>",
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
  "priority": <0到N排序>,
  "truncation_policy": { "mode": "tokens", "limit": 204800 },
  "supports_parallel_tool_calls": true
}
```

至少放用户要的那个模型。鼓励多列几个该供应商的模型让用户选。

**常见模型预填参考：**

| 供应商 | 常见模型 slug |
|--------|-------------|
| DeepSeek | `deepseek-chat`, `deepseek-reasoner` |
| 千问 | `qwen-plus`, `qwen-max`, `qwen-turbo` |
| Groq | `llama-3.3-70b-versatile`, `mixtral-8x7b-32768` |
| Mistral | `mistral-large-latest`, `mistral-small-latest` |
| SiliconFlow | `Qwen/Qwen2.5-72B-Instruct`, `deepseek-ai/DeepSeek-V3` |
| 智谱 | `glm-4-plus`, `glm-4-flash` |
| Moonshot | `moonshot-v1-8k`, `moonshot-v1-32k` |
| OpenRouter | 任意模型 slug，如 `openai/gpt-4o`, `anthropic/claude-3.5-sonnet` |

---

### 第 7 步：修 proxy.mjs（关键！）

**先确认** codex-bridge 的 `proxy.mjs` 是否需要修。

codex-bridge 原始代码可能有两类问题：

#### 问题 A：URL query string 干扰路由

`proxy.mjs` 用 `req.url` 直接匹配路由，但 Codex 请求带 `?query=params` 会导致匹配失败。

**打开** `~/.codex/codex-bridge/proxy.mjs`

找到 `http.createServer` 回调开头，在 `const authHeader =` 之前插入：

```js
const urlPath = req.url.split("?")[0];
```

然后将文件中**所有路由判断**的 `req.url ===` 或 `req.url.startsWith(` 改为 `urlPath ===` / `urlPath.startsWith(`。

#### 问题 B：/v1/models 需要免认证 + 正确响应格式

找 auth gate 区域（含 `isHealth` 的那几行），确保 `/v1/models` 和 `/models` 加入免认证列表：

```js
const isHealth = req.method === "GET" && (
  urlPath === "/health" || urlPath === "/" ||
  urlPath === "/v1/models" || urlPath === "/models"
);
```

找到 `/v1/models` 路由的 `sendJson` 调用，改响应为：

```js
sendJson(res, 200, {
  object: "list",
  data: modelCatalog,
  models: modelCatalog,
  default_provider: getFallbackProvider(),
});
```

#### 问题 C：多供应商支持检查

如果 `proxy.mjs` 是单供应商版本（只读 `DEEPSEEK_API_KEY`），需要让它支持多供应商。检查 `.env` 中是否有 `PROVIDER_API_KEY` 和 `PROVIDER_BASE_URL` 字段，如果没有，告诉用户需要升级 codex-bridge 或修改 proxy.mjs 用通用的 `PROVIDER_*` 环境变量。

**总之：修改完 proxy.mjs 后，确保所有需要的路由正确匹配、模型列表接口返回正确格式。**

---

### 第 8 步：macOS 开机自启

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
        <string>--env-file=/Users/<用户名>/.codex/codex-bridge/.env</string>
        <string>/Users/<用户名>/.codex/codex-bridge/proxy.mjs</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/Users/<用户名>/.codex/codex-bridge</string>
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

### 第 9 步：Shell 环境变量

追加到用户 shell 配置（`~/.zshrc`）：

```bash
# codex-bridge 旁路系统代理
export NO_PROXY="127.0.0.1,localhost${NO_PROXY:+,$NO_PROXY}"
export no_proxy="127.0.0.1,localhost${no_proxy:+,$no_proxy}"

alias proxy-start='launchctl load ~/Library/LaunchAgents/com.codex.bridge.plist'
alias proxy-stop='launchctl unload ~/Library/LaunchAgents/com.codex.bridge.plist'
alias proxy-status='curl -s http://127.0.0.1:4000/health'
```

---

### 第 10 步：验证

```bash
# 1. 代理是否活着
curl -s http://127.0.0.1:4000/health
# 期望: {"status":"ok"...}

# 2. Codex 端到端
codex -c model_provider=local_proxy exec "回复一个字：好"
# 期望输出包含"好"
```

两步都通过 = 接入成功。告诉用户重启 Codex 桌面版使生效。

---

## 🔧 故障排查（遇到问题喂给 Codex）

当用户反馈不工作时，按此优先级排查。**每一步都要给用户解释你在做什么、为什么。**

### 🔴 优先级 1：代理活着吗？

```bash
curl -s http://127.0.0.1:4000/health
```

- **Connection refused** → `launchctl load` 或手动 `node proxy.mjs` 看报错
- **返回非 JSON / HTML** → 系统代理拦截了 localhost。检查 proxy 环境变量，加 `NO_PROXY`

### 🔴 优先级 2：Codex 是不是绕过代理直连 OpenAI 了？

Codex 报错提到 `api.openai.com` = model_provider 没生效。

检查：
1. `~/.codex/config.toml` 里 `model_provider = "local_proxy"`
2. CLI 加 `-c model_provider=local_proxy`
3. 桌面版 CC Switch 中 `local_proxy` 是否为 Enabled

### 🟡 优先级 3：502 / 路由不匹配

代理在跑但返回 502 = URL 匹配问题。

→ 检查 proxy.mjs 步骤 7 的三处修复是否都做了。

### 🟡 优先级 4：模型列表没有目标模型

→ Codex 默认去 OpenAI marketplace 拉模型。检查 `model_catalog_json` 路径正确。

### 🟡 优先级 5：config.toml 被 Codex/CC Switch 反复覆盖

→ 通过 CC Switch UI 添加 `local_proxy` 供应商并启用，让它管理。

### 🟢 优先级 6：401 Unauthorized

→ `PROXY_AUTH_KEY` 与 `auth.json` 里的 `OPENAI_API_KEY` 对不上。从 `.env` 复制到 `auth.json`，值要完全一致。

---

## 📋 推理级别对照（所有模型通用）

| Codex 推理设置 | API 端行为 |
|---------------|-----------|
| 关闭推理 | `thinking: disabled` / 不发送 reasoning |
| 极低 / 低 | `reasoning_effort: low` |
| 中 | `reasoning_effort: medium` |
| 高 | `reasoning_effort: high` |
| 极高 | `reasoning_effort: xhigh` |

---

## 🔄 模型切换

用户可在对话中切换：
```
/model <模型slug>       # 切换到指定模型
```

---

## 📚 参考

- codex-bridge: https://github.com/wujfeng712-ui/codex-bridge
- CC Switch: https://github.com/farion1231/cc-switch
