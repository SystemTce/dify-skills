---
skill_name: dify-plugin
version: 1.0.0
parent_skill: dify-master
description: Dify 插件开发完整指南，涵盖工具、模型、数据源和扩展四种插件类型
category: plugin-development
tags:
  - dify
  - plugin
  - tool
  - model
  - datasource
  - extension
  - python
  - oauth
dependencies: []
last_updated: 2026-03-04
---

# Dify 插件开发 SKILL

## 概述

Dify 插件系统采用创新的 Beehive（蜂巢）架构，支持三种运行时模式，提供四种插件类型，为开发者提供强大的扩展能力。本 SKILL 提供从入门到精通的完整插件开发指南。

### 适用场景

当你需要以下场景时使用本 SKILL：

- **开发 Dify 插件**：工具、模型、数据源、扩展四种类型
- **集成第三方服务**：API 包装、OAuth 授权
- **实现自定义工具**：扩展 AI 能力
- **扩展 Dify 平台功能**：触发器、Agent 策略
- **将现有 API 迁移为插件**：API 到插件的转换
- **调试和优化插件性能**：性能调优和问题排查

### 核心价值

- **Beehive 架构**：去中心化调度、主从选举、故障自愈
- **多运行时支持**：Local、Debug、Serverless 三种模式
- **完整工具链**：dify CLI、Python SDK、调试运行时
- **丰富生态**：70+ 模型插件、20+ 工具插件、18+ 数据源插件
- **生产级质量**：OAuth 2.0、权限控制、监控追踪

---

## 快速开始

### 1. 环境准备

```bash
# 安装 Python 3.12+
python --version  # 确保 >= 3.12

# 安装 uv 包管理器（推荐）
curl -LsSf https://astral.sh/uv/install.sh | sh

# 安装 dify CLI
pip install dify-cli

# 验证安装
dify --version
```

### 2. 创建第一个插件

```bash
# 初始化插件项目
dify plugin init

# 选择插件类型
? Select plugin type: Tool
? Plugin name: my_first_tool
? Author: your_name
? Description: My first Dify tool plugin

# 进入项目目录
cd my_first_tool

# 安装依赖
uv pip install -r requirements.txt
```

### 3. 实现插件逻辑

编辑 `tools/my_tool.py`：

```python
from typing import Any, Generator
from dify_plugin.entities.tool import ToolInvokeMessage
from dify_plugin import Tool


class MyTool(Tool):
    def _invoke(
        self, tool_parameters: dict[str, Any]
    ) -> Generator[ToolInvokeMessage, None, None]:
        """
        工具调用入口
        """
        # 获取参数
        text = tool_parameters.get("text", "")

        # 处理逻辑
        result = f"处理结果: {text.upper()}"

        # 返回结果
        yield self.create_text_message(result)
```

### 4. 本地调试

```bash
# 启动 Debug Runtime
dify plugin run

# 在 Dify 界面中测试插件
# 访问 http://localhost:5001
```

### 5. 打包部署

```bash
# 打包插件
dify plugin package

# 生成 my_first_tool.difypkg 文件
# 上传到 Dify Marketplace 或私有部署
```

---

## 核心概念

### Beehive 架构

Dify 插件系统采用 Beehive（蜂巢）架构，核心特点：

1. **去中心化调度**：每个守护进程节点独立管理本地插件
2. **主从选举**：通过 Redis 分布式锁实现自动主节点选举
3. **状态同步**：所有节点状态存储在 Redis，支持跨节点访问
4. **故障自愈**：节点宕机时自动重新分配插件

详细架构说明请参考：[types/architecture.md](types/architecture.md)

### 三种运行时模式

| 运行时模式 | 适用场景 | 通信方式 | 优势 | 劣势 |
|----------|---------|---------|------|------|
| **Local Runtime** | 本地开发、小规模部署 | STDIN/STDOUT | 低延迟、资源隔离 | 内存占用高、扩展性有限 |
| **Debug Runtime** | 远程调试、开发环境 | TCP 全双工 | 开发友好、灵活部署 | 网络依赖、状态管理复杂 |
| **Serverless Runtime** | 大规模生产环境 | HTTP/SSE | 自动扩展、成本优化 | 冷启动延迟、调试复杂 |

详细对比请参考：[development/runtime-modes.md](development/runtime-modes.md)

### 四种插件类型

#### 1. Tool（工具插件）

扩展 AI 能力的工具，如翻译、搜索、数据处理等。

```yaml
# 示例：Google Translate
type: plugin
plugins:
  tools:
    - provider/google_translate.yaml
```

**使用场景**：
- API 包装（REST API → Tool）
- 数据处理（格式转换、计算）
- 外部服务集成（搜索、翻译）

#### 2. Model（模型插件）

接入 LLM 提供商，如 Anthropic、OpenAI、Gemini 等。

```yaml
# 示例：Anthropic Claude
type: plugin
plugins:
  models:
    - provider/anthropic.yaml
```

**使用场景**：
- 接入新的 LLM 提供商
- 自定义模型配置
- 多模态模型支持

#### 3. Datasource（数据源插件）

连接存储系统，如 S3、Google Drive、Notion 等。

```yaml
# 示例：AWS S3
type: plugin
plugins:
  datasources:
    - provider/aws_s3.yaml
```

**使用场景**：
- 云存储集成
- 文档平台连接
- 数据库访问

#### 4. Extension（扩展插件）

平台功能扩展，如 Slack、企业微信等第三方平台集成。

```yaml
# 示例：Slack
type: plugin
plugins:
  extensions:
    - provider/slack.yaml
```

**使用场景**：
- 第三方平台集成
- 触发器实现
- 自定义 Agent 策略

详细类型说明请参考：[types/](types/)

---

## 项目结构

标准的 Dify 插件项目结构：

```
my_plugin/
├── manifest.yaml           # 插件清单（必需）
├── main.py                 # 入口文件（必需）
├── requirements.txt        # Python 依赖
├── pyproject.toml         # 项目配置
├── README.md              # 插件说明
├── PRIVACY.md             # 隐私政策（如需 OAuth）
├── .env.example           # 环境变量示例
│
├── _assets/               # 资源文件
│   └── icon.svg          # 插件图标
│
├── provider/              # 提供者配置
│   ├── provider_name.yaml # 提供者配置文件
│   └── provider_name.py   # 提供者实现
│
└── tools/                 # 工具实现（Tool 插件）
    ├── tool_name.yaml    # 工具配置文件
    └── tool_name.py      # 工具实现
```

### 核心文件说明

#### manifest.yaml

插件清单文件，定义插件的基本信息和配置：

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
  memory: 1048576  # 1MB = 1048576 bytes
  permission:
    tool:
      enabled: true
    model:
      enabled: true
      llm: true

tags:
  - utilities
```

详细配置说明请参考：[development/manifest.md](development/manifest.md)

#### main.py

插件入口文件：

```python
from dify_plugin import Plugin, DifyPluginEnv

# 创建插件实例
plugin = Plugin(DifyPluginEnv(MAX_REQUEST_TIMEOUT=120))

if __name__ == '__main__':
    # 启动插件
    plugin.run()
```

---

## 开发指南

### Tool 插件开发

#### 1. 创建 Provider

编辑 `provider/my_provider.yaml`：

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
  tags:
    - utilities

extra:
  python:
    source: provider/my_provider.py

tools:
  - tools/my_tool.yaml
```

编辑 `provider/my_provider.py`：

```python
from typing import Any
from dify_plugin.errors.tool import ToolProviderCredentialValidationError
from dify_plugin import ToolProvider
from tools.my_tool import MyTool


class MyProvider(ToolProvider):
    def _validate_credentials(self, credentials: dict[str, Any]) -> None:
        """
        验证凭证（如果需要）
        """
        try:
            # 验证 API Key 或其他凭证
            api_key = credentials.get("api_key")
            if not api_key:
                raise ToolProviderCredentialValidationError("API Key is required")

            # 测试凭证有效性
            MyTool().invoke(
                tool_parameters={"text": "test"}
            )
        except Exception as e:
            raise ToolProviderCredentialValidationError(str(e))
```

#### 2. 创建 Tool

编辑 `tools/my_tool.yaml`：

```yaml
identity:
  author: your_name
  name: my_tool
  label:
    en_US: My Tool
    zh_Hans: 我的工具

description:
  human:
    en_US: Tool description for humans
    zh_Hans: 给人类看的工具描述
  llm: Tool description for LLM

extra:
  python:
    source: tools/my_tool.py

parameters:
  - name: text
    type: string
    required: true
    label:
      en_US: Input Text
      zh_Hans: 输入文本
    human_description:
      en_US: The text to process
      zh_Hans: 要处理的文本
    llm_description: The input text to be processed
    form: llm
```

编辑 `tools/my_tool.py`：

```python
from typing import Any, Generator
from dify_plugin.entities.tool import ToolInvokeMessage
from dify_plugin import Tool


class MyTool(Tool):
    def _invoke(
        self, tool_parameters: dict[str, Any]
    ) -> Generator[ToolInvokeMessage, None, None]:
        """
        工具调用入口

        Args:
            tool_parameters: 工具参数字典

        Yields:
            ToolInvokeMessage: 工具调用消息
        """
        # 1. 获取参数
        text = tool_parameters.get("text", "")

        if not text:
            yield self.create_text_message("Error: text parameter is required")
            return

        try:
            # 2. 处理逻辑
            result = self._process_text(text)

            # 3. 返回结果
            yield self.create_text_message(result)

        except Exception as e:
            yield self.create_text_message(f"Error: {str(e)}")

    def _process_text(self, text: str) -> str:
        """
        处理文本的具体逻辑
        """
        return text.upper()
```

详细开发指南请参考：[development/tool.md](development/tool.md)

---

## OAuth 2.0 集成

### OAuth 流程概述

Dify 支持完整的 OAuth 2.0 授权流程：

1. **获取授权 URL**：用户点击授权按钮
2. **用户授权**：跳转到第三方服务授权页面
3. **获取授权码**：用户授权后返回授权码
4. **交换访问令牌**：使用授权码交换访问令牌
5. **刷新令牌**：访问令牌过期时使用刷新令牌获取新令牌

### 配置 OAuth Schema

在 `provider/my_provider.yaml` 中添加 OAuth 配置：

```yaml
oauth_schema:
  client_schema:
    - name: client_id
      type: secret-input
      required: true
      label:
        en_US: Client ID
        zh_Hans: 客户端 ID
      placeholder:
        en_US: Enter your OAuth Client ID
        zh_Hans: 输入您的 OAuth 客户端 ID
      help:
        en_US: Get your client ID from the service provider
        zh_Hans: 从服务提供商获取客户端 ID
      url: https://console.example.com/credentials

    - name: client_secret
      type: secret-input
      required: true
      label:
        en_US: Client Secret
        zh_Hans: 客户端密钥
      placeholder:
        en_US: Enter your OAuth Client Secret
        zh_Hans: 输入您的 OAuth 客户端密钥

  credentials_schema:
    - name: access_token
      type: secret-input
      label:
        en_US: Access Token
        zh_Hans: 访问令牌

    - name: refresh_token
      type: secret-input
      label:
        en_US: Refresh Token
        zh_Hans: 刷新令牌
```

### 实现 OAuth Provider

```python
from typing import Any
from dify_plugin import ToolProvider
from dify_plugin.errors.tool import ToolProviderCredentialValidationError
import requests


class MyOAuthProvider(ToolProvider):
    def _validate_credentials(self, credentials: dict[str, Any]) -> None:
        """
        验证 OAuth 凭证
        """
        access_token = credentials.get("access_token")
        if not access_token:
            raise ToolProviderCredentialValidationError(
                "Access token is required"
            )

        # 测试访问令牌有效性
        try:
            response = requests.get(
                "https://api.example.com/user",
                headers={"Authorization": f"Bearer {access_token}"}
            )
            response.raise_for_status()
        except Exception as e:
            raise ToolProviderCredentialValidationError(
                f"Invalid access token: {str(e)}"
            )
```

### 使用 OAuth 令牌

在 Tool 中使用 OAuth 令牌：

```python
class MyOAuthTool(Tool):
    def _invoke(
        self, tool_parameters: dict[str, Any]
    ) -> Generator[ToolInvokeMessage, None, None]:
        # 获取 OAuth 凭证
        credentials = self.runtime.credentials
        access_token = credentials.get("access_token")

        # 调用 API
        response = requests.get(
            "https://api.example.com/data",
            headers={"Authorization": f"Bearer {access_token}"}
        )

        yield self.create_text_message(response.text)
```

完整 OAuth 集成指南请参考：[development/oauth-integration.md](development/oauth-integration.md)

---

## 测试和调试

### 单元测试

使用 pytest 编写单元测试：

```python
# tests/test_my_tool.py
import pytest
from tools.my_tool import MyTool


def test_my_tool():
    tool = MyTool()
    result = list(tool.invoke(tool_parameters={"text": "hello"}))

    assert len(result) == 1
    assert "HELLO" in result[0].message
```

运行测试：

```bash
pytest tests/
```

### 集成测试

测试插件与 Dify 系统的集成：

```python
# tests/test_integration.py
def test_plugin_integration():
    # 测试插件安装
    # 测试工具调用
    # 测试 OAuth 流程
    pass
```

### Debug Runtime 调试

启动 Debug Runtime 进行远程调试：

```bash
# 启动 Debug Runtime
dify plugin run --debug

# 在 Dify 界面中测试
# 查看实时日志和错误信息
```

### 常见问题排查

#### 插件启动失败

- 检查 Python 版本（需要 3.12+）
- 检查依赖安装（`requirements.txt`）
- 检查 `manifest.yaml` 配置
- 检查入口文件路径（`main.py`）

#### 权限验证失败

- 检查 `credentials` 配置
- 检查 OAuth 令牌是否过期
- 检查 API Key 是否有效
- 检查权限声明（`resource.permission`）

#### 运行时错误

- 调整超时时间（`MAX_REQUEST_TIMEOUT`）
- 优化内存使用（`resource.memory`）
- 检查网络连接
- 检查反向调用权限

详细排查指南请参考：[testing/troubleshooting.md](testing/troubleshooting.md)

---

## 从 API 到插件迁移

### 迁移策略

1. **API 包装模式**：将 REST API 包装为 Tool 插件
2. **SDK 集成模式**：使用第三方 Python SDK
3. **OAuth 迁移**：实现 OAuth 2.0 授权流程
4. **批量操作优化**：使用流式响应处理大数据

### 迁移步骤

1. **分析现有 API**
   - API 端点和参数
   - 认证方式（API Key / OAuth）
   - 响应格式和错误处理

2. **选择插件类型**
   - REST API → Tool 插件
   - LLM API → Model 插件
   - 存储 API → Datasource 插件

3. **创建插件项目**
   ```bash
   dify plugin init
   ```

4. **实现 API 调用**
   ```python
   import requests

   class APIWrapperTool(Tool):
       def _invoke(self, tool_parameters: dict[str, Any]):
           response = requests.post(
               "https://api.example.com/endpoint",
               json=tool_parameters,
               headers={"Authorization": f"Bearer {api_key}"}
           )
           yield self.create_text_message(response.text)
   ```

5. **配置认证**
   - API Key：在 `provider.yaml` 中配置 `credentials`
   - OAuth：配置 `oauth_schema`

6. **添加错误处理**
   ```python
   try:
       response = requests.post(...)
       response.raise_for_status()
   except requests.exceptions.RequestException as e:
       yield self.create_text_message(f"API Error: {str(e)}")
   ```

7. **编写测试**
   ```python
   def test_api_wrapper():
       tool = APIWrapperTool()
       result = list(tool.invoke(tool_parameters={...}))
       assert result[0].message
   ```

8. **打包部署**
   ```bash
   dify plugin package
   ```

### 实际案例

- **REST API → Tool**：Google Translate（无认证）
- **OAuth API → Tool**：Google Contacts（OAuth 2.0）
- **LLM API → Model**：Anthropic Claude（API Key）
- **存储 API → Datasource**：AWS S3（IAM 认证）

详细迁移指南请参考：[development/migration-guide.md](development/migration-guide.md)

---

## 最佳实践

### 代码组织

1. **单一职责**：每个 Tool 只做一件事
2. **错误处理**：捕获所有异常并返回友好错误信息
3. **参数验证**：在 `_invoke` 开始时验证所有参数
4. **日志记录**：使用结构化日志记录关键操作

### 性能优化

1. **流式响应**：对于大数据使用 Generator 流式返回
2. **缓存策略**：缓存频繁访问的数据
3. **异步处理**：使用 async/await 处理 I/O 密集操作
4. **资源限制**：合理配置 `resource.memory`

### 安全考虑

1. **输入验证**：验证所有用户输入
2. **敏感数据**：使用 `secret-input` 类型存储敏感信息
3. **权限控制**：最小权限原则，只声明必需的权限
4. **错误信息**：不要在错误信息中泄露敏感数据

### 用户体验

1. **清晰的描述**：提供详细的 `description` 和 `help`
2. **合理的默认值**：为参数提供合理的默认值
3. **多语言支持**：至少支持 `en_US` 和 `zh_Hans`
4. **友好的错误**：返回用户可理解的错误信息

---

## 部署和运维

### 运行时模式选择

| 场景 | 推荐模式 | 理由 |
|-----|---------|------|
| 本地开发 | Local Runtime | 低延迟、易调试 |
| 远程调试 | Debug Runtime | 灵活、支持远程 |
| 小规模生产 | Local Runtime | 简单、成本低 |
| 大规模生产 | Serverless Runtime | 自动扩展、高可用 |

### Serverless 部署

部署到 AWS Lambda：

```bash
# 1. 打包插件
dify plugin package

# 2. 创建 Lambda 函数
aws lambda create-function \
  --function-name my-plugin \
  --runtime python3.12 \
  --handler main.handler \
  --zip-file fileb://my_plugin.difypkg

# 3. 配置环境变量
aws lambda update-function-configuration \
  --function-name my-plugin \
  --environment Variables={DIFY_PLUGIN_DAEMON_URL=https://...}

# 4. 在 Dify 中注册端点
# 访问 Dify 管理界面，添加 Serverless 端点
```

### 监控和日志

1. **OpenTelemetry 追踪**：自动集成分布式追踪
2. **Sentry 错误追踪**：捕获和报告错误
3. **结构化日志**：使用 Python logging 模块
4. **性能指标**：监控响应时间、错误率、资源使用

### 故障恢复

1. **自动重启**：WatchDog 监控插件健康状态
2. **重试机制**：Serverless 模式支持指数退避重试
3. **降级处理**：故障时返回友好错误信息
4. **备份恢复**：定期备份插件配置和状态

---

## 参考资源

### 官方文档

- [Dify Plugin Development Cheatsheet](https://docs.dify.ai/en/develop-plugin/dev-guides-and-walkthroughs/cheatsheet)
- [Dify Plugin System Design](https://dify.ai/blog/dify-plugin-system-design-and-implementation)
- [Initialize Development Tools](https://legacy-docs.dify.ai/plugins/quick-start/develop-plugins/initialize-development-tools)

### 开源项目

- [dify-plugin-daemon](https://github.com/langgenius/dify-plugin-daemon) - 插件守护进程
- [dify-official-plugins](https://github.com/langgenius/dify-official-plugins) - 官方插件集合
- [dify-plugin-sdks](https://github.com/langgenius/dify-plugin-sdks) - Python SDK

### 社区资源

- [Dify Discord](https://discord.gg/FngNHpbcY7) - 官方 Discord 社区
- [Dify Reddit](https://reddit.com/r/difyai) - Reddit 讨论区
- [Dify Marketplace](https://marketplace.dify.ai/) - 插件市场

### 详细文档

- [types/](types/) - 四种插件类型详解
- [development/](development/) - 开发指南
- [templates/](templates/) - 插件模板
- [testing/](testing/) - 测试指南

---

**版本**: v1.0.0
**最后更新**: 2026-03-04
**维护者**: Dify SKILL Team
