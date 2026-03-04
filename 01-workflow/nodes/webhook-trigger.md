# Webhook 触发器节点详解

## 概述

Webhook 触发器使工作流能够在外部系统发送 HTTP 请求到生成的端点时自动执行，实现事件驱动的自动化。

## 核心功能

- 监听外部 HTTP 请求
- 提取请求参数作为工作流变量
- 自定义响应内容
- 支持多种 HTTP 方法和内容类型

## 关键特性

### 自动生成端点
- 每个 webhook 触发器生成唯一的 URL
- 支持自定义 URL 前缀（自托管部署）
- 测试和生产环境使用不同的 URL

### 参数提取
从三个来源提取数据：
- Query 参数（URL 中 `?` 后的参数）
- 请求头（如认证令牌）
- 请求体（核心事件数据）

### 响应自定义
- 默认返回 `200 OK`
- 可自定义状态码（200-399）
- 支持 JSON 或纯文本响应

## 配置方法

### 基础配置

```yaml
- data:
    type: webhook-trigger
    webhook:
      method: POST
      content_type: application/json
  id: webhook_trigger
```

### HTTP 方法

支持的方法：

```yaml
# POST 请求
webhook:
  method: POST

# GET 请求
webhook:
  method: GET

# PUT 请求
webhook:
  method: PUT

# DELETE 请求
webhook:
  method: DELETE
```

### 内容类型

```yaml
# JSON 格式
webhook:
  content_type: application/json

# 表单数据
webhook:
  content_type: application/x-www-form-urlencoded

# 纯文本
webhook:
  content_type: text/plain
```

## 参数提取

### Query 参数

从 URL 查询字符串提取：

```yaml
- data:
    type: webhook-trigger
    webhook:
      method: GET
      parameters:
      - name: user_id
        type: string
        source: query
        required: true
      - name: page
        type: number
        source: query
        required: false
  id: webhook_trigger

# URL 示例: https://api.dify.ai/webhook/xxx?user_id=123&page=1
# 输出变量: {{#webhook_trigger.user_id#}}, {{#webhook_trigger.page#}}
```

### 请求头参数

从 HTTP 头部提取：

```yaml
- data:
    type: webhook-trigger
    webhook:
      method: POST
      parameters:
      - name: api_key
        type: string
        source: header
        required: true
      - name: user_agent
        type: string
        source: header
        required: false
  id: webhook_trigger

# 请求头示例:
# Authorization: Bearer xxx
# User-Agent: Mozilla/5.0
```

### 请求体参数

从请求正文提取：

```yaml
- data:
    type: webhook-trigger
    webhook:
      method: POST
      content_type: application/json
      parameters:
      - name: order_id
        type: string
        source: body
        required: true
      - name: customer_email
        type: string
        source: body
        required: true
      - name: total_amount
        type: number
        source: body
        required: true
  id: webhook_trigger

# 请求体示例:
# {
#   "order_id": "ORD-12345",
#   "customer_email": "user@example.com",
#   "total_amount": 99.99
# }
```

## 响应配置

### 默认响应

```yaml
webhook:
  response:
    status_code: 200
    body: '{"status": "success"}'
```

### 自定义响应

```yaml
webhook:
  response:
    status_code: 201
    content_type: application/json
    body: |
      {
        "status": "received",
        "message": "Order processing started",
        "order_id": "{{#webhook_trigger.order_id#}}"
      }
```

### 动态响应

使用工作流变量：

```yaml
webhook:
  response:
    status_code: 200
    body: |
      {
        "result": "{{#process.result#}}",
        "timestamp": "{{#sys.timestamp#}}"
      }
```

## 输出变量

提取的参数自动成为输出变量：

```yaml
{{#webhook_trigger.参数名#}}    # 提取的参数值
{{#webhook_trigger.raw_body#}}  # 原始请求体
{{#webhook_trigger.headers#}}   # 所有请求头
```

## 实际案例

### 案例1：订单处理

```yaml
- data:
    type: webhook-trigger
    webhook:
      method: POST
      content_type: application/json
      parameters:
      - name: order_id
        type: string
        source: body
        required: true
      - name: customer_email
        type: string
        source: body
        required: true
      - name: items
        type: array
        source: body
        required: true
      - name: total_amount
        type: number
        source: body
        required: true
      response:
        status_code: 200
        body: '{"status": "received", "order_id": "{{#webhook_trigger.order_id#}}"}'
  id: order_webhook

# 处理订单
- data:
    type: code
    code: |
      def main(order_id, items, total):
          # 处理订单逻辑
          return {
              "processed": True,
              "order_id": order_id
          }
    inputs:
      order_id: "{{#order_webhook.order_id#}}"
      items: "{{#order_webhook.items#}}"
      total: "{{#order_webhook.total_amount#}}"
  id: process_order

# 发送确认邮件
- data:
    type: tool
    tool_name: send_email
    parameters:
      to: "{{#order_webhook.customer_email#}}"
      subject: "订单确认"
      body: "您的订单 {{#order_webhook.order_id#}} 已收到"
  id: send_confirmation
```

### 案例2：GitHub Webhook

```yaml
- data:
    type: webhook-trigger
    webhook:
      method: POST
      content_type: application/json
      parameters:
      - name: event_type
        type: string
        source: header
        required: true
      - name: repository
        type: string
        source: body
        required: true
      - name: action
        type: string
        source: body
        required: true
      - name: pull_request
        type: object
        source: body
        required: false
      response:
        status_code: 200
        body: '{"status": "ok"}'
  id: github_webhook

# 分类事件
- data:
    type: if-else
    conditions:
    - variable: "{{#github_webhook.event_type#}}"
      operator: "equals"
      value: "pull_request"
  id: check_event_type

# 处理 PR
- data:
    type: llm
    model:
      provider: openai
      name: gpt-4
    prompt_template:
    - role: user
      text: |
        分析 PR:
        仓库: {{#github_webhook.repository#}}
        标题: {{#github_webhook.pull_request.title#}}
        描述: {{#github_webhook.pull_request.body#}}
  id: analyze_pr
```

### 案例3：支付回调

```yaml
- data:
    type: webhook-trigger
    webhook:
      method: POST
      content_type: application/json
      parameters:
      - name: transaction_id
        type: string
        source: body
        required: true
      - name: status
        type: string
        source: body
        required: true
      - name: amount
        type: number
        source: body
        required: true
      - name: signature
        type: string
        source: header
        required: true
      response:
        status_code: 200
        body: '{"code": 0, "message": "success"}'
  id: payment_webhook

# 验证签名
- data:
    type: code
    code: |
      import hashlib
      def main(signature, transaction_id, amount):
          # 验证签名逻辑
          expected = hashlib.sha256(f"{transaction_id}{amount}".encode()).hexdigest()
          return {"valid": signature == expected}
    inputs:
      signature: "{{#payment_webhook.signature#}}"
      transaction_id: "{{#payment_webhook.transaction_id#}}"
      amount: "{{#payment_webhook.amount#}}"
  id: verify_signature

# 更新订单状态
- data:
    type: http-request
    method: POST
    url: "https://api.example.com/orders/update"
    body:
      transaction_id: "{{#payment_webhook.transaction_id#}}"
      status: "{{#payment_webhook.status#}}"
  id: update_order
```

### 案例4：表单提交

```yaml
- data:
    type: webhook-trigger
    webhook:
      method: POST
      content_type: application/x-www-form-urlencoded
      parameters:
      - name: name
        type: string
        source: body
        required: true
      - name: email
        type: string
        source: body
        required: true
      - name: message
        type: string
        source: body
        required: true
      response:
        status_code: 200
        content_type: text/html
        body: '<html><body><h1>感谢您的提交！</h1></body></html>'
  id: form_webhook

# 保存到数据库
- data:
    type: http-request
    method: POST
    url: "https://api.example.com/submissions"
    body:
      name: "{{#form_webhook.name#}}"
      email: "{{#form_webhook.email#}}"
      message: "{{#form_webhook.message#}}"
      timestamp: "{{#sys.timestamp#}}"
  id: save_submission

# 发送通知
- data:
    type: tool
    tool_name: send_email
    parameters:
      to: "admin@example.com"
      subject: "新表单提交"
      body: "来自 {{#form_webhook.name#}} ({{#form_webhook.email#}})"
  id: notify_admin
```

## 使用场景

### 第三方集成
- 支付网关回调
- 物流状态更新
- 短信发送回执
- 第三方 API 通知

### 表单处理
- 网站表单提交
- 用户注册
- 反馈收集
- 调查问卷

### 系统集成
- 内部系统通知
- 微服务通信
- 事件总线集成
- 数据同步

### 自动化流程
- CI/CD 触发
- 部署通知
- 监控告警
- 日志收集

## 最佳实践

### 1. 安全性
- 实现请求签名验证
- 使用 HTTPS 传输
- 验证请求来源 IP
- 限制请求频率

```yaml
# 签名验证示例
- data:
    type: code
    code: |
      import hmac
      import hashlib
      def main(signature, body, secret):
          expected = hmac.new(
              secret.encode(),
              body.encode(),
              hashlib.sha256
          ).hexdigest()
          if signature != expected:
              raise ValueError("Invalid signature")
          return {"valid": True}
  id: verify_signature
```

### 2. 参数验证
- 标记必需参数
- 验证数据类型
- 检查参数范围
- 提供默认值

### 3. 错误处理
- 返回有意义的错误信息
- 记录失败请求
- 实现重试机制
- 提供降级方案

### 4. 幂等性
- 处理重复请求
- 使用唯一标识符
- 检查处理状态
- 避免重复操作

```yaml
# 幂等性检查
- data:
    type: code
    code: |
      def main(transaction_id, processed_ids):
          if transaction_id in processed_ids:
              return {"skip": True, "reason": "Already processed"}
          return {"skip": False}
  id: check_idempotency
```

### 5. 响应设计
- 快速返回响应
- 异步处理耗时操作
- 提供处理状态
- 包含必要的元数据

## 测试流程

### 本地测试

使用 curl 命令：

```bash
# POST 请求
curl -X POST \
  https://api.dify.ai/webhook/xxx \
  -H "Content-Type: application/json" \
  -d '{"order_id": "123", "amount": 99.99}'

# GET 请求
curl -X GET \
  "https://api.dify.ai/webhook/xxx?user_id=123&page=1"

# 带认证头
curl -X POST \
  https://api.dify.ai/webhook/xxx \
  -H "Authorization: Bearer token123" \
  -H "Content-Type: application/json" \
  -d '{"data": "test"}'
```

### 测试工具

推荐使用：
- Postman
- Insomnia
- HTTPie
- curl

### 测试步骤

1. 点击"运行此步骤"进入监听模式
2. 发送测试请求
3. 查看"最后运行"日志
4. 验证接收到的数据
5. 检查响应内容

## 环境配置

### 自托管部署

设置 `TRIGGER_URL` 环境变量：

```bash
# .env 文件
TRIGGER_URL=https://your-domain.com
```

### URL 格式

```
https://{TRIGGER_URL}/api/v1/workflows/trigger/{workflow_id}/{trigger_id}
```

### 测试 vs 生产

- 测试 URL：用于开发和调试
- 生产 URL：用于实际业务
- 分离数据避免混淆

## 常见问题

### Q: 如何保护 webhook 端点？

A:
1. 实现签名验证
2. 使用 API 密钥认证
3. 限制来源 IP
4. 启用 HTTPS
5. 实现速率限制

### Q: 如何处理大量并发请求？

A:
1. 使用队列机制
2. 实现异步处理
3. 增加工作流实例
4. 优化处理逻辑
5. 考虑使用缓存

### Q: webhook 没有触发怎么办？

A:
1. 检查 URL 是否正确
2. 验证 HTTP 方法匹配
3. 确认内容类型正确
4. 查看工作流日志
5. 测试网络连接

### Q: 如何调试 webhook？

A:
1. 使用"最后运行"日志
2. 查看原始请求数据
3. 使用测试工具发送请求
4. 添加日志节点记录数据
5. 检查参数提取配置

### Q: 响应超时怎么办？

A:
1. 快速返回响应
2. 异步处理耗时操作
3. 优化工作流性能
4. 增加超时时间
5. 使用队列延迟处理

## 监控和维护

### 监控指标
- 请求频率
- 成功率
- 响应时间
- 错误类型

### 日志记录
- 记录所有请求
- 保存失败详情
- 追踪处理状态
- 分析异常模式

### 维护任务
- 定期检查端点状态
- 更新安全配置
- 优化处理性能
- 清理过期日志

## 相关节点

- [触发器概览](./trigger.md) - 触发器类型总览
- [定时触发器](./schedule-trigger.md) - 定时任务
- [插件触发器](./plugin-trigger.md) - 外部系统集成
- [HTTP 请求节点](./http-request.md) - 发送 HTTP 请求

## 返回

[工作流 SKILL](../SKILL.md)
