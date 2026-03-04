# 人工介入节点详解

## 概述

人工介入节点在关键节点暂停工作流，请求人工干预后再继续执行。它在自动化效率和人工监督之间取得平衡，在关键工作流阶段嵌入人工判断。

## 核心功能

- 在关键点暂停工作流
- 发送自定义请求表单
- 收集人工输入和决策
- 根据响应路由工作流

## 核心价值

通过在关键阶段嵌入人工判断，实现：
- 质量控制和审核
- 关键决策确认
- 数据验证和修正
- 异常情况处理

## 配置区域

### 1. 交付方式

**Web App**:
直接在界面中显示请求表单（仅当前用户）

```yaml
- data:
    type: human-input
    delivery:
      method: webapp
  id: human_review
```

**Email**:
发送请求链接到一个或多个收件人

```yaml
- data:
    type: human-input
    delivery:
      method: email
      recipients:
      - user1@example.com
      - user2@example.com
  id: human_review
```

**注意**: 请求在收到第一个响应后关闭

### 2. 表单内容

使用 Markdown 格式和动态变量自定义表单：

```yaml
- data:
    type: human-input
    form:
      title: "内容审核"
      description: |
        ## 请审核以下内容

        **生成的文章**:
        {{#llm.text#}}

        **关键词**: {{#start.keywords#}}
        **目标受众**: {{#start.audience#}}

        请确认内容是否符合要求。
      fields:
      - name: feedback
        type: textarea
        label: "修改建议"
        required: false
      - name: rating
        type: number
        label: "质量评分 (1-10)"
        required: true
        min: 1
        max: 10
  id: content_review
```

### 3. 用户操作

定义决策按钮路由工作流：

```yaml
- data:
    type: human-input
    actions:
    - id: approve
      label: "批准发布"
      style: primary
    - id: reject
      label: "拒绝"
      style: danger
    - id: revise
      label: "需要修改"
      style: secondary
  id: approval_decision
```

### 4. 超时策略

设置等待响应的时间：

```yaml
- data:
    type: human-input
    timeout:
      duration: 3600  # 秒
      action: end     # 或 fallback
  id: human_review
```

**超时动作**:
- `end`: 自动结束工作流
- `fallback`: 执行备用分支

## 输入字段类型

### 文本输入

```yaml
fields:
- name: comment
  type: text
  label: "评论"
  placeholder: "请输入您的评论"
  required: true
```

### 多行文本

```yaml
fields:
- name: feedback
  type: textarea
  label: "详细反馈"
  rows: 5
  required: false
```

### 数字输入

```yaml
fields:
- name: score
  type: number
  label: "评分"
  min: 1
  max: 10
  default: 5
  required: true
```

### 选择框

```yaml
fields:
- name: category
  type: select
  label: "分类"
  options:
  - value: tech
    label: "技术"
  - value: business
    label: "商业"
  - value: other
    label: "其他"
  required: true
```

### 复选框

```yaml
fields:
- name: confirmed
  type: checkbox
  label: "我已确认所有信息"
  required: true
```

### 预填充字段

使用上游节点的变量预填充：

```yaml
fields:
- name: title
  type: text
  label: "标题"
  default: "{{#llm.title#}}"
  required: true
```

## 输出变量

```yaml
{{#human_review.action#}}           # 用户选择的操作
{{#human_review.feedback#}}         # 用户输入的反馈
{{#human_review.rating#}}           # 用户输入的评分
{{#human_review.responded_at#}}     # 响应时间
{{#human_review.responded_by#}}     # 响应者
{{#human_review.timeout#}}          # 是否超时
```

## 实际案例

### 案例1：内容审核发布

```yaml
# 生成内容
- data:
    type: llm
    model:
      provider: openai
      name: gpt-4
    prompt_template:
    - role: user
      text: "写一篇关于 {{#start.topic#}} 的文章"
  id: generate_content

# 人工审核
- data:
    type: human-input
    delivery:
      method: webapp
    form:
      title: "内容审核"
      description: |
        ## 请审核生成的内容

        {{#generate_content.text#}}

        请确认是否可以发布。
      fields:
      - name: feedback
        type: textarea
        label: "修改建议"
        required: false
    actions:
    - id: publish
      label: "批准发布"
      style: primary
    - id: regenerate
      label: "重新生成"
      style: secondary
    timeout:
      duration: 7200
      action: end
  id: content_review

# 根据决策分支
- data:
    type: if-else
    conditions:
    - variable: "{{#content_review.action#}}"
      operator: "equals"
      value: "publish"
  id: check_decision

# 发布内容
- data:
    type: http-request
    method: POST
    url: "https://api.example.com/publish"
    body:
      content: "{{#generate_content.text#}}"
      feedback: "{{#content_review.feedback#}}"
  id: publish_content

# 重新生成
- data:
    type: llm
    model:
      provider: openai
      name: gpt-4
    prompt_template:
    - role: user
      text: |
        重新生成文章，考虑以下反馈:
        {{#content_review.feedback#}}

        原文: {{#generate_content.text#}}
  id: regenerate_content
```

### 案例2：数据验证

```yaml
# 提取数据
- data:
    type: parameter-extractor
    input: "{{#start.document#}}"
    parameters:
    - name: company_name
      type: string
    - name: amount
      type: number
    - name: date
      type: string
  id: extract_data

# 人工验证
- data:
    type: human-input
    delivery:
      method: email
      recipients:
      - validator@example.com
    form:
      title: "数据验证"
      description: "请验证提取的数据是否正确"
      fields:
      - name: company_name
        type: text
        label: "公司名称"
        default: "{{#extract_data.company_name#}}"
        required: true
      - name: amount
        type: number
        label: "金额"
        default: "{{#extract_data.amount#}}"
        required: true
      - name: date
        type: text
        label: "日期"
        default: "{{#extract_data.date#}}"
        required: true
      - name: notes
        type: textarea
        label: "备注"
        required: false
    actions:
    - id: confirm
      label: "确认"
    - id: reject
      label: "拒绝"
    timeout:
      duration: 86400  # 24小时
      action: fallback
  id: validate_data

# 保存验证后的数据
- data:
    type: http-request
    method: POST
    url: "https://api.example.com/save"
    body:
      company: "{{#validate_data.company_name#}}"
      amount: "{{#validate_data.amount#}}"
      date: "{{#validate_data.date#}}"
      notes: "{{#validate_data.notes#}}"
  id: save_data
```

### 案例3：异常处理

```yaml
# 执行任务
- data:
    type: code
    code: |
      def main(data):
          # 处理逻辑
          if error_detected:
              return {"status": "error", "message": "异常情况"}
          return {"status": "success"}
  id: process_task

# 检查错误
- data:
    type: if-else
    conditions:
    - variable: "{{#process_task.status#}}"
      operator: "equals"
      value: "error"
  id: check_error

# 人工介入处理异常
- data:
    type: human-input
    delivery:
      method: email
      recipients:
      - admin@example.com
    form:
      title: "异常处理"
      description: |
        ## 检测到异常情况

        **错误信息**: {{#process_task.message#}}

        请选择处理方式。
      fields:
      - name: resolution
        type: textarea
        label: "处理方案"
        required: true
    actions:
    - id: retry
      label: "重试"
    - id: skip
      label: "跳过"
    - id: manual
      label: "手动处理"
    timeout:
      duration: 3600
      action: end
  id: handle_exception
```

### 案例4：多级审批

```yaml
# 第一级审批
- data:
    type: human-input
    delivery:
      method: email
      recipients:
      - manager@example.com
    form:
      title: "经理审批"
      description: "请审批以下申请"
      fields:
      - name: manager_comment
        type: textarea
        label: "审批意见"
    actions:
    - id: approve
      label: "批准"
    - id: reject
      label: "拒绝"
  id: manager_approval

# 检查第一级结果
- data:
    type: if-else
    conditions:
    - variable: "{{#manager_approval.action#}}"
      operator: "equals"
      value: "approve"
  id: check_manager

# 第二级审批
- data:
    type: human-input
    delivery:
      method: email
      recipients:
      - director@example.com
    form:
      title: "总监审批"
      description: |
        经理已批准，请进行最终审批

        **经理意见**: {{#manager_approval.manager_comment#}}
      fields:
      - name: director_comment
        type: textarea
        label: "审批意见"
    actions:
    - id: approve
      label: "最终批准"
    - id: reject
      label: "拒绝"
  id: director_approval
```

## 使用场景

### 内容审核
- 文章发布前审核
- 社交媒体内容检查
- 营销材料批准
- 用户生成内容审查

### 数据验证
- 提取数据确认
- 表单信息核对
- 财务数据审核
- 合同信息验证

### 审批流程
- 费用报销审批
- 采购申请审批
- 休假申请审批
- 项目立项审批

### 异常处理
- 错误情况决策
- 边缘案例处理
- 质量问题解决
- 冲突调解

## 最佳实践

### 1. 表单设计
- 提供清晰的上下文信息
- 使用 Markdown 格式化内容
- 预填充已知信息
- 只要求必要的输入

### 2. 操作按钮
- 使用清晰的标签
- 提供 2-4 个选项
- 区分主要和次要操作
- 考虑所有可能的路径

### 3. 超时设置
- 根据任务紧急程度设置
- 提供合理的响应时间
- 实现超时备用方案
- 发送提醒通知

### 4. 交付方式
- Web App 适合当前用户
- Email 适合多人协作
- 考虑响应时效性
- 选择合适的通知渠道

### 5. 推理标签分离

如果引用推理模型的输出，启用"推理标签分离"：

```yaml
# LLM 节点配置
- data:
    type: llm
    model:
      provider: anthropic
      name: claude-3-opus
    enable_reasoning_tag_separation: true
  id: reasoning_llm

# 人工介入节点
- data:
    type: human-input
    form:
      description: |
        **最终答案**: {{#reasoning_llm.text#}}
        # 不会显示思考过程
  id: review
```

## 性能考虑

### 响应时间
- 设置合理的超时时间
- 考虑用户时区
- 提供紧急联系方式
- 实现提醒机制

### 并发处理
- 多个请求可能同时等待
- 考虑资源占用
- 实现队列管理
- 监控待处理请求

## 常见问题

### Q: Web App 和 Email 如何选择？

A:
- 当前用户实时操作 → Web App
- 需要特定人员审批 → Email
- 多人协作 → Email
- 紧急任务 → Web App

### Q: 如何处理超时？

A:
1. 设置合理的超时时间
2. 实现备用处理分支
3. 发送超时通知
4. 记录超时事件
5. 提供手动重试机制

### Q: 如何追踪审批状态？

A:
1. 使用输出变量记录响应者
2. 记录响应时间
3. 保存审批意见
4. 实现审批历史记录
5. 发送状态更新通知

### Q: 多人收到邮件，谁来响应？

A:
- 第一个响应者的输入生效
- 其他人的请求自动关闭
- 考虑使用轮流或指定机制
- 发送响应确认通知

### Q: 如何处理必填字段？

A:
- 必填字段不能被隐藏
- 提供清晰的验证提示
- 使用合理的默认值
- 在描述中说明要求

## 监控和维护

### 监控指标
- 平均响应时间
- 超时率
- 批准/拒绝比例
- 响应者分布

### 维护任务
- 定期审查超时设置
- 优化表单设计
- 更新收件人列表
- 清理过期请求

## 相关节点

- [If-Else 节点](./ifelse.md) - 条件分支
- [Question Classifier 节点](./question-classifier.md) - 问题分类
- [Variable Assigner 节点](./variable-assigner.md) - 变量赋值

## 返回

[工作流 SKILL](../SKILL.md)
