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
---

# Codex + DeepSeek V4 Pro 接入

让 Codex（桌面版和 CLI）通过本地代理 `codex-bridge` 连接 DeepSeek V4 Pro。

## 架构

```
Codex ──Responses API──▶ codex-bridge (:4000) ──Chat Completions──▶ api.deepseek.com
```

Codex 只说 **Responses API**，DeepSeek 只说 **Chat Completions API**，两者协议不兼容。codex-bridge 在本机做双向协议翻译（包括流式 SSE、工具调用、thinking-mode 往返）。

## 前置检查

```bash
node --version        # ≥ 18
codex --version       # 确认已安装
ls ~/.codex/codex-bridge/proxy.mjs  # 代理是否已克隆
```

## 一、安装 codex-bridge

```bash
git clone https://github.com/wujfeng712-ui/codex-bridge.git ~/.codex/codex-bridge
```

## 二、配置代理 .env

文件路径：`~/.codex/codex-bridge/.env`

```bash
PROXY_AUTH_KEY=sk-proxy-local-$(openssl rand -hex 24)
DEEPSEEK_API_KEY=sk-<DeepSeek密钥>
DEEPSEEK_MODELS=deepseek-v4-pro,deepseek-v4-flash,deepseek-reasoner
DEFAULT_PROVIDER=deepseek
LOG_LEVEL=info
MODEL_CATALOG_PATH=/Users/<用户名>/.codex/proxy-models.json
```

**注意：** 用户需要从 https://platform.deepseek.com/api_keys 获取 DeepSeek API Key。

## 三、配置 Codex config.toml

文件路径：`~/.codex/config.toml`

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

## 四、配置 auth.json

文件路径：`~/.codex/auth.json`

```json
{
  "auth_mode": "apikey",
  "OPENAI_API_KEY": "sk-proxy-local-<和.env中PROXY_AUTH_KEY一致>"
}
```

## 五、模型元数据 catalog

文件路径：`~/.codex/proxy-models.json`

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

## 六、修复代理路由（必须）

codex-bridge 的 `proxy.mjs` 需要两处修复，否则 Codex 请求无法到达代理：

### 修复1：URL 路径解析

Codex 发送 `/v1/models?client_version=0.130.0` 带查询参数，原始代码用 `req.url === "/v1/models"` 精确匹配会失败。

在 `http.createServer` 回调开头添加：
```js
const urlPath = req.url.split("?")[0];
```

然后将所有路由匹配中的 `req.url` 替换为 `urlPath`。

### 修复2：/v1/models 免认证

Codex 刷新模型列表时不带 auth header。修改 auth gate：
```js
const isHealth = req.method === "GET" && (
  urlPath === "/health" || urlPath === "/" ||
  urlPath === "/v1/models" || urlPath === "/models"
);
```

### 修复3：模型列表响应增加 `models` 字段

Codex 期望响应中有 `models` 字段：
```js
sendJson(res, 200, {
  object: "list",
  data: modelCatalog,
  models: modelCatalog,  // 新增
  default_provider: getFallbackProvider(),
});
```

## 七、代理开机自启（macOS launchd）

plist 文件：`~/Library/LaunchAgents/com.codex.bridge.plist`

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

```bash
launchctl load ~/Library/LaunchAgents/com.codex.bridge.plist
```

## 八、Shell 环境变量

追加到 `~/.zshrc`：

```bash
# 必须：绕过系统代理访问 localhost
export NO_PROXY="127.0.0.1,localhost${NO_PROXY:+,$NO_PROXY}"
export no_proxy="127.0.0.1,localhost${no_proxy:+,$no_proxy}"

# 别名
alias proxy-start='launchctl load ~/Library/LaunchAgents/com.codex.bridge.plist'
alias proxy-stop='launchctl unload ~/Library/LaunchAgents/com.codex.bridge.plist'
alias proxy-status='curl -s http://127.0.0.1:4000/health'
```

## 验证

```bash
# 1. 代理健康检查
curl -s http://127.0.0.1:4000/health
# 预期: {"status":"ok","proxy":"codex-bridge","providers":["deepseek"],...}

# 2. 端到端测试
codex -c model_provider=local_proxy exec "回复一个字：好"
# 预期输出中包含 "好"
```

## 坑与解决方案

### 坑1：Codex 直连 api.openai.com 不经过代理
**原因：** 系统 HTTP 代理（Clash/Surge）拦截了 localhost 请求。  
**修复：** 必须设置 `NO_PROXY="127.0.0.1,localhost"`。桌面版需配合 CC Switch 启用供应商。

### 坑2：502 Bad Gateway
**原因：** 系统代理拦截 + 代理 URL 路由不匹配查询参数。  
**修复：** 设置 NO_PROXY + 修复 proxy.mjs 的 urlPath 路由。

### 坑3：模型列表不显示 deepseek
**原因：** Codex 从 OpenAI marketplace 拉取模型，不认 deepseek。  
**修复：** 创建 proxy-models.json，设置 `model_catalog_json` 指向它。

### 坑4：config.toml 被反复覆盖
**原因：** Codex/CC Switch 启动时重写 config.toml。  
**修复：** 用 CC Switch 添加并启用供应商，CC Switch 写入的配置能持久化。

### 坑5：CLI 模式下 model_provider 不生效
**原因：** Codex CLI v0.130.0 不读取 config.toml 的 model_provider。  
**修复：** CLI 使用 `-c model_provider=local_proxy` 参数；桌面版用 CC Switch。

### 坑6：401 Unauthorized on api.openai.com
**原因：** model_provider 未生效，Codex 仍用默认 OpenAI 供应商。  
**修复：** 检查 CC Switch 是否 Enabled，重启 Codex。

## 推理级别对照

| Codex 设置 | DeepSeek 行为 |
|-----------|--------------|
| 关闭推理 | thinking: disabled |
| 极低 / 低 | reasoning_effort: low |
| 中 | reasoning_effort: medium |
| 高 | reasoning_effort: high |
| 极高 | reasoning_effort: xhigh |

## 模型切换

在 Codex 对话中输入：
```
/model deepseek-v4-pro      # 复杂编码
/model deepseek-v4-flash    # 轻量编辑
/model deepseek-reasoner    # 深度推理
```

## 参考

- codex-bridge: https://github.com/wujfeng712-ui/codex-bridge
- CC Switch: https://github.com/farion1231/cc-switch
- DeepSeek API: https://platform.deepseek.com/api-docs
