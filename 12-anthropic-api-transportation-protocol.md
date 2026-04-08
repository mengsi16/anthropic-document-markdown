# Anthropic API 传输协议详解

> 本文档详细解析 Anthropic Claude API 的底层通信协议、HTTP 规范、SSE 流式传输机制以及错误处理机制。

## 目录

1. [API 基础架构](#1-api-基础架构)
2. [HTTP 协议规范](#2-http-协议规范)
3. [认证机制](#3-认证机制)
4. [Messages API 完整协议](#4-messages-api-完整协议)
5. [SSE 流式传输协议](#5-sse-流式传输协议)
6. [事件类型与数据流](#6-事件类型与数据流)
7. [错误处理协议](#7-错误处理协议)
8. [速率限制与配额管理](#8-速率限制与配额管理)
9. [SDK 集成模式](#9-sdk-集成模式)
10. [安全最佳实践](#10-安全最佳实践)

---

## 1. API 基础架构

### 1.1 端点地址

```
Base URL: https://api.anthropic.com
主端点:   POST /v1/messages
```

### 1.2 可用 API 端点

| 端点 | 方法 | 状态 | 描述 |
|------|------|------|------|
| `/v1/messages` | POST | GA | Messages API - 对话交互 |
| `/v1/messages/batches` | POST | GA | 批量消息处理（50%成本优惠）|
| `/v1/messages/count_tokens` | POST | GA | Token 计数 |
| `/v1/models` | GET | GA | 列出可用模型 |
| `/v1/files` | POST/GET | Beta | 文件上传和管理 |
| `/v1/skills` | POST/GET | Beta | 自定义 Agent 技能 |

### 1.3 请求大小限制

| 端点类型 | 最大请求大小 |
|----------|-------------|
| 标准（Messages, Token Counting）| **32 MB** |
| Batch API | **256 MB** |
| Files API | **500 MB** |

超过限制返回 `413 request_too_large` 错误。

---

## 2. HTTP 协议规范

### 2.1 必需的请求头

所有请求必须包含以下 HTTP 头部：

```http
POST /v1/messages HTTP/1.1
Host: api.anthropic.com
Content-Type: application/json
x-api-key: sk-ant-api03-xxxxxxxxxxxxxxx
anthropic-version: 2023-06-01
```

| Header | 值 | 必填 | 说明 |
|--------|-----|------|------|
| `x-api-key` | 你的 API Key | ✅ | 从 Console 获取的密钥 |
| `anthropic-version` | 版本号 (如 `2023-06-01`) | ✅ | API 版本控制 |
| `content-type` | `application/json` | ✅ | 请求体格式 |

### 2.2 响应头

每个 API 响应都包含以下头部：

```
request-id: req_018EeWyXxfu5pfWkrYcMdjWG
anthropic-organization-id: org_xxxxxxxxxxxxxxx
```

| Header | 说明 |
|--------|------|
| `request-id` | 请求的全局唯一标识符，用于调试和支持 |
| `anthropic-organization-id` | 与 API Key 关联的组织 ID |

---

## 3. 认证机制

### 3.1 API Key 认证

Anthropic 使用简单的 API Key 认证方式：

```bash
curl https://api.anthropic.com/v1/messages \
  --header "x-api-key: $ANTHROPIC_API_KEY" \
  --header "anthropic-version: 2023-06-01" \
  --header "content-type: application/json" \
  --data '{
    "model": "claude-opus-4-6",
    "max_tokens": 1024,
    "messages": [{"role": "user", "content": "Hello, Claude"}]
  }'
```

### 3.2 API 密钥获取流程

1. 注册 Anthropic Console 账户
2. 在 Account Settings 中生成 API Key
3. 使用 Workspaces 分割 API Key 并控制支出
4. 设置使用场景级别的访问控制

### 3.3 第三方平台认证

Claude 也通过云服务商提供：

| 平台 | 文档 |
|------|------|
| Amazon Bedrock | AWS IAM + Bedrock API |
| Google Cloud Vertex AI | GCP OAuth + Vertex API |
| Microsoft Azure AI | Azure AD + Azure OpenAI 兼容 API |

> 注意：第三方平台可能有功能延迟或差异，且请求大小限制不同：
> - Vertex AI: 30 MB
> - Amazon Bedrock: 20 MB

---

## 4. Messages API 完整协议

### 4.1 请求结构

#### 基本 URL

```
POST https://api.anthropic.com/v1/messages
```

#### 请求体 Schema

```json
{
  // 必填参数
  "model": "claude-opus-4-6",
  "max_tokens": 1024,
  "messages": [
    {
      "role": "user" | "assistant",
      "content": string | ContentBlock[]
    }
  ],

  // 可选参数
  "stream": boolean,
  "system": string | SystemContentBlock[],
  "temperature": number,
  "top_p": number,
  "top_k": number,
  "stop_sequences": string[],
  "tools": ToolDefinition[],
  "tool_choice": ToolChoice,
  "thinking": ThinkingConfig,
  "metadata": MetadataConfig,
  "output_config": OutputConfig
}
```

#### messages 字段详解

`content` 支持多种格式：

**纯文本：**
```json
{"role": "user", "content": "Hello, Claude"}
```

**多模态内容块：**
```json
{
  "role": "user",
  "content": [
    {"type": "text", "text": "这是什么图片？"},
    {"type": "image", "source": {
      "type": "base64",
      "media_type": "image/png",
      "data": "iVBORw0KGgoAAAANSUhEUgAA..."
    }}
  ]
}
```

#### tools 工具定义格式

```json
{
  "name": "get_weather",
  "description": "Get the current weather in a given location",
  "input_schema": {
    "type": "object",
    "properties": {
      "location": {
        "type": "string",
        "description": "The city and state, e.g. San Francisco, CA"
      }
    },
    "required": ["location"]
  }
}
```

### 4.2 响应结构（非流式）

```json
{
  "id": "msg_01XFDUDYJgAACzvnptvVoYEL",
  "type": "message",
  "role": "assistant",
  "model": "claude-opus-4-6",
  "stop_reason": "end_turn",
  "stop_sequence": null,
  "content": [
    {
      "type": "text",
      "text": "Hello! How can I assist you today?"
    }
  ],
  "usage": {
    "input_tokens": 12,
    "output_tokens": 8,
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 0
  }
}
```

#### stop_reason 枚举值

| 值 | 含义 |
|-----|------|
| `end_turn` | 正常结束，轮次完成 |
| `max_tokens` | 达到 max_tokens 上限 |
| `stop_sequence` | 匹配到 stop_sequence |
| `tool_use` | 模型请求调用工具 |
| `end_turn` | 正常结束 |

---

## 5. SSE 流式传输协议

### 5.1 启用流式传输

在请求中设置 `"stream": true`：

```bash
curl https://api.anthropic.com/v1/messages \
     --header "x-api-key: $ANTHROPIC_API_KEY" \
     --header "anthropic-version: 2023-06-01" \
     --header "content-type: application/json" \
     --data '{
       "model": "claude-opus-4-6",
       "messages": [{"role": "user", "content": "Hello"}],
       "max_tokens": 256,
       "stream": true
     }'
```

### 5.2 SSE 事件流结构

流式响应使用 Server-Sent Events (SSE) 格式。每个事件由 `event:` 行和 `data:` 行组成：

```
event: message_start
data: {...}

event: content_block_start
data: {...}

event: content_block_delta
data: {...}

...

event: message_stop
data: {...}
```

### 5.3 完整事件生命周期

一个完整的 SSE 流遵循以下顺序：

```mermaid
flowchart TD
    A["1. message_start<br/>包含空 Message 对象<br/>(id, role, model)"] --> B

    subgraph B ["2. 内容块序列 (可重复多个)"]
        direction TB
        B1["content_block_start<br/>定义块类型和索引"] --> B2["content_block_delta ×N<br/>增量数据"]
        B2 --> B3["content_block_stop<br/>标记块结束"]
    end

    B --> C["3. message_delta<br/>包含 stop_reason + usage"]
    C --> D["4. message_stop<br/>标记整个响应完成"]

    style A fill:#e1f5fe
    style D fill:#c8e6c9
    style B fill:#fff3e0
    style C fill:#fce4ec
```

### 5.4 基础流式示例（文本）

**Request:**
```bash
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-opus-4-6","messages":[{"role":"user","content":"Hello"}],"max_tokens":256,"stream":true}'
```

**Response (SSE Stream):**
```
event: message_start
data: {"type":"message_start","message":{"id":"msg_1nZdL29xx5MUA1yADyHTEsnR8uuvGzszyY","type":"message","role":"assistant","content":[],"model":"claude-opus-4-6","stop_reason":null,"stop_sequence":null,"usage":{"input_tokens":25,"output_tokens":1}}}

event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

event: ping
data: {"type":"ping"}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"Hello"}}

event: content_block_delta
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"!"}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"end_turn","stop_sequence":null},"usage":{"output_tokens":15}}

event: message_stop
data: {"type":"message_stop"}
```

---

## 6. 事件类型与数据流

### 6.1 事件类型总览

| 事件名称 | 类型 | 说明 |
|---------|------|------|
| `message_start` | MessageStart | 响应开始，包含元数据 |
| `content_block_start` | ContentBlockStart | 新的内容块开始 |
| `content_block_delta` | ContentBlockDelta | 内容块增量数据 |
| `content_block_stop` | ContentBlockStop | 内容块结束 |
| `message_delta` | MessageDelta | 顶层消息更新 |
| `message_stop` | MessageStop | 响应结束 |
| `ping` | Ping | 心跳/保活事件 |
| `error` | Error | 错误信息 |

### 6.2 各事件的详细 Schema

#### message_start

```json
{
  "type": "message_start",
  "message": {
    "id": "msg_01XFDUDYJgAACzvnptvVoYEL",
    "type": "message",
    "role": "assistant",
    "content": [],
    "model": "claude-opus-4-6",
    "stop_reason": null,
    "stop_sequence": null,
    "usage": {
      "input_tokens": 10,
      "output_tokens": 1
    }
  }
}
```

#### content_block_start

```json
{
  "type": "content_block_start",
  "index": 0,
  "content_block": {
    "type": "text",          // 或 "tool_use", "thinking"
    "text": ""              // 初始为空
  }
}
```

支持的 content_block type:
- `text`: 文本输出
- `tool_use`: 工具调用
- `thinking`: 扩展思考
- `server_tool_use`: 服务端工具（如 web_search）

#### content_block_delta - 文本增量

```json
{
  "type": "content_block_delta",
  "index": 0,
  "delta": {
    "type": "text_delta",
    "text": "Hello, world!"
  }
}
```

#### content_block_delta - 工具输入增量

```json
{
  "type": "content_block_delta",
  "index": 1,
  "delta": {
    "type": "input_json_delta",
    "partial_json": "{\"location\": \"San Fra"
  }
}
```

> **重要**: tool_use 的 input 是以部分 JSON 字符串的形式增量发送的。需要累积所有 delta 后解析。

#### content_block_delta - 思考增量

```json
{
  "type": "content_block_delta",
  "index": 0,
  "delta": {
    "type": "thinking_delta",
    "thinking": "I need to find the GCD of 1071 and 462..."
  }
}
```

#### content_block_delta - 签名增量（用于验证）

```json
{
  "type": "content_block_delta",
  "index": 0,
  "delta": {
    "type": "signature_delta",
    "signature": "EqQBCgIYAhIM1gbcDa9GJwZA2b3hGgxBdjrkzLoky3dl1pkiMOYds..."
  }
}
```

#### message_delta

```json
{
  "type": "message_delta",
  "delta": {
    "stop_reason": "end_turn",      // 或 "tool_use", "max_tokens"
    "stop_sequence": null
  },
  "usage": {
    "output_tokens": 15            // 累计输出 token 数
  }
}
```

#### ping

```json
{
  "type": "ping"
}
```

Ping 事件可以在流中的任何位置出现，数量不限。用于保持连接活跃。

#### error

```json
event: error
data: {"type":"error","error":{"type":"overloaded_error","message":"Overloaded"}}
```

### 6.3 工具调用的流式传输

当 Claude 决定使用工具时，流式响应会包含两个内容块：

**Request:**
```json
{
  "model": "claude-opus-4-6",
  "max_tokens": 1024,
  "tools": [{
    "name": "get_weather",
    "description": "Get the current weather",
    "input_schema": {
      "type": "object",
      "properties": {
        "location": {"type": "string"}
      },
      "required": ["location"]
    }
  }],
  "tool_choice": {"type": "any"},
  "messages": [{"role": "user", "content": "What's the weather in SF?"}],
  "stream": true
}
```

**Response (关键部分):**
```
event: content_block_start
data: {"type":"content_block_start","index":0,"content_block":{"type":"text","text":""}}

event: content_block_delta  × N (文本回复)
data: {"type":"content_block_delta","index":0,"delta":{"type":"text_delta","text":"Okay, let me check"}}

event: content_block_stop
data: {"type":"content_block_stop","index":0"}

event: content_block_start  ← 第二个块：工具调用开始
data: {"type":"content_block_start","index":1,"content_block":{"type":"tool_use","id":"toolu_01T1x1fJ34qAmk2tNTrN7Up6","name":"get_weather","input":{}}}

event: content_block_delta  × N (工具参数逐步构建)
data: {"type":"content_block_delta","index":1,"delta":{"type":"input_json_delta","partial_json":""}}
data: {"type":"content_block_delta","index":1,"delta":{"type":"input_json_delta","partial_json":"{\"location\":""}}
data: {"type":"content_block_delta","index":1,"delta":{"type":"input_json_delta","partial_json":" \"San Francisco\""}}
data: {"type":"content_block_delta","index":1,"delta":{"type":"input_json_delta","partial_json":", \"unit\": \"fahrenheit\"}"}}

event: content_block_stop
data: {"type":"content_block_stop","index":1}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"tool_use"},"usage":{"output_tokens":89}}
```

### 6.4 扩展思考的流式传输

启用 extended thinking 时，流中会先出现 thinking 块，再是 text 块：

```
event: message_start
data: {...}

event: content_block_start           # index=0, type="thinking"
data: {"type":"content_block_start","index":0,"content_block":{"type":"thinking","thinking":"","signature":""}}

event: content_block_delta × N       # 思考过程
data: {"type":"content_block_delta","index":0,"delta":{"type":"thinking_delta","thinking":"I need to find the GCD..."}}

event: content_block_delta           # 签名（验证完整性）
data: {"type":"content_block_delta","index":0,"delta":{"type":"signature_delta","signature":"EqQB..."}}

event: content_block_stop
data: {"type":"content_block_stop","index":0}

event: content_block_start           # index=1, type="text"
data: {...}

event: content_block_delta × N       # 最终回答
data: {"type":"content_block_delta","index":1,"delta":{"type":"text_delta","text":"The answer is 21."}}

event: content_block_stop
data: ...

event: message_delta → message_stop
```

---

## 7. 错误处理协议

### 7.1 HTTP 错误码映射

| HTTP 状态码 | Error Type | 说明 |
|-------------|-----------|------|
| **400** | `invalid_request_error` | 请求格式或内容有问题 |
| **401** | `authentication_error` | API Key 问题 |
| **402** | `billing_error` | 账单或支付问题 |
| **403** | `permission_error` | API Key 无权限访问该资源 |
| **404** | `not_found_error` | 请求的资源未找到 |
| **413** | `request_too_large` | 请求超过最大字节数 |
| **429** | `rate_limit_error` | 达到速率限制 |
| **500** | `api_error` | Anthropic 内部意外错误 |
| **504** | `timeout_error` | 请求处理超时 |
| **529** | `overloaded_error` | API 暂时过载 |

### 7.2 错误响应格式

所有错误都以 JSON 格式返回：

```json
{
  "type": "error",
  "error": {
    "type": "not_found_error",
    "message": "The requested resource could not be found."
  },
  "request_id": "req_011CSHoEeqs5C35K2UUqR7Fy"
}
```

### 7.3 SSE 流中的错误

在 SSE 流式响应中，错误可能在返回 HTTP 200 之后发生，此时通过 error 事件传递：

```
HTTP/1.1 200 OK
Content-Type: text/event-stream

event: message_start
data: {...}

event: content_block_delta
data: {...}

event: error
data: {"type":"error","error":{"type":"overloaded_error","message":"Overloaded"}}
```

### 7.4 错误恢复策略

对于被中断的流式请求，可以从中断处恢复：

**Claude 4.5 及更早版本:**
1. 保存已接收的部分响应
2. 构建续传请求，将部分 assistant response 作为新消息的开始
3. 继续流式传输

**Claude 4.6:**
添加一条 user message 指示模型从断点继续：

```json
{
  "role": "user",
  "content": "Your previous response was interrupted and ended with [previous_response]. Continue from where you left off."
}
```

### 7.5 最佳实践

1. **使用 SDK 的内置功能**: 利用 SDK 的消息累积和错误处理能力
2. **注意内容类型**: 消息可能包含多种内容块（text, tool_use, thinking）
3. **工具调用和扩展思考块不能部分恢复**: 只能从最近的 text 块恢复流式传输

---

## 8. 速率限制与配额管理

### 8.1 令牌桶算法

API 使用令牌桶算法进行速率限制：

```mermaid
flowchart LR
    subgraph Bucket ["Token Bucket"]
        direction TB
        Tokens["●●●●● (tokens)"]
        Fill["以恒定速率填充"]
        Consume["每个请求消耗 tokens"]
    end

    Fill --> Tokens
    Tokens --> Consume
    style Tokens fill:#bbdefb
```

### 8.2 限制层级

| 层级 | RPM | TPM | 月度消费限额 |
|------|-----|-----|-------------|
| Tier 1 | 50 | 20,000 | $150 |
| Tier 2 | 100 | 50,000 | $500 |
| Tier 3 | 200 | 100,000 | $2,000 |
| Tier 4+ | 更高 | 更高 | 无限 |

随着使用量增加自动升级层级。联系销售可获得 Priority Tier。

### 8.3 529 错误的特殊情况

- 在高流量期间可能出现
- 组织使用量急剧增加时也可能遇到 429 错误（加速限制）
- **建议**: 逐渐增加流量，保持一致的使用模式

---

## 9. SDK 集成模式

### 9.1 Python SDK

```python
from anthropic import Anthropic

client = Anthropic()  # 自动读取 ANTHROPIC_API_KEY 环境变量

# 同步调用
message = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello, Claude"}],
)

# 流式调用 - 逐行处理
with client.messages.stream(
    model="claude-opus-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}],
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)

# 流式调用 - 获取完整消息（推荐用于大输出）
with client.messages.stream(
    model="claude-opus-4-6",
    max_tokens=128000,
    messages=[{"role": "user", "content": "Write detailed analysis..."}],
) as stream:
    message = stream.get_final_message()
    print(message.content[0].text)
```

### 9.2 TypeScript/Node.js SDK

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

// 流式处理
const stream = client.messages.stream({
  model: "claude-opus-4-6",
  max_tokens: 1024,
  messages: [{ role: "user", content: "Hello" }],
});

stream.on("text", (text) => process.stdout.write(text));

// 获取最终消息
const message = await stream.finalMessage();
console.log(message);
```

### 9.3 其他语言 SDK

| 语言 | 安装方式 |
|------|---------|
| Java | Maven/Gradle: `com.anthropic:client-sdk` |
| Go | `go get github.com/anthropics/anthropic-sdk-go` |
| C# | NuGet: `Anthropic.SDK` |
| Ruby | `gem install anthropic` |
| PHP | `composer require anthropic/anthropic-php` |

所有官方 SDK 都提供：
- ✅ 自动头部管理（x-api-key, anthropic-version, content-type）
- ✅ 类型安全的请求和响应处理
- ✅ 内置重试逻辑和错误处理
- ✅ 流式支持
- ✅ 请求超时和连接管理

---

## 10. 安全最佳实践

### 10.1 API Key 管理

```bash
# ❌ 不要硬编码在代码中
api_key = "sk-ant-api03-xxx"

# ✅ 使用环境变量
export ANTHROPIC_API_KEY="sk-ant-api03-xxx"

# ✅ 使用 .env 文件（不提交到 Git）
echo "ANTHROPIC_API_KEY=sk-ant-api03-xxx" > .env
```

### 10.2 长时间运行的请求

对于超过 10 分钟的长时间运行请求：

| 方案 | 适用场景 | 优势 |
|------|---------|------|
| Streaming Messages API | 实时交互 | 保持连接活跃，减少超时 |
| Message Batches API | 异步批量处理 | 不需要持续网络连接，支持轮询结果 |
| TCP Keep-alive | 直接集成 | 减少空闲连接超时的风险 |

### 10.3 数据驻留控制

可选的 `inference_geo` 参数用于指定模型推理位置：

```json
{
  "model": "claude-opus-4-6",
  "metadata": {
    "user_id": "user-123"
  },
  "inference_geo": {
    "region": "eu"  // 或 "us"
  }
}
```

---

## 附录 A: 完整 curl 示例集合

### A1: 基础非流式请求

```bash
curl https://api.anthropic.com/v1/messages \
  --header "x-api-key: $ANTHROPIC_API_KEY" \
  --header "anthropic-version: 2023-06-01" \
  --header "content-type: application/json" \
  --data '{
    "model": "claude-opus-4-6",
    "max_tokens": 1024,
    "messages": [
      {"role": "user", "content": "Hello, Claude"}
    ]
  }'
```

### A2: 多轮对话

```bash
curl https://api.anthropic.com/v1/messages \
  --header "x-api-key: $ANTHROPIC_API_KEY" \
  --header "anthropic-version: 2023-06-01" \
  --header "content-type: application/json" \
  --data '{
    "model": "claude-opus-4-6",
    "max_tokens": 1024,
    "system": "You are a helpful coding assistant.",
    "messages": [
      {"role": "user", "content": "What is Python?"},
      {"role": "assistant", "content": "Python is a programming language..."},
      {"role": "user", "content": "Can you give me an example?"}
    ]
  }'
```

### A3: 带工具定义的请求

```bash
curl https://api.anthropic.com/v1/messages \
  --header "content-type: application/json" \
  --header "x-api-key: $ANTHROPIC_API_KEY" \
  --header "anthropic-version: 2023-06-01" \
  --data '{
    "model": "claude-opus-4-6",
    "max_tokens": 1024,
    "tools": [{
      "name": "get_weather",
      "description": "Get weather for a location",
      "input_schema": {
        "type": "object",
        "properties": {
          "location": {"type": "string"}
        },
        "required": ["location"]
      }
    }],
    "messages": [
      {"role": "user", "content": "Weather in SF?"}
    ]
  }'
```

### A4: 扩展思考请求

```bash
curl https://api.anthropic.com/v1/messages \
  --header "x-api-key: $ANTHROPIC_API_KEY" \
  --header "anthropic-version: 2023-06-01" \
  --header "content-type: application/json" \
  --data '{
    "model": "claude-opus-4-6",
    "max_tokens": 20000,
    "stream": true,
    "thinking": {
      "type": "enabled",
      "budget_tokens": 16000
    },
    "messages": [
      {"role": "user", "content": "What is the GCD of 1071 and 462?"}
    ]
  }'
```

---

## 附录 B: 事件流可视化

### B1: 文本生成流

```mermaid
sequenceDiagram
    participant C as Client
    participant A as Anthropic API

    C->>A: POST /v1/messages<br/>{stream: true}
    A-->>C: HTTP 200<br/>Content-Type: text/event-stream

    A-->>C: event: message_start
    A-->>C: event: content_block_start<br/>(index=0, type=text)
    A-->>C: event: content_block_delta<br/>(text="Hello")
    A-->>C: event: content_block_delta<br/>(text="!")
    A-->>C: event: content_block_stop
    A-->>C: event: message_delta<br/>(stop_reason=end_turn)
    A-->>C: event: message_stop
```

### B2: 工具调用流

```mermaid
sequenceDiagram
    participant C as Client
    participant A as Anthropic API

    C->>A: POST /v1/messages<br/>{stream: true, tools:[...]}

    A-->>C: HTTP 200

    Note right of A: Block [0]: Text
    A-->>C: message_start
    A-->>C: content_block_start [0]<br/>(type=text)
    A-->>C: content_block_delta ×N<br/>("Okay let me check...")
    A-->>C: content_block_stop [0]

    Note right of A: Block [1]: Tool Use
    A-->>C: content_block_start [1]<br/>(type=tool_use)
    A-->>C: content_block_delta ×N<br/>(partial JSON input)
    A-->>C: content_block_stop [1]

    A-->>C: message_delta<br/>(stop_reason=tool_use)
    A-->>C: message_stop
```

---

## 参考资源

- **[API Overview](https://platform.claude.com/docs/en/api/overview)** - API 总览
- **[Streaming Messages](https://platform.claude.com/docs/en/build-with-claude/streaming)** - 流式传输文档
- **[Errors Reference](https://platform.claude.com/docs/en/api/errors)** - 错误参考
- **[Rate Limits](https://platform.claude.com/docs/en/api/rate-limits)** - 速率限制详情
- **[Client SDKs](https://platform.claude.com/docs/en/api/client-sdks)** - 官方 SDK 列表

---

*最后更新: 2026年*
