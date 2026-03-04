# Webhook 集成

> Dify Webhook 配置和使用指南

## 概述

Webhook 允许 Dify 与外部系统进行事件驱动集成：
- 工作流事件回调
- 外部触发工作流
- 实时通知推送

## 工作流 Webhook 触发

### 配置 Webhook 节点

在 Dify 工作流中配置 Webhook Trigger 节点：

```yaml
nodes:
  - id: webhook-trigger
    data:
      type: webhook-trigger
      method: POST
      path: /webhook/my-workflow
      parameters:
        - name: data
          type: string
          required: true
        - name: user_id
          type: string
          required: false
```

### 触发工作流

```bash
curl -X POST 'https://dify.example.com/webhook/my-workflow' \
  -H 'Content-Type: application/json' \
  -d '{
    "data": "触发数据",
    "user_id": "user123"
  }'
```

## Webhook 回调

### 配置回调端点

```python
from flask import Flask, request, jsonify
import hmac
import hashlib

app = Flask(__name__)
SECRET_KEY = "your-webhook-secret"

def verify_signature(payload: bytes, signature: str) -> bool:
    """验证 Webhook 签名"""
    expected = hmac.new(
        SECRET_KEY.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)

@app.route('/webhook/callback', methods=['POST'])
def handle_callback():
    # 验证签名
    signature = request.headers.get('X-Dify-Signature')
    if signature:
        payload = request.get_data()
        if not verify_signature(payload, signature):
            return jsonify({"error": "Invalid signature"}), 401

    # 处理事件
    payload = request.json
    event = payload.get('event')

    if event == 'workflow.started':
        handle_workflow_started(payload)
    elif event == 'workflow.finished':
        handle_workflow_finished(payload)
    elif event == 'workflow.failed':
        handle_workflow_failed(payload)

    return jsonify({"status": "ok"})

def handle_workflow_finished(payload):
    data = payload.get('data', {})
    workflow_id = data.get('workflow_id')
    output = data.get('output', {})

    # 处理工作流输出
    print(f"Workflow {workflow_id} finished with output: {output}")
```

### 事件类型

| 事件 | 说明 | 数据结构 |
|------|------|----------|
| `workflow.started` | 工作流启动 | workflow_id, inputs, started_at |
| `workflow.running` | 工作流运行中 | workflow_id, node_id,进度 |
| `workflow.finished` | 工作流完成 | workflow_id, output, elapsed |
| `workflow.failed` | 工作流失败 | workflow_id, error, elapsed |
| `node.started` | 节点启动 | workflow_id, node_id |
| `node.finished` | 节点完成 | workflow_id, node_id, output |

### 事件数据格式

```json
{
  "event": "workflow.finished",
  "timestamp": 1234567890,
  "data": {
    "workflow_id": "wf_xxx",
    "conversation_id": "conv_xxx",
    "inputs": {
      "query": "用户查询"
    },
    "output": {
      "result": "输出内容"
    },
    "elapsed": 2.5,
    "total_tokens": 150
  }
}
```

## 安全配置

### 签名验证

```python
import hmac
import hashlib
import time

def generate_signature(payload: str, secret: str, timestamp: str) -> str:
    """生成 Webhook 签名"""
    message = f"{timestamp}.{payload}"
    signature = hmac.new(
        secret.encode(),
        message.encode(),
        hashlib.sha256
    ).hexdigest()
    return signature

# 验证签名
def verify_webhook(request, secret: str):
    timestamp = request.headers.get('X-Dify-Timestamp', '')
    signature = request.headers.get('X-Dify-Signature', '')

    # 检查时间戳（5分钟过期）
    if abs(time.time() - int(timestamp)) > 300:
        return False

    # 验证签名
    payload = request.get_data(as_text=True)
    expected = generate_signature(payload, secret, timestamp)

    return hmac.compare_digest(signature, expected)
```

### IP 白名单

在生产环境中，建议配置 IP 白名单：

```yaml
# Nginx 配置示例
location /webhook/ {
    # 只允许 Dify 服务器 IP
    allow 1.2.3.4;
    allow 5.6.7.8;
    deny all;

    # 代理到应用
    proxy_pass http://backend;
}
```

## 重试机制

Dify Webhook 默认重试 3 次，建议在回调端点实现幂等处理：

```python
@app.route('/webhook/callback', methods=['POST'])
def handle_callback():
    payload = request.json

    # 提取事件 ID（用于幂等）
    event_id = payload.get('event_id')

    # 检查是否已处理
    if is_processed(event_id):
        return jsonify({"status": "ok", "message": "already processed"})

    # 处理事件
    process_event(payload)

    # 标记为已处理
    mark_processed(event_id)

    return jsonify({"status": "ok"})
```

## 完整示例

```python
from flask import Flask, request, jsonify
import hmac
import hashlib
import json

app = Flask(__name__)
WEBHOOK_SECRET = "your-secret-key"

def verify_signature(payload: bytes, signature: str) -> bool:
    expected = hmac.new(
        WEBHOOK_SECRET.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)

@app.route('/webhook/dify', methods=['POST'])
def dify_webhook():
    # 验证签名
    signature = request.headers.get('X-Dify-Signature')
    if signature:
        payload = request.get_data()
        if not verify_signature(payload, signature):
            return jsonify({"error": "Invalid signature"}), 401

    # 解析事件
    payload = request.json
    event = payload.get('event')
    data = payload.get('data', {})

    if event == 'workflow.finished':
        # 处理工作流完成
        output = data.get('output', {})

        # 发送到通知系统
        send_notification({
            "type": "workflow_completed",
            "workflow_id": data.get('workflow_id'),
            "output": output
        })

    elif event == 'workflow.failed':
        # 处理工作流失败
        send_alert({
            "type": "workflow_failed",
            "workflow_id": data.get('workflow_id'),
            "error": data.get('error')
        })

    return jsonify({"received": True})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```
