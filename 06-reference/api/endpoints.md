# API 端点参考

> Dify REST API 完整端点参考

## 认证

所有 API 请求需要在 HTTP Header 中携带 API Key：

```bash
Authorization: Bearer YOUR_API_KEY
```

## Base URL

```
https://api.dify.ai/v1
```

## 应用 API

### 聊天消息

#### 发送聊天消息

```
POST /chat-messages
```

**请求体**:

```json
{
  "query": "用户查询内容",
  "user": "user_123",
  "conversation_id": "可选的会话ID",
  "response_mode": "blocking",
  "files": [],
  "retrieval_model": {
    "query": "{{#sys.query#}}",
    "top_k": 3,
    "score_threshold": 0.7
  },
  "extras": {}
}
```

**响应 (blocking)**:

```json
{
  "message_id": "msg_abc123",
  "conversation_id": "conv_xyz789",
  "answer": "AI 回复内容",
  "metadata": {
    "usage": {
      "prompt_tokens": 100,
      "completion_tokens": 50,
      "total_tokens": 150
    },
    "retrieval_resources": [
      {
        "position": 1,
        "document_id": "doc_123",
        "document_name": "文档名称",
        "content": "相关内容片段..."
      }
    ]
  }
}
```

#### 流式聊天消息

```
POST /chat-messages
```

设置 `response_mode: streaming` 获取流式响应。

**SSE 事件**:

```
event: message
data: {"event":"message","task_id":"...","message_id":"...","answer":"首","created_at":1234567890}

event: message_end
data: {"event":"message_end","message_id":"...","metadata":{...}}
```

### 文本补全

```
POST /completions
```

**请求体**:

```json
{
  "prompt": "用户提示词",
  "user": "user_123",
  "response_mode": "blocking",
  "model": "gpt-3.5-turbo",
  "temperature": 0.7,
  "max_tokens": 1000
}
```

### 获取应用参数

```
GET /parameters
```

**响应**:

```json
{
  "user_input_form": [
    {
      "text_input": {
        "label": "Query",
        "variable": "query",
        "required": true,
        "default": ""
      }
    }
  ]
}
```

### 文件上传

```
POST /file-upload
```

**请求体** (multipart/form-data):

| 字段 | 类型 | 说明 |
|------|------|------|
| file | File | 上传的文件 |
| user | String | 用户标识 |

**响应**:

```json
{
  "id": "file_abc123",
  "name": "document.pdf",
  "size": 1024000,
  "extension": "pdf"
}
```

## 工作流 API

### 运行工作流

```
POST /workflows/run
```

**请求体**:

```json
{
  "inputs": {
    "query": "输入参数"
  },
  "user": "user_123",
  "response_mode": "blocking",
  "thread_id": "可选的线程ID"
}
```

**响应**:

```json
{
  "task_id": "task_abc123",
  "workflow_id": "wf_xyz789",
  "inputs": {
    "query": "输入参数"
  },
  "outputs": {
    "result": "输出结果"
  },
  "status": "succeeded",
  "elapsed": 2.5,
  "total_tokens": 150
}
```

### 获取任务状态

```
GET /workflows/tasks/{task_id}
```

### 停止运行

```
POST /workflows/{task_id}/stop
```

## 数据集 API

### 列出数据集

```
GET /datasets
```

**查询参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| page | int | 页码 |
| limit | int | 每页数量 |

**响应**:

```json
{
  "data": [
    {
      "id": "dataset_123",
      "name": "知识库名称",
      "description": "描述",
      "document_count": 10,
      "word_count": 50000,
      "created_from": "api",
      "created_at": 1234567890
    }
  ],
  "total": 10
}
```

### 创建数据集

```
POST /datasets
```

**请求体**:

```json
{
  "name": "知识库名称",
  "description": "描述",
  "indexing_technique": "high_quality",
  "permission": "only_me"
}
```

### 添加文档

```
POST /datasets/{dataset_id}/documents
```

**请求体**:

```json
{
  "name": "文档名称",
  "text": "文档内容",
  "indexing_technique": "high_quality"
}
```

### 检索知识

```
POST /datasets/{dataset_id}/retrieve
```

**请求体**:

```json
{
  "query": "检索查询",
  "top_k": 3,
  "score_threshold": 0.7
}
```

## 错误响应

### 错误格式

```json
{
  "code": "invalid_api_key",
  "message": "错误描述",
  "status_code": 401
}
```

### 错误码列表

| 错误码 | HTTP 状态码 | 说明 |
|--------|-------------|------|
| `invalid_api_key` | 401 | API Key 无效 |
| `api_key_missing` | 401 | 缺少 API Key |
| `invalid_parameter` | 400 | 参数错误 |
| `invalid_message_type` | 400 | 消息类型错误 |
| `quota_exceeded` | 403 | 配额超出 |
| `rate_limit_exceeded` | 429 | 速率限制 |
| `workflow_not_found` | 404 | 工作流未找到 |
| `dataset_not_found` | 404 | 数据集未找到 |
| `internal_server_error` | 500 | 服务器错误 |
| `service_unavailable` | 503 | 服务不可用 |

## 速率限制

| 端点 | 限制 |
|------|------|
| `/v1/chat-messages` | 60/分钟 |
| `/v1/completions` | 60/分钟 |
| `/v1/workflows/run` | 120/分钟 |
| `/v1/datasets/*` | 30/分钟 |

超过限制返回 `429 Too Many Requests`，响应头中包含：
- `X-RateLimit-Limit`: 限制次数
- `X-RateLimit-Remaining`: 剩余次数
- `X-RateLimit-Reset`: 重置时间戳
