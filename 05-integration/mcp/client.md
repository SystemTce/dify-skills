# MCP Client

> Dify 调用外部 MCP Server 指南

## 概述

Dify 可以作为 MCP Client，调用外部 MCP Server 提供的工具，实现与外部系统的集成。

## MCP 客户端架构

```
Dify Workflow/Plugin
       |
       v
  MCP Client
       |
       v
  MCP Server (外部服务)
       |
       v
  外部系统 (数据库、API、文件系统等)
```

## 快速开始

### 安装依赖

```bash
pip install mcp
```

### 基本客户端实现

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from mcp.types import TextContent
import asyncio
import json

class MCPClient:
    def __init__(self, command: str, args: list[str]):
        self.server_params = StdioServerParameters(
            command=command,
            args=args
        )

    async def call_tool(self, tool_name: str, arguments: dict) -> list[TextContent]:
        """调用 MCP 工具"""
        async with stdio_client(self.server_params) as (read, write):
            async with ClientSession(read, write) as session:
                await session.initialize()
                result = await session.call_tool(tool_name, arguments)
                return result

    async def list_tools(self) -> list:
        """列出可用工具"""
        async with stdio_client(self.server_params) as (read, write):
            async with ClientSession(read, write) as session:
                await session.initialize()
                tools = await session.list_tools()
                return tools.tools

    async def read_resource(self, uri: str) -> str:
        """读取资源"""
        async with stdio_client(self.server_params) as (read, write):
            async with ClientSession(read, write) as session:
                await session.initialize()
                result = await session.read_resource(uri)
                return result.contents[0].text

# 使用示例
async def main():
    client = MCPClient("node", ["mcp-server.js"])

    # 列出工具
    tools = await client.list_tools()
    print("Available tools:", [t.name for t in tools])

    # 调用工具
    result = await client.call_tool("get_weather", {"city": "北京"})
    print(result)

asyncio.run(main())
```

## Dify 插件集成

### 创建 MCP 工具插件

```python
from dify_plugin import Tool
from dify_plugin.entities.tool import ToolInvokeMessage
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from typing import Generator
import asyncio

class MCPTool(Tool):
    """MCP 工具插件"""

    def __init__(self):
        super().__init__()
        self.server_command = "python"
        self.server_args = ["mcp_external_tool.py"]

    async def _call_mcp_tool(self, tool_name: str, arguments: dict):
        """调用外部 MCP 工具"""
        server_params = StdioServerParameters(
            command=self.server_command,
            args=self.server_args
        )

        async with stdio_client(server_params) as (read, write):
            async with ClientSession(read, write) as session:
                await session.initialize()
                result = await session.call_tool(tool_name, arguments)
                return result

    def _invoke(
        self,
        tool_parameters: dict
    ) -> Generator[ToolInvokeMessage, None, None]:
        tool_name = tool_parameters.get("tool_name")
        arguments = tool_parameters.get("arguments", {})

        # 同步调用异步 MCP
        result = asyncio.run(self._call_mcp_tool(tool_name, arguments))

        # 转换结果
        for item in result:
            if item.type == "text":
                yield self.create_text_message(item.text)
            elif item.type == "image":
                yield self.create_image_message(item.data, item.mime_type)
            elif item.type == "resource":
                yield self.create_json_message(json.loads(item.resource.text))
```

### 配置 MCP 节点

在 Dify 工作流中配置 MCP 节点：

```yaml
nodes:
  - id: mcp-tool
    data:
      type: tool
      provider_type: mcp
      server_config:
        type: stdio
        command: node
        args:
          - mcp-server.js
        env:
          API_KEY: "xxx"
      tool_mapping:
        - name: search
          description: 搜索工具
        - name: database_query
          description: 数据库查询
```

## 工具类型映射

### Dify 工具类型与 MCP 转换

| MCP 返回类型 | Dify 消息类型 |
|-------------|---------------|
| text | TextContent → create_text_message |
| image | ImageContent → create_image_message |
| resource | Resource → create_json_message |

### 完整转换示例

```python
from mcp.types import TextContent, ImageContent, ResourceContent

def convert_mcp_result(mcp_result: list) -> list[ToolInvokeMessage]:
    """转换 MCP 结果为 Dify 消息"""
    messages = []

    for item in mcp_result:
        if isinstance(item, TextContent):
            messages.append(
                self.create_text_message(item.text)
            )
        elif isinstance(item, ImageContent):
            messages.append(
                self.create_image_message(
                    data=item.data,
                    mime_type=item.mime_type
                )
            )
        elif isinstance(item, ResourceContent):
            # 解析资源内容
            text = item.resource.text
            messages.append(
                self.create_json_message(json.loads(text))
            )

    return messages
```

## 高级用法

### 连接池

```python
from mcp import ClientSession
from mcp.client.stdio import stdio_client
import asyncio

class MCPConnectionPool:
    """MCP 连接池"""

    def __init__(self, server_params: StdioServerParameters, pool_size: int = 5):
        self.server_params = server_params
        self.pool_size = pool_size
        self.semaphore = asyncio.Semaphore(pool_size)
        self._sessions = []

    async def get_session(self) -> ClientSession:
        """获取会话"""
        await self.semaphore.acquire()

        read, write = await stdio_client(self.server_params)
        session = ClientSession(read, write)
        await session.initialize()

        return session

    async def release_session(self, session: ClientSession):
        """释放会话"""
        session.close()
        self.semaphore.release()

    async def call_tool(self, tool_name: str, arguments: dict):
        """使用连接池调用工具"""
        session = await self.get_session()
        try:
            result = await session.call_tool(tool_name, arguments)
            return result
        finally:
            await self.release_session(session)
```

### 错误处理和重试

```python
import asyncio
from functools import wraps

def retry(max_attempts: int = 3, delay: float = 1.0):
    """重试装饰器"""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return await func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise
                    await asyncio.sleep(delay * (attempt + 1))
        return wrapper
    return decorator

class ReliableMCPClient:
    def __init__(self, server_params: StdioServerParameters):
        self.server_params = server_params

    @retry(max_attempts=3, delay=1.0)
    async def call_tool(self, tool_name: str, arguments: dict):
        """带重试的工具调用"""
        async with stdio_client(self.server_params) as (read, write):
            async with ClientSession(read, write) as session:
                await session.initialize()
                return await session.call_tool(tool_name, arguments)
```

## 配置示例

### 外部 MCP Server (Node.js)

```javascript
// mcp-server.js
const { Server } = require('@modelcontextprotocol/server');
const { StdioServerTransport } = require('@modelcontextprotocol/server');

const server = new Server({
  name: 'example-server',
  version: '1.0.0'
}, {
  capabilities: {
    tools: {}
  }
});

server.setRequestHandler('tools/list', async () => {
  return {
    tools: [{
      name: 'get_weather',
      description: '获取天气信息',
      inputSchema: {
        type: 'object',
        properties: {
          city: { type: 'string', description: '城市名称' }
        },
        required: ['city']
      }
    }]
  };
});

server.setRequestHandler('tools/call', async (request) => {
  const { name, arguments: args } = request.params;

  if (name === 'get_weather') {
    const weather = await getWeather(args.city);
    return {
      content: [{
        type: 'text',
        text: JSON.stringify(weather)
      }]
    };
  }
});

const transport = new StdioServerTransport();
server.run(transport);
```

### Dify 插件配置

```yaml
# plugin.yaml
version: 0.0.1
type: plugin
name: mcp-integration

tools:
  - provider/mcp_provider.yaml
```

```yaml
# provider/mcp_provider.yaml
name: mcp_external
tools:
  - name: get_weather
    label: 获取天气
    description: 通过 MCP 获取天气信息
    parameters:
      - name: city
        type: string
        required: true
```
