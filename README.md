# 🐋 Codex × DeepSeek V4 Pro — 让你的 Codex 用上国产最强模型

<p align="center">
  <img src="https://img.shields.io/badge/Codex-Desktop-blue?style=for-the-badge&logo=openai" alt="Codex">
  <img src="https://img.shields.io/badge/DeepSeek-V4_Pro-purple?style=for-the-badge&logo=deepseek" alt="DeepSeek V4 Pro">
  <img src="https://img.shields.io/badge/Protocol-Responses_API_↔_Chat_Completions-green?style=for-the-badge" alt="Protocol Translation">
  <img src="https://img.shields.io/badge/Auto_Setup-Skill_Powered-orange?style=for-the-badge" alt="Auto Setup">
</p>

<p align="center">
  <b>不是一份文档，而是一个能替你干活儿 Codex Skill。</b><br>
  把 DeepSeek V4 Pro 接入 Codex 的配置、代理、修复——<b>全自动</b>完成。
</p>

---

## 🤯 一句话说清楚

Codex 说 Responses API，DeepSeek 说 Chat Completions API，两者母语不通。

本项目提供两样东西：
- 🧠 **一个 Local Proxy**：实时翻译两种协议（`codex-bridge`）
- 🤖 **一个 Codex Skill**：让 Codex 自己帮你配好一切

**结果：你对 Codex 说「帮我接入 DeepSeek」，Codex 读这份 skill，然后自己搞定全部。**

---

## 🪄 使用方式（两种）

### 方式一：Skill 自动配置（推荐 ⭐）

1. 将 `codex-deepseek-setup.md` 安装为 Codex Skill：
   ```bash
   mkdir -p ~/.codex/skills/codex-deepseek
   cp codex-deepseek-setup.md ~/.codex/skills/codex-deepseek/SKILL.md
   ```

2. 打开 Codex 对话框，输入：
   ```
   帮我接入 DeepSeek
   ```

3. Codex 读 skill → 自动检查环境 → 克隆代理 → 写配置 → 修复路由 → 启动服务 → 验证连通。

   **你唯一需要做的：提供一个 DeepSeek API Key。**

### 方式二：手动参照文档

打开 [codex-deepseek-setup.md](./codex-deepseek-setup.md)，照着十步走。

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

## 📖 Skill 能自动完成的事

| 步骤 | 内容 | 方式 |
|------|------|------|
| 环境检查 | Node.js / Codex 版本 | 执行命令 |
| 克隆代理 | codex-bridge 到本地 | 执行命令 |
| 生成密钥 | PROXY_AUTH_KEY | 自动生成 |
| 写入配置 | .env / config.toml / auth.json | 文件编辑 |
| 模型目录 | proxy-models.json | 文件创建 |
| 修复代理 | proxy.mjs 路由补丁 | 文件编辑 |
| 开机自启 | macOS launchd plist | 文件创建 + 命令 |
| 环境变量 | NO_PROXY + 别名 | Shell 配置 |
| 验证连通 | 代理 + 端到端 | 执行命令 |
| 故障排查 | 6 类常见问题 | 自动诊断 |

---

## 🧪 已验证环境

| 平台 | Codex 版本 | DeepSeek 模型 | 状态 |
|------|-----------|--------------|------|
| macOS Sequoia | 桌面版 | V4 Pro | ✅ |
| macOS Sequoia | 桌面版 | V4 Flash | ✅ |
| macOS Sequoia | 桌面版 | Reasoner | ✅ |

---

## 📦 文件说明

```
.
├── README.md                    # 你正在看的这个
└── codex-deepseek-setup.md      # Codex Skill 定义（核心文件）
```

---

## 🔗 依赖项目

- [codex-bridge](https://github.com/wujfeng712-ui/codex-bridge) — 本地协议代理
- [CC Switch](https://github.com/farion1231/cc-switch) — Codex 供应商管理（推荐）

---

## ⭐ Star History

如果这个项目省了你半小时折腾配置的时间，给个 Star ⭐ 让更多人少走弯路！

---

<p align="center">
  <sub>Made with ❤️ by the Codex community · 让 AI 编程没有语言障碍</sub>
</p>
