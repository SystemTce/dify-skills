# Python SDK 参考

> Dify Python SDK 完整 API 参考

## 安装

```bash
pip install dify-client
```

## 快速开始

```python
from dify_client import DifyClient

# 初始化客户端
client = DifyClient(api_key="app-xxx")

# 发送聊天
response = client.chat.messages.create(
    query="你好",
    user="user_123"
)
print(response.answer)
```

## DifyClient

### 初始化

```python
from dify_client import DifyClient

client = DifyClient(
    api_key="app-xxx",           # API Key
    base_url="https://api.dify.ai/v1"  # 基础 URL
)
```

### 属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `chat` | ChatClient | 聊天客户端 |
| `workflows` | WorkflowClient | 工作流客户端 |
| `datasets` | DatasetClient | 数据集客户端 |

## ChatClient

### 聊天消息

```python
# 发送聊天消息（阻塞）
response = client.chat.messages.create(
    query="用户查询",
    user="user_123",
    conversation_id=None,      # 可选，默认新建会话
    response_mode="blocking",  # blocking 或 streaming
    files=[],                 # 可选，上传的文件
    retrieval_model={},        # 可选，检索配置
    extras={}                 # 可选，额外参数
)

# 获取响应
print(response.message_id)
print(response.conversation_id)
print(response.answer)
print(response.metadata)
```

### 流式聊天

```python
# 流式响应
stream = client.chat.messages.create(
    query="讲个故事",
    user="user_123",
    response_mode="streaming"
)

for event in stream:
    if event.event == "message":
        print(event.data.answer, end="")
    elif event.event == "message_end":
        print("\n[完成]")
```

### 会话管理

```python
# 获取会话历史
messages = client.chat.messages.list(
    conversation_id="conv_xxx",
    first_id=None,  # 可选，第一条消息 ID
    limit=20        # 可选，返回数量
)

# 获取会话列表
conversations = client.chat.conversations.list(
    user="user_123",
    limit=20
)
```

## WorkflowClient

### 运行工作流

```python
# 运行工作流（阻塞）
response = client.workflows.run(
    inputs={"query": "测试"},
    user="user_123",
    response_mode="blocking",
    thread_id=None  # 可选
)

print(response.task_id)
print(response.outputs)
print(response.status)
print(response.elapsed)
```

### 流式运行

```python
# 流式执行
stream = client.workflows.run(
    inputs={"query": "测试"},
    user="user_123",
    response_mode="streaming"
)

for event in stream:
    if event.event == "node_finished":
        print(f"节点完成: {event.data.node_id}")
        print(f"输出: {event.data.output}")
```

## DatasetClient

### 数据集操作

```python
# 列出数据集
datasets = client.datasets.list(page=1, limit=10)
for ds in datasets.data:
    print(ds.id, ds.name)

# 创建数据集
dataset = client.datasets.create(
    name="我的知识库",
    description="描述",
    indexing_technique="high_quality"
)
print(dataset.id)
```

### 文档操作

```python
# 添加文档
document = client.datasets.documents.create(
    dataset_id="dataset_xxx",
    name="文档名称",
    text="文档内容...",
    indexing_technique="high_quality"
)
print(document.id)

# 列出文档
documents = client.datasets.documents.list(
    dataset_id="dataset_xxx",
    page=1,
    limit=10
)

# 删除文档
client.datasets.documents.delete(
    dataset_id="dataset_xxx",
    document_id="doc_xxx"
)
```

### 检索

```python
# 检索知识
results = client.datasets.retrieve(
    dataset_id="dataset_xxx",
    query="检索查询",
    top_k=3,
    score_threshold=0.7
)

for r in results:
    print(r.content)
    print(r.score)
```

## 响应类型

### ChatMessage

```python
@dataclass
class ChatMessage:
    message_id: str
    conversation_id: str
    answer: str
    metadata: ChatMetadata
```

### WorkflowRun

```python
@dataclass
class WorkflowRun:
    task_id: str
    workflow_id: str
    inputs: dict
    outputs: dict
    status: str  # succeeded, failed, stopped
    elapsed: float
    total_tokens: int
```

### Dataset

```python
@dataclass
class Dataset:
    id: str
    name: str
    description: str
    document_count: int
    word_count: int
    created_at: int
```

## 错误处理

```python
from dify_client import DifyClient
from dify_client.errors import (
    DifyAPIError,
    InvalidAPIKeyError,
    RateLimitError,
    ValidationError
)

try:
    response = client.chat.messages.create(
        query="测试",
        user="user_123"
    )
except InvalidAPIKeyError as e:
    print(f"API Key 无效: {e}")
except RateLimitError as e:
    print(f"速率限制: {e}")
except ValidationError as e:
    print(f"参数错误: {e}")
except DifyAPIError as e:
    print(f"API 错误: {e.code} - {e.message}")
```

## 完整示例

```python
from dify_client import DifyClient

def main():
    # 初始化
    client = DifyClient(api_key="app-xxx")

    # 1. 简单聊天
    response = client.chat.messages.create(
        query="你好",
        user="user_123"
    )
    print(f"回复: {response.answer}")

    # 2. 带上下文的聊天
    response = client.chat.messages.create(
        query="继续",
        user="user_123",
        conversation_id=response.conversation_id
    )
    print(f"回复: {response.answer}")

    # 3. 运行工作流
    workflow_response = client.workflows.run(
        inputs={"query": "测试工作流"},
        user="user_123"
    )
    print(f"工作流输出: {workflow_response.outputs}")

    # 4. 知识检索
    results = client.datasets.retrieve(
        dataset_id="dataset_xxx",
        query="什么是 Dify",
        top_k=5
    )
    for r in results:
        print(f"[{r.score:.2f}] {r.content[:100]}...")

if __name__ == "__main__":
    main()
```
