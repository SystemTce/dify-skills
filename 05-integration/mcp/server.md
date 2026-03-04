# MCP Server

> Dify 作为 MCP Server 实现指南

## 概述

Dify 可以作为 MCP (Model Context Protocol) Server，向 AI 助手（如 Claude）暴露工具和资源。

## 快速开始

### 安装依赖

```bash
pip install mcp
```

### 基本实现

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent, Resource
import asyncio
import json

class DifyMCPServer:
    def __init__(self, api_key: str, base_url: str = "https://api.dify.ai/v1"):
        self.api_key = api_key
        self.base_url = base_url
        self.server = Server("dify-mcp")

    async def start(self):
        """启动 MCP Server"""
        self._register_handlers()
        await stdio_server(self.server.run)

    def _register_handlers(self):
        """注册处理器"""

        @self.server.list_tools()
        async def list_tools() -> list[Tool]:
            """列出可用工具"""
            return [
                Tool(
                    name="dify_chat",
                    description="与 Dify 应用进行对话",
                    inputSchema={
                        "type": "object",
                        "properties": {
                            "query": {
                                "type": "string",
                                "description": "用户查询内容"
                            },
                            "user": {
                                "type": "string",
                                "description": "用户标识"
                            }
                        },
                        "required": ["query", "user"]
                    }
                ),
                Tool(
                    name="dify_workflow",
                    description="执行 Dify 工作流",
                    inputSchema={
                        "type": "object",
                        "properties": {
                            "workflow_id": {
                                "type": "string",
                                "description": "工作流 ID"
                            },
                            "inputs": {
                                "type": "object",
                                "description": "工作流输入参数"
                            },
                            "user": {
                                "type": "string",
                                "description": "用户标识"
                            }
                        },
                        "required": ["workflow_id", "inputs", "user"]
                    }
                ),
                Tool(
                    name="dify_parameters",
                    description="获取应用参数配置",
                    inputSchema={
                        "type": "object",
                        "properties": {
                            "app_id": {
                                "type": "string",
                                "description": "应用 ID"
                            }
                        },
                        "required": ["app_id"]
                    }
                )
            ]

        @self.server.call_tool()
        async def call_tool(name: str, arguments: dict) -> list[TextContent]:
            """调用工具"""
            if name == "dify_chat":
                return await self._handle_chat(arguments)
            elif name == "dify_workflow":
                return await self._handle_workflow(arguments)
            elif name == "dify_parameters":
                return await self._handle_parameters(arguments)
            else:
                raise ValueError(f"Unknown tool: {name}")

        @self.server.list_resources()
        async def list_resources() -> list[Resource]:
            """列出可用资源"""
            return [
                Resource(
                    uri="dify://app/info",
                    name="app_info",
                    description="当前应用信息"
                )
            ]

        @self.server.read_resource()
        async def read_resource(uri: str) -> str:
            """读取资源"""
            if uri == "dify://app/info":
                return json.dumps({"version": "0.15.0", "type": "chat"})
            raise ValueError(f"Unknown resource: {uri}")

    async def _handle_chat(self, args: dict) -> list[TextContent]:
        """处理聊天请求"""
        import requests

        response = requests.post(
            f"{self.base_url}/chat-messages",
            headers={"Authorization": f"Bearer {self.api_key}"},
            json={
                "query": args["query"],
                "user": args["user"]
            }
        )
        result = response.json()
        return [TextContent(type="text", text=json.dumps(result))]

    async def _handle_workflow(self, args: dict) -> list[TextContent]:
        """处理工作流请求"""
        import requests

        response = requests.post(
            f"{self.base_url}/workflows/run",
            headers={"Authorization": f"Bearer {self.api_key}"},
            json={
                "workflow_id": args["workflow_id"],
                "inputs": args["inputs"],
                "user": args["user"]
            }
        )
        result = response.json()
        return [TextContent(type="text", text=json.dumps(result))]

    async def _handle_parameters(self, args: dict) -> list[TextContent]:
        """处理参数请求"""
        import requests

        response = requests.get(
            f"{self.base_url}/parameters",
            headers={"Authorization": f"Bearer {self.api_key}"}
        )
        result = response.json()
        return [TextContent(type="text", text=json.dumps(result))]

# 运行服务器
if __name__ == "__main__":
    server = DifyMCPServer(api_key="app-xxxxx")
    asyncio.run(server.start())
```

## 工具定义

### 工具结构

```python
Tool(
    name="tool_name",
    description="工具描述",
    inputSchema={
        "type": "object",
        "properties": {
            "param_name": {
                "type": "string",
                "description": "参数描述",
                "default": "默认值"
            }
        },
        "required": ["param_name"]
    }
)
```

### 完整示例

```python
# 复杂工具定义示例
Tool(
    name="knowledge_search",
    description="在知识库中搜索相关内容",
    inputSchema={
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "搜索关键词"
            },
            "dataset_id": {
                "type": "string",
                "description": "知识库 ID"
            },
            "top_k": {
                "type": "integer",
                "description": "返回结果数量",
                "default": 5
            },
            "score_threshold": {
                "type": "number",
                "description": "相似度阈值",
                "default": 0.7
            }
        },
        "required": ["query"]
    }
)
```

## 资源定义

### 资源类型

```python
from mcp.types import Resource, ResourceTemplate

# 静态资源
Resource(
    uri="dify://config/app",
    name="app_config",
    description="应用配置信息",
    mimeType="application/json"
)

# 动态资源模板
ResourceTemplate(
    uri="dify://workflow/{workflow_id}",
    name="workflow_detail",
    description="获取工作流详情",
    mimeType="application/json"
)
```

## 高级特性

### 认证处理

```python
async def authenticated_tool(name: str, arguments: dict) -> list[TextContent]:
    """带认证的工具调用"""
    # 从环境变量或配置获取认证信息
    api_key = os.environ.get("DIFY_API_KEY")
    if not api_key:
        raise ValueError("Missing API key")

    # 验证用户权限
    user_id = arguments.get("user")
    if not has_permission(user_id, name):
        raise ValueError(f"User {user_id} not authorized to call {name}")

    # 执行工具
    return await execute_tool(name, arguments)
```

### 错误处理

```python
@self.server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    try:
        return await execute_tool(name, arguments)
    except ValueError as e:
        return [TextContent(type="text", text=f"Error: {str(e)}")]
    except requests.exceptions.RequestException as e:
        return [TextContent(type="text", text=f"Network error: {str(e)}")]
    except Exception as e:
        return [TextContent(type="text", text=f"Unexpected error: {str(e)}")]
```

### 工具组合

```python
@self.server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    if name == "chat_with_knowledge":
        # 组合多个工具
        # 1. 搜索知识库
        search_result = await self._execute_tool("knowledge_search", {
            "query": arguments["query"]
        })

        # 2. 使用搜索结果进行对话
        chat_result = await self._execute_tool("dify_chat", {
            "query": f"基于以下上下文回答: {search_result}\n\n问题: {arguments['query']}",
            "user": arguments["user"]
        })

        return chat_result
```

## 运行方式

### Stdio 模式

```bash
# 直接运行
python dify_mcp_server.py

# 或使用 Claude Code 配置
```

### HTTP 模式

```python
from mcp.server.sse import SseServerTransport

async def start_http_server():
    transport = SseServerTransport("/messages")
    await transport.start_server()
```

## 配置示例

### Claude Desktop 配置

```json
{
  "mcpServers": {
    "dify": {
      "command": "python",
      "args": ["/path/to/dify_mcp_server.py"],
      "env": {
        "DIFY_API_KEY": "app-xxxxx"
      }
    }
  }
}
```

### 环境变量

```bash
export DIFY_API_KEY="app-xxxxx"
export DIFY_BASE_URL="https://api.dify.ai/v1"
```
