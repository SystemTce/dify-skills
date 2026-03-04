# Datasource 插件开发

Datasource 插件用于连接存储系统，如 S3、Google Drive、Notion 等。

## 快速开始

```bash
dify plugin init
? Select plugin type: Datasource
? Plugin name: my_datasource
```

## Provider 配置

```yaml
identity:
  name: my_datasource
  label:
    en_US: My Datasource

credentials_schema:
  - name: access_key
    type: secret-input
    required: true

datasources:
  - datasources/my_datasource.yaml
```

## Datasource 实现

```python
from dify_plugin import DatasourceProvider
from typing import Any, List


class MyDatasource(DatasourceProvider):
    def list_files(self, path: str) -> List[dict]:
        # 列出文件
        pass

    def read_file(self, path: str) -> bytes:
        # 读取文件
        pass
```

## 常见数据源

- AWS S3
- Google Cloud Storage
- Azure Blob Storage
- Notion
- Google Drive
