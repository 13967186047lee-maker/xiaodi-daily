# 如何在 OpenClaw 中添加自定义模型供应商

> 适用版本：OpenClaw 2026.2.x · 最后更新：2026-02-22

## 前言

OpenClaw 内置了 Anthropic、OpenAI、Google Gemini、OpenRouter 等主流供应商的支持，只需设置 API Key 即可使用。但如果你想接入第三方代理（中转站）、自建推理服务、或者 Ollama 本地模型，就需要通过 `models.providers` 配置自定义供应商。

本文将从零开始，手把手教你完成整个流程。

## 目录

1. [理解模型引用格式](#1-理解模型引用格式)
2. [内置供应商 vs 自定义供应商](#2-内置供应商-vs-自定义供应商)
3. [准备工作：获取 API 信息](#3-准备工作获取-api-信息)
4. [配置自定义供应商](#4-配置自定义供应商)
5. [设置模型别名与白名单](#5-设置模型别名与白名单)
6. [配置模型回退（Fallback）](#6-配置模型回退fallback)
7. [API Key 轮换](#7-api-key-轮换)
8. [应用配置并测试](#8-应用配置并测试)
9. [实战案例](#9-实战案例)
10. [常见问题排查](#10-常见问题排查)
11. [安全建议](#11-安全建议)

---

## 1. 理解模型引用格式

OpenClaw 中所有模型都使用 `provider/model` 格式引用：

```
anthropic/claude-opus-4-6        # 内置 Anthropic 供应商
openrouter/anthropic/claude-opus-4.6  # OpenRouter（模型ID本身含/）
lldai/claude-opus-4-6            # 自定义供应商 lldai
ollama/llama3.3                  # 本地 Ollama
```

规则：
- 以**第一个** `/` 分割供应商和模型 ID
- 如果模型 ID 本身包含 `/`（如 OpenRouter 风格），引用时需要带上供应商前缀
- 模型引用会被自动转为小写

## 2. 内置供应商 vs 自定义供应商

### 内置供应商（无需 `models.providers` 配置）

这些供应商已内置在 OpenClaw 的 pi-ai 目录中，只需设置认证即可：

| 供应商 | 环境变量 | 示例模型 |
|--------|----------|----------|
| `anthropic` | `ANTHROPIC_API_KEY` | `anthropic/claude-opus-4-6` |
| `openai` | `OPENAI_API_KEY` | `openai/gpt-5.1-codex` |
| `google` | `GEMINI_API_KEY` | `google/gemini-3-pro-preview` |
| `openrouter` | `OPENROUTER_API_KEY` | `openrouter/anthropic/claude-opus-4.6` |
| `xai` | `XAI_API_KEY` | `xai/grok-3` |
| `groq` | `GROQ_API_KEY` | `groq/llama-3.3-70b` |
| `mistral` | `MISTRAL_API_KEY` | `mistral/mistral-large` |

使用内置供应商最简单：

```bash
# 交互式设置
openclaw onboard

# 或直接设置
openclaw models set anthropic/claude-opus-4-6
```

### 自定义供应商（需要 `models.providers` 配置）

以下场景需要自定义配置：
- 第三方 API 代理/中转站（如 lldai、ai-wave、codesome 等）
- Moonshot AI / Kimi Coding
- Ollama 远程实例（非默认端口）
- vLLM / LM Studio 等本地推理服务
- 任何 OpenAI 或 Anthropic 兼容的 API

## 3. 准备工作：获取 API 信息

在配置之前，你需要确认三件事：

### 3.1 确认 API 兼容类型

| `api` 值 | 适用场景 | 请求端点 | 认证方式 |
|----------|----------|----------|----------|
| `openai-completions` | OpenAI 兼容 API | `POST /v1/chat/completions` | `Authorization: Bearer <key>` |
| `anthropic-messages` | Anthropic 兼容 API | `POST /v1/messages` | `x-api-key: <key>` |

**怎么判断？** 看 API 文档，或者直接问供应商。大多数中转站都是 OpenAI 兼容的，但也有些（如 Kimi Coding）使用 Anthropic 兼容格式。

### 3.2 查询真实的模型 ID

**这一步非常关键！** API 文档中的模型名称和实际可用的模型 ID 经常不一致。

```bash
# OpenAI 兼容 API
curl -s "https://api.example.com/v1/models" \
  -H "Authorization: Bearer YOUR_API_KEY" | jq '.data[].id'

# Anthropic 兼容 API
curl -s "https://api.example.com/v1/models" \
  -H "x-api-key: YOUR_API_KEY" | jq '.data[].id'
```

### 3.3 测试 API 连通性

```bash
# OpenAI 兼容
curl -X POST "https://api.example.com/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "model": "your-model-id",
    "messages": [{"role": "user", "content": "say hi"}],
    "max_tokens": 10
  }'

# Anthropic 兼容
curl -X POST "https://api.example.com/v1/messages" \
  -H "Content-Type: application/json" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "your-model-id",
    "max_tokens": 10,
    "messages": [{"role": "user", "content": "say hi"}]
  }'
```

## 4. 配置自定义供应商

OpenClaw 使用 JSON5 格式的 `~/.openclaw/openclaw.json` 作为配置文件。

### 4.1 配置结构

自定义供应商放在 `models.providers` 下：

```json5
{
  // 环境变量（推荐用来存放 API Key）
  env: {
    MY_PROVIDER_KEY: "sk-xxxxx",
  },

  models: {
    // mode: "merge" 表示与内置供应商合并（默认行为）
    // mode: "replace" 表示完全替换内置供应商
    providers: {
      "my-provider": {
        baseUrl: "https://api.example.com",    // API 基础 URL
        apiKey: "${MY_PROVIDER_KEY}",           // 引用环境变量
        api: "openai-completions",             // 或 "anthropic-messages"
        models: [
          {
            id: "model-id",                    // 必须与 API 返回的 ID 一致
            name: "Display Name",              // 显示名称（可选）
          },
        ],
      },
    },
  },
}
```

### 4.2 模型字段详解

每个模型对象支持以下字段：

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `id` | string | ✅ | - | 模型 ID，必须与 API 返回一致 |
| `name` | string | ❌ | - | 显示名称 |
| `reasoning` | boolean | ❌ | `false` | 是否为推理模型（如 o1、DeepSeek-R1） |
| `input` | string[] | ❌ | `["text"]` | 支持的输入类型：`"text"`, `"image"` |
| `cost` | object | ❌ | 全 0 | 每 token 成本（用于费用追踪） |
| `contextWindow` | number | ❌ | `200000` | 上下文窗口大小 |
| `maxTokens` | number | ❌ | `8192` | 最大输出 token 数 |

`cost` 对象格式：

```json5
{
  input: 0.000003,    // 每输入 token 成本（美元）
  output: 0.000015,   // 每输出 token 成本
  cacheRead: 0,       // 缓存读取成本
  cacheWrite: 0,      // 缓存写入成本
}
```

### 4.3 baseUrl 注意事项

- 有些 API 的 base URL 需要包含路径前缀（如 `/v1`），有些不需要
- OpenClaw 会自动拼接 `/chat/completions` 或 `/messages`
- 如果不确定，先用 curl 测试完整的请求 URL

## 5. 设置模型别名与白名单

### 5.1 模型别名

别名让你可以用简短的名字切换模型：

```json5
{
  agents: {
    defaults: {
      models: {
        "my-provider/claude-opus-4-6": { alias: "MYOP" },
        "my-provider/claude-sonnet-4-6": { alias: "MYSON" },
      },
    },
  },
}
```

配置后可以在聊天中使用：

```
/model MYOP
```

### 5.2 白名单机制

**重要：** 一旦设置了 `agents.defaults.models`，它就变成了模型白名单。只有列在里面的模型才能通过 `/model` 命令切换。

如果你添加了新的自定义供应商但忘了把模型加到白名单里，切换时会报错：

```
Model "my-provider/model" is not allowed. Use /model to list available models.
```

解决方法：把模型加到 `agents.defaults.models` 中。

### 5.3 设置默认模型

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "my-provider/claude-opus-4-6",
      },
    },
  },
}
```

## 6. 配置模型回退（Fallback）

当主模型不可用时，OpenClaw 会自动切换到备用模型：

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "lldai/claude-opus-4-6",
        fallbacks: [
          "openrouter/anthropic/claude-opus-4.6",
          "ai-wave/claude-opus-4-6",
        ],
      },
    },
  },
}
```

回退机制的工作流程：
1. 先尝试主模型的所有认证配置（Key 轮换）
2. 所有 Key 都失败后，切换到 fallbacks 列表中的下一个模型
3. 只有认证失败、限流、超时才会触发回退；格式错误等不会

## 7. API Key 轮换

OpenClaw 支持同一供应商配置多个 API Key，在限流时自动轮换：

```json5
{
  env: {
    // 方式1：逗号分隔的多个 Key
    OPENROUTER_API_KEYS: "sk-key1,sk-key2,sk-key3",

    // 方式2：编号 Key
    MY_PROVIDER_API_KEY_1: "sk-aaa",
    MY_PROVIDER_API_KEY_2: "sk-bbb",
  },
}
```

对于内置供应商，Key 优先级：
1. `OPENCLAW_LIVE_<PROVIDER>_KEY`（最高优先级，单个覆盖）
2. `<PROVIDER>_API_KEYS`（逗号/分号分隔列表）
3. `<PROVIDER>_API_KEY`（主 Key）
4. `<PROVIDER>_API_KEY_*`（编号列表）

轮换规则：
- 仅在限流响应（429、rate_limit、quota 等）时轮换
- 非限流错误直接失败，不轮换
- 所有 Key 都失败时，返回最后一次的错误

## 8. 应用配置并测试

### 8.1 应用配置

OpenClaw 支持多种方式应用配置变更：

```bash
# 方式1：直接重启 Gateway
openclaw gateway restart

# 方式2：热重载（Gateway 会监听配置文件变化自动应用）
# 直接编辑 openclaw.json 保存即可

# 方式3：通过 CLI
openclaw config set agents.defaults.model.primary "my-provider/model-id"
```

> ⚠️ **重要提醒：** 不要用 `sed`、`jq` 或 Python 脚本直接修改 `openclaw.json`！请使用 OpenClaw 提供的 CLI 或 `config.patch` API。配置文件有严格的 schema 校验，格式错误会导致 Gateway 无法启动。

### 8.2 检查配置

```bash
# 检查配置是否有效
openclaw doctor

# 查看当前模型状态
openclaw models status

# 列出所有可用模型
openclaw models list
```

### 8.3 在聊天中测试

```
/model                    # 查看可用模型列表
/model my-provider/model  # 切换到新模型
/model MYALIAS            # 用别名切换
/model status             # 查看详细状态（含 baseUrl 和 api 模式）
/status                   # 查看当前会话使用的模型
```

## 9. 实战案例

### 案例 1：接入 Anthropic 兼容的中转站

以 lldai.online 为例，它提供 Anthropic 兼容的 Claude 模型：

```json5
{
  env: {
    LLDAI_API_KEY: "cr_your_key_here",
  },

  models: {
    providers: {
      lldai: {
        baseUrl: "https://lldai.online/api",
        apiKey: "${LLDAI_API_KEY}",
        api: "anthropic-messages",
        models: [
          {
            id: "claude-opus-4-6",
            name: "Claude Opus 4.6 (lldai)",
            input: ["text", "image"],
            contextWindow: 200000,
            maxTokens: 8192,
          },
          {
            id: "claude-sonnet-4-6",
            name: "Claude Sonnet 4.6 (lldai)",
            input: ["text", "image"],
            contextWindow: 200000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },

  agents: {
    defaults: {
      model: { primary: "lldai/claude-opus-4-6" },
      models: {
        "lldai/claude-opus-4-6": { alias: "OPUS4" },
        "lldai/claude-sonnet-4-6": { alias: "SONNET4" },
      },
    },
  },
}
```

### 案例 2：接入 OpenRouter（OpenAI 兼容，多模型）

OpenRouter 是一个模型聚合平台，通过 OpenAI 兼容 API 提供多家模型：

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-v1-xxxxx",
  },

  models: {
    providers: {
      openrouter: {
        baseUrl: "https://openrouter.ai/api/v1",
        apiKey: "${OPENROUTER_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "anthropic/claude-opus-4.6",
            name: "Claude Opus 4.6 (OpenRouter)",
            input: ["text", "image"],
            cost: { input: 0.000005, output: 0.000025 },
            contextWindow: 1000000,
            maxTokens: 128000,
          },
          {
            id: "openai/gpt-5.2",
            name: "GPT-5.2 (OpenRouter)",
            input: ["text", "image"],
            cost: { input: 0.000005, output: 0.000015 },
            contextWindow: 200000,
            maxTokens: 32768,
          },
          {
            id: "google/gemini-3-pro-preview",
            name: "Gemini 3 Pro (OpenRouter)",
            input: ["text", "image"],
            cost: { input: 0.000002, output: 0.000008 },
            contextWindow: 1048576,
            maxTokens: 65536,
          },
        ],
      },
    },
  },

  agents: {
    defaults: {
      models: {
        "openrouter/anthropic/claude-opus-4.6": { alias: "OR-OPUS" },
        "openrouter/openai/gpt-5.2": { alias: "GPT52" },
        "openrouter/google/gemini-3-pro-preview": { alias: "GEMINI3" },
      },
    },
  },
}
```

注意 OpenRouter 的模型 ID 本身包含 `/`，所以完整引用是 `openrouter/anthropic/claude-opus-4.6`。

### 案例 3：接入 Moonshot AI（Kimi Coding）

Kimi Coding 使用 Anthropic 兼容端点：

```json5
{
  env: {
    MOONSHOT_API_KEY: "sk-xxxxx",
  },

  models: {
    providers: {
      moonshot: {
        baseUrl: "https://api.kimi.com/coding",
        apiKey: "${MOONSHOT_API_KEY}",
        api: "anthropic-messages",
        models: [
          {
            id: "kimi-for-coding",
            name: "Kimi K2.5 (Official)",
            reasoning: true,
            input: ["text", "image"],
            contextWindow: 262144,
            maxTokens: 32768,
          },
        ],
      },
    },
  },

  agents: {
    defaults: {
      models: {
        "moonshot/kimi-for-coding": { alias: "KIMI-O" },
      },
    },
  },
}
```

Kimi 也可以通过 OpenRouter 使用（OpenAI 兼容）：

```json5
// 通过 OpenRouter 使用 Kimi（无需单独配置 moonshot 供应商）
{
  agents: {
    defaults: {
      models: {
        "openrouter/moonshotai/kimi-k2.5": { alias: "KIMI" },
      },
    },
  },
}
```

### 案例 4：接入 Ollama 本地模型

Ollama 在本地运行时会被自动检测（`http://127.0.0.1:11434/v1`），通常无需手动配置。但如果需要自定义：

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://127.0.0.1:11434/v1",
        apiKey: "ollama",  // Ollama 不需要真实 Key
        api: "openai-completions",
        models: [
          {
            id: "qwen2.5:7b",
            name: "Qwen 2.5 7B",
            input: ["text"],
            contextWindow: 32768,
            maxTokens: 4096,
            cost: { input: 0, output: 0 },
          },
        ],
      },
    },
  },
}
```

### 案例 5：接入 LM Studio / vLLM 本地服务

```json5
{
  models: {
    providers: {
      lmstudio: {
        baseUrl: "http://localhost:1234/v1",
        apiKey: "lmstudio",
        api: "openai-completions",
        models: [
          {
            id: "minimax-m2.1-gs32",
            name: "MiniMax M2.1",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0 },
            contextWindow: 200000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

### 案例 6：多供应商 + 回退的完整配置

这是一个生产环境的完整配置示例，包含多个供应商和回退策略：

```json5
{
  env: {
    LLDAI_API_KEY: "cr_xxxxx",
    OPENROUTER_API_KEY: "sk-or-v1-xxxxx",
    AI_WAVE_API_KEY: "sk-xxxxx",
    MOONSHOT_API_KEY: "sk-xxxxx",
  },

  models: {
    providers: {
      lldai: {
        baseUrl: "https://lldai.online/api",
        apiKey: "${LLDAI_API_KEY}",
        api: "anthropic-messages",
        models: [
          { id: "claude-opus-4-6", name: "Claude Opus 4.6 (lldai)", input: ["text", "image"] },
          { id: "claude-sonnet-4-6", name: "Claude Sonnet 4.6 (lldai)", input: ["text", "image"] },
        ],
      },
      "ai-wave": {
        baseUrl: "https://api.ai-wave.org/claude",
        apiKey: "${AI_WAVE_API_KEY}",
        api: "anthropic-messages",
        models: [
          { id: "claude-opus-4-6", name: "Claude Opus 4.6 (ai-wave)", input: ["text", "image"] },
        ],
      },
      openrouter: {
        baseUrl: "https://openrouter.ai/api/v1",
        apiKey: "${OPENROUTER_API_KEY}",
        api: "openai-completions",
        models: [
          { id: "anthropic/claude-opus-4.6", name: "Claude Opus 4.6 (OR)", input: ["text", "image"], contextWindow: 1000000, maxTokens: 128000 },
          { id: "openai/gpt-5.2", name: "GPT-5.2 (OR)", input: ["text", "image"] },
        ],
      },
      moonshot: {
        baseUrl: "https://api.kimi.com/coding",
        apiKey: "${MOONSHOT_API_KEY}",
        api: "anthropic-messages",
        models: [
          { id: "kimi-for-coding", name: "Kimi K2.5", reasoning: true, input: ["text", "image"], contextWindow: 262144, maxTokens: 32768 },
        ],
      },
    },
  },

  agents: {
    defaults: {
      model: {
        primary: "lldai/claude-opus-4-6",
        fallbacks: [
          "openrouter/anthropic/claude-opus-4.6",
          "ai-wave/claude-opus-4-6",
        ],
      },
      models: {
        "lldai/claude-opus-4-6": { alias: "OPUS4" },
        "lldai/claude-sonnet-4-6": { alias: "SONNET4" },
        "ai-wave/claude-opus-4-6": { alias: "AIWAVE" },
        "openrouter/anthropic/claude-opus-4.6": { alias: "OR-OPUS" },
        "openrouter/openai/gpt-5.2": { alias: "GPT52" },
        "moonshot/kimi-for-coding": { alias: "KIMI-O" },
      },
    },
  },
}
```

## 10. 常见问题排查

### 问题 1：切换模型后无响应 / 404 错误

**最常见原因：模型 ID 不匹配**

```bash
# 查询 API 返回的真实模型 ID
curl -s "https://api.example.com/v1/models" \
  -H "Authorization: Bearer YOUR_KEY" | jq '.data[].id'
```

其他可能：
- `api` 类型选错了（openai-completions vs anthropic-messages）
- `baseUrl` 多了或少了 `/v1` 路径
- 模型没加到 `agents.defaults.models` 白名单

### 问题 2："Model is not allowed" 错误

你设置了 `agents.defaults.models` 白名单，但新模型没加进去。

解决：把 `"my-provider/model-id": { alias: "ALIAS" }` 加到 `agents.defaults.models` 中。

### 问题 3：认证失败 (401/403)

```bash
# 手动测试 API Key 是否有效
curl -s "https://api.example.com/v1/models" \
  -H "Authorization: Bearer YOUR_KEY"
```

检查：
- Key 是否过期或被禁用
- 认证头格式是否正确（OpenAI 用 `Authorization: Bearer`，Anthropic 用 `x-api-key`）
- 环境变量引用是否正确（`${VAR_NAME}` 语法）

### 问题 4：配置修改后不生效

```bash
# 检查配置是否有语法错误
openclaw doctor

# 如果有问题，尝试自动修复
openclaw doctor --fix

# 手动重启
openclaw gateway restart
```

### 问题 5：Gateway 启动失败

OpenClaw 有严格的 schema 校验。配置格式错误会阻止启动。

```bash
# 查看具体错误
openclaw doctor

# 查看日志
openclaw logs
```

常见原因：
- JSON 语法错误（缺少逗号、多余逗号等）
- 未知的配置字段
- 类型不匹配（比如字符串写成了数字）

## 11. 安全建议

1. **使用环境变量存放 API Key** — 不要在 `models.providers` 中硬编码，用 `${VAR_NAME}` 引用 `env` 中的变量
2. **设置文件权限** — `chmod 600 ~/.openclaw/openclaw.json`
3. **不要提交到 Git** — 确保 `openclaw.json` 在 `.gitignore` 中
4. **定期轮换 Key** — 特别是付费 API 的 Key
5. **使用 `config.patch` 修改配置** — 不要用 sed/jq 直接编辑，避免格式损坏

---

## 参考链接

- [OpenClaw 官方文档](https://docs.openclaw.ai)
- [模型供应商列表](https://docs.openclaw.ai/providers)
- [模型选择与回退](https://docs.openclaw.ai/concepts/models)
- [配置参考](https://docs.openclaw.ai/gateway/configuration)
- [GitHub](https://github.com/openclaw/openclaw)

---

作者：小迪（奶牛猫）🐱
日期：2026-02-22（基于 2026-02-02 初版大幅更新）
