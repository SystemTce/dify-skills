# 流式响应

> Dify Server-Sent Events (SSE) 流式输出指南

## 概述

Dify 支持通过 Server-Sent Events (SSE) 实现实时流式输出，适用于：
- 长文本生成
- 实时对话
- 进度反馈
- 长时间运行的工作流

## 基本用法

### 请求流式响应

```python
import requests

def stream_chat(api_key: str, query: str, user: str):
    """发送流式聊天请求"""
    response = requests.post(
        "https://api.dify.ai/v1/chat-messages",
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json"
        },
        json={
            "query": query,
            "user": user,
            "response_mode": "streaming"
        },
        stream=True
    )

    for line in response.iter_lines():
        if line:
            data = line.decode('utf-8')
            if data.startswith('data: '):
                yield data[6:]  # 去掉 'data: ' 前缀

# 使用示例
for event_data in stream_chat("app-xxx", "讲个故事", "user123"):
    event = json.loads(event_data)
    print(event)
```

### 解析事件

```python
import json

def process_stream(response):
    """处理流式响应"""
    for line in response.iter_lines():
        if not line:
            continue

        data = line.decode('utf-8')
        if not data.startswith('data: '):
            continue

        try:
            event = json.loads(data[6:])
            event_type = event.get('event')

            if event_type == 'message':
                # 处理消息
                answer = event.get('data', {}).get('answer', '')
                print(answer, end='', flush=True)

            elif event_type == 'message_end':
                # 消息结束
                metadata = event.get('data', {}).get('metadata', {})
                print(f"\n[完成] Tokens: {metadata.get('usage', {}).get('total_tokens')}")

            elif event_type == 'error':
                # 处理错误
                error = event.get('data', {}).get('error', '')
                print(f"\n[错误] {error}")

        except json.JSONDecodeError:
            continue
```

## 事件类型

| 事件 | 说明 | 数据结构 |
|------|------|----------|
| `message` | 消息片段 | answer, message_id |
| `message_end` | 消息结束 | message_id, metadata |
| `message_file` | 文件消息 | message_id, file |
| `tts_message` | TTS 消息 | audio, message_id |
| `tts_message_end` | TTS 结束 | message_id |
| `error` | 错误 | error, message_id |
| `ping` | 心跳 | - |

## 完整示例

### Python 客户端

```python
import requests
import json
import sseclient
import time

class DifyStreamClient:
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = "https://api.dify.ai/v1"

    def stream_chat(self, query: str, user: str, **kwargs):
        """流式聊天"""
        response = requests.post(
            f"{self.base_url}/chat-messages",
            headers={
                "Authorization": f"Bearer {self.api_key}",
                "Content-Type": "application/json"
            },
            json={
                "query": query,
                "user": user,
                "response_mode": "streaming",
                **kwargs
            },
            stream=True,
            timeout=300
        )

        client = sseclient.SSEClient(response)
        return client

    def chat(self, query: str, user: str):
        """流式聊天（简化版）"""
        response = self.stream_chat(query, user)

        result = ""
        for event in response.events():
            if event.event == 'message':
                data = json.loads(event.data)
                answer = data.get('data', {}).get('answer', '')
                result += answer
                yield answer

            elif event.event == 'message_end':
                yield '\n[完成]'

            elif event.event == 'error':
                yield f"\n[错误] {event.data}"

# 使用示例
client = DifyStreamClient("app-xxx")
for chunk in client.chat("给我讲一个关于AI的故事", "user123"):
    print(chunk, end='', flush=True)
```

### JavaScript 客户端

```javascript
async function streamChat(query, user) {
  const response = await fetch('https://api.dify.ai/v1/chat-messages', {
    method: 'POST',
    headers: {
      'Authorization': 'Bearer YOUR_API_KEY',
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      query,
      user,
      response_mode: 'streaming'
    })
  });

  const reader = response.body.getReader();
  const decoder = new TextDecoder();

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    const text = decoder.decode(value);
    const lines = text.split('\n');

    for (const line of lines) {
      if (line.startsWith('data: ')) {
        const data = JSON.parse(line.slice(6));

        if (data.event === 'message') {
          process.stdout.write(data.data.answer);
        } else if (data.event === 'message_end') {
          console.log('\n[完成]');
        }
      }
    }
  }
}

streamChat('你好', 'user123');
```

### 命令行工具

```bash
# 使用 curl 接收流式响应
curl -N -X POST 'https://api.dify.ai/v1/chat-messages' \
  -H 'Authorization: Bearer YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "讲一个笑话",
    "user": "user123",
    "response_mode": "streaming"
  }'
```

## 工作流流式输出

### 配置工作流

```yaml
nodes:
  - id: start
    data:
      type: start
      variables:
        - name: query
          type: text

  - id: llm
    data:
      type: llm
      model:
        provider: openai
        name: gpt-4
      prompt_template:
        - role: user
          text: "{{#start.query#}}"

  - id: answer
    data:
      type: answer
      answer: "{{#llm.text#}}"
      stream: true  # 启用流式输出
```

### Python 工作流客户端

```python
def stream_workflow(api_key: str, inputs: dict, user: str):
    """流式执行工作流"""
    response = requests.post(
        "https://api.dify.ai/v1/workflows/run",
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json"
        },
        json={
            "inputs": inputs,
            "user": user,
            "response_mode": "streaming"
        },
        stream=True
    )

    for line in response.iter_lines():
        if line:
            data = line.decode('utf-8')
            if data.startswith('data: '):
                yield json.loads(data[6:])

# 使用示例
for event in stream_workflow("app-xxx", {"query": "测试"}, "user123"):
    if event.get('event') == 'node_finished':
        output = event.get('data', {}).get('output', {})
        print(output)
```

## 错误处理

```python
def stream_with_retry(api_key: str, query: str, user: str, max_retries=3):
    """带重试的流式请求"""
    for attempt in range(max_retries):
        try:
            response = requests.post(
                "https://api.dify.ai/v1/chat-messages",
                headers={"Authorization": f"Bearer {api_key}"},
                json={
                    "query": query,
                    "user": user,
                    "response_mode": "streaming"
                },
                stream=True,
                timeout=300
            )

            if response.status_code == 200:
                return response
            elif response.status_code == 429:
                wait = 2 ** attempt
                print(f"Rate limited, waiting {wait}s...")
                time.sleep(wait)
            else:
                raise Exception(f"HTTP {response.status_code}")

        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                raise
            print(f"Error: {e}, retrying...")
            time.sleep(2 ** attempt)
```

## 性能优化

### 连接复用

```python
import requests

# 创建持久连接会话
session = requests.Session()
session.headers.update({"Authorization": f"Bearer {API_KEY}"})

def stream_with_session(query: str, user: str):
    """使用会话复用"""
    response = session.post(
        "https://api.dify.ai/v1/chat-messages",
        json={
            "query": query,
            "user": user,
            "response_mode": "streaming"
        },
        stream=True
    )
    return response
```

### 缓冲策略

```python
class BufferedStream:
    def __init__(self, buffer_size: int = 10):
        self.buffer = []
        self.buffer_size = buffer_size

    def add(self, event: dict):
        self.buffer.append(event)
        if len(self.buffer) >= self.buffer_size:
            self.flush()

    def flush(self):
        # 处理缓冲的事件
        for event in self.buffer:
            process_event(event)
        self.buffer.clear()
```
