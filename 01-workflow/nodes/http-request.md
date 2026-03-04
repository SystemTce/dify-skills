# HTTP 请求节点详解

## 概述

HTTP 请求节点将工作流连接到外部 API 和 Web 服务，用于数据检索、Webhook 传递、文件操作和服务集成。

## 核心功能

- 支持所有 HTTP 方法（GET, POST, PUT, PATCH, DELETE, HEAD）
- 动态变量插入
- 多种认证方式
- 文件上传和下载
- 错误处理和重试
- 超时控制

## 支持的 HTTP 方法

### 数据检索
- **GET**: 获取资源
- **HEAD**: 获取响应头

### 数据提交
- **POST**: 创建资源
- **PUT**: 更新资源（完整）
- **PATCH**: 更新资源（部分）

### 资源管理
- **DELETE**: 删除资源

## 配置结构

### 基础配置

```yaml
- data:
    type: http-request
    method: GET
    url: "https://api.example.com/data"
    headers:
      Content-Type: application/json
    timeout:
      connect: 10
      read: 30
      write: 30
  id: http_request
```

## 动态变量插入

使用 `{{variable_name}}` 语法插入变量：

### URL 中的变量

```yaml
url: "https://api.example.com/users/{{#start.user_id#}}/posts"
```

### 请求头中的变量

```yaml
headers:
  Authorization: "Bearer {{#start.api_token#}}"
  X-User-ID: "{{#start.user_id#}}"
```

### 请求体中的变量

```yaml
body:
  type: json
  content: |
    {
      "name": "{{#start.name#}}",
      "email": "{{#start.email#}}",
      "data": {{#code.processed_data#}}
    }
```

### 嵌套对象访问

```yaml
url: "https://api.example.com/items/{{#api_response.data.items[0].id#}}"
```

## 认证选项

### 无认证

```yaml
auth:
  type: none
```

### API Key - Basic

Base64 编码的用户名:密码：

```yaml
auth:
  type: api_key
  api_key:
    type: basic
    username: "{{#start.username#}}"
    password: "{{#start.password#}}"
```

### API Key - Bearer Token

```yaml
auth:
  type: api_key
  api_key:
    type: bearer
    token: "{{#start.api_token#}}"
```

### API Key - Custom Header

```yaml
auth:
  type: api_key
  api_key:
    type: custom
    header_name: "X-API-Key"
    value: "{{#start.api_key#}}"
```

## 请求体类型

### JSON

结构化数据：

```yaml
body:
  type: json
  content: |
    {
      "query": "{{#start.query#}}",
      "limit": 10,
      "filters": {
        "category": "{{#start.category#}}"
      }
    }
```

### Form Data

Web 表单：

```yaml
body:
  type: form-data
  content:
    name: "{{#start.name#}}"
    email: "{{#start.email#}}"
    message: "{{#start.message#}}"
```

### Binary

文件上传：

```yaml
body:
  type: binary
  content: "{{#start.file#}}"
```

### Raw Text

自定义内容类型：

```yaml
body:
  type: raw
  content: "{{#start.text_data#}}"
  content_type: "text/plain"
```

## 超时设置

```yaml
timeout:
  connect: 10    # 连接超时（秒）
  read: 30       # 读取超时（秒）
  write: 30      # 写入超时（秒）
```

## 文件处理

### 文件上传

```yaml
- data:
    type: http-request
    method: POST
    url: "https://api.example.com/upload"
    body:
      type: binary
      content: "{{#start.file#}}"
    headers:
      Content-Type: "application/octet-stream"
  id: upload_file
```

### 文件下载

系统自动检测文件响应：

```yaml
- data:
    type: http-request
    method: GET
    url: "https://api.example.com/download/{{#start.file_id#}}"
  id: download_file

# 输出变量
{{#download_file.files#}}  # 文件内容
{{#download_file.size#}}   # 文件大小
```

## 错误管理

### 重试配置

```yaml
retry:
  enabled: true
  max_retries: 3
  retry_interval: 1000  # 毫秒，最大 5000ms
```

### 错误处理分支

```yaml
error_handling:
  enabled: true
  on_error:
    # 连接到错误处理节点
    target: error_handler
```

## 响应结构

### 响应变量

```yaml
{{#http_request.body#}}         # 响应体内容
{{#http_request.status_code#}}  # HTTP 状态码
{{#http_request.headers#}}      # 响应头（键值对）
{{#http_request.files#}}        # 文件内容（如果是文件）
{{#http_request.size#}}         # 文件大小（可读格式）
```

## 实际案例

### 案例1：GET 请求

```yaml
- data:
    type: http-request
    method: GET
    url: "https://api.github.com/users/{{#start.username#}}"
    headers:
      Accept: application/json
    timeout:
      connect: 10
      read: 30
  id: get_user
```

### 案例2：POST 请求（JSON）

```yaml
- data:
    type: http-request
    method: POST
    url: "https://api.example.com/orders"
    auth:
      type: api_key
      api_key:
        type: bearer
        token: "{{#start.api_token#}}"
    headers:
      Content-Type: application/json
    body:
      type: json
      content: |
        {
          "customer_id": "{{#start.customer_id#}}",
          "items": {{#code.items#}},
          "total": {{#code.total#}}
        }
    retry:
      enabled: true
      max_retries: 3
      retry_interval: 2000
  id: create_order
```

### 案例3：文件上传

```yaml
- data:
    type: http-request
    method: POST
    url: "https://api.example.com/upload"
    auth:
      type: api_key
      api_key:
        type: custom
        header_name: "X-API-Key"
        value: "{{#start.api_key#}}"
    body:
      type: binary
      content: "{{#start.document#}}"
    headers:
      Content-Type: "application/pdf"
    timeout:
      connect: 10
      read: 60
      write: 60
  id: upload_document
```

### 案例4：Webhook 通知

```yaml
- data:
    type: http-request
    method: POST
    url: "{{#start.webhook_url#}}"
    headers:
      Content-Type: application/json
    body:
      type: json
      content: |
        {
          "event": "workflow_completed",
          "data": {
            "workflow_id": "{{#sys.workflow_id#}}",
            "result": "{{#llm.text#}}",
            "timestamp": "{{#sys.timestamp#}}"
          }
        }
  id: webhook_notify
```

### 案例5：多步骤 API 调用

```yaml
# 步骤 1: 获取访问令牌
- data:
    type: http-request
    method: POST
    url: "https://api.example.com/auth/token"
    body:
      type: json
      content: |
        {
          "client_id": "{{#start.client_id#}}",
          "client_secret": "{{#start.client_secret#}}"
        }
  id: get_token

# 步骤 2: 使用令牌调用 API
- data:
    type: http-request
    method: GET
    url: "https://api.example.com/data"
    auth:
      type: api_key
      api_key:
        type: bearer
        token: "{{#get_token.body.access_token#}}"
  id: fetch_data

# 步骤 3: 更新数据
- data:
    type: http-request
    method: PUT
    url: "https://api.example.com/data/{{#fetch_data.body.id#}}"
    auth:
      type: api_key
      api_key:
        type: bearer
        token: "{{#get_token.body.access_token#}}"
    body:
      type: json
      content: |
        {
          "status": "processed",
          "result": "{{#llm.text#}}"
        }
  id: update_data
```

## 常见使用场景

### API 数据丰富

从外部 API 获取额外信息：

```yaml
- data:
    type: http-request
    method: GET
    url: "https://api.weather.com/v1/current?city={{#start.city#}}"
  id: get_weather
```

### Webhook 通知

向外部系统发送事件通知：

```yaml
- data:
    type: http-request
    method: POST
    url: "{{#start.webhook_url#}}"
    body:
      type: json
      content: |
        {
          "event": "task_completed",
          "data": {{#code.result#}}
        }
  id: send_webhook
```

### 文档处理

上传文档进行处理：

```yaml
- data:
    type: http-request
    method: POST
    url: "https://api.ocr.com/process"
    body:
      type: binary
      content: "{{#start.document#}}"
  id: ocr_process
```

### 多步骤 API 工作流

链接多个 API 请求：

```yaml
# 认证 → 查询 → 更新 → 通知
```

## 最佳实践

### 1. 错误处理
- 启用重试机制
- 设置合理的超时时间
- 实现错误处理分支
- 记录失败原因

### 2. 安全性
- 使用环境变量存储敏感信息
- 不在 URL 中暴露密钥
- 使用 HTTPS
- 验证响应数据

### 3. 性能优化
- 设置适当的超时
- 使用缓存减少重复请求
- 并行处理独立请求
- 压缩大文件

### 4. 变量使用
- 验证变量存在性
- 处理空值情况
- 正确转义特殊字符
- 使用嵌套访问获取深层数据

### 5. 调试技巧
- 记录请求和响应
- 使用测试端点
- 验证请求格式
- 检查响应状态码

## 常见问题

### Q: HTTP 请求节点和工具节点有什么区别？

A:
- HTTP 请求节点：灵活的 HTTP 接口，需要手动配置
- 工具节点：预配置的集成，提供结构化接口和错误处理

### Q: 如何处理 API 认证？

A: 支持多种认证方式：
- Bearer Token
- Basic Auth
- API Key（自定义头）
- 在请求体中传递凭证

### Q: 如何处理大文件上传？

A:
1. 增加超时时间
2. 使用 binary 请求体类型
3. 考虑分块上传（如果 API 支持）
4. 监控上传进度

### Q: 请求失败了怎么办？

A:
1. 启用重试机制
2. 检查网络连接
3. 验证 URL 和参数
4. 查看响应状态码和错误信息
5. 实现错误处理分支

## 相关节点

- [工具节点](./tools.md) - 预配置的 API 集成
- [Code 节点](./code.md) - 处理 API 响应
- [If-Else 节点](./if-else.md) - 根据响应分支

## 返回

[工作流 SKILL](../SKILL.md)
