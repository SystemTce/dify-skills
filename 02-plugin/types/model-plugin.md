# Model 插件开发

Model 插件用于接入 LLM 提供商，如 Anthropic、OpenAI、Gemini 等。

## 快速开始

```bash
dify plugin init
? Select plugin type: Model
? Plugin name: my_model
```

## Provider 配置

```yaml
identity:
  name: my_model
  label:
    en_US: My Model Provider

credentials_schema:
  - name: api_key
    type: secret-input
    required: true
    label:
      en_US: API Key

models:
  - models/my_model.yaml
```

## Model 实现

```python
from dify_plugin import ModelProvider
from typing import Any, Generator


class MyModelProvider(ModelProvider):
    def _invoke(
        self, model: str, parameters: dict[str, Any]
    ) -> Generator[str, None, None]:
        # 调用 LLM API
        for chunk in self._call_api(model, parameters):
            yield chunk
```

## 多模态支持

- 图像输入
- 音频输入
- 视频输入
- 结构化输出

详细文档请参考官方 SDK。
