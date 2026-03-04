# 从 API 到插件迁移指南

## 迁移策略

### 1. API 包装模式

将 REST API 包装为 Dify Tool 插件。

**适用场景**：
- 简单的 REST API
- 无需复杂状态管理
- 请求-响应模式

**示例**：Google Translate API → Tool 插件

### 2. SDK 集成模式

使用第三方 Python SDK 集成服务。

**适用场景**：
- 服务提供官方 Python SDK
- 需要复杂的 API 交互
- 需要处理认证和会话

**示例**：Anthropic SDK → Model 插件

### 3. OAuth 迁移模式

实现 OAuth 2.0 授权流程。

**适用场景**：
- 需要用户授权
- 访问用户数据
- 第三方服务集成

**示例**：Google Contacts API → OAuth Tool 插件

### 4. 批量操作模式

使用流式响应处理大数据。

**适用场景**：
- 大量数据处理
- 长时间运行的操作
- 需要实时反馈

## 迁移步骤

### 步骤 1：分析现有 API

```python
# 现有 API 调用
import requests

def translate_text(text, target_lang):
    response = requests.post(
        "https://api.example.com/translate",
        json={"text": text, "target": target_lang},
        headers={"Authorization": f"Bearer {api_key}"}
    )
    return response.json()["translated_text"]
```

**分析要点**：
- API 端点和方法
- 请求参数和格式
- 认证方式（API Key / OAuth）
- 响应格式和错误处理
- 速率限制和重试策略

### 步骤 2：选择插件类型

| API 类型 | 插件类型 | 示例 |
|---------|---------|------|
| REST API | Tool | Google Translate |
| LLM API | Model | Anthropic Claude |
| 存储 API | Datasource | AWS S3 |
| 平台 API | Extension | Slack |

### 步骤 3：创建插件项目

```bash
dify plugin init
? Select plugin type: Tool
? Plugin name: my_api_wrapper
? Author: your_name
```

### 步骤 4：实现 API 调用

```python
from typing import Any, Generator
from dify_plugin.entities.tool import ToolInvokeMessage
from dify_plugin import Tool
import requests


class APIWrapperTool(Tool):
    def _invoke(
        self, tool_parameters: dict[str, Any]
    ) -> Generator[ToolInvokeMessage, None, None]:
        # 获取参数
        text = tool_parameters.get("text", "")
        target_lang = tool_parameters.get("target_lang", "en")

        # 获取凭证
        credentials = self.runtime.credentials
        api_key = credentials.get("api_key")

        try:
            # 调用 API
            response = requests.post(
                "https://api.example.com/translate",
                json={"text": text, "target": target_lang},
                headers={"Authorization": f"Bearer {api_key}"},
                timeout=30
            )
            response.raise_for_status()

            # 解析响应
            result = response.json()["translated_text"]

            # 返回结果
            yield self.create_text_message(result)

        except requests.exceptions.Timeout:
            yield self.create_text_message("Error: Request timeout")
        except requests.exceptions.HTTPError as e:
            yield self.create_text_message(f"API Error: {e.response.status_code}")
        except Exception as e:
            yield self.create_text_message(f"Error: {str(e)}")
```

### 步骤 5：配置认证

#### API Key 认证

```yaml
# provider.yaml
credentials_schema:
  - name: api_key
    type: secret-input
    required: true
    label:
      en_US: API Key
    help:
      en_US: Get your API key from the service provider
    url: https://console.example.com/api-keys
```

#### OAuth 认证

```yaml
# provider.yaml
oauth_schema:
  client_schema:
    - name: client_id
      type: secret-input
      required: true
    - name: client_secret
      type: secret-input
      required: true
  credentials_schema:
    - name: access_token
      type: secret-input
    - name: refresh_token
      type: secret-input
```

### 步骤 6：添加错误处理

```python
def _invoke(self, tool_parameters):
    try:
        # API 调用
        response = requests.post(url, json=data, headers=headers)

        # 处理不同的错误状态
        if response.status_code == 400:
            yield self.create_text_message("Error: Invalid parameters")
        elif response.status_code == 401:
            yield self.create_text_message("Error: Invalid API key")
        elif response.status_code == 429:
            yield self.create_text_message("Error: Rate limit exceeded")
        elif response.status_code == 500:
            yield self.create_text_message("Error: Server error")
        else:
            response.raise_for_status()
            result = response.json()
            yield self.create_text_message(result)

    except requests.exceptions.ConnectionError:
        yield self.create_text_message("Error: Connection failed")
    except requests.exceptions.Timeout:
        yield self.create_text_message("Error: Request timeout")
    except Exception as e:
        yield self.create_text_message(f"Error: {str(e)}")
```

### 步骤 7：添加重试机制

```python
from tenacity import retry, stop_after_attempt, wait_exponential


class APIWrapperTool(Tool):
    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=2, max=10)
    )
    def _call_api(self, url, data, headers):
        response = requests.post(url, json=data, headers=headers)
        response.raise_for_status()
        return response.json()

    def _invoke(self, tool_parameters):
        try:
            result = self._call_api(url, data, headers)
            yield self.create_text_message(result)
        except Exception as e:
            yield self.create_text_message(f"Error: {str(e)}")
```

### 步骤 8：编写测试

```python
# tests/test_api_wrapper.py
import pytest
from unittest.mock import Mock, patch
from tools.api_wrapper import APIWrapperTool


def test_api_wrapper_success():
    tool = APIWrapperTool()

    with patch('requests.post') as mock_post:
        mock_response = Mock()
        mock_response.json.return_value = {"translated_text": "Hello"}
        mock_response.status_code = 200
        mock_post.return_value = mock_response

        result = list(tool.invoke(
            tool_parameters={"text": "你好", "target_lang": "en"}
        ))

        assert len(result) == 1
        assert "Hello" in result[0].message


def test_api_wrapper_error():
    tool = APIWrapperTool()

    with patch('requests.post') as mock_post:
        mock_post.side_effect = requests.exceptions.Timeout()

        result = list(tool.invoke(
            tool_parameters={"text": "你好", "target_lang": "en"}
        ))

        assert "timeout" in result[0].message.lower()
```

### 步骤 9：打包部署

```bash
# 打包插件
dify plugin package

# 生成 my_api_wrapper.difypkg
# 上传到 Dify Marketplace 或私有部署
```

## 实际案例

### 案例 1：REST API → Tool 插件

**原始 API**：
```python
import requests

def search_web(query):
    response = requests.get(
        "https://api.search.com/search",
        params={"q": query},
        headers={"X-API-Key": api_key}
    )
    return response.json()["results"]
```

**迁移后**：
```python
class WebSearchTool(Tool):
    def _invoke(self, tool_parameters):
        query = tool_parameters.get("query", "")
        api_key = self.runtime.credentials.get("api_key")

        response = requests.get(
            "https://api.search.com/search",
            params={"q": query},
            headers={"X-API-Key": api_key}
        )

        results = response.json()["results"]
        yield self.create_json_message(results)
```

### 案例 2：OAuth API → Tool 插件

**原始 API**：
```python
def get_contacts(access_token):
    response = requests.get(
        "https://api.contacts.com/contacts",
        headers={"Authorization": f"Bearer {access_token}"}
    )
    return response.json()["contacts"]
```

**迁移后**：
```python
class ContactsTool(Tool):
    def _invoke(self, tool_parameters):
        access_token = self.runtime.credentials.get("access_token")

        response = requests.get(
            "https://api.contacts.com/contacts",
            headers={"Authorization": f"Bearer {access_token}"}
        )

        if response.status_code == 401:
            raise ToolProviderCredentialValidationError("Token expired")

        contacts = response.json()["contacts"]
        yield self.create_json_message(contacts)
```

### 案例 3：LLM API → Model 插件

**原始 API**：
```python
def call_llm(prompt):
    response = requests.post(
        "https://api.llm.com/chat",
        json={"prompt": prompt},
        headers={"Authorization": f"Bearer {api_key}"}
    )
    return response.json()["response"]
```

**迁移后**：
```python
class LLMModelProvider(ModelProvider):
    def _invoke(self, model, parameters):
        prompt = parameters.get("prompt", "")
        api_key = self.runtime.credentials.get("api_key")

        response = requests.post(
            "https://api.llm.com/chat",
            json={"prompt": prompt, "stream": True},
            headers={"Authorization": f"Bearer {api_key}"},
            stream=True
        )

        for line in response.iter_lines():
            if line:
                chunk = json.loads(line)
                yield chunk["text"]
```

## 迁移检查清单

- [ ] 分析 API 端点和参数
- [ ] 确定认证方式
- [ ] 选择合适的插件类型
- [ ] 创建插件项目
- [ ] 实现 API 调用逻辑
- [ ] 配置认证（API Key / OAuth）
- [ ] 添加错误处理
- [ ] 添加重试机制
- [ ] 编写单元测试
- [ ] 编写集成测试
- [ ] 测试错误场景
- [ ] 优化性能
- [ ] 编写文档
- [ ] 打包部署

## 常见问题

### Q: 如何处理速率限制？

A: 使用 tenacity 库实现指数退避重试：

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
def _call_api(self, url, data):
    response = requests.post(url, json=data)
    if response.status_code == 429:
        raise Exception("Rate limit exceeded")
    return response.json()
```

### Q: 如何处理大文件上传？

A: 使用流式上传：

```python
def _invoke(self, tool_parameters):
    file_path = tool_parameters.get("file_path")

    with open(file_path, 'rb') as f:
        response = requests.post(
            url,
            files={'file': f},
            stream=True
        )

    yield self.create_text_message("Upload complete")
```

### Q: 如何处理异步 API？

A: 使用轮询或 Webhook：

```python
def _invoke(self, tool_parameters):
    # 提交任务
    response = requests.post(url, json=data)
    task_id = response.json()["task_id"]

    # 轮询结果
    while True:
        status_response = requests.get(f"{url}/{task_id}")
        status = status_response.json()["status"]

        if status == "completed":
            result = status_response.json()["result"]
            yield self.create_text_message(result)
            break
        elif status == "failed":
            yield self.create_text_message("Task failed")
            break

        time.sleep(2)
```

## 最佳实践

1. **参数验证**：在调用 API 前验证所有参数
2. **错误处理**：捕获所有可能的异常
3. **重试机制**：对临时错误实现重试
4. **超时设置**：设置合理的请求超时
5. **日志记录**：记录关键操作和错误
6. **测试覆盖**：编写完整的单元测试和集成测试
7. **文档完善**：提供清晰的使用说明
