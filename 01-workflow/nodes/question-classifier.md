# 问题分类器节点详解

## 概述

问题分类器是 Dify 工作流节点，基于语义理解智能地将用户输入路由到不同的工作流路径，无需复杂的条件逻辑。

## 核心功能

- 语义理解分类
- 多类别路由
- LLM 驱动的智能判断
- 自定义分类规则
- 对话历史集成

## 工作原理

分类器输出 `class_name` 变量，包含匹配的类别标签，用于后续节点的条件分支。

## 配置结构

### 基础配置

```yaml
- data:
    type: question-classifier
    model:
      provider: openai
      name: gpt-3.5-turbo
    query_variable_selector: ["sys", "query"]
    classes:
    - name: product_inquiry
      description: 产品咨询、功能介绍、价格查询
    - name: technical_support
      description: 技术问题、故障排查、使用帮助
    - name: account_management
      description: 账户相关、订单查询、退换货
    - name: other
      description: 其他类型的查询
  id: classifier
```

## 关键配置元素

### 输入设置

选择要分类的内容：

```yaml
query_variable_selector: ["sys", "query"]  # 用户问题
# 或
query_variable_selector: ["start", "user_input"]  # 自定义输入
```

### 模型选择

根据复杂度选择模型：

**简单分类**:
- GPT-3.5-turbo
- Claude 3 Haiku
- Qwen-Turbo

**复杂分类**:
- GPT-4
- Claude 3 Opus
- Gemini Pro

### 类别定义

创建清晰、描述性的类别：

```yaml
classes:
- name: category_name
  description: 详细描述该类别包含的内容
```

**最佳实践**:
- 使用清晰的类别名称
- 提供具体的描述
- 定义明确的边界
- 避免类别重叠

## 输出变量

```yaml
{{#classifier.class_name#}}  # 匹配的类别名称
```

## 实际案例

### 案例1：客服分类

```yaml
- data:
    type: question-classifier
    model:
      provider: openai
      name: gpt-3.5-turbo
    query_variable_selector: ["sys", "query"]
    classes:
    - name: post_sale_support
      description: |
        售后支持相关问题：
        - 保修政策和期限
        - 退货和换货流程
        - 维修服务
        - 产品质量问题
    - name: product_usage
      description: |
        产品使用相关问题：
        - 设置和配置
        - 功能使用方法
        - 故障排查
        - 操作指南
    - name: purchase_inquiry
      description: |
        购买咨询相关问题：
        - 产品价格
        - 促销活动
        - 库存查询
        - 购买渠道
    - name: other
      description: 不属于以上类别的其他查询
  id: customer_service_classifier
```

**路由示例**:
- "iPhone 14 怎么设置联系人？" → `product_usage`
- "保修期是多久？" → `post_sale_support`
- "现在有什么优惠活动？" → `purchase_inquiry`

### 案例2：内容分类

```yaml
- data:
    type: question-classifier
    model:
      provider: anthropic
      name: claude-3-sonnet
    query_variable_selector: ["start", "content"]
    classes:
    - name: technical_article
      description: 技术文章、教程、代码示例、技术分析
    - name: news_update
      description: 新闻报道、行业动态、产品发布
    - name: opinion_piece
      description: 观点评论、分析文章、专栏文章
    - name: user_story
      description: 用户故事、案例研究、成功案例
  id: content_classifier
```

### 案例3：意图识别

```yaml
- data:
    type: question-classifier
    model:
      provider: openai
      name: gpt-4
    query_variable_selector: ["sys", "query"]
    instructions: |
      分析用户意图，考虑以下因素：
      - 用户的紧急程度
      - 问题的复杂度
      - 是否需要人工介入
    classes:
    - name: urgent_issue
      description: |
        紧急问题，需要立即处理：
        - 系统故障
        - 安全问题
        - 支付失败
        - 数据丢失
    - name: standard_inquiry
      description: |
        标准查询，可以自动处理：
        - 常见问题
        - 信息查询
        - 简单操作指导
    - name: complex_request
      description: |
        复杂请求，需要详细分析：
        - 定制需求
        - 技术咨询
        - 业务合作
    - name: feedback
      description: |
        用户反馈：
        - 产品建议
        - 使用体验
        - 投诉建议
  id: intent_classifier
```

### 案例4：多语言分类

```yaml
- data:
    type: question-classifier
    model:
      provider: openai
      name: gpt-4
    query_variable_selector: ["sys", "query"]
    memory:
      enabled: true  # 启用对话历史
    classes:
    - name: greeting
      description: 问候、打招呼、开场白（任何语言）
    - name: question
      description: 提出问题、寻求帮助（任何语言）
    - name: command
      description: 执行命令、请求操作（任何语言）
    - name: feedback
      description: 提供反馈、评价（任何语言）
  id: multilingual_classifier
```

### 案例5：情感分类

```yaml
- data:
    type: question-classifier
    model:
      provider: anthropic
      name: claude-3-opus
    query_variable_selector: ["sys", "query"]
    instructions: |
      分析用户消息的情感倾向和紧急程度。
      考虑用户的语气、用词和表达方式。
    classes:
    - name: positive
      description: |
        积极正面的消息：
        - 表扬和感谢
        - 满意的反馈
        - 愉快的交流
    - name: neutral
      description: |
        中性的消息：
        - 普通查询
        - 信息请求
        - 客观陈述
    - name: negative
      description: |
        消极负面的消息：
        - 投诉和不满
        - 问题报告
        - 失望的反馈
    - name: urgent
      description: |
        紧急的消息：
        - 严重问题
        - 需要立即处理
        - 情绪激动
  id: sentiment_classifier
```

## 高级功能

### 指令字段

添加详细的分类指南：

```yaml
instructions: |
  分类时请考虑：
  1. 用户的主要意图
  2. 问题的紧急程度
  3. 是否需要专业知识
  4. 历史对话上下文

  特殊规则：
  - 涉及支付问题优先归类为 urgent
  - 技术术语较多归类为 technical_support
  - 情绪激动的消息归类为 complaint
```

### 记忆集成

启用对话历史以提高准确性：

```yaml
memory:
  enabled: true
  window_size: 5  # 保留最近5轮对话
```

### 下游使用

#### 路由到专业知识库

```yaml
# 分类后根据类别检索不同知识库
- data:
    type: if-else
    conditions:
    - variable_selector: ["classifier", "class_name"]
      comparison_operator: "equals"
      value: "technical_support"
    logical_operator: and
  id: route_technical

# 技术支持分支
- data:
    type: knowledge-retrieval
    dataset_ids: ["tech_kb"]
  id: tech_knowledge

# 产品咨询分支
- data:
    type: knowledge-retrieval
    dataset_ids: ["product_kb"]
  id: product_knowledge
```

#### 应用自定义响应模板

```yaml
# 根据分类使用不同的 LLM 提示词
- data:
    type: llm
    prompt_template:
    - role: system
      text: |
        {% if classifier.class_name == "technical_support" %}
        你是技术支持专家，提供详细的技术解决方案。
        {% elif classifier.class_name == "product_inquiry" %}
        你是产品顾问，介绍产品特性和优势。
        {% else %}
        你是客服助手，提供友好的帮助。
        {% endif %}
  id: llm
```

#### 跟踪查询分布

```yaml
# 记录分类结果用于分析
- data:
    type: code
    code: |
      def main(classifier_result):
          # 记录分类统计
          log_classification(classifier_result)
          return {"logged": True}
  id: log_stats
```

## 最佳实践

### 1. 类别设计
- 定义非重叠的类别
- 使用具体的描述
- 包含示例场景
- 保持类别数量合理（3-8个）

### 2. 模型选择
- 简单分类：使用快速模型
- 复杂分类：使用强大模型
- 考虑成本和延迟
- 测试不同模型的效果

### 3. 描述优化
- 提供清晰的边界
- 包含关键词和短语
- 添加正面和负面示例
- 定期更新描述

### 4. 测试和优化
- 使用真实数据测试
- 处理边界情况
- 监控分类准确性
- 根据反馈调整

### 5. 错误处理
- 添加 "other" 类别
- 处理模糊的输入
- 提供降级方案
- 记录分类失败

## 常见问题

### Q: 如何提高分类准确性？

A:
1. 优化类别描述
2. 使用更强大的模型
3. 添加详细的指令
4. 启用对话历史
5. 提供更多上下文

### Q: 类别太多怎么办？

A:
1. 合并相似类别
2. 使用层级分类（先粗分再细分）
3. 考虑使用多个分类器
4. 简化分类逻辑

### Q: 如何处理模糊的输入？

A:
1. 添加 "other" 或 "unclear" 类别
2. 使用更强大的模型
3. 启用对话历史获取更多上下文
4. 实现二次确认机制

### Q: 分类器和 If-Else 节点有什么区别？

A:
- 分类器：基于语义理解，适合自然语言
- If-Else：基于规则判断，适合结构化数据

### Q: 如何优化分类成本？

A:
1. 使用较小的模型
2. 缓存常见查询的分类结果
3. 先用规则过滤明显的情况
4. 批量处理分类请求

## 相关节点

- [If-Else 节点](./if-else.md) - 基于规则的条件分支
- [LLM 节点](./llm.md) - 处理分类后的内容
- [知识检索节点](./knowledge-retrieval.md) - 根据分类检索不同知识库

## 返回

[工作流 SKILL](../SKILL.md)
