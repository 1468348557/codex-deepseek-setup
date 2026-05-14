# 🌍 Codex 万能模型接入 — 一个 Skill，接入任意大模型

<p align="center">
  <img src="https://img.shields.io/badge/Codex-Desktop-blue?style=for-the-badge&logo=openai" alt="Codex">
  <img src="https://img.shields.io/badge/Any_Model-DeepSeek|Qwen|Claude|Gemini|Groq|...-purple?style=for-the-badge" alt="Any Model">
  <img src="https://img.shields.io/badge/Protocol-Responses_↔_Chat_Completions-green?style=for-the-badge" alt="Protocol Translation">
  <img src="https://img.shields.io/badge/Setup-Skill_Autopilot-orange?style=for-the-badge" alt="Auto Setup">
</p>

<p align="center">
  <b>安装这个 Skill → 对 Codex 说"接入 XXX 模型" → 全自动搞定。</b><br>
  不管 DeepSeek、千问还是 Groq，原理一样，Skill 替你干活。
</p>

---

## 🪄 一句话

Codex 只会说 Responses API，第三方模型只会说 Chat Completions API。

**这个 Skill 教 Codex 怎么在两套协议间搭桥——然后 Codex 自己帮你搭。**

---

## ⚡ 三秒接入

```bash
# 1. 安装 Skill
mkdir -p ~/.codex/skills/codex-model-integration
cp codex-deepseek-setup.md ~/.codex/skills/codex-model-integration/SKILL.md

# 2. 打开 Codex，随便挑一个说
```

> "帮我接入 DeepSeek"
>
> "帮我把 Codex 换成通义千问"
>
> "接入 Groq"
>
> "Codex 模型连不上了帮我看看"

**Codex 读 Skill → 识别你的目标模型 → 拉代理 → 写配置 → 修路由 → 验证 → 搞定。**

---

## 🎯 支持的模型（代表性，任意 Chat Completions 都行）

| 供应商 | 模型 | 接入难度 |
|--------|------|---------|
| DeepSeek | V4 Pro / Reasoner / Flash | ⭐ |
| 通义千问 / Qwen | Plus / Max / Turbo | ⭐ |
| Groq | Llama 3.3 70B / Mixtral | ⭐ |
| Mistral | Large / Small | ⭐ |
| SiliconFlow 硅基流动 | Qwen / DeepSeek / Llama | ⭐ |
| 智谱 GLM | GLM-4 Plus / Flash | ⭐ |
| Moonshot Kimi | moonshot-v1 | ⭐ |
| 百川 Baichuan | Baichuan4 | ⭐ |
| OpenRouter | Anthropic / OpenAI / Google 等 | ⭐ |

> 🤖 **原理通用：只要模型 API 兼容 OpenAI Chat Completions 格式，这个 Skill 就能接入。**

---

## 🧠 为什么「一个 Skill 解决所有模型」？

```
Codex   ──Responses API──▶  codex-bridge (:4000)  ──Chat Completions──▶  任意模型
                              ↑ 本地双向协议翻译                              ↑
                              所有模型的共同瓶颈                          所有模型的共同接口
```

**核心瓶颈只有一个：协议翻译。** 不同模型只是改个 base URL + API Key。

Skill 里预置了常见供应商的 base URL、模型 slug、配置模板。碰上不在表的，用户给个 URL 就行。

---

## 🚀 Skill 自动完成的全部步骤

| # | 步骤 | 咋自动 |
|---|------|--------|
| 0 | 识别目标模型 | 对话中问清模型名 / API 地址 / Key |
| 1 | 环境检查 | `node --version` |
| 2 | 克隆代理 | `git clone codex-bridge` |
| 3 | 生成密钥 + 写 .env | 自动生成 + 文件工具写入 |
| 4 | 编辑 config.toml | 合并到已有配置 |
| 5 | 写入 auth.json | 对齐代理密钥 |
| 6 | 生成模型目录 | 根据供应商填充 model catalog |
| 7 | 修复 proxy.mjs 路由 | 定位代码 + 精准修改 |
| 8 | 配置开机自启 | macOS launchd plist |
| 9 | 设置 Shell 变量 | NO_PROXY + 别名 |
| 10 | 端到端验证 | curl + codex exec |

---

## 🔧 故障自愈

Skill 内置了 6 级排查清单。接入出问题不用你查——**直接对 Codex 说"模型连不上了"**，Codex 按清单逐项诊断。

| 优先级 | 问题 | Codex 自动检查 |
|--------|------|---------------|
| 🔴 P1 | 代理没跑 | `curl health` → 启动/报错分析 |
| 🔴 P2 | 绕过代理直连 OpenAI | 检查 config.toml model_provider |
| 🟡 P3 | 502 路由不匹配 | 检查 proxy.mjs 修复 |
| 🟡 P4 | 模型列表不对 | 检查 model_catalog_json |
| 🟡 P5 | 配置被反复覆盖 | 指导 CC Switch 持久化 |
| 🟢 P6 | 401 认证失败 | 对齐 PROXY_AUTH_KEY 与 auth.json |

---

## 📦 仓库文件

```
.
├── README.md                    # 你正在看的（人读）
└── codex-deepseek-setup.md      # Skill 定义（Codex 读，核心）
```

---

## 🔗 依赖

- [codex-bridge](https://github.com/wujfeng712-ui/codex-bridge) — 本地协议代理
- [CC Switch](https://github.com/farion1231/cc-switch) — Codex 供应商切换（推荐）

---

## ⭐ 值的话，给个 Star

如果这个 Skill 帮你省了折腾模型接入的时间，Star ⭐ 让更多码农解放出来写代码，而不是配模型。

---

<p align="center">
  <sub>原理统一，接入任意模型。Codex + Any LLM = ❤️</sub>
</p>
