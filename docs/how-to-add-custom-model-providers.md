# 如何在OpenClaw中添加自定义模型供应商

## 前言

本文介绍如何在OpenClaw中配置自定义的LLM模型供应商，以OpenAI兼容或Anthropic兼容的API为例。

## 步骤概览

1. 准备API密钥
2. 查询API支持的模型列表
3. 配置openclaw.json
4. 重启Gateway
5. 测试验证

## 详细步骤

### 1. 准备API密钥

首先获取API提供商的密钥，将其保存到安全位置：

```bash
# 保存到文件（设置适当权限）
echo "your-api-key-here" > ~/.openclaw/workspace/.provider-api-key
chmod 600 ~/.openclaw/workspace/.provider-api-key
```

### 2. 查询API支持的模型列表

**非常重要！** 在配置前，先查询API返回的模型ID：

```bash
# 查询OpenAI兼容API
curl -X GET "https://api.example.com/v1/models" \
  -H "Authorization: Bearer YOUR_API_KEY"

# 查询Anthropic兼容API
curl -X GET "https://api.example.com/v1/models" \
  -H "x-api-key: YOUR_API_KEY"
```

**常见陷阱：** 很多时候API文档中的模型ID和实际返回的不一致，必须以查询结果为准！

### 3. 配置openclaw.json

编辑 `/root/.openclaw/openclaw.json`：

#### 3.1 添加环境变量（推荐）

将API密钥添加到 `env` 部分：

```json
{
  "env": {
    "MY_PROVIDER_API_KEY": "your-api-key-here"
  }
}
```

#### 3.2 配置models.providers

添加自定义供应商配置：

```json
{
  "models": {
    "providers": {
      "my-provider": {
        "baseUrl": "https://api.example.com",
        "apiKey": "${MY_PROVIDER_API_KEY}",
        "api": "openai-completions",  // 或 "anthropic-messages"
        "models": [
          {
            "id": "gpt-4-turbo",  // 使用查询返回的真实ID
            "name": "GPT-4 Turbo"
          }
        ]
      }
    }
  }
}
```

**API类型说明：**

| `api` 值 | 适用场景 | 端点格式 |
|----------|----------|----------|
| `openai-completions` | OpenAI兼容API | `/v1/chat/completions` |
| `anthropic-messages` | Anthropic兼容API | `/v1/messages` |

#### 3.3 添加模型别名（可选）

在 `agents.defaults.models` 中添加别名方便调用：

```json
{
  "agents": {
    "defaults": {
      "models": {
        "my-provider/gpt-4-turbo": {
          "alias": "GPT4"
        }
      }
    }
  }
}
```

### 4. 重启Gateway

```bash
# 方法1：使用命令
openclaw gateway restart

# 方法2：发送信号（查找进程ID后）
ps aux | grep openclaw-gateway
kill -USR1 <PID>

# 方法3：systemctl（如果作为服务运行）
systemctl restart openclaw-gateway
```

### 5. 测试验证

#### 5.1 手动测试API

```bash
# OpenAI兼容API测试
curl -X POST "https://api.example.com/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "model": "gpt-4-turbo",
    "messages": [{"role": "user", "content": "say hello"}],
    "max_tokens": 10
  }'

# Anthropic兼容API测试
curl -X POST "https://api.example.com/v1/messages" \
  -H "Content-Type: application/json" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-opus-4-5",
    "max_tokens": 10,
    "messages": [{"role": "user", "content": "say hello"}]
  }'
```

#### 5.2 在OpenClaw中测试

使用 `/status` 命令或切换模型测试：

```bash
# 设置默认模型
openclaw models set my-provider/gpt-4-turbo

# 或者在对话中切换模型
/model OPUS1
```

## 常见问题排查

### 问题1：切换模型后无响应/404错误

**可能原因：**
- 模型ID不匹配（最常见！）
- API类型选择错误（openai vs anthropic）
- Base URL配置错误

**解决方法：**
1. 使用 `curl` 查询 `/v1/models` 获取真实模型ID
2. 确认API是OpenAI兼容还是Anthropic兼容
3. 检查 `baseUrl` 是否需要包含 `/v1` 路径

### 问题2：认证失败

**可能原因：**
- API密钥错误或过期
- 认证头格式错误

**解决方法：**
1. 手动用curl测试API密钥是否有效
2. 确认是否使用 `Authorization: Bearer` 或 `x-api-key`

### 问题3：配置修改后不生效

**可能原因：**
- Gateway未重启
- 配置文件语法错误

**解决方法：**
1. 运行 `openclaw doctor` 检查配置
2. 确保Gateway进程已重启

## 实例：配置codesome.cn的Claude Opus

以下是一个完整实例，配置codesome.cn的两个Claude Opus供应商：

### 步骤1：查询模型

```bash
curl -X GET "https://v3.codesome.cn/v1/models" \
  -H "Authorization: Bearer YOUR_API_KEY"

# 返回：
{
  "data": [
    {"id": "claude-opus-4-5-20251101", "type": "model", ...}
  ]
}
```

### 步骤2：配置openclaw.json

```json
{
  "env": {
    "CS_OPUS1_API_KEY": "sk-xxxxx",
    "CS_OPUS2_API_KEY": "sk-yyyyy"
  },
  "models": {
    "providers": {
      "cs-opus1": {
        "baseUrl": "https://v3.codesome.cn",
        "apiKey": "${CS_OPUS1_API_KEY}",
        "api": "anthropic-messages",
        "models": [
          {
            "id": "claude-opus-4-5-20251101",
            "name": "Claude Opus 4.5"
          }
        ]
      },
      "cs-opus2": {
        "baseUrl": "https://v3.codesome.cn",
        "apiKey": "${CS_OPUS2_API_KEY}",
        "api": "anthropic-messages",
        "models": [
          {
            "id": "claude-opus-4-5-20251101",
            "name": "Claude Opus 4.5"
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "models": {
        "cs-opus1/claude-opus-4-5-20251101": {
          "alias": "OPUS1"
        },
        "cs-opus2/claude-opus-4-5-20251101": {
          "alias": "OPUS2"
        }
      }
    }
  }
}
```

### 步骤3：重启并测试

```bash
openclaw gateway restart
# 等待重启完成，然后在对话中切换到OPUS1或OPUS2测试
```

## 安全建议

1. **不要在配置文件中硬编码API密钥**，使用环境变量引用
2. **设置敏感文件权限**为600：`chmod 600 .api-key`
3. **定期轮换API密钥**
4. **将配置文件和密钥文件添加到.gitignore**，不要提交到版本控制

## 总结

配置自定义模型供应商的核心要点：

1. ✅ 先查询API获取真实的模型ID
2. ✅ 确认API类型（OpenAI vs Anthropic）
3. ✅ 使用环境变量管理密钥
4. ✅ 配置完成后重启Gateway
5. ✅ 用curl手动测试验证

按照以上步骤，你就可以在OpenClaw中使用各种自定义的模型供应商了！

---

作者：小迪（奶牛猫）🐱
日期：2026-02-02
