# 🐋 Codex × DeepSeek V4 — 一个 Skill，让 Codex 桌面版流畅使用 DeepSeek V4

<p align="center">
  <img src="https://img.shields.io/badge/Codex-Desktop-blue?style=for-the-badge&logo=openai" alt="Codex">
  <img src="https://img.shields.io/badge/DeepSeek-V4_Pro-purple?style=for-the-badge" alt="DeepSeek V4 Pro">
  <img src="https://img.shields.io/badge/Bridge-codex--bridge-orange?style=for-the-badge" alt="codex-bridge">
  <img src="https://img.shields.io/badge/Setup-One_Click-green?style=for-the-badge" alt="One Click Setup">
</p>

<p align="center">
  <b>一个 Codex Skill，解决一个具体问题：Codex 桌面版接入 DeepSeek V4 Pro。</b><br>
  安装 → 说一句话 → Codex 自己配好 → 开始用。
</p>

---

## 🤔 问题

Codex 桌面版默认只能用 OpenAI 的模型。

DeepSeek V4 Pro 便宜、中文好、推理强——但它和 Codex 说的不是同一套协议。

**Codex 说 Responses API，DeepSeek 说 Chat Completions API，两边天生对不上。**

---

## ✨ 我做了什么

写了一个 Codex Skill：`codex-deepseek-setup`。

把协议翻译、代理配置、模型目录、路由修复、开机自启——全部打包进 Skill 里。

**效果：你把 Skill 装好，对 Codex 说「接入 DeepSeek」，Codex 自己把剩下十步全干了。**

```
你：接入 DeepSeek，Key 是 sk-xxx

Codex：好的，我开始配置。
  ✓ 检查环境
  ✓ 克隆 codex-bridge
  ✓ 生成代理密钥
  ✓ 写入 .env / config.toml / auth.json
  ✓ 生成模型目录
  ✓ 修复 proxy.mjs 路由
  ✓ 配置开机自启
  ✓ 验证通过 — curl health OK，codex exec 返回正常

搞定，重启 Codex 即可使用 DeepSeek V4 Pro。
```

---

## 🚀 怎么用

```bash
# 1. 安装 Skill
mkdir -p ~/.codex/skills/codex-deepseek
cp codex-deepseek-setup.md ~/.codex/skills/codex-deepseek/SKILL.md

# 2. 打开 Codex，说：
```

> 帮我接入 DeepSeek

或者：

> 帮我把 Codex 换成 DeepSeek V4

**Codex 会问你 DeepSeek API Key，剩下全自动。**

---

## 🏗️ 原理

```
┌─────────┐  Responses API   ┌──────────────┐  Chat Completions   ┌──────────────┐
│  Codex  │ ────────────────▶│ codex-bridge │ ──────────────────▶│  DeepSeek    │
│ 桌面版   │ ◀────────────────│  :4000       │ ◀──────────────────│  V4 Pro      │
└─────────┘  SSE / Tool Call  └──────────────┘  SSE / Reasoning   └──────────────┘
```

- Codex 把请求发给本地 `localhost:4000`
- codex-bridge 把 Responses API 翻译成 Chat Completions API 发给 DeepSeek
- DeepSeek 返回的 SSE 流式、reasoning、tool call 再翻译回来给 Codex
- 延迟 < 5ms，本地几乎无感

---

## 🤖 Skill 自动完成的 10 步

| # | 做什么 | 怎么自动 |
|---|--------|----------|
| 0 | 问清 API Key | 对话收集 |
| 1 | 环境检查 | `node --version` |
| 2 | 克隆代理 | `git clone` |
| 3 | 生成密钥 + .env | 自动 openssl |
| 4 | config.toml | 合并写入 |
| 5 | auth.json | 对齐密钥 |
| 6 | proxy-models.json | 模型目录 |
| 7 | 🔑 修复 proxy.mjs 路由 | 精准改代码 |
| 8 | 开机自启 | launchd plist |
| 9 | Shell 别名 | .zshrc 追加 |
| 10 | 端到端验证 | curl + codex exec |

---

## 🧪 实测

| 环境 | 模型 | 结果 |
|------|------|------|
| macOS Sequoia | DeepSeek V4 Pro | ✅ 流畅 |
| macOS Sequoia | DeepSeek V4 Flash | ✅ 流畅 |
| macOS Sequoia | DeepSeek Reasoner | ✅ 流畅 |

---

## 🔧 出了毛病？

不用翻文档。直接对 Codex 说：

> DeepSeek 连不上了，帮我看看

Skill 内置了 6 级诊断清单，Codex 会逐项排查。

---

## 📦 文件

```
.
├── README.md                    # 你在看的
└── codex-deepseek-setup.md      # Skill 本体（Codex 读这个）
```

---

## 🔗 感谢

- [codex-bridge](https://github.com/wujfeng712-ui/codex-bridge)
- [CC Switch](https://github.com/farion1231/cc-switch)

---

<p align="center">
  <b>一个 Skill，搞定一件事。⭐ Star 让更多人用上 DeepSeek + Codex。</b>
</p>
