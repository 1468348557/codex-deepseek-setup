---
name: codex-deepseek-setup
description: 让 Codex 桌面版接入 DeepSeek V4 Pro。自动完成 codex-bridge 代理搭建、协议翻译配置、模型目录生成、路由修复、开机自启、端到端验证。用户只需提供 DeepSeek API Key，其余全部自动完成。触发词：接入 DeepSeek、换成 DeepSeek、Codex 用 DeepSeek、DeepSeek 连不上 等。
metadata:
  type: skill
  triggers:
    - deepseek
    - deepseek 接入
    - 接入 deepseek
    - Codex 用 deepseek
    - 换成 deepseek
    - 配置 deepseek
    - deepseek 连不上
    - deepseek 配置
    - codex deepseek
---

# Codex 桌面版 + DeepSeek V4 Pro 接入 Skill

> **你的角色：** 你是用户的 Codex 助手。用户想把 Codex 桌面版接入 DeepSeek V4 Pro / V4 Flash / Reasoner。本 Skill 教你自动完成全套配置——从搭代理到验证连通。用户提供 DeepSeek API Key，你干剩下的活。

## 问题背景

Codex 默认说 **Responses API**（OpenAI 私有协议），DeepSeek 说 **Chat Completions API**（行业标准）。两者协议不兼容，Codex 无法直接调用 DeepSeek。

解决方案：本机运行 `codex-bridge` 代理，实时双向翻译两套协议。

```
Codex 桌面版 ──Responses──▶ codex-bridge (:4000) ──Chat Completions──▶ api.deepseek.com
```

---

## 🤖 自动配置流程

严格按以下顺序执行。每步完成后向用户报告状态。

### 步骤 0：收集信息

执行 `whoami` 获取 macOS 用户名。

询问用户：
> 请提供 DeepSeek API Key（从 https://platform.deepseek.com/api_keys 获取）。

如果用户已提供，直接继续。

---

### 步骤 1：环境检查

```bash
node --version    # ≥ 18
whoami            # 确认用户名
```

不满足则告知用户安装方法。

---

### 步骤 2：克隆 codex-bridge

```bash
git clone https://github.com/wujfeng712-ui/codex-bridge.git ~/.codex/codex-bridge
```

目录已存在则跳过。

---

### 步骤 3：生成 .env

路径：`~/.codex/codex-bridge/.env`

先生成密钥：
```bash
openssl rand -hex 24
```

然后用文件工具写入以下内容（替换 `<用户名>` 和实际值）：

```
PROXY_AUTH_KEY=sk-proxy-local-<RANDOM_HEX>
DEEPSEEK_API_KEY=<用户提供的API_KEY>
DEEPSEEK_MODELS=deepseek-v4-pro,deepseek-v4-flash,deepseek-reasoner
DEFAULT_PROVIDER=deepseek
LOG_LEVEL=info
MODEL_CATALOG_PATH=/Users/<用户名>/.codex/proxy-models.json
```

**把 `PROXY_AUTH_KEY` 的值记下来，步骤 5 要用。**

---

### 步骤 4：配置 config.toml

读取 `~/.codex/config.toml`，合并以下内容（已有字段保留，不要覆盖无关项）：

```toml
model = "deepseek-v4-pro"
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

### 步骤 5：配置 auth.json

路径：`~/.codex/auth.json`

```json
{
  "auth_mode": "apikey",
  "OPENAI_API_KEY": "<步骤3的PROXY_AUTH_KEY>"
}
```

---

### 步骤 6：创建模型目录

路径：`~/.codex/proxy-models.json`

写入以下内容（Codex 用它识别可用模型）：

```json
{
  "models": [
    {
      "slug": "deepseek-v4-pro",
      "display_name": "DeepSeek v4 Pro",
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
      "priority": 0,
      "truncation_policy": { "mode": "tokens", "limit": 204800 },
      "supports_parallel_tool_calls": true
    },
    {
      "slug": "deepseek-v4-flash",
      "display_name": "DeepSeek v4 Flash",
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
      "priority": 1,
      "truncation_policy": { "mode": "tokens", "limit": 204800 },
      "supports_parallel_tool_calls": true
    },
    {
      "slug": "deepseek-reasoner",
      "display_name": "DeepSeek Reasoner",
      "default_reasoning_level": "high",
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
      "priority": 2,
      "truncation_policy": { "mode": "tokens", "limit": 204800 },
      "supports_parallel_tool_calls": true
    }
  ]
}
```

---

### 步骤 7：修复 proxy.mjs（最重要）

codex-bridge 的 `proxy.mjs` 有路由缺陷，必须修复才能正常工作。

打开 `~/.codex/codex-bridge/proxy.mjs`，做以下三处修改：

#### 7a：提取 URL 路径（防止 query string 干扰路由）

在 `http.createServer` 回调开头，`const authHeader =` 之前插入：

```js
const urlPath = req.url.split("?")[0];
```

然后**全局搜索替换**：所有路由匹配中的 `req.url` 改为 `urlPath`（约 4-6 处）。

#### 7b：/v1/models 免认证

找到 `isHealth` 定义行，改为：

```js
const isHealth = req.method === "GET" && (
  urlPath === "/health" || urlPath === "/" ||
  urlPath === "/v1/models" || urlPath === "/models"
);
```

#### 7c：模型列表响应增加 models 字段

找到 `/v1/models` 路由的 `sendJson` 调用，改为：

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

### 步骤 9：Shell 环境变量

追加到 `~/.zshrc`：

```bash
# Codex DeepSeek 代理 — 旁路系统代理
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
# 期望: {"status":"ok","proxy":"codex-bridge","providers":["deepseek"]}

# 验证 2：端到端测试
codex -c model_provider=local_proxy exec "回复一个字：好"
# 期望输出包含 "好"
```

两步都通过 → 告诉用户 **重启 Codex 桌面版** 即可使用 DeepSeek V4 Pro。

---

## 🔧 故障排查

用户反馈"连不上了"时，按优先级逐项排查。

### 1. 代理没跑？

```bash
curl -s http://127.0.0.1:4000/health
```

- **Connection refused** → 代理没启动。执行 launchctl load 或手动 `cd ~/.codex/codex-bridge && node proxy.mjs` 看报错。
- **返回非 JSON** → 系统代理拦截了 localhost。检查 `env | grep -i proxy`，确认 NO_PROXY 包含 `127.0.0.1,localhost`。

### 2. Codex 绕过代理直连 OpenAI？

Codex 报错中提到 `api.openai.com` → model_provider 没生效。

检查：
- `~/.codex/config.toml` 中 `model_provider = "local_proxy"`
- CLI 用户加 `-c model_provider=local_proxy` 参数
- 桌面版 CC Switch 中 local_proxy 为 Enabled

### 3. 502 Bad Gateway？

代理在跑但返回 502 → 路由匹配问题。

→ 检查 proxy.mjs 的步骤 7 三处修复是否全部完成。

### 4. 模型列表无 DeepSeek？

→ 检查 `model_catalog_json` 路径是否正确指向 `proxy-models.json`。

### 5. config.toml 被反复覆盖？

→ Codex 启动时会重写 config.toml。用 CC Switch 添加 local_proxy 供应商并启用，让它管理持久化。

### 6. 401 Unauthorized？

→ PROXY_AUTH_KEY 和 auth.json 的 OPENAI_API_KEY 不一致。从 `.env` 复制值到 `auth.json`，对齐两者。

---

## 📋 推理级别对照

| Codex 推理设置 | DeepSeek 对应 |
|---------------|-------------|
| 关闭推理 | thinking: disabled |
| 极低 / 低 | reasoning_effort: low |
| 中 | reasoning_effort: medium |
| 高 | reasoning_effort: high |
| 极高 | reasoning_effort: xhigh |

---

## 🔄 模型切换

用户可在 Codex 对话中：

```
/model deepseek-v4-pro      # 复杂编码任务
/model deepseek-v4-flash    # 轻量快速编辑
/model deepseek-reasoner    # 深度推理
```

---

## 📚 参考

- codex-bridge: https://github.com/wujfeng712-ui/codex-bridge
- CC Switch: https://github.com/farion1231/cc-switch
- DeepSeek API: https://platform.deepseek.com/api-docs
