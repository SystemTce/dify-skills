# REST API 集成

> Dify REST API 完整调用指南

## 概述

Dify 提供完整的 RESTful API，支持：
- 聊天对话
- 文本补全
- 工作流执行
- 文件上传
- 应用管理

## 认证

所有 API 请求需要使用 API Key 进行认证：

```bash
curl -X POST 'https://api.dify.ai/v1/chat-messages' \
  -H 'Authorization: Bearer YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "你好",
    "user": "user123"
  }'
```

## 核心端点

### 聊天消息

**端点**: `POST /v1/chat-messages`

```python
import requests

def send_chat_message(api_key: str, query: str, user: str, **kwargs):
    """发送聊天消息"""
    response = requests.post(
        "https://api.dify.ai/v1/chat-messages",
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json"
        },
        json={
            "query": query,
            "user": user,
            **kwargs
        }
    )
    return response.json()
```

**请求参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| query | string | 是 | 用户查询 |
| user | string | 是 | 用户标识 |
| conversation_id | string | 否 | 会话 ID（空为新会话） |
| response_mode | string | 否 | blocking/streaming |
| files | array | 否 | 上传的文件 |

**响应**:

```json
{
  "message_id": "msg_xxx",
  "conversation_id": "conv_xxx",
  "answer": "AI 回复内容",
  "metadata": {
    "usage": {
      "prompt_tokens": 100,
      "completion_tokens": 50,
      "total_tokens": 150
    }
  }
}
```

### 工作流执行

**端点**: `POST /v1/workflows/run`

```python
def run_workflow(api_key: str, inputs: dict, user: str, **kwargs):
    """运行工作流"""
    response = requests.post(
        "https://api.dify.ai/v1/workflows/run",
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json"
        },
        json={
            "inputs": inputs,
            "user": user,
            **kwargs
        }
    )
    return response.json()
```

**请求参数**:

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| inputs | object | 是 | 工作流输入参数 |
| user | string | 是 | 用户标识 |
| response_mode | string | 否 | blocking/streaming |
| thread_id | string | 否 | 线程 ID |

### 文件上传

**端点**: `POST /v1/file-upload`

```python
def upload_file(api_key: str, file_path: str, user: str):
    """上传文件"""
    with open(file_path, 'rb') as f:
        response = requests.post(
            "https://api.dify.ai/v1/file-upload",
            headers={
                "Authorization": f"Bearer {api_key}"
            },
            files={"file": f},
            data={"user": user}
        )
    return response.json()
```

### 应用参数

**端点**: `GET /v1/parameters`

```python
def get_parameters(api_key: str):
    """获取应用参数配置"""
    response = requests.get(
        "https://api.dify.ai/v1/parameters",
        headers={
            "Authorization": f"Bearer {api_key}"
        }
    )
    return response.json()
```

## 错误处理

```python
class DifyAPIError(Exception):
    def __init__(self, status_code: int, message: str):
        self.status_code = status_code
        self.message = message
        super().__init__(f"API Error {status_code}: {message}")

def handle_response(response):
    if response.status_code == 200:
        return response.json()
    elif response.status_code == 401:
        raise DifyAPIError(401, "Invalid API key")
    elif response.status_code == 400:
        raise DifyAPIError(400, response.json().get("message", "Bad request"))
    elif response.status_code == 429:
        raise DifyAPIError(429, "Rate limit exceeded")
    else:
        raise DifyAPIError(response.status_code, "Unknown error")
```

## 完整示例

```python
import requests

class DifyClient:
    BASE_URL = "https://api.dify.ai/v1"

    def __init__(self, api_key: str):
        self.api_key = api_key
        self.headers = {
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json"
        }

    def chat(self, query: str, user: str, **kwargs):
        """发送聊天消息"""
        response = requests.post(
            f"{self.BASE_URL}/chat-messages",
            headers=self.headers,
            json={
                "query": query,
                "user": user,
                **kwargs
            }
        )
        return self._handle_response(response)

    def run_workflow(self, inputs: dict, user: str, **kwargs):
        """运行工作流"""
        response = requests.post(
            f"{self.BASE_URL}/workflows/run",
            headers=self.headers,
            json={
                "inputs": inputs,
                "user": user,
                **kwargs
            }
        )
        return self._handle_response(response)

    def _handle_response(self, response):
        if response.status_code == 200:
            return response.json()
        raise DifyAPIError(response.status_code, response.text)

# 使用示例
client = DifyClient("app-xxxxx")
result = client.chat("你好", "user123")
print(result)
```

## 速率限制

| 端点 | 限制 |
|------|------|
| /v1/chat-messages | 60 请求/分钟 |
| /v1/completions | 60 请求/分钟 |
| /v1/workflows/run | 120 请求/分钟 |

超过限制返回 `429 Too Many Requests`，建议实现重试机制。
