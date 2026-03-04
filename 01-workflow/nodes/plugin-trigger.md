# 插件触发器节点详解

## 概述

插件触发器使工作流能够在外部事件发生时自动执行。通过订阅外部系统的事件，当这些事件发生时，工作流会自动运行。

## 核心概念

插件触发器通过事件订阅机制工作。例如，GitHub 触发器插件可能提供 Pull Request、Push 或 Issue 事件的订阅。当你将相应的触发器添加到工作流中，这些事件发生时工作流就会自动运行。

## 核心功能

- 订阅外部系统事件
- 自动响应事件触发
- 支持多种集成平台
- 灵活的订阅管理

## 设置流程

### 1. 添加触发器节点

在工作流画布中：
1. 点击"添加节点"
2. 选择"开始" > "触发器"
3. 选择可用的插件触发器

### 2. 配置订阅

选择现有订阅或创建新订阅：

```yaml
- data:
    type: plugin-trigger
    plugin:
      provider: github
      event: pull_request
      subscription_id: sub_github_001
  id: github_trigger
```

## 订阅方法

### 自动创建

Dify 使用 OAuth 或 API 密钥在外部系统中自动设置 webhook：

```yaml
- data:
    type: plugin-trigger
    plugin:
      provider: slack
      event: message_received
      subscription:
        method: auto
        credentials:
          oauth_token: "{{#workspace.slack_token#}}"
  id: slack_auto_trigger
```

**优势**:
- 自动配置 webhook
- 无需手动操作
- 集中管理凭证

### 手动创建

使用 Dify 提供的回调 URL 手动配置 webhook：

```yaml
- data:
    type: plugin-trigger
    plugin:
      provider: custom_service
      event: data_updated
      subscription:
        method: manual
        callback_url: "https://your-dify.com/api/v1/workflows/trigger/..."
  id: manual_trigger
```

**适用场景**:
- 不支持自动配置的系统
- 需要自定义配置
- 内部系统集成

## 关键约束

### 应用类型限制
- 仅适用于工作流应用
- 不支持对话流应用

### 订阅配额
- 每个工作区每个触发器插件最多 10 个订阅
- 云部署可能有不同的配额限制

### 输出变量
- 由触发器插件定义
- 不能修改变量结构
- 可在下游节点中引用

## 常见插件类型

### 通信平台

**Slack**:
```yaml
- data:
    type: plugin-trigger
    plugin:
      provider: slack
      event: message_received
      channel: "#support"
      subscription_id: sub_slack_001
  id: slack_trigger
```

**Discord**:
```yaml
- data:
    type: plugin-trigger
    plugin:
      provider: discord
      event: message_created
      channel_id: "123456789"
      subscription_id: sub_discord_001
  id: discord_trigger
```

### 开发平台

**GitHub**:
```yaml
- data:
    type: plugin-trigger
    plugin:
      provider: github
      event: pull_request
      repository: "owner/repo"
      subscription_id: sub_github_001
  id: github_trigger
```

**GitLab**:
```yaml
- data:
    type: plugin-trigger
    plugin:
      provider: gitlab
      event: merge_request
      project_id: "12345"
      subscription_id: sub_gitlab_001
  id: gitlab_trigger
```

### 邮件服务

**Gmail**:
```yaml
- data:
    type: plugin-trigger
    plugin:
      provider: gmail
      event: email_received
      label: "INBOX"
      subscription_id: sub_gmail_001
  id: gmail_trigger
```

## 输出变量

插件触发器的输出变量由插件定义，常见的包括：

```yaml
# Slack 消息触发器
{{#slack_trigger.message_text#}}      # 消息内容
{{#slack_trigger.user_id#}}           # 发送者 ID
{{#slack_trigger.channel_id#}}        # 频道 ID
{{#slack_trigger.timestamp#}}         # 时间戳

# GitHub PR 触发器
{{#github_trigger.pr_number#}}        # PR 编号
{{#github_trigger.pr_title#}}         # PR 标题
{{#github_trigger.author#}}           # 作者
{{#github_trigger.repository#}}       # 仓库名称

# Gmail 触发器
{{#gmail_trigger.subject#}}           # 邮件主题
{{#gmail_trigger.from#}}              # 发件人
{{#gmail_trigger.body#}}              # 邮件正文
{{#gmail_trigger.attachments#}}       # 附件列表
```

## 实际案例

### 案例1：Slack 客服自动回复

```yaml
- data:
    type: plugin-trigger
    plugin:
      provider: slack
      event: message_received
      channel: "#customer-support"
      subscription_id: sub_slack_support
  id: slack_message

# 连接到 LLM 节点处理消息
- data:
    type: llm
    model:
      provider: openai
      name: gpt-4
    prompt_template:
    - role: system
      text: "你是客服助手"
    - role: user
      text: "{{#slack_message.message_text#}}"
  id: process_message

# 回复到 Slack
- data:
    type: tool
    tool_name: slack_send_message
    parameters:
      channel: "{{#slack_message.channel_id#}}"
      text: "{{#process_message.text#}}"
  id: send_reply
```

### 案例2：GitHub PR 自动审查

```yaml
- data:
    type: plugin-trigger
    plugin:
      provider: github
      event: pull_request
      repository: "company/project"
      subscription_id: sub_github_pr
  id: pr_trigger

# 分析 PR 内容
- data:
    type: llm
    model:
      provider: anthropic
      name: claude-3-opus
    prompt_template:
    - role: user
      text: |
        审查以下 PR:
        标题: {{#pr_trigger.pr_title#}}
        描述: {{#pr_trigger.pr_description#}}

        提供审查意见
  id: review_pr

# 发布审查评论
- data:
    type: tool
    tool_name: github_create_comment
    parameters:
      pr_number: "{{#pr_trigger.pr_number#}}"
      comment: "{{#review_pr.text#}}"
  id: post_comment
```

### 案例3：Gmail 邮件分类

```yaml
- data:
    type: plugin-trigger
    plugin:
      provider: gmail
      event: email_received
      label: "INBOX"
      subscription_id: sub_gmail_inbox
  id: email_trigger

# 分类邮件
- data:
    type: question-classifier
    query: "{{#email_trigger.subject#}} {{#email_trigger.body#}}"
    classes:
    - name: urgent
      description: "紧急邮件"
    - name: normal
      description: "普通邮件"
    - name: spam
      description: "垃圾邮件"
  id: classify_email

# 根据分类处理
- data:
    type: if-else
    conditions:
    - variable: "{{#classify_email.class_name#}}"
      operator: "equals"
      value: "urgent"
  id: check_urgent
```

## 最佳实践

### 1. 订阅管理
- 创建订阅时选择所有可用事件
- 允许未来的触发器使用同一订阅
- 避免重复创建订阅
- 定期清理未使用的订阅

### 2. 事件过滤
- 在触发器层面配置过滤条件
- 使用 If-Else 节点进一步筛选
- 避免处理不相关的事件
- 减少不必要的工作流执行

### 3. 错误处理
- 实现事件处理失败的通知
- 记录详细的执行日志
- 设置重试机制
- 提供降级方案

### 4. 安全考虑
- 验证事件来源
- 使用安全的凭证管理
- 限制触发器权限
- 监控异常活动

### 5. 性能优化
- 处理高频事件时注意限流
- 使用队列机制处理突发流量
- 优化工作流执行时间
- 考虑并发执行限制

## 测试流程

### 测试未发布的触发器

1. 点击"运行此步骤"进入监听状态
2. 在外部系统触发相应事件
3. 查看触发器的"最后运行"日志
4. 验证接收到的数据

### 测试已发布的触发器

1. 确保触发器已启用
2. 在外部系统触发事件
3. 查看工作流执行日志
4. 验证完整的执行流程

## 常见问题

### Q: 如何选择订阅方法？

A:
- 优先使用自动创建（如果支持）
- 自动创建更简单、更可靠
- 手动创建适用于自定义场景
- 考虑系统的 API 支持情况

### Q: 订阅配额不够怎么办？

A:
1. 复用现有订阅
2. 创建订阅时选择所有事件
3. 使用 If-Else 节点过滤事件
4. 联系管理员增加配额

### Q: 触发器没有响应怎么办？

A:
1. 检查订阅状态是否正常
2. 验证外部系统的 webhook 配置
3. 查看工作流日志了解错误
4. 确认触发器已启用
5. 测试网络连接和权限

### Q: 如何处理重复事件？

A:
1. 在工作流中实现去重逻辑
2. 使用会话变量记录已处理事件
3. 检查事件 ID 或时间戳
4. 设置合理的处理窗口

### Q: 输出变量不符合预期怎么办？

A:
1. 查看插件文档了解变量结构
2. 使用模板节点查看原始数据
3. 测试触发器查看实际输出
4. 联系插件开发者获取支持

## 监控和维护

### 监控指标
- 触发频率
- 执行成功率
- 平均执行时间
- 错误类型和频率

### 维护任务
- 定期检查订阅状态
- 更新过期的凭证
- 清理未使用的触发器
- 优化工作流性能

## 相关节点

- [触发器概览](./trigger.md) - 触发器类型总览
- [定时触发器](./schedule-trigger.md) - 定时任务
- [Webhook 触发器](./webhook-trigger.md) - 自定义 webhook

## 返回

[工作流 SKILL](../SKILL.md)
