# 工具节点详解

## 概述

工具节点通过预构建的集成将工作流连接到外部服务和 API，提供结构化接口、内置错误处理和简化配置。

## 核心功能

- 结构化接口配置
- 内置错误处理和重试逻辑
- 类型安全和数据验证
- 自动重试机制（最多 10 次，可配置间隔最长 5000ms）
- 集中凭证管理
- 预定义输入/输出 schema

## 工具类型

### 内置工具

Dify 维护的流行服务工具：
- Google Search
- Weather API
- Calculator
- Time/Date
- 等等

### 自定义工具

通过 OpenAPI/Swagger 规范导入：

```yaml
- data:
    type: tool
    provider_name: custom
    tool_name: my_api
    tool_parameters:
      endpoint: "{{#start.endpoint#}}"
      api_key: "{{#start.api_key#}}"
  id: custom_tool
```

### 工作流工具

将复杂工作流发布为可重用组件：

```yaml
- data:
    type: tool
    provider_name: workflow
    tool_name: data_processor
    tool_parameters:
      input_data: "{{#start.data#}}"
  id: workflow_tool
```

### MCP 工具

来自外部 Model Context Protocol 服务器：

```yaml
- data:
    type: tool
    provider_name: mcp
    tool_name: external_service
    tool_parameters:
      query: "{{#start.query#}}"
  id: mcp_tool
```

## 配置要点

### 认证设置

在工作区的工具部分配置 API 凭证。

### 参数配置

使用结构化表单和验证：

```yaml
tool_parameters:
  query: "{{#start.search_query#}}"
  limit: 10
  language: "zh-CN"
```

### 错误处理

```yaml
retry:
  enabled: true
  max_retries: 3
  retry_interval: 1000

error_handling:
  enabled: true
  on_error:
    target: error_handler
```

## 工具 vs HTTP 请求

| 特性 | 工具节点 | HTTP 请求节点 |
|------|---------|--------------|
| 配置 | 结构化表单 | 手动配置 |
| 错误处理 | 内置 | 需要配置 |
| 类型安全 | 是 | 否 |
| 文档 | 包含 | 无 |
| 灵活性 | 中等 | 高 |

## 实际案例

### 案例1：Google 搜索

```yaml
- data:
    type: tool
    provider_name: google
    tool_name: search
    tool_parameters:
      query: "{{#start.search_query#}}"
      num_results: 5
  id: google_search
```

### 案例2：天气查询

```yaml
- data:
    type: tool
    provider_name: weather
    tool_name: get_weather
    tool_parameters:
      city: "{{#start.city#}}"
      units: "metric"
  id: weather_tool
```

### 案例3：自定义 API

```yaml
- data:
    type: tool
    provider_name: custom
    tool_name: crm_api
    tool_parameters:
      customer_id: "{{#start.customer_id#}}"
      action: "get_profile"
    retry:
      enabled: true
      max_retries: 3
  id: crm_tool
```

## 使用场景

### 外部服务集成
- 搜索引擎（Google、Bing）
- 天气服务
- 地图和位置服务
- 翻译服务

### 数据处理
- 计算器和数学运算
- 日期时间处理
- 数据转换
- 格式化工具

### 业务系统
- CRM 系统
- ERP 系统
- 支付网关
- 通知服务

### 开发工具
- 代码执行
- API 测试
- 数据验证
- 格式转换

## 最佳实践

### 1. 工具选择
- 优先使用内置工具（更可靠）
- 有官方插件时使用插件
- 复杂逻辑考虑自定义工具
- 简单 HTTP 调用可用 HTTP 请求节点

### 2. 错误处理
- 配置重试策略处理临时故障
- 实现错误处理分支
- 记录失败详情
- 提供降级方案

### 3. 参数管理
- 使用变量传递动态参数
- 验证参数格式和范围
- 提供合理的默认值
- 文档化参数要求

### 4. 凭证管理
- 在工作区级别管理 API 密钥
- 不要在工作流中硬编码凭证
- 定期轮换密钥
- 限制凭证权限

### 5. 性能优化
- 缓存工具结果
- 并行调用独立工具
- 设置合理的超时时间
- 监控工具响应时间

## 工具管理

### 访问工具配置
在工作区的"工具"部分：
- 管理 API 凭证
- 导入自定义工具
- 配置 MCP 服务器
- 发布工作流为工具

### 导入自定义工具
1. 准备 OpenAPI/Swagger 规范
2. 在工具部分点击"导入"
3. 配置认证信息
4. 测试工具连接
5. 在工作流中使用

### 发布工作流工具
1. 创建完整的工作流
2. 定义输入输出参数
3. 测试工作流功能
4. 发布为工具
5. 在其他工作流中复用

## 常见问题

### Q: 工具节点和 HTTP 请求节点如何选择？

A:
- 有可用工具 → 工具节点
- 需要灵活配置 → HTTP 请求节点
- 简单 API 调用 → HTTP 请求节点
- 需要错误处理和重试 → 工具节点

### Q: 如何处理工具调用失败？

A:
1. 启用自动重试
2. 配置错误处理分支
3. 记录失败详情
4. 实现降级逻辑
5. 通知相关人员

### Q: 工具调用有速率限制吗？

A:
- 取决于外部服务的限制
- 实现速率限制逻辑
- 使用队列管理请求
- 监控调用频率

### Q: 如何调试工具调用？

A:
1. 查看工具执行日志
2. 检查参数是否正确
3. 验证凭证配置
4. 测试工具连接
5. 使用模板节点显示输出

### Q: 可以创建自己的工具吗？

A:
- 使用 OpenAPI 规范导入
- 开发 Dify 插件
- 发布工作流为工具
- 使用 MCP 协议集成

## 相关节点

- [HTTP 请求节点](./http-request.md) - 灵活的 HTTP 接口
- [Agent 节点](./agent.md) - 自主工具调用
- [Code 节点](./code.md) - 处理工具输出

## 返回

[工作流 SKILL](../SKILL.md)
