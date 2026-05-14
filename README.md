# 🌍 Codex 万能模型接入 Skill — 一个 Skill，接入任意大模型

<p align="center">
  <img src="https://img.shields.io/badge/Codex-Desktop-blue?style=for-the-badge&logo=openai" alt="Codex">
  <img src="https://img.shields.io/badge/Any_Model-DeepSeek|Qwen|Groq|Mistral|GLM|Kimi|...-purple?style=for-the-badge" alt="Any Model">
  <img src="https://img.shields.io/badge/Protocol-Responses_↔_Chat_Completions-green?style=for-the-badge" alt="Protocol Translation">
  <img src="https://img.shields.io/badge/Setup-Skill_Autopilot-orange?style=for-the-badge" alt="Auto Setup">
</p>

<p align="center">
  <b>安装这个 Skill → 说「接入 XXX 模型」→ Codex 自己配好一切。</b><br>
  不管 DeepSeek、千问、Groq 还是 Kimi——原理一样，Skill 替你干活。
</p>

---

## 🤔 问题

Codex 桌面版默认只能用 OpenAI 模型。但市面上一堆好用的模型——DeepSeek V4 便宜推理强、千问中文好、Groq 速度飞快——都想用怎么办？

**根本问题：Codex 说 Responses API，所有第三方模型说 Chat Completions API，协议不兼容。**

---

## ✨ 我做了什么

写了一个**通用** Codex Skill。

它不绑定任何特定模型。内置了 DeepSeek / 千问 / Groq / Mistral / 智谱 / Kimi / 百川 / SiliconFlow / OpenRouter 等常见供应商的 Base URL 和模型列表。

**碰上不在表的模型？告诉 Codex Base URL + API Key 就行，其余全自动。**

---

## ⚡ 三秒上手

```bash
# 1. 安装
mkdir -p ~/.codex/skills/model-integration
cp codex-deepseek-setup.md ~/.codex/skills/model-integration/SKILL.md

# 2. 打开 Codex，随便说
```

> "接入 DeepSeek"
>
> "帮我换成通义千问"
>
> "接入 Groq 的 llama 模型"
>
> "用智谱 GLM-4"

**Codex 会问你 API Key，然后自动配完十步。**

---

## 🎯 模型覆盖

| 供应商 | 自带的模型 | 你没列出的？ |
|--------|----------|------------|
| DeepSeek | chat / reasoner | 给 Base URL 就行 |
| 通义千问 Qwen | plus / max / turbo | 给 Base URL 就行 |
| Groq | llama-3.3-70b / mixtral | 给 Base URL 就行 |
| Mistral | large / small | 给 Base URL 就行 |
| 智谱 GLM | glm-4-plus / flash | 给 Base URL 就行 |
| Moonshot Kimi | moonshot-v1 | 给 Base URL 就行 |
| 百川 Baichuan | Baichuan4 | 给 Base URL 就行 |
| SiliconFlow | Qwen2.5-72B / DeepSeek-V3 | 给 Base URL 就行 |
| OpenRouter | 任意模型 | 给完整路径 |

> **核心逻辑：所有 OpenAI 兼容 Chat Completions 的模型都能接。Skill 里预填只是帮你省事。**

---

## 🏗️ 原理

```
┌─────────┐  Responses API   ┌──────────────┐  Chat Completions   ┌──────────────┐
│  Codex  │ ────────────────▶│ codex-bridge │ ──────────────────▶│  任意模型 API │
│ 桌面版   │ ◀────────────────│  :4000       │ ◀──────────────────│              │
└─────────┘  SSE / Tool Call  └──────────────┘  SSE / Reasoning   └──────────────┘
```

**不同模型只是改 Base URL + API Key。协议翻译层通用。**

---

## 🤖 Skill 自动完成的 10 步

| # | 做什么 | 关键 |
|---|--------|------|
| 0 | 识别目标模型 | 匹配内置表 or 用户给 URL |
| 1 | 环境检查 | node ≥ 18 |
| 2 | 克隆代理 | git clone codex-bridge |
| 3 | 生成密钥 + .env | 自动 openssl + 通用变量名 |
| 4 | config.toml | 合并到已有配置 |
| 5 | auth.json | 对齐代理密钥 |
| 6 | proxy-models.json | 按供应商生成模型目录 |
| 7 | 🔑 修复 proxy.mjs | URL 解析 + 免认证 + models 字段 |
| 8 | 开机自启 | macOS launchd plist |
| 9 | Shell 别名 | .zshrc NO_PROXY + 别名 |
| 10 | 端到端验证 | curl health + codex exec |

---

## 🔧 故障自愈

出问题别翻文档，直接说：

> "模型连不上了"

Skill 内置 7 级诊断清单，Codex 逐项查：代理死活 → 是否绕过直连 OpenAI → 路由匹配 → 模型列表 → 配置覆盖 → 认证 → 环境变量名兼容。

---

## 📦 文件

```
.
├── README.md                    # 人看
└── codex-deepseek-setup.md      # Skill（Codex 看）
```

---

## 🔗 依赖

- [codex-bridge](https://github.com/wujfeng712-ui/codex-bridge)
- [CC Switch](https://github.com/farion1231/cc-switch)

---

<p align="center">
  <b>原理统一，模型随便换。⭐ Star 让更多人知道。</b>
</p>
