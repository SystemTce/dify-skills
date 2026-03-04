# 触发器节点详解

## 概述

触发器是启动节点，使工作流能够按计划或在外部事件发生时自动运行，而不需要手动激活或 API 调用。

## 核心概念

触发器让工作流从被动响应转变为主动执行，支持定时任务和事件驱动的自动化场景。

## 触发器类型

### 1. 定时触发器 (Schedule Trigger)

按指定时间运行工作流，每个工作流最多一个定时触发器。

**使用场景**:
- 每日销售报告生成
- 定期数据同步
- 定时备份任务
- 周期性监控检查

**配置示例**:
```yaml
- data:
    type: schedule-trigger
    schedule:
      type: cron
      cron: "0 9 * * *"  # 每天上午9点
  id: daily_trigger
```

### 2. 插件触发器 (Plugin Trigger)

订阅集成系统（如 Slack、Gmail）的事件，当订阅的事件发生时激活工作流。

**使用场景**:
- Slack 消息触发
- Gmail 邮件接收
- GitHub 事件通知
- 第三方系统集成

**配置示例**:
```yaml
- data:
    type: plugin-trigger
    plugin:
      provider: slack
      event: message_received
      subscription_id: sub_123
  id: slack_trigger
```

### 3. Webhook 触发器 (Webhook Trigger)

使用自定义 webhook 监听来自第三方系统的外部 HTTP 请求。

**使用场景**:
- 订单处理
- 数据同步
- 自定义集成
- 外部系统通知

**配置示例**:
```yaml
- data:
    type: webhook-trigger
    webhook:
      method: POST
      content_type: application/json
      parameters:
      - name: order_id
        type: string
        required: true
      - name: amount
        type: number
        required: true
  id: webhook_trigger
```

## 关键特性

### 多触发器支持
- 一个工作流可以有多个并行触发器
- 不同触发器可以独立启用/禁用
- 触发源会显示在工作流日志中

### 触发器管理
- 已发布的触发器可通过快速设置启用/禁用
- 测试模式支持同时运行所有触发器
- 每个触发器都有独立的配置和状态

### 配额限制
- 沙盒计划：每个工作流最多 2 个触发器
- 云部署：根据定价层级有配额限制
- 定时触发器：每个工作流只能有一个

## 选择指南

### 何时使用定时触发器
- 需要按固定时间表执行任务
- 周期性数据处理
- 定时报告生成
- 计划性维护任务

### 何时使用插件触发器
- 集成的系统有可用的插件
- 需要响应特定平台的事件
- 官方支持的集成场景
- 需要可靠的事件订阅

### 何时使用 Webhook 触发器
- 插件不支持所需的事件
- 自定义集成需求
- 需要灵活的 HTTP 接口
- 与内部系统集成

## 实际案例

### 案例1：每日报告生成

```yaml
- data:
    type: schedule-trigger
    schedule:
      type: cron
      cron: "0 8 * * 1-5"  # 工作日早上8点
  id: daily_report_trigger

# 连接到数据查询和报告生成节点
```

### 案例2：Slack 消息响应

```yaml
- data:
    type: plugin-trigger
    plugin:
      provider: slack
      event: message_received
      channel: "#support"
      subscription_id: sub_slack_001
  id: slack_message_trigger

# 连接到消息处理和回复节点
```

### 案例3：订单处理 Webhook

```yaml
- data:
    type: webhook-trigger
    webhook:
      method: POST
      content_type: application/json
      parameters:
      - name: order_id
        type: string
        required: true
        source: body
      - name: customer_email
        type: string
        required: true
        source: body
      - name: total_amount
        type: number
        required: true
        source: body
      response:
        status_code: 200
        body: '{"status": "received"}'
  id: order_webhook
```

### 案例4：多触发器工作流

```yaml
# 定时触发器 - 每小时检查
- data:
    type: schedule-trigger
    schedule:
      type: cron
      cron: "0 * * * *"
  id: hourly_check

# Webhook 触发器 - 手动触发
- data:
    type: webhook-trigger
    webhook:
      method: POST
      content_type: application/json
  id: manual_trigger

# 两个触发器连接到同一个处理流程
```

## 最佳实践

### 1. 定时触发器
- 使用 Cron 表达式实现复杂调度
- 考虑时区设置
- 避免在高峰时段执行重任务
- 设置合理的执行频率

### 2. 插件触发器
- 优先使用官方插件
- 订阅所有可用事件以便复用
- 配置适当的过滤条件
- 测试事件接收和处理

### 3. Webhook 触发器
- 实现适当的认证机制
- 验证请求来源
- 处理重复请求
- 返回有意义的响应

### 4. 通用建议
- 为触发器设置清晰的命名
- 在日志中记录触发源
- 实现错误处理和重试
- 监控触发器执行状态

## 性能考虑

### 定时触发器
- 避免过于频繁的执行
- 考虑任务执行时间
- 错开多个定时任务

### 事件触发器
- 处理高频事件时注意限流
- 实现队列机制处理突发流量
- 考虑并发执行限制

## 常见问题

### Q: 一个工作流可以有多少个触发器？

A:
- 定时触发器：最多 1 个
- 其他触发器：沙盒计划最多 2 个，云部署根据定价层级
- 可以混合使用不同类型的触发器

### Q: 如何测试触发器？

A:
- 定时触发器：可以立即执行或等待下次计划时间
- Webhook 触发器：点击"运行此步骤"进入监听状态，然后发送测试请求
- 插件触发器：点击"运行此步骤"后触发相应的外部事件

### Q: 触发器失败了怎么办？

A:
1. 检查触发器配置是否正确
2. 查看工作流日志了解失败原因
3. 验证外部系统的连接和权限
4. 实现错误处理和通知机制

### Q: 如何选择合适的触发器类型？

A:
- 定时任务 → 定时触发器
- 已有插件支持 → 插件触发器
- 自定义集成 → Webhook 触发器
- 优先使用插件触发器，其次是 Webhook

## 相关节点

- [定时触发器详解](./schedule-trigger.md)
- [Webhook 触发器详解](./webhook-trigger.md)
- [插件触发器详解](./plugin-trigger.md)

## 返回

[工作流 SKILL](../SKILL.md)
