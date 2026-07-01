# to-agents 核心协议规范 v1.0.0

## 1. 概述

to-agents 定义了一套与框架无关、传输无关的多智能体通信约定。协议的全部约定分两层：

- **发现层**：通过 `agents.json` 声明服务器上可用的 Agent
- **消息层**：通过统一信封在 Agent 之间传递请求、响应、错误和事件

### 1.1 设计原则

- **唯一固定入口**：仅 `/agents.json` 是硬编码端点，其余一切从它派生
- **框架无关**：不绑定 LangChain、AutoGen、CrewAI 等任何特定框架
- **传输无关**：约定层面独立于 HTTP、WebSocket、gRPC 等传输层
- **渐进式**：简单场景开箱即用，复杂场景按需扩展

---

## 2. 发现协议：`agents.json`

服务器在根路径暴露一个 `agents.json` 文件，作为服务发现的唯一入口。

```
GET /agents.json
```

### 2.1 顶层结构

```json
{
  "version": "1.0.0",
  "agents": [
    {
      "id": "translator",
      "name": "翻译助手",
      "version": "1.2.0",
      "description": "多语言翻译 Agent",
      "capabilities": ["translation", "text-processing"],
      "endpoint": "/agents/translator/message",
      "auth": "bearer",
      "rate_limit": "60/min"
    }
  ]
}
```

### 2.2 字段说明

| 字段 | 层级 | 类型 | 必填 | 说明 |
|------|------|------|------|------|
| `version` | 服务器 | string | ✅ | SemVer 格式，表示 `agents.json` 自身的格式版本 |
| `agents` | 服务器 | array | ✅ | Agent 列表 |
| `agents[].id` | Agent | string | ✅ | 全局唯一标识，纯 ID（不含服务器地址） |
| `agents[].name` | Agent | string | ✅ | 人类可读的名称 |
| `agents[].version` | Agent | string | ✅ | SemVer 格式，该 Agent 的协议版本 |
| `agents[].description` | Agent | string | ❌ | 功能描述 |
| `agents[].capabilities` | Agent | string[] | ✅ | 能力标签列表，用于发现和匹配 |
| `agents[].endpoint` | Agent | string | ✅ | 消息交互端点（相对路径或绝对 URL） |
| `agents[].auth` | Agent | string | ❌ | 认证类型，目前仅支持 `"bearer"`。省略表示无需认证 |
| `agents[].rate_limit` | Agent | string | ❌ | 速率限制提示，如 `"60/min"`。实际限流以 HTTP 响应头为准 |

### 2.3 SSE 端点

SSE（Server-Sent Events）端点为服务器级别统一端点，不在 `agents.json` 中逐 Agent 声明。

约定路径：`/sse`

调用方连接 `/sse` 后可接收该服务器上所有 Agent 的事件推送和流式响应，通过事件中的 `from` 字段区分来源 Agent。

---

## 3. 消息协议

### 3.1 消息信封

所有消息共享统一的外层信封结构：

```json
{
  "message_id": "550e8400-e29b-41d4-a716-446655440000",
  "session_id": "optional-session-uuid",
  "timestamp": "2026-07-01T12:00:00Z",
  "version": "1.2.0",
  "from": "caller-agent-id",
  "to": "translator",
  "type": "request",
  "payload": { }
}
```

### 3.2 字段说明

| 字段 | 必填 | 类型 | 说明 |
|------|------|------|------|
| `message_id` | ✅ | string (UUID) | 消息唯一标识，用于链路追踪和日志关联 |
| `session_id` | ❌ | string (UUID) | 会话标识，仅用于日志关联和调用方自行维护上下文。服务端**不存储**会话状态 |
| `timestamp` | ❌ | string (ISO 8601) | 消息发送时间 |
| `version` | ❌ | string (SemVer) | 调用方使用的协议版本。用于服务端版本协商降级。不填时服务端按自身默认版本处理 |
| `from` | ✅ | string | 发送方 Agent ID（纯标识符，不含服务器地址） |
| `to` | ✅ | string | 接收方 Agent ID（纯标识符，不含服务器地址） |
| `type` | ✅ | enum | 消息类型：`request` / `response` / `error` / `event` |
| `payload` | ✅ | object | 消息体，格式由 Agent 自行定义，协议层不做约束 |

### 3.3 消息类型

#### 3.3.1 `request`

调用方发起请求。`payload` 格式由目标 Agent 的能力定义。

```json
{
  "message_id": "...",
  "from": "caller",
  "to": "translator",
  "type": "request",
  "payload": {
    "text": "Hello, world!",
    "target_lang": "zh"
  }
}
```

#### 3.3.2 `response`

成功响应。`payload` 格式由 Agent 自行定义。

```json
{
  "message_id": "...",
  "from": "translator",
  "to": "caller",
  "type": "response",
  "payload": {
    "translated_text": "你好，世界！"
  }
}
```

#### 3.3.3 `error`

错误响应。`payload` 遵循固定结构：

```json
{
  "message_id": "...",
  "from": "translator",
  "to": "caller",
  "type": "error",
  "payload": {
    "code": "VERSION_MISMATCH",
    "message": "Unsupported protocol version 3.0.0",
    "details": {
      "supported_versions": ["1.0.0", "1.2.0", "2.0.0"]
    }
  }
}
```

| 字段 | 必填 | 说明 |
|------|------|------|
| `code` | ✅ | 机器可读错误码（如 `VERSION_MISMATCH`、`AUTH_REQUIRED`、`AGENT_NOT_FOUND`） |
| `message` | ✅ | 人类可读错误描述 |
| `details` | ❌ | 结构化补充信息 |

#### 3.3.4 `event`

服务端通过 SSE 主动推送的事件。`payload` 格式由 Agent 自行定义。

```json
{
  "message_id": "...",
  "from": "translator",
  "to": "caller",
  "type": "event",
  "payload": {
    "event_type": "progress",
    "progress": 0.5
  }
}
```

---

## 4. 传输层

### 4.1 消息发送

调用方向 Agent 的消息端点发送 HTTP POST 请求：

```
POST {agent.endpoint}
Content-Type: application/json
Authorization: Bearer <token>

{ ... message envelope ... }
```

同步返回对应的 `response` 或 `error` 消息。

### 4.2 SSE（Server-Sent Events）

调用方连接统一 SSE 端点接收事件和流式响应：

```
GET /sse
Authorization: Bearer <token>
```

SSE 事件格式：

```
event: message
data: {"message_id":"...","from":"translator","to":"caller","type":"event","payload":{...}}

event: message
data: {"message_id":"...","from":"translator","to":"caller","type":"response","payload":{"chunk":"你好","done":false}}

event: message
data: {"message_id":"...","from":"translator","to":"caller","type":"response","payload":{"chunk":"，世界！","done":true}}
```

SSE 端点同时承载：
- **事件推送**（`type: "event"`）：服务端主动通知
- **流式响应**（`type: "response"`）：长文本/生成式结果的流式输出，通过 `done` 字段标记结束

### 4.3 速率限制

实际限流状态通过 HTTP 响应头反馈：

```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1719849600
Retry-After: 30
```

`agents.json` 中的 `rate_limit` 字段仅作为发现阶段的静态提示，不作为限流依据。

---

## 5. 版本协商

### 5.1 版本层级

| 层级 | 位置 | 含义 |
|------|------|------|
| 服务器版本 | `agents.json` → `version` | `agents.json` 自身格式版本 |
| Agent 版本 | `agents.json` → `agents[].version` | 单个 Agent 的协议版本 |
| 调用方版本 | 消息信封 → `version` | 调用方期望使用的协议版本 |

### 5.2 协商流程

1. 调用方在消息信封的 `version` 字段声明自身协议版本
2. 服务端收到后检查版本兼容性
3. 若不兼容，服务端**主动降级**到调用方版本处理（如果支持）
4. 若无法降级，返回 `error`（`code: "VERSION_MISMATCH"`），`details` 中附带支持的版本列表

### 5.3 兼容性规则

- `MAJOR` 版本号不同 → 不兼容，需要协商
- `MINOR` / `PATCH` 版本号不同 → 向后兼容，自动容忍

---

## 6. 认证

协议仅定义 Bearer Token 认证方式。Agent 在 `agents.json` 中声明 `"auth": "bearer"` 表示需要认证。

Token 的获取方式（OAuth、手动配置、扫码等）由实现方自行决定，协议不做约束。

```
Authorization: Bearer <token>
```

---

## 7. 幂等性与重试

- 服务端**不做**消息去重
- 同一个 `message_id` 多次发送，每次都被当作新请求独立处理
- 幂等性由 Agent 自身实现保证
- 网络超时后调用方可自行决定是否重试

---

## 8. 设计决策摘要

| 决策 | 选择 |
|------|------|
| 发现入口 | 唯一固定端点 `/agents.json` |
| Agent 列表格式 | 数组 |
| 能力描述 | 简单标签列表 |
| 消息格式 | 统一信封 + 自定义 payload |
| 消息类型 | `request` / `response` / `error` / `event` |
| 必填字段 | `message_id`, `from`, `to`, `type`, `payload` |
| 寻址方式 | 纯 Agent ID，通过 `agents.json` 解析端点 |
| 版本格式 | SemVer，服务器级 + Agent 级两层 |
| 版本协商 | 服务端主动降级 |
| 事件推送 | SSE，服务器级统一端点 `/sse` |
| 流式响应 | 复用 SSE 端点 |
| 会话状态 | 无状态（`session_id` 仅用于日志关联） |
| 认证 | Bearer Token，Token 获取方式由实现方自行决定 |
| 速率限制 | `agents.json` 静态提示 + HTTP 响应头实时状态 |
| 错误格式 | `code` + `message` + `details` |
| 幂等性 | 协议不保证，由 Agent 自行处理 |
