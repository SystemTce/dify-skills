# Extension 插件开发

Extension 插件用于平台功能扩展，如 Slack、企业微信等第三方平台集成。

## 快速开始

```bash
dify plugin init
? Select plugin type: Extension
? Plugin name: my_extension
```

## Provider 配置

```yaml
identity:
  name: my_extension
  label:
    en_US: My Extension

extensions:
  - extensions/my_extension.yaml
```

## Extension 实现

```python
from dify_plugin import ExtensionProvider


class MyExtension(ExtensionProvider):
    def handle_event(self, event: dict):
        # 处理事件
        pass
```

## 常见扩展

- Slack 集成
- 企业微信集成
- 触发器
- Agent 策略
