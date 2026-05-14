---
name: codex-deepseek-setup
description: Configure Codex (desktop/CLI) to use DeepSeek V4 Pro via codex-bridge proxy. Handles protocol translation (Responses API ↔ Chat Completions), model catalog setup, proxy auto-start, and troubleshooting. Use when user wants to connect Codex to DeepSeek, switch Codex backend to DeepSeek, or fix Codex-DeepSeek integration issues.
metadata:
  type: skill
  triggers:
    - codex deepseek
    - codex 接入 deepseek
    - codex-bridge
    - codex deepseek v4
    - codex 换 deepseek
    - codex 配置 deepseek
    - deepseek 配置
    - deepseek 接入
---

# Codex + DeepSeek V4 Pro 接入

> **你的角色：** 你是用户的 Codex 助手。本 skill 指导你**自动**帮用户完成 DeepSeek 接入的每一步——执行命令、编辑配置、验证连通性。用户只需提供 DeepSeek API Key，其余全部由你完成。

## 架构

```
Codex ──Responses API──▶ codex-bridge (:4000) ──Chat Completions──▶ api.deepseek.com
```

Codex 只说 **Responses API**，DeepSeek 只说 **Chat Completions API**，协议不兼容。codex-bridge 做本地双向翻译（SSE 流式、工具调用、thinking-mode）。

---

## 🤖 自动接入流程

按以下顺序执行，每步完成后报告状态给用户。

### 步骤 0：收集信息

先询问用户两个信息：
1. **DeepSeek API Key**（从 https://platform.deepseek.com/api_keys 获取）
2. **用户名**（macOS 用户目录名，默认执行 `whoami` 推断）

执行 `whoami` 获取用户名，与用户确认。如果用户已提供 API Key，直接继续。

---

### 步骤 1：环境检查

依次检查并报告结果：

```bash
node --version          # 需 ≥ 18
codex --version         # 确认已安装
whoami                  # 记录用户名，后续配置要用
```

如果依赖缺失，告诉用户安装方法再继续。

---

### 步骤 2：克隆 codex-bridge

```bash
git clone https://github.com/wujfeng712-ui/codex-bridge.git ~/.codex/codex-bridge
```

如果目标目录已存在，跳过克隆，直接进入下一步。

---

### 步骤 3：生成 .env 配置

**用文件写入工具**创建 `~/.codex/codex-bridge/.env`，内容如下（将 `<用户名>` 和 `<API_KEY>` 替换为实际值）：

```bash
PROXY_AUTH_KEY=sk-proxy-local-<RANDOM_HEX_24>
DEEPSEEK_API_KEY=<用户提供的API_KEY>
DEEPSEEK_MODELS=deepseek-v4-pro,deepseek-v4-flash,deepseek-reasoner
DEFAULT_PROVIDER=deepseek
LOG_LEVEL=info
MODEL_CATALOG_PATH=/Users/<用户名>/.codex/proxy-models.json
```

`<RANDOM_HEX_24>` 用 `openssl rand -hex 24` 生成。记下 `PROXY_AUTH_KEY` 的值，步骤 5 要用。

---

### 步骤 4：配置 Codex config.toml

**读取或创建** `~/.codex/config.toml`，确保包含以下内容（合并到已有配置，不要覆盖无关字段）：

```toml
cli_auth_credentials_store = "file"
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

**创建** `~/.codex/auth.json`（注意 `OPENAI_API_KEY` 的值来自步骤 3 的 `PROXY_AUTH_KEY`）：

```json
{
  "auth_mode": "apikey",
  "OPENAI_API_KEY": "sk-proxy-local-<步骤3的RANDOM_HEX>"
}
```

---

### 步骤 6：创建模型元数据 catalog

**创建** `~/.codex/proxy-models.json`：

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
    }
  ]
}
```

---

### 步骤 7：修复 proxy.mjs 路由（关键！）

codex-bridge 的 `proxy.mjs` 有两处必须修复，否则 Codex 请求无法到达。

**打开** `~/.codex/codex-bridge/proxy.mjs`，做以下三处修改：

#### 7a：URL 路径解析

在 `http.createServer` 回调开头（`const authHeader = ...` 之前），添加：

```js
const urlPath = req.url.split("?")[0];
```

然后将文件中**所有路由匹配**的 `req.url` 替换为 `urlPath`（约 4-6 处，如 `req.url === "/v1/models"` 改为 `urlPath === "/v1/models"`）。

#### 7b：/v1/models 免认证

找到 auth gate 的 `isHealth` 行，改为：

```js
const isHealth = req.method === "GET" && (
  urlPath === "/health" || urlPath === "/" ||
  urlPath === "/v1/models" || urlPath === "/models"
);
```

#### 7c：模型列表响应增加 models 字段

找到 `/v1/models` 的 `sendJson` 响应，改为：

```js
sendJson(res, 200, {
  object: "list",
  data: modelCatalog,
  models: modelCatalog,
  default_provider: getFallbackProvider(),
});
```

---

### 步骤 8：配置代理开机自启（macOS）

**创建** `~/Library/LaunchAgents/com.codex.bridge.plist`：

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

然后执行：
```bash
launchctl load ~/Library/LaunchAgents/com.codex.bridge.plist
```

---

### 步骤 9：Shell 环境变量

追加到用户的 shell 配置文件（检查是 `~/.zshrc` 还是 `~/.bashrc`）：

```bash
# Codex DeepSeek 代理：绕过系统代理访问 localhost
export NO_PROXY="127.0.0.1,localhost${NO_PROXY:+,$NO_PROXY}"
export no_proxy="127.0.0.1,localhost${no_proxy:+,$no_proxy}"

# 快捷命令
alias proxy-start='launchctl load ~/Library/LaunchAgents/com.codex.bridge.plist'
alias proxy-stop='launchctl unload ~/Library/LaunchAgents/com.codex.bridge.plist'
alias proxy-status='curl -s http://127.0.0.1:4000/health'
```

---

### 步骤 10：验证

全部配置完成后，执行验证：

```bash
# 验证1：代理健康检查
curl -s http://127.0.0.1:4000/health
# 预期: {"status":"ok","proxy":"codex-bridge","providers":["deepseek"]}

# 验证2：端到端测试
codex -c model_provider=local_proxy exec "回复一个字：好"
# 预期输出包含 "好"
```

两步都通过后，告诉用户**重启 Codex 桌面版**即可使用。

---

## 🔧 故障排查（Codex 自动诊断）

当用户反馈接入有问题时，按以下顺序排查：

### 排查 1：代理是否在运行？
```bash
curl -s http://127.0.0.1:4000/health
```
- **无响应 / Connection refused** → 代理没启动。执行 `launchctl load ~/Library/LaunchAgents/com.codex.bridge.plist`，或手动 `cd ~/.codex/codex-bridge && node proxy.mjs` 查看报错。
- **响应非 JSON** → 可能是系统代理拦截了 localhost。检查 `env | grep -i proxy`，确保 `NO_PROXY` 包含 `127.0.0.1,localhost`。

### 排查 2：Codex 直连 api.openai.com
**症状：** Codex 报错指向 `api.openai.com` 而非 `127.0.0.1:4000`。
**原因：** `model_provider` 未生效，或 CC Switch 未启用。
**修复：**
1. 检查 `~/.codex/config.toml` 中 `model_provider = "local_proxy"` 是否存在
2. 桌面版用户需确认 CC Switch 中 local_proxy 处于 Enabled 状态
3. CLI 用户加 `-c model_provider=local_proxy` 参数

### 排查 3：502 Bad Gateway
**原因：** 系统代理拦截 localhost + 代理路由不匹配。
**修复：**
1. 检查 `NO_PROXY` 环境变量已设置
2. 检查 proxy.mjs 是否已执行步骤 7 的三处修复

### 排查 4：模型列表无 DeepSeek
**原因：** Codex 默认从 OpenAI marketplace 拉模型。
**修复：** 检查 `config.toml` 中 `model_catalog_json` 路径正确指向 `proxy-models.json`。

### 排查 5：config.toml 被反复覆盖
**原因：** Codex/CC Switch 启动时会重写 config.toml。
**修复：** 通过 CC Switch 添加 local_proxy 供应商并启用，让 CC Switch 管理配置持久化。

### 排查 6：401 Unauthorized
**原因：** 代理的 `PROXY_AUTH_KEY` 与 Codex 的 `auth.json` 中 `OPENAI_API_KEY` 不一致。
**修复：** 确保两者值相同。重新生成 `.env` 并同步更新 `auth.json`。

---

## 📋 推理级别对照

| Codex 设置 | DeepSeek 行为 |
|-----------|--------------|
| 关闭推理 | thinking: disabled |
| 极低 / 低 | reasoning_effort: low |
| 中 | reasoning_effort: medium |
| 高 | reasoning_effort: high |
| 极高 | reasoning_effort: xhigh |

---

## 🔄 模型切换

告诉用户可在 Codex 对话中切换模型：
```
/model deepseek-v4-pro      # 复杂编码
/model deepseek-v4-flash    # 轻量编辑
/model deepseek-reasoner    # 深度推理
```

---

## 📚 参考

- codex-bridge: https://github.com/wujfeng712-ui/codex-bridge
- CC Switch: https://github.com/farion1231/cc-switch
- DeepSeek API: https://platform.deepseek.com/api-docs
