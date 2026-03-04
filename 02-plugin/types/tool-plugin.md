# Tool 插件开发

Tool 插件是 Dify 最常用的插件类型，用于扩展 AI 的能力，如翻译、搜索、数据处理等。

## 快速开始

### 创建 Tool 插件

```bash
dify plugin init
? Select plugin type: Tool
? Plugin name: my_tool
? Author: your_name
```

### 项目结构

```
my_tool/
├── manifest.yaml
├── main.py
├── provider/
│   ├── my_provider.yaml
│   └── my_provider.py
└── tools/
    ├── my_tool.yaml
    └── my_tool.py
```

## Provider 配置

### provider.yaml

```yaml
identity:
  author: your_name
  name: my_provider
  label:
    en_US: My Provider
    zh_Hans: 我的提供者
  description:
    en_US: Provider description
    zh_Hans: 提供者描述
  icon: icon.svg

extra:
  python:
    source: provider/my_provider.py

tools:
  - tools/my_tool.yaml
```

### provider.py

```python
from typing import Any
from dify_plugin import ToolProvider
from dify_plugin.errors.tool import ToolProviderCredentialValidationError


class MyProvider(ToolProvider):
    def _validate_credentials(self, credentials: dict[str, Any]) -> None:
        """验证凭证"""
        api_key = credentials.get("api_key")
        if not api_key:
            raise ToolProviderCredentialValidationError("API Key required")
```

## Tool 实现

### tool.yaml

```yaml
identity:
  name: my_tool
  author: your_name
  label:
    en_US: My Tool
    zh_Hans: 我的工具

description:
  human:
    en_US: Tool description
    zh_Hans: 工具描述
  llm: Tool description for LLM

parameters:
  - name: text
    type: string
    required: true
    label:
      en_US: Input Text
      zh_Hans: 输入文本
    form: llm
```

### tool.py

```python
from typing import Any, Generator
from dify_plugin.entities.tool import ToolInvokeMessage
from dify_plugin import Tool


class MyTool(Tool):
    def _invoke(
        self, tool_parameters: dict[str, Any]
    ) -> Generator[ToolInvokeMessage, None, None]:
        text = tool_parameters.get("text", "")
        result = text.upper()
        yield self.create_text_message(result)
```

## 参数类型

### 基本类型

- `string`: 字符串
- `number`: 数字
- `boolean`: 布尔值
- `select`: 下拉选择
- `file`: 文件上传

### 参数配置

```yaml
parameters:
  - name: text
    type: string
    required: true
    default: "default value"
    options:  # 仅用于 select 类型
      - value: option1
        label:
          en_US: Option 1
    form: llm  # llm 或 form
```

## 返回消息类型

### 文本消息

```python
yield self.create_text_message("结果文本")
```

### 链接消息

```python
yield self.create_link_message("https://example.com")
```

### 图片消息

```python
yield self.create_image_message("https://example.com/image.png")
```

### JSON 消息

```python
yield self.create_json_message({"key": "value"})
```

## 完整示例

### Google Translate 插件

```python
from typing import Any, Generator
import requests
from dify_plugin.entities.tool import ToolInvokeMessage
from dify_plugin import Tool


class GoogleTranslate(Tool):
    def _invoke(
        self, tool_parameters: dict[str, Any]
    ) -> Generator[ToolInvokeMessage, None, None]:
        content = tool_parameters.get("content", "")
        dest = tool_parameters.get("dest", "en")

        if not content:
            yield self.create_text_message("Error: content required")
            return

        try:
            result = self._translate(content, dest)
            yield self.create_text_message(result)
        except Exception as e:
            yield self.create_text_message(f"Error: {str(e)}")

    def _translate(self, content: str, dest: str) -> str:
        url = "https://translate.googleapis.com/translate_a/single"
        params = {
            "client": "gtx",
            "sl": "auto",
            "tl": dest,
            "dt": "t",
            "q": content
        }
        response = requests.get(url, params=params)
        response.raise_for_status()
        result = response.json()[0]
        return "".join([item[0] for item in result if item[0]])
```

## 最佳实践

1. **参数验证**：在开始处理前验证所有参数
2. **错误处理**：捕获所有异常并返回友好错误
3. **流式响应**：对大数据使用 Generator
4. **日志记录**：记录关键操作
