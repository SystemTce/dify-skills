# manifest.yaml 配置详解

`manifest.yaml` 是插件的清单文件，定义插件的基本信息、运行时配置和权限。

## 基本结构

```yaml
author: your_name
name: my_plugin
version: 0.0.1
type: plugin

description:
  en_US: Plugin description
  zh_Hans: 插件描述

label:
  en_US: My Plugin
  zh_Hans: 我的插件

icon: icon.svg

meta:
  version: 0.0.1
  arch:
    - amd64
    - arm64
  runner:
    language: python
    version: '3.12'
    entrypoint: main

plugins:
  tools:
    - provider/my_provider.yaml

resource:
  memory: 1048576
  permission:
    tool:
      enabled: true
    model:
      enabled: true
      llm: true

tags:
  - utilities
```

## 字段说明

### 基本信息

- **author**: 插件作者
- **name**: 插件名称（唯一标识符）
- **version**: 插件版本（语义化版本）
- **type**: 固定为 `plugin`

### 描述和标签

- **description**: 多语言描述
- **label**: 多语言显示名称
- **icon**: 图标文件路径
- **tags**: 分类标签

### 运行时配置

```yaml
meta:
  version: 0.0.1  # 元数据版本
  arch:           # 支持的架构
    - amd64
    - arm64
  runner:
    language: python  # 运行语言
    version: '3.12'   # Python 版本
    entrypoint: main  # 入口文件（不含 .py）
```

### 插件类型

```yaml
plugins:
  tools:              # 工具插件
    - provider/tool_provider.yaml
  models:             # 模型插件
    - provider/model_provider.yaml
  datasources:        # 数据源插件
    - provider/datasource_provider.yaml
  extensions:         # 扩展插件
    - provider/extension_provider.yaml
```

### 资源配置

```yaml
resource:
  memory: 1048576  # 内存限制（字节）
                   # 256MB = 268435456
                   # 512MB = 536870912
                   # 1GB = 1073741824

  permission:
    tool:
      enabled: true  # 允许调用其他工具
    model:
      enabled: true  # 允许调用模型
      llm: true      # 允许调用 LLM
    datasource:
      enabled: true  # 允许访问数据源
```

## 版本管理

遵循语义化版本控制（SemVer）：

- **MAJOR (x.0.0)**: 不兼容的 API 修改
- **MINOR (0.x.0)**: 向后兼容的功能新增
- **PATCH (0.0.x)**: 向后兼容的问题修正

每次更新必须增加版本号。

## 多语言支持

支持的语言代码：

- `en_US`: 英语（美国）
- `zh_Hans`: 简体中文
- `zh_Hant`: 繁体中文
- `ja_JP`: 日语
- `ko_KR`: 韩语
- `fr_FR`: 法语
- `de_DE`: 德语
- `es_ES`: 西班牙语

## 标签分类

常用标签：

- `utilities`: 实用工具
- `productivity`: 生产力
- `communication`: 通信
- `data`: 数据处理
- `ai`: AI 相关
- `search`: 搜索
- `translation`: 翻译

## 完整示例

```yaml
author: langgenius
name: google_translate
version: 0.0.3
type: plugin
created_at: '2024-09-20T08:03:44.658609186Z'

description:
  en_US: Translate text using Google
  zh_Hans: 使用 Google 进行翻译
  ja_JP: Google を使用してテキストを翻訳

label:
  en_US: Google Translate
  zh_Hans: 谷歌翻译
  ja_JP: Google 翻訳

icon: icon.svg

meta:
  version: 0.0.1
  arch:
    - amd64
    - arm64
  runner:
    language: python
    version: '3.12'
    entrypoint: main

plugins:
  tools:
    - provider/google_translate.yaml

resource:
  memory: 1048576
  permission:
    model:
      enabled: true
      llm: true
    tool:
      enabled: true

tags:
  - utilities
  - translation
```

## 验证配置

使用 dify CLI 验证配置：

```bash
dify plugin validate
```

## 常见错误

1. **版本号格式错误**：必须是 `x.y.z` 格式
2. **入口文件不存在**：检查 `entrypoint` 是否正确
3. **内存配置过小**：至少 256MB（268435456 字节）
4. **缺少必需字段**：`author`、`name`、`version` 必须提供
