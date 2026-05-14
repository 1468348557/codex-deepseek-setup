# 🐋 Codex × DeepSeek V4 Pro — 让你的 Codex 用上国产最强模型

<p align="center">
  <img src="https://img.shields.io/badge/Codex-Desktop-blue?style=for-the-badge&logo=openai" alt="Codex">
  <img src="https://img.shields.io/badge/DeepSeek-V4_Pro-purple?style=for-the-badge&logo=deepseek" alt="DeepSeek V4 Pro">
  <img src="https://img.shields.io/badge/Protocol-Responses_API_↔_Chat_Completions-green?style=for-the-badge" alt="Protocol Translation">
  <img src="https://img.shields.io/badge/Status-Stable-success?style=for-the-badge" alt="Status">
</p>

<p align="center">
  <b>一桥飞架南北，天堑变通途。</b><br>
  让 OpenAI Codex 无缝接入 DeepSeek V4 Pro，享受国产顶级大模型的代码能力。
</p>

---

## 🤔 为什么需要这个？

|  | OpenAI (Codex 默认) | DeepSeek V4 Pro |
|---|---|---|
| **价格** | 💰💰💰 | 💰 |
| **代码能力** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **中文理解** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Thinking Mode** | ✅ | ✅ |
| **协议兼容** | Responses API | Chat Completions ❌ |

**问题：** Codex 只说 Responses API，DeepSeek 只说 Chat Completions API——两种协议天生不兼容。

**解决：** `codex-bridge` 在本地做实时双向协议翻译，让你零感知切换后端。

---

## 🏗️ 架构

```
┌─────────┐    Responses API     ┌──────────────┐    Chat Completions    ┌──────────────┐
│  Codex  │ ──────────────────▶ │ codex-bridge │ ────────────────────▶ │  DeepSeek    │
│ (桌面版) │ ◀────────────────── │  :4000       │ ◀──────────────────── │  V4 Pro      │
└─────────┘   SSE / Tool Call     └──────────────┘   SSE / Reasoning     └──────────────┘
```

- 🔄 **流式 SSE** 双向翻译
- 🧠 **Thinking/Reasoning** 透传支持
- 🛠️ **Tool Calls** 完整转换
- 🚀 本地代理，延迟 < 5ms

---

## ⚡ 三分钟接入

### 前置要求

- Node.js ≥ 18
- Codex 已安装
- DeepSeek API Key（[免费获取](https://platform.deepseek.com/api_keys)）

### 1️⃣ 克隆代理

```bash
git clone https://github.com/wujfeng712-ui/codex-bridge.git ~/.codex/codex-bridge
```

### 2️⃣ 配置密钥

```bash
# ~/.codex/codex-bridge/.env
PROXY_AUTH_KEY=sk-proxy-local-$(openssl rand -hex 24)
DEEPSEEK_API_KEY=sk-你的DeepSeek密钥
DEEPSEEK_MODELS=deepseek-v4-pro,deepseek-v4-flash,deepseek-reasoner
DEFAULT_PROVIDER=deepseek
LOG_LEVEL=info
MODEL_CATALOG_PATH=~/.codex/proxy-models.json
```

### 3️⃣ 配置 Codex

```toml
# ~/.codex/config.toml
model = "deepseek-v4-pro"
model_provider = "local_proxy"
model_catalog_json = "/Users/你的用户名/.codex/proxy-models.json"

[model_providers.local_proxy]
name = "local_proxy"
base_url = "http://127.0.0.1:4000/v1"
wire_api = "responses"
requires_openai_auth = true
```

### 4️⃣ 启动！

重启 Codex，一切就绪 🎉

---

## 📖 完整文档

👉 [codex-deepseek-setup.md](./codex-deepseek-setup.md) — 包含完整配置、自动启动、故障排查等所有细节。

---

## 🧪 已验证环境

| 平台 | Codex 版本 | DeepSeek 模型 | 状态 |
|------|-----------|--------------|------|
| macOS Sequoia | 桌面版 | V4 Pro | ✅ |
| macOS Sequoia | 桌面版 | V4 Flash | ✅ |
| macOS Sequoia | 桌面版 | Reasoner | ✅ |

---

## 🔧 常见问题

<details>
<summary><b>代理无法启动？</b></summary>

```bash
# 检查端口占用
lsof -i :4000
# 手动启动测试
cd ~/.codex/codex-bridge && node proxy.mjs
```
</details>

<details>
<summary><b>Codex 连不上代理？</b></summary>

确认 `~/.codex/config.toml` 中 `base_url` 指向 `http://127.0.0.1:4000/v1`
</details>

<details>
<summary><b>如何切换回 OpenAI？</b></summary>

将 `config.toml` 中 `model_provider` 改回默认值，或直接注释掉相关配置。
</details>

---

## ⭐ Star History

如果这个项目帮到了你，请给一个 Star ⭐ 让更多人看到！

---

<p align="center">
  <sub>Made with ❤️ by the Codex community · 让 AI 编程没有边界</sub>
</p>
